# Opening prompt — mod_scorecard Moodle 5.2 compatibility review

Paste this as the FIRST message in a fresh claude.ai conversation. The conversation strategist (you-equivalent in the new session) drafts the audit and remediation kickoff prompts; Claude Code in the separate session executes them.

---

## Prompt to paste

```
I'm starting a Moodle 5.2 compatibility review for mod_scorecard. mod_scorecard is a Moodle activity-mod plugin built under a supervised-agentic methodology — currently at v0.6.0, last shipped 2026-04-28, tested and passing on Moodle 5.1.3.

I'll be running implementation in a separate Claude Code session against the local repo at `/Users/John/projects/mutms/moodle/public/mod/scorecard/`. Your role in this conversation is strategy: drafting kickoff prompts for Claude Code, dispositioning Q's that surface from pre-flight findings, naming methodology insights worth banking. Same supervised-agentic pattern that produced v0.1.0 through v0.6.0.

## The methodology archive

Read these before drafting anything. They are load-bearing context, not optional reference material.

- `https://github.com/jport500/lms-light-docs/blob/main/MOODLE-ACTIVITY-MOD-PHASES.md` — activity-mod phase progression template extracted from mod_scorecard's six phases. Calibration anchors, sub-step shape, gate-discipline shapes, when-the-template-doesn't-apply boundaries. NOTE: A compatibility review is NOT a phase per the template — it's audit-and-remediate work, fundamentally different shape. The template's methodology disciplines transfer; the phase shape doesn't apply.

- `https://github.com/jport500/lms-light-docs/blob/main/CONTEXT.md` — LMS Light project conventions.

- `https://github.com/jport500/lms-light-docs/blob/main/LESSONS.md` — portable methodology rhythm.

- `https://github.com/jport500/moodle-mod_scorecard/blob/main/docs/PHASE-5B-RETROSPECTIVE.md` — most recent retrospective. Particularly relevant sections for compatibility-review work:
  - Section 3 (three-Q1-reversal pattern revealing maxfiles=0 architectural fact): kickoff defensive defaults can be wrong when architectural state precludes the threat
  - Section 4 (five-shape empirical-bootstrap-state-verification gate discipline): real-data verification catches regressions test fixtures miss
  - Section 5 (course-correction-as-scope refinement): authorize correction in scope rather than direction

- `https://github.com/jport500/moodle-mod_scorecard/blob/main/docs/PHASE-5A-RETROSPECTIVE.md` and `https://github.com/jport500/moodle-mod_scorecard/blob/main/docs/PHASE-4-RETROSPECTIVE.md` — earlier methodology archaeology; reach for these if specific patterns surface (Grade API patterns at Phase 5a; reporting + helper-decomposition patterns at Phase 4).

- `https://github.com/jport500/moodle-mod_scorecard/blob/main/CHANGES.md` — what shipped at each version; v0.6.0 entry has the current quality gates baseline (168 PHPUnit tests / 728 assertions; phpcs zero/zero plugin-wide).

- `https://github.com/jport500/moodle-mod_scorecard/blob/main/docs/SPEC.md` — feature/contract reference; v0.4.2 sha c1ac688608724bf585299e9e2a556947b7608f1ba52a790a19ca2eb6ba903010.

## What's different about compatibility-review work

It's audit-then-remediate, not feature build. Two-stage shape:

**Stage 1: Audit** (read-only). Surface what's broken or warned by Moodle 5.2 against current mod_scorecard at HEAD. Stage 1 outputs a findings document, no code changes. Predicting round-trips is meaningless until findings exist.

**Stage 2: Remediation** (code changes per finding). Drafted against actual audit findings; scope sized to what surfaced. Shape is closer to a small phase — sub-steps, kickoffs, gate verification, retrospective if substantial.

Don't conflate the two. Today's first deliverable is the Stage 1 audit kickoff prompt. Stage 2 kickoff drafting waits for audit findings to surface.

## Constraints worth pre-flagging

- **mod_scorecard MUST continue to work on Moodle 5.1.3.** This is not a "drop 5.1.3 and run on 5.2" upgrade; it's a "supports 5.1.3 and 5.2" compatibility expansion. Two-version support unless something forces hard upgrade (e.g., 5.2 deprecates an API mod_scorecard depends on with no backward-compatible alternative).
- **No production deployments yet** per v0.6.0 CHANGES.md ### Spec status. MATURITY_ALPHA stays unless 5.2 review surfaces a reason to revisit.
- **Test environment.** DDEV at `mutms.ddev.site:8443` is currently running Moodle 5.1.3. The 5.2 review needs a 5.2 test environment — that's its own setup work that may surface as Stage 0 (environment provisioning) before Stage 1 audit can happen.
- **No customer-facing release planned at compatibility-review close** unless remediation is substantial. Expected output: v0.6.1 or similar minor bump if findings are mechanical; v0.7.0 if findings require architectural change.

## What I want you to do first

Don't draft the audit kickoff yet. First:

1. Read the methodology archive linked above. Surface a brief acknowledgment of what you've absorbed — not a summary, just enough that I know we're synchronized on the disciplines and the calibration framing.

2. Surface any pre-flight Q's about the review work itself before drafting the audit kickoff:
   - Is Moodle 5.2 released yet, in beta, or pre-beta? (Affects what we can audit against — a stable 5.2 vs a moving target.)
   - Are there known 5.1 → 5.2 deprecations / API changes in Moodle's release notes I should be aware of? (If yes, the audit kickoff can pre-flag specific surfaces; if no, it's more discovery-shaped.)
   - Is there a Moodle 5.2 testing environment expectation (clean install vs upgrade-from-5.1.3 vs both)? (Affects Stage 0 environment work.)

3. After Q's resolve, draft the Stage 1 audit kickoff prompt for Claude Code. The audit kickoff itself should specify:
   - Pre-flight verification (SPEC sha unchanged at c1ac688...3010; current HEAD; tag list including v0.6.0)
   - Audit scope (what surfaces to inspect: deprecation warnings, API signature changes, namespace changes, schema/upgrade compatibility, PHPUnit fixture API stability, behat scenario stability if any)
   - Output shape (findings document at `docs/MOODLE-5.2-AUDIT.md` — single commit, internal archaeology)
   - What's OUT of scope for Stage 1 (no remediation, no code changes, no test additions; just findings)

The audit kickoff is the deliverable from this opening conversation. After Claude Code runs it and surfaces findings, we draft Stage 2 remediation kickoffs against actual findings.

## Discipline reminders that apply

These are the methodology insights from Phases 4 / 5a / 5b that bear directly on compatibility-review work:

- **Kickoff-evidence-grounding reflex.** Read actual files before drafting accumulated-state framings. The audit kickoff in particular should NOT default to "audit everything everywhere" — it should pre-flight against Moodle's 5.1 → 5.2 release notes (if available) and scope the audit to known-changing surfaces plus a smaller "general defensive sweep" of unchanged surfaces.

- **Empirical-bootstrap-state-verification at gate.** The audit's gate is finding-discovery + finding-confirmation. PHPUnit running clean on Moodle 5.2 is a real signal; phpcs running clean against any new Moodle 5.2 phpcs rules is another; manual install + upgrade walkthrough on 5.2 is a third. The audit findings document should name which signals were verified and which findings are inferred-from-docs vs verified-empirically.

- **Three-Q1-reversal pattern (kickoff defensive defaults are provisional).** If the audit kickoff defaults to "remediate every deprecation warning defensively," pre-flight evidence may show that some deprecations have backward-compatible alternatives that work on both 5.1 and 5.2 (no remediation needed) or some deprecations only fire under specific configurations mod_scorecard doesn't exercise. Verify before defaulting.

- **Course-correction-as-scope.** The audit may surface scope much larger or smaller than anticipated. Authorize correction in scope at kickoff time rather than naming a specific fallback option.

- **Calibration honesty bidirectionally.** The audit may surface zero findings (mod_scorecard already 5.2-compatible — calibration-honest in lower-bound-allowance sense). Or may surface substantial findings (architectural changes needed — calibration-honest in upper-bound-allowance sense). Both are valid outcomes; predict the audit's findings range honestly rather than always anticipating one direction.

Standing by for your acknowledgment + any pre-flight Q's about the review work itself, before drafting the Stage 1 audit kickoff.
```

---

## Why this prompt is shaped this way

**Five intentional structural choices worth calling out so you understand what the prompt is doing:**

1. **It bootstraps the methodology archive explicitly.** A fresh claude.ai conversation has no awareness of Phase retrospectives or MOODLE-ACTIVITY-MOD-PHASES.md unless the opening prompt tells it where to look. The links are durable on origin; the prompt names them as load-bearing context.

2. **It distinguishes audit work from phase work explicitly.** Activity-mod phase template doesn't apply to compatibility review. Naming this prevents the new conversation from force-fitting "we're doing Phase 6" framing onto fundamentally different shape.

3. **It enforces the two-stage discipline at the start.** Audit (read-only findings) before remediation (code changes). The prompt explicitly says "don't draft Stage 2 yet" so the new conversation doesn't conflate audit with remediation.

4. **It pre-flags constraints that affect scope.** Two-version support, no production deployments, MATURITY_ALPHA stays unless 5.2 forces revisit, expected release shape (v0.6.1 vs v0.7.0). These are decisions that affect the audit kickoff's framing; surfacing them at conversation-open prevents them from getting re-litigated mid-audit.

5. **It applies methodology disciplines specifically to compatibility-review surfaces.** Each banked discipline (kickoff-evidence-grounding, empirical verification, Q1-reversal pattern, course-correction-as-scope, calibration honesty bidirectional) has a compatibility-review-specific application named. Generic discipline reminders are noise; specific applications are usable.

**One thing the prompt deliberately doesn't do:** specify the audit kickoff's content. The new conversation's strategist drafts that against findings from the pre-flight Q's. If you specified the audit kickoff in this opening prompt, you'd be doing the strategist's work for it — and the strategist might disposition things differently based on what Moodle 5.2 release-notes evidence surfaces.

The prompt's goal is bootstrapping a conversation, not pre-deciding its outputs.

---

## Practical things to know when running it

**The methodology-archive links assume the new claude.ai conversation can fetch GitHub raw URLs.** This worked at the start of today's session via the methodology-capture conversation. If it doesn't work in your future session (e.g., Anthropic changes web_fetch behavior), the strategist can ask you to paste the relevant retrospective sections inline.

**The strategist's first response should be acknowledgment + pre-flight Q's, NOT the audit kickoff itself.** If the strategist immediately drafts the audit kickoff without acknowledging the methodology archive or surfacing Q's about Moodle 5.2 release status, that's a sign the strategist hasn't actually absorbed the archive — push back and ask explicitly.

**Stage 0 may surface as needed.** If the strategist's pre-flight Q's reveal you don't have a Moodle 5.2 environment yet, environment provisioning becomes Stage 0. The two-stage framing in the prompt accommodates this — it's "audit then remediate" but Stage 0 (environment) precedes both.

**Compatibility review may surface zero findings.** mod_scorecard at v0.6.0 has 168 PHPUnit tests passing on 5.1.3, phpcs zero/zero plugin-wide, methodology disciplines applied throughout. It's possible 5.2 just works. The prompt names this explicitly via "calibration honesty bidirectionally" so the strategist doesn't reflexively predict "must be findings."
