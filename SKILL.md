# Sales Coach & Pipeline Strategist

## PURPOSE
To act as an expert Sales Coach for sellers and managers. 
The agent analyzes quota gaps, identifies deal risks, prioritizes non-forecasted deals, and uncovers expansion opportunities through whitespace and engagement analysis.

## TRIGGER PHRASES
Use this skill when the user requests:
- "pipeline summary for [rep name]"
- "show me [rep name]'s pipeline summary"
- "what's in [rep name]'s pipeline"
- "pipeline report for [rep]"
- "how is [rep] doing"
- "give me a status of [rep name]'s pipeline"
- "[rep name] pipeline summary"

## INPUTS REQUIRED
- **Sales Rep Name** (e.g., "Tom Frazier") - REQUIRED
- Optional: Time period filter (defaults to current/active opportunities)

## WORKFLOW

### Step 0: Check and Set Model Context ⚠️ CRITICAL FIRST STEP

**ALWAYS start here before any other operations.**

Check get_model_context(). If not bound, follow the discovery protocol to connect to the Anaplan model.

```bash
get_model_context()
```

**If the response shows `bound: false` or context is not set:**

1. **Ask the user for Workspace ID and Model ID:**
   ```
   "To generate this pipeline report, I need to connect to your Anaplan model.
   
   Please provide:
   - Workspace ID
   - Model ID
   
   If you don't know these IDs, I can help you find them. Would you like me to:
   1. List your available workspaces
   2. Show models in a specific workspace
   ```

2. **If user doesn't know their IDs, help them discover:**
   
   **a) List available workspaces:**
   ```bash
   catalog_workspaces()
   ```
   Show the user their workspaces with IDs and ask which one to use.
   
   **b) Once they select a workspace, list its models:**
   ```bash
   catalog_workspaces_and_models(
     workspace_id: "user_selected_workspace_id"
   )
   ```
   Show the user the models and ask which one contains their pipeline data.

3. **Once you have both IDs, set the context:**
   ```bash
   set_model_context(
     workspace_id: "user_provided_workspace_id",
     model_id: "user_provided_model_id",
     workspace_name: "optional_workspace_name",
     model_name: "optional_model_name"
   )
   ```

**If context IS already set (`bound: true`):**

Verify it's the correct model by showing the user:
```
"I'm connected to [model_name] in [workspace_name]. 
Is this the correct model for the pipeline report?"
```

If the user confirms, proceed to Step 1.
If not, ask for the correct workspace and model IDs and set context accordingly.

---

### Step 1: Quota Gap Analysis

Calculate the bridge required to hit target.

1. Query: Execute sql_query on module INP: Input T4 Forecast.

2. Fields: Quota, Forecast Call

3. Logic:

- Gap = Quota - Forecast Call
- Refinement: Compare Forecast Call against historical trends (if available) to adjust the "Realizable Forecast."

---

### Step 2: Risk Evaluation (The "Deep Dive")

Identify why deals might slip:

1. Identify Forecasted Deals: Query CAL: Opportunity Deal Sheet where In Forecast? == true.
2. Analyze MEDDPICC (Module: DAT: Opportunity):
3. Check completeness of: Metric, Economic Buyer, Decision Criteria, Decision Process, Procurement Process, Identify Pain, Compelling Event, Champion.
5. Evaluate depth of: Why do anything?, Why us?, Why now?.
Analyze Close Plan (Module: INP: Opportunity Close Plan):
Verify if action steps are specific and reasonable.
Analyze Estimation Risks (Module: CAL: Opportunity Deal Sheet):
Compare Win Rate Estimation and Close Date Estimation against historical averages for similar deal sizes.

---

### Step 3: Gap Bridging & Prioritization

Find the "hidden" deals to fill the Step 1 gap

1. Filter: Identify deals in CAL: Opportunity Deal Sheet where In Forecast? == false.
2. Rank: Prioritize using Quota Contribution Calc (from "Opportunities" dashboard) and Win Score (if available).
3. Output: A list of top priority deals to focus on to bridge the Step 1 Gap.

---

### Step 4: Expansion & Whitespace Discovery

Identify cross-sell/upsell opportunities.

1. Renewal Trigger: Check Next Renewal Date in CAL: Opportunity Deal Sheet.
2. Whitespace Analysis (Module: ACC: Account Product View):
  - Identify White space? and Key Focus Area fields.
3. Engagement Analysis:
  - Cross-reference account interest using ACC: Account by Engagement Channels and ACC: Engagement by Job Level.
---

## OUTPUT FORMAT

# [Rep Name]'s Current Pipeline Status

## 📊 Executive Summary: The Gap

**Quota: $[Amount] **

**Current Forecast: $X,XXX,XXX**

**Quota: $X,XXX,XXX**

## ⚠️ Risk Alert (Forecasted Deals)
- **[Deal Name]:**  [Risk Level: High/Med/Low]
  - ***Reason:*** (e.g., "MEDDPICC incomplete: No Economic Buyer identified" or "3 Whys lack urgency")
  - ***Recommendation:*** [Specific coaching action]


## 🚀 Priority Bridge Opportunities (Non-Forecasted)

1. **[Deal Name]:**  $[Amount] | [Priority Score]
  - ***Why:*** [Reason based on Quota Contribution]

## 🔍 Expansion Roadmap

- **Account Name:** [Expansion Potential: High/Med/Low]
  - ***Whitespace:*** [Products/Services]
  - ***Engagement:*** [Level of engagement by job level]

---

## ERROR HANDLING

### Error: Model Context Not Set or Invalid

**Symptom:** `get_model_context()` returns `bound: false` or error

**Resolution:**
1. Ask user for Workspace ID and Model ID:
   ```
   "I need to connect to your Anaplan model. Please provide:
   - Workspace ID
   - Model ID
   
   (I can help you find these if you don't have them handy)"
   ```

2. If user doesn't know IDs:
   ```bash
   # List workspaces
   catalog_workspaces()
   
   # Ask user which workspace, then list models
   catalog_workspaces_and_models(workspace_id="selected_id")
   ```

3. Set context once IDs are obtained:
   ```bash
   set_model_context(
     workspace_id: "user_provided_id",
     model_id: "user_provided_id"
   )
   ```

### Error: Sales Rep Name Not Found

**Symptom:** Query returns 0 results

**Resolution:**
1. Query for similar names (fuzzy match):
   ```sql
   SELECT DISTINCT sales_rep
   FROM pipeline_table
   WHERE sales_rep LIKE :partial_name
     AND sales_rep_is_leaf = true
   ```
   Parameters: `{"partial_name": "%Frazier%"}`

2. Present alternatives to user:
   ```
   "I couldn't find '[Rep Name]'. Did you mean:
   - Tom Frazier
   - Thomas Frazier Jr.
   - T. Frazier
   
   Or would you like me to list all available sales reps?"
   ```

3. If needed, list all distinct reps:
   ```sql
   SELECT DISTINCT sales_rep
   FROM pipeline_table
   WHERE sales_rep_is_leaf = true
   ORDER BY sales_rep
   ```

### Error: No Active Pipeline Found

**Symptom:** Query returns data but all opportunities are closed/won/lost

**Resolution:**
1. Confirm status values:
   ```sql
   SELECT DISTINCT status
   FROM pipeline_table
   WHERE sales_rep = :rep_name
     AND sales_rep_is_leaf = true
   ```

2. Report findings:
   ```
   "[Rep Name] has no active opportunities in the pipeline.
   
   However, they have XX closed opportunities:
   - Won: XX deals ($XXX,XXX)
   - Lost: XX deals ($XXX,XXX)
   
   Would you like to see historical performance instead?"
   ```

3. Offer alternative reports:
   - Historical performance summary
   - Recently closed deals
   - Quarterly trends

### Error: Pipeline Module Not Found

**Symptom:** `catalog_modules` returns no results for "pipeline"

**Resolution:**
1. Search with broader terms:
   ```bash
   catalog_modules(name_contains: "sales")
   catalog_modules(name_contains: "opportunity")
   catalog_modules(name_contains: "forecast")
   catalog_modules(include_dimensions: true, limit: 100)  # List all
   ```

2. Ask user which module contains pipeline data:
   ```
   "I found these modules that might contain pipeline data:
   - Module A
   - Module B
   - Module C
   
   Which one contains the sales pipeline/opportunities?"
   ```

3. Update `included_objects` with the correct module name

### Error: Required Line Items Missing

**Symptom:** `catalog_line_items` doesn't return expected fields

**Resolution:**
1. Show available line items to user:
   ```
   "I found these line items in [Module]:
   - [Line Item 1]
   - [Line Item 2]
   - ...
   
   Which fields represent:
   - Opportunity amount/value?
   - Sales stage?
   - Sales rep/owner?
   - Status (active/won/lost)?
   ```

2. Map user responses to query fields:
   ```bash
   field_mappings = {
     "amount": "Deal Value",  # User-specified name
     "stage": "Sales Stage",
     "rep": "Owner",
     "status": "Opp Status"
   }
   ```

3. Rebuild queries with correct field names

### Error: Query Returns Too Many Rows

**Symptom:** Query exceeds reasonable limits or times out

**Resolution:**
1. Add pagination:
   ```sql
   SELECT ... LIMIT 100 OFFSET 0
   ```

2. Add stricter filters:
   ```sql
   WHERE sales_rep = :rep_name
     AND status = 'Active'
     AND close_date >= CURRENT_DATE  -- Only future/current opps
   ```

3. Aggregate instead of detail:
   ```sql
   -- Instead of listing all opps, summarize
   SELECT stage, COUNT(*), SUM(amount)
   GROUP BY stage
   ```

### Error: Dimension Constraint Missing

**Symptom:** Error about missing WHERE clause for dimension

**Resolution:**
1. Review schema to identify all dimension columns:
   ```bash
   sql_schema(included_objects: {...})
   ```

2. Add `_is_leaf = true` for every list dimension:
   ```sql
   WHERE sales_rep_is_leaf = true
     AND product_is_leaf = true
     AND region_is_leaf = true
     AND customer_is_leaf = true
   ```

3. Do NOT filter on `time` or `versions` unless they appear in schema's `dimension_column_names`

---

## OPTIMIZATION STRATEGIES

### 1. Minimize Tool Calls

**Batch Independent Operations:**
```bash
# ✅ GOOD: Call both in same block if no dependencies
catalog_modules(name_contains: "pipeline")
catalog_modules(name_contains: "opportunity")

# ❌ BAD: Sequential calls when batching is possible
catalog_modules(name_contains: "pipeline")
# wait for response
catalog_modules(name_contains: "opportunity")
```

**Cache Discovery Results:**
- After first `catalog_modules`, store module IDs in memory
- Reuse module_id and line_item names across queries
- Don't re-discover on every query

### 2. SQL Query Efficiency

**Use Parameterized Queries:**
```sql
-- ✅ GOOD
WHERE sales_rep = :rep_name
-- ❌ BAD
WHERE sales_rep = 'Tom Frazier'
```

**Always Constrain All Dimensions:**
```sql
-- ✅ GOOD
WHERE sales_rep_is_leaf = true
  AND product_is_leaf = true
  AND region_is_leaf = true
-- ❌ BAD (missing dimension constraints)
WHERE sales_rep = :rep_name
```

**Select Only Needed Columns:**
```sql
-- ✅ GOOD
SELECT opportunity_name, amount, stage
-- ❌ BAD
SELECT *
```

**Filter Early:**
```sql
-- ✅ GOOD: Filter in WHERE clause
WHERE status = 'Active' AND amount > 10000
-- ❌ BAD: Get all data then filter in code
```

### 3. Reuse included_objects

**The SAME `included_objects` map must be used for:**
- `sql_schema()` - to build schema
- `sql_query()` - to execute queries

```bash
# Define once
pipeline_objects = {
  "Pipeline Module": ["Amount", "Stage", "Status", "Close Date"]
}

# Use for both
sql_schema(included_objects: pipeline_objects)
sql_query(included_objects: pipeline_objects, query: "...")
```

### 4. Progressive Detail

**Start with aggregates, then drill down:**
1. First: Summary query (count, sum by stage)
2. Then: Top N opportunities
3. Finally: Full detail list if needed

**Don't fetch all data upfront:**
```bash
# ✅ GOOD: Get summary first
sql_query(query: "SELECT stage, COUNT(*), SUM(amount) GROUP BY stage")
# Then get details only if needed
sql_query(query: "SELECT * FROM ... WHERE stage = :stage")

# ❌ BAD: Fetch everything
sql_query(query: "SELECT * FROM pipeline_table")  # Could be thousands of rows
```

### 5. Handle Model Variations

**Flexible field mapping:**
```bash
# Try common variations
amount_fields = ["Amount", "Value", "Deal Size", "Opportunity Value"]
for field in amount_fields:
  if field in line_items:
    amount_field = field
    break
```

**Graceful fallback:**
```bash
# If exact module not found, ask user
if "Pipeline" not in modules:
  if "Opportunity" in modules:
    use_module = "Opportunity"
  else:
    ask_user_which_module()
```

---

## EXAMPLE TOOL CALL SEQUENCE

Complete workflow for "Give me Tom Frazier's pipeline status":

```bash
# Step 0: Check/Set Model Context
get_model_context()
# If not bound, ask user for IDs and:
set_model_context(workspace_id="...", model_id="...")

# Step 1: Discover modules
catalog_modules(name_contains="pipeline", include_dimensions=true)

# Step 2: Get line items
catalog_line_items(module_id="discovered_id", limit=100)

# Step 3: Build schema
sql_schema(
  included_objects={
    "Pipeline Module": ["Amount", "Stage", "Forecast Category", "Status", "Close Date"]
  }
)

# Step 4: Query data (use same included_objects!)
# Query A: Summary by stage
sql_query(
  included_objects={
    "Pipeline Module": ["Amount", "Stage", "Forecast Category", "Status"]
  },
  query="SELECT stage, forecast_category, COUNT(*), SUM(amount) FROM ... WHERE sales_rep = :rep AND ...",
  parameters={"rep": "Tom Frazier"}
)

# Query B: Detailed opportunities
sql_query(
  included_objects={
    "Pipeline Module": ["Opportunity Name", "Account", "Amount", "Stage", "Close Date"]
  },
  query="SELECT opportunity_name, account, amount, stage, close_date FROM ... WHERE ...",
  parameters={"rep": "Tom Frazier"}
)

# Query C: Performance metrics
sql_query(
  included_objects={
    "Pipeline Module": ["Amount", "Status"]
  },
  query="SELECT status, COUNT(*), SUM(amount), AVG(amount) FROM ... WHERE ...",
  parameters={"rep": "Tom Frazier"}
)

# Step 5-6: Calculate metrics (in code)

# Step 7: Generate report
create_artifact(
  name="Tom Frazier Pipeline Status Report",
  format_type="markdown",
  description="...",
  data="[formatted report markdown]"
)
```

**Total tool calls: ~7-8**
- 1 get_model_context
- 0-1 set_model_context (if needed)
- 1-2 catalog_modules
- 1 catalog_line_items
- 1 sql_schema
- 3 sql_query
- 1 create_artifact

---

## DEPENDENCIES

### Required Tools:
- `get_model_context` - Check if model is bound
- `set_model_context` - Bind workspace and model
- `catalog_workspaces` - List available workspaces
- `catalog_workspaces_and_models` - List available models
- `catalog_modules` - Discover pipeline modules
- `catalog_line_items` - Get available fields
- `sql_schema` - Build query schema
- `sql_query` - Execute queries
- `create_artifact` - Save report

### Optional Tools:
- `explain_cell` - For understanding specific calculations
- `get_model_status` - Check if model is ready
- `catalog_lists` - Get dimension members (for validation)

### Output Tools:
- `create_artifact` - Primary output (markdown report)
- Alternative: Direct response for quick summaries

---

## NOTES

### Model Context Best Practice
- **ALWAYS check `get_model_context()` first** before any model operations
- **Never assume or hardcode workspace/model IDs** - always ask the user
- If context is not set, **help the user discover their workspace and model**
- Even if context is set, **verify it's the correct model** with the user

### SQL Constraints
- **MUST constrain every dimension column** from the schema
- Use `dimension_name_is_leaf = true` for all list dimensions
- Do NOT filter on `time` or `versions` unless they appear in `dimension_column_names` in the schema
- If a dimension appears in schema but you don't want to filter it, use: `dimension_is_leaf = true` without an equality filter

### included_objects Requirement
- **REQUIRED** on both `sql_schema()` and `sql_query()`
- **MUST be identical** between schema and query calls
- Format: `{"Module Display Name": ["Line Item 1", "Line Item 2"]}`
- Use empty list `[]` to include all line items: `{"Pipeline": []}`

### Parameterized Queries
- **Always use `:param_name` placeholders**
- Pass matching `parameters` dict: `{"param_name": "value"}`
- Never concatenate user input into SQL strings
- Parameter values can be string, number, or boolean

### Field Name Variations
Common field name variations across models:
- **Amount:** Amount, Value, Deal Size, Opportunity Value, Deal Value
- **Stage:** Stage, Sales Stage, Opportunity Stage, Phase
- **Rep:** Sales Rep, Owner, Rep Name, Account Executive
- **Status:** Status, Opportunity Status, State, Stage Status
- **Category:** Forecast Category, Commit Category, Confidence Level

Adapt queries to actual field names discovered in the model.

### Performance Tips
- Start with aggregates before details
- Use LIMIT on detail queries
- Filter on indexed dimensions early (rep, status)
- Avoid SELECT * - specify needed columns
- Batch independent catalog calls

### Report Quality
- Use clear section headers with emojis
- Include both summary metrics and details
- Provide actionable insights, not just data
- Format currency consistently ($X,XXX,XXX)
- Show percentages with context
- Highlight important items (top opps, risks)
- End with recommendations

---

## VERSION HISTORY

- v1.0: Initial skill creation
- v1.1: Added model context verification (Step 0)
- v1.2: Enhanced error handling for context management

---

*This skill provides a complete, production-ready workflow for generating sales pipeline status reports from Anaplan models.*