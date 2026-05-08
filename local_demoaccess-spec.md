# local_demoaccess — Plugin SPEC v1

Point-in-time design document for `local_demoaccess`, the plugin
that powers one-click role-based login on the LMS Light demo site.

Companion to `demo-site-spec.md`. This document is the
authoritative reference for the plugin's behavior, safety model,
and acceptance criteria. Implementation may make judgment calls on
matters not specified here, but must not contradict anything
specified here without amending this SPEC first.

---

## Purpose

Expose a public, unauthenticated HTTP endpoint that mints a Moodle
session for a designated demo account when a visitor clicks a role
button on the demo site's front page. No forms, no credentials
displayed, no email entry, no signup.

The plugin is installed on every LMS Light tenant (shared code
stack) but is functionally inert on every tenant except the demo
tenant. Activation requires four independent guards to all pass.

---

## Scope

### In scope (v1)

- A public HTTP entry point that accepts a role parameter and
  logs the visitor in as the corresponding pre-existing demo
  account.
- A four-layer safety model that prevents the endpoint from
  functioning on any tenant other than the explicitly configured
  demo tenant.
- Admin settings page exposing the safety configuration and its
  current evaluation state.
- Per-IP rate limiting on the public endpoint.
- Audit logging of demo-access events.
- Role-specific post-login redirects.

### Explicitly out of scope (v1)

- Account provisioning. The three demo accounts
  (`demo_admin`, `demo_manager`, `demo_learner`) are seeded by
  the demo tenant's content snapshot, not by this plugin. The
  plugin reads from a configured allowlist of usernames; it does
  not create or manage them.
- Magic-link flow for real users. That is `auth_magiclink`'s job
  and remains untouched. This plugin does not extend, modify, or
  depend on `auth_magiclink`'s tables, settings, or code paths.
- Session management beyond standard Moodle session creation.
  Concurrent-visitor session collisions on shared accounts are an
  accepted v1 tradeoff, documented in `demo-site-spec.md`.
- Front-page UI. The role buttons live in the demo tenant's theme
  / front-page configuration. This plugin only provides the
  endpoint they POST/GET to.
- Per-tenant theming or branding.

---

## Safety model

The endpoint must refuse to function unless all four of the
following guards pass. Each guard is independently sufficient to
disable the plugin; defense in depth means a single
misconfiguration cannot expose customer tenants to anonymous
admin login.

### Layer 1 — Plugin enabled flag

A plugin-level admin setting `local_demoaccess/enabled`, default
`false`. Hardcoded default in `settings.php` so a fresh install
on any tenant is inert.

### Layer 2 — Wwwroot allowlist

A plugin-level admin setting `local_demoaccess/allowed_wwwroots`,
default empty. At runtime, the endpoint checks `$CFG->wwwroot`
against the allowlist. No match → refuse.

The setting accepts one wwwroot per line, each must be an exact
match (no wildcards, no path matching, no protocol coercion). The
v1 production allowlist contains exactly one entry:
`https://demo.lmslight.io`.

### Layer 3 — Account allowlist

A plugin-level admin setting
`local_demoaccess/allowed_usernames`, default empty. The endpoint
will only mint sessions for usernames in this list. v1 contents:

```
demo_admin
demo_manager
demo_learner
```

The endpoint accepts a `role` parameter (`admin` | `manager` |
`learner`), maps it to a username via plugin config (see
Configuration below), and verifies the resulting username is in
the allowlist before proceeding. A request with a `role` that
maps to a username not in the allowlist is refused.

### Layer 4 — Site admin refusal

Even if Layers 1–3 pass, the endpoint refuses to log in any user
that has `moodle/site:config` capability at the system context.
This catches the "operator accidentally promoted a demo account
to site admin" failure mode. The check uses Moodle's standard
capability API, not a username comparison.

### Developer-mode local override

For local development against Docker / DDEV where the wwwroot
won't match the production allowlist, Layer 2 may be bypassed
**only** when **both** of the following are true:

- `$CFG->debug` is set to `DEBUG_DEVELOPER` (32767) in
  `config.php`
- `$CFG->local_demoaccess_dev_override` is set to `true` in
  `config.php`

Both must be present in `config.php` directly. Neither is
configurable through any admin UI. Layers 1, 3, and 4 still
apply unconditionally — the dev override only relaxes the
wwwroot check, not any other guard.

This mechanism exists to support local build/test on Docker
without weakening the production safety posture. A production
`config.php` should never contain `local_demoaccess_dev_override`.
A pre-deployment check (operator runbook, not enforced by the
plugin) should grep for the flag in any production config.

### Failure mode

Any guard failure returns HTTP 404 with a generic Moodle
"page not found" response. No information about which guard
failed, whether the plugin is installed, or whether the role
parameter was valid. Logging is server-side only.

---

## HTTP interface

### Endpoint

```
GET /local/demoaccess/login.php?role={admin|manager|learner}
```

GET is acceptable here despite mutating session state because:
- The action is intentionally idempotent from the visitor's
  perspective (clicking the button always logs them in fresh).
- The buttons are anchor elements on the front page, not form
  submissions.
- CSRF is not a meaningful threat for an intentionally public
  unauthenticated endpoint that creates a session for a
  pre-known demo account.

### Parameters

| Parameter | Required | Values                          |
|-----------|----------|---------------------------------|
| `role`    | Yes      | `admin`, `manager`, `learner`   |

Any other value, missing parameter, or extra parameter → 404.

### Successful response

HTTP 302 redirect to the role-specific landing URL with a fresh
authenticated Moodle session set in cookies.

### Role-to-landing mapping

| Role      | Username        | Landing URL                                |
|-----------|-----------------|--------------------------------------------|
| `admin`   | `demo_admin`    | `/admin/index.php` (site admin dashboard)  |
| `manager` | `demo_manager`  | `/my/courses.php` *(see note below)*       |
| `learner` | `demo_learner`  | `/my/` (learner dashboard)                 |

Note on manager landing: the "manager dashboard" depends on
which `tool_muhome` / `tool_murelation` views are configured on
the demo tenant. Implementation should land the manager on
whichever URL the demo tenant's seed configuration designates as
the manager team-progress view. If the seed config is not yet in
place at implementation time, default to `/my/courses.php` and
note in the manual smoke doc that this needs revisiting once the
manager view is built.

### Rate limiting

Per-IP cap: **30 successful logins per IP per hour**. Excess
requests return HTTP 429 with `Retry-After` header. Rate-limit
state stored in Moodle cache (MUC), keyed by IP. Rate-limit
counter resets are handled by cache TTL, not by a separate
cleanup task.

Global cap: **600 successful logins per hour across all IPs**.
Excess returns 429. Same storage mechanism.

Both numbers are starting points and may be tuned via plugin
config (`local_demoaccess/rate_limit_per_ip`,
`local_demoaccess/rate_limit_global`) without code changes.

Failed attempts (guard failures, 404 responses) are not
counted against rate limits — rate limiting is for legitimate
demo traffic shaping, not security.

---

## Configuration

### Admin settings page

Located at *Site administration → Plugins → Local plugins →
Demo access*. Settings page must:

- Display a prominent warning at the top:

  > **Demo access exposes a public, unauthenticated login
  > endpoint.** This plugin is intended only for the LMS Light
  > demo tenant. Do not enable on customer tenants.

- Show current site wwwroot.
- Show current allowlist evaluation: ✅ in allowlist / ❌ not in
  allowlist (with the actual values displayed).
- Show whether developer override is active (and warn loudly if
  it is, with a reminder to remove it before production).
- Disable the "enabled" toggle if Layer 2 evaluation fails — the
  operator must add their wwwroot to the allowlist first, as a
  deliberate two-step.

### Settings keys

| Key                                       | Type     | Default |
|-------------------------------------------|----------|---------|
| `local_demoaccess/enabled`                | bool     | `false` |
| `local_demoaccess/allowed_wwwroots`       | textarea | empty   |
| `local_demoaccess/allowed_usernames`      | textarea | empty   |
| `local_demoaccess/role_admin_username`    | text     | `demo_admin`   |
| `local_demoaccess/role_manager_username`  | text     | `demo_manager` |
| `local_demoaccess/role_learner_username`  | text     | `demo_learner` |
| `local_demoaccess/role_admin_landing`     | text     | `/admin/index.php` |
| `local_demoaccess/role_manager_landing`   | text     | `/my/courses.php`  |
| `local_demoaccess/role_learner_landing`   | text     | `/my/` |
| `local_demoaccess/rate_limit_per_ip`      | int      | `30`    |
| `local_demoaccess/rate_limit_global`      | int      | `600`   |

Username and landing settings are configurable so the operator
can adjust without code changes, but the SPEC's
recommended-default values are the v1 expectation.

---

## Audit and logging

Every endpoint hit, regardless of outcome, generates a Moodle
log event:

- Successful login: standard event log entry with role, target
  username, IP, user agent.
- Guard failure: log entry with which layer failed and the IP.
  No information is ever returned to the client about which
  layer failed.
- Rate limit hit: log entry with IP and which limit (per-IP or
  global) tripped.

Events follow Moodle's standard event API
(`\core\event\base`-derived events under
`\local_demoaccess\event\`). Events are queryable through the
standard log report.

---

## File layout

Standard Moodle local plugin structure. Agent may add files as
needed; the following are minimum required:

```
local/demoaccess/
├── version.php
├── settings.php
├── login.php                    # public endpoint
├── lang/
│   └── en/
│       └── local_demoaccess.php
├── classes/
│   ├── guard.php                # four-layer evaluation
│   ├── session_minter.php       # session creation
│   ├── rate_limiter.php         # MUC-backed limiter
│   └── event/
│       ├── demo_login_succeeded.php
│       ├── demo_login_blocked.php
│       └── demo_rate_limited.php
├── tests/
│   ├── guard_test.php
│   ├── rate_limiter_test.php
│   └── login_endpoint_test.php
├── db/
│   └── caches.php               # MUC cache definition for rate limiter
├── README.md
├── CHANGES.md
├── MANUAL_SMOKE.md
└── docs/
    ├── DECISIONS.md
    └── LESSONS.md               # may start empty, populated as built
```

`version.php` should declare:
- `$plugin->component = 'local_demoaccess';`
- `$plugin->maturity = MATURITY_ALPHA;` for v1
- Moodle version requirements consistent with the rest of the
  LMS Light stack (Moodle 5.1)

---

## Implementation notes

### Guard evaluation

The four-layer evaluation should live in a single class
(`\local_demoaccess\guard`) with a single public method that
returns either an authorized username or a structured failure
reason. Failures are logged but not exposed to callers beyond
"refused." The endpoint code should not duplicate guard logic.

### Session creation

Use Moodle's standard authenticated-session mechanisms
(`complete_user_login()` after looking up the user record by
username). Do not bypass standard session setup, do not skip
`require_login()` initialization, do not construct sessions
manually.

### Rate limiter

MUC-backed counter with TTL. Keyed by `ip:{addr}` and `global`.
Increment on successful guard pass *before* session creation, so
rate limits cap real-world demo traffic rather than only counting
fully-completed logins.

### What this plugin must not do

- Must not call into `auth_magiclink` code, tables, or settings.
- Must not modify any other plugin's data.
- Must not alter the demo accounts (no role assignment, no
  password reset, no profile changes — the snapshot owns
  account state).
- Must not write to `mdl_user` or any user-table at runtime.
- Must not expose any endpoint other than the documented login
  endpoint and the standard admin settings page.

---

## Testing requirements

PHPUnit tests must cover, at minimum:

**Guard layer tests** (`guard_test.php`):
- Layer 1 disabled → refuse, regardless of other layers.
- Layer 2 wwwroot not in allowlist → refuse.
- Layer 2 wwwroot in allowlist (case sensitivity, trailing slash
  handling, protocol matching) → pass.
- Layer 2 dev override active with correct config → pass.
- Layer 2 dev override active without `DEBUG_DEVELOPER` → refuse.
- Layer 3 username not in allowlist → refuse.
- Layer 4 user has site:config → refuse.
- All four pass → authorize.

**Rate limiter tests** (`rate_limiter_test.php`):
- Per-IP limit enforced.
- Global limit enforced.
- Limits independent (one IP hitting per-IP doesn't affect
  others until global trips).
- TTL behavior.

**Endpoint tests** (`login_endpoint_test.php`):
- Missing `role` parameter → 404.
- Invalid `role` value → 404.
- Each valid role with all guards passing → 302 to correct
  landing URL with session established.
- Each valid role with any guard failing → 404.
- Rate limit hit → 429.

All tests must be runnable via the standard
`vendor/bin/phpunit --testsuite local_demoaccess_testsuite`
invocation.

---

## Acceptance criteria

The plugin is "done" for v1 when **all** of the following are
verifiable:

1. **phpcs Moodle ruleset clean.** `phpcs` against the Moodle
   coding standard returns zero errors and zero warnings on all
   plugin files.
2. **PHPUnit green.** All tests in the SPEC's testing
   requirements pass. No skipped tests in the v1 suite.
3. **Manual smoke complete.** All checks in `MANUAL_SMOKE.md`
   pass on a real Moodle instance (local Docker is fine for v1
   smoke; production smoke happens at demo-tenant deployment).
4. **Safety model verified end-to-end.** The agent (or the
   operator) demonstrates each of the four guards individually
   refuses access by deliberately misconfiguring one layer at a
   time and confirming the endpoint returns 404. This is
   documented as a discrete section in `MANUAL_SMOKE.md`.
5. **Real-transport verification.** The endpoint is exercised
   over actual HTTP (not just unit tests) on at least the local
   Docker instance, with the dev override active, for all three
   roles, with each landing on the correct URL with a working
   authenticated session.
6. **Customer-tenant inertness verified.** Installed on a
   non-demo tenant (or a simulated one — local Docker with
   wwwroot pointed elsewhere and dev override off), the
   endpoint returns 404 for all role values. This is the most
   important check in the suite; it must be in
   `MANUAL_SMOKE.md`.
7. **Documentation present.** `README.md`, `CHANGES.md`,
   `MANUAL_SMOKE.md`, `docs/DECISIONS.md` exist and are filled
   in per the per-plugin doc convention.

---

## Open questions for the agent to surface (not decide)

The following are decisions that may surface during
implementation. The agent should propose, but not unilaterally
decide:

- Exact phpcs / Moodle coding standard version pinned by the
  LMS Light stack (the agent should confirm against existing
  jport500 plugins rather than assume).
- Whether `MATURITY_ALPHA` is the right marker for v1 or whether
  the LMS Light convention is different.
- Cache definition specifics in `db/caches.php` (TTL strategy,
  store type) — propose based on similar plugins in the stack.
- Lang string keys naming convention — match existing jport500
  plugin conventions.

---

## Things this SPEC deliberately does not specify

The agent has implementation judgment on:

- Internal class structure beyond the named classes above.
- Exact Moodle event class shape (follow standard Moodle
  conventions).
- Settings page rendering details beyond the required content.
- Code style within Moodle ruleset bounds.
- Comments, PHPDoc level of detail (follow LMS Light
  conventions in existing plugins).

---

## Reference

- `demo-site-spec.md` — parent design document for the demo site
- `CONTEXT.md` — LMS Light infrastructure, working rhythm, and
  conventions
- `LESSONS.md` — portable failure modes; relevant during build,
  especially around guard evaluation order and rate-limit
  storage edge cases
- Existing jport500-maintained plugins (`auth_magiclink`,
  `local_welcomeemail`, `local_quizgenpro`) — reference for
  code style, doc conventions, and `version.php` shape
