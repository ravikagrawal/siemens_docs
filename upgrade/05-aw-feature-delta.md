# 05 — Active Workspace Capability Delta: TC14 Baseline → 2606

Derived from `AW_Feature_Matrix.xlsx` in the source repository. Full row-level data
in `data/aw_feature_delta_tc14_to_2606.csv`.

---

## 5.1 Why this document exists

The API report tells you what *breaks*. This tells you what you *gain* — and it is
the number that reframes the upgrade for people funding it. A TC14 site upgrading
to 2606 inherits roughly **1,290 Active Workspace features** it does not have
today.

That volume is also a change-management problem. It is far too much to roll out at
once, and users who get all of it in a single cutover will experience the upgrade
as disruption rather than as capability.

---

## 5.2 Release-by-release volume

| AW release | Ships with | Features added |
|---|---|---:|
| 6.1 | TC 14.1 | 35 |
| 6.2 | TC 14.2 | 130 |
| 6.3 | TC 14.3 | 154 |
| 2312 | TC 2312 | 131 |
| 2406 | TC 2406 | 137 |
| 2412 | TC 2412 | 245 |
| 2506 | TC 2506 | 188 |
| 2512 | TC 2512 | 146 |
| 2606 | TC 2606 | 163 |

**Total from a TC14.0 baseline: ~1,329.** From a TC14.3 baseline, subtract the
6.1–6.3 rows: **~1,010**.

Note that Active Workspace and Teamcenter version numbering merged at 2312 — AW 6.3
was the last release under the old scheme, and there is no AW 7.

---

## 5.3 Where the capability landed

| Category | Area | Features gained |
|---|---|---:|
| Deliver | Quality | 192 |
| Deliver | Manufacturing | 174 |
| Plan | Bill of Materials | 154 |
| Deliver | Service | 145 |
| Develop | Visualization | 100 |
| Foundation | Usability | 83 |
| Plan | Systems | 81 |
| Plan | Process | 69 |
| Deliver | Analytics | 60 |
| Plan | Configuration | 57 |
| Foundation | Industry Solutions | 52 |
| Develop | Simulation | 48 |
| Plan | Requirements | 28 |
| Foundation | Extensibility | 26 |
| Deliver | Suppliers | 16 |
| Develop | Documents | 13 |
| Deliver | Sustainability | 11 |
| Foundation | Deployability | 8 |
| Foundation | Connectivity | 8 |
| Deliver | Materials | 3 |
| Develop | Electronics | 1 |

**Reading this table:** the concentration in Quality, Manufacturing, BOM and
Service means the upgrade's business value is heavily weighted toward those
functions. If none of those groups are in the project's stakeholder set, the
business case is thinner than the headline number suggests — and conversely, if
you have a quality or manufacturing organisation on TC14, they are the sponsors
who will fund this.

Foundation/Usability at 83 is the one that touches everyone: this is where the
day-one "everything looks different" experience comes from.

---

## 5.4 Using the dataset

The workbook's `Automated View` sheet is a working tool in its own right: select
an Active Workspace version and a Teamcenter version in the two dropdowns at the
top and it marks each of its 2,665 features as available or not for that
combination. Useful for answering "will we get X if we go to 2606?" precisely.

The extracted CSV supports the same question from a script:

```bash
# everything gained in a specific area
awk -F, '$3=="Quality"' data/aw_feature_delta_tc14_to_2606.csv

# only what 2606 itself adds
awk -F, '$1=="2606"' data/aw_feature_delta_tc14_to_2606.csv

# count by topic within an area
awk -F, '$3=="Bill of Materials" {print $4}' data/aw_feature_delta_tc14_to_2606.csv | sort | uniq -c | sort -rn
```

Columns: `aw_release`, `category`, `area`, `topic`, `feature`.

The workbook also carries a `WN Links` sheet mapping each release and category to
the corresponding Siemens "What's New in Active Workspace" deck. Those links point
at Siemens Highspot and need appropriate access.

---

## 5.5 Turning this into a phased adoption plan

Do not present 1,290 features to users. Structure adoption in waves:

**Wave 0 — at cutover (unavoidable).**
Foundation/Usability changes users cannot opt out of: new themes, responsive
layouts, table view persistence, copy/paste changes, Add command consistency.
Communicate these before go-live with screenshots. This is the wave that generates
support tickets.

**Wave 1 — 0–3 months, low config, high visibility.**
Home page personalization, enhanced copy/paste with Paste Special, deletion
assistance, property-specific search, reports on the home page. Cheap to enable,
immediately noticeable, builds goodwill for later waves.

**Wave 2 — 3–9 months, department-led.**
The Quality (192), Manufacturing (174), BOM (154) and Service (145) blocks. Each
needs its own business owner, its own configuration and its own training. Treat
each as a small project. Do not attempt more than one at a time per department.

**Wave 3 — 9 months+, needs governance.**
AI and Copilot capability, Knowledge Pulse, M365 Copilot integration. These need
data-governance review, licensing decisions, and a policy on what AI is permitted
to touch. Starting these before Waves 0–2 have settled is a common way to make an
upgrade look like a failure.

A practical rule: enable nothing in Wave 2 or 3 without a named business owner who
will answer user questions about it. Features enabled without an owner become
support burden without benefit.
