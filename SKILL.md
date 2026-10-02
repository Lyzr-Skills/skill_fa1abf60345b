# SKILL: Get Current Period (Current Quarter) & Forecast-vs-Quota Gap Analysis

## Description
Two chained workflows for the Anaplan **Template Sales Forecasting** model:

1. **Part A — Get Current Period**: Retrieves the current planning period, defined by the user as the value of the line item **"Current Quarter"** in the module **"SYS: Time Settings"**.
2. **Part B — Forecast vs Quota Gap Analysis (downstream)**: Using the current period from Part A, compares the **Forecast Call** amount against the **Quota** target from the module **"INP: Input T4 Forecast"**, optionally adjusts the forecast based on the historical forecast trend (forecast accuracy in prior quarters), and computes the **gap to quota** — the amount sellers or sales managers need to bridge.

Use this skill whenever the user asks for the "current period" / "current quarter", or asks about the **gap between forecast and quota**, forecast attainment, quota coverage, or "how much do we need to bridge".

## Model Context
| Property | Value |
|---|---|
| Workspace ID | `dcfa3da2a2544dc9b3ab6fce4df5bbac` |
| Model ID | `4AA3ED8D87AA49FBAD787AE3A4174EA3` |
| Model Name | Template Sales Forecasting |

### Anaplan objects used
| Purpose | Module | Line Item(s) |
|---|---|---|
| Current period | SYS: Time Settings | Current Quarter |
| Quota target | INP: Input T4 Forecast | Quota |
| Forecast amount | INP: Input T4 Forecast | Forecast Call |
| Historical trend (optional) | INP: Input T4 Forecast | Actuals (vs Forecast Call in prior quarters) |

### "INP: Input T4 Forecast" module dimensionality
| Dimension | Kind | Notes |
|---|---|---|
| Time | Time (Month scale) | Quarter labels (e.g. `Q3 FY26`) are valid non-leaf slices |
| Sub Region T4 | List | Leaf level = individual sub-regions (e.g. "Sub Region 1 Northeast") |
| T4 | List | Leaf member observed: `T4` |

No Versions dimension → never ask the user for a version on this module.

## Prerequisites
- MCP session with access to the workspace/model above.
- Tools required: `set_model_context`, `get_model_status`, `sql_schema`, `sql_query` (optionally `catalog_modules` / `catalog_line_items` to re-verify object names).

---

## Part A — Get Current Period

### Step 1 — Bind the model context
Call `set_model_context`:
```json
{
  "workspace_id": "dcfa3da2a2544dc9b3ab6fce4df5bbac",
  "model_id": "4A08859E10F749D2A1C11725890A0908",
  "model_name": "Template Sales Forecasting - Last Version"
}
```

### Step 2 — Verify model readiness (handle cold-start)
The model may be closed/loading (states seen in practice: `allocating`, `loading`). If any model-scoped call returns `CORE_TIMEOUT` or the model state is not `ready`:
1. Call `get_model_status`.
2. If `ready` is `false`, wait per the `retry.after_seconds` guidance (typically 5–30 seconds) and poll `get_model_status` again.
3. Proceed only when `state` = `ready`.

### Step 3 — Fetch the schema for the target line item
Call `sql_schema` with:
```json
{
  "included_objects": { "SYS: Time Settings": ["Current Quarter"] }
}
```
Expected schema result:
- Table name: `template sales forecasting.sys: time settings`
- Column: `current quarter` (VARCHAR)
- No Time or Versions dimension columns → no WHERE clause constraints needed.

### Step 4 — Query the value
Call `sql_query` with the SAME `included_objects` map:
```json
{
  "included_objects": { "SYS: Time Settings": ["Current Quarter"] },
  "query": "SELECT \"current quarter\" FROM \"template sales forecasting.sys: time settings\""
}
```

### Step 5 — Present the result
Report the returned value as the current period (e.g. **Q3 FY26**) and carry it into Part B as `:qtr`.

---

## Part B — Forecast vs Quota Gap Analysis (downstream)

**Business logic**: The gap between forecast and quota is the amount the sellers or sales managers need to bridge for the current quarter.

```
Gap to Quota            = Quota − Forecast Call
Trend Factor (optional) = avg( Actuals / Forecast Call ) over recent closed quarters
Adjusted Forecast       = Forecast Call × Trend Factor
Adjusted Gap            = Quota − Adjusted Forecast
Forecast Attainment %   = Forecast Call / Quota
```

### Step 6 — Fetch the schema for the forecast module
Call `sql_schema` with:
```json
{
  "included_objects": { "INP: Input T4 Forecast": ["Quota", "Forecast Call"] }
}
```
Expected table: `template sales forecasting.inp: input t4 forecast` with:
- Dimension columns: `time`, `sub region t4`, `t4` (each with a matching `*_is_leaf` BOOLEAN flag)
- Measure columns: `quota` (DOUBLE), `forecast call` (DOUBLE)

Add `"Actuals"` to the line-item list when computing the historical trend (Step 8).

### Step 7 — Query total Quota and Forecast Call for the current quarter
Call `sql_query` (same `included_objects` as Step 6), slicing time to the current quarter from Part A and requesting leaf on both list dimensions:
```json
{
  "included_objects": { "INP: Input T4 Forecast": ["Quota", "Forecast Call"] },
  "query": "SELECT SUM(\"quota\") AS total_quota, SUM(\"forecast call\") AS total_forecast FROM \"template sales forecasting.inp: input t4 forecast\" WHERE \"time\" = :qtr AND \"sub region t4_is_leaf\" = TRUE AND \"t4_is_leaf\" = TRUE",
  "parameters": { "qtr": "Q3 FY26" }
}
```

### Step 8 — (Optional) Compute the historical forecast trend factor
For each of the 1–3 most recent **closed** quarters (e.g. `Q2 FY26`, `Q1 FY26`), run **one query per quarter** (the SQL engine rejects `time IN (...)` — see Gotchas):
```json
{
  "included_objects": { "INP: Input T4 Forecast": ["Quota", "Forecast Call", "Actuals"] },
  "query": "SELECT SUM(\"forecast call\") AS total_forecast, SUM(\"actuals\") AS total_actuals FROM \"template sales forecasting.inp: input t4 forecast\" WHERE \"time\" = :qtr AND \"sub region t4_is_leaf\" = TRUE AND \"t4_is_leaf\" = TRUE",
  "parameters": { "qtr": "Q2 FY26" }
}
```
Then:
```
Trend Factor = average over prior quarters of (total_actuals / total_forecast)
Adjusted Forecast = current-quarter Forecast Call × Trend Factor
```
If prior-quarter data is missing or forecast is zero, skip adjustment and use the raw Forecast Call (Trend Factor = 1.00). Note: in this model, closed quarters show Forecast Call = Actuals (factor 1.00), so the adjustment is currently neutral — still compute it, as this can change as data evolves.

### Step 9 — Compute and present the gap
```
Gap to Quota  = total_quota − total_forecast            (raw)
Adjusted Gap  = total_quota − adjusted_forecast          (trend-adjusted)
Attainment %  = total_forecast / total_quota × 100
```
Present as a summary table, e.g.:

| Metric | Value |
|---|---|
| Current Quarter | Q3 FY26 |
| Quota | 5,658,070 |
| Forecast Call | 319,425 |
| Trend Factor | 1.00 |
| Adjusted Forecast | 319,425 |
| **Gap to Quota (to bridge)** | **5,338,645** |
| Forecast Attainment % | 5.6% |

State clearly: "The gap between forecast and quota is **<gap>** — this is the amount sellers/sales managers need to bridge in <current quarter>."

### Step 10 — (Optional) Per-sub-region breakdown for prioritization
To show sellers/managers where the gap is concentrated:
```json
{
  "included_objects": { "INP: Input T4 Forecast": ["Quota", "Forecast Call"] },
  "query": "SELECT \"sub region t4\", SUM(\"quota\") AS quota, SUM(\"forecast call\") AS forecast, SUM(\"quota\") - SUM(\"forecast call\") AS gap FROM \"template sales forecasting.inp: input t4 forecast\" WHERE \"time\" = :qtr AND \"sub region t4_is_leaf\" = TRUE AND \"t4_is_leaf\" = TRUE GROUP BY \"sub region t4\" ORDER BY gap DESC",
  "parameters": { "qtr": "Q3 FY26" }
}
```
Rank sub-regions by gap descending to highlight where bridging effort is most needed.

---

## Notes & Gotchas
- The module name in the user's preference may contain a trailing space ("SYS: Time Settings "). Use the canonical name **"SYS: Time Settings"** when calling tools.
- "Current Quarter" is a text/VARCHAR value (e.g., "Q3 FY26"), not a native Time period — read it via SQL, not via time metadata.
- `included_objects` is REQUIRED and must be identical on both `sql_schema` and `sql_query`.
- **SQL slicing rule**: every dimension column must be either sliced with `=` or constrained to leaf via `<dim>_is_leaf = TRUE`. Violations return: *"Only queries that slice or request leaf for every dimension are permitted"*.
  - `time IN ('Q1 FY26','Q2 FY26')` is NOT accepted as a slice — run one query per quarter instead.
  - `SELECT DISTINCT` on dimension columns without leaf constraints also fails.
- The module's time scale is Month, but quarter labels (e.g. `Q3 FY26`) work as time slices and return quarter-aggregated values.
- Related line items available in "INP: Input T4 Forecast" if deeper analysis is requested (verify before use): `Forecast Attainment %`, `Forecast Call To Go`, `LW Forecast Call`, `1 Week Change`, `Won`, `Commit`, `Upside`, `Pipeline`, `Total Pipeline`, `Anaplan Prediction`, `Actuals`.
- Do not invent or extrapolate Anaplan object names; use exactly the module and line item names above.

## Last Validated Results (as of skill update)
| Item | Value |
|---|---|
| Current Quarter | **Q3 FY26** |
| Q3 FY26 Total Quota | 5,658,070.03 |
| Q3 FY26 Total Forecast Call | 319,424.61 |
| Historical Trend Factor (Q1+Q2 FY26 Actuals/Forecast) | 1.00 |
| Adjusted Forecast | 319,424.61 |
| **Gap to Quota (amount to bridge)** | **5,338,645.42** |
| Forecast Attainment % | 5.65% |
