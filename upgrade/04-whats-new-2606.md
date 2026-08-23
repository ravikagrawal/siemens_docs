# 04 — New Capability in Teamcenter 2606

The full feature set delivered in 2606, organised by product area, with the
adoption question flagged for each group: does it need enablement, configuration,
or licensing?

Legend: **[ADMIN]** requires administrator configuration · **[OFF]** off by
default · **[AUTO]** available immediately · **[LIC]** may require additional
licensing or component installation.

> This is the 2606 release only. If you are coming from TC14 you also inherit
> everything from 14.1, 14.2, 14.3, 2312, 2406, 2412, 2506 and 2512 — see
> `05-aw-feature-delta.md` for the scale of that.

---

## 4.1 AI and Copilot — the headline theme

| Capability | Notes |
|---|---|
| **Teamcenter AI BOM agent** **[LIC]** | The first Teamcenter *agent*, as distinct from assistive AI. Understands BOM context, proposes changes and executes multi-step workflows with human oversight. Moves impact analysis and change execution from manual assembly to a guided end-to-end process. |
| **AI-powered MBSE in Copilot** **[LIC]** | Checks requirements against configurable INCOSE-based rules, presents suggested changes side by side, flags issues such as conflicting systems of measure. Interprets requirements to help create and link parameters. Recommends test cases and tracks coverage as requirements evolve. |
| **Copilot uses object properties** **[ADMIN]** | Previously Copilot for documents and requirements drew only on file contents and administrator-embedded requirements. It now also returns results from indexed and embedded object properties. **Requires explicit configuration to include object properties.** |
| **Copilot uses metadata** | Draws on issue descriptions, findings and resolutions to surface similar past problem reports and change requests. |
| **Quality & compliance in Copilot** **[LIC]** | AI across audits, problem solving, FMEA, control planning and training, with prompt validation and output guardrails. |
| **Copilot Quality prompt configuration** **[ADMIN]** | Prompts now configurable through the user interface. |
| **Copilot for Structures** | Enhancements in both Structure Management and Engineering BOM Management. |
| **Manufacturing planning in Copilot** **[LIC]** | Generates initial plans from natural-language prompts and specifications, with guardrails keeping changes in scope. |
| **Microsoft 365 Copilot integration** **[LIC]** | Combines PLM context with SharePoint, OneDrive and email. Finds similar problem reports, surfaces due workflow tasks, relates emails to parts and PRs. Data stays governed by its source system. |
| **Knowledge Pulse** **[LIC]** | High-speed APIs to move PLM data — BOMs, relationships — into analytics platforms such as Snowflake, replacing static exports. |

**Adoption note:** almost nothing here is automatic. Budget a discovery phase for
AI capability after go-live, and expect data-governance review before enabling
cross-system Copilot access.

---

## 4.2 Fundamentals and user experience

| Capability | Notes |
|---|---|
| **Home page personalization** **[AUTO]** | Builds on the 2512 dashboard. Users add, delete, replace, resize and move cards. The same configurable home page is expanding across Siemens Xcelerator applications. |
| **New Siemens themes** **[ADMIN]** | Three new designs — framed, light, dark — plus System Theme matching the OS preference. **Framed is the default for new installations.** See `02-breaking-changes.md` §2.13. |
| **Responsive layouts** **[AUTO]** | Dynamic layouts for mobile, tablet, laptop and desktop widths; full functionality in narrow embedded containers; intelligent work-area switching in narrow mode. Relevant if you embed AW in another product. |
| **Deletion assistance** **[ADMIN]** | Shows what is blocking a deletion and how to contact owners of related objects. Auto-selects related data for deletion per administrator-defined rules, with a preview and override before deleting. |
| **Table views retained across groups and roles** **[AUTO]** | Personalised column arrangements now persist when a user switches group or role. Significant for multi-role users. |
| **All properties of loaded types in More Columns** **[AUTO]** | Removes the need for administrators to create individual column-type entries per type. |
| **Inline Relations Tree** **[OFF]** **[ADMIN]** | The Relations Tree can now render as an inline tree in the Explorer Tree view. **Must be activated by an administrator.** |
| **Enhanced copy and paste** **[AUTO]** | Single or multiple objects, same or mixed types, to single or multiple targets in one operation. New **Paste Special** command chooses the relation to the target. Reports a count of copied and pasted objects and lists failures with reasons. |

---

## 4.3 Search and indexing

- **Property-specific search building** **[ADMIN]** — users compose complex searches
  from indexable properties, operators and values, combining conditions.
  Administrators define a preferred-property list.
- **Compound property indexing without a storage class** **[ADMIN]** — the previous
  restriction is lifted; compound properties on business objects without their own
  storage class can now be indexed.

---

## 4.4 Installation, deployment and administration

- **Copy Environment in Deployment Center** — clone a production environment with a
  button rather than a manual export/import sequence. This is the mechanism behind
  practical blue-green upgrades and the fastest route to a dry-run environment.
- **Blue-green deployment** — prepare and sync an upgraded environment in parallel
  with production, then switch over with far less downtime. Siemens positions this
  explicitly as a route for customers on older versions.
- **Ukrainian localization (`uk_UA`)** — full environment support across servers,
  rich client and Active Workspace, with localized data entry and display.
  **A full index of your data is required after adding a language.**
- **Microsoft email OAuth2 configuration** — see `02-breaking-changes.md` §2.12.
- **Teamcenter X self-service administration** — cloud customers gain direct control
  over translations, stylesheets, queries and integration monitoring.

---

## 4.5 Security

- **FIPS-compliant encryption** **[ADMIN]** — for all HTTPS/SSL client and server
  communication, enabled at environment level in Deployment Center. Not supported
  by TEM. All integrations must then be FIPS-compliant.
- **ADA licenses in Active Workspace** **[LIC]** — view and manage ADA license
  members in the Active Admin workspace instead of the rich client. Requires
  installing License Management Active Workspace in Deployment Center.
- **Hardened SSO configuration** — `sso.properties` with login-time hash
  verification. See `02-breaking-changes.md` §2.4.
- **CSP-compliant expression engine** — replaces AngularJS `$parse`. See
  `02-breaking-changes.md` §2.3.
- **CSP directives in Deployment Center** — validated and persisted rather than
  hand-edited. See `02-breaking-changes.md` §2.6.

---

## 4.6 Customization and configuration

- **Configurable folder-deletion bypass** — per folder subtype. See §2.7.
- **Paste visibility honours GRM rules** — see §2.8.
- **Column arrangement management** **[ADMIN]** — view and reset user-level column
  configurations in the table configurator; export column config definitions with
  their scope (Site, Group, Role, Workspace) embedded, so importing into another
  environment applies them to the right scope automatically. This meaningfully
  reduces the effort of promoting configuration between environments.
- **Consistent Add / Add to behaviour** — the object types offered are now the same
  regardless of which Add button is used.
- **Configurable Alternate ID / Alias ID creation dialogs** **[ADMIN]** — choose
  which attributes appear, via stylesheets, matching the rich client approach.
- **BMIDE double-precision formatter separator** — now locale-dependent. See §2.9.

---

## 4.7 Change Management

- **Stacked changes** — merge changes across multiple item revisions. Changes to
  different revisions of the same impacted item are treated as concurrent and can
  be merged.
- **Revert changes to commercial parts in change notices** — revert property
  changes, individual attachment changes or vendor part changes.
  `CM_keep_solution_item_after_revert` controls individual (`true`, revert from
  the Details panel) versus all-at-once (`false`, revert from the assembly).
- **Save As and Replace with change tracking** — create copies with identical
  properties but new IDs; the Change Summary tracks modifications with redlines.
- **`CM_derive_thumbnail_from_relations` default change** — see §2.2.
- **AI workflow template and assignee recommendations** — see §4.15.

---

## 4.8 Classification

- Classification filters now align with class definitions in Active Workspace.
- Suggested values available when authoring classifications.

---

## 4.9 Structure and BOM Management

**Structure Management**
- Copilot for Structures enhancements
- Solution variant enhancements
- Search for elements by their occurrence properties
- `effupgrade` improvements for legacy effectivity migration — see `06-data-migration.md`
- Structure Manager enhancements; refined structure duplication

**Smart Discovery for Structures**
- Filter product structure content using freeform zones
- Bounding box and TruShape file generation enhancements

**Structure Partitions**
- Shortcut-menu filtering of a structure by partition

**Engineering BOM Management**
- Copilot for Structures enhancements; solution variants; refined duplication
- **Color library** — create and manage one; visualize products using color
  appearance definitions; scope color appearances for products and parts; manage
  color themes; generate color parts at assembly level. This completes the Color
  BOM capability introduced in 2512 and pairs with the Digital Reality Viewer.

**Design and EBOM Alignment**
- Guided interface for initial alignments and for alignment updates

**Scale improvements** (from the release announcement): ~80% faster digital-thread
navigation with roughly 10× more content visible at once, and reliable duplication
of structures containing millions of parts.

---

## 4.10 Product Configurator

- **Variant expression migration to JSON** **[ADMIN]** — new utilities migrate
  variant expressions from the nested grid structure to a JSON representation,
  improving authoring, update, delete and load performance. See
  `06-data-migration.md`.
- **Optional family solve performance** **[OFF]** **[ADMIN]** — rank sequence
  numbers and allocation sequences so the solver can decide which
  system-generated defaults to drop. **Enable with the
  `Cfg0EnableOptionalFamilyPriority` site preference.**
- Optional families hidden in Guided mode when constraints exclude them
- Universal viewer for configurator objects
- **Intent-based scoping** — scope a modular configuration using intents and solve;
  each discipline limits the families and features it works with while all work on
  the same agreed configurable product.
- Mathematical functions in free-form constraints

---

## 4.11 Visualization

**Base:** expanded FIPS support · LOD access level enhancements · configurable
point display · body visibility and reference set control · color themes in the 3D
viewer · additional PMI show/hide controls · part properties on hover · automatic
USD conversion for photorealistic digital twins · additional Digital Reality Viewer
capabilities · Product Structure dialog and assembly tree icon enhancements

**Professional:** document unit value persists when exporting to JT

**Mockup:** cancel assembly filters while running · save a Working Set of filters ·
filter assemblies by zones

**Digital Reality Viewer:** enhanced Color BOM visualization; new DRV widget for
Mendix, bringing live 3D into custom applications.

---

## 4.12 Quality

- **Master/variant FMEA**: control what gets compared; show all FMEA elements in one
  action; choose which properties and relations to align after comparing.
  `Failure Cause Actions` renamed to **`Failure Root Cause Actions`** (§2.10).
- **Control and Inspection Planning**: Characteristic Limits Chart; Statistical
  Process Control rules on control methods.
- **Quality Action Management**: additional timeline information fields; skip
  workflow for specific quality action subtypes.
- **Quality Project Management**: identify attachments requiring review in checklists.
- **Training and Qualification**: start requalification before expiry; organise
  qualification profiles by plant and department.
- **SAP inspection planning integration**: create inspection plans in the context of
  a change notice, transfer to SAP at release, link approved results back.

---

## 4.13 Supplier Management, Substance Compliance, Partner Collaboration

- **Bulk supplier onboarding via Excel** — collect supplier and external user
  details in a spreadsheet, import, create accounts in bulk with group and role
  assignment at import. The same process supports ongoing updates. Import reports
  show successes and records needing attention.
- **Bulk import of vendor data** via Excel (Partner Collaboration)
- **Export a product hierarchy to Asset Administration Shell**
- **Collaborative product data sharing with suppliers**; Supplier Connect usability
- **Compliance**: grading modes for compliance checking; bulk categorisation and
  override of vendor parts and materials; dedicated UI for compliance
  configurations; **PFAS regulation support** added alongside REACH, RoHS and
  Conflict Minerals; access control over material composition data.

---

## 4.14 Other product areas

**MBSE** — AI conformance checking and parameter suggestions; improved offline
editing with Microsoft Word; improved bulk trace-link authoring; verification test
report export to Word; **workset functionality now the default for verification
requests**; simplified parameter creation when selecting a KPI; Relations Graph;
MBSE Integration Services diagnostics and more flexible integration definition file
management. Tighter Teamcenter↔Polarion requirements and change orchestration —
changes in Teamcenter cannot complete until related Polarion tasks finish.

**Simulation (SPDM)** — HEEDS process automation diagrams in Active Workspace; view
and replace HEEDS input files from the diagram; pie chart visibility control;
refined relations view; 1D parameters on the Overview tab; ZAERO data management
through Femap; launch history highlighting changed or missing inputs.

**Service Lifecycle Management** — fault codes on Service BOM parts; Find in complex
BOM structures; remove characteristics from the SBOM view; automatically generated
spare definitions for parts with alternates or substitutes; usage-based parts in
SBOM views; service discrepancy reassignment; serialized asset lifecycle control.

**Sustainability** — real-time LCA calculation status monitoring; automatic
lifecycle assessment initiation on release workflows.

**Consumer Packaged Goods** — total product cost and mass calculation; brand
standard assets on item SKUs and higher-level packaging SKUs.

**Data Sharing** — updated reverse mapping in the standalone Advanced Multi-Schema
Exchanger; improved troubleshooting of import and export operations.

**Dispatcher** — manage translation requests directly in Active Workspace.

**Microsoft Office Integration** — integration through **SharePoint Embedded**;
current Office browser editing and co-authoring; both Teamcenter and M365 security
applied across the document lifecycle; larger files, macros and custom properties;
Share or Open in Desktop directly from the edit session. Ukrainian support in
Teamcenter Client for Office.

**Reporting** — report search in the Generate reports panel; Search Data panel first
tab renamed `Results` → `Keyword`; **reports can be added to the Home page**;
**new Power BI integration** with out-of-the-box templates.

**Resource Management** — Manufacturing Resource Library configuration for NX CAM.
NX CAM, CMM Inspection, Additive Manufacturing and Machine Line Planner templates
are now imported automatically during fresh installation *and system updates*, with
unit system and NX version embedded in item IDs and names.

**Product Cost Management** — modernised user interface; updated global energy
prices and rates.

---

## 4.15 Workflow

- **Workflow template suggestions** **[AUTO]** — offered in the Submit to Workflow
  panel when the list has five or more entries. Recommendations are based on user
  profile, role, group, object type, and the recency and frequency of selections by
  similar users, improving with use.
- **Participant suggestions** **[OFF]** **[ADMIN]** — context-based recommendations
  for users, groups, roles and teams. **Off by default; enable with
  `AWA_enable_people_picker_suggestions = true`.**
- **Assignment matrix supports dynamic participants and key roles** — previously
  only group members or resource pools. Assignees can now come from the workflow
  target, the assignment source or elsewhere.
- **Assignment matrix handler default changes** — see `02-breaking-changes.md` §2.1.
  **This is a breaking change, not a feature.**

---

## 4.16 Documentation changes

Worth knowing because it changes where your team looks things up:

- **1st Stop: Teamcenter Documentation Home** replaces the separate "Browse
  Teamcenter help by product area" and "Browse Active Workspace help by product
  area" pages.
- The **Release Bulletin** is now published with the rest of the documentation
  rather than only in the Downloads area of Support Center.
- Previously scattered deliverables — EDA Integration, Easy Plan, Electronic Work
  Instructions, Service Lifecycle Management — are consolidated under a single
  Teamcenter heading.
- Embedded **videos** and **hotspot-linked graphics** are included, but work only in
  the HTML version of the documentation, not the PDFs.
