# Moodle activity-mod phase progression

Structural reference for supervised-agentic activity-mod plugin
development under LMS Light's working conventions. Distilled from
mod_scorecard's six-phase progression (v0.1.0 → v0.6.0, shipped
2026-04-26 through 2026-04-28). Companion to `CONTEXT.md`
(project conventions) and `LESSONS.md` (portable methodology). The
methodology synthesis document `METHODOLOGY.md` references this
document as a known concept rather than inlining its content.

---

## 1. Purpose and audience

This document is the structural reference for activity-mod phase
progression. It exists to spare future activity-mod plugin work from
re-deriving "what does Phase N cover" from mod_scorecard's commit
archaeology each time.

**Operator** drafting kickoff prompts reaches for "we're doing
Phase 3" and uses this doc to know roughly what that means — what
code surface gets touched, what dependencies must already be in
place, how many sub-steps the work typically decomposes into, what
calibration tax to anticipate.

**Implementer** in Claude Code reads the relevant phase section
during pre-flight verification to ground "what does this phase
cover" in archive rather than re-derivation. The same reflex
applies that grounds kickoff Q dispositions in pre-flight evidence
rather than memory.

This is not a tutorial, not a how-to, not prescriptive. The phase
progression is a template, not a mandate. mod_scorecard's actual
progression (Phase 1 skeleton → 2 authoring → 3 learner submission
→ 4 reporting → 5a gradebook+completion → 5b privacy+backup/restore)
is one valid path through the activity-mod problem space; another
activity-mod might legitimately reorder phases (e.g., gradebook
before reporting if reporting depends on grade integration), merge
adjacent phases (5a + 5b into a single Phase 5 if scope is light),
or split heavy phases (Phase 4 into 4a reporting + 4b export).

The methodology disciplines transfer; the phase shape adapts.

---

## 2. Phase progression overview

| Phase | Scope summary | Sub-steps | mod_scorecard actual | mod_scorecard exemplar |
|-------|---------------|-----------|---------------------|------------------------|
| 1 — Skeleton | Install schema, capabilities, mod_form, view, privacy provider scaffold, settings-only backup/restore, skeleton tests | 6–10 first time | 6 sub-steps + 1 fix-forward = 7 round-trips | v0.1.0 |
| 2 — Authoring | Manage screen with CRUD tabs, soft-delete, lifecycle gates | 4–7 first time | 6 sub-steps = 6 round-trips | v0.2.0 |
| 3 — Learner submission | Submission form, validation, attempt + response save, scoring/processing engine, snapshotting, result page | 5–7 first time | 6 sub-steps + 2 fix-forwards = 8 round-trips | v0.3.0 |
| 4 — Reporting | Reports surface (capability-gated table, expandable detail, group filter, CSV export, pagination) | 6–8 first time | 7 sub-steps + 1 fix-forward = 8 round-trips | v0.4.0 |
| 5a — Gradebook + completion | Grade API integration with grade method (latest/highest/etc), activity completion via custom rule | 5–8 first time | 7 sub-steps + 1 fix-forward = 8 round-trips | v0.5.0 |
| 5b — Privacy + backup/restore | Full Privacy API provider implementation, nested backup steps, restore steps with id mapping | 5–7 first time | 6 sub-steps = 6 round-trips | v0.6.0 |

**Phases 1–3 are the activity functional core.** You can't ship
learner submissions without Phase 3. Phases 1 and 2 are
prerequisites; Phase 3 is where the activity becomes useful for
learners.

**Phase 4 is reporting.** Valuable but not strictly required for
v0.x.0 ship if a workflow doesn't need cross-attempt visibility.
Some activity-mods may legitimately defer Phase 4 to v1.x or skip
it entirely (e.g., if the activity has no per-attempt history
worth reporting on).

**Phase 5a (gradebook + completion)** makes the activity a
first-class Moodle citizen integrated with course-level workflows.
Course-level completion criteria, gradebook propagation, course
total computation — these all flow from Phase 5a being shipped.

**Phase 5b (privacy + backup/restore)** makes the activity a
responsible Moodle citizen — operators can fulfil GDPR / data
subject requests; courses can be backed up and restored cleanly,
preserving learner submissions across course duplications.

**Phases beyond 5b are plugin-specific feature work.** No general
template applies. Once an activity-mod has shipped Phases 1–5b, it
is functionally complete by Moodle convention; subsequent work is
either feature expansion or refinement specific to the activity's
domain.

The numbering convention (5a / 5b instead of 5 / 6) is deliberate.
Gradebook and privacy are independent surfaces that often ship as
separate releases despite being conceptually "Phase 5 maturity
work." Splitting the maturity phase recognises that an
activity-mod might ship gradebook integration in one release cycle
and privacy + backup/restore in another, and the calibration tax
for each surface is independent. Future activity-mods might
collapse to a single Phase 5 if scope warrants, or split further
if the work is heavy.

mod_scorecard's actual progression took 43 round-trips total
across the six phases (38 forward sub-steps + 5 fix-forwards). At
the upper-middle of what you'd predict for a first activity-mod
under this methodology — calibration-tax-honest in the
upper-bound-allowance sense.

---

## 3. Phase-by-phase detail

### Phase 1 — Skeleton

The activity-mod foundation. After Phase 1 ships, the plugin is
installable, has its capability set declared, has a settings-only
form, renders an empty view page, and has scaffold-level
backup/restore + privacy provider stubs. No learner-facing
behaviour beyond "the activity exists in the course."

**Scope.**

- `db/install.xml` — schema for the activity's main table plus any
  child tables needed in early phases (items, bands, attempts,
  responses, etc.). Hand-written XMLDB; Moodle's reserved-word
  list applies.
- `db/access.php` — capability declarations. At minimum: addinstance,
  view, manage, submit, viewreports, export. Archetypes assigned per
  Moodle convention.
- `db/upgrade.php` — empty stub (`xmldb_<plugin>_upgrade`); first
  savepoint adds in Phase 5a or wherever the first schema migration
  lands.
- `mod_form.php` — settings-only form using `moodleform_mod`.
  Standard sections: header, intro, plugin-specific settings, common
  module settings, course module settings.
- `view.php` — minimal view page with sesskey validation, capability
  check, course module resolution, basic header render.
- `lib.php` — instance lifecycle (`<plugin>_add_instance`,
  `<plugin>_update_instance`, `<plugin>_delete_instance`),
  `<plugin>_supports` declarations.
- `backup/moodle2/backup_<plugin>_stepslib.php` + `restore_<plugin>_stepslib.php`
  + activity_task.class.php files — settings-only structure;
  attempts and responses come in Phase 5b.
- `classes/privacy/provider.php` — metadata declaration only;
  export and delete contracts come in Phase 5b.
- `version.php`, `lang/en/<plugin>.php`, basic PHPUnit fixtures.

**Dependencies.** None. This is the foundational phase.

**Typical sub-step decomposition.**

1. Install schema + capabilities + version + base lang strings.
2. Activity contract (mod_form, view, icon, expanded lang).
3. Privacy provider scaffold (metadata declared; subjects stubbed).
4. Backup/restore scaffold (settings round-trip; nested data
   deferred).
5. PHPUnit skeleton tests (lib CRUD + schema integrity).
6. Docs (README, CHANGES; SPEC verified canonical).

mod_scorecard followed this exact decomposition (6 sub-steps:
1.1–1.6). One fix-forward followed (`946d09b`, SPEC §9.1 +
access.php capability matrix correction) when a manage capability
declaration was found inconsistent with the SPEC.

**Calibration anchor.**

- First activity-mod (no prior activity-mod pattern bank): 6–10
  sub-steps; **15–20 round-trips** including pattern absorption
  for the activity-mod skeleton conventions. mod_scorecard's
  actual: 6 sub-steps + 1 fix-forward.
- N+1 activity-mod (mature pattern bank from prior activity-mod):
  6 sub-steps; **6–8 round-trips**. The skeleton conventions are
  among the most portable; pattern bank inheritance is rich.

**Common Type B friction surfaces.**

- phpcs comment-style nits — Moodle's standard differs from PSR
  conventions in several specific places (block comments, function
  docblock voice).
- MOODLE_INTERNAL guard placement — required for files setting
  global state at top level (top-level `require_once`, function
  definitions outside classes); forbidden for namespace + use +
  class-only files.
- version.php numeric ordinal convention (YYYYMMDDXX with
  same-day-increment / new-day-reset).
- install.xml schema validation against Moodle's reserved-word list
  (column names like `order`, `key`, `from` are reserved).
- privacy provider metadata declaration shape — subtle field-level
  invariants (e.g., the `itemid` graph-traversal link gap that
  surfaces only at Phase 5b export-contract time if missed at
  Phase 1 metadata declaration time).

**Gate-discipline shapes that apply.**

- PHPUnit on the skeleton tests (instance CRUD lifecycle, schema
  integrity).
- phpcs zero/zero plugin-wide.
- Walkthrough: "create an instance, see it in the course, edit it,
  delete it" — the bare-minimum activity-mod admin flow.
- Empirical install application: install the plugin in a fresh
  Moodle DDEV; confirm the capability set propagates and the
  activity is creatable.

**Reflexes typically banked at Phase 1.**

- phpcs Moodle conventions (commit-by-commit phpcs runs catch nits
  early).
- MOODLE_INTERNAL contextual rule.
- version stamp ordinal convention.
- db/install.xml field type conventions (int vs char vs text;
  LENGTH; SEQUENCE; KEYS).
- Capability archetype propagation discipline (archetype edits in
  access.php need db/upgrade.php savepoints; new caps don't
  auto-propagate to existing roles).

**mod_scorecard archaeology.** Commits `f3fd62d` through `662763c`
for the six Phase 1 sub-steps; `946d09b` for the SPEC §9.1
capability matrix fix-forward. No retrospective specifically for
Phase 1, but Phase 4 retrospective Section 1 (trajectory data) and
Section 4 (kickoff-evidence-grounding reflex) are most directly
relevant to early-phase calibration.

---

### Phase 2 — Authoring

The operator-facing surface for content authoring. After Phase 2,
managers can define the structured content the activity wraps —
items, bands, prompts, options, whatever the domain requires.
Learners can't yet submit (that's Phase 3) but the activity has
real authored content.

**Scope.**

- `manage.php` — entry point, tab routing, capability gate.
- `classes/output/manage_renderer.php` (or similar) — the manage
  screen's render orchestration.
- `classes/form/<entity>_form.php` for each authored entity (e.g.,
  `item_form.php`, `band_form.php`).
- `locallib.php` extensions — CRUD helpers per entity, soft-delete
  semantics, reorder semantics, lifecycle gates.
- `db/access.php` capabilities for authoring caps if not already
  declared at Phase 1.
- Lang strings for tabs, form fields, error messages, soft-delete
  markers.
- `styles.css` — visual treatment for the manage screen.

**Dependencies.** Phase 1 schema + capabilities + base lang.

**Typical sub-step decomposition.**

1. Manage scaffold (tabtree, tab routing, capability gate).
2. First entity authoring (locallib, form, renderer, lifecycle
   gate).
3. Second entity authoring if applicable (form, locallib, list
   rendering, shared deleted-marker conventions).
4. Cross-entity validation if applicable (e.g., overlap detection,
   coverage validation).
5. Lifecycle enforcement (locks that activate after first attempt).
6. Docs (README authoring section, CHANGES, version bump, SPEC sha
   verified).

mod_scorecard: 6 sub-steps (2.1–2.6), no fix-forward. This was the
shortest phase by round-trip count under the methodology.

**Calibration anchor.**

- First activity-mod: 4–7 sub-steps; **7–12 round-trips**.
- N+1 activity-mod: 4–6 sub-steps; **5–8 round-trips**. Manage UI
  patterns absorbed; form scaffolding patterns absorbed; soft-
  delete semantics absorbed.

mod_scorecard's actual at 6 sub-steps / 6 round-trips landed at the
lower bound of the first-activity-mod range — the manage scaffold
absorbed cleanly because Moodle's `tabtree` + `flexible_table` +
`moodleform` patterns are well-trodden. Subsequent activity-mods
should expect similar.

**Common Type B friction surfaces.**

- Tab navigation conventions — Moodle's `tabtree` API has subtle
  conventions around currenttab / inactive / disabled state that
  surface only when multiple tabs are added.
- Soft-delete vs hard-delete semantics for first-encounter — once
  attempts exist, hard-delete on items must transition to
  soft-delete to preserve historical attempt detail. The transition
  point is Phase 3 (after the first submission); Phase 2 must
  support both modes.
- Lifecycle gate convention (e.g., `<plugin>_can_manage_items`
  shaped checks) — when does the soft-delete gate activate, when
  does the scale-lock gate activate, when does the new-item-warning
  surface.

**Gate-discipline shapes that apply.**

- PHPUnit on each form's validation rules and each helper's CRUD
  semantics.
- phpcs zero/zero plugin-wide.
- Walkthrough: "as a manager, author the activity end-to-end —
  create entities, edit them, soft-delete one, reorder them, save
  with validation errors and confirm error messaging."
- Lifecycle-state walkthrough: simulate "before any attempts" vs
  "after an attempt is created" to confirm gates activate
  correctly. (At Phase 2 there are no real attempts yet, so this
  is typically deferred to Phase 3 gate.)

**Reflexes typically banked at Phase 2.**

- Soft-delete semantics and the lifecycle-gate convention.
- Tab navigation pattern (currenttab handling, inactive vs
  disabled).
- Form-level validation + inline error messaging conventions.
- Capability-graduation pattern (different views/forms for
  different caps within the same surface).

**mod_scorecard archaeology.** Commits `885d748` through `9898faf`
for the six Phase 2 sub-steps. No retrospective specifically; this
phase's reflexes informed Phase 4 (which has the formal retrospective
at `080fe57`).

---

### Phase 3 — Learner submission

The phase where the activity becomes useful. After Phase 3, learners
can submit attempts, the scoring/processing engine produces results,
results render via the result page. The activity has a complete
end-to-end learner workflow.

This is typically the calibration-tax-richest phase. The
processing engine is domain-specific (each activity-mod has its
own scoring or processing logic); transactional save lifecycle has
subtle invariants (atomic write of attempt + responses + snapshot);
snapshot semantics need careful design to satisfy historical-
attempt-stability requirements.

**Scope.**

- `submit.php` (or similar entry point) — the form handler.
- `classes/form/submit_form.php` — the learner submission form.
- `classes/local/scoring_engine.php` (or domain-specific
  processing) — the pure-logic compute function. Typically returns
  a structured result without side effects; the caller persists.
- Attempt + response save lifecycle — single transaction with
  attempt insert + response inserts + snapshot capture.
- Snapshotting at submit time — copy band labels, message text,
  format, score values onto the attempt row so the result is
  stable against future authoring changes.
- `result.php` (or rendered as part of `view.php`) — snapshot-only
  render; no recomputation from current entity state.
- Retake handling — previous-attempt callout, allow-retake
  gating.
- Lang strings for form fields, validation messages, submit button,
  result heading, etc.

**Dependencies.** Phase 1 schema (attempts + responses tables).
Phase 2 authored content (items, bands).

**Typical sub-step decomposition.**

1. Learner form scaffold (view branching, render_learner_form,
   submit handler stub).
2. Scoring/processing engine (pure-logic compute function with
   PHPUnit cases; audit-only response handling for invalid
   inputs).
3. Submit endpoint (handler extraction, single-transaction persist,
   event fire, audit-write semantics).
4. Result page (snapshot-only render, conditional percentage / band
   / item-summary rendering, audit-honest item summary for
   soft-deleted entities).
5. Retake callout (previous attempt summary above form when
   retakes enabled).
6. Docs (README learner experience section, CHANGES, version bump,
   SPEC sha verified).

mod_scorecard: 6 sub-steps (3.1–3.6) plus 2 fix-forwards (`f3e2928`
empty-state link for managers; `3d6e5f9` persistent manage
affordance on learner-facing view). 8 round-trips total.

**Calibration anchor.**

- First activity-mod: 5–7 sub-steps; **8–15 round-trips**. The
  processing engine and snapshot semantics absorb the calibration
  tax.
- N+1 activity-mod: 6–8 sub-steps; **6–10 round-trips**. Form
  scaffolding patterns and submit-handler patterns absorbed; the
  processing engine remains domain-specific so its complexity is
  not amortised.

**Common Type B friction surfaces.**

- Validation messaging conventions — Moodle's form validation
  produces error arrays that flow through specific lifecycle hooks;
  inline error messaging on individual form elements requires
  specific render-layer plumbing.
- Snapshot field semantics — what gets snapshotted vs computed at
  render time. Phase 3 mistakes here surface as "result page
  changes for historical attempts when band is edited later" or
  similar. SPEC §11.2 (or its plugin equivalent) is load-bearing.
- Transactional save lifecycle — atomic write of attempt + responses
  + event triggering + audit write. Concurrent submission edge
  cases.
- Capability gating for submit vs view-own-submission — different
  caps protect different surfaces; ensure the capability set
  enforces the intended affordances.
- Empty-state UX — what learners see before any items have been
  authored, what managers see when they happen to land on the
  learner view.

**Gate-discipline shapes that apply.**

- PHPUnit on the processing engine (pure-logic; trivially testable).
- PHPUnit on the submit endpoint (state-modifying with transactional
  rollback).
- phpcs zero/zero plugin-wide.
- Walkthrough: "as a learner, submit an attempt, see the result;
  retake (if enabled) and confirm previous-attempt callout; submit
  with validation errors and confirm inline error messaging."
- Manager walkthrough: "verify manager-facing affordances on the
  learner view (e.g., 'Add items' link in empty state)."

**Reflexes typically banked at Phase 3.**

- Snapshot semantics (snapshot what's user-facing; compute what's
  internal).
- Transactional save patterns (single atomic write of attempt +
  responses; event fires after commit; audit writes inside the
  transaction).
- getDataGenerator quirks for activity-specific records (the
  framework's data generator may not cover all activity-mod
  records; per-plugin generator class extensions are common).
- Empty-state UX for both learner and manager perspectives.

**mod_scorecard archaeology.** Commits `56b3787` through `3a34d5e`
for the six Phase 3 sub-steps; `f3e2928` and `3d6e5f9` for the two
fix-forwards. No retrospective specifically; Phase 4 retrospective
references Phase 3's helper-establishment patterns.

---

### Phase 4 — Reporting

Manager-facing surface for reviewing learner submissions across
attempts. After Phase 4, managers can see who submitted what when,
filter by group, drill into per-attempt detail, export to CSV. The
plugin gains operational value beyond per-learner result pages.

**Scope.**

- `report.php` — entry point, capability gate, table render.
- `classes/output/report_renderer.php` — report-specific render
  orchestration.
- `classes/output/report_table.php` — `flexible_table` subclass for
  the attempts table with pagination + sorting.
- `export.php` — CSV export endpoint, capability-gated separately
  from view (per SPEC convention).
- `locallib.php` helpers — data shape helpers (`<plugin>_get_attempts`,
  `<plugin>_get_attempt_responses`, etc.) returning structured data
  rather than rendered HTML.
- Group filter integration — `groups_get_activity_group` +
  `groups_get_activity_allowed_groups` for the standard Moodle
  group selector.
- Lang strings for table column headers, filter labels, empty
  states.

**Dependencies.** Phase 3 attempts + responses schema. Phase 2
authored content (so the report has something to report on).

**Typical sub-step decomposition.**

1. Report data layer + page scaffold (capability-gated table
   render, empty-state branch, manage tab redirect).
2. Expandable per-attempt detail block.
3. Group filter integration.
4. CSV export.
5. Pagination via `flexible_table` subclass.
6. Polish — scope-prefix consistency on report-table CSS;
   capability-graduation refinements.
7. Docs (README reports section, CHANGES, version bump, SPEC sha
   verified).

mod_scorecard: 7 sub-steps (4.1–4.7) plus 1 fix-forward (`5701621`,
the missing `limitedwidth` body class on top-level pages — caught
during release-readiness side-by-side comparison against
mod_quiz). 8 round-trips total. The fix-forward established the
parallel-surface-comparison gate discipline (banked as
`feedback_parallel_surface_comparison.md`).

**Calibration anchor.**

- First activity-mod: 6–8 sub-steps; **7–10 round-trips**.
- N+1 activity-mod: 5–7 sub-steps; **6–7 round-trips**. The
  flexible_table conventions, capability-graduation patterns, and
  helper-decomposition disciplines absorb cleanly.

**Common Type B friction surfaces.**

- `flexible_table` dynamic-property reflex — Moodle's base class
  uses runtime properties (`$this->use_pages = true`,
  `$this->is_collapsible = false`, etc.) that PHP 8.2's strict
  property semantics flag as deprecation warnings unless the
  subclass declares them explicitly. PHPUnit surfaces this; phpcs
  doesn't.
- Capability-graduation in operator UI — different views for
  different caps within the same surface. Manager sees full table;
  manager-with-export-cap also sees export button; admin sees
  additional admin-only controls. Ensure each capability boundary
  is enforced at both UI render and endpoint dispatch.
- Helper-decomposition for testability — the data-layer helpers
  (`<plugin>_get_attempts` etc.) should return data structures
  rather than streaming output, so PHPUnit can target the data
  shape directly.
- limitedwidth body class on top-level pages — easy to forget at
  Phase 1 when only `view.php` exists; surfaces visibly only when
  the manage / report / submit pages render alongside core
  activity views with the expected width constraint.

**Gate-discipline shapes that apply.**

- PHPUnit on each helper's data shape and each table's render
  output.
- phpcs zero/zero plugin-wide.
- Walkthrough: "as a manager, view the report, filter by group,
  expand a row, export CSV; switch to a manager-without-export-
  cap and confirm the export button is hidden."
- Parallel-surface comparison at release-readiness — side-by-side
  visual against mod_quiz / mod_assign / mod_choice's report
  surface to catch missing chrome-level conventions
  (`limitedwidth` body class; activity-header rendering;
  breadcrumb consistency).

**Reflexes typically banked at Phase 4.**

- `flexible_table` dynamic-property declaration discipline.
- Capability-graduation pattern.
- Helper-decomposition-for-testability pattern.
- Parallel-surface comparison reflex.
- Write-time alphabetical / voice consistency on lang strings.

**mod_scorecard archaeology.** Commits `9e9e116` through `96dc281`
for the seven Phase 4 sub-steps; `5701621` for the limitedwidth
fix-forward. **Phase 4 retrospective at `080fe57`** is the canonical
archaeological record — Section 1 trajectory data, Section 2
compound-dividend-from-pattern-bank framing, Section 4
kickoff-evidence-grounding reflex, Section 5
helper-decomposition-for-testability pattern, Section 6
gate-discipline-evolution meta-pattern (with the 4.6 → 4.6.5
fix-forward as the worked example).

---

### Phase 5a — Gradebook + completion

The phase where the activity becomes a first-class Moodle citizen
in course-level workflows. After Phase 5a, scorecard submissions
propagate to the gradebook, the activity supports custom completion
rules, course-level completion criteria can include "this activity
is complete." Without Phase 5a, the activity is an isolated tool;
with it, the activity participates in Moodle's broader course
infrastructure.

This is a calibration-tax-rich phase. The Grade API is genuinely
new conceptual surface for any plugin that hasn't shipped it before.
Expect the upper-bound prediction to bite at least once.

**Scope.**

- `lib.php` Grade API callbacks: `<plugin>_grade_item_update`,
  `<plugin>_update_grades`, `<plugin>_grade_item_delete`,
  `<plugin>_get_user_grades` (or the equivalent quartet for the
  chosen grade method).
- Grade method selection logic — latest / highest / first / average,
  per SPEC convention. Most activity-mods default to latest-attempt
  overwrites; alternatives are typically v1.1+ scope.
- Submit-time grade propagation hook — wired into the Phase 3
  submit endpoint's after-commit lifecycle.
- `lib.php` completion rule registration:
  `<plugin>_get_completion_active_rule_descriptions`,
  `FEATURE_COMPLETION_HAS_RULES` declaration, custom rule logic.
- `classes/completion/custom_completion.php` — Moodle 5.x's
  `activity_custom_completion` API class for operator-facing
  completion-rule integration.
- `db/upgrade.php` savepoint — typically the first schema migration
  if the completion rule requires a new column on the activity row.
- `mod_form.php` — completion rule checkbox + completion-rule
  registration helpers (`add_completion_rules`,
  `completion_rule_enabled`).
- Items-CRUD lifecycle hooks if the grade method depends on item
  state (e.g., auto-grademax recompute on item add/update/delete
  while no attempts exist).
- Backfill upgrade savepoint if existing v0.x deployments need
  retroactive grade-item creation for already-attempted activities.

**Dependencies.** Phase 1 schema + capability set. Phase 3 attempts
+ responses (the data the gradebook propagates from). Phase 2
authored content (so grades are anchored to real activity state).

**Typical sub-step decomposition.**

1. SPEC clarification if the grade method selection wasn't already
   sharp at the SPEC level.
2. Gradebook callbacks + `FEATURE_GRADE_HAS_GRADE` flip.
3. Items-CRUD lifecycle hooks for auto-grademax recompute.
4. Submit-time grade propagation hook.
5. Completion integration (`FEATURE_COMPLETION_HAS_RULES` +
   completion rule column + custom_completion class).
6. Backfill upgrade savepoint for existing deployments (if
   applicable).
7. Docs (README gradebook + completion sections, CHANGES, version
   bump, SPEC sha verified).

mod_scorecard: 7 sub-steps (5a.0–5a.6) plus 1 fix-forward (`b3ac78b`,
the synchronous backfill iteration that broke production because
`grade_update` is structurally blocked during plugin upgrade
context). 8 round-trips total. The fix-forward established the
empirical-upgrade-application gate discipline.

**Calibration anchor.**

- First activity-mod: 5–8 sub-steps; **6–10 round-trips**.
  Calibration-tax phase — Grade API is genuinely new conceptual
  surface; predictions should NOT extrapolate from Phase 4's pace.
- N+1 activity-mod: 5–7 sub-steps; **5–7 round-trips**. The Grade
  API quartet absorbed; the completion-rule pattern absorbed; the
  upgrade-context-divergence reflex banked.

**Common Type B friction surfaces.**

- Grade callback signature conventions — Moodle's Grade API has
  specific parameter shapes that don't match other Moodle subsystem
  callbacks. Get them wrong and PHPUnit fails opaquely.
- Completion rule registration boilerplate — the
  `add_completion_rules` / `completion_rule_enabled` /
  `FEATURE_COMPLETION_HAS_RULES` triad has subtle interactions.
- Upgrade.php savepoint discipline — especially for backfill
  operations. `grade_update()` is **structurally blocked** during
  upgrade context (it fires `upgrade_ensure_not_running()` via
  internal `grade_item->is_locked()` path); PHPUnit doesn't
  replicate the guard so tests can pass against
  100%-broken-in-production code.
- Test-context vs production-context divergence — PHPUnit upgrade-
  path tests bypass Moodle's main upgrade flow setup, so guards
  that fire in production don't fire in tests. Empirical upgrade
  application via DDEV is required gate verification, not optional.
- `grade_get_grades` is display-oriented; gradetype/grademax
  assertions need `grade_item::fetch` for canonical introspection.
- MODE_REQUEST cache priming — new gate-check callers should use
  direct `$DB->count_records` rather than cached helpers to avoid
  priming the cache and breaking later callers in the same request.

**Gate-discipline shapes that apply.**

- PHPUnit on grade callbacks (state-modifying; grade items + grades
  inserted).
- phpcs zero/zero plugin-wide.
- **Empirical upgrade application** (`php admin/cli/upgrade.php`
  in DDEV) for any sub-step touching `db/upgrade.php` — this is
  the new gate discipline Phase 5a established.
- Walkthrough: "create activity with gradeenabled=1, submit attempts,
  confirm gradebook column appears with correct values; toggle
  gradeenabled to 0, confirm column disappears but grade history
  preserved; toggle back, confirm column reappears."
- Walkthrough completion: "set completion rule, submit, confirm
  Done checkmark appears in course; mark course-complete with
  completion criteria including the activity, confirm flow."

**Reflexes typically banked at Phase 5a.**

- `feedback_moodle_grade_item_fetch_for_introspection.md` — display-
  oriented vs canonical-introspection helper distinction.
- `feedback_moodle_request_cache_priming.md` — direct DB count for
  gate checks vs cached helper.
- `feedback_moodle_upgrade_mode_residual_flag.md` — post-savepoint
  cache-rebuild behavior in test contexts.
- `feedback_moodle_grade_update_blocked_in_upgrade_context.md` —
  never call `grade_update()` from a savepoint; lifecycle-hook
  fallback or cron-deferred adhoc task.
- `feedback_moodle_phpunit_reinit_after_version_bump.md` (carried
  forward from Phase 4.7; load-bearing throughout Phase 5a's five
  version bumps).
- `feedback_moodle_cs_diverges_from_core.md` — copying boilerplate
  from a Moodle core mod plugin doesn't guarantee phpcs cleanliness.

**mod_scorecard archaeology.** Commits `a562554` through `01c3889`
for the seven Phase 5a sub-steps plus `b3ac78b` for the fix-forward.
**Phase 5a retrospective at `ffdcddb`** is the canonical
archaeological record — Section 3 (the 5a.5 → fix-forward sequence
as worked example), Section 4 (test-context vs production-context
divergence as load-bearing methodology), Section 5 (six Type B
reflex archaeological summaries), Section 6 (gate-discipline
evolution meta-pattern).

The Phase 5a retrospective is required reading before drafting any
plugin's first Grade API surface kickoff. The 5a.5 → fix-forward
narrative is methodology-rich — the synchronous backfill broke as
committed because the test environment didn't replicate production's
upgrade-context guard. The fix-forward's lifecycle-hook fallback
plus the empirical-upgrade-application gate discipline are both
durable methodology contributions.

---

### Phase 5b — Privacy + backup/restore

The phase where the activity becomes a responsible Moodle citizen.
After Phase 5b, operators can fulfil GDPR / data subject requests
through Moodle's standard Privacy and policies tooling, and courses
containing the activity can be backed up and restored cleanly with
all learner submission data round-tripping verbatim. Without Phase
5b, the activity has compliance-and-portability gaps that block
production deployment in regulated contexts.

Two distinct conceptual surfaces share this phase: the Privacy API
(metadata + export + delete contracts) and the Backup API (nested
backup steps + restore-side processors with id mapping).
Per-subsystem calibration-tax compression curves apply — each
surface absorbs in 1–2 sub-steps and the overall phase compresses
quickly given mod_assign / mod_choice as canonical references.

**Scope.**

- `classes/privacy/provider.php` — three interface implementations:
  - `\core_privacy\local\metadata\provider` (metadata declaration —
    typically scaffolded at Phase 1)
  - `\core_privacy\local\request\plugin\provider` (export +
    plugin-level delete contract)
  - `\core_privacy\local\request\core_userlist_provider` (user
    listing for context-scoped delete)
- Three deletion methods on the plugin provider:
  `delete_data_for_all_users_in_context`, `delete_data_for_users`,
  `delete_data_for_user`. Children-first deletion ordering required
  by advisory FKs.
- `backup/moodle2/backup_<plugin>_stepslib.php` — nested backup
  elements via `backup_nested_element`, source declarations via
  `set_source_table`, userdata-gating via
  `if ($this->get_setting_value('userinfo'))` for user-data tables.
- `backup/moodle2/restore_<plugin>_stepslib.php` — `restore_path_element`
  declarations matching backup XML structure, `process_<element>`
  methods, `set_mapping`/`get_mappingid` for in-plugin cross-
  references.
- `backup/moodle2/restore_<plugin>_activity_task.class.php` —
  `define_decode_contents` and `define_decode_rules` if any text
  fields contain `@@PLUGINFILE@@` tokens (NOT NEEDED if the
  authoring editors use `maxfiles=0` — see Phase 5b kickoff Q1
  reversal pattern below).

**Dependencies.** Phase 1 schema + capability set. Phase 3 attempts
+ responses (the user-data tables backup/restore + privacy address).
Phase 5a is **not** a hard prerequisite — privacy + backup/restore
can ship before gradebook integration.

**Typical sub-step decomposition.**

1. Privacy provider metadata fix + export contract (any v0.x
   metadata-completeness gaps closed; export contract implemented
   with LEFT JOIN for soft-deleted entities).
2. Privacy provider delete contract (three deletion scopes +
   children-first ordering).
3. Backup steps for authoring entities (always backed up; soft-
   deleted rows preserved per spec convention).
4. Backup steps for user-data tables (userdata-gated via the
   userinfo setting).
5. Restore steps for full nested structure (process methods +
   set_mapping/get_mappingid for in-plugin cross-references).
6. Docs (README privacy + backup/restore sections, CHANGES, version
   bump, SPEC sha verified).

mod_scorecard: 6 sub-steps (5b.1–5b.6), no fix-forward. 6 round-
trips total. The cleanest phase by absence-of-fix-forwards under
the methodology.

**Calibration anchor.**

- First activity-mod: 5–7 sub-steps; **5–10 round-trips**. Two
  distinct conceptual surfaces (Privacy API and Backup API) plus an
  integration sub-step (restore) plus a docs/release sub-step.
- N+1 activity-mod: 5–7 sub-steps; **5–7 round-trips**. Privacy +
  Backup APIs absorbed via prior plugin work; only domain-specific
  metadata declarations and entity-specific structure declarations
  vary per plugin.

mod_scorecard's actual at 6 sub-steps / 6 round-trips landed at
the lower bound of the first-activity-mod range — the Privacy and
Backup conceptual surfaces compressed quickly under per-subsystem
pattern banks (each surface had its own calibration tax paid in 1
sub-step and absorbed in the next).

**Common Type B friction surfaces.**

- Privacy API metadata declaration completeness — graph-traversal
  links between user-data tables (e.g., response.itemid linking
  responses to items) must be declared in metadata for export to
  resolve relationships. mod_scorecard's v0.5.0 missed itemid
  metadata; bundled the fix at 5b.1.
- `backup_nested_element` + `add_child` sequencing — declarations
  precede tree-building precede sources. Get the order wrong and
  the backup framework silently produces incomplete output.
- `set_mapping`/`get_mappingid` sequencing at restore — parents
  before children (items + bands processed before
  attempts + responses; framework enforces via path-element order).
  In-plugin cross-references (attempt.bandid → bands;
  response.itemid → items) round-trip via this mechanism, NOT
  via `annotate_ids` at backup time.
- File annotation rules — look at `maxfiles` config in the
  authoring editors FIRST before adding defensive `annotate_files`
  calls in backup or `define_decode_contents` entries in restore.
  If the editors use `maxfiles=0`, no @@PLUGINFILE@@ tokens reach
  the database and file-related defensive measures are misleading
  no-ops.
- `setAdminUser()` reflex for `backup_controller` /
  `restore_controller` PHPUnit fixtures — both controllers require
  a valid global `$USER` which PHPUnit doesn't authenticate by
  default.
- MOODLE_INTERNAL contextual rule for files setting global state
  (top-level `require_once`, top-level statements) — required when
  global state is set, forbidden when it isn't.
- Test classloader behaviour for abstract base classes in
  `tests/<subdir>/` paths — Moodle's test classloader doesn't
  autoload bases from subdirs even when sibling `*_test.php` files
  ARE autoloaded. Explicit `require_once(__DIR__ . '/<base>.php')`
  in consumer files is the workaround; banked at Phase 5b.5.

**Gate-discipline shapes that apply.**

- PHPUnit on each privacy contract (read-only export; state-
  modifying delete with transactional rollback).
- PHPUnit on each backup processor (file artifact production +
  inspection).
- PHPUnit on the full backup → restore round-trip with state-
  comparison.
- phpcs zero/zero plugin-wide.
- **Empirical bootstrap-state verification at every sub-step gate**
  in the operationalization shape appropriate to the sub-step's
  work. Phase 5b operationalized this discipline in five distinct
  shapes — see PHASE-5B-RETROSPECTIVE.md Section 4 for the full
  walkthrough. Future activity-mods should expect to operationalize
  whatever new shapes their conceptual surface introduces.
- Walkthrough: "as a site admin, run privacy data export for a user
  with attempts, confirm export contains scorecard data with prompt
  text + response values + snapshot fields; run privacy delete for
  a user, confirm attempts removed but items + bands + activity
  intact." Plus: "back up a course containing the activity with
  user data, restore into a new course, confirm structure round-
  trips and user data round-trips."

**Reflexes typically banked at Phase 5b.**

- `feedback_moodle_internal_contextual.md` — required for global-
  state files, forbidden for pure-class files.
- `feedback_moodle_setadminuser_for_backup_controller.md` —
  authenticated-API entry points need `$this->setAdminUser()` in
  PHPUnit.
- `feedback_commit_message_heredoc_escapes.md` — file-based commit
  message bodies for nontrivial bodies (avoids backslash-escape
  pitfalls in single-quoted heredocs).
- Autoloader-via-explicit-require_once for cross-subdir base
  classes (within-Q-disposition refinement; no separate memory
  file but documented in PHASE-5B-RETROSPECTIVE.md Section 5).

**mod_scorecard archaeology.** Commits `0ecb11b` through `1a8fb0a`
for the six Phase 5b sub-steps. **Phase 5b retrospective at
`f699fad`** is the canonical archaeological record — Section 2
(per-subsystem calibration-tax compression curves), Section 3 (the
three-Q1-reversal pattern revealing the maxfiles=0 architectural
fact), Section 4 (five-shape empirical-bootstrap-state-verification
operationalization), Section 5 (course-correction-as-scope
methodology refinement), Section 6 (within-phase pattern bank
compounding to zero friction).

The Phase 5b retrospective is required reading before drafting any
plugin's first backup/restore phase kickoff. The three-Q1-reversal
pattern is particularly valuable — kickoff defensive defaults can
be wrong when architectural state precludes the threat the defaults
address. Future kickoffs should distinguish architectural questions
(verifiable at pre-flight; defensive defaults inappropriate without
verification) from decisional questions (genuine choices among
legitimate options; defensive defaults appropriate as starting
points).

---

## 4. Sub-step shape

Independent of phase, every sub-step has a consistent structure.
Mirrors what mod_scorecard's sub-steps actually looked like across
all six phases.

A sub-step is the atomic unit of supervised-agentic phase work. Its
shape:

**1. Kickoff prompt** drafted by operator. Standard sections:

- *Context.* What's been shipped, what this sub-step adds, what's
  still pending after this sub-step ships.
- *Pre-flight verification block.* Specific commands the implementer
  runs before drafting begins — SPEC sha verification, git log
  range, git status, file reads against current state. Surfaces
  actual state before structural assumptions get committed.
- *Scope.* Specific files and code surfaces this sub-step touches.
- *Sub-step plan.* Decomposition into discrete operations within
  the sub-step.
- *Round-trip prediction.* Honest expectation, lower bound, upper
  bound. Calibration-tax framing if the surface is genuinely new.
- *Q1–Qn design questions* with kickoff recommendations. Each Q is
  a decision point with options; recommendation is provisional
  pending pre-flight evidence.
- *Pre-flags.* Anticipated friction surfaces, implementation
  shape post-confirmation.
- *Type B prediction.* What kinds of friction are anticipated.
- *Awaiting confirmation.* Explicit pause point.

Length: typically 200–600 lines depending on scope. Phase 1 sub-
step kickoffs are shorter (well-trodden conventions); Phase 5a sub-
step kickoffs longer (calibration-tax-rich surfaces).

**2. Pre-flight verification** by Claude Code in the implementing
session. SHA verification of SPEC, git log range, git status, file
reads against current state. Surfaces actual state before drafting
begins. **Q dispositions are confirmed (or revised) based on
pre-flight evidence.** This is the kickoff-evidence-grounding
reflex Phase 4 retrospective Section 4 named; Phase 5b sharpened
it (Section 3 of PHASE-5B-RETROSPECTIVE.md): kickoff defensive
defaults are themselves provisional pending architectural
verification.

**3. Implementation** against confirmed Q dispositions. Implementation
pauses at gate before commit.

**4. Gate verification**. Three layers:

- *Code-level.* PHPUnit pass; phpcs zero/zero.
- *Empirical bootstrap-state.* Operationalization shape varies per
  sub-step type — read-only export verification, state-modifying
  with transactional rollback, file artifact production + inspection,
  multi-invocation comparison, bidirectional round-trip with state-
  comparison. The discipline's roots are in Phase 5a.5-fix-forward;
  Phase 5b extended it to every sub-step's gate.
- *Operator-walkthrough.* Recommended for sub-steps touching
  operator-facing surfaces. Confirms affordances and integration
  points the test suite doesn't exercise.

**5. Walkthrough** (optional; recommended for operator-facing
surface changes). User confirms via explicit message.

**6. Commit** with file-based message body if non-trivial. Banked
discipline from Phase 5b.4 — write to `/tmp/commit_msg_<topic>.txt`,
invoke via `git commit -F` rather than single-quoted heredocs with
potential backslash interactions.

**7. Push** outside Claude (manual; harness restriction). Push
output surfaces in the next user message; the round-trip closes.

**Round-trip = one cycle** of (kickoff prompt → implementation →
gate → user response). Predicted at kickoff time; honest accounting
at gate. Type B friction (tooling vs code-logic) tracked separately.

A sub-step that requires a fix-forward becomes two round-trips: the
original sub-step plus the fix-forward sub-step. Fix-forwards are
methodologically important — they typically establish a new gate
discipline that prevents the same failure mode going forward (Phase
4.6 → 4.6.5 established parallel-surface-comparison; Phase 5a.5 →
fix-forward established empirical-upgrade-application; both became
durable disciplines applied at all subsequent gates).

---

## 5. Calibration anchors

Reference data for predicting round-trips. Drawn from mod_scorecard's
actuals plus the inheritance-aware predictions for subsequent
activity-mod plugins.

### First activity-mod calibration tax

mod_scorecard was the first activity-mod under this methodology;
its actuals are the canonical "first activity-mod" anchors:

| Phase | mod_scorecard sub-steps | Round-trips (incl. fix-forwards) |
|-------|-------------------------|----------------------------------|
| 1 — Skeleton | 6 | 7 (1 fix-forward) |
| 2 — Authoring | 6 | 6 |
| 3 — Learner submission | 6 | 8 (2 fix-forwards) |
| 4 — Reporting | 7 | 8 (1 fix-forward) |
| 5a — Gradebook + completion | 7 | 8 (1 fix-forward) |
| 5b — Privacy + backup/restore | 6 | 6 |
| **Total** | **38** | **43 (5 fix-forwards)** |

Five of the six phases produced at least one fix-forward — Phase 2
and Phase 5b were the exceptions. Each fix-forward typically
established a new gate discipline that prevented the same failure
mode going forward.

### N+1 activity-mod calibration tax

Predicted ranges for the second activity-mod under this methodology
(prior activity-mod pattern bank available):

| Phase | Sub-steps | Round-trips |
|-------|-----------|-------------|
| 1 — Skeleton | 6 | 6–8 (skeleton conventions absorbed) |
| 2 — Authoring | 4–6 | 5–8 (manage UI patterns absorbed) |
| 3 — Learner submission | 6–8 | 6–10 (submission patterns absorbed; processing engine still domain-specific) |
| 4 — Reporting | 5–7 | 6–7 (reporting patterns absorbed) |
| 5a — Gradebook + completion | 5–7 | 5–7 (Grade API absorbed; completion rule absorbed) |
| 5b — Privacy + backup/restore | 5–7 | 5–7 (Privacy + Backup APIs absorbed) |
| **Total** | **31–41** | **33–47** |

These are predictions, not promises. The compression depends on:

- Whether the new activity-mod's domain-specific processing engine
  resembles mod_scorecard's scoring engine (some pattern transfer)
  or is genuinely novel (no transfer; full Phase 3 calibration tax
  again).
- Whether the new activity-mod's authoring entities resemble
  mod_scorecard's items + bands (form scaffolding transfers cleanly)
  or have qualitatively different shapes (e.g., file-bearing
  content with @@PLUGINFILE@@ token handling — completely different
  Phase 2 + Phase 5b conceptual surface).
- Whether new gate disciplines surface during the new activity-mod's
  work that weren't anticipated by mod_scorecard's progression.

**Calibration honesty operates bidirectionally.** Phases inheriting
rich pattern banks against well-known surfaces predict toward
lower-bound; phases introducing genuinely new conceptual surfaces
predict toward upper-bound with fix-forward allowance. mod_scorecard
Phase 5b hit lower-bound (6 round-trips against 5–10 predicted);
mod_scorecard Phase 5a hit upper-middle (8 round-trips against 5–10
predicted). Both calibration-honest in different directions.
PHASE-5B-RETROSPECTIVE.md Section 7 has the full framing.

### Per-phase Type B budget

Roughly 6–10 friction instances per phase regardless of sub-step
count. mostly tooling Type B (phpcs nits, fixture quirks, tool-use
issues) absorbing as the pattern bank fills. Code-logic Type B
(genuine implementation errors that would ship if not caught) is
rarer; typically 0–2 per phase.

The per-phase Type B count appears roughly invariant to sub-step
count — Phase 4 absorbed ~8 Type B across 8 sub-steps; Phase 5a
absorbed ~8 Type B across 8 sub-steps; Phase 5b absorbed 8 Type B
across 6 sub-steps. The bound seems to be conceptual-surface-driven
rather than sub-step-count-driven. Worth tracking across future
activity-mod work to see whether the invariance holds.

### Predicting honestly

The kickoff prediction shape that's worked across mod_scorecard:

- *Range.* 5-95th-percentile round-trip count.
- *Direction.* Where in the range the trajectory is likely to land,
  based on pattern-bank coverage and conceptual-surface novelty.
- *Sub-step decomposition.* Specific sub-step plan with per-sub-step
  round-trip predictions.
- *Type B anticipation.* Tooling vs code-logic friction expected.
- *Fix-forward allowance.* Explicitly named if a fix-forward is
  plausible.

Vague "predict honestly" framings underperform specific
"predict honestly relative to expected calibration tax for the
conceptual surfaces in scope" framings. Name the surfaces; name the
pattern-bank coverage; name whether fix-forwards are anticipated.

---

## 6. When the template doesn't apply

Worth naming explicitly so future operators don't force-fit.

The phase progression assumes:

- **Activity-mod plugin** (`mod/<name>/`) — not auth, local, block,
  theme, format plugin types.
- **Learners submit content; operators view results.** The activity
  has a learner-facing submission flow producing per-attempt data.
- **Activity has gradeable output** (or activity has completion-
  gateable output). Phase 5a's gradebook integration assumes there's
  something to grade.
- **Operators want backup/restore + privacy support eventually.**
  Phase 5b assumes these are in scope; some prototype-stage plugins
  may legitimately defer.

The template **doesn't apply cleanly** to:

- **Plugins where there's no learner submission** (e.g., pure-content
  activity-mods like mod_book or mod_resource). Phase 3 collapses to
  nothing or near-nothing; Phase 4 collapses (no attempts to report
  on); Phase 5a's gradebook half collapses (nothing to grade).
- **Plugins without grading** (Phase 5a's gradebook half collapses;
  completion rule may still apply, so Phase 5a survives in reduced
  scope).
- **Non-activity-mod plugin types** (auth, local, block, theme,
  format). Different lifecycle, different conventions, different
  phase shape entirely. The methodology disciplines transfer
  (kickoff-evidence-grounding, empirical-bootstrap-state-verification,
  pause-verify-commit rhythm); the phase shape doesn't.
- **Plugins where the activity is itself an authoring tool** (editor
  extensions, structured-content authoring tools). Phase 2's manage
  UI takes much heavier scope; Phase 3 may be inverted (operators
  submit, learners consume).
- **Plugins integrating with Moodle subsystems that mod_scorecard
  doesn't touch** (events subsystem if the activity is event-driven;
  message subsystem if it sends notifications; AI subsystem via the
  AI Providers API; file subsystem for file-bearing content). Each
  introduces its own conceptual surface and its own calibration
  tax; the phase shape may need a Phase 5c or Phase 5d added to
  cover them.

When the template doesn't apply cleanly, fall back to first-
principles scoping. The retrospectives are still useful for
methodology disciplines (kickoff-evidence-grounding,
empirical-bootstrap-state-verification, calibration-tax framing,
gate-discipline-evolution meta-pattern, course-correction-as-scope
authority) even when the phase progression itself doesn't transfer.

For non-activity-mod plugin types, future LMS Light plugin work may
warrant analogous progression templates — e.g.,
MOODLE-LOCAL-PLUGIN-PHASES.md, MOODLE-AUTH-PLUGIN-PHASES.md — with
their own canonical exemplars (local_welcomeemail; auth_magiclink).
Out of scope for this document; flagged here as future work.

---

## 7. Closing

mod_scorecard's six-phase progression (Phase 1 skeleton → 2
authoring → 3 learner submission → 4 reporting → 5a gradebook +
completion → 5b privacy + backup/restore) is one path through the
activity-mod problem space, not THE path. The numbering convention
(5a / 5b instead of 5 / 6) is deliberate, recognising that the
maturity work at the end of an activity-mod's v0.x cycle often
splits across multiple release cycles based on operator priority
and calibration-tax independence.

Future activity-mods may legitimately:

- **Reorder phases** — e.g., if Phase 4 reporting depends on
  gradebook integration for course-total computation, ship Phase
  5a before Phase 4. mod_scorecard's reporting was independent of
  gradebook so this didn't apply; another activity-mod's might.
- **Merge phases** — e.g., Phase 5a + Phase 5b collapsed into a
  single Phase 5 if the maturity scope is light. mod_scorecard had
  enough work in each surface to warrant separate phases; another
  might not.
- **Split phases** — e.g., Phase 4 split into 4a reporting + 4b
  export if the export side has heavy domain-specific shape.
  mod_scorecard's CSV export was lightweight enough to fit in one
  Phase 4 sub-step; another might warrant its own phase.
- **Add phases** — e.g., a Phase 5c integration with a Moodle
  subsystem mod_scorecard doesn't touch (events, messages, AI,
  files).
- **Skip phases** — e.g., a prototype-stage plugin that defers
  Phase 4 reporting to v1.x, or a non-graded activity that defers
  Phase 5a's gradebook half indefinitely.

The methodology disciplines are the durable contributions:
kickoff-evidence-grounding, pre-flight verification with Q
disposition confirmation, pause-verify-commit rhythm, gate-
discipline-checklist accumulation across phases, empirical-
bootstrap-state-verification scaling across operation shapes,
calibration-tax framing, per-subsystem compression curves, course-
correction-as-scope authority, file-based commit message hygiene.
None of these are activity-mod-specific. All transfer to non-
activity-mod plugin work and to LMS Light development generally.

The phase shape adapts to the plugin; the methodology adapts to
the operator-implementer collaboration.

---

## References

- **mod_scorecard repo** — `github.com/jport500/moodle-mod_scorecard`.
  v0.6.0 at commit `1a8fb0a`; Phase 4/5a/5b retrospectives at
  `080fe57` / `ffdcddb` / `f699fad`.
- **Phase 4 retrospective** — `docs/PHASE-4-RETROSPECTIVE.md` in
  mod_scorecard's repo. Establishes the retrospective shape;
  Section 5 helper-decomposition pattern is portable.
- **Phase 5a retrospective** — `docs/PHASE-5A-RETROSPECTIVE.md` in
  mod_scorecard's repo. Calibration-tax framing established; Grade
  API archaeology; Section 4 test-context vs production-context
  divergence as load-bearing methodology.
- **Phase 5b retrospective** — `docs/PHASE-5B-RETROSPECTIVE.md` in
  mod_scorecard's repo. Per-subsystem compression curves;
  three-Q1-reversal pattern revealing maxfiles=0; five-shape
  empirical-bootstrap-state-verification; course-correction-as-
  scope refinement; within-phase pattern bank compounding to zero
  friction.
- **CONTEXT.md** in this repo — LMS Light project context, deployment
  model, plugin ecosystem, working conventions, code quality gates.
  Read first for any new plugin work.
- **LESSONS.md** in this repo — portable methodology lessons from
  prior LMS Light plugin work. Methodology disciplines that apply
  across plugin types.
- **METHODOLOGY.md** in this repo — synthesis document drawing
  from Phase 4 / 5a / 5b retrospectives plus this phase progression
  template plus banked memory files. To be drafted in a separate
  session; references this document as a known concept rather than
  inlining.
