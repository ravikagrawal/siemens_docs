# 08 — Blue-Green Migration Strategy: TC14 → TC2606

**Primary source:** *Teamcenter Update Using Deployment Center*, Software Version 2606
(Siemens, published 21 May 2026) — `Teamcenter Update Using Deployment Center.pdf`
in the source repository. Cross-referenced with *What's New in Teamcenter 2606* and
general industry practice for blue-green deployment.

---

## Part 1 — What blue-green actually means

### 1.1 The general pattern

Blue-green deployment is a release technique from web operations, formalised around
2010. Two production-capable environments exist:

- **Blue** — the environment currently serving users
- **Green** — an idle environment where the new version is built and validated

Release is a **traffic switch**, not an upgrade. Users move from Blue to Green in
one cutover step. Rollback is the reverse switch, because Blue is still standing
and still working.

The core claim is a change in what "downtime" means. In a conventional in-place
upgrade, downtime spans the entire upgrade — you take the system down, migrate it,
test it, and bring it back. In blue-green, the migration and most testing happen
while Blue is still serving users, so downtime shrinks to the switchover.

Four properties are what people are actually buying:

| Property | What it gives you |
|---|---|
| **Reduced downtime** | Long-running work moves outside the maintenance window |
| **Real rollback** | Blue is intact and running, not a backup you hope restores |
| **Test on production data** | Green holds a real clone, not a synthetic subset |
| **Decoupled schedule** | Build and validation are not hostage to a weekend window |

### 1.2 Where the classic pattern breaks for PLM

Textbook blue-green assumes stateless application servers behind a load balancer.
Teamcenter is the opposite: a large stateful database, file volumes, FMS keys,
licence servers, CAD integrations, and desktop clients with local configuration.

Three problems follow:

1. **State divergence.** From the moment you clone the database, Blue keeps
   accepting user changes. By switchover, Green's clone is stale by however long
   the upgrade took — potentially weeks.
2. **Schema change.** This is not deploying the same application twice. Green runs
   a *different schema*. You cannot simply replay Blue's transactions into Green.
3. **Shared file volumes.** Volumes are large and cannot be trivially duplicated,
   yet both environments reference them.

Problem 1 combined with problem 2 is what historically made blue-green impractical
for Teamcenter upgrades. **Solving it is exactly what Siemens shipped.**

---

## Part 2 — Siemens' implementation

### 2.1 Blue-green is the official platform update method

The 2606 guide is explicit:

> *"This Platform Update is based on the industry blue-green deployment approach,
> using intermediate servers to maximize uptime during the update process."*

This is not an optional technique bolted onto the side. For a TC14 source, the
Platform Update Quick Start **is** the blue-green procedure.

### 2.2 Your upgrade path is supported and direct

> *"Teamcenter 2606 supports direct update from Teamcenter 14.0 or later. If your
> current version is earlier than 14.0, you must update to version 14.0 or later
> before you update to Teamcenter 2606."*

**TC14 → 2606 is a single-hop, directly supported update.** No stepping stone
release is required. TC14.0 is precisely the floor.

Two consequences of sitting on that floor:

- You are at the oldest supported source. There is no margin — if this upgrade
  slips past the 2612 or 2706 timeframe, verify the floor has not moved.
- You cross the 2312 API removal boundary in one jump (see `03-api-remediation.md`).

Confirm the specifics against the **Teamcenter Interoperability Matrix**
(`Teamcenter Interoperability Matrix <date>.xlsx`) on the Support White Papers
Certifications page, which the guide names as the authority for complete supported
update paths.

**Basic Update vs. Platform Update.** The Basic Update route applies only when
updating from 2512 or later, where no new `<TC_ROOT>` and `<TC_DATA>` directories
are needed. TC14 does not qualify. You take the Platform Update route, and
Deployment Center will **automatically create new home and data directories** when
it recognises the platform-update-level change. You can rename them before deploying.

### 2.3 The Instance Data Sync utility — the mechanism that makes this work

IDS is the answer to the state-divergence problem. Siemens describes it as a user
data synchronization utility that lets users remain active in Blue significantly
longer, replicating data between the active Blue source database and the cloned
Green target.

**How it works:**

- **Change tracking** — lightweight triggers monitor INSERT, UPDATE and DELETE on
  all lightweight POM classes and `POM_object`, recording into an
  `INSTANCEDATASYNCCHANGES` table.
- **Recipe-based transformation** — JSON recipes morph data between schema versions,
  which is what makes syncing *across* a schema change possible rather than just
  replaying transactions.
- **Cross-database** — Oracle and Microsoft SQL Server.
- **Version migration** — handles schema evolution through recipe positioning.

**Supported versions include 14.0, 14.1, 14.2 and 14.3 as sources, and 2606 as a
target.** When synchronizing across a gap, the system identifies all intermediate
versions between source and target and applies transformation recipes in
chronological order. For a TC14 source, that means the recipe chain runs
14.x → 2306 → 2312 → 2406 → 2412 → 2506 → 2512 → 2606 automatically.

**The four actions:**

| Action | Runs on | Does |
|---|---|---|
| `StartSource` | Blue | Creates the changes table, installs triggers, begins monitoring |
| `StartDestination` | Green | Removes any restored triggers, cleans the changes table, **advances the UID generator to prevent conflicts** |
| `Sync` | Green | Reads change records, groups by class and operation, applies recipes, replicates INSERT/UPDATE/DELETE in correct order |
| `FinishSource` | Blue | Optional. Drops triggers, removes the changes table and temporary objects |

Example invocation (Oracle):

```
InstanceDataSync.exe --action StartSource --sourceDbType "Oracle"
  --sourceDbServer "<host>" --sourceDbUsername "<user>"
  --sourceDbPassword "<password>" --sourceDatabase "<schema>"
  --sourceSsid "<SID>" --log "<logs-directory>"
```

The `Sync` action additionally takes `--destinationDb*` parameters and an optional
`--recipe` directory containing `*_maprecipe.json` files.

### 2.4 Recipes — read this if you have custom data model

Recipes are JSON transformation definitions applied during sync:

```json
[
  {
    "version": "13.1",
    "pomClass": "Cst0Base",
    "elements": [
      {
        "type": "SetAttribute",
        "attributeName": "cst0RevisionAnchor",
        "expression": "x.cst0IRDI.Replace(\"#01-\", \"#\")"
      }
    ]
  }
]
```

Element types include `SetAttribute` (dynamic expression assignment) and
`VlaToRelation` / `RelationToVla` (converting between variable-length arrays and
relation objects). Expressions use dynamic LINQ over the source object `x`.

Filtering is supported globally via LINQ predicates, with defined semantics:
matching rows are transformed, **non-matching rows are preserved unchanged and
merged back**, and Siemens states the design goal as zero data loss — all rows
synchronize regardless of filter criteria.

**Why this matters for a TC14 site:** Siemens supplies recipes for OOTB schema
evolution. If your BMIDE templates changed custom classes in ways that need
transformation, you may need to author your own. Assess this during Phase 0 — it
is specialist work and it is on the critical path.

### 2.5 Volume handling

IDS processes database content but **does not process Teamcenter volume files**.
Two supported approaches:

| Approach | How | Constraint |
|---|---|---|
| **Shared volumes** (Siemens best practice: *"Both Blue and Green databases should point to the same volumes"*) | Share user volumes between Blue and Green; use a **copy** of Blue's dba volume for Green's dba volume | **Do not modify production volume data in Green while testing** |
| **Full copy** | Complete copy of Blue's volumes for Green | Requires volume-scale storage and a sync before switchover |

If you copy rather than share, synchronize volumes by copying from production into
the intermediate environment machine **before making Green active**. Not required
with shared volumes.

### 2.6 Item ID conflicts — a real hazard

Testing on Green means creating data on Green, on a clone of production, while
production keeps allocating IDs. Siemens gives three mitigations:

1. **Restrict item creation** during Green testing
2. **Use a different naming range** for test data
3. **Restore the clone and re-sync user data** after testing completes

`StartDestination` advances the UID generator to prevent conflicts with future
source changes, but that protects UIDs, not business item IDs. Option 3 is the
cleanest and I would treat it as the default: test freely, then wipe and re-sync.

### 2.7 The topology: three environments, not two

The Siemens procedure uses **three** environments, which surprises people expecting
a two-box model:

| Environment | Role |
|---|---|
| **Blue** | Existing TC14 production. Stays live throughout. |
| **Intermediate / upgrade server** | Single-box replica of Blue that receives the clone and is upgraded to 2606. This is where the schema migration actually runs. |
| **Green** | Newly built, clean TC2606 production environment with the correct scaled architecture. Receives the upgraded database from the intermediate server. |

The intermediate server is deliberately minimal — the guide directs you to strip
the exported Quick Deploy XML down to a single box, keeping only Corporate Server,
FSC, Database Server, License Server, FSC Group, FSC Keys, and required components
such as Teamcenter Vault and HTTPS Config, removing extra server managers,
Dispatcher modules and microservice nodes.

**Why three.** The intermediate box exists to run a long, resource-hungry, possibly
repeated schema migration without contaminating the environment users will land on.
Green is built clean at full production scale. This separation is what lets you
re-run the migration after a failure without rebuilding your production topology.

---

## Part 3 — The migration strategy for a TC14 customer

### Stage 0 — Planning and assessment

**Software planning**
1. Read *What's New* for 2606 **and for every intervening version** — 14.1, 14.2,
   14.3, 2312, 2406, 2412, 2506, 2512. The guide explicitly calls for this. Nine
   releases of behaviour change land at once (see `02-breaking-changes.md`).
2. Teamcenter Release Bulletin — issues affecting your update.
3. `README.csv` for 2606 — resolved defects and PRs.
4. Deployment and Administration guides for each application you run, via
   *1st Stop: Teamcenter Documentation Home*.
5. **Platform and Software Certification Matrix** (`Tc2606PlatformMatrix-date.xlsx`)
   — certified OS and third-party software. Download from Support White Papers
   Certifications.
6. **Product Compatibility Matrices** — 2606-compatible versions of every commercial
   integration you run.
7. **Interoperability Matrix** — complete supported update paths.

**Customization impact**
- Review *BMIDE for Data Model Design* for template compatibility.
- Consult *What's changed in Teamcenter APIs* on Support Center.
- **Run the Upgrade Assistant ITK Reporter** (`Tc2606_UpgradeAssistantITKReporter.zip`,
  under Additional Downloads → Tools for Teamcenter). It takes compiled server-side
  customization libraries (DLLs) as input, runs a dumpbin-style import-symbol
  analysis, and produces a CSV marking whether rework is needed **immediately**
  (obsolete ITK in use) or **should be planned** (deprecated ITK in use):

  ```
  TcUpgradeAssistantITKReporter.bat -apps=<custom-path> -from_release=14.0 -out=D:\Temp\impact_14_2606.csv
  ```

  It can be run **before you download 2606**, so it belongs at the very start.

  > **Note on tooling overlap.** The Upgrade Assistant analyses *compiled binaries*
  > and is the authoritative Siemens tool. `tools/tc_api_scan.py` in this library
  > analyses *source text*, covers SOA operations as well as ITK, and points at the
  > exact file and line. Use both: the scanner tells you where in the source to go,
  > the Upgrade Assistant confirms what actually links. Neither replaces a rebuild.

**Infrastructure planning**
- Download the **Teamcenter Deployment Reference Architecture** ZIP from Support
  Center. Identify components added to or removed from the architecture since
  TC14, and which machines need OS updates.
- Plan the preproduction architecture: **future Green production environment plus
  an intermediate upgrade server.**
- Size storage for the volume approach you chose (§2.5).

**Deliverable:** impact register, certified-platform gap list, environment design,
and a go/no-go on the target date.

### Stage 1 — Remediate customizations

Runs in parallel with Stage 2. See `03-api-remediation.md`. Gate: clean rebuild
against the 2606 SDK using the compilers named in the Platform and Software
Certification Matrix.

### Stage 2 — Build the new target Green production environment

Per the guide, before touching production data:

1. Install prerequisite software and settings for a new installation.
2. Install **Deployment Center 2606**.
3. Install standalone BMIDE to migrate custom templates to 2606.
4. Create a new 2606 production environment **with the same custom templates as
   the source**. Check `<TC_DATA>\model\master.xml` on TC14 for the installed
   template list.
5. **Update the site ID of the new target to match the source production site ID.**
6. Install the same applications and components as production — but **do not
   install multiple instances** of components like FSC or Active Workspace Gateway
   yet. Scale after the update completes.
7. **Ensure passwords match those from existing Deployment Center instances.**
   Database credentials, FMS keystore passwords and similar are now stored in the
   Deployment Center Vault, and a mismatch breaks the later import.
8. Conduct UAT on this clean environment to confirm the installation is valid.

**This stage requires zero production downtime.** It can start as soon as your
templates are migrated.

### Stage 3 — Prepare Blue and take the clone ⚠️ *first downtime*

1. **Preserve CSP directives.** Open `microservices/gateway/config.json` on the
   current environment and record all CSP directive IDs and their URLs — these must
   be re-entered after the update (in 2606 they live in Deployment Center, see
   `02-breaking-changes.md` §2.6).
2. Terminate Teamcenter sessions.
3. Back up existing Teamcenter data.
4. **Oracle only — clean unused columns.** Upgrade performance degrades when
   dropping columns from a large Teamcenter class. Mark columns unused
   (`ALTER TABLE <table> SET UNUSED <column>`), then reclaim with
   `install -clean_unused_columns <user> <password> dba`. This generates large redo
   logs on big tables — resize redo logs first and hold exclusive schema access.
   **Doing this measurably shortens the upgrade.**
5. Set FMS keys to their default values.
6. **Export the database.** No users connected during export. **Keep the database
   in read-only mode after the export completes** to maintain integrity.
7. Restore FMS keys to previous values.
8. Back up secrets in the Teamcenter Secrets Manager (Teamcenter Vault).
9. Copy volumes (or configure sharing) per §2.5.
10. **Run `InstanceDataSync --action StartSource` against the Blue database.**

**Then let users back on.** From this point Blue is live again and every change is
tracked. **This is the entire first outage** — measured in hours, not days.

⚠️ **From here until switchover: do not perform a BMIDE deployment to Blue.**
Siemens states this explicitly. It is a change freeze on the data model, not on
user work.

### Stage 4 — Build the intermediate environment

Zero production downtime. Users are working in Blue.

1. On the preproduction machine, install prerequisites at the **same software
   versions as production** (this box first replicates TC14, then gets upgraded).
2. Install the **same Deployment Center version as source production**, plus any
   migration hotfixes.
3. Export source config via Quick Deploy:
   ```
   dc_quick_deploy -mode=export -dcurl=<dc-url> -environment=<source-env>
     -exportType=Full -exportfile=<export-file> -dcusername=<user> -dcpassword=<password>
   ```
4. **Edit the exported XML down to a single box** (§2.7), updating hostnames.
5. Import it, keeping the **same environment name and site ID**. If
   `-importenvironmentmetadata` is unsupported in your DC version, set the site ID
   manually in the DC web portal under Environment → Overview.
6. Generate and run deployment scripts to create the single-box corporate server
   with all data model templates.
7. Stop all Teamcenter services, **including `am_read_expression_manager` and
   `revision_config_accelerator`**.
8. Delete the existing database user and tablespaces.
9. Import the production dump (`impdp` for Oracle).
10. Start all Teamcenter services.
11. **Resolve FSC keys.** Either copy `symmetric_key_keystore.jceks` from production
    into `<TC_ROOT>\fsc\fsc_config`, or align the database key using:
    ```
    install_encryptionkeys.exe -u=<admin> -p=<password> -f=list
    install_encryptionkeys.exe -u=<admin> -p=<password> -f=modify -key=<key-from-keystore>
    ```
    Verify with `-f=list`.
12. **Run `InstanceDataSync --action StartDestination`** on the intermediate database
    to clear any restored tracking triggers and advance the UID generator.
13. Synchronize volumes if not sharing. Read the production volume ID from
    `<TC_ROOT>\fsc\backup.xml` → `volumeUiD` → `volumeUid`, update volume ID and path
    in the intermediate FMS master XML, restart FSC.
14. Verify the replica: business objects, preferences, rule tree, stylesheets, and
    service health — Vault running, FMS ping at `:4544/Ping` returning the FSC ID,
    Management Console at `:8083/mgmt/console`, Server Manager pools running,
    AW Gateway ping at `:3000/ping`, Eureka at `:8787/eureka/v2/apps`, a sample
    indexing flow, and Dispatcher.

You now have a working TC14 replica isolated from production.

### Stage 5 — Upgrade the intermediate environment to 2606

Still zero production downtime. **This is the long stage** and the reason
blue-green is worth the effort.

1. Install/update prerequisites to certified versions; upgrade the preproduction
   Deployment Center server.
2. Upgrade Deployment Center on the intermediate server to **Deployment Center 2606**.
3. **Migrate BMIDE templates:** install standalone BMIDE, upgrade the template
   project to the current data model format, create and deploy an updated package.
4. **Update extensions:** copy from source control, refactor for obsolete APIs,
   rebuild against certified compilers, merge file-based changes with new default
   files, migrate or de-customize to use new 2606 capability, and merge custom
   Active Workspace stylesheets, rule tree and admin data with new default templates.
5. Add the 2606 software kit and your BMIDE package to the software repository.
   **Deployment Center automatically detects the platform update and creates new
   `<TC_ROOT>` and `<TC_DATA>` directories** — rename before deploying if you want
   a specific convention.
6. Compare applications against the Green 2606 environment to confirm nothing is
   missing.
7. Deploy, following the **Deployment Sequence** in the Deployment Reference
   Architecture. On Linux, deploy microservices on Docker Swarm or Kubernetes.
8. Configure certificates: import from your CA, provide the full chain to the Vault,
   place machine-specific certificates in each `<TC_ROOT>`.
9. Add administrative client components — two-tier rich client, BMIDE client,
   Management Console.
10. Perform post-update tasks (see `06-data-migration.md`; note `effupgrade`,
    re-index, and any release-note utilities such as
    `add_update_fnd0impactingobjects` for Configurator).
11. **UAT on the updated intermediate environment.**

**Repeat this stage until it is clean.** Failures here cost nothing in production
terms. This is where you get your timings.

### Stage 6 — Move the upgraded database into Green

1. Export the upgraded intermediate database.
2. Stop all Teamcenter services in Green.
3. Delete the database user and tablespaces in Green; import the upgraded dump.
4. Sync volumes into Green (skip if sharing).
5. Update the FMS Master file volume path and ID from the intermediate server's
   `<fsc_installation_path>\backup.xml`.
6. Resolve FSC key mismatch as in Stage 4 step 11.
7. Update the **`Fms_BootStrap_Urls`** preference for the correct FSC URL.
8. Update the **`AWS_FullTextSearch_Solr_URL`** preference for the correct Solr URL.
9. Restart all services and verify as before.
10. Re-enter the CSP directives captured in Stage 3, now via Deployment Center.

### Stage 7 — Final sync and switchover ⚠️ *second downtime — the real cutover*

1. Users log off Blue. **This is the switchover outage.**
2. If you tested with data creation on Green, restore the clone and re-sync (§2.6).
3. **Run `InstanceDataSync --action Sync`** from Blue to Green, with `--recipe`
   pointing at the recipe directory. Back up the destination database first.
4. Synchronize volumes if not sharing.
5. Run required post-upgrade manual utilities named in the release notes.
6. Smoke test — see `07-cutover-checklist.md` Gate C.
7. **Make Green active.** Users log on. Blue can be retired.
8. Run mass client deploy scripts for rich client and other client components.
9. Log on from a non-administrative client machine to verify normal user function.
10. Optionally run `InstanceDataSync --action FinishSource` to remove tracking
    infrastructure from Blue.

**Keep Blue running, untouched, for your agreed rollback window.** That is the
entire point of the pattern. Do not decommission on day one.

---

## Part 4 — Downtime profile

| Stage | Production impact |
|---|---|
| 0 Planning | None |
| 1 Customization remediation | None |
| 2 Build Green | None |
| **3 Clone Blue** | **⚠️ Outage — hours** (export with no users connected) |
| 4 Build intermediate | None — users active in Blue |
| 5 Upgrade intermediate | None — users active in Blue. **The long stage.** |
| 6 Move DB to Green | None — users active in Blue |
| **7 Final sync + switchover** | **⚠️ Outage — the cutover** |

Two bounded outages instead of one long one. On a TC14 estate the in-place
equivalent would hold the system down across the entire schema migration, template
migration, post-update processing and re-index.

**Blue-green does not reduce total work. It moves the work off the outage clock.**

---

## Part 5 — Where blue-green earns its cost, and where it does not

### It is clearly worth it when

- Downtime tolerance is low — multi-site, multi-timezone, or 24/7 manufacturing
- Data volume is large enough that migration alone exceeds any available window
- Re-index duration is significant (frequently the longest single step)
- You need multiple upgrade attempts without production risk
- Rollback confidence must be high — regulated industries, audit obligations

### It is questionable when

- You genuinely have a long window (extended shutdown, plant holiday)
- Data volume is small enough that in-place fits comfortably
- Infrastructure or cloud budget cannot support the extra environments
- The team lacks capacity to operate three environments concurrently

### Costs you must fund

| Cost | Note |
|---|---|
| **Infrastructure** | Green at production scale, plus the intermediate server |
| **Storage** | Full volume copies, unless sharing |
| **Complexity** | Three environments, FSC keys, site IDs, FMS config, volume IDs — each a documented failure mode |
| **Recipe authoring** | If custom classes need transformation |
| **Change freeze on BMIDE** | No data model deployment to Blue between StartSource and switchover |
| **Discipline** | Item ID management during Green testing |

---

## Part 6 — Risks specific to this approach

| Risk | Consequence | Mitigation |
|---|---|---|
| Site ID mismatch between Green and source | Import fails or data is misattributed | Explicitly set site ID (Stage 2 step 5, Stage 4 step 5) |
| Password mismatch with Deployment Center Vault | Credentials fail after import | Verify all passwords match before building Green (Stage 2 step 7) |
| FSC keystore mismatch | Files inaccessible after database restore | `install_encryptionkeys` procedure; verify with `-f=list` |
| Volume ID / path not updated | FMS cannot resolve files | Read from `backup.xml`, update FMS master, restart FSC |
| Item ID collision from Green testing | Duplicate or conflicting IDs at switchover | Restrict creation, separate range, or restore-and-resync |
| Volume writes during Green testing on shared volumes | Production data corruption | **Never modify production volume data from Green** |
| BMIDE deployment to Blue mid-flight | Sync recipes mismatch the schema | Data model freeze from StartSource to switchover |
| Blue database not held read-only after export | Untracked divergence | Enforce read-only per the guide, then StartSource before reopening |
| Custom classes with no recipe | Data lands untransformed | Assess in Stage 0; author recipes early |
| Post-upgrade utilities skipped after sync | Silent data inconsistency | Follow release notes; e.g. `add_update_fnd0impactingobjects` |
| Blue decommissioned too early | Rollback impossible | Hold Blue for the agreed window |

---

## Part 7 — Recommendation for a TC14 customer

**Adopt blue-green with Instance Data Sync.** For this specific starting point the
argument is straightforward:

1. **It is the officially prescribed method.** Platform Update *is* blue-green. You
   are not choosing an exotic route; you are choosing the supported one.
2. **TC14 → 2606 is a direct supported hop**, and IDS explicitly covers 14.0–14.3
   as sources with automatic chronological recipe chaining to 2606.
3. **Nine releases of change means iteration.** Stage 5 will not succeed first time.
   Blue-green makes repeated attempts free in production terms; in-place makes each
   attempt an outage.
4. **The long poles sit outside the window.** Schema migration, template migration,
   post-update processing and full re-index all run while users work in Blue.
5. **Rollback is a running system, not a restore.** With 188 removed APIs and seven
   silent behaviour changes, the probability of discovering something in the first
   days is not low.

**Conditions on that recommendation:**

- Provision the third environment. Attempting a two-box shortcut will fail on FSC
  keys, site IDs or volume configuration.
- Assess recipe requirements for custom classes in Stage 0, not Stage 5.
- Enforce the BMIDE freeze on Blue in writing, with a named owner.
- Rehearse Stages 4–7 end to end at least twice.
- Hold Blue for a defined rollback window agreed before switchover.

**If those conditions cannot be met** — particularly the third environment or the
data model freeze — a conventional in-place upgrade with a long planned window is
the more honest choice. A half-implemented blue-green is worse than a
well-executed in-place upgrade: it carries the complexity cost without delivering
the rollback guarantee.

---

## Appendix — Authority for each fact

| Claim | Source |
|---|---|
| Blue-green is the platform update basis | *Teamcenter Update Using Deployment Center* 2606, ch. 2 |
| Direct update supported from 14.0+ | ch. 2 |
| IDS mechanism, actions, recipes, version support | ch. 3, *Using the Instance Data Sync utility* |
| Volume handling and best practices | ch. 3 |
| Three-environment topology and step detail | ch. 2, ch. 8, ch. 10, ch. 11 |
| Upgrade Assistant ITK Reporter | ch. 3 |
| CSP preservation, unused column cleanup | ch. 8 |
| New `<TC_ROOT>`/`<TC_DATA>` on platform update | ch. 2 |
| Blue-green positioning for older versions | *Introducing Teamcenter 2606*, Siemens blog, 12 Jun 2026 |
| Certification and path authority documents | ch. 2 (Interoperability Matrix, Platform Matrix) |

Items not in the repository and requiring Support Center access: Release Bulletin,
Interoperability Matrix, Platform and Software Certification Matrix, Product
Compatibility Matrices, Deployment Reference Architecture, `README.csv`, and the
Upgrade Assistant download itself.
