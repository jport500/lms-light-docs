# Primary demo site — design spec

Point-in-time design document for the primary LMS Light demo
site, serving the B2B SaaS customer-success ICP. Lives at
`demo.lmslight.io` (or equivalent), a standalone LMS Light
tenant on the shared code stack.

A second demo site for the niche-training-operator ICP, focused
on CE tracking and certification workflows, will be specified
separately. That CE demo is out of scope for this document; this
spec deliberately excludes CE/certification machinery so the
primary demo stays sharply shaped for SaaS-CS evaluators.

For the supervised agentic working rhythm, plugin conventions,
and multi-tenant model, see `CONTEXT.md` in this repo. For the
funnel context that motivates two front-door demos, see
`lmslight-strategy-shift.md`. For the role-button login plumbing
this demo depends on, see `local_demoaccess-spec.md` and the
plugin repo `jport500/moodle-local_demoaccess`.

---

## ICP and audience

The primary demo serves **B2B SaaS companies that need training
platforms for their customers or partners** — customer education,
partner certification, product training.

The evaluator landing on this demo is asking: *"can LMS Light
give my customers a polished training experience without me
having to build a training company?"* The demo answers that
question by showing what a real LMS Light tenant looks like in
production for a plausible B2B SaaS customer.

The CE-tracking demo serves a different evaluator (a niche
training operator asking about compliance workflows) and is
intentionally not addressed here.

---

## The fictional context

### Meridian CRM — the fictional LMS Light customer

Mid-market vertical CRM for professional services firms (law
firms, accounting practices, consulting boutiques). ~200
employee SaaS company. Its customers are firms of 20–500 people.

This is the company whose LMS Light tenant the evaluator is
exploring. Meridian uses LMS Light to deliver:

- **End-user training** for staff at its customer firms learning
  to use the Meridian product.
- **Partner certification** for implementation consultants who
  deploy Meridian for new customer firms.

Brand setup (all configurable through Moodle's admin UI; no
theme code or custom CSS):

- **Name:** Meridian CRM
- **Tagline:** "The CRM built for professional services firms."
- **Brand color:** deep teal or navy (final shade picked during
  setup; intentionally distinct from Salesforce blue and HubSpot
  orange to avoid visual competitor association)
- **Logo:** wordmark "Meridian" set in a clean typeface in the
  brand color — no designed mark
- **Footer:** "© Meridian CRM"
- **Login page:** customized to show Meridian branding via
  Moodle's standard login customization

### Hartwell & Associates — the fictional learner firm

The Meridian customer whose staff are taking the courses. A
mid-sized consulting boutique, ~80 staff, multi-office.
Recognizable across audiences as a professional services firm
without requiring vertical expertise to grasp.

Seeded as 12 learners (see "Fictional team" below) plus a
manager persona.

---

## The two courses

Both courses use `format_pathway` (one-section-at-a-time linear
journey). Neither uses `tool_muprog`, `tool_mucertify`,
`tool_mutrain`, or `tool_murelation` — all CE/program/credit
machinery lives on the CE demo site, not here.

### Course 1 — "Getting Started with Meridian for Hartwell Staff"

End-user onboarding course for Hartwell staff being onboarded
to Meridian.

**Modules cover:**

- Navigating the Meridian interface
- Logging client communications
- Managing the engagement pipeline
- Running conflict checks
- Integrating Meridian with calendar and time-tracking tools

**Shape:** linear `format_pathway` journey. Modules are short,
practical, screenshot-heavy.

**Completion criteria:** all modules viewed + final quiz passed
(≥70%).

**Output:** course-completion record in Moodle's standard
completion system. No certificate artifact beyond the completion
record itself.

### Course 2 — "Meridian Implementation Specialist Certification"

Partner-facing certification course for implementation
consultants who deploy Meridian for new customer firms. Denser
and longer than Course 1.

**Modules cover:**

- Standard configuration patterns for professional-services
  firms
- Data migration from common predecessor tools
- Integration architecture with practice-management systems
- Post-launch support patterns
- A capstone scenario module that synthesizes the prior modules
  against a fictional implementation case

**Shape:** linear `format_pathway` journey, longer than Course 1.
Includes practical scenarios and the capstone.

**Completion criteria:** all modules viewed + final certification
quiz at ≥80% + capstone scenario module completed.

**Output:** course-completion record + a "Meridian
Implementation Specialist" certificate of completion (Moodle's
standard certificate, or `mod_customcert` if installed in the
stack — *the implementation phase will confirm which is
available; this is not a `tool_mucertify` artifact*).

The "certification" in the course name is the *fictional
Meridian certification*, not a technical Moodle certification.

---

## The fictional team — for the manager view

12 Hartwell staff seeded as learners on Course 1, plus 2 Hartwell
implementation consultants enrolled in Course 2. Progress
distributed so the manager view feels alive:

**Course 1 — 12 learners:**

- 3 learners: completed Course 1, currently working through
  scattered modules (or marked as having reviewed for refresh)
- 4 learners: mid-way through Course 1, varying progress (~30%,
  ~50%, ~60%, ~75%)
- 3 learners: just started Course 1 in the last week (~5–15%)
- 2 learners: enrolled but haven't started

**Course 2 — 2 learners:**

- 1 learner: ~40% through, mid-implementation
- 1 learner: just enrolled, hasn't started

**The manager persona** is the Hartwell training coordinator.
They see:

- Team progress across both courses
- Recent activity (who completed what when)
- Learners who haven't engaged (the 2 unstarted in Course 1)
- Course completion summary

Manager landing URL during build: `/my/courses.php` as the
local_demoaccess SPEC placeholder, revisited if a better team-
focused landing surfaces during build (see local_demoaccess
SPEC notes on manager landing being soft).

---

## Brand setup checklist

All items are configurable through Moodle's admin UI. No theme
code, no custom CSS, no plugin development.

1. **Tenant name** → Site administration → Site information →
   "Meridian CRM"
2. **Logo (wordmark)** → Site administration → Appearance → Logo
3. **Theme primary color** → Site administration → Appearance →
   theme config (brand color)
4. **Front page HTML block** → Site administration → Front page →
   Front page settings — the inline-styled HTML for the three
   role buttons + tagline framing (uses the approach established
   for local_demoaccess role-button placement)
5. **Footer** → "© Meridian CRM" via site settings
6. **Login page customization** → Moodle's standard login
   customization for the Meridian wordmark and tagline

---

## What "reset" includes and excludes on this demo

The demo runs on a nightly reset cycle (mechanism per
`demo-site-spec.md` once that exists; for now, snapshot-and-
restore from a known-good state).

**Reset restores:**

- The three shared demo accounts (`demo_admin`,
  `demo_manager`, `demo_learner`) to their initial state
- The 12 Hartwell learners' progress to their seeded
  distribution
- Any user-created content during the day (forum posts on
  any seeded forums, comments, etc.)
- Settings toggled during the day

**Reset preserves:**

- Course content itself (modules, quizzes, scenarios) —
  not re-seeded nightly
- Brand setup (logo, color, front page HTML, footer)
- Plugin code stack (unchanged by reset)

---

## Non-goals for this demo

The following are deliberately out of scope for the primary
demo. Each one is the CE demo site's territory:

- **No `tool_mucertify` certifications.** The Course 2
  certificate is a course-completion certificate, not a MuTMS
  certification artifact.
- **No `tool_mutrain` credit tracking, credits, or frameworks.**
- **No `tool_murelation` supervisor relationships.** The manager
  view uses course-completion data, not supervisor-team
  hierarchies.
- **No external credit submission workflows.**
- **No expiry, recertification, or renewal cycles.**
- **No compliance reporting or audit-shaped surfaces.**
- **No CE-hour seeding on the courses or the learners.**

If an evaluator asks "where's the CE/compliance/certification
shape," the answer is "that's our other demo site" — and
ideally lmslight.io's funnel routes those evaluators there
directly, not via this demo.

---

## Build sequencing (rough)

The supervised-agentic rhythm applies. Suggested phases:

1. **Tenant provisioning.** Stand up the demo tenant on
   standalone deployment. Standard LMS Light stack with
   local_demoaccess installed. Verify the role buttons work end
   to end.
2. **Brand setup.** Six items above. Verify front-page role
   buttons render correctly with Meridian branding.
3. **Course 1 content.** Write modules, build quizzes, configure
   completion. Verify a learner can complete end to end.
4. **Course 2 content.** Same shape as Course 1 plus the
   capstone scenario module. Verify completion and certificate.
5. **Seed the fictional team.** Create the 12 Hartwell learners
   + 2 Course-2 partner learners + the manager. Distribute
   progress per the spec.
6. **Verify the three role views.** Walk through admin, manager,
   and learner landings to confirm each role's experience is
   shaped correctly.
7. **Reset cycle verification.** Run the reset, verify the
   seeded distribution restores correctly.

Each phase ends with a review checkpoint per the working rhythm
in CONTEXT.md.

---

## Reference

- `CONTEXT.md` — working rhythm, plugin conventions, multi-tenant
  model
- `lmslight-strategy-shift.md` — why the demo site exists in the
  funnel
- `local_demoaccess-spec.md` — role-button login plumbing
- The CE demo spec will live alongside this one in
  `lms-light-docs` when drafted

---

## Status

Frozen at v1 as the historical design record for the primary
demo's build. If the demo's structure or content diverges from
this spec materially during build, the divergence should be
captured in build commit notes; this doc itself stays frozen
as the original intent.
