# LMS Light WordPress site — project context

This document describes the LMS Light WordPress marketing and
commerce site, its infrastructure, and conventions. Read it before
starting any WP-side work for LMS Light. It's the WordPress-side
analog to `CONTEXT.md` (Moodle) in this repo. Both documents
describe the same product from different angles.

For the strategy that drives the funnel and copy on the WP site,
see `lmslight-strategy-shift.md` in this repo. CONTEXT-WP.md
describes engineering reality; the strategy doc describes
commercial intent.

For findings — security, operational, or strategy-shift cleanup
items — that need decisions or work, see `FINDINGS-WP.md`.

---

## Product

The lmslight.io WordPress site is the marketing front door,
commerce surface, and customer-provisioning bridge for LMS Light.
It does three jobs:

1. **Marketing.** Public pages, blog, FAQ, lead capture. Communicates
   what LMS Light is, who it's for, what it costs.
2. **Commerce.** WooCommerce + WC Subscriptions runs the catalog,
   checkout, and recurring billing. Three subscription tiers
   (Core $100 / Grow $200 / Pro $300 per month) plus a $100
   simple Consultation product.
3. **Provisioning bridge to Moodle.** When a customer subscribes,
   the WP site triggers the orchestration that creates their
   Moodle tenant, monitors deployment, and reacts to subscription
   lifecycle events (plan switches, payment failures, cancellations)
   by calling Rundeck and an internal core API to provision,
   pause, or destroy tenant infrastructure.

The third job is the most consequential and the most LMS-Light-
specific. It's also the source of the single most consequential
structural fact about the codebase — see "Bridge architecture"
below.

### Status

Two pre-launch customers, same as the Moodle side. Site is in
active development, mid-strategy-shift (see
`lmslight-strategy-shift.md`). Several sections of the live site
still reflect the previous trial-first funnel and are being
updated section by section.

---

## Funnel and commercial model

The funnel is being rebuilt around a consultative two-track model:

1. **Build a plan** (AI Plan Builder, ~10 minutes, downloadable output)
2. **Explore the demo site** (`demo.lmslight.io`, role-based one-click logins)
3. **Discovery call** (30-minute conversation, tier + consulting recommendation)
4. **Subscribe and provision** (WooCommerce checkout, automated tenant deployment)

Self-serve subscription remains available — buyers who know what
they want can subscribe directly via the pricing page. Trials
are being removed entirely.

Commercial model:

- **Subscriptions:** $100 / $200 / $300 per month, no annual
  contract. Tiers differ on concurrent user count, SSO, and
  API access. Modeled in WooCommerce as a single variable
  subscription product (ID 476) with three variations.
- **Consultation product** ($100, simple WC product, ID 1134)
  exists today. Productized consulting add-ons are planned but
  packaging is undecided.
- **Demo site** (`demo.lmslight.io`) is a tenant of the production
  Moodle multi-tenant infrastructure with curated seed data and
  three role-based logins (admin, manager, learner). Surfaced
  on the WP site as the "Explore a live demo" path.

For the full strategy and the rationale behind each step, see
`lmslight-strategy-shift.md`.

---

## Hosting and environments

* **Production and staging:** EC2 instances on an LMS-Light-owned
  AWS account
* **Local development:** DDEV on macOS (matches the Moodle dev
  environment convention). DDEV project name `wordpress`,
  reachable at `https://wordpress.ddev.site:8443`. Uses OrbStack
  as the Docker provider.
* **Deploy pipeline (current):** GitLab CI hosted by SmartAppTech
  (the development partner who originally built the site). CI
  SSHes to EC2, runs `git checkout` over the docroot, removes
  `.git`, then runs `npm install && npm run build` inside the
  active theme.
* **Deploy pipeline (planned):** Migrating to LMS-Light-owned
  GitHub (`lmslight/wordpress`) with a deploy pipeline TBD. See
  "Repository conventions" below.

The mismatch between current and planned pipelines is the
foundational migration work for the WP side. Until it's done,
the WP site's source of truth is SmartAppTech's GitLab, not a
repo we control.

---

## WordPress platform

* **WordPress 6.9.1**
* **PHP 8.4**
* **MariaDB 11.8** (production runs MySQL 8.4 on RDS)
* **Single-site** — not multisite
* **AWS RDS** for production database
* **WP_DEBUG_LOG enabled** in the local replica; production
  state TBD
* **DISABLE_WP_CRON = true** — WP-Cron is driven by an external
  scheduler in production. Locally this means `lms_site_*` cron
  hooks (see "Bridge architecture") don't run unless invoked
  manually with `ddev exec wp cron event run --all`.

### Theme

The active theme is **`lmslight`** — a custom hybrid theme built
by SmartAppTech on their `vite-tailwind-boilerplate` starter.
"Hybrid" because it combines:

- Block-theme features (`theme.json`, `templates/`, `parts/`,
  patterns)
- Classic procedural PHP (`functions.php`, an `inc/` directory of
  40+ includes loaded via glob from `inc/inc.php`)
- 24 custom Gutenberg / ACF blocks under `blocks/`
- 12 WooCommerce template overrides under `woocommerce/`
- Vite 4.5 + Tailwind 3.3 build pipeline (`npm run build` outputs
  to `dist/`)

The theme is not just presentation — it contains the entire
provisioning bridge to Moodle (see below). This is the most
important fact about the codebase.

### Bridge architecture — provisioning, switching, lifecycle

When a customer subscribes, switches plans, fails payment, or
cancels, code in the active theme reacts. There is no plugin or
mu-plugin handling this — it lives entirely in
`wp-content/themes/lmslight/inc/inc.moodle-*.php`.

**Trigger surface.** The bridge listens to:

- `woocommerce_order_status_changed` (provisioning trigger on
  status → processing)
- `woocommerce_subscriptions_switch_completed` (sets a
  plan-switch flag)
- A custom WC Blocks checkout field `lms/subdomain` (where the
  customer chooses their tenant subdomain at checkout)
- Five custom cron hooks running on a `x_minutes` schedule
  (interval set by `LMS_INTERVAL_DOMAIN_CHECK_MIN`):
  - `check_lms_site_deployments` — polls for completed deployments
  - `process_plan_switches` — handles plan upgrades/downgrades
  - `lms_site_failed_payments` — pauses tenants on payment failure
  - `lms_site_unfailed_payments` — unpauses on payment recovery
  - `lms_site_cancel_subscription` — destroys tenants on cancel

**Outbound calls** (all via PHP, no WC webhooks):

- **Rundeck** at `http://rundeck.lmslight.io:4440/api/47/job/<uuid>/run`
  — three jobs (create, change, delete instance). Auth via
  `LMS_LIGHT_TOKEN` constant in `wp-config.php`.
- **Internal core API** at `http://api.core.lmslight.io/site/<domain>/...`
  — endpoints for `/details`, `/disabled`, others. Used to
  check subdomain availability, pause/unpause tenants, fetch
  deployment tokens.
- **Per-tenant Moodle webservice** at
  `https://<subdomain>.lmslight.io/webservice/rest/server.php`
  — `auth_userkey_request_login_url` for SSO passthrough. Token
  per tenant stored in the `lms_deployment_token` ACF field on
  the subscription.

**ACF fields on `shop_subscription` posts** that the bridge writes
and reads: `lms_deployment_token`, `lms_subdomain`,
`lms_site_deployed`, `lms_deployment_email_sent`,
`lms_deployment_email_sent_date`, `lms_need_plan_type_switch`.
Plus `_wc_other/lms/subdomain` post meta from the checkout field.

**Important consequences:**

- Switching themes silently breaks customer provisioning, plan
  switches, payment-failure handling, cancellation, and SSO
  passthrough. The theme is load-bearing infrastructure.
- The bridge does not depend on WC webhooks. Stripe webhooks
  matter for payment events reaching WC; everything beyond that
  is internal PHP action hooks and cron.
- All bridge HTTP traffic is plaintext. The Rundeck token rides
  cleartext over the network. See `FINDINGS-WP.md`.

The long-term placement of the bridge — extract to plugin or
mu-plugin, leave in theme, or rebuild as a service outside WP —
is undecided.

---

## Plugin ecosystem

22 active plugins on production. Categorized:

### Commerce

* **WooCommerce** (10.7.0) — catalog, checkout, orders
* **WooCommerce Subscriptions** (8.4.0) — recurring billing,
  plan switching, payment retry. Update available; routine but
  load-bearing for the bridge.
* **WooCommerce Stripe Gateway** (10.6.1) — primary payment
  processor (test mode locally)
* **WooCommerce Payments** (10.7.1) — Apple Pay / Google Pay
  surfaces only

### Forms and lead capture

* **Contact Form 7** (6.1.5) — one form ("LMS Form", ID 459)
* **HubSpot Leadin** (11.3.45) — lead capture, contact tracking,
  and the live chat widget that appears bottom-right on
  marketing pages

### SEO and analytics

* **Yoast SEO** (27.5)
* **Google Site Kit** (1.177.0)

### Performance

* **W3 Total Cache** (2.9.4) — owns the "Performance" admin bar
  item. Free tier (`license_key = no_key`). `advanced-cache.php`
  drop-in present. `WP_CACHE = false` locally.

### Security and admin

* **Magic Login Pro** (2.6.2) — passwordless login for admin
  users (separate from `auth_magiclink` on the Moodle side)
* **User Switching** (1.11.2) — admin "log in as" tool

### Custom — `lmslight`-authored

* **lmslight-ai-interview** — the AI Plan Builder. Captures a
  conversational interview with site visitors via Anthropic's
  API and produces a custom implementation plan. Conversations
  are stored for review in wp-admin. Strategic surface — see
  `lmslight-strategy-shift.md`. Reads `LMSLIGHT_ANTHROPIC_API_KEY`
  and `LMSLIGHT_USE_MOCK` from `wp-config.php`. Repo currently
  at `jport500/lmslight-ai-interview`; planned to move to
  `lmslight/lmslight-ai-interview` as part of org consolidation.
  Not yet installed on production or staging — install on local
  replica is imminent.

### Other

* **All-in-One WP Migration** (7.105) — backup tool
* **Pretty Link** (3.6.21) — link shortening (no published
  pretty-links currently)
* **Redirection** (5.7.5) — URL redirects
* **WP Crontrol** (1.21.0) — cron debugging UI
* **WP Mail SMTP** (4.8.0) — outbound email
* **Akismet** (5.7) — inactive
* **Schema and Structured Data for WP** (1.59) — inactive
* **Hello Dolly** (1.7.2) — inactive
* **Tawk.to Live Chat** (0.9.3) — active but unconfigured. The
  actual chat widget is HubSpot Leadin. See `FINDINGS-WP.md`.

### mu-plugins

"mu-plugins" (must-use plugins) are PHP files in
`wp-content/mu-plugins/` that WordPress loads on every request,
before regular plugins. They're always active, can't be
deactivated through wp-admin, and don't appear in the normal
Plugins list. They're the conventional home for infrastructure
code that should never be accidentally turned off — orchestration
glue, custom constants, security hooks, anything where silent
deactivation would break the system. Tradeoffs: no update
mechanism through wp-admin, no activation lifecycle hooks
(`register_activation_hook` doesn't fire), and lower visibility
to operators doing routine maintenance.

None currently exist for LMS Light. `wp-content/mu-plugins/` does
not exist. If we want to land orchestration glue or infra-level
code in a place that can never be deactivated, this directory is
the natural home; nothing currently lives there. The Moodle
provisioning bridge is the obvious candidate if/when we extract
it from the theme — see "Bridge architecture" above.

---

## Repository conventions

The WP-side repo conventions are **in transition** and not yet
mature. What follows describes the current state and the
direction.

### Source of truth (current vs target)

* **Current:** `gitlab.com/SmartAppTech/lms-light` — private
  repo owned by SmartAppTech, the original development partner.
  Used by GitLab CI to deploy to EC2.
* **Target:** `github.com/lmslight/wordpress` — owned by the
  LMS Light GitHub org. Migration is a pre-launch priority.
  Until done, code we write locally cannot be cleanly deployed
  through the existing pipeline without going through SmartAppTech.

### Custom plugin repos

* **One repo per custom plugin** in the `lmslight` GitHub org:
  `lmslight/lmslight-<name>` (e.g. `lmslight/lmslight-ai-interview`)
* Note: this differs from the Moodle convention
  (`jport500/moodle-<type>_<name>`) by both org and naming
  scheme. Different platform, different conventions; that's
  fine.
* Existing custom plugins under `jport500` (currently
  `jport500/lmslight-ai-interview`) will be transferred to
  the `lmslight` org as part of the migration.

### Plugin update workflow

**Undecided.** The standard tradeoff for WordPress is between:

- Tracking everything in Git and deploying all updates through
  CI (engineering rigor, but routine vendor updates become
  pull requests)
- Letting wp-admin manage vendor plugin updates and tracking
  only custom code in Git (operator convenience, but production
  filesystem becomes the source of truth)
- A hybrid where vendor plugins update via wp-admin while
  custom plugins go through Git + CI, with a manifest tracking
  vendor versions

A decision will be made before the source-of-truth migration
completes. Until then, vendor plugins are updated via wp-admin
on production and the local replica reflects whatever the
production tarball contained at restore time.

### SmartAppTech relationship

SmartAppTech built the `lmslight` theme and historically owned
the WP codebase. They remain on a $400/month support retainer
— responsive but no longer actively building. Time-zone offset
(Eastern Europe). The trajectory of the relationship is under
review, particularly in light of supervised-agentic development
capabilities. Either way, all new feature work is moving to
supervised-agentic development under LMS Light's own GitHub org;
SmartAppTech's role is bounded to the existing theme codebase
and any specific support items raised against it.

---

## Working process — supervised agentic development

Same rhythm as the Moodle side: kickoff prompts reference repo +
docs + scope, phase boundaries with explicit human review,
decisions surfaced before implementation, manual smoke testing,
real-transport verification before push, per-phase commits, clean
git history.

The WP side has a precedent — `lmslight-ai-interview` was built
this way using paired Opus 4.7 agents (one in chat for design and
review, one in Claude Code for implementation), the same pattern
this and future WP plugin sessions will use.

The supervised-agentic discipline can be applied to any new
custom plugin work. It cannot yet be applied cleanly to theme
work or any cross-cutting WP-site changes, because the
source-of-truth repo is still owned by SmartAppTech. Establishing
`lmslight/wordpress` is what unlocks supervised-agentic work
across the whole site.

---

## Code quality gates

**Currently absent.** The WP side has no phpcs config, no PHPStan,
no Psalm, no PHPUnit suite, no ESLint, and no equivalent of the
Moodle "phpcs clean + PHPUnit green at every phase" gate. This
is greenfield from a quality-gate standpoint.

For new custom plugins (`lmslight-ai-interview` and successors),
the working assumption is that we adopt the same kind of
discipline as the Moodle plugins, scaled appropriately for the WP
side:

- WordPress Coding Standards via phpcs (`wp-coding-standards/wpcs`)
- PHPUnit where the plugin has logic worth testing
- Real-transport verification before tagging releases
  (the Moodle-side rule generalizes — any feature that talks to
  external systems gets verified end-to-end through the real
  transport, not just through unit-test mocks)

Theme-level adoption of phpcs is a separate question that
intersects with the SmartAppTech relationship and the source-of-
truth migration. Not in scope until those are resolved.

---

## Filesystem and tooling

* **Working directory (local):** `/Users/John/projects/wordpress/`
* **Container path:** `/var/www/html/`
* **DDEV project:** `wordpress` (`https://wordpress.ddev.site:8443`)
* **Mailpit:** `https://wordpress.ddev.site:8026`
* **WP-CLI:** `ddev exec wp ...` from anywhere in the project
* **Theme path:** `wp-content/themes/lmslight/`
* **Custom plugins path:** `wp-content/plugins/lmslight-*/`
  (when present)
* **mu-plugins path:** `wp-content/mu-plugins/` (currently
  absent; create when needed)

---

## Cross-cutting engineering rules

The same general rules from the Moodle `CONTEXT.md` apply on the
WP side. A few points where the WP side requires specific
emphasis:

**Credential handling.** Same redaction rule as the Moodle side —
never echo values for keys matching `*pass*`, `*password*`,
`*secret*`, `*token*`, `*key*`, `*credential*`, `*apikey*`. On
the WP side this includes the bridge constants in `wp-config.php`
(`LMS_LIGHT_TOKEN`, `LMSLIGHT_ANTHROPIC_API_KEY`), Stripe API
keys in WC settings, the hardcoded encryption key in
`MoodleAjaxHandler::get_login_link_lms_callback` (see
`FINDINGS-WP.md`), and any per-tenant Moodle webservice tokens.

**File-memory drift.** Same as Moodle. Re-read before relying.
Especially relevant for `wp-config.php`, theme `inc/` files (the
glob loader makes "what's actually included" non-obvious), and
ACF field group state (DB and `acf-json/` can disagree).

**Spec verification.** Before designing against a WP, WC, or
WC Subscriptions API — hooks, REST endpoints, capability names,
class structure — verify against the actual codebase rather
than against memory of how WP/WC typically work. WC versions in
particular have ongoing API changes between minors.

**Transport testing.** Email goes through Mailpit locally and
real SMTP before release. Bridge calls to Rundeck and the core
API need real-transport verification before any change ships,
not just unit-test mocks. Stripe webhooks get tested through
Stripe's CLI / test events, not just through mocked payloads.

**Multi-tenant awareness.** The WP site is single-tenant — there
is one lmslight.io site. But the data it manages (subscriptions,
ACF fields on subscriptions, the Moodle bridge state) is the
control plane for many Moodle tenants. A bug in WP-side code can
affect every customer's Moodle environment. Treat the bridge as
production-critical infrastructure regardless of how marketing-
flavored its surroundings are.

**AI integration.** Custom plugins on the WP side that use AI
capability (currently `lmslight-ai-interview`, future plugins
likely) call provider APIs directly — there's no equivalent of
Moodle's AI Providers subsystem to abstract this. API keys live
in `wp-config.php` constants. Mock-mode flags
(`LMSLIGHT_USE_MOCK` for the AI Interview plugin) should be the
norm so plugins can be developed and tested without burning real
API quota.

**Strategy-shift discipline.** Until the trial-first → consultative
funnel migration is complete, every change to copy, CTAs, or
funnel UX should be checked against `lmslight-strategy-shift.md`.
Trial-era language and CTAs are being removed, not preserved.

---

## Documentation conventions

For each custom plugin in the `lmslight` org, the same document
set as the Moodle plugins:

* `README.md` — what it does, install, configure, troubleshoot,
  roadmap
* `CHANGES.md` — release notes
* `MANUAL_SMOKE.md` — operator walkthrough where relevant
* `docs/DECISIONS.md` — architectural decisions with rationale
* `docs/LESSONS.md` — process lessons (can defer to v1.1 for
  brand-new plugins)
* `docs/SPEC.md` — point-in-time design document where
  applicable

Site-wide documents (this CONTEXT-WP.md, FINDINGS-WP.md, the
strategy doc, eventually a LESSONS-WP.md) live in `lms-light-docs`
alongside the Moodle equivalents.

---

## Reference material

Read these before starting WP-side work:

1. **This document** — current state and conventions
2. **`FINDINGS-WP.md`** in this repo — known issues, security
   items, strategy-shift cleanup tasks
3. **`lmslight-strategy-shift.md`** in this repo — the funnel
   strategy and copy direction
4. **`CONTEXT.md`** in this repo (Moodle side) — the other half
   of the platform, especially the bridge endpoints the WP side
   calls into
5. **`LESSONS.md`** in this repo — portable failure modes from
   the Moodle plugin work; many generalize to WP
6. **`github.com/jport500/lmslight-ai-interview`** (will move to
   `lmslight/lmslight-ai-interview`) — the existing custom
   plugin and a reference for what supervised-agentic-built
   WP plugins look like

---

## Using this context

When kicking off a new WP-side conversation:

1. Start the kickoff prompt with: "Read
   `github.com/jport500/lms-light-docs/blob/main/CONTEXT-WP.md`
   and `FINDINGS-WP.md` in the same repo before we begin."
2. Follow with the plugin- or task-specific scope, motivation,
   and constraints
3. Claude Code orients on this context, reads referenced docs,
   surfaces design decisions before implementation

This document evolves when WP infrastructure, plugin ecosystem,
or conventions change. The strategy shift is in flight; expect
this document to update as that work lands.
