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
See also¶
Grain: what one row means — the grains a query and a level of detail work at
Derived metrics — filter and compose
Window metrics — shifts, frames, ranks
Understand and start using LOD — exclude, fixed, include
Query metrics — the query and its response
 Back to top
Previous
Grain: what one row means
Next
Install manually
Copyright © Datus · Licensed under the Elastic License 2.0
Dosi
What Is Apache Ossie? And Why Dosi?



 
Dosi
Home
Get started
Get started
Quickstart
What is Apache Ossie & Why Dosi
Table of contents
Apache Ossie at a glance
The problem: everyone redefines "revenue"
What an Apache Ossie model looks like
One model per document
An interchange, not just a format
How Dosi relates to Apache Ossie
What Dosi does
What an AI agent needs from a metric system
What makes the numbers trustworthy
Who it's for
FAQ
Is OSI the same as Apache Ossie?
Does Dosi work with any Apache Ossie model?
Why do some Dosi flags still say osi?
Next steps
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

Skip to content
Dosi
Grain: what one row means



 
Dosi
Home
Get started
Get started
Quickstart
What is Apache Ossie & Why Dosi
Grain: what one row means
Table of contents
The example
Four grains in one query
The rule: aggregate from fine to coarse
Two facts in one answer: align, don't join rows
A metric at its own grain: LOD
LOD and alignment together
Reading grain errors
Where to next
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

Limits¶
No time spine: a period with no rows has no row, and nothing invents it. A shift lines periods up by calendar, so a missing previous month reads NULL.
A window can read another window's output three levels deep; a fourth is refused.
Computing parts apart is enabled on DuckDB, PostgreSQL, Hologres, GaussDB, DWS, MySQL, TiDB, Trino, Athena, Snowflake, BigQuery, Databricks and ClickHouse. On other dialects a query that needs it is refused.

Skip to content
Dosi
How metrics compose



 
Dosi
Home
Get started
Get started
Quickstart
What is Apache Ossie & Why Dosi
Grain: what one row means
How metrics compose
Table of contents
The example
A metric is a function of the query
The building blocks
The order of blocks
What is still refused
Where a condition applies
Limits
See also
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
See also¶
Grain: what one row means — the grains a query and a level of detail work at
Derived metrics — filter and compose
Window metrics — shifts, frames, ranks
Understand and start using LOD — exclude, fixed, include
Query metrics — the query and its response
 Back to top
Next
Install manually
Copyright © Datus · Licensed under the Elastic License 2.0
