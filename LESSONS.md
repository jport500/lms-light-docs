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


## LESSONS: extend credential rule to cover script location

Credential-touching scratch scripts must run from outside the web
docroot, not just respect presence-and-length output handling. Output
hygiene without location hygiene leaves a window where a script
reading credentials is web-accessible. First flagged during P0-A
verification of aiplacement_aiagent (2026-04-24): groq_probe.php was
placed in local/welcomeemail/ to read configured Groq provider config;
output was correct, location was not. No leak occurred.

## `is_siteadmin()` and `has_capability('moodle/site:config', …)` are not equivalent

Two Moodle APIs answer the question "is this user an administrator," and
they answer it differently:

- **`is_siteadmin($user)`** reads `$CFG->siteadmins` (a comma-separated
  string of user IDs maintained through *Site administration → Users →
  Permissions → Site administrators*) and returns true iff `$user` is in
  that list.

- **`has_capability('moodle/site:config', $context, $user)`** also returns
  true for `$CFG->siteadmins` membership (Moodle short-circuits site admins
  to true inside `has_capability`), **plus** returns true for any user
  assigned a role at `$context` (or a parent context) that grants
  `moodle/site:config`. The latter is the path `is_siteadmin()` does not
  cover.

**The pattern:** a security boundary that gates on "is this user an
administrator?" using `is_siteadmin()` will be silently bypassed by an
operator who creates a custom role granting `moodle/site:config` and
assigns it at system context — a legitimate Moodle action with no
security alarm bells, and one Moodle's own admin pages explicitly support.

**The lesson:** for any boundary check whose semantics are "block users
who can administer the site," use `has_capability('moodle/site:config',
\context_system::instance(), $user)`. Reserve `is_siteadmin()` for cases
where the literal `$CFG->siteadmins` list is the intended population
(e.g., "show this prompt only to the named site admins, not to other
admin-capable users").

**Verification (PHPUnit):** the difference is testable end-to-end. Build
a system-context role granting the capability, assign a user, and assert
both APIs disagree:

```php
$context = \context_system::instance();
$roleid = create_role('test_admin', 'test_admin', 'Test');
assign_capability('moodle/site:config', CAP_ALLOW, $roleid, $context->id, true);
role_assign($roleid, $user->id, $context->id);

$this->assertTrue(has_capability('moodle/site:config', $context, $user));
$this->assertFalse(is_siteadmin($user));
```

If your security check uses `is_siteadmin()` and that pair of asserts
holds, the check is bypassable.

**Discovered:** during `local_demoaccess` v0.1.0, building the Layer 4
guard (refusing demo-account login for any user with `moodle/site:config`).
The SPEC required the capability API explicitly; testing both code paths
is what made the distinction concrete enough to bank as a lesson.

---

## PHPUnit 11 still accepts `@covers` PHPDoc; PHPUnit 12 will not

Moodle 5.1 ships with PHPUnit 11.5. Test classes today use `@covers` in
the class docblock to declare what they exercise, e.g.:

```php
/**
 * @covers \local_welcomeemail\template\resolver
 */
final class resolver_test extends \advanced_testcase { … }
```

Running such a test under PHPUnit 11 with `--display-phpunit-deprecations`
prints:

> Metadata found in doc-comment for class …. Metadata in doc-comments is
> deprecated and will no longer be supported in PHPUnit 12. Update your
> test code to use attributes instead.

PHPUnit 12 will drop the doc-comment form entirely. The replacement is
the attribute form:

```php
use PHPUnit\Framework\Attributes\CoversClass;

#[CoversClass(\local_welcomeemail\template\resolver::class)]
final class resolver_test extends \advanced_testcase { … }
```

**The pattern:** every jport500 plugin's test suite uses `@covers` in
PHPDoc. Each one will need migration before the Moodle release that
bumps to PHPUnit 12.

**The lesson:** treat this as a coordinated cross-plugin cleanup, not a
per-plugin drive-by. Migrating one plugin's tests in isolation buys
nothing — the deprecation is silent in PHPUnit 11, and the timing is
forced by the Moodle PHPUnit bump, not by us. A single LESSONS entry
across the plugin set lets us schedule one sweep at the right moment.

**The trap:** the deprecation only surfaces with
`--display-phpunit-deprecations`. Default `vendor/bin/phpunit` runs hide
it under the "but there were issues!" line at the bottom. Watch for it
when reviewing PHPUnit output during phase wrap-ups; it's easy to miss.

**Plugins affected (verified 2026-05-08):** `local_welcomeemail`
(`tests/resolver_test.php:33` and likely siblings), `local_demoaccess`
(`tests/guard_test.php:33`). Other plugins in the jport500 set should be
audited for `@covers` usage in the same sweep.

**Discovered:** during `local_demoaccess` v0.1.0 PHPUnit run; same
deprecation appears on identical convention in `local_welcomeemail`.
Decision recorded in `local_demoaccess/docs/DECISIONS.md` to match
existing convention rather than migrate solo.

---

## Fail-open semantics require narrow exception catches

Plugins that implement fault-tolerance — especially "if the cache is down, allow the request through" or "if the external service errors, return a default" — pair the tolerance with a `try { ... } catch (...) { fail open }` block. The instinct is to catch broadly: `catch (\Throwable $e)` or `catch (\Exception $e)`, on the reasoning that *any* unexpected error in the try block should be treated as a transient failure.

That instinct is the bug.

**Three layers can combine into silent breakage** when fail-open is paired with a broad catch:

1. A defensive default in some configuration. (Example from `local_demoaccess` v0.4.0: `db/caches.php` declared `simplekeys => true`, the secure-by-default option.)
2. The defensive default rejecting valid plugin behavior. (The rate limiter uses composite keys like `'ip:1.2.3.4'` containing colons, which the simple-key validator throws `coding_exception` on.)
3. The broad catch swallowing the resulting *coding* error as if it were *infrastructure* failure. (`catch (\Throwable $e)` matched `coding_exception`; the fail-open path then unconditionally allowed every request.)

Net result: the plugin would have shipped with rate limits silently disabled — passing all unit tests (every `assertNull` succeeded because every check returned `null` from the fail-open path), passing `phpcs` (the broad catch is syntactically valid), and failing only in production where there was no signal.

In the local_demoaccess case the bug was caught because a diagnostic test was added during iteration to surface what the cache was actually doing. Without that diagnostic, the broken state would have been indistinguishable from the working state.

**The pattern:** fail-open semantics live or die by their predictability. An operator reading the SPEC expects fail-open to trigger only on the specific failure mode the SPEC promises to handle. A broad catch turns *every new code path inside the try block* into a place where fail-open can silently activate — and worse, programmer errors stop being errors, they become silent traffic-let-through events.

**The lesson:** anywhere fault-tolerance is paired with broad exception handling, the broad catch is the bug. Narrow the catch to the exact exception type that represents the failure mode you intend to handle. Coding errors, type errors, configuration mistakes, and unrelated exceptions must propagate normally — they're how you find out you have a bug.

In the local_demoaccess case the fix was a one-line change:

```php
// Before — masks programmer errors as infrastructure failure:
} catch (\Throwable $e) {
    debugging('cache failed, failing open', DEBUG_DEVELOPER);
    return null;
}

// After — only infrastructure-level cache failures fail open:
} catch (\cache_exception $e) {
    debugging('cache failed, failing open', DEBUG_DEVELOPER);
    return null;
}
```

**The discipline:** when writing or reviewing any `catch` block paired with fault-tolerant fallback, ask explicitly: *which exception types does the SPEC promise to handle by failing open? Catch only those.* If you cannot name the exception types, you do not have fail-open semantics — you have "swallow everything," which is never what you want.

**Verification:** for every fail-open catch in a plugin, a test should exist that demonstrates the failure mode actually triggers the fail-open path (e.g. `local_demoaccess`'s `test_unavailable_cache_fails_open` builds a stub that throws `\cache_exception` specifically). Tests that pass through generic exceptions catch nothing useful — they prove "the catch is at least this broad," not "the catch handles the right thing."

**See also:** *"Verify your narrowness assumptions; PHP's class hierarchy can betray you"* — extends this principle one step earlier, into how to verify a candidate narrow-catch is actually narrow.

**Discovered:** during `local_demoaccess` v0.4.0, when the rate-limiter test suite reported "every limit returns null" and a diagnostic test surfaced the `coding_exception` being swallowed by the original `catch (\Throwable $e)`.

**Generalizes to:** every plugin in the LMS Light set that catches exceptions for fault-tolerance — auth plugins gracefully degrading on external IdP outage, webhook handlers absorbing transient delivery failures, AI provider integrations falling back when the model is unreachable, scheduled-task wrappers that don't want to crash the runner. Each of these has a fail-open or fail-soft path; each must catch only the specific exception types that represent its intended failure mode.

**Related security note:** the same principle applies in reverse for fail-closed paths. If the SPEC says "refuse on any auth failure," narrow the catch to the auth-specific exception types and let unrelated exceptions surface — otherwise an unrelated bug presenting as an exception silently produces a refusal, masking the bug.

---

## Verify your narrowness assumptions; PHP's class hierarchy can betray you

> Direct extension of *"Fail-open semantics require narrow exception catches."* That entry establishes the principle: catch only the specific failure-mode exception, never `\Throwable`. This entry establishes how to verify that a candidate narrow-catch is actually narrow.

A narrow-catch is only as narrow as the exception class's parent chain allows. PHP's class hierarchy — and Moodle's, layered on top of PHP's — can make a "narrow" catch silently broad if the class you intended to catch is itself a parent of unrelated classes you don't want to catch.

**The trap** (caught at write-time during `local_demoaccess` v0.5.0):

The SPEC proposed `catch (\moodle_exception | \dml_exception)` for the event-firing fail-open path. Surface reading: "catch infrastructure failures specifically." Actual reading, after grep:

```
$ grep -n "extends" lib/classes/exception/coding_exception.php lib/dmllib.php
lib/classes/exception/coding_exception.php:28:class coding_exception extends moodle_exception {
lib/dmllib.php:65:class dml_exception extends moodle_exception {
```

`\dml_exception extends \moodle_exception` — so the second clause of the union is redundant. Worse: `\coding_exception` *also* extends `\moodle_exception`. The "narrow" catch on `\moodle_exception` would have swallowed every coding error in the event-firing path, re-introducing the v0.4.0 trap one iteration later.

The fix was a one-character-per-keyword change: `catch (\dml_exception)` only. The verification was a 30-second grep before the catch was written.

**The pattern:** before writing a narrow catch, grep the inheritance chain of every exception class you intend to catch. Specifically:

- Confirm the exception you want to catch is a leaf class (no unrelated subclasses).
- Confirm no other exception class in the same library/framework extends from it for unrelated reasons.
- If the intended class has children you don't want to catch, narrow further (catch a more specific class) or use a runtime check (`if ($e instanceof \unwanted_subclass) throw $e`).

**The lesson:** banked principles ("narrow your catches") still require local verification. The v0.4.0 LESSONS entry told us to be narrow. v0.5.0's verification step ("but check what 'narrow' actually means in this hierarchy") is what made the principle reliable in a new context.

**Verification:**

```bash
# Before writing `catch (\X)`, list X's children:
grep -rn "extends X" lib/

# And X's parent (to understand what catching X also catches):
grep -A1 "class X" path/to/X.php | head -3
```

If the child list contains anything that represents a programmer error (`\coding_exception`, `\TypeError`, `\AssertionError`), narrow further.

**Discovered:** `local_demoaccess` v0.5.0 implementation. The grep-before-write reflex (banked in the plugin's local LESSONS) caught it before any code shipped. The v0.4.0 banked principle plus the v0.5.0 verification step worked as intended together — the discipline preventing the same trap one iteration later.

**Generalizes to:** every fault-tolerant catch in every Moodle plugin. The hierarchy of `\moodle_exception` (which `\dml_exception`, `\coding_exception`, `\file_exception`, `\cache_exception`, and others extend) is particularly worth knowing. The same trap shape exists in PHP-native classes (`\Error` is the parent of `\TypeError`, `\AssertionError`, etc.).

**See also:** *"Fail-open semantics require narrow exception catches"* — the principle this entry verifies.

---

## Named Moodle constants defined in `lib/setuplib.php` are not available in `config.php`

A small, expensive trap that the LMS Light plugin set will hit again: documenting `config.php` tweaks that reference Moodle's named debug constants produces a fatal error at the operator's first use, but passes every form of automated verification.

**The trap** (surfaced during `local_demoaccess` v1.0 acceptance walkthrough, fixed in the same iteration):

The SPEC, README, and MANUAL_SMOKE all said:

```php
// In config.php — the operator's edit:
$CFG->debug = DEBUG_DEVELOPER;
$CFG->local_demoaccess_dev_override = true;
```

This fatals on the first request:

```
Fatal error: Uncaught Error: Undefined constant "DEBUG_DEVELOPER"
in /var/www/html/moodle/config.php:32
```

`DEBUG_DEVELOPER` is defined at `lib/setuplib.php:40`. `lib/setuplib.php` loads through `lib/setup.php`, which `config.php` requires at its end:

```php
// config.php structure:
$CFG = new stdClass();
// ... operator-edited values ...
require_once(__DIR__ . '/lib/setup.php');  // <-- loads setuplib.php here
```

So the operator's `$CFG->debug = DEBUG_DEVELOPER;` reference runs *before* the constant is defined. PHP fatals on the undefined constant.

**Why this trap is invisible to automated verification:**

- Unit tests never touch `config.php` — Moodle's PHPUnit bootstrap runs `setup.php` first, so `DEBUG_DEVELOPER` is defined long before tests look at `$CFG->debug`.
- `phpcs` reads PHP syntactically; an undefined constant at runtime is not a syntax error.
- Real-HTTP smoke at every `local_demoaccess` iteration prior to v1.0 didn't catch it either, because none of those iterations deliberately exercised the dev-override-active-with-mismatched-allowlist path. The walkthrough step that exercises it is the only step that runs the operator's exact `config.php` edit.

Across five iterations of design and review (v0.1.0 → v0.5.0), three documents (SPEC, README, MANUAL_SMOKE) all said the same wrong thing. None of the standard gates caught it. Only the v1.0 deliberate end-to-end walkthrough surfaced it.

**The fix:** use PHP-native constants that *are* defined at parse time, in expressions that evaluate to the same numeric value Moodle's named constant resolves to:

```php
// In config.php — correct:
$CFG->debug = (E_ALL | E_STRICT);  // = 32767, same value as DEBUG_DEVELOPER at runtime
$CFG->local_demoaccess_dev_override = true;
```

`(E_ALL | E_STRICT)` is the canonical Moodle convention for this exact case. `E_ALL` and `E_STRICT` are PHP-native constants defined at parse time; the bitwise OR evaluates to 32767, identical to `DEBUG_DEVELOPER`'s runtime value.

The plugin's runtime check (`(int) $CFG->debug === DEBUG_DEVELOPER`) is unaffected — that runs after `setup.php` has loaded, where `DEBUG_DEVELOPER` is defined.

**The pattern:** any documentation that asks an operator to edit `config.php` must restrict itself to constants defined at parse time. PHP-native constants (`E_ALL`, `E_STRICT`, `M_PI`, etc.) are always available. Moodle constants are NOT available unless they live in a file loaded before `config.php` (a small set; the bulk of Moodle constants live in `setuplib.php` or later).

**The lesson:** when documenting `config.php` tweaks, verify the example by running it. Don't trust your knowledge of which constants are "Moodle constants" vs "PHP constants" — the distinction matters at parse time and the boundary is not where you'd guess (Moodle defines a few constants in `config-dist.php`'s preamble; everything else is post-`setup.php`). The cost of the verification is one execution; the cost of skipping it is operators hitting a fatal at first use of the documented feature.

**Verification:**

```bash
# Drop the documented config.php edit into a clean Moodle's config.php,
# then run any CLI script that loads config.php. If the constant is
# undefined at parse time, PHP fatals and the script halts immediately.
ddev exec 'php /var/www/html/moodle/admin/cli/cfg.php 2>&1 | head -3'
```

If a documented operator edit is going to fatal, this script surfaces it in two seconds.

**Constants commonly mistaken as parse-time-available** (all of these are post-`setup.php`):

- `DEBUG_DEVELOPER`, `DEBUG_ALL`, `DEBUG_NORMAL`, `DEBUG_NONE`
- `MUST_EXIST`, `IGNORE_MISSING`, `IGNORE_MULTIPLE`
- `MOODLE_INTERNAL` (this one is actually defined slightly later in setup.php's flow)
- `CONTEXT_SYSTEM`, `CONTEXT_COURSE`, etc.
- `FORMAT_HTML`, `FORMAT_PLAIN`, etc.

**Discovered:** `local_demoaccess` v1.0 acceptance-criteria walkthrough. Caught by deliberately running SPEC criterion 5 ("real-HTTP exercise of the dev override") end-to-end. The walkthrough specifically tested an operator's exact `config.php` edit; nothing else in the build sequence had.

**Generalizes to:** every plugin in the LMS Light set that documents `config.php` tweaks. `auth_magiclink` may have similar exposure if it documents `$CFG->forced_plugin_settings` examples; future plugins that introduce their own `$CFG->*` flags will hit this if they document them with named Moodle constants.

---

## Match upstream test conventions; don't solve them solo

When PHPUnit (CLI_SCRIPT) calls `complete_user_login()`, it warns: `session_regenerate_id(): Session ID cannot be regenerated when there is no active session`. The test passes (the warning isn't fatal), but PHPUnit reports it under "warnings" in the run summary.

Two ways to handle this in plugin tests:

**Match upstream:** prefix the call with `@` — Moodle's own tests already do this. See `public/lib/tests/task/send_login_notifications_test.php:45`:

```php
$SESSION->isnewsessioncookie = true;
@complete_user_login($loginuser);
```

**Solve solo:** initialize an empty PHP session in the test before the call, e.g. `\core\session\manager::init_empty_session()`, so `session_regenerate_id()` has something to regenerate.

The "solve solo" path looks cleaner — it suppresses the warning at the root cause rather than blanketing it. But it diverges from upstream: the moment Moodle changes how the session manager interacts with PHP sessions in tests (an entirely plausible 5.x → 6.0 detail), every plugin that hand-rolled `init_empty_session()` has its own forward-port problem to solve. The plugins that matched upstream's `@` get the fix for free with the next composer update.

**The pattern:** when an upstream test convention exists for a known test-environment quirk — even one that looks like a hack — match it. The convention encodes context the plugin author doesn't have: *upstream knows where this quirk surfaces, has weighed alternatives, and will be the one to update it when the underlying API changes.*

**The lesson:** "cleaner than upstream convention" is a trap. It inherits maintenance debt the plugin doesn't have to carry. Solo-fix only when (a) upstream convention is genuinely wrong (rare) or (b) no upstream convention exists for this case yet.

**Verification:** before adding a workaround for a test-environment warning, grep the Moodle test corpus for the same warning text or the same root function call:

```
grep -rn "@complete_user_login\|session_regenerate_id" \
  public/lib/tests/ public/lib/classes/
```

If upstream is already handling it, copy their handling. If they aren't, you may have found something genuinely new — and at that point a Moodle Tracker issue is the better contribution than a plugin-local workaround.

**Discovered:** surfaced during `local_demoaccess` v0.2.0 implementation; banked as a LESSONS candidate at review time rather than recorded as a DECISIONS entry, since the conclusion (match upstream) is portable engineering knowledge rather than plugin-specific architecture.

**Generalizes to:** any test-environment-specific quirk where Moodle core's own tests have established a workaround. The same logic applies to Mailpit-vs-mocked-SMTP setup, MUC-cache initialization in tests, event-redirect handling, and any future test-environment surface that behaves differently from production.

---

## `admin_setting_*` base classes are not auto-loaded for PHPUnit

Plugins extending Moodle's admin settings widgets (for example, custom subclasses of `admin_setting_configtext`, `admin_setting_configcheckbox`, or `admin_setting_heading`) hit a class-resolution failure the moment a unit test instantiates them:

```
Error: Class "admin_setting_configtext" not found
```

These base classes live in `lib/adminlib.php` rather than under `lib/classes/`. Moodle loads `adminlib.php` only when the admin tree is being rendered — request-time only. The PHPUnit bootstrap does not load it. So a test that does `new my_validating_setting(...)` fails on class resolution before any assertion runs, even though the file exists, the autoloader is correct, and PHPCS is clean.

**The pattern:** any plugin test file that touches an `admin_setting` subclass needs to explicitly require `adminlib.php` at the top:

```php
namespace my_plugin\admin;

defined('MOODLE_INTERNAL') || die();

global $CFG;
require_once($CFG->libdir . '/adminlib.php');
```

The require_once is on the test file, not on the class file under `classes/admin/`. The class file only needs the parent class to be loaded *at instantiation time*; production code path always renders through `settings.php` which lives behind `$hassiteconfig` (which means adminlib.php has loaded). The test path is the only one where the load order is wrong.

**The lesson:** "PHPUnit bootstrap loads what request-time loads" is false. Anything that lives in `lib/<file>.php` outside the autoloader's classes/ tree must be manually required by the test. `adminlib.php` is the most common offender; `weblib.php` (some helper functions), `enrollib.php`, and `gradelib.php` have similar exposure.

**Verification:** before adding any test for an admin_setting subclass, add the require_once at the top of the test file. Cheaper than diagnosing the misleading "Class … not found" error after the fact.

**Discovered:** during `local_demoaccess` v0.3.0, when the new `tests/admin/*_test.php` files all failed with admin_setting_* not found. Fixed in three places by adding `require_once($CFG->libdir . '/adminlib.php');` at the top of each test file.

**Generalizes to:** every plugin in the LMS Light set that ships custom admin-settings widgets. None do today; future plugins built on the same pattern will hit this on first PHPUnit run.

---

## `resetAfterTest()` does not reset custom `$CFG->...` properties

Moodle's `advanced_testcase::resetAfterTest()` resets the database, the moodledata directory, MUC caches, and `$CFG` keys it knows about (those backed by the `config` and `config_plugins` tables, plus the bootstrap defaults). It does **not** unset arbitrary properties added to the `$CFG` global at runtime.

Plugins that read feature toggles or dev-override flags from `$CFG->some_custom_property` will see these properties leak across tests:

```php
public function test_a(): void {
    $this->resetAfterTest();
    $CFG->local_demoaccess_dev_override = true;
    // ... assertions ...
}

public function test_b(): void {
    $this->resetAfterTest();
    // $CFG->local_demoaccess_dev_override is STILL true here.
    // assertion expecting "no override active" fails surprisingly.
}
```

Symptoms: tests pass when run individually (`--filter test_b`) but fail in the full suite. Errors look like state-dependent assertions ("expected X, got Y") with no obvious cause in the test under inspection.

**The pattern:** plugins using `$CFG` for plugin-specific runtime flags must reset those properties explicitly in `setUp()`:

```php
protected function setUp(): void {
    parent::setUp();
    global $CFG;
    unset($CFG->my_plugin_dev_override);
}
```

Reset before each test, not after. resetAfterTest() owns the after.

**The lesson:** the test framework is not omniscient about plugin-specific `$CFG` extensions. The plugin owns its own config surface and must own its own test isolation for that surface. If a custom property appears in a plugin's runtime code, a corresponding `unset()` belongs in the plugin's test setUp.

**Worth noting about `$CFG->debug`:** this one IS a Moodle-known property, but `resetAfterTest()` doesn't reset it because PHPUnit's default debug level is set during bootstrap and persists. Tests that mutate `$CFG->debug` (typically to `DEBUG_DEVELOPER` to exercise debug-conditional code) need to reset it manually.

**Verification:** for any test that writes to `$CFG->...` for a plugin-specific or debug-related property, write a corresponding unset/reset in setUp the same hour. Cheaper than diagnosing intermittent test failures driven by run order.

**Discovered:** during `local_demoaccess` v0.3.0, when the new dashboard tests would pass individually but fail in the full suite because `$CFG->local_demoaccess_dev_override` and `$CFG->debug` were leaking from `guard_test`'s dev-override test cases. Fixed by adding `setUp()` to both `guard_test.php` and `safety_dashboard_setting_test.php`.

**Generalizes to:** every plugin that uses `$CFG->...` for configuration outside the `config_plugins` table — feature flags, dev overrides, environment-specific flags injected from `config.php`, anything `is_*_enabled()` predicates hard-code against `$CFG`.

---

## Anonymous-class test stubs need explicit per-method docblocks

PHPUnit-friendly test stubs are sometimes built as anonymous classes extending a Moodle base class:

```php
$brokencache = new class extends \cache_application {
    public function __construct() { /* skip parent */ }
    public function get($key, $strictness = ...) { throw new \cache_exception(...); }
    public function set($key, $data) { throw new \cache_exception(...); }
};
```

Moodle's phpcs ruleset requires per-method docblocks even on anonymous classes. `phpcs:disable Squiz.Commenting.FunctionComment` directives inside the anonymous class do not appear to be honored — phpcs still flags `Missing docblock for function get in testcase`. The fix is to write actual docblocks:

```php
$brokencache = new class extends \cache_application {
    /** Skip parent constructor: avoids instantiating a real MUC backend. */
    public function __construct() {}

    /**
     * Throws to simulate a broken MUC store.
     *
     * @param string|int $key Cache key (unused).
     * @param int $strictness Cache strictness (unused).
     * @return mixed Never returns; always throws.
     */
    public function get($key, $strictness = \cache_store::DEREFERENCES_OBJECTS) {
        throw new \cache_exception('simulated cache failure');
    }

    /**
     * Throws to simulate a broken MUC store.
     *
     * @param string|int $key Cache key (unused).
     * @param mixed $data Cache data (unused).
     * @return bool Never returns; always throws.
     */
    public function set($key, $data) {
        throw new \cache_exception('simulated cache failure');
    }
};
```

**The pattern:** Moodle CS treats anonymous-class methods like any other method — full docblock with one-line description, `@param`s, `@return`. `phpcs:disable` annotations don't help here.

**The lesson:** when building anonymous class stubs for tests, plan for docblocks from the start. Three short docblocks for a test stub take 30 seconds to write and one minute to debug if you skip them.

**Discovered:** during `local_demoaccess` v0.4.0, while writing the `test_unavailable_cache_fails_open` test (anonymous `\cache_application` subclass throwing on `get`/`set`).

**Generalizes to:** every plugin that uses anonymous classes as test fixtures — particularly plugins testing fault-tolerance against cache backends, repository interfaces, observer dispatchers, and other abstract Moodle infrastructure surfaces.
