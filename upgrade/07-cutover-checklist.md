# 07 — Cutover Checklist

Working checklist for a TC14 → 2606 cutover. Adapt to your topology; the ordering
and the decision gates are the part worth keeping.

---

## Gate A — Pre-flight (T minus 4 weeks)

**Customizations**
- [ ] `tc_api_scan.py` returns exit code **0** across every customization repository
- [ ] All customizations rebuilt against the 2606 SDK, clean build, no warnings suppressed
- [ ] Deprecated-API warnings triaged; anything targeted for 2712 removal is fixed or has a written, owned deferral
- [ ] Siemens expression-validation script run across all AW/SWF customizations; findings fixed
- [ ] BMIDE templates migrated, deployed and verified in TEST

**Platform**
- [ ] OS, database, JDK, web tier and browser versions confirmed against the 2606 Release Bulletin
- [ ] Deployment Center version current and capable of the target deployment
- [ ] FIPS decision made and recorded; if enabled, every integration confirmed FIPS-compliant
- [ ] Licence server updated and validated for 2606
- [ ] Capacity confirmed (blue-green needs headroom for two environments)

**Integrations** — for each of NX, Simcenter, Office, Dispatcher, ERP, Polarion, MES, Mendix, third-party:
- [ ] Certified against 2606
- [ ] Tested in TEST against upgraded data
- [ ] Owner identified and available during the cutover window
- [ ] Microsoft email reconfigured for OAuth2 and verified (§2.12)

**Dry runs**
- [ ] **Two** full-volume dry runs completed on a production clone
- [ ] Every step timed; total duration + 50% contingency fits the window
- [ ] Re-index duration measured separately and explicitly
- [ ] `effupgrade` Report mode run; volume and batch plan agreed

**Readiness**
- [ ] UAT signed off, including every SILENT change in `02-breaking-changes.md`
- [ ] Rollback plan written, tested in dry run, with a named decision owner and a hard decision time
- [ ] Comms sent: outage window, what changes visually on day one, where to get help
- [ ] Support capacity increased for the first week
- [ ] Theme decision made (Classic vs. Framed) and communicated

---

## Gate B — T minus 1 week

- [ ] Change freeze in effect on TC14
- [ ] Backups verified by **restore test**, not by the backup job reporting success
- [ ] Cutover runbook walked through with every participant, with named owners per step
- [ ] Blue-green only: green environment built, synced, and sync verified
- [ ] Go/no-go held; decision recorded

---

## Gate C — Cutover window

**Shutdown**
- [ ] Users notified and sessions terminated
- [ ] Integrations and scheduled jobs stopped (Dispatcher, ERP feeds, translators, batch)
- [ ] Final backup taken and verified
- [ ] Baseline metrics captured (object counts, key query timings) for post-upgrade comparison

**Upgrade**
- [ ] Schema upgrade — record start and end time
- [ ] Volume migration — record duration
- [ ] `effupgrade` batches (if on the critical path) — verify after each batch
- [ ] Deployment Center deployment of application tier
- [ ] Configuration applied: preferences, stylesheets, column configs, CSP directives, workflow templates
- [ ] SSO: `sso.properties` in place with a valid hash (§2.4)

**Index**
- [ ] Full re-index started (or verified complete on the green environment)
- [ ] Completeness verified, not just "the job finished"

**Smoke test** — before anyone else is allowed in:
- [ ] Login: each authentication path, each client (AW, rich client, Office, mobile)
- [ ] Search returns results for known objects
- [ ] Open a known item revision, a known assembly, a known dataset
- [ ] Check-out / check-in a CAD file through each CAD integration
- [ ] Submit a workflow through **every template that uses the assignment matrix handler** (§2.1)
- [ ] Create a change object; confirm thumbnail behaviour (§2.2)
- [ ] Delete an object from a custom folder subtype as a non-owner (§2.7)
- [ ] Paste each business-critical relationship (§2.8)
- [ ] Verify numeric formatting of a formatted double property (§2.9)
- [ ] Each integration performs one real transaction end to end
- [ ] Custom code paths: at least one exercise of each remediated area

**Go/no-go**
- [ ] Smoke test passed
- [ ] Rollback decision time not exceeded
- [ ] Named owner authorises release to users

---

## Gate D — First 72 hours

- [ ] Elevated support staffing in place and visible to users
- [ ] Performance monitored against the pre-cutover baseline
- [ ] Error and exception logs reviewed at least twice daily
- [ ] Ticket themes triaged daily — cluster by cause, not by reporter
- [ ] Known-issues page published and updated as things are found

**Expect these tickets.** Having answers ready converts them from incidents into
FAQ entries:

| Symptom | Cause |
|---|---|
| "The Paste button is gone" | GRM rules now enforced client-side (§2.8) |
| "My workflow is stuck on Select Signoff Team" | Handler no longer auto-completes (§2.1) |
| "Approvers I added disappeared / didn't disappear" | `-assignment_option` default changed (§2.1) |
| "Everything looks different" | New default theme (§2.13) |
| "Numbers are formatted wrong" | Locale-dependent thousands separator (§2.9) |
| "I can't delete this any more" | Folder subtype bypass off by default (§2.7) |
| "Search finds nothing" | Index incomplete — check before assuming data loss |

---

## Gate E — First 30 days

- [ ] Deferred data migrations executed (configurator JSON, remaining `effupgrade` batches)
- [ ] Deprecated-API register reviewed and scheduled against the 2712 deadline
- [ ] Wave 1 adoption started (`05-aw-feature-delta.md` §5.5)
- [ ] Lessons-learned session held while memory is fresh
- [ ] Dry-run timings archived — they are the starting estimate for the next upgrade
- [ ] Decide the next upgrade cadence **now**. The cost of this project is
      overwhelmingly a function of having skipped nine releases. A site that
      upgrades every second release never accumulates a 188-API removal backlog.

---

## Rollback triggers

Define these before the window opens, in writing, with the authoriser named. A
rollback decision made under pressure without pre-agreed criteria is a decision
made badly.

Suggested triggers:
- Schema or volume migration fails and cannot be resolved within *N* hours
- Smoke test fails on any authentication path
- Any business-critical integration cannot complete a transaction
- Data integrity discrepancy found against the pre-cutover baseline
- The rollback decision time is reached with the outcome still uncertain

That last one matters most. The decision time exists so that rollback remains
*possible*; passing it while hoping for improvement is how a recoverable upgrade
becomes an unrecoverable one.
