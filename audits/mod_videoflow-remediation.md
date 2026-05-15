# mod_videoflow — Remediation list

**Audit:** Initial audit, 2026-05-15.
**Plugin version audited:** v1.0.0 (`$version = 2026012106`).
**Audit deliverables (reference):**
- `docs/AUDIT.md` — file-by-file walkthrough
- `docs/SPEC.md` — descriptive specification
- `docs/DECISIONS.md` — architectural decisions
- `docs/SECURITY_REVIEW.md` — capability/input/output/file/CSRF/JS/dep review
- `MANUAL_SMOKE.md` — operator smoke walkthrough

**Severity scale.**

- **HIGH** — must fix before continuing development on this plugin. Either currently exploitable, currently leaks user data, or actively breaks shipped product behavior.
- **MEDIUM** — should fix soon. Either bypassable with operator awareness, or a stated product capability is missing/broken.
- **LOW** — defer if not blocking. Cosmetic, low-impact, defense-in-depth, or speculative.

**How to use this list.** Work top-down through HIGH. After HIGH is clear, address MEDIUM. LOW items can be grouped into a single cleanup sweep. Each item has a file:line reference, what the problem is, why it matters, and a proposed fix at the design level (not implementation).

---

## HIGH

### H1 — Server-side completion forgery via direct AJAX

**Source:** Phase 3 finding S1.

**Where:** `classes/external/update_progress.php:64–127`.

**What.** The `mod_videoflow_update_progress` external function trusts the client-reported `percentwatched` and `totalwatchtime`. The server clamps `percentwatched` to `[0, 100]` and treats `totalwatchtime` as monotonic non-decreasing, but does not cross-check either claim against actual playback. A client posting `{percentwatched: 100, totalwatchtime: <completion_threshold>}` directly to the endpoint (e.g., via `Ajax.call` from devtools, curl with a session cookie, or any web-service token holder) marks the activity complete without watching.

**Why it matters.** Q11 confirmed skip-prevention is an important product feature; D4 records that current enforcement is client-side only. For LMS Light's CE-tracking customers (CONTEXT.md ICP: niche training operators with certification CE hour requirements), completion forgery is a direct compliance liability. The smoke test step 37 demonstrates the bypass in one console command.

**Proposed fix (design level).**

1. Add a `last_update_timestamp` column to `videoflow_watch_sessions` (Moodle XMLDB upgrade step).
2. On each `update_progress` call, compute `elapsed = time() - session.last_update_timestamp`. Reject the update if `(reported_currentposition - session.currentposition) > elapsed * 1.5` (the 1.5× tolerance covers minor clock skew and tab-throttling without permitting outright fast-forward).
3. For local-source activities, additionally derive video duration from `file_storage` metadata at the time the activity is first viewed; store on `videoflow` and reject reports where `percentwatched > 100 * (currentposition / stored_duration) + tolerance`.
4. For Vimeo/YouTube sources, server-side duration is unavailable without an oEmbed API call to each provider — optional v1.2 step: cache provider-reported duration at instance-create time. v1.1 can ship the velocity check without per-provider duration and leave provider sources slightly less guarded than local.

**Estimated remediation scope:** 1 XMLDB upgrade, ~30 lines of new server-side validation, 4–6 PHPUnit cases. Recommend pairing with H3 (gradebook integration build).

---

### H2 — Privacy export will fatal — missing `use` import

**Source:** Phase 1 finding F1.

**Where:** `classes/privacy/provider.php:137, 139, 140`.

**What.** `provider::export_user_data()` calls `transform::yesno($session->completed)` and `transform::datetime($session->time*)`, but the file's `use` block at the top of `classes/privacy/provider.php` does not import `core_privacy\local\request\transform`. PHP resolves the unqualified class name `transform` against the file's own namespace (`mod_videoflow\privacy`), where no class of that name exists. The call site fatals with `Error: Class "mod_videoflow\privacy\transform" not found` the moment a privacy export touches a videoflow watch session.

Verified that `core_privacy\local\request\transform` exists in Moodle 5.1 (`mutms/moodle/public/privacy/classes/local/request/transform.php`); the class is correct, it is simply not imported.

**Why it matters.** GDPR data export is a regulated user-facing right. A fatal during export blocks compliance against an LMS Light tenant's GDPR responsibilities. Currently latent because exports against the test data are rare during dev, but every customer with any active videoflow user has the bomb on their disk.

**Proposed fix (design level).** Add `use core_privacy\local\request\transform;` to the file's `use` block. Manual smoke step 32 then re-exercises the export to confirm.

**Estimated remediation scope:** 1 line. Plus 1 PHPUnit case that exercises the export path end-to-end (the bug is purely about class resolution, so a unit-level export-roundtrip test would catch it).

---

### H3 — Gradebook integration wired but inert; no writer for `watch_sessions.grade`

**Source:** Phase 1 finding F4 + Phase 2 confirmation Q16 (BUILD intent).

**Where:** `lib.php:413–448` (`videoflow_update_grades` + `videoflow_get_user_grades`); `db/install.xml` (`videoflow_watch_sessions.grade` column); `mod_form.php:131` (grading section); `classes/external/update_progress.php:101` (session-row update path — does not write `grade`).

**What.** The plugin declares `FEATURE_GRADE_HAS_GRADE`, exposes a grade-max widget in the form, creates a gradebook item on activity save, declares `update_grades` integration with Moodle's gradebook, and reads from `videoflow_watch_sessions.grade` in `videoflow_get_user_grades()`. But no code path in the plugin writes a value to `videoflow_watch_sessions.grade`. Operators configuring grade-max see a gradebook entry but no scores flow in.

**Why it matters.** Q16 confirmed gradebook integration is intended product behavior, not vestigial scaffolding. An operator today who sets grade-max on a videoflow activity sees a gradebook entry that silently stays empty. The expected behavior (per the wired integration and the form widget) is that learner scores derived from completion / watch percentage flow into the gradebook.

**Proposed fix (design level).** Decide on the grade derivation formula. Likely candidates, ordered by simplicity:

- **(a) Pass/fail at completion.** When `videoflow_watch_sessions.completed` flips to 1, write `grade = videoflow.grade` (full grade) to the session row; when uncompleted, leave NULL. Simple, matches the existing completion semantics.
- **(b) Linear by percent watched.** `grade = (percentwatched / 100) * videoflow.grade`. Gives partial credit; works without completion thresholds being configured.
- **(c) Operator-selectable formula.** Add a form field "Grade is derived from: [completion / percent watched]"; store the choice on `videoflow`; branch in the writer.

Recommend (a) for v1.1 (simplest, matches current binary completion behavior) and document (b)/(c) as v1.2 enhancements.

Implementation plan:

1. Add a writer call to `update_progress::execute()` immediately after the `completed` flag transitions to 1: load the activity's `grade` field, write to session row, call `videoflow_update_grades($videoflow, $USER->id)`.
2. Backfill: optional CLI script for sites with existing watch sessions to retroactively populate the `grade` column.
3. Tests: PHPUnit case for "completion fires → grade populated → gradebook reflects grade."

**Estimated remediation scope:** ~20 lines of writer code in `update_progress.php`, optional ~30 lines of CLI backfill, 2–3 PHPUnit cases. Pairs naturally with H1 (both touch `update_progress.php`).

---

## MEDIUM

### M1 — Backup omits `headertext` and `footertext`

**Source:** Phase 1 finding F3 + Phase 2 confirmation Q15 (backup capability wanted).

**Where:** `backup/moodle2/backup_videoflow_stepslib.php:41–46`.

**What.** The `backup_nested_element` for `videoflow` enumerates 15 fields but omits `headertext` and `footertext`. Both are populated form fields with persistent operator-edited values. On backup → restore round-trip, both values are silently lost.

**Why it matters.** Operator-edited content silently vanishes through backup/restore. Q15 confirmed backup capability is wanted (the omission was scope cut, not deliberate). Customers cloning courses for next-term reuse lose their custom header/footer text without notice.

**Proposed fix (design level).** Add `'headertext', 'footertext'` to the field list in `backup_videoflow_stepslib.php`. Restore stepslib already handles arbitrary fields via `process_videoflow` → `insert_record`, so no restore-side change required. Add a backup-restore round-trip PHPUnit case that asserts both fields survive.

**Estimated remediation scope:** 2 names to add, 1 round-trip test.

---

### M2 — User-progress backup is missing; needed per Q15

**Source:** Phase 2 Q15 (backup of user progress is wanted, was scope cut).

**Where:** `backup/moodle2/backup_videoflow_stepslib.php`; restore counterpart.

**What.** The current backup stepslib backs up only the `videoflow` table. `videoflow_watch_sessions` and `videoflow_tracking_events` are not part of the backup. Moodle's convention for plugin user-data is to expose a userdata-included flag (`$userinfo` from `backup::VAR_SETTING_USERINFO`) and conditionally include user-data tables when the operator opts in. This plugin does not implement that branch.

**Why it matters.** Q15: operator wants backup-capability for user progress. Without it, full-fidelity course backups (typical use case: tenant migration, full-archive backups) lose learner state.

**Proposed fix (design level).**

1. In `backup_videoflow_stepslib::define_structure()`, check `$this->get_setting_value(backup::VAR_SETTING_USERINFO)`.
2. When userinfo is enabled, add `backup_nested_element` definitions for `videoflow_watch_sessions` (under `videoflow`) and `videoflow_tracking_events` (under each session), wire `set_source_table()` for each, and annotate any related files.
3. In `restore_videoflow_stepslib`, parallel restore paths: `process_videoflow_session()` and `process_videoflow_event()` that re-insert rows with mapped foreign keys (`videoflowid → new id`, `userid → mapped user id via `restore_dbops::set_backup_ids_record`).
4. Tests: PHPUnit cases for both userinfo-on and userinfo-off backup→restore.

**Estimated remediation scope:** ~80 lines added to backup/restore stepslib, 4–6 PHPUnit cases. Touches user-id mapping (which is non-trivial); plan for ~1 phase of focused work.

---

### M3 — No PHPUnit suite

**Source:** Phase 1 observation + Phase 2 confirmation Q19 (slipped, not deliberate).

**Where:** absence of `tests/` directory.

**What.** The plugin ships with zero automated test coverage. LMS Light's quality gate (CONTEXT.md § "Code quality gates"): "PHPUnit full suite green — all plugin tests pass, no regressions from the prior phase" is currently unsatisfiable for this plugin.

**Why it matters.** Q19 confirmed the omission was a slip, not a scoped decision. For a plugin entering active development (H1 + H3 will land non-trivial server-side logic), shipping changes without tests is high-risk. Every H1/H3 design change needs a test scaffold to land against.

**Proposed fix (design level).**

1. Create `tests/generator/lib.php` with `mod_videoflow_generator extends testing_module_generator` providing `create_instance($record, $options)` so other tests can fixture videoflow activities.
2. Add test classes (one per logical unit, modeled on SAD-built plugin conventions):
   - `tests/lib_test.php` — `videoflow_supports`, add/update/delete instance, grade item, get_or_create_session, reset_userdata.
   - `tests/external/update_progress_test.php` — the AJAX endpoint, including the new H1 velocity check, the existing percent/time clamps, and the H3 grade writer.
   - `tests/completion/custom_completion_test.php` — the class-based completion API.
   - `tests/privacy/provider_test.php` — full export/delete round-trip (this would have caught H2/F1).
   - `tests/backup/backup_test.php` — backup→restore round-trip including the M1/M2 fields.
3. Match LMS Light conventions (CONTEXT.md): `@covers` declarations (with the migration caveat from LESSONS.md about PHPUnit 11 → 12); `@final` classes; consistent naming.

**Estimated remediation scope:** ~600–800 lines of test code for a v1.0-equivalent suite. Recommend landing test scaffolding in a dedicated PR before H1/H3 work so subsequent feature PRs land against tests that catch regressions.

---

## LOW

### L1 — Privacy metadata strings missing from `lang/en/videoflow.php`

**Source:** Phase 1 finding F2.

**Where:** `classes/privacy/provider.php:50–78` references 15 `privacy:metadata:*` keys; `lang/en/videoflow.php` defines none of them.

**What.** The privacy metadata admin pages (Site admin → Users → Privacy and policies → Data registry → Activity modules → Video Flow) render `[[privacy:metadata:videoflow_watch_sessions:videoflowid]]` etc. as literal text instead of the intended human-readable descriptions.

**Why it matters.** Cosmetic. Affects only the admin-facing privacy disclosure page, not data flows. Resolves once strings are added.

**Proposed fix (design level).** Add 15 string definitions to `lang/en/videoflow.php`. The set is enumerated by the `add_database_table` calls in `provider.php:50–78`.

**Estimated remediation scope:** ~15 lines of language strings.

---

### L2 — Dead `videoflow_get_completion_state()` in `lib.php`

**Source:** Phase 1 finding F5.

**Where:** `lib.php:459–471`.

**What.** Legacy pre-3.11 completion callback. Moodle 5.x only invokes the class-based completion API (verified at `mutms/moodle/public/lib/completionlib.php:707`). The function is never called.

**Why it matters.** Code clutter; misleading to a reader unfamiliar with the Moodle 4.x → 5.x completion API migration.

**Proposed fix (design level).** Remove the function. No callers; no replacement needed.

**Estimated remediation scope:** ~13 lines removed.

---

### L3 — `amd/build/player.min.js` not actually minified

**Source:** Phase 1 finding F6.

**Where:** `amd/build/player.min.js`, byte-identical to `amd/src/player.js` (23326 bytes each).

**What.** The file at the `.min.js` path is the unminified source verbatim. Moodle's `requirejs.php` serves whichever exists; the production-cached path uses `amd/build/`.

**Why it matters.** Maintainability trap: a developer who edits `amd/src/player.js` and runs Moodle's normal cache-purge cycle may not see their changes reflected (because requirejs serves the stale `amd/build/player.min.js`). Currently both files are identical so the symptom is hidden, but the trap is loaded.

**Proposed fix (design level).** Either:

- **(a)** Add Grunt-based AMD minification to the plugin's `Gruntfile.js` (none currently exists) — matches Moodle convention and produces a real `*.min.js`.
- **(b)** Acknowledge the choice not to minify (matches the LMS Light pattern noted in `mod_knowledgecheck` repo's docs/DECISIONS.md and the auto-memory entry on format_pathway-style AMD-not-ESM): keep the build folder as a copy of src, document the convention, and add a CI check that `build === src` (so the drift is enforced if it occurs).

Recommend (b) for consistency with the other LMS Light plugins, with a note added to a project-level conventions doc that says "build/ is a copy of src/ until we add minification."

**Estimated remediation scope:** if (a): build config + grunt task. If (b): zero code changes, one note added to LESSONS.md or a future ONBOARDING.md.

---

### L4 — Placeholder `@copyright Your Name <your@email.com>` across every PHP file

**Source:** Phase 1 finding F7 + Phase 2 Q21 (remove).

**Where:** every `.php` file's docblock; same for `version.php`'s `$plugin` block comment.

**What.** Untouched scaffolding text. Q21 confirmed: remove.

**Proposed fix (design level).** Bulk-replace `@copyright  2026 Your Name <your@email.com>` with the intended LMS Light attribution. Pending decision on canonical attribution string for LMS Light plugins (consistent with other LMS Light plugins — TBD if there's a standard yet).

**Estimated remediation scope:** ~20 docblock edits, sed-replaceable.

---

### L5 — Guest archetype on `mod/videoflow:view`

**Source:** Phase 3 finding S2 + Phase 2 Q18 ("not used, not sure why it's there").

**Where:** `db/access.php:43`.

**What.** `'guest' => CAP_ALLOW` was set on `mod/videoflow:view` without deliberate intent; no LMS Light tenant uses guest course access. The capability grant allows guest users to hit `view.php`, which currently has no `isguestuser()` bail (`videoflow_cm_info_view` does have one). Result: a guest visit triggers session-row creation for the guest pseudo-user — benign noise but pollutes analytics.

**Why it matters.** Defense in depth. Removing what isn't used reduces audit surface and analytics noise.

**Proposed fix (design level).**

1. Remove `'guest' => CAP_ALLOW` from `db/access.php`.
2. Optionally add `isguestuser()` bail to `view.php` (parallels `videoflow_cm_info_view`).
3. Add `db/upgrade.php` step to remove the capability from existing role assignments (if it has ever been granted to any custom role in production).

**Estimated remediation scope:** 1–2 lines removed, optional 3-line bail in view.php, optional upgrade step.

---

### L6 — External SDKs loaded from CDN without version pin or SRI

**Source:** Phase 3 finding S3 + Phase 2 Q13 (no firewalled-CDN customers yet).

**Where:** `amd/src/player.js:161` (Vimeo), `amd/src/player.js:240–244` (YouTube).

**What.** Vimeo Player SDK and YouTube IFrame API are loaded at runtime without version pinning or Subresource Integrity (SRI) hashes. A CDN compromise of either vendor would propagate.

**Why it matters.** Supply-chain risk. Q13: no current operational risk (no firewalled tenants); residual risk is the compromise scenario alone.

**Proposed fix (design level).**

- Vimeo: switch to a versioned URL (e.g., `https://player.vimeo.com/api/player.2.x.x.js` or unpkg.com-mirrored URL with SRI hash).
- YouTube: SRI not officially supported for `iframe_api`; document residual risk in a project-level security memo.
- Optionally: implement a custom CSP that restricts script-src to the two vendor domains.

**Estimated remediation scope:** if vendor cooperation allows: 1 URL change + 1 integrity attribute. If not: ~1 paragraph of documentation.

---

### L7 — 24-hour browser cache on local-video pluginfile responses

**Source:** Phase 3 finding S4.

**Where:** `lib.php:509` — `send_stored_file($file, 86400, ...)`.

**What.** Local-source videos are served with a 24-hour browser cache lifetime. Cached videos remain accessible to a logged-out user via back/forward navigation until the cache expires.

**Why it matters.** Low operational risk for typical training-video content. Matters for tenants distributing confidential or proprietary content on shared-device deployments.

**Proposed fix (design level).** Either:

- **(a)** Reduce default cache lifetime: `send_stored_file($file, 60, ...)` for 1-minute cache.
- **(b)** Expose a per-activity "Cache lifetime" setting; operators choose.

Recommend (a) for v1.1; (b) as v1.2 if a customer requests granular control.

**Estimated remediation scope:** 1 line change for (a). For (b): form field + storage + 1 lookup in lib.php.

---

### L8 — Sessionid enumeration via differential exception messages

**Source:** Phase 3 finding S5.

**Where:** `classes/external/update_progress.php:74` (MUST_EXIST throws on missing) vs `:84` (throws different exception when session belongs to another user).

**What.** Differential error messages let an authenticated attacker enumerate session ids and infer user-activity relationships. Sessionids are sequential auto-increment integers; enumeration is trivial.

**Why it matters.** Low impact info-leak. The leaked info is "user X has watched activity Y" — useful for reconnaissance, not directly exploitable.

**Proposed fix (design level).** Replace the two-step lookup with a single `WHERE id = :sessionid AND userid = :userid` query; throw a uniform error on miss regardless of root cause.

**Estimated remediation scope:** ~10 lines refactor.

---

### L9 — README describes features that don't exist

**Source:** Phase 1 § "Documentation drift" + Phase 2 Q1–Q9 (all aspirational).

**Where:** `README.md` (every section describing chapters / info tab / chat tab / transcript tab / preview mode / reports page / `:submit` / `:viewreports` capabilities).

**What.** Q1–Q9 confirmed that every feature claimed in the README beyond what's in the code is aspirational — never built. The README is currently describing a roadmap, not the shipped product.

**Why it matters.** Misleading to operators (who set up the plugin expecting features that don't work) and to future maintainers (who try to understand the code through the README's lens). README rewrite is queued as a separate workstream from the audit per Phase 1 confirmation, but recording it here for cumulative tracking.

**Proposed fix (design level).**

1. Rewrite `README.md` to describe only the shipped behavior (use `docs/SPEC.md` as the source of truth).
2. Create a separate `ROADMAP.md` (or use `docs/SPEC.md` § "Future enhancements") for the aspirational features, clearly tagged as "not yet implemented."
3. Remove the "Inspired by bdecent.de" attribution per Q20.
4. Replace `[Your Name] <your@email.com>` placeholder credits per Q21.

**Estimated remediation scope:** README rewrite (~200 lines new; existing README ~230 lines to revise/remove). Recommend pairing with L4 (copyright cleanup) so all attribution changes land together.

---

## Summary

**Total findings:** 15 (3 HIGH, 3 MEDIUM, 9 LOW).

**Recommended sequencing.**

1. **Hotfix batch (HIGH):** H2 (privacy export fatal) lands immediately as a one-line patch; H1 + H3 follow as a paired piece of work since both touch `update_progress.php`. M3 (test scaffolding) lands before H1/H3 implementation so the new server-side logic ships with tests.
2. **Backup capability work (MEDIUM):** M1 + M2 together — both touch backup stepslib. M1 is trivial; M2 is the substantive backup-userdata enablement.
3. **Cleanup sweep (LOW):** L1, L2, L4, L5, L6, L7, L8, L9 grouped into one or two cleanup PRs. L3 only blocking if L9 (README rewrite) doesn't already address the "what's the build/min convention" question.

**Cross-references.**

- All HIGH and MEDIUM items have file:line references in this document. AUDIT.md provides additional context per file.
- DECISIONS.md provides the design rationale (or in some cases, the absence of one) for the choices that produced the findings.
- SPEC.md describes the shipped behavior the remediations are working from.
- SECURITY_REVIEW.md provides the security-area framing for S1–S5.
- MANUAL_SMOKE.md provides the operator-runnable verification for each finding (smoke steps 32, 37 directly probe H1 and H2; steps 29, 35 confirm M1 and L5 respectively).

**Severities preliminary on first pass; revise during remediation kickoff if customer-context changes (especially around H3's customer impact today).**
