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