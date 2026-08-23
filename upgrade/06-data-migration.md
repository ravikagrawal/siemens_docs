# 06 — Data Migration and Utilities

Data-level work required or enabled by the move to 2606. These are the steps that
consume the cutover window, so each one needs a timed dry run.

---

## 6.1 Legacy effectivity migration — `effupgrade`

**Applies to:** any site still holding legacy effectivity data. On TC14 this is
common, since the migration off `CFM_*` effectivity has been available but
optional for several releases.

**What changed in 2606.** The utility was previously a manual multi-step process —
run `Report` mode, read the report files to understand the details, run `Upgrade`,
then run `Cleanup` — in a specific sequence. In 2606:

- **`Report` mode gives deeper analysis and statistics** on your legacy data, so
  you can plan the migration from the report rather than from guesswork.
- **`Upgrade` mode now migrates `CFM_date_info` to the effectivity object and
  cleans up the legacy `CFM_date_info` objects itself.** You no longer run cleanup
  as a separate step.
- **Batch processing.** The utility uses a default batch size, but you can specify
  a custom batch size and a number of batches to run. This is what makes the
  migration schedulable around a business calendar instead of demanding one long
  window.
- **Summary on the console.** Migration status appears directly at the Teamcenter
  Command Prompt rather than only in files.

**Recommended sequence**

1. Run `Report` mode against a production clone. Read the statistics; they tell you
   the volume and therefore the duration.
2. Estimate total runtime from a timed batch on representative data.
3. Decide batch size and batch count against your available windows.
4. Run `Upgrade` in batches, verifying after each.
5. Confirm cleanup happened (it is now part of upgrade, but verify rather than
   assume).

**Coordinate with code.** The `CFM_effectivity_*` ITK functions were removed at
2312 and replaced by `PS_occ_eff_*` — see `03-api-remediation.md` §3.3. Custom code
touching effectivity and the data migration must land together.

---

## 6.2 Product Configurator — variant expression migration to JSON

**Applies to:** sites using Product Configurator with any meaningful volume of
variant expressions.

**What it is.** Product Configurator historically saved most variant expressions as
a grid — a deep nested structure in the database — which causes performance
problems on data lookup. 2606 introduces a JSON string representation instead, and
ships **new utilities to migrate existing variant expressions to JSON**.

**Why bother.** The stated benefit is faster authoring, updating, deleting and
loading of variant configuration objects. On configurator-heavy sites this is one
of the more tangible performance wins in the release.

**Planning notes**

- This is an *optional* migration, not a mandatory upgrade step — but the
  performance characteristics of the old structure do not improve on their own.
- Run it on a production clone first and measure both the migration duration and
  the resulting performance change before committing.
- Because it is optional, it is a good candidate for **post-cutover** execution:
  it removes work from the critical path and lets you measure the benefit against
  a stable 2606 baseline.

**Related configurator setting.** `Cfg0EnableOptionalFamilyPriority` (site
preference, default off) enables sequence-number and allocation-sequence ranking so
the solver can decide which system-generated defaults to drop. On configurations
with many optional families and multi-select features this addresses a real solve
performance problem — but it requires someone to define sensible sequence values,
so treat it as a configuration project rather than a switch.

---

## 6.3 Full re-index

**Mandatory when:** you add a language. Siemens states plainly that a full index of
your data is required after adding a new language — relevant if you are taking
advantage of the new Ukrainian (`uk_UA`) locale.

**Strongly advisable regardless**, given this release's indexing changes:

- Compound properties on business objects **without their own storage class** can
  now be indexed — a previously hard limitation. If you have wanted this, the data
  needs re-indexing to pick it up.
- **Copilot for documents and requirements can now draw on indexed and embedded
  object properties**, but only after an administrator configures it to include
  them. Those properties must be in the index.
- Property-specific search depends on properties being indexable, and on an
  administrator-defined preferred-property list.

**Planning notes**

- On large sites this is frequently the single longest step in the whole cutover.
  **Time it in the dry run.** Do not estimate it.
- In a blue-green topology, indexing can often run against the green environment
  before switchover, taking it off the critical path entirely. This is one of the
  strongest practical arguments for blue-green on a large TC14 estate.
- Verify index completeness before releasing to users — a partially indexed system
  looks to users like data loss.

---

## 6.4 Schema and BMIDE template migration

Standard upgrade work, but note for a nine-release jump:

- Every custom BMIDE template must be migrated forward and rebuilt against the 2606
  BMIDE. Template dependencies on OOTB constructs that changed across nine releases
  are found here.
- Verify GRM rules explicitly. 2606's Paste command now evaluates server-side GRM
  rules (§2.8), so any inconsistency between what your GRM rules say and what users
  actually do becomes visible as a missing command. Reviewing GRM rules during
  template migration is cheaper than debugging it in UAT.
- Business object constants, including `SiteMasterLanguage`, affect the new
  locale-dependent numeric formatter (§2.9). Confirm the value is what you intend.

---

## 6.5 Configuration promotion between environments

New in 2606 and directly useful during the upgrade itself: **column configuration
export now carries its scope** — Site, Group, Role or Workspace — inside the
definition file, and importing applies it to the correct scope automatically.

This is the kind of small improvement that saves real time on an upgrade project,
where the same configuration gets promoted DEV → TEST → UAT → PROD repeatedly. It
eliminates the manual tracking of which file belongs to which scope, and the
resulting misapplied-configuration errors.

Similarly, **CSP directives are now stored in the Deployment Center database** and
re-applied on every redeploy, so they stop being something you re-merge by hand
after each deployment.

---

## 6.6 Migration sequencing summary

| Step | Critical path? | Can defer to post-cutover? | Must time in dry run |
|---|---|---|---|
| Schema / BMIDE migration | Yes | No | Yes |
| Volume migration | Yes | No | Yes |
| `effupgrade` legacy effectivity | Yes, if code depends on it | Partially — batchable | Yes |
| Full re-index | Yes, unless blue-green | No (users need search) | **Yes — usually the longest** |
| Configurator JSON migration | No | **Yes** | Optional |
| Column config promotion | No | No | No |
| Optional family priority setup | No | **Yes** | No |

**Rule of thumb:** anything in the "can defer" column should be deferred. The
cutover window is the scarcest resource in the project, and every optional step
inside it is risk bought for no schedule benefit.
