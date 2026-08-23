# 02 — Breaking Changes and Behaviour Changes in 2606

Everything here is a change that can make a working TC14 configuration behave
differently — or stop working — after the upgrade. Ordered by how likely it is to
bite you silently.

> **How to use this:** the items marked **SILENT** produce no error. Users notice
> them weeks later as "the system used to do X." Put every SILENT item into the
> UAT test plan explicitly.

---

## 2.1 Workflow — assignment matrix handler defaults changed — **SILENT, HIGH IMPACT**

Two default changes in the same handler:

**`-assignment_option` default changed from `override_if_exist` to `merge_if_exist`.**

- Before 2606: assignees from the assignment matrix *replaced* existing ad hoc
  assignees. Ad hoc signoffs added manually at submission, or by another handler,
  could be removed by Teamcenter.
- In 2606: matrix assignees are *merged* with existing assignees, preserving
  ad hoc signoffs.

To restore TC14 behaviour, set the argument explicitly in the handler config:

```
-assignment_option=override_if_exist
```

**Auto-completion removed.** The assignment matrix handler no longer auto-completes
the `Select Signoff Team` task after adding reviewers. The task now stays open for
manual completion, which aligns it with `EPM-adhoc-signoffs` and
`EPM-fill-in-reviewers`. To restore auto-completion, use the `EPM-adhoc-signoffs`
handler with `-auto_complete` on the task's start action.

**Why this matters:** any process that relied on the signoff team task closing
itself will now stall waiting for a human. This is the highest-risk silent change
in the release for sites with mature workflow automation.

**Test:** every workflow template that uses the assignment matrix handler, end to
end, in UAT.

---

## 2.2 Change Management — `CM_derive_thumbnail_from_relations` default — **SILENT**

The preference's default behaviour has been updated. It now behaves as:

- `true` (the default) — change object thumbnails derive dynamically from related
  items such as problem items.
- `false` — thumbnails do not derive automatically.

If your TC14 site relies on change objects showing their own thumbnail, set this
to `false` explicitly rather than depending on the default.

---

## 2.3 Active Workspace — new CSP-compliant expression engine — **LOW NOW, HIGH LATER**

The AngularJS `$parse`-based evaluation of dynamic expressions in JSON-defined
components has been replaced by a new Content Security Policy-compliant engine.
The driver is regulatory: NIS2 and the EU Cyber Resilience Act.

**Impact on this upgrade:** minimal. Siemens states existing conditional
expressions — `visibility`, `activeWhen`, `visibleWhen` — continue to work
unchanged, and existing application JSON does not need editing.

**Impact on the next upgrade:** a future Siemens Web Framework release will
enforce stricter validation of expressions in views, view models and config files.
Invalid expressions that are tolerated today will be rejected then.

**Action while the project is funded:** Siemens ships a script that identifies
invalid expressions. Run it across all AW/SWF customizations during Phase 1 and
fix what it finds. Doing this after go-live means re-opening cold code with no
budget.

---

## 2.4 Rich client SSO — `sso.properties` with integrity hashing — **CONFIGURATION BREAK**

SSO configuration has been pulled out of `client_specific.properties` into a new
dedicated `sso.properties` file, with three consequences:

1. **New file location.** SSO entries no longer live in `client_specific.properties`.
   Any deployment script, image build, or configuration-management recipe that
   writes SSO settings to the old file must be updated.
2. **Hash verification at login.** A unique hash is generated for `sso.properties`
   at installation and verified at each login. If the file has been modified, the
   login is terminated. Post-deployment scripts that patch this file will now
   break logins unless they also regenerate the hash.
3. **Admin-restricted hash utilities.** The utilities that generate or update the
   hash are restricted to administrators.

This hardens the 4-tier HTTP SSO mechanism, but it means any "we tweak the
properties file after install" habit has to be replaced with the supported
hash-regeneration step.

---

## 2.5 FIPS encryption — environment-wide constraint — **ARCHITECTURAL**

2606 supports FIPS-compliant encryption for all HTTPS/SSL client and server
communication, enabled through Deployment Center on the Options tab as an
environment-level property that propagates to component-level properties.

Two hard constraints:

- **If FIPS is enabled, every integration must be FIPS-compliant.** Siemens names
  openSAML as an example of something that is not. Audit every integration before
  enabling.
- **TEM does not support FIPS.** FIPS requires Deployment Center.

Decide this early. Turning it on late invalidates integration testing.

---

## 2.6 Content Security Policy directives move to Deployment Center — **CONFIGURATION BREAK**

From 2512 you created CSP directives by hand-editing `config.json` in
`microservices/gateway`. In 2606 you define them in Deployment Center, which
validates them against the allowed-value list and persists them to its database
so every redeploy or update regenerates `config.json` with your configuration.

**Migration action:** capture your current hand-edited CSP directives, re-enter
them in Deployment Center, then stop editing `config.json`. Any remaining manual
edits will be silently overwritten on the next deployment.

---

## 2.7 Object deletion from folder subtypes — **BEHAVIOUR CHANGE, permission-visible**

TC14 and earlier applied a bypass allowing a user to delete an object even when it
was referenced by a folder they had no write permission on. In 2606:

- **Standard folders** — bypass retained. Users can still remove objects without
  write access to the folder.
- **Subtypes of `Folder`** — bypass is **off by default**. Deleting an object from
  a specialised folder now requires write access to that folder.

The bypass is configurable per folder subtype. If you have custom folder subtypes
in an established deletion process, users will start hitting permission failures
they never saw on TC14. Review your subtypes and configure the bypass where the
old behaviour is required.

---

## 2.8 Paste command now honours server-side GRM rules — **UI BEHAVIOUR CHANGE**

Previously the Active Workspace client decided Paste visibility from client-side
conditions alone, which allowed the UI to offer a paste that the server's Generic
Relationship Management rules would then reject. In 2606 the client evaluates
server-side conditions, so Paste appears only when the GRM rules permit the
relationship.

Net effect: fewer failed pastes, but the command will now be **absent** in places
where TC14 users were used to seeing it (and getting an error). Expect "the Paste
button disappeared" tickets. The fix, where the operation should be legal, is to
correct the GRM rules in BMIDE — the client is now telling you the truth about your
data model.

---

## 2.9 BMIDE — thousands separator now locale-dependent — **SILENT, DATA DISPLAY**

When creating a double-precision property formatter, the 1000-place separator was
always a comma. In 2606 it depends on the `SiteMasterLanguage` constant:

- European language (e.g. `de_DE`) → period as thousands separator: `1.000.000,00`
- Non-European language → comma, as before: `1,000,000.00`

Sites with `SiteMasterLanguage` set to a European locale will see numeric
formatting change across formatted double properties. Anything downstream that
parses those formatted strings — reports, exports, integrations, test assertions —
needs checking.

---

## 2.10 Teamcenter Quality — FMEA section renamed — **COSMETIC, but breaks docs/training**

`Failure Cause Actions` is now `Failure Root Cause Actions`. Update training
material, work instructions and any automated UI tests that select by label.

---

## 2.11 Reporting — tab renamed — **COSMETIC**

In the Search Data panel of the Generate report flow, the first tab is renamed from
`Results` to `Keyword`. Same caveat: training material and UI test selectors.

---

## 2.12 Microsoft email requires OAuth2 — **INTEGRATION BREAK**

Microsoft email servers now require a different authentication type. If Teamcenter
sends mail through a Microsoft server, you must configure OAuth2 protocol
authentication by setting the required preferences. Basic authentication mail
configuration carried over from TC14 will stop working.

This one is driven by Microsoft's timeline, not Siemens' — verify current state
before assuming your existing config still works even on TC14.

---

## 2.13 New default theme — **COSMETIC, but user-visible on day one**

The **Framed Theme** is the default for new installations. Siemens documentation
screenshots continue to use the Classic Theme. Administrators can configure a
different default.

For an upgraded environment, decide deliberately: leaving users on Classic reduces
change noise at cutover; moving to Framed gets the change over with while you have
support capacity mobilised. Either is defensible — drifting into it by accident is
not.

---

## 2.14 Teamcenter apps for Microsoft Teams / M365 Copilot decouple from releases

Separate apps now exist for Teamcenter for Microsoft Teams and Teamcenter for M365
Copilot, and they work across all release versions. You no longer swap or reinstall
the app when upgrading Teamcenter, and enhancements arrive independently of the
Teamcenter release schedule.

**Upgrade impact:** positive, but it removes the app from your change-control
boundary. Functionality can now change without a Teamcenter deployment. Flag this
to anyone who validates the environment for regulated use.

---

## 2.15 API removals

Covered in full in `03-api-remediation.md`. Summary: **188 ITK artifacts** and
**54 SOA operations** that exist in TC14 have already been removed at or before
2606. This is a compile/link failure, not a behaviour change — you will find it
immediately, which makes it the least dangerous item on this page.

---

## SILENT-change test matrix

Give this to UAT. These produce no error message.

| # | Change | Test |
|---|---|---|
| 2.1 | Assignment matrix `-assignment_option` | Submit a workflow with ad hoc signoffs + matrix; confirm both sets present |
| 2.1 | Select Signoff Team no auto-complete | Confirm task closes as expected, or accept manual step |
| 2.2 | `CM_derive_thumbnail_from_relations` | Open a change object with related problem items; check thumbnail |
| 2.7 | Folder subtype deletion | Delete an object from each custom folder subtype as a non-owner |
| 2.8 | Paste visibility | Attempt each business-critical paste relationship |
| 2.9 | Numeric formatting | Compare a formatted double property before/after; check downstream parsers |
| 2.13 | Default theme | Confirm the theme users actually land on |
