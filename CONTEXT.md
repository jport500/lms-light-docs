# LMS Light — project context

This document describes the LMS Light product, infrastructure, and
conventions. Read it before starting any plugin-specific work for
LMS Light. It's stable across projects and evolves only when
infrastructure or conventions change.

For portable engineering lessons and failure modes accumulated across
plugin work, see LESSONS.md in this repo.

---

## Status note — workflow consolidation in progress

LMS Light is currently consolidating plugin management into a single
GitHub organization (`LMSLight`) with a PR-based gatekeeping workflow.
During this transition:

* Some plugins originated under Claude SAD with full LMS Light
  documentation conventions; others originated pre-SAD (Replit) and
  carry no project documentation
* Pre-SAD plugins are pending technical audit before they should be
  trusted as canonical reference material
* The plugin inventory below records each plugin's location, origin,
  and audit status — treat this as the source of truth, not feature
  descriptions or version claims found elsewhere

A separate ONBOARDING.md (forthcoming) will document the PR-based
workflow, reviewer responsibilities, and per-plugin audit status
once Phase 1 governance work is complete.

---

## Product

LMS Light (lmslight.io) is a managed Moodle service. We deliver
standard Moodle deployments, curated and extended with custom
plugins, to customers who want LMS functionality without managing
the stack themselves. Our value is in the operational layer:
deployment, multi-tenancy infrastructure, customer-specific
branding, and custom plugins tailored to our ICPs.

### Target customers

* **B2B SaaS companies** that need training platforms for their
  customers or partners (customer education, partner certification,
  product training)
* **Niche training operators** — specialized training businesses,
  especially those requiring certification and continuing-education
  (CE) hour tracking
* **Associations** — professional and membership bodies delivering
  member education and certification, often with CE hour
  requirements
* **Non-profits** — mission-driven organizations delivering training
  to staff, volunteers, or the communities they serve

Feature priorities flow from these ICPs. A feature that only
benefits large universities or K-12 schools is probably not aligned
with LMS Light's focus. A feature that benefits certification
workflows, customer-facing training deployments, member or volunteer
education, or multi-tenant operator efficiency is probably in scope.

### Status

Two pre-launch customers, one B2B SaaS and one niche training
operator. Product is in active development.

---

## Deployment model

* **One Moodle instance per tenant.** Each customer gets their own
  Moodle deployment, not a shared installation.
* **Per-tenant privately credentialed database.** No database
  sharing between tenants.
* **Partitioned moodledata directories.** Tenant data is
  filesystem-isolated.
* **Shared code stack.** All tenants run the same Moodle version
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

* **Moodle 5.2.1** (Build: 20260608)
* **Deployment codebase:** sites are deployed from
  `github.com/mutms/mutms` (the MuTMS distribution: core Moodle +
  plugins assembled via git subtrees)
* **Docroot pattern:** `public/` subdirectory (Moodle's post-4.x
  pattern where code lives in `public/`)
* **Dev environment:** DDEV on macOS
* **Production domains:** `<subdomain>.lmslight.io` by default;
  custom vanity domains supported per customer
* **Per-customer branding:** via Moodle's standard theme
  configuration (logos, brand colors, fonts) — no per-tenant
  custom theme code, just theme config
* **AI providers:** all sites pre-configured with Groq as the AI
  provider (text generation only), exposed via Moodle's native AI
  Providers subsystem through the `aiprovider_groq` custom plugin.
  New AI-integrated plugins should use Moodle's AI Providers API
  rather than direct LLM integration.

---

## Custom plugin ecosystem

LMS Light's deployment includes custom plugins from two sources:
the MuTMS project (external open-source, GPL-3.0) and plugins
maintained at the LMSLight GitHub organization. New plugins should
be aware of what already exists to avoid duplication and compose
cleanly with the stack.

### Plugin audit status

Each LMS Light custom plugin is classified by origin and audit
status. This classification is load-bearing: it determines how much
trust to place in any documentation found in the plugin's repo,
and whether that repo's conventions match the LMS Light standard.

* **SAD-built** — developed under Claude supervised agentic
  development; carries DECISIONS.md / LESSONS.md / MANUAL_SMOKE.md
  per LMS Light convention; trustworthy as reference material
* **Pending audit** — pre-SAD (Replit-origin) or otherwise
  unaudited; documentation may be minimal or absent; behavior
  must be verified against code rather than docs

All plugins listed below are deployed in production to current
LMS Light customers regardless of audit status. Audit work is
underway as part of the workflow consolidation.

### MuTMS plugin set — external dependency

MuTMS (Multi-Tenant Management System, mutms.org,
github.com/mutms) is a GPL-3.0 plugin suite for Moodle that
provides multi-tenancy infrastructure and related features not
available in standard Moodle. We deploy and depend on it; we do
not maintain it. Distribution is at github.com/mutms/mutms (core
Moodle patches + plugins assembled via git subtrees).

Components in production at LMS Light:

* **tool\_mutenancy** — Multi-tenancy. Partitions a Moodle instance
  into isolated tenants. Foundational for the LMS Light deployment
  model.
* **tool\_muprog** — Programs. Structured learning paths,
  enrolments, progress tracking, program-level completion.
* **tool\_mucertify** — Certifications tied to program completion,
  with expiry and renewal cycles. Directly relevant to CE
  workflows for niche training operators.
* **tool\_mutrain** — Training credits. Budget allocation gating
  access to learning activities.
* **tool\_murelation** — Supervisors and teams. Learner-supervisor
  relationships for manager visibility into progress and
  compliance.
* **tool\_muhome** — Cohort- and tenant-specific dashboards and
  landing pages.
* **mod\_mubook** — Interactive book module (structured page-based
  content).
* **tool\_mupwned** — Blocks known breached passwords via
  HaveIBeenPwned k-Anonymity (passwords never leave Moodle).
* **tool\_musudo** — Sudo-style privilege escalation for admins.
* **tool\_muloginas** — Log-in-as via a new Incognito window,
  preserving admin session.

Plugins we build should compose with MuTMS where relevant. If a
new plugin is doing something in the adjacency of MuTMS
functionality (programs, certifications, training credits,
relationships, tenant scoping), check the MuTMS docs at
docs.mutms.org before designing to avoid duplication.

Additional MuTMS components may exist in the wider ecosystem
beyond the list above. When in doubt, check
github.com/orgs/mutms/repositories for the current inventory.

### LMS Light custom plugins

The following plugins are maintained by LMS Light. For each, only
the canonical repo, current version, origin, and audit status are
recorded here. Feature descriptions, configuration, and operator
guidance belong in each plugin's own README and live with the
plugin, where they can be kept current.

**auth\_magiclink** — Passwordless magic-link authentication.

* Repo: github.com/LMSLight/moodle-auth\_magiclink
* Current: v2.0 (May 2026)
* Origin: Replit
* Audit status: **pending** — security-critical (handles
  authentication); first scheduled audit target

**aiprovider\_groq** — Groq AI provider integration for Moodle's
native AI Providers subsystem.

* Repo: github.com/LMSLight/moodle-aiprovider\_groq
* Current: v0.9-beta (MATURITY_BETA)
* Origin: Replit
* Audit status: **pending** — security-sensitive (handles AI
  provider credentials); second scheduled audit target

**format\_pathway** — Custom course format providing a focused,
one-section-at-a-time layout with a custom progress-tracking
sidebar.

* Repo: github.com/LMSLight/moodle-format\_pathway
* Origin: Replit
* Audit status: **pending**

**block\_availablecourses** — Custom block surfacing available
courses to learners.

* Repo: github.com/LMSLight/moodle-block\_availablecourses
* Origin: Replit
* Audit status: **pending**

**local\_quizgenpro** — AI-powered question generator for Moodle
quiz activities. Uses Moodle's native AI Providers subsystem.

* Repo: github.com/LMSLight/moodle-local\_quizgenpro
* Current: v3.0.0-beta (MATURITY_BETA)
* Origin: SAD
* Audit status: SAD-built

**mod\_knowledgecheck** — Knowledge-check activity module (part of
the knowledgecheck cluster, alongside `filter_knowledgecheck` and
`tiny_knowledgecheck`).

* Repo: github.com/jport500/moodle-mod\_knowledgecheck *(pending
  migration to LMSLight org)*
* Origin: SAD
* Audit status: SAD-built; documentation completeness to be
  verified during migration

**filter\_knowledgecheck** — Filter component of the
knowledgecheck cluster.

* Repo: github.com/jport500/moodle-filter\_knowledgecheck *(pending
  migration to LMSLight org)*
* Origin: SAD
* Audit status: SAD-built; documentation completeness to be
  verified during migration

**tiny\_knowledgecheck** — TinyMCE editor plugin component of the
knowledgecheck cluster.

* Repo: github.com/jport500/moodle-tiny\_knowledgecheck *(pending
  migration to LMSLight org)*
* Origin: SAD
* Audit status: SAD-built; documentation completeness to be
  verified during migration

**mod\_scorecard** — Scorecard activity module.

* Repo: github.com/jport500/moodle-mod\_scorecard *(pending
  migration to LMSLight org)*
* Origin: SAD
* Audit status: SAD-built; documentation completeness to be
  verified during migration

**mod\_videoflow** — Video activity plugin supporting Vimeo,
YouTube, and locally-hosted video with tracking and completion
rules.

* Repo: github.com/jport500/moodle-mod\_videoflow *(pending
  migration to LMSLight org)*
* Origin: Replit
* Audit status: **pending**

### CE tracking

Continuing-education hour tracking for niche training operators
requiring certification CE workflows. Currently delivered via
`tool_mucertify` from MuTMS; no separate LMS Light plugin at this
time.

### Planned plugins

An **AI agent plugin** is planned, modeled roughly after
lmscloud.io/products/ai-agent. Purpose: in-Moodle AI assistance
for learners and instructors. Details TBD during v1.0 design.
Will integrate via Moodle's AI Providers subsystem consistent
with `local_quizgenpro` and `aiprovider_groq`.

---

## Repository conventions

* **One GitHub repo per plugin:** `LMSLight/moodle-<type>_<name>`
  (e.g., `moodle-auth_magiclink`, `moodle-format_pathway`)
* **MuTMS plugins** live in `github.com/mutms/*` and follow their
  own versioning; we don't maintain them
* **Main branch:** `main`
* **Tags:** annotated for releases (`v1.0`, `v3.3`)
* **Commit messages explain WHY, not just WHAT**
* **Each phase of work lands in its own commit** — clean history,
  reviewable chunks
* **Release commits** bump version.php and release string; tag
  after push
* **Plugin work lands via pull request** against `main`; direct
  push to `main` is reserved for emergency operator action. See
  ONBOARDING.md (forthcoming) for the PR-based workflow detail.

Note: some plugins are mid-migration from the `jport500` namespace
to the `LMSLight` org (see plugin inventory above). Until migration
completes, those plugins still live at jport500.

---

## Working process — supervised agentic development

All LMS Light plugin work is done in supervised agentic mode:

* Kickoff prompts explicitly reference the repo, its docs, and scope
* Phase boundaries with explicit human review before proceeding
* Decisions surfaced BEFORE implementation, not after
* Manual smoke testing via browser and real infrastructure before push
* Real-transport verification before tagging releases
* Per-phase commits with clear messages
* Clean git history over speed

The rhythm is: propose → pause for review → implement → pause for
review → commit → pause for review → push. Verification output
between each step. This isn't ceremony; it's where interpretation
mismatches get caught before they ship.

See LESSONS.md in this repo for concrete examples of what the
rhythm has caught in practice.

---

## Code quality gates (after every phase)

* **phpcs --standard=moodle clean plugin-wide** — zero errors,
  zero warnings. Moodle's opinionated code standard catches style
  issues, documentation gaps, and some real bug classes. Fix at
  every phase boundary; never accumulate debt.
* **PHPUnit full suite green** — all plugin tests pass, no
  regressions from the prior phase
* **CLI smoke (where present)** — scripted end-to-end exercise of
  the pipeline
* **Real-transport verification before release** — PHPUnit's
  in-memory mocking isn't sufficient for features with external
  I/O. Mailpit locally + real SMTP for email features; equivalent
  real-transport verification for webhooks, APIs, AI provider
  calls, etc.

---

## Filesystem and tooling

* Working directory: `/Users/John/projects/mutms/moodle/`
* Plugin path in container:
  `/var/www/html/moodle/public/<type>/<name>/`
* DDEV project: `mutms` (URL: `mutms.ddev.site:8443`)
* phpcs in container: `~/.composer/vendor/bin/phpcs` with
  `moodlehq/moodle-cs` installed
* Mailpit at `mutms.ddev.site:8026` (SMTP sink at localhost:1025)
* Real SMTP: Gmail app-password setup, configured in Moodle's
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

**Documentation-reality drift.** Project docs (including this one)
can describe planned, experimental, or aspirational work as if it
shipped. Before relying on a documentation claim about a plugin's
version, features, or APIs, verify against the plugin's actual
repo and code. The plugin inventory in this document records
canonical repo locations; treat those as ground truth.

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

* Store config in Moodle's standard config mechanisms (automatic
  per-tenant isolation via per-tenant DB)
* Use Moodle's filesystem abstractions rather than direct paths
  (automatic moodledata partitioning)
* Not embed single-tenant assumptions or cross-tenant references
* Be aware of MuTMS (`tool_mutenancy`) tenant context where
  functionality overlaps

**AI integration standard.** Plugins that use AI capability must
integrate via Moodle's native AI Providers subsystem, not via
direct LLM calls. This enables per-tenant provider configuration,
centralizes credential management, and keeps the plugin portable
across AI backends. LMS Light currently deploys with Groq as the
configured provider for text generation, via the `aiprovider_groq`
custom plugin.

---

## Documentation conventions

LMS Light's standard for plugin documentation is:

* `README.md` — what the plugin does, installation, configuration,
  operator guide, troubleshooting, roadmap
* `CHANGES.md` — release notes, one entry per version, factual
  tone
* `MANUAL_SMOKE.md` — if the plugin has user-facing behavior, a
  walkthrough operators can run to verify shipped behavior
* `docs/DECISIONS.md` — architectural decisions with rationale,
  alternatives considered, "would revisit if" triggers
* `docs/LESSONS.md` — process lessons and portable failure modes
  learned during development (can be deferred to v1.1 for
  brand-new plugins)
* `docs/SPEC.md` — if applicable, point-in-time design document

SAD-built plugins establish these during v1.0 development.
Pending-audit (Replit-origin) plugins may not yet have the full
set; bringing them up to standard is part of the audit workflow.
Documents that live in the repo persist; those that live only in
conversation don't.

---

## Reference material

Read these before starting any new plugin work:

1. **This repo's LESSONS.md** — portable failure modes and
   recovery patterns applicable to all LMS Light plugin work
2. **This document's plugin inventory** — for canonical repo
   locations and audit status. Do not assume version numbers or
   feature claims from prior conversations or older versions of
   this document; verify against the plugin's actual repo.

Once audits complete, audited plugins will have `docs/DECISIONS.md`
and `docs/LESSONS.md` that serve as reference material for future
work on those plugins. Pending-audit plugins should not be treated
as reference material until their audit is complete.

---

## Using this context

When kicking off a new plugin conversation:

1. Start the kickoff prompt with: "Read
   `github.com/LMSLight/lms-light-docs/blob/main/CONTEXT.md` and
   the LESSONS.md in the same repo before we begin."
2. Follow with the plugin-specific section describing scope,
   motivation, and constraints
3. Claude Code orients on this context, reads referenced docs,
   and surfaces design decisions before implementation

This context document is stable. When LMS Light's infrastructure,
plugin ecosystem, or conventions change, update this document
once; every future plugin kickoff picks up the change
automatically.
