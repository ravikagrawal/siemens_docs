# 03 — Customization API Remediation

Source: `TCCustomization_APIs_Report_1400_2606.csv` (Siemens), normalised into
`data/*.csv` and `data/api_signatures.json`.

---

## 3.1 The four categories

Siemens splits the report into deprecated and obsoleted, for ITK and for SOA
services. The distinction is what determines your deadline.

| Category | Count | Meaning for a TC14→2606 upgrade |
|---|---|---|
| **Obsoleted ITK** | 188 | **Already gone.** Code will not compile or link on 2606. |
| **Obsoleted SOA operations** | 54 | **Already gone.** Client calls will fail. |
| **Deprecated ITK** | 170 | Still present in 2606. Removal targeted 2712 (133) or 2912 (37). |
| **Deprecated SOA operations** | 244 | Still present. Removal targeted 2712 (178) or 2912 (66). |

**When the removals landed:**

| Release | ITK removed | SOA removed |
|---|---|---|
| 2312 | 184 | 24 |
| 2406 | — | 1 |
| 2412 | 3 | — |
| 2512 | 1 | 29 |

184 of 188 ITK removals happened at 2312 — the first release after the 14.x line.
This is why TC14 is a hard baseline: you cross that boundary in a single hop.

---

## 3.2 Obsoleted ITK by library — where the damage is

| Library | Removed | No replacement | Character of the change |
|---|---:|---:|---|
| `libappr` | 48 | 48 | Whole `APPR_*` attribute-mapping surface withdrawn |
| `libdmi` | 26 | 26 | `DMI_*` markup / viewer interface withdrawn |
| `libbom` | 21 | 0 | Mechanical rename to `AOM_*` / `PROP_*` equivalents |
| `libbooleanmath` | 21 | 0 | `CFG_*` tokens renamed to `CFG_PROFILE_*` |
| `libqry` | 19 | 19 | Legacy Autonomy search error macros withdrawn |
| `libtccore` | 9 | 0 | Participant and AOM API modernisation |
| `libcfm` | 7 | 1 | Effectivity moved from `CFM_*` to `PS_occ_eff_*` |
| `libps` | 6 | 3 | Error-token renames |
| `libproperty` | 5 | 0 | `ATTRMAP_*` consolidated onto `*2` variants |
| `libptn0partition` | 5 | 1 | Partition tokens |
| others | 21 | ~4 | Scattered |

**87 of 188** have a documented replacement. **101 have none** — those are
redesign work, not port work. Budget them differently.

---

## 3.3 Mechanical replacement families

These are safe, high-volume, largely search-and-replace. Do them first to shrink
the register.

### `CFG_*` → `CFG_PROFILE_*` (21 macros, `booleanmath/cfg_tokens.h`)

Deprecated back in 11.5, removed at 2312. Pure prefix rename:

```
CFG_active_constraint_type_filter_key  → CFG_PROFILE_active_constraint_type_filter_key
CFG_active_intent_filter_key           → CFG_PROFILE_active_intent_filter_key
CFG_apply_config_constraints_key       → CFG_PROFILE_apply_config_constraints_key
CFG_apply_defaults_key                 → CFG_PROFILE_apply_defaults_key
CFG_compute_all_problems_key           → CFG_PROFILE_compute_all_problems_key
...
```

Full list: `data/obsoleted_itk_apis.csv`, filter `library == libbooleanmath`.

### `BOM_line_ask_attribute_*` → `AOM_ask_value_*` (`bom/bom.h`)

```
BOM_line_ask_attribute_double   → AOM_ask_value_double
BOM_line_ask_attribute_int      → AOM_ask_value_int
BOM_line_ask_attribute_logical  → AOM_ask_value_logical
BOM_attribute_mode_t            → PROP_value_type_t
```

Not purely textual — `AOM_*` operates on the object tag rather than the BOM line
handle, so each call site needs the tag resolved. Mechanical but not blind.

### `ITEM_rev_*_participant` → `PARTICIPANT_*` (`tccore/item.h`)

```
ITEM_rev_add_participant     → PARTICIPANT_add_participant
ITEM_rev_ask_participants    → PARTICIPANT_ask_participants
ITEM_rev_remove_participant  → PARTICIPANT_remove_participant
```

### `ATTRMAP_resolve_*` → `ATTRMAP_resolve_mapping2` / `ATTRMAP_resolve_mappings2`

Four separate entry points collapse onto two `*2` variants with a different
signature. Check argument order at every call site.

### `CFM_effectivity_*` → `PS_occ_eff_*`

```
CFM_effectivity_ask_date_ranges → PS_occ_eff_ask_date_range
CFM_effectivity_ask_id          → PS_occ_eff_ask_id
CFM_effectivity_create          → PS_occ_eff_create
```

This one is **not just an API rename** — it is the code-level half of the
effectivity data migration described in `06-data-migration.md`. Coordinate the
code change with the `effupgrade` data run.

### AOM save and bulk-save

```
AOM_UIF_ask_values        → AOM_ask_displayable_values      (removed, 2312)
AOM_bulk_save_instances   → AOM_bulk_save_instances_partial_errors  (removed, 2312)
AOM_save                  → AOM_save_with_extensions | AOM_save_without_extensions
                                                             (deprecated, removal 2712)
```

`AOM_save` is *deprecated*, not removed — it still works in 2606. Fix it anyway;
it is one of the most widely used calls in Teamcenter customization and you do not
want to discover its footprint under 2712 time pressure. The choice between the
`with_extensions` and `without_extensions` variants is a behavioural decision:
extensions fire your own and OOTB post-actions. Read each call site.

---

## 3.4 The no-replacement clusters — redesign required

### `APPR_*` (48 functions, `appr/appr.h`, deprecated 13.1, removed 2312)

The entire legacy attribute-mapping ITK surface — `APPR_ask_all_attr_values`,
`APPR_ask_appr_root`, `APPR_ask_attr_list`, `APPR_ask_attr_mapping`,
`APPR_ask_attr_mapping_as_string`, and 43 more — was withdrawn with **no
replacement**.

If your TC14 site uses `APPR_*`, this is the largest single redesign item in the
upgrade. Typical usage is legacy attribute mapping to ERP or to CAD integrations.
Rebuild on the current attribute-mapping framework (`ATTRMAP_*2` variants) or on
the appropriate integration-specific API. Scope this before you commit to a date.

### `DMI_*` (26 functions, `dmi/dmi.h`, deprecated 13.2, removed 2312)

Markup and viewer interface: `DMI_ask_dataset_ref`, `DMI_ask_image_markups`,
`DMI_ask_markup_types2`, `DMI_ask_tempmarkup_type2`, and others. No replacements.
Custom markup handling must be rebuilt against current visualization interfaces.

### `QRY_autonomy_*` (19 macros, `qry/qry_errors.h`, deprecated 13.2, removed 2312)

Error tokens from the retired Autonomy full-text search engine. If your code
catches these, it is catching errors that can no longer be raised. Usually the
remediation is deletion of dead error-handling branches rather than replacement —
but confirm the surrounding search code is not also targeting the old engine.

---

## 3.5 Obsoleted SOA operations (54)

| Service library | Removed ops | Notes |
|---|---:|---|
| `S2clSoaServices` | 20 | Largest single cluster |
| `TcSoaProjectmanagement` | 11 | Project/program management clients |
| `Mdo0SoaMdomanagement` | 4 | |
| `Crt1SoaValidationcontractaw` | 3 | Analysis→Verification request rename |
| `Fnd0SoaUiconfig` | 3 | UI configuration |
| `Mdl0SoaModelcore` | 3 | |
| `TcSoaCalendarmanagement` | 3 | |
| others | 7 | |

Two common patterns in the replacement guidance:

**Namespace bump** — the operation still exists under a newer dated namespace:

```
Asp0::Services::Asp0aspect::_2015_07::Aspectmanagement::expandAspects
  → use expandAspects from the _2018_06 namespace
```

Usually a low-effort fix: change the namespace, verify the request/response
structures, which often gained fields.

**Rename with semantic shift** — e.g. in `Crt1SoaValidationcontractaw`,
`createAnalysisRequest` → `createVerificationRequest`, `getAnalysisRequestInfo` →
`getVerificationRequestInfo`. This tracks the product-level rename of Analysis
Requests to Verification Requests, so the data model changed too, not just the
operation name.

**No replacement** — a smaller set, including
`Att0::...::synchronizeMeasuableAttributes` (removed 2512), where the underlying
functionality is gone entirely.

---

## 3.6 Deprecated but not yet removed — do these now

**Removal targeted at 2712** (the release after next): 133 ITK artifacts + 178 SOA
operations. **Targeted at 2912**: 37 ITK + 66 SOA.

Highest-concentration areas in the deprecated ITK set, by header module:

| Module | Count |
|---|---:|
| `Cls0classification` | 18 |
| `Mdc0mdc` | 17 |
| `tccore` | 12 |
| `ps` | 11 |
| `ics` | 9 |
| `rdv` | 8 |
| `Cfg0configurator` | 7 |
| `pom` | 6 |

And in the deprecated SOA set:

| Service library | Count |
|---|---:|
| `TcSoaClassification` | 29 |
| `TcSoaCore` | 29 |
| `Mdc0SoaMdconnectivity` | 21 |
| `TcSoaCad` | 17 |
| `TcSoaStructuremanagement` | 15 |
| `Mei0SoaMesinteg` | 12 |

Classification is the standout: 18 deprecated ITK artifacts plus 29 deprecated SOA
operations, reflecting the ongoing move from the legacy `ICS` classification model
to `Cls0` Advanced Classification. If you run classification customizations on
TC14, treat this as a modernisation project rather than a set of API swaps, and
read the Advanced Classification deployment guide in the source repository.

Note that 29 of the 244 deprecated SOA operations have **no replacement at all** —
those become redesign items with a 2712/2912 deadline. Find them now:

```bash
grep -i "no replacement" data/deprecated_soa_operations.csv
```

---

## 3.7 Suggested remediation sequence

1. **Scan.** `python3 tools/tc_api_scan.py <src> --format csv --out impact.csv`
2. **Triage.** Split into: mechanical (§3.3), redesign (§3.4), namespace bump
   (§3.5), and deferred-but-do-anyway (§3.6).
3. **Kill the mechanical items first.** They are the bulk of the count and almost
   none of the risk, and clearing them makes the real number visible to
   stakeholders.
4. **Scope the redesign items properly.** `APPR_*` and `DMI_*` clusters are design
   work with requirements attached. If they are large, this is where your date
   comes from.
5. **Rebuild against the 2606 SDK** and let the compiler find what the scanner's
   regex missed — macro indirection, generated code and string-built service names
   will slip past a text scan.
6. **Re-scan after remediation** and gate the build on exit code 0.

---

## 3.8 Wiring the scanner into CI

```bash
# fail the build on any already-removed API
python3 tools/tc_api_scan.py "$SRC_ROOT" --severity BLOCKER
# exit 1 => blocker present
```

Once blockers are clear, tighten to warnings so nothing new creeps in:

```bash
python3 tools/tc_api_scan.py "$SRC_ROOT" --format csv --out impact.csv
test "$(wc -l < impact.csv)" -eq 1   # header only
```

**Scanner limits — read these.** It is a text scanner: it matches whole-word
symbol names in source files and skips obvious comment lines. It will not catch
symbols built at runtime, reached through macros, or produced by code generators;
it can flag a same-named symbol in unrelated code. It is a triage instrument that
tells you where to look, not a compiler. The compiler is the authority.
