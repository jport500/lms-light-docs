# Process Lessons — from previous LMS Light SAD projects

This document captures lessons learned during the plugin's development. Not about *what* was built but about *how* it was built — the rhythm, the failure modes, and the patterns that worked.

These lessons are portable. They should apply to future plugin work, future AI-assisted development, and future supervised-dev projects generally. When you start the next project, read this first.

---

## Supervised AI-assisted development — the rhythm

The plugin was built across five phases using Claude Code in supervised mode. The working pattern: explicit phase boundaries, review gates between phases, decisions surfaced before implementation, pause-verify-commit as the default cadence.

**What "supervised mode" actually meant in practice:**

- Every phase started with Claude Code surfacing 3–10 decision questions before writing any code.
- Every phase ended with a commit, a verification report, and an explicit pause for human review.
- Every significant deviation from spec got surfaced, not buried.
- Every verification finding got reported honestly, including "I did X; is that what you meant?" when interpretation was ambiguous.

**Why this worked:**

The rhythm caught real issues at every phase:
- Phase 2: `\core\event\user_confirmed` doesn't exist (spec bug).
- Phase 3: XSS test fixture used Moodle's user generator which scrubs HTML (test-setup bug).
- Phase 4: Two interpretation gaps — `skip_no_email` "display as read-only" and `test_send` "confirmation prompt" — both narrower than intended.
- Phase 4 follow-up: Test-send template dropdown was decorative (integration bug).
- Phase 5: Manual smoke surfaced admin nav consolidation and save-feedback issues.

None of these would have been caught by automated tests alone. The verification step is where ambiguity gets resolved and integration gaps get noticed.

**The cost:** many rounds of messages, many pauses, visible verification output. The first-order cost is real.

**The benefit:** no silent defects shipped, no surprise decisions landed, clean git history, honest scope tracking. The second-order benefit dwarfs the first-order cost.

**When to use this rhythm:** Production work, security-sensitive work, anything with a real cost of being wrong. Explicitly not worth it for experiments, prototypes, or throwaway scripts.

---

## Specifications must verify referenced APIs exist

The `\core\event\user_confirmed` assumption survived four spec drafts (v0.1 through v0.4) because the event is what a reasonable framework *should* emit. Moodle doesn't. Phase 2 Claude Code caught it with a 30-second `find lib/classes/event/ -name 'user_*'`.

**Lesson:** During spec drafting, verify that every referenced framework API (events, classes, functions, config keys, capabilities) actually exists in the target version. Don't assume — grep. Add this as a checklist item for future specs:

- [ ] Every referenced event class has been verified to exist in the target Moodle version.
- [ ] Every referenced core API function has been checked in the current codebase, not just moodledev.io docs (docs lag behind source).
- [ ] Every referenced config key has been confirmed to exist in `admin/cli/cfg.php --shows`.
- [ ] Every referenced capability has been checked in the target `db/access.php`.

The cost of this discipline during drafting is small. The cost of discovering an assumption-error mid-implementation is a full rescoping conversation.

---

## Automated tests verify what the author considered

44 green tests, plugin-wide phpcs clean, CLI smoke PASS — all indicated "ready to ship" before the manual walkthrough. The manual smoke surfaced four issues within 20 minutes.

**The pattern:** Automated tests verify the code paths the author thought to write tests for. Humans walking through the UI encounter unfamiliar paths because they follow affordances, not code. Both are needed. Neither substitutes for the other.

**Specific gaps that automated tests missed:**

1. **Admin nav consolidation.** Three `admin_externalpage` registrations were being displayed as three separate entries under Local plugins. No PHPUnit test asserts "the admin tree has one category, not three siblings." Found by human eyes.

2. **Save feedback.** `edit_template.php`'s `redirect()` after save used the 2-arg form, producing a silent success. Post-save experience indistinguishable from "save didn't work." No test asserts "the success banner is visually prominent."

3. **Test-send template dropdown was decorative.** Passed unit tests in isolation for manager and task; gap was in the integration.

4. **Target-user field label.** Accepts username or email; label says "Target user" with no hint. No test asserts "form field labels mention what inputs they accept."

**Lesson:** Before shipping, walk through the UI yourself. Ask at each screen: "if I were a new operator seeing this for the first time, what would I try, what would I expect, what would surprise me?" Write down the surprises. Those are the bugs your tests won't catch.

---

## End-to-end testing requires real transports

PHPUnit's `redirectEmails()` captures the PHP `\stdClass` representation of an outgoing email. It does not capture the bytes that would go on the SMTP wire. That matters because:

- URL encoding bugs in the SMTP transport layer (how `+` becomes `%2B` in practice) pass PHP-object tests and fail real delivery.
- MIME structure issues (multipart alternative, content-type boundaries) can't be observed at the object level.
- Character-encoding issues in headers (RFC 2047 encoded-word, quoted-printable) only manifest after the message hits SMTP.

This project's architectural acceptance test — `%2B` in email body after a `+` email address — was verified at three levels:

1. **Unit test** (`task_test::test_email_body_encodes_plus_as_percent_2b`) using `$this->redirectEmails()`. Passes.
2. **CLI smoke** (`tests/cli_smoke.php`) with `$CFG->noemailever = 1`. Passes.
3. **Real SMTP** through mailpit (`localhost:1025` SMTP sink with web UI inspection). Passes.
4. **Production SMTP** to a real inbox during MANUAL_SMOKE step 16.

The first two catch code-level bugs. The third catches transport-level bugs. The fourth catches deliverability bugs (spam filtering, authentication, TLS).

**Lesson for any project that outputs to external systems (email, webhooks, APIs, message queues):**

- Code-level tests are necessary but insufficient.
- Transport-level testing through a local inspectable sink catches a different class of bug.
- End-to-end testing through the real external system catches yet another class.

All three levels. Not optional.

Tools for local inspectable sinks:
- **Email:** mailpit (bundled with DDEV), MailHog, smtp4dev
- **Webhooks:** webhook.site, RequestBin, ngrok (with inspect)
- **S3:** MinIO, LocalStack
- **DNS:** dnsmasq with custom zones
- **Any HTTP:** mitmproxy, WireMock

---

## Never echo secret-bearing config values

Phase 2 included a credential exposure: Claude Code dumped a live Gmail app password to the conversation transcript while introspecting Moodle's SMTP config. The operator rotated the credential immediately.

**The rule, to be carried into every future session:**

When introspecting any config (Moodle, environment, secrets files, API keys), never echo values for keys matching these patterns:
- `*pass*`, `*password*`
- `*secret*`
- `*token*`
- `*key*` (context-dependent — `public_key` may be fine; `api_key` never)
- `*credential*`
- `*apikey*`

Echo **presence and length** only:
```
smtppass_set=yes (length=19)
api_token_set=no
```

Not:
```
smtppass=xxxxxxxxxxxxxxx
```

The "length" information is useful for diagnostics (correct length confirms non-truncation) without exposing the credential.

This applies to CLI introspection, debug output, log messages, and any code written by Claude (or any assistant). It's a project-agnostic rule; add it to supervised-dev kickoff prompts permanently.

---

## File memory is not authoritative across sessions

Twice in this project, Claude Code's working memory of a file diverged from the file on disk:

1. **Phase 1 start.** A previous session had left partial scaffolding in `public/local/welcomeemail/`. Claude Code didn't know about it and nearly overwrote it.
2. **Phase 3 start.** The spec had been updated from v0.4 to v0.5 between sessions. Claude Code's context held v0.4 and it started Phase 3 against that until told explicitly to re-read the file.

**Lesson:** AI working memory is approximate. When a source-of-truth file may have changed between sessions or between phases, explicit re-read is warranted.

**Concrete rule for future sessions:**

> If you have read a file earlier — in this session or a previous one — re-read it before relying on its content. Disk is authoritative. Memory is not.

This should be part of the standard kickoff prompt for any supervised-dev work.

---

## "Feature exists, UI doesn't advertise it" is a recurring pattern

`test_send.php` accepted either username or email for the target-user field since Phase 4. The form label said "Target user" with no hint. Caught during manual smoke when the operator asked "can you add email support?"

**The pattern:** Developer implements a feature with thoughtful fallback logic, writes tests that prove the fallback works, ships it. Weeks later a user asks "can you add X?" — and X already exists, it's just not discoverable.

**Why this happens:** Implementation and discoverability are separate concerns with separate failure modes. Tests catch functional regressions. Nothing catches "the label doesn't mention a feature."

**Lesson:** When reviewing any form field or UI element, ask two separate questions:
1. Does it work correctly? (functional check — tests cover this)
2. Does the operator *see* what it can do? (discoverability check — human review only)

For a form field specifically: does the label tell the operator what inputs are accepted and what actions are available? If not, fix the label.

---

## Clipboard scaffolding is a real failure mode

During MANUAL_SMOKE walkthrough, the operator pasted an HTML template body copied from a Notion document. Notion's clipboard wrapped the paste in `notion-inline-code-container` classes and HTML-entity-escaped the angle brackets. The Moodle WYSIWYG editor accepted the payload verbatim. The rendered email showed literal `<p>` tags in Gmail.

**The root cause was operator workflow, not a plugin bug.** But it surfaced a legitimate question about defensive design.

**Lesson:** When a form accepts structured content (HTML, markdown, JSON), consider how it behaves when an operator pastes from:
- A WYSIWYG source (Notion, Google Docs, web browsers)
- A different formatting convention (Markdown into HTML, HTML into Markdown)
- A terminal (with ANSI escape codes)
- A code editor (with smart quotes, auto-formatting)

Each of these can silently corrupt the payload. Options for defense:
- Plain textarea instead of WYSIWYG (forces raw, obvious in/out)
- Paste sanitization on submit (detect known-bad scaffolding patterns)
- Post-save validation with preview (show the operator what got stored)

For this plugin, the operator correctly identified the Notion paste as a one-time artifact and not a chronic problem. Pattern worth remembering for projects where operator content gets displayed to end users verbatim.

---

## Verification steps aren't ceremony

During Phase 5 alone, there were five explicit pauses:
1. Clean-room verification before starting work.
2. Privacy provider review before moving to documentation.
3. Three-document review after writing README/CHANGES/MANUAL_SMOKE.
4. CLI smoke execution review.
5. Version bump review before commit.

Each pause surfaced real issues:
- Pause 3 caught a missing "Do" step in MANUAL_SMOKE step 12, a broken SMTP reroute command (quoting bug), and the absence of a real-SMTP verification step.
- Pause 4 confirmed teardown cleanliness.
- Pause 5 let the ceremonial MATURITY_STABLE bump get eyeballed once before it landed.

**Lesson:** The pause-verify-commit rhythm is where interpretation mismatches get caught before they land in production. It looks ceremonial. It isn't. The alternative — "implement all of phase X and show me the commit" — merges a dozen interpretations into one review. Issues get missed. Some reach the next phase. Some reach production.

**Specifically:** after saying "go" on a commit, the next message should be commit output. If it's anything else (a new question, a verification result, a diagnostic), notice that the commit hasn't happened yet and decide whether to revisit or let the new thing jump the queue.

---

## Severity calibration — "is this a bug" vs "will this happen to others"

During MANUAL_SMOKE, three observations surfaced. My initial call on observation 3 (HTML-as-literal-tags in inbox): ship-blocker. The operator's call after reflection: not a plugin bug, just a one-time Notion-paste artifact from how they happened to be reading the walkthrough.

**The operator was right.** My severity call was based on "this output looks broken" without asking "will this happen to someone who isn't doing what just happened here."

**Lesson:** At release time, the right question isn't "is this problem real" but "will this problem happen to someone who isn't me, doing the exact thing I just did." Operator context is authoritative for severity calls. The person who knows the deployment context, the operator population, and the workflow patterns gets final say on whether a finding ships as a bug or as a known quirk.

For AI assistants involved in review: flag findings honestly, but defer to the operator on severity when the context is outside your observation.

---

## Projects need durable memory outside the chat

This document exists because conversation transcripts aren't a reliable long-term repository. Future-you, a future collaborator, or a future AI in a new session needs durable access to:

- Why specific architectural decisions were made (captured in `DECISIONS.md`)
- Process lessons and failure modes (captured in this file)
- Current specification state (captured in `SPEC.md` or external)
- What ships and how to use it (captured in `README.md`)
- What's changed (captured in `CHANGES.md`)

**The general principle:**

Think of each conversation as a workspace, not a filing cabinet. What's discussed is working memory. The repo is long-term memory. Moving durable knowledge from workspace to filing cabinet before the workspace closes is part of the work, not an optional bonus.

**Practical rhythm for future major work:**

1. Before starting a new major work cycle, update DECISIONS.md / LESSONS.md / SPEC.md with anything significant learned since last cycle.
2. Start the new cycle in a fresh conversation with a hand-off prompt pointing at the repo's docs.
3. Develop with the phase-pause-commit rhythm.
4. Update docs as you go, with doc updates in their own commits.
5. Repeat.

New sessions should orient by reading the repo. Not by reading conversation history.

---

## What would make v1.1 work better

Meta-lessons specifically for future revisions of this plugin:

1. **Form-submission integration tests.** CLI smoke exercises the backend chain but not the form-layer. A v1.1 addition: scripted form submission tests (via `curl` + cookie jar, similar to what manual-smoke verification did for the admin nav) that catch the "dropdown is decorative" class of bug.

2. **UI audit checklist.** A companion to the functional test suite. For each admin page, explicit checks: does every form field label describe accepted inputs, does every destructive action confirm, does every success produce a clear signal. Can be a `UI_AUDIT.md` document reviewed at phase boundaries, not just at release.

3. **Separate kickoff prompts for major vs minor revisions.** The Phase 1 kickoff for v1.0 was detailed and rigorous because nothing existed. For v1.1, the kickoff should be lighter — existing patterns to follow, existing docs to reference, scope constrained to the four items.

4. **Spec revision with every major feature.** v1.0 had five spec revisions (v0.1 through v0.5) because the design evolved during development. For v1.1, aim to settle the spec before Phase 1 starts. If the spec evolves mid-development, that's a signal that something was under-thought.
