# MOODLE-ACTIVITY-MOD-PHASES.md drafting prompt

## Context

The mod_scorecard plugin is the canonical implementation of supervised-agentic Moodle activity-mod plugin development for LMS Light. Six phases shipped (1, 2, 3, 4, 5a, 5b) at v0.6.0. Three retrospectives on origin (`docs/PHASE-4-RETROSPECTIVE.md`, `docs/PHASE-5A-RETROSPECTIVE.md`, `docs/PHASE-5B-RETROSPECTIVE.md`) capture the methodology archaeology phase-by-phase.

This prompt produces the **first** of two synthesis documents in `lms-light-docs/`: the activity-mod **phase progression template**. The phase template extracts the generalizable phase shape from mod_scorecard's specific progression so future activity-mod plugins (mod_certifier, mod_anything_next) can reach for "Phase N covers X, Y, Z" without re-deriving from mod_scorecard's archaeology.

The second synthesis document, METHODOLOGY.md, follows in a separate session and references this document as a known concept rather than inlining its content.

## Deliverable

A markdown document at `lms-light-docs/MOODLE-ACTIVITY-MOD-PHASES.md`. Single commit. Pushed via the manual-out-of-Claude path (assume same harness behavior as mod_scorecard's repo — verify at pre-flight).

Audience: future supervised-agentic Moodle plugin work. Operator (drafting kickoff prompts) and implementer (executing in Claude Code) both read this. NOT customer-facing; NOT end-user documentation.

Tone: structural reference. Less narrative than retrospectives, more reference-shaped — but with examples grounded in mod_scorecard's actual experience rather than abstract templates.

## Pre-flight verification (load-bearing)

**lms-light-docs current state must be read before drafting.** This is the kickoff-evidence-grounding reflex Phase 4 retrospective Section 4 named, applied to this drafting work specifically. I (the prompt author) don't know the current state of lms-light-docs from my drafting environment; pre-flight confirms before structural assumptions get committed to the document.

Pre-flight commands (assuming lms-light-docs is cloned alongside mod_scorecard locally — verify path with John before running):

- `ls lms-light-docs/` → directory listing. What's actually there?
- `cat lms-light-docs/README.md` (if exists) → current README state. Does it orient on the docs archive at any depth? If yes, MOODLE-ACTIVITY-MOD-PHASES.md should be referenced from it after landing.
- `cat lms-light-docs/CONTEXT.md` → existing project-context document. Cross-reference for anything that would duplicate.
- `cat lms-light-docs/LESSONS.md` → existing lessons document. Cross-reference for anything that would duplicate or refine.
- `git -C lms-light-docs log --oneline | head -20` → recent commit history. Establishes voice/precedent for new file additions.
- `git -C lms-light-docs status` → expect clean before drafting.

Pre-flight output should surface BEFORE drafting. If lms-light-docs's actual state differs from what this prompt assumes (e.g., MOODLE-ACTIVITY-MOD-PHASES.md already exists in some draft form; CONTEXT.md already covers phase progression at some depth; LESSONS.md has phase-specific content that should be referenced rather than duplicated), surface as an architectural Q before drafting.

Cross-references for content sourcing (in mod_scorecard's repo at `/Users/John/projects/mutms/moodle/public/mod/scorecard/`):

- `docs/PHASE-4-RETROSPECTIVE.md` — Phase 4 archaeology; Section 1 trajectory data + Section 5 helper-decomposition pattern particularly relevant.
- `docs/PHASE-5A-RETROSPECTIVE.md` — Phase 5a archaeology; calibration-tax framing established here.
- `docs/PHASE-5B-RETROSPECTIVE.md` — Phase 5b archaeology; per-subsystem compression curves, three-Q1-reversal pattern, five-shape verification, course-correction-as-scope.
- `README.md` — Phase status table is the consumer-facing version of phase progression.
- `CHANGES.md` — operator-facing release notes per phase.
- `docs/SPEC.md` — feature-shape contract; Section 9 covers capabilities, grade methods, backup/restore, Privacy API.

## Document scope

Eight sections. Not all sections will be the same length; some are heavy (Section 3 phase-by-phase content), some are light (Section 7 closing).

### Section 1 — Purpose + audience

What this document is, who reads it, when. Short — half a page.

Specifically: this is the structural reference for activity-mod phase progression. Operator drafting kickoff prompts reaches for "we're doing Phase N" and uses this doc to know roughly what that means. Implementer in Claude Code reads relevant phase section before pre-flight to ground "what does this phase cover?" in archive rather than re-derivation.

NOT a tutorial; NOT a how-to; NOT prescriptive. The phase progression is a template, not a mandate. mod_scorecard's actual progression is one valid path; another activity-mod might legitimately reorder, merge, or split phases based on its specific scope.

### Section 2 — Phase progression overview

The phase list with one-paragraph summary each. Reference table at top:

| Phase | Scope summary | Calibration anchor | mod_scorecard exemplar |
|-------|---------------|-------------------|------------------------|
| 1 — Skeleton | Install schema, capabilities, mod_form, view, privacy provider scaffold, settings-only backup/restore, skeleton tests | 6-10 sub-steps; ~15-20 round-trips first time | mod_scorecard Phase 1 (v0.1.0) |
| 2 — Authoring | Manage screen with CRUD tabs, soft-delete, lifecycle gates | 4-7 sub-steps; ~7-12 round-trips | mod_scorecard Phase 2 (v0.2.0) |
| 3 — Learner submission | Submission form, validation, attempt + response save, scoring/processing engine, snapshotting, result page | 5-7 sub-steps; ~8-15 round-trips | mod_scorecard Phase 3 (v0.3.0) |
| 4 — Reporting | Reports surface (capability-gated table, expandable detail, group filter, CSV export, pagination) | 6-8 sub-steps; ~7-10 round-trips | mod_scorecard Phase 4 (v0.4.0) |
| 5a — Gradebook + completion | Grade API integration with grade method (latest/highest/etc), activity completion via custom rule | 5-8 sub-steps; ~6-10 round-trips | mod_scorecard Phase 5a (v0.5.0) |
| 5b — Privacy + backup/restore | Full Privacy API provider implementation, nested backup steps, restore steps with id mapping | 5-7 sub-steps; ~5-10 round-trips | mod_scorecard Phase 5b (v0.6.0) |

The summary paragraph below the table covers:
- Phases 1-3 are the activity functional core; you can't ship learner submissions without Phase 3.
- Phase 4 is reporting; valuable but not strictly required for v0.x.0 ship if a workflow doesn't need cross-attempt visibility.
- Phase 5a (gradebook + completion) makes the activity a first-class Moodle citizen integrated with course-level workflows.
- Phase 5b (privacy + backup/restore) makes the activity a responsible Moodle citizen — operators can fulfill GDPR requests; courses can be backed up and restored cleanly.
- Phases beyond 5b are plugin-specific feature work; no general template applies.

The numbering convention (5a / 5b instead of 5 / 6) is deliberate: gradebook and privacy are independent surfaces that often ship as separate releases despite being conceptually "Phase 5 maturity work." Future activity-mods might collapse to a single Phase 5 if scope warrants, or split further if the work is heavy.

### Section 3 — Phase-by-phase detail

Six subsections, one per phase. Each subsection covers:

- **Scope** — what code surface this phase touches. Specific files (`db/install.xml`, `mod_form.php`, `lib.php`, etc.) named.
- **Dependencies** — what must be in place before this phase can start. Phase 1 has none; Phase 2 needs Phase 1's schema + capabilities; etc.
- **Typical sub-step decomposition** — 4-7 sub-steps with one-line descriptions.
- **Calibration anchor** — sub-step count + round-trip range. **First-time prediction** vs **with-mature-pattern-bank prediction** named separately. mod_scorecard's actuals as evidence.
- **Common Type B friction surfaces** — what tooling/convention friction this phase typically surfaces. Used by future operators to anticipate where Type B work lands.
- **Gate-discipline shapes that apply** — empirical-bootstrap-state-verification shapes (read-only? state-modifying? file artifact? round-trip?), walkthrough scope, etc.
- **Reflexes the phase typically banks** — which `feedback_*.md` memories tend to surface during this phase work.
- **mod_scorecard's archaeology pointer** — specific commit ranges + retrospective sections to read for examples.

Phase 1 — Skeleton:
- Scope: db/install.xml, db/access.php, mod_form.php, view.php (skeleton), lib.php (instance lifecycle: add/update/delete), backup/restore (settings-only), classes/privacy/provider.php (metadata only), version.php, lang/en/, basic PHPUnit fixtures.
- Calibration anchor for first activity-mod: 6-10 sub-steps; ~15-20 round-trips (calibration tax for the activity-mod skeleton conventions). With mature pattern bank from prior activity-mod: 6-10 sub-steps; ~8-12 round-trips.
- Common Type B: phpcs comment-style nits (Moodle convention divergent from PSR), MOODLE_INTERNAL guard placement, version.php numeric ordinal convention, install.xml schema validation.
- Reflexes typically banked: phpcs Moodle conventions, MOODLE_INTERNAL contextual rule, version stamp ordinal convention, db/install.xml field type conventions.

Phase 2 — Authoring:
- Scope: manage.php, classes/output/manage_renderer.php (or similar), classes/form/*.php for authoring forms, db/access.php capabilities for authoring caps, item-soft-delete + reorder lifecycle, lang strings.
- Calibration anchor: 4-7 sub-steps; ~7-12 round-trips first time / ~5-8 with mature pattern bank.
- Common Type B: tab navigation conventions, soft-delete vs hard-delete semantics for first-encounter, lifecycle gate (canManageItems-shaped checks).
- Reflexes typically banked: soft-delete semantics, lifecycle-gate convention, tab navigation pattern.

Phase 3 — Learner submission:
- Scope: submit.php (or similar entry point), classes/form/submit_form.php, classes/local/scoring_engine.php (or domain-specific processing), attempt + response save lifecycle, snapshotting at submit time, result.php, lang strings.
- Calibration anchor: 5-7 sub-steps; ~8-15 round-trips first time / ~6-10 with mature pattern bank.
- Common Type B: validation messaging conventions, snapshot field semantics, transactional save lifecycle, capability gating for submit vs view-own-submission.
- Gate-discipline shape: state-modifying with transactional rollback (fixture creates + processes attempt; assert state then teardown).
- Reflexes: snapshot semantics, transactional save patterns, getDataGenerator quirks for activity-specific records.

Phase 4 — Reporting:
- Scope: report.php, classes/output/report_renderer.php, classes/output/report_table.php (flexible_table subclass), export.php, helper functions in locallib.php (data shape: `*_get_attempts()`, `*_get_attempt_responses()`).
- Calibration anchor: 6-8 sub-steps; ~7-10 round-trips first time / ~6-7 with mature pattern bank.
- Common Type B: flexible_table dynamic-property reflex (Moodle base class uses runtime properties), capability-graduation in operator UI (different views for different caps), helper-decomposition for testability.
- Gate-discipline shape: read-only display verification + capability-separated walkthrough.
- Reflexes: flexible_table dynamic-property, capability-graduation pattern, helper-decomposition-for-testability, parallel-surface comparison at release time.

Phase 5a — Gradebook + completion:
- Scope: grade_item callbacks in lib.php (`*_grade_item_update`, `*_update_grades`, `*_grade_item_delete`), grade method selection + UI, attempts→grade propagation hooks, completion rule registration in lib.php (`*_get_completion_active_rule_descriptions`, custom rule logic).
- Calibration anchor: 5-8 sub-steps; ~6-10 round-trips. Calibration-tax phase — Grade API is genuinely new conceptual surface; predictions should NOT extrapolate from Phase 4's pace.
- Common Type B: grade callback signature conventions, completion rule registration boilerplate, upgrade.php savepoint discipline (especially for backfill operations).
- Gate-discipline shape: state-modifying (grade items + grades inserted); empirical upgrade application against real production data when backfill scope is involved.
- Reflexes: grade item fetch for introspection, request cache priming, upgrade context residual flag, grade update blocked in upgrade context, PHPUnit re-init after version bump.
- mod_scorecard archaeology: PHASE-5A-RETROSPECTIVE.md Section 4 (calibration-tax framing introduced). Particularly the 5a.5 → fix-forward sub-step is methodology-rich; the synchronous backfill broke as committed because it called `grade_update()` in upgrade context where the call is structurally blocked. Worth reading before any plugin's first Grade API surface.

Phase 5b — Privacy + backup/restore:
- Scope: classes/privacy/provider.php (full export + delete contracts), backup/moodle2/backup_*_stepslib.php (nested elements for content + user data), backup/moodle2/restore_*_stepslib.php (process methods + set_mapping/get_mappingid for in-plugin cross-references).
- Calibration anchor: 5-7 sub-steps; ~5-10 round-trips. Two distinct conceptual surfaces (Privacy API and Backup API), each with its own calibration tax. Per-subsystem compression curves apply — Privacy API absorbs in 1-2 sub-steps; Backup API absorbs in 1-2 sub-steps; restore is conceptually distinct from backup but inherits backup's pattern bank.
- Common Type B: Privacy API metadata declaration completeness, backup_nested_element + add_child sequencing, set_mapping/get_mappingid sequencing (parents before children), file annotation rules (look at maxfiles config FIRST before defensive annotation).
- Gate-discipline shapes operationalized in five distinct types across this phase — reference PHASE-5B-RETROSPECTIVE.md Section 4 for the full walkthrough.
- Reflexes: MOODLE_INTERNAL contextual, setAdminUser-for-backup-controller, commit-message-via-file (heredoc-escape avoidance), autoloader require_once for cross-subdir base classes.
- mod_scorecard archaeology: PHASE-5B-RETROSPECTIVE.md Section 3 (three-Q1-reversal pattern revealing maxfiles=0 architectural fact). Worth reading before drafting any backup/restore phase kickoff — kickoff defensive defaults can be wrong when architectural state precludes the threat.

### Section 4 — Sub-step shape

Generalized sub-step structure independent of phase. Mirrors what mod_scorecard's sub-steps actually looked like.

A sub-step is the atomic unit of supervised-agentic phase work. Its shape:

1. **Kickoff prompt** drafted by operator. Sections: context, pre-flight verification block, scope, sub-step plan, round-trip prediction, Q1-Q4 design questions with recommendations, pre-flags, Type B prediction, awaiting-confirmation. Length 200-600 lines depending on scope.

2. **Pre-flight verification** by Claude Code in the implementing session. SHA verification of SPEC, git log range, git status, file reads against current state. Surfaces actual state before drafting begins. Q dispositions are confirmed (or revised) based on pre-flight evidence.

3. **Implementation** against confirmed Q dispositions. Implementation pauses at gate before commit.

4. **Gate verification** — PHPUnit, phpcs, empirical-bootstrap-state-verification (shape varies per sub-step type).

5. **Walkthrough** (optional; recommended for operator-facing surface changes). User confirms by check item.

6. **Commit** with file-based message body if non-trivial.

7. **Push** outside Claude (manual; harness restriction).

Round-trip = one cycle of (kickoff prompt → implementation → gate → user response). Predicted at kickoff time; honest accounting at gate. Type B friction (tooling vs code-logic) tracked separately.

### Section 5 — Calibration anchors

Reference data for predicting round-trips. Drawn from mod_scorecard's actuals.

**First activity-mod calibration tax** (no prior activity-mod pattern bank):
- Phase 1: ~15-20 round-trips
- Phase 2: ~7-12
- Phase 3: ~8-15
- Phase 4: ~7-10
- Phase 5a: ~6-10
- Phase 5b: ~5-10

mod_scorecard was the first activity-mod under this methodology; actual round-trip counts in the upper range of these anchors.

**N+1 activity-mod calibration tax** (prior activity-mod pattern bank available):
- Phase 1: ~8-12 (skeleton conventions absorbed)
- Phase 2: ~5-8 (manage UI patterns absorbed)
- Phase 3: ~6-10 (submission patterns absorbed; processing engine still domain-specific)
- Phase 4: ~6-7 (reporting patterns absorbed)
- Phase 5a: ~5-7 (Grade API absorbed; completion rule absorbed)
- Phase 5b: ~5-7 (Privacy + Backup APIs absorbed)

These anchors are predictions, not promises. Calibration honesty operates bidirectionally — phases inheriting rich pattern banks against well-known surfaces predict toward lower-bound; phases introducing genuinely new conceptual surfaces predict toward upper-bound with fix-forward allowance.

**Per-phase Type B budget**: roughly 6-10 friction instances per phase regardless of sub-step count. Mostly tooling Type B (phpcs nits, fixture quirks, tool-use issues) absorbing as the pattern bank fills. Code-logic Type B (genuine implementation errors) is rarer; typically 0-2 per phase.

### Section 6 — When the template doesn't apply

Worth naming explicitly so future operators don't force-fit.

The phase progression assumes:
- Activity-mod plugin (`mod/<name>/`) — not auth, local, block, theme, format plugin types.
- Learners submit content; operators view results.
- Activity has gradeable output (or activity has completion-gateable output).
- Operators want backup/restore + privacy support eventually.

Template doesn't apply cleanly to:
- Plugins where there's no learner submission (e.g., pure-content activity-mods like mod_book or mod_resource — Phase 3 collapses to nothing).
- Plugins without grading (Phase 5a's gradebook half collapses; completion rule may still apply).
- Non-activity-mod plugin types (auth, local, block, theme, format) — different lifecycle, different conventions, different phase shape entirely.
- Plugins where the activity is itself an authoring tool (editor extensions, etc.) — Phase 2's manage UI takes much heavier scope; Phase 3 may be inverted (operators submit, learners consume).

When the template doesn't apply, fall back to first-principles scoping. The retrospectives are still useful for methodology disciplines (kickoff-evidence-grounding, empirical-bootstrap-state-verification, etc.) even when the phase progression itself doesn't transfer.

### Section 7 — Closing

Short. ~half a page.

Acknowledge that mod_scorecard's progression is one path, not THE path. Future activity-mods may legitimately reorder (e.g., Phase 5a before Phase 4 if reporting depends on gradebook integration), merge (Phase 5a + 5b into a single Phase 5 if scope is light), or split (Phase 4 into 4a reporting + 4b export if heavy).

The methodology disciplines transfer; the phase shape adapts.

## Out of scope

- Methodology synthesis (that's METHODOLOGY.md, separate session).
- Reflex bank itself (that's REFLEXES/, separate work).
- Kickoff prompt templates (will become an appendix to this document, but not in this initial draft).
- Plugin-type-specific phase progressions for non-activity-mods (auth, local, block, theme, format) — out of scope for this template.
- mod_scorecard plugin documentation (lives in mod_scorecard's repo).

## Length target

Honest target: 600-900 lines. Phase 5b retrospective taught the lesson that target ranges should reflect content surface, not anchor on prior-document line counts. This document's content surface is moderate — six phase sections at ~100 lines each = ~600 lines for Section 3 alone, plus Sections 1, 2, 4, 5, 6, 7 at ~50-100 lines each = ~300-600 additional. Range 600-900 honest.

If drafting comes in materially over (>1100), surface the overshoot honestly with compression options before commit (same discipline Phase 5b retrospective drafting used). If drafting comes in materially under (<500), check whether content was skipped or whether the document is genuinely tighter than expected.

## Pre-flag for implementation

Single commit, one file added (`MOODLE-ACTIVITY-MOD-PHASES.md` at the lms-light-docs repo root). Commit body short:

```
Add MOODLE-ACTIVITY-MOD-PHASES.md

Activity-mod phase progression template extracted from mod_scorecard's
six-phase progression (Phase 1 skeleton → Phase 2 authoring → Phase 3
learner submission → Phase 4 reporting → Phase 5a gradebook+completion
→ Phase 5b privacy+backup/restore). Reference shape for future activity-
mod plugin work. Calibration anchors drawn from mod_scorecard's actual
round-trip counts.

Sections: purpose+audience; phase progression overview with reference
table; phase-by-phase detail (scope, dependencies, sub-step decomposition,
calibration anchors, Type B surfaces, gate-discipline shapes, banked
reflexes, archaeology pointers); sub-step shape; calibration anchors
reference data; when the template doesn't apply; closing.

Companion document METHODOLOGY.md follows in separate session.
```

After commit lands locally, surface the manual push command for John.

## Awaiting

No design questions assuming pre-flight surfaces lms-light-docs in expected state (CONTEXT.md + LESSONS.md present; no pre-existing MOODLE-ACTIVITY-MOD-PHASES.md; standard repo structure). If pre-flight surfaces material divergence (existing draft of this document; structural conflict with existing files; CONTEXT.md or LESSONS.md already covers phase progression at substantive depth), surface as an architectural Q before drafting.

If during drafting any section feels like it needs an architectural disposition (e.g., specific calibration anchor numbers feel too aggressive/conservative; phase scope description for a specific phase resists generalization), surface as a small Q at that point. Otherwise straight-through to gate.

After this document lands and pushes, the next deliverable is METHODOLOGY.md in a separate fresh session — referencing this document's phase progression as a known concept rather than inlining.
