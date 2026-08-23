# 01 — Upgrade Strategy: TC14.x → TC2606

## 1.1 The shape of the problem

A TC14 → 2606 upgrade is not a point upgrade. Nine feature releases separate the
two versions, and the 2312 boundary in particular removed a large body of ITK and
SOA surface area that TC14 code is free to use. Three workstreams run in parallel:

| Workstream | Driver | Typical critical path |
|---|---|---|
| **Platform** | OS, database, JDK, web tier, Deployment Center version | Usually shortest — but gates everything else |
| **Data** | Schema migration, effectivity migration, configurator JSON migration, full re-index | Longest for large volumes; drives cutover window |
| **Customization** | 188 removed ITK artifacts, 54 removed SOA operations, AW/SWF client rewrite pressure | Longest overall; start first |

Start the customization scan on day one. It is the workstream most likely to
discover work that cannot be compressed.

---

## 1.2 Choosing the upgrade route

### Option A — Blue-green (recommended for 2606)

2606 is the release where Siemens made blue-green the headline upgrade story. A
parallel upgraded environment is prepared and kept in sync with production, then
switched over. Production keeps running throughout preparation, so the outage is
reduced to the switchover itself rather than the whole migration.

Siemens states explicitly that this is intended to help customers on **older**
Teamcenter versions get current, not just customers moving one release forward —
which is precisely the TC14 case.

**Choose blue-green when:** downtime tolerance is low, data volume is large, or
the business cannot absorb a multi-day freeze.

**Cost:** you need the hardware/cloud capacity to run two full environments, plus
the sync mechanism, plus a rollback plan that accounts for data written to the
green environment after switchover.

### Option B — Classic in-place upgrade

Backup → upgrade schema and volumes in place → validate → release. Simpler, fewer
moving parts, well-understood rollback (restore from backup).

**Choose in-place when:** you have a real maintenance window (a long weekend or
more), moderate data volume, and no appetite for running duplicate infrastructure.

### Option C — Teamcenter X (managed cloud)

Hands upgrades to Siemens permanently. Relevant if the driver for leaving TC14 is
"we don't want to do this again in three years." It converts the upgrade project
into a migration project with a different risk profile — customization
portability becomes the dominant constraint, since heavy server-side ITK
customization does not travel well to a managed environment.

---

## 1.3 Deployment Center is now the assumed installer

Plan on **Deployment Center**, not TEM. Two 2606 facts make this concrete:

- **`Copy Environment`** — Deployment Center can now clone a Teamcenter
  production environment directly. Previously, producing the intermediate
  environment for an upgrade meant a sequence of manual export/import steps.
  This is the button that makes blue-green practical, and it is the natural way
  to stand up your dry-run environment.
- **FIPS is Deployment Center-only.** The 2606 documentation states plainly that
  the TEM installer does not support FIPS. If FIPS-compliant encryption is a
  requirement (US government / defence contracting), TEM is not an option at all.

Deployment Center also now owns **Content Security Policy directive
configuration**, which in 2512 was a manual edit of `config.json` under
`microservices/gateway`. If you have hand-edited CSP directives anywhere, they
must move into Deployment Center or they will be overwritten on the next deploy.

---

## 1.4 Environment plan

Minimum four environments. Do not compress this.

| Env | Purpose | Notes |
|---|---|---|
| **DEV** | Customization port and rebuild against 2606 libraries | Small dataset. First environment to exist. |
| **TEST / dry-run** | Full-volume upgrade rehearsal, timed | Must be a real clone of production data, not a subset. This is where you get your cutover duration estimate. |
| **UAT** | Business validation against upgraded data | Where the behaviour changes in `02-breaking-changes.md` get found by the people who will notice. |
| **PROD** | — | |

Run the full-volume dry run **at least twice**. The first run finds the errors;
the second run gives you a trustworthy duration. Cutover planning based on a
single dry run is planning based on a number that includes your mistakes.

---

## 1.5 Phase model

### Phase 0 — Assess (2–4 weeks)
- Run `tools/tc_api_scan.py` against every customization repository, including
  workflow handler code, ITK user exits, SOA client code, AW/SWF client
  extensions, and any BMIDE-attached code.
- Inventory: BMIDE templates, preferences deltas from OOTB, stylesheets, workflow
  templates, custom handlers, saved queries, report definitions, integrations.
- Verify platform certification for 2606 (OS, DB, JDK, browser) against the
  Siemens Release Bulletin — this can invalidate an otherwise-viable plan.
- Check integration certification: NX, Simcenter, Office integration, Dispatcher
  translators, ERP connectors, any Mendix or third-party apps.
- **Deliverable:** impact register with effort estimates, and a go/no-go on the
  target date.

### Phase 1 — Remediate customizations (longest phase)
- Fix all BLOCKERs from the scan. See `03-api-remediation.md`.
- Fix WARNINGs too. Removal is targeted for 2712 and 2912 — deferring them means
  doing this again in one or two releases, with the code cold.
- Rebuild against the 2606 SDK. Migrate BMIDE templates.
- Review AW/SWF client customizations against the new CSP-compliant expression
  engine (see `02-breaking-changes.md` §2.3) and run the Siemens-supplied
  expression validation script.

### Phase 2 — Platform build
- Stand up 2606 in DEV via Deployment Center. Establish the deployment
  configuration as code where possible so the same config produces TEST and PROD.
- Decide FIPS on/off now — it is an environment-level property in Deployment
  Center and it constrains every integration (openSAML, for example, is not
  FIPS-compliant).

### Phase 3 — Data migration dry runs
- Clone production (`Copy Environment`), upgrade, time every step.
- Run the data migration utilities in `06-data-migration.md`, notably
  `effupgrade` for legacy effectivity and the configurator JSON migration.
- **Full re-index is mandatory** if you add a locale, and strongly advisable
  regardless given the indexing changes in this release. Time it — on large sites
  it is frequently the single longest step and it can often be run against the
  green environment ahead of cutover.

### Phase 4 — UAT
- Business validation focused on the behaviour changes, not on "does it start."
  Users will not report a changed default; they will report that a workflow
  suddenly needs a manual click. Give testers the breaking-changes list.

### Phase 5 — Cutover
- Follow `07-cutover-checklist.md`.
- Rollback decision point must be defined **before** the window opens, with a
  named owner and a hard time.

### Phase 6 — Adopt
- 2606 ships a large amount of new capability that is **off by default** and
  needs deliberate enablement: People Picker suggestions, optional-family solve
  priority, the inline Relations Tree, the new theming default, Copilot object-
  property indexing. See `04-whats-new-2606.md`. Treat adoption as a funded phase,
  not as something that happens on its own after go-live.

---

## 1.6 Risk register (starting set)

| Risk | Impact | Mitigation |
|---|---|---|
| Removed ITK APIs with no replacement (101 of them) | Custom code must be redesigned, not ported | Scan in Phase 0; redesign is a design task, budget accordingly |
| Re-index duration on large sites | Cutover window overrun | Time it in dry run; consider indexing the green environment pre-switchover |
| Assignment matrix handler default change | Workflows silently behave differently | See `02-breaking-changes.md` §2.1 — explicit argument restores old behaviour |
| CSP expression validation tightening | AW customizations break in a *future* release, not this one | Run the validation script now while the project is funded |
| FIPS decision made late | Integration incompatibilities discovered post-build | Decide in Phase 2 |
| Platform certification gap (OS/DB) | Whole plan invalid | Verify against Release Bulletin in Phase 0 |
| Single dry run | Cutover estimate is wrong | Two dry runs minimum |
