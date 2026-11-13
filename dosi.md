How metrics compose¶
A metric in Dosi is not a number and not a SQL fragment. It is a function of the query: give it a group-by, a time range and some conditions, and it returns one value per group. Derived metrics are built from other metrics with a small set of building blocks, and each block takes any metric as its input — a filter over a ratio, a window over a sum of two facts, a share beside a running total. When a combination cannot be computed correctly, Dosi refuses it with an error code instead of returning a wrong number.

This page shows the building blocks on a small example, the order they apply in, where each kind of condition applies, and what is still refused.

The example¶
Two tables. Every result below was run against them.

customers	region
C1	East
C2	East
C3	West
orders	customer	order_date	amount	is_new
O1	C1	2024-01-05	100	true
O2	C1	2024-01-20	300	false
O3	C2	2024-01-12	200	true
O4	C3	2024-02-03	400	true
O5	C2	2024-02-15	150	false
O6	C3	2024-02-20	250	false
revenue is SUM(orders.amount) and order_count is COUNT(orders.order_id); the other metrics are declared from these two below.

A metric is a function of the query¶
The model declares revenue once. Each query supplies the arguments:


dosi query --model model.yaml --metrics revenue \
    --group-by customers.region --execute --db shop.duckdb
region  revenue
East    750
West    650
2 rows

dosi query --model model.yaml --metrics revenue \
    --group-by metric_time:month --where "customers.region = 'East'" \
    --order metric_time__month --execute --db shop.duckdb
metric_time__month   revenue
2024-01-01 00:00:00  600
2024-02-01 00:00:00  150
2 rows
Same metric, two queries, two results. What a query passes in is its context: the group-by, the time range, and the conditions, kept apart by where they came from because the building blocks read them differently (see Where a condition applies).

The building blocks¶
Block	What it does to the query it is evaluated under	How a model writes it
Aggregate	Reads the rows the query selects and aggregates them per group	the metric's expression, SUM(orders.amount)
Filter	Adds a condition the aggregates apply to their input; no group disappears	derive: filter
Arithmetic	Evaluates every input under the same query and combines them group by group	derive: compose, a ratio
Shift	Moves the query's time range, then lines the result up with the current period	window offset, pop
Window	Runs over the result rows: running totals, moving averages, ranks	window frame, rank, value
Exclude	Evaluates its input with some group-by dimensions removed, and attaches the coarser value to each row	exclude(...)
Fixed	Evaluates its input at exactly the dimensions it names, before where	fixed(...)
Include	Evaluates its input at a finer grain, then aggregates back	avg(include(...))
Any block can sit on any metric. A filter over a ratio is the ratio's two aggregates, each filtered:


dosi query --model model.yaml --metrics aov,new_aov \
    --group-by customers.region --execute --db shop.duckdb
region  aov    new_aov
East    187.5  150
West    325    400
2 rows
aov is the compose revenue / order_count, and new_aov is a filter over aov with orders.is_new. East's new orders are O1 and O3, (100 + 200) / 2.

A share is arithmetic over a metric and its excluded total:


dosi query --model model.yaml --metrics revenue,revenue_total,revenue_share \
    --group-by customers.region --execute --db shop.duckdb
region  revenue  revenue_total  revenue_share
East    750      1400           0.5357142857142857
West    650      1400           0.4642857142857143
2 rows
Blocks that cannot share one pass over the rows still share one query. A month-over-month change and a running total are windows over the monthly rows; an average per customer regroups the orders by customer first. Asked for together, Dosi computes them as separate parts and lines the parts up on the group-by:


dosi query --model model.yaml \
    --metrics revenue,revenue_mom,running_revenue,revenue_per_customer \
    --group-by metric_time:month --order metric_time__month \
    --execute --db shop.duckdb
metric_time__month   revenue  revenue_mom  running_revenue  revenue_per_customer
2024-01-01 00:00:00  600      NULL         600              300
2024-02-01 00:00:00  800      200          1400             400
2 rows
January has no previous month, so revenue_mom is NULL there, not 0.

The order of blocks¶
Some blocks commute, so their order does not matter:

A filter commutes with every other block. A filter over a compose is the compose of the filtered members, and two filters are one condition.
Excluding a, then b, is excluding both.
Others do not, and Dosi keeps them in the order the model wrote:

Two one-month shifts are not one two-month shift at a daily grain: 31 March 2024 less a month is 29 February, and less another month is 29 January, where two months back is 31 January.
A window over a ratio is not the ratio of two windows: the running sum of a ratio is not the running sum of its numerator over the running sum of its denominator.
What is still refused¶
A few combinations need a block to read its input's result rows before it can start, and Dosi does not compute those yet:

a window whose base is a level-of-detail metric;
a level of detail whose base is a window;
a filter over either.
They are refused when the model is validated, never computed wrong:


dosi validate --model model.yaml
✗ NotImplemented in metric 'running_revenue_per_customer': window.base "revenue_per_customer" is a level-of-detail metric; a window over a level of detail is not implemented
  hint: see docs/datus-extensions.md#d-lod-view-relative
The window and level-of-detail references list the rest.

Where a condition applies¶
A query's conditions are kept apart by where they came from, because the building blocks read them differently:

Condition	Read by
context_filter	every aggregate, before any level of detail — fixed included
where conditions on rows	every aggregate except a fixed one
a filter derivation	only the aggregates under it
where conditions on a selected metric — in a window query, also on group-by fields alone	the result rows
That table is why the same condition in where and in context_filter can give a fixed metric two different values: one reaches it, the other does not.

Limits¶
No time spine: a period with no rows has no row, and nothing invents it. A shift lines periods up by calendar, so a missing previous month reads NULL.
A window can read another window's output three levels deep; a fourth is refused.
Computing parts apart is enabled on DuckDB, PostgreSQL, Hologres, GaussDB, DWS, MySQL, TiDB, Trino, Athena, Snowflake, BigQuery, Databricks and ClickHouse. On other dialects a query that needs it is refused.

What Is Apache Ossie? And Why Dosi?¶
Apache Ossie, formerly known as Open Semantic Interchange (OSI), is an open specification for exchanging semantic models, metrics, dimensions, and business context across BI tools, AI agents, and data platforms.

An Ossie model is a plain YAML file. Define a metric in it once, and every tool that speaks Ossie reads that one definition instead of re-implementing it. Dosi is the first Apache Ossie-native semantic layer engine: it loads an Ossie model, validates it against the upstream spec, and compiles metric requests into correct SQL for 16 warehouse dialects.


Apache Ossie at a glance¶
What it is	An open, vendor-neutral specification for semantic models
Formerly	Open Semantic Interchange (OSI)
Format	YAML (or JSON)
Describes	Datasets, relationships, fields and dimensions, metrics, and AI context for each
Leaves to engines	How to turn a metric into correct SQL on a specific warehouse
Upstream	ossie.apache.org and apache/ossie: the core spec and a reference validator; format converters live in apache/ossie-converters
Engine on this site	Dosi: validates, compiles, and runs Ossie models
The problem: everyone redefines "revenue"¶
Write "August revenue" as SQL from scratch and there are three separate ways to get it wrong: the join (which table has revenue — and does joining it fan out the rows and quietly double the total?), the dimension (order date or ship date, gross or net, daily grain or monthly), and the expression (SUM(amount), or SUM(unit_price * quantity) from a line-items table that skips the discount column, or SUM(CASE WHEN status != 'refunded' THEN amount ELSE 0 END) to back out refunds). Miss any one and the result still looks plausible.

That's exactly why a BI dashboard, a finance spreadsheet, and a hand-written query end up as three different "revenues" — one excludes cancelled orders, another doesn't, a third's join doubles the total nobody checked. And it's why an AI agent asked to write the query from raw table names makes the same three mistakes, just with more confidence. The result is familiar to everyone: two "correct" numbers that don't match, and a meeting to figure out which one to trust.

A semantic model fixes this by defining each metric once — what revenue means, which table it comes from, how tables relate — as a single source of truth. Every query then derives from that one definition instead of re-implementing it. Apache Ossie is an open, standard way to write that model down.

What an Apache Ossie model looks like¶
A model lists your datasets (tables) and their fields, the relationships between them, and your metrics as ordinary SQL expressions. This one is complete and valid:


version: "0.2.0.dev0"

name: orders_model
description: Orders, the customers who placed them, and revenue
datasets:
  - name: orders
    source: main.orders
    primary_key: [order_id]
    fields:
      - name: order_id
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: order_id
      - name: customer_id
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: customer_id
      - name: order_date
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: order_date
        dimension:
          is_time: true
      - name: amount
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: amount
  - name: customers
    source: main.customers
    primary_key: [customer_id]
    fields:
      - name: customer_id
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: customer_id
      - name: region
        expression:
          dialects:
            - dialect: ANSI_SQL
              expression: region

relationships:
  - name: orders_to_customers
    from: orders
    to: customers
    from_columns: [customer_id]
    to_columns: [customer_id]

metrics:
  - name: revenue
    description: Total order amount
    expression:
      dialects:
        - dialect: ANSI_SQL
          expression: SUM(orders.amount)
    ai_context:
      synonyms: [sales, order revenue]
Datasets and fields name the physical tables and the columns you can group and filter by; dimension: {is_time: true} marks a time dimension.
Relationships say how tables join, so a consumer never has to guess the join key.
Metrics are SQL aggregate expressions over those fields, written once.
ai_context carries business context — synonyms, instructions, examples — for AI agents that read the model.
Ossie describes what your metrics are. It deliberately does not say how to turn them into correct SQL for a specific warehouse. That is the gap Dosi fills.

One model per document¶
A document holds one semantic model: version and the model's own fields sit at the document root, as in the example above. That is the shape upstream's schema accepts since apache/ossie#383.

Dosi automatically upgrades historical model files to this root structure during normal file loading. The original bytes are retained in a hidden .bak file beside the model, and the replacement is atomic. Existing flat files are never rewritten. New models and text inputs (including REST/MCP validation and Python model_text) must use the root structure.

An unwritable historical file, multiple models, conflicting fields, or a layout that cannot be rewritten without changing values produces an error and keeps the original file. A safely upgraded model may still have validation or compile errors; edit its new root structure to repair them. A migration requires write access to the model directory. No separate migration command is needed.

An interchange, not just a format¶
The Interchange in the original name still matters: the upstream Apache Ossie project maintains bidirectional converters between Ossie and other semantic-layer formats — dbt, Snowflake, Databricks, Salesforce/Tableau, GoodData, Honeydew, Omni, and more. One neutral hub instead of point-to-point migrations.

Dosi reads standard Ossie models as they are, so a converter's output works unchanged as long as its metrics are SQL expressions (see S-EXPR-1). A model can arrive from dbt today and leave for Snowflake tomorrow; your metric definitions are never hostage to one vendor's format. Define once, and move freely.

How Dosi relates to Apache Ossie¶
Apache Ossie is the contract; Dosi is the engine that executes it. Dosi does not define its own model format: its input is an Ossie model, and everything it adds either happens at query time or rides inside the extension point the spec itself provides.

Apache Ossie defines	Dosi adds
Datasets, relationships, fields, and metrics as SQL expressions	Join planning with fan-out protection, and SQL for 20 dialects
A reference validator	dosi validate --osi-basic runs it alongside Dosi's own checks
custom_extensions, the spec's vendor extension point	The Datus extensions — windows, derived metrics, join type, null-fill, time grain — carried inside it
AI context on every object	A native MCP server that serves the model to agents, with structured errors
Converters to and from other formats	Converted models with SQL metrics accepted as input, unchanged
Dosi runs in one of two modes, chosen per invocation:

Basic (--osi-basic): strict standard Apache Ossie. The model means exactly what the spec says; Datus extensions are ignored with a warning.
Datus (--osi-datus, the default): Apache Ossie plus the Datus extensions.
Datus extensions live inside custom_extensions, so they never make a model invalid: other Ossie tools read it and skip the extensions they don't know. Datus mode is more lenient than the spec in one respect — it also accepts vendor SQL dialect tags such as POSTGRESQL that the core spec does not define — so run dosi validate --osi-basic when a model has to stay valid for other Ossie tools.

Dosi is developed by Datus. It implements the Apache Ossie specification but is not itself an Apache Software Foundation project.

What Dosi does¶
Dosi is a small, fast (Rust) engine that reads a pure Apache Ossie model and turns a metric request into correct, warehouse-specific SQL — then, optionally, runs it:

Your Apache Ossie model
metrics defined once

Dosi

Correct SQL
for your warehouse

Results

You ask for a metric, a few dimensions to group by, and a warehouse dialect. Dosi works out the joins, the aggregation, and the exact SQL idioms that dialect needs — the same model produces correct SQL for 20 dialects: DuckDB, Postgres, MySQL, ClickHouse, Snowflake, BigQuery, StarRocks, Trino, Databricks, Oracle, and more. Switching warehouses changes one flag, not your metric definitions.

You can use it four ways — a command-line tool, a REST + Arrow server, a native MCP server for AI agents, or Python bindings — all sharing the same engine and the same answers.

What an AI agent needs from a metric system¶
When the consumer of a metric is an AI agent rather than a person reading a dashboard, the metric layer has to answer questions a human analyst normally settles by judgment. For every metric, four properties decide what can be done with it safely:

Additive — can it be summed across a dimension, or would adding it double-count (distinct counts, ratios)?
Rollable — do daily values roll up into a monthly one, or must the month be recomputed from raw rows?
Predictable — is it a flow you can forecast forward, or a stock where extrapolation is meaningless?
Fan-out-safe — will a join multiply rows and silently inflate the total?
Dosi treats these as properties the engine must know, not conventions in an analyst's head. Today it infers every metric's kind (aggregate / ratio / expression) and never silently double-counts: when a metric would join tables at different levels of detail — the classic fan-out that doubles your revenue — it either computes each part at its own grain, or stops with a structured error. It will not hand back a quietly inflated total.

Built-in attribution, chosen per metric kind: when an agent asks "why did revenue drop?", one call computes the contribution breakdown with the algorithm that is exact for that metric's algebra — dimension attribution for additive metrics, an LMDI mix/rate decomposition for ratios — so the agent's reasoning is observable and checkable, not hallucinated.

On the authoring side, datus-agent — Datus' own open-source data agent — is the best first-party way to produce these models: it generates Dosi-ready Apache Ossie YAML with the Datus extensions, which add more advanced window and derived metrics and more complete join-relationship detection than the base spec. The consumption side stays open: any agent can query the result through the MCP server, for example from Claude Code or Codex through the Dosi plugin.

What makes the numbers trustworthy¶
Beyond the metric algebra above, the engine closes off the everyday mistakes that produce mismatched numbers:

Time ranges are unambiguous. A range like Jan 2024 means [2024-01-01, 2025-01-01) — start included, end excluded — so days never get double-counted at month boundaries.
Errors tell you how to fix them. A wrong metric name doesn't get a stack trace; it gets a message naming the closest valid options. In --format json, every error carries a stable code and suggested fix, so automated tools (and AI agents) can self-correct.
These guarantees are written down precisely — as a normative contract — in the semantics reference. You don't need to read it to use Dosi, but it's there when you want to know exactly what the engine promises.

Grain: what one row means¶
Every number Dosi returns was computed at some grain, the answer to the question what does one row mean here? One row of orders is one order. One row of a result grouped by region is one region. Most wrong metric numbers come from getting this wrong without noticing: a join that quietly turns one customer into three rows, or an average taken over orders when the question was about customers.

This page names the four grains in a query, gives the one rule Dosi enforces between them, and shows the two ways to bring more than one grain into an answer: aligning two fact tables, and letting a metric compute at its own grain with LOD.

The example¶
Three small tables. Every result below was run against them.

customers	region	sales_target
C1	East	500
C2	East	300
C3	West	600
orders	customer	order_date	amount
O1	C1	2024-01-05	100
O2	C1	2024-01-20	300
O3	C2	2024-01-12	200
O4	C3	2024-02-03	400
visits holds eight site visits: five in January and three in February. Each order and each visit belongs to one customer, so both relate many-to-one to customers.

Four grains in one query¶
Take revenue (SUM(orders.amount)) by region:


dosi query --model model.yaml --metrics revenue,order_count \
    --group-by customers.region --execute --db shop.duckdb
region  revenue  order_count
East    600      3
West    400      1
2 rows
Four grains are at work in that query:

Grain	What one row means here	Set by
Row grain	orders: one order. customers: one customer.	The dataset's primary_key
Calculation grain	Each order's amount enters the SUM once.	The metric. By default, the row grain of the dataset it aggregates
Time grain	Not used here. With orders.order_date:month, one bucket per month.	:day … :year on a time dimension
Query grain	One region.	group_by, or --from for detail rows
In an ordinary query, the calculation grain is the row grain and the query grain is the group-by, and you never think about either. The rest of this page is about what happens when they differ.

The rule: aggregate from fine to coarse¶
Data can always be aggregated from a fine grain to a coarser one: each fine row lands in exactly one coarse group, so nothing is counted twice. Go the other way, looking at a coarse value at a finer grain, and the join copies it onto several rows; sum those rows and it is counted several times. That is fan-out.

A join in the many-to-one direction is safe. An order has exactly one customer, so joining orders to customers gives each order row one region. That adds a column, never a row, which is why revenue by region above is correct.

A join in the other direction fans out. customers.sales_target lives at the customer grain. Ask for it by order month and each customer is copied onto every one of their orders. A hand-written join returns 1,300 for January, because C1 has two January orders and its target of 500 is counted twice. Dosi refuses instead:


dosi query --model model.yaml --metrics total_target \
    --group-by orders.order_date:month --execute --db shop.duckdb
error: FanOutRisk: metric "total_target" aggregates Sum over dataset "customers", which has no many→one path to ["orders"]; evaluating a duplicate-sensitive aggregate (SUM/AVG/COUNT) across that join would double-count rows
  hint: group by dimensions reachable from the metric's dataset, or restate the metric with a duplicate-insensitive aggregate (COUNT DISTINCT/MIN/MAX)
Grouped by customers.region, the same metric is fine (East 800, West 600): that grouping is at or above the customer grain. Semantics — fan-out protection has the exact rules.

Time grain follows the same rule. Daily rows roll up into months. A column stored monthly, such as a month-end snapshot, cannot be split back into days, so asking for :day on a column whose declared native grain is month is a grain_too_fine error.

Two facts in one answer: align, don't join rows¶
orders and visits are both facts, and neither belongs to the other: a visit is not part of an order. Join their rows directly and every order is copied onto every visit by the same customer. On this data that reports nine orders and nine visits in January, a conversion of 1.0. There were three orders and five visits.

Dosi aggregates first, then joins the results. Each fact is aggregated on its own up to the query grain (one row per month, say), and the two results are then joined on the columns they share. So there is still a join, but it joins one aggregated row to one aggregated row, and nothing fans out. When both facts reach a shared dimension table, that needs no extra declaration:


dosi query --model model.yaml --metrics order_count,visit_count \
    --group-by customers.region --execute --db shop.duckdb
region  order_count  visit_count
East    3            6
West    1            2
2 rows
Dates are the common case with no dimension table between them. Without a declaration, orders.order_date means nothing to visits, and a metric that needs both is refused:


error: UnconformedDimension: dimension "orders.order_date" is not conformed for derived metric "conversion": member(s) ["visit_count"] have no many→one path to "orders"
D-CONFORM declares that visits.visit_date and orders.order_date are the same dimension, spelled differently on each fact:


relationships:
  - name: visits_orders_by_date
    from: visits
    to: orders
    from_columns: [visit_date]
    to_columns: [order_date]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.8", "requires": ["cardinality"], "cardinality": "many_to_many"}'
With that pairing, each fact groups its own date, and the branches merge on the month:


dosi query --model model.yaml --metrics order_count,visit_count,conversion \
    --group-by orders.order_date:month --execute --db shop.duckdb
order_date__month    order_count  visit_count  conversion
2024-02-01 00:00:00  1            3            0.3333333333333333
2024-01-01 00:00:00  3            5            0.6
2 rows
The generated SQL makes it plain: one GROUP BY per fact, then a FULL OUTER JOIN on the month. Order rows and visit rows are never joined to each other. D-CONFORM does not change any grain; it only tells Dosi which column to match the two results on once each side is aggregated.

A metric at its own grain: LOD¶
Some questions need a calculation grain that is neither the row grain nor the query grain. "Average lifetime revenue per customer, by region" is one: first total each customer's orders, then average those totals within each region. Averaging the order amounts gives East 200, an answer to a different question, because C1 counts twice.

LOD (level of detail) declares a metric's calculation grain separately from the query grain. The query still only picks metrics and a group-by; the model says where the intermediate step happens. Measured against the query grain, there are three moves:

Declaration	Calculation grain	Compared with the query grain	Back at the query grain	Example result
fixed(revenue, customers.customer_id)	exactly the listed keys	independent of it	as a field: aggregate it, group by it, or filter on it	AVG of it by region: East 300, West 400
AVG(include(order_count, orders.customer_id))	the query grain plus the keys	finer	must be aggregated back: the outer AVG	orders per buyer by month: January 1.5, February 1
exclude(revenue, orders.order_date)	the query grain minus the keys	coarser	copied onto each row	by customer and date, C1's two rows both read 400
The rule from above decides each "back at the query grain" column:

include computes at a finer grain, so its values cannot come back without being combined. Dosi requires the outer aggregate that says how.
exclude computes at a coarser grain, so each query row reads exactly one value, and copying it is safe.
fixed has a grain of its own that does not follow the query. As a metric, the group-by must include all of its keys, without a time grain such as :day. Other dimensions may be added, and the same fixed value then repeats on each of their rows. Dosi will not roll it up to a coarser grain for you. As a field it becomes an attribute of its entity, one value per customer, which ordinary metrics can then aggregate, group or filter by.
LOD also changes which records a calculation reads: a fixed calculation ignores ordinary display filters. That is a scope question, not a grain question; Filters decide which records enter the calculation covers it.

LOD and alignment together¶
The two mechanisms compose. Each fact can define its own fixed field, for example a buyer classification on orders and another on visits, and a D-CONFORM pairing between those fields makes them one dimension. Each fact then computes its classification and its metrics alone, and the grouped results align, exactly as dates did above.

What does not work is a fixed field on one fact that directly aggregates the other, grouped together with both facts' metrics: that is a row-to-row path between two facts again, and it is refused as fan_out_risk. LOD across datasets has the full rules.

Reading grain errors¶
Most planning errors that are not typos are a grain mismatch. The code tells you which grain:

Code	The mismatch	What to change
fan_out_risk	a metric would be copied onto finer rows than its row grain	group by something its dataset reaches many-to-one, or use COUNT DISTINCT / MIN / MAX
unconformed_dimension	two facts have no shared column at the requested grain	group by a shared dimension, or pair the columns with D-CONFORM
grain_too_fine	a time grain finer than the column stores	use the stored grain or coarser
grain_on_non_time_dimension	a time grain on a field that is not a time dimension	drop the :grain, or mark the field as time
lod_dimension_not_grouped	an LOD metric's keys are missing from the query grain	add the keys to group_by
time_axis_not_grouped, window_reset_too_fine	a window's time axis is missing from, or finer than, the query grain	group by the axis at the window's grain
Every one of these carries candidates or a suggested_retry. See Errors

How metrics compose¶
A metric in Dosi is not a number and not a SQL fragment. It is a function of the query: give it a group-by, a time range and some conditions, and it returns one value per group. Derived metrics are built from other metrics with a small set of building blocks, and each block takes any metric as its input — a filter over a ratio, a window over a sum of two facts, a share beside a running total. When a combination cannot be computed correctly, Dosi refuses it with an error code instead of returning a wrong number.

This page shows the building blocks on a small example, the order they apply in, where each kind of condition applies, and what is still refused.

The example¶
Two tables. Every result below was run against them.

customers	region
C1	East
C2	East
C3	West
orders	customer	order_date	amount	is_new
O1	C1	2024-01-05	100	true
O2	C1	2024-01-20	300	false
O3	C2	2024-01-12	200	true
O4	C3	2024-02-03	400	true
O5	C2	2024-02-15	150	false
O6	C3	2024-02-20	250	false
revenue is SUM(orders.amount) and order_count is COUNT(orders.order_id); the other metrics are declared from these two below.

A metric is a function of the query¶
The model declares revenue once. Each query supplies the arguments:


dosi query --model model.yaml --metrics revenue \
    --group-by customers.region --execute --db shop.duckdb
region  revenue
East    750
West    650
2 rows

dosi query --model model.yaml --metrics revenue \
    --group-by metric_time:month --where "customers.region = 'East'" \
    --order metric_time__month --execute --db shop.duckdb
metric_time__month   revenue
2024-01-01 00:00:00  600
2024-02-01 00:00:00  150
2 rows
Same metric, two queries, two results. What a query passes in is its context: the group-by, the time range, and the conditions, kept apart by where they came from because the building blocks read them differently (see Where a condition applies).

The building blocks¶
Block	What it does to the query it is evaluated under	How a model writes it
Aggregate	Reads the rows the query selects and aggregates them per group	the metric's expression, SUM(orders.amount)
Filter	Adds a condition the aggregates apply to their input; no group disappears	derive: filter
Arithmetic	Evaluates every input under the same query and combines them group by group	derive: compose, a ratio
Shift	Moves the query's time range, then lines the result up with the current period	window offset, pop
Window	Runs over the result rows: running totals, moving averages, ranks	window frame, rank, value
Exclude	Evaluates its input with some group-by dimensions removed, and attaches the coarser value to each row	exclude(...)
Fixed	Evaluates its input at exactly the dimensions it names, before where	fixed(...)
Include	Evaluates its input at a finer grain, then aggregates back	avg(include(...))
Any block can sit on any metric. A filter over a ratio is the ratio's two aggregates, each filtered:


dosi query --model model.yaml --metrics aov,new_aov \
    --group-by customers.region --execute --db shop.duckdb
region  aov    new_aov
East    187.5  150
West    325    400
2 rows
aov is the compose revenue / order_count, and new_aov is a filter over aov with orders.is_new. East's new orders are O1 and O3, (100 + 200) / 2.

A share is arithmetic over a metric and its excluded total:


dosi query --model model.yaml --metrics revenue,revenue_total,revenue_share \
    --group-by customers.region --execute --db shop.duckdb
region  revenue  revenue_total  revenue_share
East    750      1400           0.5357142857142857
West    650      1400           0.4642857142857143
2 rows
Blocks that cannot share one pass over the rows still share one query. A month-over-month change and a running total are windows over the monthly rows; an average per customer regroups the orders by customer first. Asked for together, Dosi computes them as separate parts and lines the parts up on the group-by:


dosi query --model model.yaml \
    --metrics revenue,revenue_mom,running_revenue,revenue_per_customer \
    --group-by metric_time:month --order metric_time__month \
    --execute --db shop.duckdb
metric_time__month   revenue  revenue_mom  running_revenue  revenue_per_customer
2024-01-01 00:00:00  600      NULL         600              300
2024-02-01 00:00:00  800      200          1400             400
2 rows
January has no previous month, so revenue_mom is NULL there, not 0.

The order of blocks¶
Some blocks commute, so their order does not matter:

A filter commutes with every other block. A filter over a compose is the compose of the filtered members, and two filters are one condition.
Excluding a, then b, is excluding both.
Others do not, and Dosi keeps them in the order the model wrote:

Two one-month shifts are not one two-month shift at a daily grain: 31 March 2024 less a month is 29 February, and less another month is 29 January, where two months back is 31 January.
A window over a ratio is not the ratio of two windows: the running sum of a ratio is not the running sum of its numerator over the running sum of its denominator.
What is still refused¶
A few combinations need a block to read its input's result rows before it can start, and Dosi does not compute those yet:

a window whose base is a level-of-detail metric;
a level of detail whose base is a window;
a filter over either.
They are refused when the model is validated, never computed wrong:


dosi validate --model model.yaml
✗ NotImplemented in metric 'running_revenue_per_customer': window.base "revenue_per_customer" is a level-of-detail metric; a window over a level of detail is not implemented
  hint: see docs/datus-extensions.md#d-lod-view-relative
The window and level-of-detail references list the rest.

Where a condition applies¶
A query's conditions are kept apart by where they came from, because the building blocks read them differently:

Condition	Read by
context_filter	every aggregate, before any level of detail — fixed included
where conditions on rows	every aggregate except a fixed one
a filter derivation	only the aggregates under it
where conditions on a selected metric — in a window query, also on group-by fields alone	the result rows
That table is why the same condition in where and in context_filter can give a fixed metric two different values: one reaches it, the other does not.

How metrics compose¶
A metric in Dosi is not a number and not a SQL fragment. It is a function of the query: give it a group-by, a time range and some conditions, and it returns one value per group. Derived metrics are built from other metrics with a small set of building blocks, and each block takes any metric as its input — a filter over a ratio, a window over a sum of two facts, a share beside a running total. When a combination cannot be computed correctly, Dosi refuses it with an error code instead of returning a wrong number.

This page shows the building blocks on a small example, the order they apply in, where each kind of condition applies, and what is still refused.

The example¶
Two tables. Every result below was run against them.

customers	region
C1	East
C2	East
C3	West
orders	customer	order_date	amount	is_new
O1	C1	2024-01-05	100	true
O2	C1	2024-01-20	300	false
O3	C2	2024-01-12	200	true
O4	C3	2024-02-03	400	true
O5	C2	2024-02-15	150	false
O6	C3	2024-02-20	250	false
revenue is SUM(orders.amount) and order_count is COUNT(orders.order_id); the other metrics are declared from these two below.

A metric is a function of the query¶
The model declares revenue once. Each query supplies the arguments:


dosi query --model model.yaml --metrics revenue \
    --group-by customers.region --execute --db shop.duckdb
region  revenue
East    750
West    650
2 rows

dosi query --model model.yaml --metrics revenue \
    --group-by metric_time:month --where "customers.region = 'East'" \
    --order metric_time__month --execute --db shop.duckdb
metric_time__month   revenue
2024-01-01 00:00:00  600
2024-02-01 00:00:00  150
2 rows
Same metric, two queries, two results. What a query passes in is its context: the group-by, the time range, and the conditions, kept apart by where they came from because the building blocks read them differently (see Where a condition applies).

The building blocks¶
Block	What it does to the query it is evaluated under	How a model writes it
Aggregate	Reads the rows the query selects and aggregates them per group	the metric's expression, SUM(orders.amount)
Filter	Adds a condition the aggregates apply to their input; no group disappears	derive: filter
Arithmetic	Evaluates every input under the same query and combines them group by group	derive: compose, a ratio
Shift	Moves the query's time range, then lines the result up with the current period	window offset, pop
Window	Runs over the result rows: running totals, moving averages, ranks	window frame, rank, value
Exclude	Evaluates its input with some group-by dimensions removed, and attaches the coarser value to each row	exclude(...)
Fixed	Evaluates its input at exactly the dimensions it names, before where	fixed(...)
Include	Evaluates its input at a finer grain, then aggregates back	avg(include(...))
Any block can sit on any metric. A filter over a ratio is the ratio's two aggregates, each filtered:


dosi query --model model.yaml --metrics aov,new_aov \
    --group-by customers.region --execute --db shop.duckdb
region  aov    new_aov
East    187.5  150
West    325    400
2 rows
aov is the compose revenue / order_count, and new_aov is a filter over aov with orders.is_new. East's new orders are O1 and O3, (100 + 200) / 2.

A share is arithmetic over a metric and its excluded total:


dosi query --model model.yaml --metrics revenue,revenue_total,revenue_share \
    --group-by customers.region --execute --db shop.duckdb
region  revenue  revenue_total  revenue_share
East    750      1400           0.5357142857142857
West    650      1400           0.4642857142857143
2 rows
Blocks that cannot share one pass over the rows still share one query. A month-over-month change and a running total are windows over the monthly rows; an average per customer regroups the orders by customer first. Asked for together, Dosi computes them as separate parts and lines the parts up on the group-by:


dosi query --model model.yaml \
    --metrics revenue,revenue_mom,running_revenue,revenue_per_customer \
    --group-by metric_time:month --order metric_time__month \
    --execute --db shop.duckdb
metric_time__month   revenue  revenue_mom  running_revenue  revenue_per_customer
2024-01-01 00:00:00  600      NULL         600              300
2024-02-01 00:00:00  800      200          1400             400
2 rows
January has no previous month, so revenue_mom is NULL there, not 0.

The order of blocks¶
Some blocks commute, so their order does not matter:

A filter commutes with every other block. A filter over a compose is the compose of the filtered members, and two filters are one condition.
Excluding a, then b, is excluding both.
Others do not, and Dosi keeps them in the order the model wrote:

Two one-month shifts are not one two-month shift at a daily grain: 31 March 2024 less a month is 29 February, and less another month is 29 January, where two months back is 31 January.
A window over a ratio is not the ratio of two windows: the running sum of a ratio is not the running sum of its numerator over the running sum of its denominator.
What is still refused¶
A few combinations need a block to read its input's result rows before it can start, and Dosi does not compute those yet:

a window whose base is a level-of-detail metric;
a level of detail whose base is a window;
a filter over either.
They are refused when the model is validated, never computed wrong:


dosi validate --model model.yaml
✗ NotImplemented in metric 'running_revenue_per_customer': window.base "revenue_per_customer" is a level-of-detail metric; a window over a level of detail is not implemented
  hint: see docs/datus-extensions.md#d-lod-view-relative
The window and level-of-detail references list the rest.

Where a condition applies¶
A query's conditions are kept apart by where they came from, because the building blocks read them differently:

Condition	Read by
context_filter	every aggregate, before any level of detail — fixed included
where conditions on rows	every aggregate except a fixed one
a filter derivation	only the aggregates under it
where conditions on a selected metric — in a window query, also on group-by fields alone	the result rows
That table is why the same condition in where and in context_filter can give a fixed metric two different values: one reaches it, the other does not.

Query metrics¶
A metric query names metrics and the dimensions to break them down by; Dosi writes the aggregation, the joins and the GROUP BY. For row-level records instead of aggregates, see Query detail rows.


dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue,order_count \
    --group-by customers.region,orders.status \
    --order -revenue \
    --execute --db orders.duckdb
region  status     revenue  order_count
east    completed  10163    12
NULL    completed  7140.49  9
west    completed  6522     18
west    cancelled  10       1
NULL    pending    0        1
5 rows
You never write a JOIN: revenue lives on orders, region on customers, and Dosi follows the relationship between them. The examples on this page use the bundled orders model and the orders.duckdb file loaded in the tutorial.

Pick metrics and dimensions¶
--metrics takes metric names; --group-by takes dataset.field items. Ask the model what it has before guessing:


dosi list metrics --model $DOSI_EXAMPLES/orders/model.yaml
dosi list dimensions --model $DOSI_EXAMPLES/orders/model.yaml
dosi show metric revenue --model $DOSI_EXAMPLES/orders/model.yaml
metric: revenue
description: Total order amount
kind: aggregate
expr: SUM(orders.amount)
datasets: orders
group_by: orders, customers, products
time: metric_time (day, week, month, quarter, year)
group_by lists the datasets a metric can be broken down by, and time the grains it accepts. A misspelled name is refused with the names that do exist:


dosi query --model $DOSI_EXAMPLES/orders/model.yaml --metrics revenu
error: UnknownMetric: unknown metric "revenu"
  candidates: revenue, order_count, unique_customers, avg_order_value, total_margin
Time grain and metric_time¶
Append :grain to a time dimension to bucket it at that time grain: day, week, month, quarter or year. The output column is named {field}__{grain}:


dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue --group-by orders.order_date:month \
    --order order_date__month --execute --db orders.duckdb
order_date__month    revenue
2023-12-01 00:00:00  12
2024-01-01 00:00:00  2059
2024-02-01 00:00:00  1059
2024-03-01 00:00:00  33
2024-07-01 00:00:00  38
2024-11-01 00:00:00  1235
2024-12-01 00:00:00  839
2025-01-01 00:00:00  4811.49
2025-02-01 00:00:00  1743
2025-03-01 00:00:00  1304
2025-04-01 00:00:00  842
2025-05-01 00:00:00  118
2026-01-01 00:00:00  3101
2026-04-01 00:00:00  1707
2026-05-01 00:00:00  1029
2026-07-01 00:00:00  1690
2026-08-01 00:00:00  1205
2026-09-01 00:00:00  1010
18 rows
metric_time stands for each metric's primary time dimension, so you can ask for "revenue by month" without knowing which column holds the date. The column is then metric_time__month. A metric's primary time comes from the D-TIME extension, or from its dataset's only time field.

Time ranges¶
--start-time is inclusive and --end-time exclusive, so a range is half-open [start, end). February 2024 is:


dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue --start-time 2024-02-01 --end-time 2024-03-01 \
    --execute --db orders.duckdb
revenue
1059
1 row
With no time dimension in the group-by, the range applies to each metric's primary time. When the group-by holds several time dimensions, Dosi won't pick one for you:


dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue \
    --group-by orders.order_date:month,customers.signup_date:month \
    --start-time 2024-02-01 --end-time 2024-03-01
error: TimeRangeNeedsDimension: the group-by contains 2 time dimensions; name the one the time range applies to
  candidates: orders.order_date, customers.signup_date
  hint: pass time_dimension (e.g. --time-dimension orders.order_date, or metric_time)
Add --time-dimension orders.order_date and it runs.

Filters¶
--where takes SQL, and Dosi places each AND conjunct by what it reads:

a condition on fields filters input rows, before aggregation;
a condition on a selected metric filters result rows, after aggregation (the HAVING you would otherwise write).
One exception: in a query that selects a window metric, a condition on group-by fields only also filters result rows, so the rows kept keep the ranks and partition counts the whole population gave them. To rank within the narrower set instead, put the condition in --context-filter.

One --where can do both:


dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue,order_count --group-by customers.region \
    --where "orders.status = 'completed' AND revenue > 100" \
    --execute --db orders.duckdb
region  revenue  order_count
east    10163    12
NULL    7140.49  9
west    6522     18
3 rows
Add --pretty without --execute to see the placement: orders.status lands in the inner WHERE, revenue > 100 in a WHERE over the finished rows. A condition some part of the query cannot read is not dropped silently — it comes back as a warning (on stderr, or under warnings with --format json).

--context-filter is the other filter slot: it always scopes the calculation population, including the inputs of level-of-detail fields. See LOD filters for when you need it. The full placement rules are in Semantics contract §8.

Order and limit¶
--order takes comma-separated keys; a - prefix sorts descending. A key names a metric or a group-by item — by its output column (order_date__month) or by the spelling you used in --group-by (orders.order_date:month). Anything else is refused with the keys you can use:


dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue --order -amount
error: UnknownOrderKey: order key "amount" is not a metric or group-by item
  candidates: revenue
NULLs sort last on every warehouse, ascending and descending alike; --order "-revenue nulls first" puts them first. --limit caps the rows after ordering, so --order -revenue --limit 10 is a top 10.

Compile or execute¶
Without --execute, dosi query only prints the SQL. That is the way to check a query, or to hand the SQL to another tool:

Flag	Effect
--pretty	format the SQL
--dialect <name>	compile for another warehouse — the model doesn't change
--explain	print the logical plan above the SQL: which joins and aggregations were chosen
--execute	run it and print the rows
--connection <name>	run it on a profile from your connections file (see Connect a warehouse)
--format json	machine-readable output: sql, outputs, warnings and, when executed, rows

dosi query --model $DOSI_EXAMPLES/orders/model.yaml \
    --metrics revenue --group-by metric_time:month \
    --start-time 2024-01-01 --end-time 2024-03-01 \
    --dialect snowflake --pretty
SELECT
  DATE_TRUNC('MONTH', orders.order_date) AS metric_time__month,
  SUM(orders.amount) AS revenue
FROM orders AS orders
WHERE
  orders.order_date >= CAST('2024-01-01' AS DATE)
  AND orders.order_date < CAST('2024-03-01' AS DATE)
GROUP BY
  DATE_TRUNC('MONTH', orders.order_date)
Without --dialect, Dosi compiles for your default connection (DuckDB when there is none), so the SQL you check is the SQL --execute will run.

Metrics computed apart¶
Some metrics cannot be computed in one pass over the same rows: an include level of detail regroups its fact, a running total must not see the earlier rows an offset reads, a rank is defined inside one period. Asked for together, Dosi computes them as separate parts — each exactly as it would be alone, under the query's group-by, filters and time range — and aligns the parts on the group-by columns. A group any part has is a row of the result; a metric with no row for that group reads NULL (a bare count reads 0, a declared fill its value).

The same happens across time_ranges: a metric that only reads one period at a time (a rank, a level of detail) is computed once per period, with the usual {metric}__{tag} columns. Ordering by such a metric's bare name ranks by its {metric}__weight column, the summed absolute per-period values, as a query in one pass does.

--explain shows such a query as Align[keys=(…)] over its parts. These things keep a query in one pass, and refused if one pass cannot hold it:

a where condition on a selected metric, which would have to filter the aligned rows;
a where condition the parts would read differently: on a group-by field it filters the result of a window query and the input of any other, so a window part beside, say, an excluded total would read it two ways;
params bindings;
a dialect the shape is not enabled for yet. It is enabled on DuckDB, PostgreSQL, Hologres, GaussDB, DWS, MySQL, TiDB, Trino, Athena, Snowflake, BigQuery, Databricks and ClickHouse.
A metric that cannot be planned on its own under the query's group-by is still refused, with its own reason.

When Dosi refuses¶
Dosi never returns a number it cannot vouch for. The refusal that matters most is fan_out_risk: the metric would be summed across a join that repeats its rows. In the bundled tpcds model, store_productivity sums a per-store value, and a store has no single date:


dosi query --model $DOSI_EXAMPLES/tpcds/model.yaml \
    --metrics total_sales,store_productivity --group-by date_dim.d_date:month
error: FanOutRisk: metric "store_productivity" aggregates Sum over dataset "store", which has no many→one path to ["date_dim"]; evaluating a duplicate-sensitive aggregate (SUM/AVG/COUNT) across that join would double-count rows
  hint: group by dimensions reachable from the metric's dataset, or restate the metric with a duplicate-insensitive aggregate (COUNT DISTINCT/MIN/MAX)
Written by hand, that join returns a number, and it is wrong. Group by something the metric's dataset reaches, or query the two metrics separately.

Every refusal carries a machine-readable code, the names involved, candidates where a name was wrong, and a suggested_retry — run with --format json to get them as JSON. The common ones:

Code	Means
unknown_metric, unknown_dimension, unknown_field	a name the model doesn't define; candidates lists the ones it does
unknown_order_key	the order key is not a metric or group-by item
time_range_needs_dimension	several time dimensions in scope; pass --time-dimension
fan_out_risk	the result would double-count; regroup or split the query
The full list is in Errors.

Beyond the CLI¶
The same metric query — same fields, same semantics, same errors — runs on every surface:

REST API: POST /v1/query/compile, /v1/query/explain, /v1/query/execute
MCP server: tools compile_sql, explain_query, run_query
Python: Engine.compile() and Engine.execute() in the dosi-engine wheel

Query detail rows¶
Dosi answers two kinds of question over one semantic model.

Question	Command	Returns
What was revenue last month, by region?	dosi query	aggregated metrics
Show me the 100 largest orders from Enterprise customers	dosi select	rows
Both compile through the same planner, the same relationship graph and the same dialect backends. They differ in one operator: query aggregates, select projects.

The first example uses the bundled orders model and the orders.duckdb created in Your first metric query. Later examples illustrate other model shapes; fields such as customers.name, customers.tier, and regions.region_name require your own model.


dosi select --model $DOSI_EXAMPLES/orders/model.yaml \
    --from orders \
    --fields orders.order_id,orders.amount,customers.region \
    --where "customers.region = 'east'" \
    --order -amount \
    --limit 100 --execute --db orders.duckdb
You never write a JOIN. You name the fields you want; Dosi works out which datasets they live on, which relationships connect them, and whether that connection is safe.

--from is the grain, not the FROM clause¶
--from orders means one output row per order: it sets the query grain. It does not mean the SQL may only mention orders.

Fields, filters, ordering and the time range may all reference other datasets. Dosi collects every dataset the query needs, finds the relationship path to each, and emits the joins:


dosi select --from orders --fields orders.order_id --where "customers.region = 'US'"
customers is never projected, but the filter needs it, so it is joined. The result is still one row per order.

This works because Dosi follows relationships in one direction only: from the many side to the one side. Each hop attaches at most one row of the target to each row you already have, so the grain you asked for survives every join in the path.

Put plainly: an order has exactly one customer, so pulling customers.name in adds a column to the row — it never adds a row to the result. It is a lookup, the way VLOOKUP fills a column in a spreadsheet without changing how many rows the sheet has.

The other direction is where hand-written SQL quietly goes wrong. One customer has many orders, so joining from customers down to orders splits each customer into several rows — and a SUM over any customer-level column then counts that customer once per order. That is the classic double-count, and nothing errors: the query succeeds and the number is simply wrong. Dosi will not emit it. Ask for an order field from --from customers and the query is refused with detail_fanout, which hands back the grain that does answer the question (--from orders) instead of a duplicated result.

Three ways to name a field¶
Spelling	Example	When
bare field	amount	the name is unique across the model
dataset-qualified	orders.amount	the usual case
relationship-prefixed	orders_to_buyer.name	you want to choose the join path
The third form matters when a model reaches the same dataset more than one way. Suppose orders links to regions both directly (where it shipped) and through customers (where the buyer lives). Asking for regions.region_name is genuinely ambiguous, and Dosi says so rather than picking:


dosi select --from orders --fields regions.region_name --format json
{
  "code": "ambiguous_join_path",
  "candidates": [
    "orders -[orders_to_customers]-> customers, customers -[customers_to_regions]-> regions",
    "orders -[orders_to_ship_region]-> regions"
  ],
  "suggested_retry": "name the path you mean by prefixing the field with its relationships, one of: orders_to_customers.customers_to_regions.<field> | orders_to_ship_region.<field>"
}
Paste one of those prefixes and the query runs:


dosi select --from orders --fields orders.order_id,orders_to_ship_region.region_name
Relationship prefixes work in --where too, at any path length, and on the metric plane's --group-by — the spelling is the same everywhere.

Output column names¶
An output column is the reference with the root prefix removed and each remaining . turned into __:

Reference (root orders)	Column
orders.order_id	order_id
customers.name	customers__name
orders_to_ship_region.region_name	orders_to_ship_region__region_name
orders.created_at:month	created_at__month
Root fields stay bare because that is what you asked about; everything else carries where it came from, so two name columns from two datasets cannot collide silently. Two fields that would produce the same column name are a duplicate_output_name error.

When Dosi refuses¶
The engine never returns rows it cannot vouch for. Three refusals are worth recognizing.

detail_fanout — you asked for a dataset that is only reachable against the join direction. One customer has many orders, so projecting an order field from --from customers would turn one customer into several rows:


dosi select --from customers --fields customers.name,orders.order_id --format json
{
  "code": "detail_fanout",
  "suggested_retry": "query at the finer grain instead: --from orders and project the customers fields you need"
}
The fix is in the message: ask the question at the finer grain. --from orders --fields orders.order_id,customers.name gives you the same information, one row per order, with no duplication.

ambiguous_join_path — several paths reach the field. Name one, as above.

metric_in_detail_query — a metric needs an aggregation, a time range and a grouping, none of which a detail query has. The message names the command that does have them.

unknown_query_key — the query carries a key it does not define, usually a misspelling. It is refused rather than ignored, because an ignored distinct or filter returns plausible rows that answer another question. candidates lists the keys the query defines.

Every refusal carries a machine-readable code, the names involved, and a suggested_retry. See Errors.

Time ranges¶
--start-time / --end-time apply a half-open [start, end) window. With no --time-dimension, the range routes to the root dataset's primary time dimension:


dosi select \
    --from orders \
    --fields orders.order_id,orders.amount \
    --start-time 2026-01-01 --end-time 2026-02-01
If the root dataset has no primary time dimension and the projection has no single time field, Dosi asks you to name one rather than guessing.

Level-of-detail fields¶
A detail query can project, filter and sort the root dataset's fixed LOD fields, including fields whose bases aggregate other datasets. Set that host as --from; reading a joined dataset's LOD field is currently refused. Root-row multiplicity is preserved, and --distinct only deduplicates the projected result.

Use --param NAME=VALUE to bind a field base's parameters. A field accepts single values, and a parameter with no receiver is unknown_metric_param. Context/time ranges and ordinary filters have different effects on fixed calculations. Each rollup reads the time range and context on its own dataset, and a condition it cannot read is reported in the result's warnings. See LOD query surfaces, filter scope and independent windows for the complete rules. A complete runnable detail query is part of the separate commerce walkthrough.

Duplicate rows¶
A row is one row of the root, so a list of ids read off a fact table repeats each id once per fact row: one line per login, not one per player. --distinct ("distinct": true on every other surface) drops the rows whose projected values coincide — SELECT DISTINCT:


dosi select --from logins --fields logins.player_id --distinct
It collapses rows; it does not change which rows exist, so a fan-out is still refused rather than deduplicated after the fact.

Row caps¶
--limit defaults to 100 and is clamped to 10000. This applies on every surface, so an agent that forgets a limit gets a sample, not a table scan.

Beyond the CLI¶
The same DetailQuery reaches every surface, with identical semantics and identical errors.


REST
MCP
Python

curl -s localhost:8080/v1/select/execute \
    -H 'content-type: application/json' \
    -d '{"from":"orders","fields":["orders.order_id","customers.name"],"limit":10}'
Also POST /v1/select/compile and POST /v1/select/explain. See the REST API.


Checking the join before trusting the rows¶
--explain prints the logical plan: the root, every join it chose, and the projection.


dosi select --from orders --fields orders.order_id,customers.name --explain
logical plan
Project[order_id, customers__name]
  JoinRelated[orders_to_customers -> customers, left]
    ReadDataset[orders]
Joins are LEFT by default, so a row with no match survives with NULLs rather than disappearing. A relationship may ask for INNER instead — see D-JOIN.

Limits in this version¶
A query may reach each dataset by one path. Two named paths to the same dataset need two table aliases, which do not exist yet, so the engine refuses rather than quietly reading one join twice.
Order keys must name projected columns. Project the field you want to order by.
Metrics and fields cannot be mixed in one query. Run both planes and join the results yourself. A per-entity value is the exception: declare it as a level-of-detail field and read it here.
A level-of-detail field must live on the --from dataset.

Metric attribution¶
Every dashboard raises the same question: why did this metric change? Answering it by hand — or letting an AI agent answer it — normally means a loop of exploratory metric queries: totals for two periods, then one grouped query after another, then joining and comparing the numbers without slipping on segments that appear or disappear between the periods.

Attribution analysis collapses that loop into one call. Give Dosi a metric, the candidate dimensions, and two date windows; the engine runs everything itself and returns a ranked, ready-to-quote decomposition:

which dimension best explains the change,
which segment drove it, and by how much (as a share of the total change),
for ratio metrics, whether the change came from structure or from rate — e.g. "average order value rose because completed orders' own AOV went 75 → 100, even though their share of orders fell".
The numbers are computed by the engine with the method that is exact for each metric's type, and they add up — every decomposition reconciles against the totals, so the explanation is checkable rather than improvised.

Available on every surface: the CLI (dosi attribute), REST (POST /v1/query/attribute), MCP (the attribute_metric tool), and Python (Engine.attribute).

Quick start¶

export DOSI_EXAMPLES=~/.local/share/dosi/examples
duckdb orders.duckdb < "$DOSI_EXAMPLES/orders/seed.sql"
dosi --model "$DOSI_EXAMPLES/orders/model.yaml" attribute \
  --metric revenue \
  --dimensions orders.status,customers.region \
  --baseline 2024-01-01..2024-02-01 \
  --current  2024-02-01..2024-03-01 \
  --db orders.duckdb
Which dimensions?¶
--dimensions is required in practice but not syntactically: omit it and the engine refuses with dimensions_required and hands you the shortlist for that metric, best first, plus a ready-to-paste fragment.


dosi --model model.yaml attribute --metric revenue \
    --baseline 2024-01-01..2024-02-01 --current 2024-02-01..2024-03-01
error: dimensions_required: name the dimensions to analyze; `candidates` lists what
  `revenue` can be grouped by, best first
  candidates: orders.status, customers.region, products.category
  retry with: "dimensions": ["orders.status", "customers.region", "products.category"]
The engine deliberately does not pick for you. Which columns explain a change is a question about the business, and the ranking the analysis itself uses cannot answer it: concentration (max |Δsegment| / |Δtotal|) rises with cardinality, so an engine left to guess puts customer ids above regions — and pays one warehouse query per guess to do it. The same shortlist is available ahead of time from dosi list dimensions --metric <name> (is_dimension: true rows, time columns aside), and what feeds it — declared or inferred — is described under D-DIM.

Windows are half-open ISO ranges (START..END). Use --connection for a named warehouse profile, --where to scope the whole analysis, and --time-dimension when the default time routing is not what you want. A parameterized metric (D-PARAM) takes --param name=value — one value per parameter, never a list — and the result echoes the resolved bindings in comparison_metadata.params.

Over REST or Python the request is the same shape:


import os
from dosi_engine import Engine

engine = Engine(os.path.join(os.environ["DOSI_EXAMPLES"], "orders/model.yaml"))
result = engine.attribute({
    "metric": "revenue",
    "dimensions": ["orders.status", "customers.region"],
    "baseline": {"start": "2024-01-01", "end": "2024-02-01"},
    "current":  {"start": "2024-02-01", "end": "2024-03-01"},
}, db_path="orders.duckdb")
What comes back¶

{
  "metric": "avg_order_value",
  "strategy": "mix_shift",                  // how the engine decomposed it
  "total_change": {"baseline_value": 75.0, "current_value": 76.67,
                   "delta": 1.67, "pct_change": 2.22},
  "dimension_ranking": [                    // ranked root-cause candidates
    {"dimension": "status", "score": 6.0}
  ],
  "top_dimension_values": [                 // the biggest movers
    {"dimension": "status", "value": "cancelled",
     "delta": 10.0, "contribution_pct": 600.0,
     "segment_kind": "entered",             // this segment is new this period
     "drill_down": {"where_sql": "status = 'cancelled'"}},
    {"dimension": "status", "value": "completed",
     "delta": -8.33, "contribution_pct": -500.0,
     "mix_effect": -28.69, "rate_effect": 20.35,
     "baseline_rate": 75.0, "current_rate": 100.0,
     "drill_down": {"where_sql": "status = 'completed'"}}
  ],
  "factor_totals": {"mix_effect": -18.7, "rate_effect": 20.4, ...},
  "warnings": []                            // structured caveats, see below
}
Reading it:

dimension_ranking orders the candidate dimensions by how well each one explains the change — read the first entry as "slice it this way".
contribution_pct is directly quotable: "this segment explains X% of the change". Percentages above 100 (or negative) mean segments moved in opposite directions and partly cancelled out — the response flags this.
segment_kind tells you when a segment is entered (new this period) or exited (gone this period) rather than a normal shift — the cases that silently skew hand-rolled comparisons. A fourth value, fallback, means the mix/rate split is undefined for that segment (zero or negative components); it still carries its whole delta, so the decomposition keeps adding up.
For ratio metrics, each segment's contribution splits into a mix effect (its weight in the population shifted) and a rate effect (its own ratio changed), with the per-segment rates and shares included — the "structure vs performance" narrative reads straight off the response, and the parts always sum to the total change. mix_effect, rate_effect, baseline_share and current_share are present only on normal segments: outside that path the underlying quantities are undefined, and the engine omits the field rather than publishing a number that does not mean what its name says. Check segment_kind (or simply the presence of mix_effect) before reading a share.
per_dimension (not shown) carries the full value-level detail per dimension, including whether that dimension's numbers reconcile with the totals.
How it helps an agent reason¶
The response is designed so an agent can go from "metric moved" to a verified root cause in one or two calls, without doing arithmetic:

No probing loop. The ranking and contributions arrive pre-computed and pre-sorted; the agent quotes them instead of orchestrating and reconciling its own query sequence.
Every finding carries its next step. Each segment includes drill_down.where_sql — a ready-to-paste filter. To go deeper, the agent calls attribute again with that filter (root-cause recursion), or hands it to a metric query to chart that segment's trend.
Caveats are machine-readable. warnings is a list of stable codes — offsetting segments, truncated high-cardinality dimensions, a dimension whose numbers don't reconcile, near-zero total change — so the agent knows which conclusions to soften without parsing prose.
Refusals guide instead of failing. A metric the engine cannot decompose faithfully (window metrics, distinct counts, complex expressions) returns strategy: "unsupported" with a structured reason, so the agent pivots — for a window metric, attribute its base metric over explicit windows — rather than retry-looping. dosi list metrics reports each metric's attribution_strategy up front, so support can be checked before asking.
Metric coverage¶
Additive metrics (term_wise) — sums, counts, and their linear combinations (including linear derived metrics, which additionally get a per-member breakdown of the change): full dimension attribution. An additive constant (SUM(x) + 100) no longer refuses: the variable part decomposes and the constant is reported in affine_constant.
Ratio metrics (mix_shift) — a single ratio of such linear forms, including averages: the structure/rate decomposition shown above.
General expressions (factor_shapley) — products of aggregates, nested ratios, and any other + − × ÷ arithmetic over SUM/COUNT measures: the engine runs one flat Shapley decomposition over the expression's leaf factors and allocates each factor's exact effect to segments in proportion to their share of that factor's change. The response adds a factors block (per-factor baseline/current/effect, summing to the total change exactly), and each segment's delta is its allocated share of the change — for these metrics baseline_value/current_value are the segment's own metric levels (context), not additive terms. Interaction terms split evenly between factors (the Shapley convention; for a product X·Y this is the familiar ΔX·Ȳ + ΔY·X̄ midpoint form). One deliberate exception: a plain ratio keeps the mix_shift structure/rate semantics rather than switching to Shapley — the two conventions differ numerically, and mix_effect/rate_effect are an established narrative.
Filter-derived metrics — attribute normally, and additionally carry filter_breakdown: the exact subset identity Δbase = Δfiltered + Δcomplement with the complement synthesized by the engine.
Offset window metrics (delta / percent_change / period-over- period) — the analysis decomposes the metric's base aggregate between your explicit windows and labels the transform in window_mapping; the transform itself is a per-bucket series, not a two-window comparison.
Not supported — frame/rank/value window metrics and distinct counts return a structured unsupported response instead of a misleading number. dosi list metrics reports each metric's attribution_strategy and an attributable flag (algebra + a usable default time axis) up front.
Levels of detail (D-LOD) — not yet. A metric that reads one (an aggregate over a fixed field, an include or exclude, or a compose over one) returns unsupported with reason level_of_detail naming what it reads, and is attributable: false up front; a candidate dimension that reads one is skipped with a dimension_skipped warning before any statement runs, and a where over one is refused. The reason is structural: a rollup is a function of each window's filter context, and the two-window statement computes it once — decomposing across a level of detail needs a rollup per window (see the D-LOD boundaries). To explain a share (revenue / revenue_grand), attribute the base metric: its mix_effect is the share's movement.
Limits¶
Up to 16 candidate dimensions per call, each analyzed independently; go deeper by recursing with drill_down.where_sql.
Per-dimension values are capped (default 500, max 1000); beyond the cap the dimension is flagged truncated. Under the cap the engine keeps one consistent top-N across both windows, ranked by combined magnitude — a segment that collapsed (or exploded) between the windows survives the cap as a whole pair, never as a half-seen phantom.
Segments present in only one window are handled for you — zero-filled or reported as entered/exited — so the comparison never silently drops them.
The engine issues one statement per dimension (plus one for the totals), each covering both windows at once — the analysis is consistent even on live-ingest warehouses, and a D-dimension call costs 1 + D round trips.
Errors¶
Two failure channels, deliberately distinct:

Bad request — the request itself is malformed (unknown metric, more than 16 dimensions, an invalid window): HTTP 400 / a CLI usage error. dimensions_required — no dimensions named — is one of these, and carries the candidate list; nothing reaches the warehouse.
Not computable — the request is fine but the data cannot support the analysis: denominator_nonpositive (a ratio metric whose denominator is zero or negative over a window) or nonfinite_evaluation (a factor_shapley expression hits a zero divisor on the windows' factor totals). REST returns HTTP 422 with {"error": {"code": "denominator_nonpositive", ...}}, Python raises NotComputableError, and the CLI prints the same structured code — retrying with different windows or filters may succeed, which is exactly what distinguishes it from a bad request.

Metric lineage¶
A semantic model answers "what is revenue?" once; the lineage graph shows where that answer comes from. Dosi projects a compiled model into a graph of four lanes:

lane	node	edge into it
physical table	main.orders	—
dataset	orders	reads_table
atomic metric	revenue	aggregates from each dataset it reads
derived metric	net_revenue	derive_base, compose_member, window_base, window_second from the metrics it builds on
Datasets are joined by join edges, one per relationship. The graph is a pure function of the model: the same model always produces the same bytes, with no timestamps and nothing about the machine that produced it.

Getting it¶

dosi lineage dump --model model.yaml > graph.json
dosi lineage view --model model.yaml            # opens a page in your browser

from dosi_engine import Engine
graph = Engine(model_path="model.yaml").lineage()
graph = Engine(model_path="model.yaml").lineage(redact_sql=True)
view writes one self-contained HTML file — no network access, opens over file:// — with the lanes left to right, badges on the nodes, edges styled by kind, a name filter, and a panel that shows a node's detail and lets you click through its edges. The derived lane is drawn as several columns, one per step of the derive chain: a derived metric sits one column to the right of the rightmost metric it is built from, so a compose chain reads left to right instead of collapsing into one column wired to itself. Without a browser (--no-open, or a server with no display) it prints the file's path instead.

redact_sql keeps the shape and replaces every SQL text — dataset sources, field expressions, metric expressions, derive predicates — and every opaque extension payload with "<redacted>". Names are never redacted.

What a node carries¶
Every node has an id (table:, dataset:, metric: prefixed), a name, a label, badges, and a detail block whose shape depends on the kind.

Dataset: the source (table name, or the query and its pretty-printed form), the tables it reads, keys, the primary time dimension, and every field with its expression, the physical columns behind it, and its label.
Metric: the authored expression and dialect, the datasets and physical columns it reads, the dimensions worth grouping it by, its time dimension, parameters, window and derive declarations, and its semantics — shape, attribution strategy, additivity.
dimensions is what dosi list dimensions --metric recommends, in the same order: primary and foreign keys, and the columns some metric aggregates, are left out, because a graph that lists every reachable field makes the reader sift them out by hand. A name that is not listed is still groupable — the judgement withholds a recommendation, it never blocks an explicit group-by.

label is the display name. Apache Ossie defines one for fields (label on a field, the same property the Databricks, dbt and other converters read as the display name) and none yet for datasets or metrics, so a dataset or metric label always equals its name. Dosi never invents a title: label == name means nobody gave one.

With an ontology¶
Load an Apache Ossie ontology and the graph gains an ontology block: one node per entity, and edges that tie the concepts to the lanes below and above them.


dosi lineage view --mapping flights.mapping.yaml
edge	from → to	what it says
mapping	dataset → concept	the dataset an entity is mapped onto — the one it is identified from, or, for an entity with no table of its own, the datasets its properties are denormalized onto
relationship	concept → concept	a navigable relationship between two entities, with its role and multiplicity; realized_by names the join edges it walks
Both are lineage: where a concept's data is. What an ontology's derived_by rule says about a metric is semantics, so it rides on the metric node, where v2 already keeps dimensions and time_dimension:


"concept": {
  "path": "Airport.average_departure_delay",   // the concept spelling
  "measure": "Flight.departure_delay",         // the value it aggregates
  "group_by": "Airport",                       // its grain
  "via": ["relationship:Flight.departs_from"], // the path that correlates them
  "realized_by": ["join:flight_departs_from"]
}
via is the reason the overlay exists. An ontology can derive two metrics with byte-identical SQL — an airport's average departure delay, and the average departure delay of the flights arriving at it — that differ only in which relationship correlates a flight to the airport. In the plain graph they are indistinguishable; here each names its own path.

path is what describe prints and what query and select accept, so the graph and the query tools speak the same names. Metrics written by hand in the model have no concept; their lineage is the aggregates edge, and the dataset it starts from carries the mapping edge to its entity.

Concept nodes say what their mapping landed on: table (a dataset of its own, one row per object), denormalized (no table of its own; its columns sit on another entity's, whose rows they are a projection of), or unmapped (nothing it declares resolves). The viewer draws the second two with a dashed border.

The overlay carries no SQL, so redact_sql leaves it unchanged. A model with no ontology has no ontology key at all — the payload is exactly what it was before ontologies existed.

Python¶
Engine accepts an ontology the ways the CLI does:


Engine(mapping_path="flights.mapping.yaml")   # the mapping finds the ontology and the model
Engine(mapping_text=yaml_text)                # one file, its documents inline
Engine(model_path="flights/")                 # a directory of all three
lineage() then includes the overlay. Constructs the ontology declares but Dosi cannot lower (an n-ary relationship, an unmapped concept) are reported as RuntimeWarnings at construction, the same channel as compile warnings. An ontology that cannot be parsed or lowered at all raises ModelError with code ontology; the ontology options are keyword-only.

Ontologies (experimental)¶
Experimental

This page documents a feature that is still being shaped. Flags, output and error payloads may change without notice.

Dosi can load an Ossie ontology alongside — or instead of — a core Ossie model. An ontology adds no information a vendor extension could not carry. What it adds is that one model can bear several of them, and they may disagree: operations means the departure airport by "airport delay", ground handling means the arrival one. Both are right, and each is a separate ontology over the same tables.

Everything below is the dosi CLI; Python shows the same calls through the Python binding. Examples use three of the bundled example models (see Verify it works for $DOSI_EXAMPLES): ossie-mini-flights (two concepts), aviation (a hand-written model plus an ontology) and ossie-flights (the upstream reference ontology).

Loading an ontology¶
An ontology reaches Dosi through a mapping document. A mapping names one ontology and one semantic model, and maps the model's columns onto the ontology's concepts. It is the only one of the three documents that points at the others, so it is the one you load.

These flags apply to every command:

Flag	Env	Meaning
--mapping <path>	DOSI_MAPPING	A mapping document
--ontology <path>	DOSI_ONTOLOGY	An ontology document, when the mapping should not find it on its own. Beside a --model directory, it picks that ontology's mappings
--model <path>	DOSI_MODEL	A semantic model. A mapping or ontology document given here is recognized by its structure, and a directory is read as a catalog
Three ways to write it¶
One file. The mapping carries the ontology and the model inline, each under document:. This is the quickest way to start, and ossie-mini-flights is written this way:


version: 0.2.0.dev0
name: mini_flights_mapping
ontology_ref:
  name: MiniFlights
  document:              # the whole ontology document
    version: 0.2.0.dev0
    name: MiniFlights
    ontology: [...]
semantic_model_ref:
  name: mini_flights_model
  document:              # the whole semantic model
    version: 0.2.0.dev0
    name: mini_flights_model
    datasets: [...]
concept_mappings: [...]
Three files. Each reference gives an iri instead, resolved against the mapping's own directory (iri: ./ontology.yaml). ossie-flights and aviation are written this way.

A directory. --model <dir> reads every .yaml, .yml and .json file at the top of the directory, and a reference with neither iri nor document is found by its name. Files that are not Ossie documents, such as a queries.yaml, are skipped. When the directory holds mappings onto more than one ontology, add --ontology <file> to load one of them: only the mappings whose ontology_ref names it are read, along with the model each one names. A mapping that carries its ontology inline is bound to that one and is passed over. If no mapping names the ontology, the load is refused (ontology_ref_unresolved). --mapping does not go with a directory.


dosi describe --mapping $DOSI_EXAMPLES/ossie-mini-flights/mapping.yaml
dosi describe --mapping $DOSI_EXAMPLES/aviation/mapping.yaml
dosi describe --model $DOSI_EXAMPLES/aviation
dosi describe --model ./catalog --ontology ./catalog/flights.ontology.yaml
Note

The inline document: is proposed in the Ossie community but is not yet in the published mapping schema, so the upstream validator rejects it until it lands. Dosi reads it in both engine modes.

These rules hold in every layout:

A reference's name must equal the name of the document it resolves to (ontology_ref_mismatch).
--ontology and --model win over the iri a mapping was written with, and the name is still checked. An inline document together with a flag for the same reference is refused (ontology_ref_conflict).
An iri is a local path or a file:// IRI. Dosi does not fetch documents over the network: resolve those through your catalog and pass the file.
Several mappings onto one model, found in a directory, are merged. A concept mapped twice is refused (ontology_mapping_conflict). Mappings onto different models are refused for now; load one at a time.
The model is the one the mapping names. A complete hand-written model such as aviation's keeps everything it declares, and where it already declares a join with the same columns as an ontology link, the concept walks the model's relationship instead of adding a second one. A model that carries only datasets and fields, such as ossie-flights', gets its keys, relationships and metrics from the ontology.
The embedded layout (deprecated)¶
An ontology may still carry its mapping under ontology_mappings, with the semantic model embedded in it; that is how the format began. --ontology o.yaml loads it, with a legacy_embedded_mapping note, and --model m.yaml --ontology o.yaml replaces the embedded model with m.yaml. One embedded mapping is loaded at a time: split several into mapping documents. Dosi will stop reading this layout when the Ossie specification does.

What loading produces¶
The ontology is lowered to an ordinary core Ossie model, which then goes through the same validate → compile → plan path a hand-written model does. A ConceptMap is kept beside it; it only names things — a concept path to the dataset, field and join path it resolves to.

Ontology construct	What it becomes
identify_by with an object mapping	the dataset's primary_key, unless the model already declares one
Entity → Value relationship	a name for an existing field; temporal value types mark it is_time
Entity → Entity, ManyToOne / OneToOne	a relationship named <owner>_<link>, e.g. flight_departs_from
Entity stored on another entity's table	no join, no primary key, not a query root — see Limitations
Unary relationship with a filtered mapping	a boolean field <owner>_<name>, reachable as Concept.name
derived_by: AGG[<value> WHERE <correlation> GROUP BY <concept>]	a metric <owner>_<relationship> — see Derived metrics
What is not lowered is reported as a note: line on stderr on every call:


dosi describe --mapping $DOSI_EXAMPLES/ossie-flights/mapping.yaml
note: City.state: State has no table of its own; reached without a join
note: Airport.city: City has no table of its own; reached without a join
note: Flight.registers_latitude_series: n-ary relationship recorded as unmapped
...
Browsing: describe and paths¶
dosi describe [concept]¶
With no argument, the concept catalog:


dosi describe --mapping $DOSI_EXAMPLES/ossie-mini-flights/mapping.yaml
ontology MiniFlights
Two airports' worth of ontology, kept small enough to read whole.
concept  identity  out  in  props  root
Airport  code      0    2   3      yes
Flight   id        2    0   2      yes
out and in count links, props counts properties, and root says whether a query can start from the concept.

With a concept name, everything needed to write a query against it:


dosi describe FlightAlert --mapping $DOSI_EXAMPLES/aviation/mapping.yaml
concept FlightAlert
identity: id  (root: flight_alerts)

properties
path                            type                    domain                   time
FlightAlert.id                  AlertId (String)
FlightAlert.type                AlertType (String)
FlightAlert.priority            AlertPriority (String)  one of [P0, P1, P2, P3]
FlightAlert.status              AlertStatus (String)    one of [OPEN, RESOLVED]
FlightAlert.created_at          DateTime                                         yes
FlightAlert.resolved_at         DateTime                                         yes
FlightAlert.resolution_comment  String

links out
name    role    to      multiplicity  join
flight  flight  Flight  ManyToOne     yes

metrics by group-by
group by        metrics
FlightAlert.id  alert_count, open_alert_count, flights_with_open_alert, maintenance_alerts

time dimensions: FlightAlert.created_at, FlightAlert.flight.actual_arrival, FlightAlert.flight.actual_departure, FlightAlert.flight.scheduled_arrival, FlightAlert.flight.scheduled_departure, FlightAlert.resolved_at
Section	What it tells you
properties	Every mapped property as a concept path, with its value type, closed value domain and time flag
links out	Name, role, target and multiplicity. join: none means the target is stored on this table
links in (query from)	Concepts that link here — the side to root a query on
metrics reachable	Metrics the ontology derived, with the group-by and the edges (via) that give them their meaning
metrics by group-by	The model's own metrics this concept can group, each under a group-by that walks from the concept the metric lives on. Every listed group-by is planned before it is printed
time dimensions	Time properties on the concept and one hop out, for a time range
unmapped	Relationships declared but not lowered: n-ary, or links with no mapping
not queryable	Properties bound to no column (row-level derived_by, mostly)
Every path in the tables can be pasted into select or query unchanged. That is the contract that makes the view worth calling.

--format json returns the same content: concept, identity, root, properties (path, type_name, domain, domain_summary, is_time, mapped), links, incoming (query_from), metrics_reachable (name, metric, grain, group_by, via), groupable_metrics (group_by, from, metrics), time_dimensions_reachable, default_projection and unmapped. The catalog is ontology, description, requires and concepts (name, identity, links_out, links_in, properties, queryable, summary).

Without an ontology, describe with no argument lists datasets and metrics instead, and naming a concept is an error:


dosi describe Airport --model $DOSI_EXAMPLES/ossie-mini-flights/model.yaml
error: this model has no ontology, so "Airport" is not a concept; pass --mapping <path>, or use `list` for datasets and metrics
dosi paths --from <concept> --to <concept>¶
Every named path between two concepts, up to three hops, with the roles walked and the pairs the ontology declares equivalent:


dosi paths --from Flight --to Airport --mapping $DOSI_EXAMPLES/aviation/mapping.yaml
paths Flight → Airport
path                              roles                        hops  multiplicity  join
Flight.arrives_at                 destination                  1     ManyToOne     yes
Flight.departs_from               origin                       1     ManyToOne     yes
Flight.follows_route.destination  follows_route → destination  2     ManyToOne     yes
Flight.follows_route.origin       follows_route → origin       2     ManyToOne     yes
equivalent: Flight.departs_from ≡ Flight.follows_route.origin  (Flight.follows_route.origin == origin)
equivalent: Flight.arrives_at ≡ Flight.follows_route.destination  (Flight.follows_route.destination == destination)
Two paths to one target is exactly the case a bare dataset.field reference cannot resolve (ambiguous_join_path); the role names are what separate them. Equivalences come from relationship requires of the form Concept.link.role == role. JSON adds unique: true when exactly one path exists.

Concept paths in queries¶
With an ontology loaded, select and query accept concept spellings wherever they take a field:

Where	Spelling
select --from	a concept name: FlightAlert
fields, --group-by	Concept.property, Concept.link.property, Concept.link.link.property
--where	the same paths; string literals are never rewritten
--time-dimension	a time property path: Delay.flight.scheduled_departure
--metrics	a derived metric's concept path: Airport.average_departure_delay
select --from, --group-by	a derived subtype: PremierCustomer — the parent's rows that satisfy its rule, or its membership as a boolean dimension
--where, D-DERIVE filter.where	<Concept> is [not] <Subtype>: Customer is PremierCustomer, Membership.held_by is not PremierCustomer
Each resolves to the dataset, field and relationship path the planner already understood, so the SQL is identical to the dataset spelling:


dosi select --mapping $DOSI_EXAMPLES/aviation/mapping.yaml \
    --from FlightAlert \
    --fields "FlightAlert.id,FlightAlert.priority,FlightAlert.flight.arrives_at.code" \
    --where "FlightAlert.status = 'OPEN' AND FlightAlert.priority = 'P1'"
SELECT flight_alerts.alert_id AS alert_id, flight_alerts.priority AS priority, airports.airport_code AS flight_alerts_flight__flights_arrives_at__airport_code FROM flight_alerts AS flight_alerts LEFT JOIN flights AS flights ON flight_alerts.flight_id = flights.flight_id LEFT JOIN airports AS airports ON flights.destination_airport_code = airports.airport_code WHERE flight_alerts.status = 'OPEN' AND flight_alerts.priority = 'P1' LIMIT 100
Output columns keep the planner's names, and --order takes those names, not concept paths.

Derived metrics¶
The correlation path of a derived_by rule is what gives the metric its meaning. Two such metrics can compile to byte-identical SQL and still mean different things:


duckdb mini.duckdb < $DOSI_EXAMPLES/ossie-mini-flights/seed.sql
dosi query --mapping $DOSI_EXAMPLES/ossie-mini-flights/mapping.yaml \
    --metrics Airport.average_departure_delay --group-by Airport.code --execute --db mini.duckdb
code  airport_average_departure_delay
ATL   16
ORD   30
DCA   20
SFO   32
4 rows

dosi query --mapping $DOSI_EXAMPLES/ossie-mini-flights/mapping.yaml \
    --metrics Airport.average_inbound_departure_delay --group-by Airport.code --execute --db mini.duckdb
code  airport_average_inbound_departure_delay
ATL   30
DCA   16
SFO   35
ORD   10
4 rows
Both are AVG(FLIGHT.dep_delay). Named by its concept path, each metric groups through its own edge — the first through the departure airport, the second through the arrival airport. Two consequences are deliberate:

Asking for the model metric by its flat name (airport_average_departure_delay) names no path, so grouping by Airport.code is ambiguous_join_path: the model alone cannot tell the two apart.
Asking for both at once names two paths that disagree, and the ambiguity is reported rather than resolved to one side.
A rule is lowered whole or not at all. A WHERE that is not a correlation (WHERE Flight.departure_delay > 10), or a correlation over a link that does not resolve, leaves the rule as text with a note:; no metric is emitted.

When a group-by is rooted on the wrong side¶
A metric grouped through a path that starts on a concept it cannot reach is refused by the planner. With an ontology loaded, the refusal names the repair — the same path, walked from the concept the metric lives on:


dosi query --mapping $DOSI_EXAMPLES/aviation/mapping.yaml \
    --metrics delayed_flights --group-by Flight.arrives_at.code
error: InvalidRelationshipPath: a relationship path pins this query to dataset "flights", which has no many→one path to "delays" — where measure "delays_flight_id_count_distinct" of metric "delayed_flights" lives
  candidates: Delay.flight.arrives_at.code
  hint: root the group-by on the concept the metric lives on: Delay.flight.arrives_at.code
Every candidate is planned before it is offered; at most three, shortest first. Under --format json they are candidates and suggested_retry.

Derived subtypes¶
A rule-defined subset of an entity — Premier customers, delayed flights, high-risk orders — is a derived subtype: an EntityType that extends one parent and carries one concept-level derived_by rule, in the specification's own words (its example is Employee extends Person, derived_by EXISTS(Person.earns)). The rule is written once, over the parent's properties; the subtype is then a query root, a dimension and a filter, and may add relationships of its own.


- concept: PremierCustomer
  type: EntityType
  extends: [Customer]
  description: Premier qualification. Eligibility, not enrollment.
  derived_by:
  - 'Customer.margin >= 3000 AND Customer.order_count >= 6 AND Customer.return_rate <= 0.20'
  relationships:
  - name: qualifies_for
    roles: [{concept: Membership}]
    multiplicity: ManyToOne
The subtype needs no object_mappings: it inherits the parent's identity, table and properties. A relationship it adds is mapped with link_mappings under its own concept and exists only there (Customer.qualifies_for stays unknown).

What lowering makes of it. One boolean field on the parent's dataset, named after the concept (customers.premier_customer), whose expression is the rule translated one hop — Customer.margin to the column that property maps to, a unary fact to its field, Customer is Gold to Gold's flag — wrapped in COALESCE(…, FALSE). A chain (PremierPlus extends [PremierCustomer]) ANDs the parent's flag in. The field is a dimension (is_dimension: true) and may be named directly by its dataset spelling; the planner never hears the word "subtype".

The rule's grammar is boolean algebra over the parent's mapped properties and nothing else — aggregation happens first, through a D-LOD fixed(metric, key) field, so margin and order_count above are properties:


expr  := expr AND expr | expr OR expr | NOT expr | '(' expr ')' | atom
atom  := attr cmp value | attr IS [NOT] NULL | attr [NOT] IN '(' value, … ')'
       | attr                                  -- a boolean (unary) property
       | Concept 'is' [NOT] Concept            -- another subtype of the parent
value := number | 'string' | TRUE | FALSE | ':' name
attr  := Concept '.' name                      -- one hop: the parent's own property
cmp   := = | == | <> | != | < | <= | > | >=
Anything else — a function, an aggregate, a two-hop path (Customer.orders.amount), EXISTS — is ontology_subtype_rule_unsupported with a hint, never guessed at.

On the query surface:


dosi describe PremierCustomer --mapping northstar.yaml
concept PremierCustomer  (extends Customer)
rule: Customer.margin >= 3000 AND Customer.order_count >= 6 AND Customer.return_rate <= 0.20
identity: id  (root: customers, inherited from Customer)
…
dosi select --from PremierCustomer --fields PremierCustomer.name,Customer.margin   # the parent's rows, filtered
dosi query --metrics booked_revenue --group-by PremierCustomer                      # true / false
dosi query --metrics booked_revenue --where "Customer is PremierCustomer"
dosi select --from Membership --fields Membership.held_by.name --where "Membership.held_by is not PremierCustomer"
is is sugar the ontology resolves before planning (customers.premier_customer = TRUE); it never reaches the SQL grammar, and IS NULL is left to SQL. A name after is that is not a subtype of that concept is unknown_subtype, with the concept's subtypes as candidates. describe Customer lists derived_subtypes. A D-DERIVE filter may use the same spelling: "where": "Customer is PremierCustomer" — on a fact metric too, since a guard on a dimension the fact reaches many→one is admitted (S-ATTR-3′).

Thresholds as parameters. A rule may hold :name placeholders, declared on the mapping document under concept_params and bound at query time with --param margin_min=4000 — a what-if that changes no file. See D-PARAM. A literal rule is standard Ossie and works under --osi-basic; a placeholder rule is a DATUS feature, and basic mode records the subtype unmapped (describable, not queryable).

The flag column takes no verbalizes of its own, and the subtype's rows are evaluated in the query's time context like every level of detail: select --from PremierCustomer --time-range … is the Premier set within the window.

Value domains¶
A ValueType whose requires enumerates a closed set makes a filter literal checkable before the query is compiled:


dosi select --mapping $DOSI_EXAMPLES/aviation/mapping.yaml \
    --from FlightAlert --fields FlightAlert.id --where "FlightAlert.status = 'unresolved'"
error: value_not_in_domain: "unresolved" is not a value of FlightAlert.status; its domain is a closed set
  allowed: OPEN, RESOLVED

{
  "code": "value_not_in_domain",
  "violations": [
    {
      "reference": "FlightAlert.status",
      "literal": "Open",
      "allowed": ["OPEN", "RESOLVED"]
    }
  ]
}
This is what profiling the warehouse cannot do: a probe finds the values a column holds today, but only the declaration asserts the set 

Dosi
Fast result transfer (Arrow)



 
Dosi
Home
Get started
Get started
Quickstart
What is Apache Ossie & Why Dosi
Grain: what one row means
How metrics compose
Install manually
Tutorial: the dosi CLI
Guides
Guides
Dosi in agents
Dosi in agents
The Dosi plugin
The Dosi plugin
The Dosi plugin
Claude Code
Codex
DeepSeek Harness
Other agents
Run the MCP server yourself
Connect a warehouse
Connect a warehouse
DuckDB
SQLite
MySQL
TiDB
StarRocks
Doris
PostgreSQL
Oracle
SAP HANA
ClickHouse
Trino
Amazon Athena
Snowflake
Redshift
BigQuery
Databricks
Query metrics
Query detail rows
Metric attribution
Metric lineage
Ontologies
Fast result transfer (Arrow)
Table of contents
Which engines support it
Do I need to change my configuration?
Build features
Which APIs get faster?
Extensions
Extensions
Overview
Window metrics
LOD
LOD
Understand and start using LOD
Commerce modeling walkthrough
Authoring reference
Parameterized metrics
Derived metrics
Cross-fact alignment
Time storage format
Versions & compatibility
Reference
Reference
CLI
SQL policy providers
REST API
MCP server
Errors
Python API
Semantics contract
Benchmarks
Release notes
Fast result transfer (Arrow)¶
Dosi can move query results as Apache Arrow — columnar end to end for engines whose wire format is already columnar. The rows are the same; what changes is how much encoding and parsing sits between the warehouse and your client. This page covers which engines support it, what to configure, and which APIs get faster. Connector setup is in Connect a warehouse.

Which engines support it¶
Engine	Arrow path
DuckDB	in-process, zero-copy
ClickHouse	FORMAT ArrowStream over the same HTTP connection
StarRocks, Doris	Arrow Flight SQL (opt in per profile, see below)
Databricks	ARROW_STREAM results
Snowflake, BigQuery	not yet — both execute normally over their REST APIs and return rows
Every other connector	rows only — their wire formats are row-oriented, so converting to Arrow on the client would gain nothing
Do I need to change my configuration?¶
Mostly no. Existing connection files keep working unchanged; the row path stays the default everywhere and behaves identically.

Engine	Config change to get Arrow
DuckDB	none — in-process Arrow is the engine's native path
ClickHouse	none — same HTTP profile; the adapter requests FORMAT ArrowStream itself
Databricks	none beyond the normal profile (host, warehouse, password: = PAT)
StarRocks	add one key: arrow_flight_port: 9408 (the FE arrow_flight_port; needs SR ≥ 3.5.1). Key present = Flight SQL; key absent = MySQL wire
Doris	add one key: arrow_flight_port: 8070 (the FE arrow_flight_sql_port; Doris ≥ 2.1.5 recommended). Same opt-in rule
Example:


datasources:
  prod-sr:
    type: starrocks
    host: sr.internal
    port: 9030                # MySQL-wire port, still used as fallback/seeding
    arrow_flight_port: 9408   # presence opts this profile into Arrow Flight SQL
    username: osi
    password: ${SR_PASSWORD}
    database: analytics
There is deliberately no result_format: knob: each adapter has one native decode, and the egress format is chosen per request (Accept header / CLI flag), not per datasource.

Build features¶
The release binaries already include every Arrow path; the -lean binaries include all of them except in-process DuckDB. Building from source, these Cargo features turn them on:

Feature	What it turns on	In default build?
exec-duckdb	in-process libduckdb, Arrow-native	yes (--no-default-features = the lean build, which shells out to a duckdb CLI)
exec-http-arrow	ClickHouse FORMAT ArrowStream	no (implies exec-http)
exec-flightsql	StarRocks/Doris Arrow Flight SQL client	no
exec-databricks	Databricks Statement Execution API, inline JSON rows	no
exec-databricks-arrow	Databricks ARROW_STREAM + EXTERNAL_LINKS	no (implies exec-databricks)
exec-all	all of the above	—
dosi-server arrow	Arrow IPC responses on /v1/query/execute	yes (server default)
Which APIs get faster?¶
POST /v1/query/execute with Accept: application/vnd.apache.arrow.stream — the headline win. Batches stream from the warehouse adapter straight into the HTTP body as Arrow IPC: no per-value JSON encoding on the server, no JSON parsing in the client, and columnar consumers (Polars/pandas/DataFusion/ another dosi) read it zero-parse. The JSON response stays byte-identical for existing clients. Benefit scales with result size: negligible for a 10-row aggregate, decisive for high-cardinality group-bys and detail/export queries (10⁵+ rows).
dosi query --execute --format arrow — Arrow IPC on stdout for piping into DuckDB/Polars/anything, skipping the table/JSON rendering entirely.
StarRocks/Doris large reads — Flight SQL reads bypass FE row forwarding (client pulls columnar results from BEs in parallel); vendors measure 20–160x vs the MySQL protocol on large result sets. Small aggregates: ~no change.
ClickHouse — server-side Arrow serialization (lz4-framed) replaces JSON encode/decode on both ends; also removes float/decimal text round-trips.
DuckDB (default engine) — in-process zero-copy replaces spawning a CLI per query and parsing its JSON: lower latency on every --execute, and in-memory databases persist across statements in one session.
What does NOT get faster: compile-only APIs (/v1/query/compile, explain) — they never touch results; small metric queries on row-wire engines (MySQL/TiDB/Postgres/Trino) — unchanged by design.
Every Arrow path is checked against the test corpus before it ships: Arrow results ≡ row results ≡ the DuckDB reference.



