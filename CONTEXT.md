# LMS Light — project context

This document describes the LMS Light product, infrastructure, and 
conventions. Read it before starting any plugin-specific work for 
LMS Light. It's stable across projects and evolves only when 
infrastructure or conventions change.

For portable engineering lessons and failure modes accumulated across 
plugin work, see LESSONS.md in this repo.

---

## Product

LMS Light (lmslight.io) is a managed Moodle service. We deliver 
standard Moodle deployments, curated and extended with custom 
plugins, to customers who want LMS functionality without managing 
the stack themselves. Our value is in the operational layer: 
deployment, multi-tenancy infrastructure, customer-specific 
branding, and custom plugins tailored to our ICPs.

### Target customers

- **B2B SaaS companies** that need training platforms for their 
  customers or partners (customer education, partner certification, 
  product training)
- **Niche training operators** — specialized training businesses, 
  especially those requiring certification and continuing-education 
  (CE) hour tracking

Feature priorities flow from these ICPs. A feature that only 
benefits large universities or K-12 schools is probably not aligned 
with LMS Light's focus. A feature that benefits certification 
workflows, customer-facing training deployments, or multi-tenant 
operator efficiency is probably in scope.

### Status

Two pre-launch customers, one B2B SaaS and one niche training 
operator. Product is in active development.

---

## Deployment model

- **One Moodle instance per tenant.** Each customer gets their own 
  Moodle deployment, not a shared installation.
- **Per-tenant privately credentialed database.** No database 
  sharing between tenants.
- **Partitioned moodledata directories.** Tenant data is 
  filesystem-isolated.
- **Shared code stack.** All tenants run the same Moodle version 
  and the same curated set of custom plugins. Code changes 
  propagate to all tenants via deployment; data stays isolated.

This isolation model matters for plugin design. A plugin storing 
config in `mdl_config_plugins` stores per-tenant config (each 
tenant has their own DB). A plugin caching data in moodledata 
caches per-tenant data. Cross-tenant data sharing would require 
explicit infrastructure that doesn't currently exist.

**Important: not Moodle Workplace.** Moodle Workplace includes a 
multitenancy tool as a commercial feature, but that's a different 
product. LMS Light deploys standard Moodle (moodle/moodle, not 
moodle/workplace) extended with the MuTMS plugin set for 
multi-tenancy.

---

## Moodle platform

- **Moodle 5.1** (plugins target `requires = 2024040100`)
- **Docroot pattern:** `public/` subdirectory (Moodle's post-4.x 
  pattern where code lives in `public/`)
- **Dev environment:** DDEV on macOS
- **Production domains:** `<subdomain>.lmslight.io` by default; 
  custom vanity domains supported per customer
- **Per-customer branding:** via Moodle's standard theme 
  configuration (logos, brand colors, fonts) — no per-tenant 
  custom theme code, just theme config
- **AI providers:** all sites pre-configured with Groq as the AI 
  provider (text generation only), exposed via Moodle's native AI 
  Providers subsystem. New AI-integrated plugins should use 
  Moodle's AI Providers API rather than direct LLM integration.

---

## Custom plugin ecosystem

LMS Light's deployment includes custom plugins from three sources: 
the MuTMS project (external open-source, GPL-3.0), plugins we 
maintain in the `jport500` GitHub namespace, and some that predate 
this convention. New plugins should be aware of what already 
exists to avoid duplication and compose cleanly with the stack.

### MuTMS plugin set — external dependency

MuTMS (Multi-Tenant Management System, mutms.org, 
github.com/mutms) is a GPL-3.0 plugin suite for Moodle that 
provides multi-tenancy infrastructure and related features not 
available in standard Moodle. We deploy and depend on it; we do 
not maintain it. Distribution is at github.com/mutms/mutms (core 
Moodle patches + plugins assembled via git subtrees).

Components in production at LMS Light:

- **tool_mutenancy** — Multi-tenancy. Partitions a Moodle instance 
  into isolated tenants. Foundational for the LMS Light deployment 
  model.
- **tool_muprog** — Programs. Structured learning paths, 
  enrolments, progress tracking, program-level completion.
- **tool_mucertify** — Certifications tied to program completion, 
  with expiry and renewal cycles. Directly relevant to CE 
  workflows for niche training operators.
- **tool_mutrain** — Training credits. Budget allocation gating 
  access to learning activities.
- **tool_murelation** — Supervisors and teams. Learner-supervisor 
  relationships for manager visibility into progress and 
  compliance.
- **tool_muhome** — Cohort- and tenant-specific dashboards and 
  landing pages.
- **mod_mubook** — Interactive book module (structured page-based 
  content).
- **tool_mupwned** — Blocks known breached passwords via 
  HaveIBeenPwned k-Anonymity (passwords never leave Moodle).
- **tool_musudo** — Sudo-style privilege escalation for admins.
- **tool_muloginas** — Log-in-as via a new Incognito window, 
  preserving admin session.

Plugins we build should compose with MuTMS where relevant. If a 
new plugin is doing something in the adjacency of MuTMS 
functionality (programs, certifications, training credits, 
relationships, tenant scoping), check the MuTMS docs at 
docs.mutms.org before designing to avoid duplication.

Additional MuTMS components may exist in the wider ecosystem 
beyond the list above. When in doubt, check 
github.com/orgs/mutms/repositories for the current inventory.

### auth_magiclink — LMS Light custom

Passwordless magic-link authentication.

- Repo: github.com/jport500/moodle-auth_magiclink
- Current: v3.3 (April 2026)
- Features: configurable auth-method allowlist, hardcoded admin 
  exclusion (users with `moodle/site:config` never eligible), 
  uniform-response email-enumeration defense, rate limiting, 
  comprehensive audit logging
- Public API: `auth_magiclink\api::is_auth_allowed($user)`, 
  `auth_magiclink\api::is_admin_user($user)` and others. External 
  plugins can generate login URLs using this API.
- Architectural decisions: see `docs/DECISIONS.md` in that repo

### local_welcomeemail — LMS Light custom

Observer-based welcome email for new users with per-role, 
per-cohort, per-auth-method assignment.

- Repo: github.com/jport500/moodle-local_welcomeemail
- Current: v1.0 (April 2026); v1.1 in planning with 
  auth-agnostic theme
- Features: template system with strict placeholder allowlist, 
  four assignment dimensions (default/role/cohort/auth_method), 
  two-step test-send with preview, comprehensive privacy provider
- Current dependency: auth_magiclink v3.2+ (to be removed in 
  v1.1)
- Architectural decisions: see `docs/DECISIONS.md` in that repo
- Process lessons: see `docs/LESSONS.md` in that repo

### format_pathway — LMS Light custom

Custom course format replacing Moodle's default "scroll of 
death" with a focused, one-section-at-a-time layout with a 
custom progress-tracking sidebar.

- Repo: github.com/jport500/moodle-format_pathway
- Design philosophy: each course is a linear learning journey, 
  not a document to scroll through. Optimized for structured 
  training (onboarding, compliance, certifications, professional 
  development) — directly aligned with LMS Light's ICPs.
- Features: custom sidebar replaces core course index (reclaims 
  ~300px), `COURSE_DISPLAY_MULTIPAGE` default, progress bars via 
  completion API, collapsible sidebar, previous/next navigation, 
  mobile-responsive stacking
- Theme integration via CSS custom properties (`--pathway-*`) so 
  per-tenant themes can override brand colors without plugin 
  modification
- Supports Moodle 5.0+ in both pre-`/public` and post-`/public` 
  directory layouts

### local_quizgenpro — LMS Light custom

AI-powered question generator for Moodle quiz activities.

- Uses Moodle's native AI Providers subsystem (not direct LLM 
  integration)
- Current: v3.0.0 beta, version `2026012301`, requires Moodle 
  `2024042200`
- Maturity: `MATURITY_BETA`
- All LMS Light sites configured with Groq as the AI provider for 
  text generation

### mod_scorecard — LMS Light custom

Scorecard activity module — operators define numeric-scale items 
and qualitative bands; learners answer items, system scores 
responses against bands to produce qualitative feedback ("strong 
fit", "developing", etc.).

- Repo: github.com/jport500/moodle-mod_scorecard
- Current: v0.7.0 (April 2026); MATURITY_ALPHA pending 
  earned-by-production-usage signal
- Features: rich-text item prompts; soft-delete preservation for 
  historical attempts; configurable scale + per-item anchor 
  overrides; band-based qualitative scoring with snapshot 
  fidelity (results stay stable across band edits); Moodle 
  gradebook + activity-completion integration; full backup + 
  restore (items, bands, attempts, responses with snapshot 
  preservation); privacy provider; JSON template export and 
  populate-existing import for distributing scorecard structure 
  across courses and instances
- ICP alignment: directly serves certification-readiness 
  inventories, reflective self-assessment, and CE-relevant 
  qualitative scoring workflows for niche training operators — 
  the second named LMS Light ICP
- Documentation: README.md, CHANGES.md, USER-GUIDE.md 
  (operator/instructor reference at docs/USER-GUIDE.md), four 
  phase retrospectives (Phases 4, 5a, 5b, 6) plus a renderer 
  refactor retrospective in docs/, plus 
  docs/MOODLE-TEMPLATING-CONVENTIONS.md (developer-facing 
  conventions banked from refactor experience)
- Architectural decisions: see SPEC.md (point-in-time design at 
  v0.5) and the phase retrospectives in docs/

### CE tracking

Continuing-education hour tracking for niche training operators 
requiring certification CE workflows. Currently delivered via 
`tool_mucertify` from MuTMS; no separate jport500 plugin at this 
time.

### Planned plugins

An **AI agent plugin** is planned, modeled roughly after 
lmscloud.io/products/ai-agent. Purpose: in-Moodle AI assistance 
for learners and instructors. Details TBD during v1.0 design. 
Will integrate via Moodle's AI Providers subsystem consistent 
with local_quizgenpro.

---

## Repository conventions

- **One GitHub repo per plugin:** `jport500/moodle-<type>_<name>` 
  (e.g., `moodle-auth_magiclink`, `moodle-local_welcomeemail`, 
  `moodle-format_pathway`)
- **MuTMS plugins** live in `github.com/mutms/*` and follow their 
  own versioning; we don't maintain them
- **Main branch:** `main`
- **Tags:** annotated for releases (`v1.0`, `v3.3`)
- **Commit messages explain WHY, not just WHAT**
- **Each phase of work lands in its own commit** — clean history, 
  reviewable chunks
- **Release commits** bump version.php and release string; tag 
  after push

---

## Working process — supervised agentic development

All LMS Light plugin work is done in supervised agentic mode:

- Kickoff prompts explicitly reference the repo, its docs, and scope
- Phase boundaries with explicit human review before proceeding
- Decisions surfaced BEFORE implementation, not after
- Manual smoke testing via browser and real infrastructure before push
- Real-transport verification before tagging releases
- Per-phase commits with clear messages
- Clean git history over speed

The rhythm is: propose → pause for review → implement → pause for 
review → commit → pause for review → push. Verification output 
between each step. This isn't ceremony; it's where interpretation 
mismatches get caught before they ship.

See LESSONS.md in this repo for concrete examples of what the 
rhythm has caught in practice.

---

## Code quality gates (after every phase)

* **phpcs --standard=moodle clean plugin-wide** — zero errors,
  zero warnings. Moodle's opinionated code standard catches
  style issues, documentation gaps, and some real bug classes.
  Fix at every phase boundary; never accumulate debt.
* **PHPUnit full suite green** — all plugin tests pass, no
  regressions from the prior phase
* **SPEC matches code** — for plugins maintained under
  supervised agentic development, `docs/SPEC.md` reflects the
  behavior of HEAD. Any iteration that changes behavior updates
  the SPEC in the same commit. See "Living SPEC convention"
  under Documentation conventions.
* **CLI smoke (where present)** — scripted end-to-end exercise
  of the pipeline
* **Real-transport verification before release** — PHPUnit's
  in-memory mocking isn't sufficient for features with external
  I/O. Mailpit locally + real SMTP for email features;
  equivalent real-transport verification for webhooks, APIs, AI
  provider calls, etc.

---

## Filesystem and tooling

- Working directory: `/Users/John/projects/mutms/moodle/`
- Plugin path in container: 
  `/var/www/html/moodle/public/<type>/<name>/`
- DDEV project: `mutms` (URL: `mutms.ddev.site:8443`)
- phpcs in container: `~/.composer/vendor/bin/phpcs` with 
  `moodlehq/moodle-cs` installed
- Mailpit at `mutms.ddev.site:8026` (SMTP sink at localhost:1025)
- Real SMTP: Gmail app-password setup, configured in Moodle's 
  config when not rerouted to mailpit

---

## Cross-cutting engineering rules

These apply to all LMS Light code regardless of plugin type.

**Credential handling.** Never echo values for config keys matching 
`*pass*`, `*password*`, `*secret*`, `*token*`, `*key*`, 
`*credential*`, `*apikey*`. When introspecting these values, echo 
presence and length only. This rule was established after a 
credential exposure incident; the rule is permanent.

**File-memory drift.** If a file has been read earlier in a session 
or in a previous session, re-read before relying on it. Disk is 
authoritative, AI working memory is not. Spec files, config files, 
and docs especially benefit from explicit re-reads.

**Spec-verification.** Before designing against a Moodle framework 
API (events, capabilities, config keys, class names, MuTMS 
components), verify the API exists in the target Moodle version by 
grepping the actual codebase. Don't assume — check. The cost is 30 
seconds during design; the cost of discovering an assumption error 
mid-implementation is a full rescoping conversation.

**Transport testing.** For any feature that outputs to external 
systems (email, APIs, webhooks, AI services), end-to-end testing 
through the real transport is non-negotiable. Unit tests catch 
code-level bugs; local sinks (mailpit, ngrok, LocalStack, etc.) 
catch transport-level bugs; real-transport verification catches 
deliverability and integration bugs. All three levels, not optional.

**Severity calibration.** When a finding surfaces, the right 
question isn't "is this a bug" but "will this problem happen to 
someone who isn't me doing the exact thing I just did." Operator 
context is authoritative for severity calls. Flag findings 
honestly, but defer to John on severity when the context is 
outside your observation.

**Multi-tenant awareness.** Every plugin operates inside a 
per-tenant Moodle instance. Plugins should:

- Store config in Moodle's standard config mechanisms (automatic 
  per-tenant isolation via per-tenant DB)
- Use Moodle's filesystem abstractions rather than direct paths 
  (automatic moodledata partitioning)
- Not embed single-tenant assumptions or cross-tenant references
- Be aware of MuTMS (`tool_mutenancy`) tenant context where 
  functionality overlaps

**AI integration standard.** Plugins that use AI capability must 
integrate via Moodle's native AI Providers subsystem, not via 
direct LLM calls. This enables per-tenant provider configuration, 
centralizes credential management, and keeps the plugin portable 
across AI backends. LMS Light currently deploys with Groq as the 
configured provider for text generation.

---

## Documentation conventions

Every plugin has these documents:

* `README.md` — what the plugin does, installation,
  configuration, operator guide, troubleshooting, roadmap
* `CHANGES.md` — release notes, one entry per version, factual
  tone
* `MANUAL_SMOKE.md` — if the plugin has user-facing behavior, a
  walkthrough operators can run to verify shipped behavior
* `docs/DECISIONS.md` — architectural decisions with rationale,
  alternatives considered, "would revisit if" triggers
* `docs/LESSONS.md` — process lessons and portable failure modes
  learned during development (can be deferred to v1.1 for
  brand-new plugins)
* `docs/SPEC.md` — **living** spec describing the plugin's
  current behavior at HEAD. Mandatory for plugins maintained
  under supervised agentic development. See "Living SPEC
  convention" below.

New plugins should establish these during v1.0 development, not
as an afterthought. Documents that live in the repo persist;
those that live only in conversation don't.

---

## Reference material

Read these before starting any new plugin work:

1. **This repo's LESSONS.md** — portable failure modes and 
   recovery patterns applicable to all LMS Light plugin work
2. **github.com/jport500/moodle-local_welcomeemail/blob/main/docs/DECISIONS.md**
   — example of what good decision records look like for LMS 
   Light plugins
3. **github.com/jport500/moodle-local_welcomeemail/blob/main/docs/LESSONS.md**
   — plugin-specific field guide (many patterns generalize)
4. **github.com/jport500/moodle-auth_magiclink/blob/main/docs/DECISIONS.md**
   — version-numbering gotchas, install lifecycle archaeology, 
   admin-exclusion security reasoning

---

## Using this context

When kicking off a new plugin conversation:

1. Start the kickoff prompt with: "Read 
   `github.com/jport500/lms-light-docs/blob/main/CONTEXT.md` and 
   the LESSONS.md in the same repo before we begin."
2. Follow with the plugin-specific section describing scope, 
   motivation, and constraints
3. Claude Code orients on this context, reads referenced docs, 
   and surfaces design decisions before implementation

This context document is stable. When LMS Light's infrastructure, 
plugin ecosystem, or conventions change, update this document 
once; every future plugin kickoff picks up the change 
automatically.

### Living SPEC convention

For plugins maintained under supervised agentic development,
`docs/SPEC.md` is the authoritative reference for what the
plugin does. It is **living**, not point-in-time: every
iteration that changes plugin behavior updates `docs/SPEC.md`
in the same commit.

The living SPEC has three jobs:

1. **Tell an agent (or a human) what the plugin does today**,
   without requiring them to reconstruct intent from code.
2. **Anchor the propose-not-decide discipline.** The briefing
   instruction "amend the SPEC before contradicting it" only
   works if the SPEC lives in the same repo as the code, so
   amending it is part of the same commit as the change.
3. **Define acceptance criteria** that the agent can verify
   against without further design questions.

The convention has three documents at three altitudes for
plugins that originated from a point-in-time design doc:

| Document | Location | Job | Update cadence |
|---|---|---|---|
| Original design doc (e.g. `<plugin>-spec.md`) | `lms-light-docs` | Historical record of v1 intent | Frozen at v1; never updated |
| `docs/SPEC.md` | plugin repo | Current behavior of HEAD | Updated with every behavior-changing commit |
| `docs/DECISIONS.md` | plugin repo | Why each decision was made, with alternatives weighed | Appended to with each decision |

The original design doc has historical value: it shows what was
originally intended, what tradeoffs were made, what was
deferred. It belongs in `lms-light-docs` permanently as a
record. The living SPEC supersedes it as the day-to-day
reference once the plugin exists.

Plugins that didn't originate from a separate design doc still
maintain `docs/SPEC.md` as the living spec; the historical-record
row simply doesn't apply.


