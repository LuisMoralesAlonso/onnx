Dosi
Overview



 
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
Extensions
Extensions
Overview
Table of contents
What each extension does
Why your model stays valid Ossie
Writing an extension
v — the declared version
requires — keys that must not be dropped
Simple extensions
D-JOIN — relationship join type
D-FILL — metric null-fill for missing merge groups
D-TIME — primary (aggregation) time dimension
D-GRAIN — native time granularity
D-DATASET — metric home dataset
D-DIM — which fields are offered as dimensions
D-MEASURE — explicit measure name
Larger extensions
D-WINDOW — window metrics
D-LOD — levels of detail
D-PARAM — query-time parameters
D-DERIVE — derived metrics
D-CONFORM — conformed relationships between facts
D-FORMAT — time storage format
Validation & engine modes
Non-goals / boundaries
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
Datus extensions to Apache Ossie¶
Status: draft, in use by Datus (dosi-engine + the Datus generation agent).
Spec version: datus-ext/1 (the major series)
Current version: 1.12 — see Extension versions for what each minor added and how a version bump is decided. Any engine can report the version it implements: dosi info, GET /v1/capabilities, or dosi_engine.DATUS_EXT_VERSION.
Scope: semantics that Apache Ossie (formerly OSI) core spec 0.2.0.dev0 cannot yet express, carried inside the Ossie-sanctioned custom_extensions field so a document stays fully Ossie-valid and every non-Datus consumer ignores them.
This page is the overview: what each extension controls, how an extension is written, and the full contract of the simple ones. The larger extensions — window metrics, levels of detail, parameters, derived metrics, cross-fact alignment, time storage formats — each have a page of their own, listed under Larger extensions.

Separately from model semantics, dosi-engine's connection configuration also speaks Datus: profiles use the agent.yml datasources: vocabulary and a full agent.yml can be passed as --connections directly — see connectors.md.

This is a private vendor extension: it does not need to be accepted by the Ossie working group to be usable. Where an extension generalizes a gap that belongs in the core spec (time granularity, cumulative windows), the long-term home is an Ossie RFC; this document is the interim vehicle Datus ships against today.

What each extension does¶
Every extension is optional. Add one only where you want to change the default; a model without any behaves exactly as pure Ossie says it should.

Extension	Carrier	Key	What it controls	Default when absent
D-JOIN	Relationship	join_type	keep or drop fact rows with no match when a join reaches a dimension	left — orphan rows are kept
D-FILL	Metric	fill_nulls_with	the number a group with no rows shows	a bare count fills 0, anything else NULL
D-TIME	Dataset / Metric	time_dimension	which time column is the business time axis (metric_time)	the dataset's single is_time field, else none
D-GRAIN	time Field	time_granularity	the grain a column is stored at	inferred from the type or format, otherwise any grain
D-DATASET	Metric	dataset	the home dataset of a COUNT(*)	derived from the SQL, else count_star_needs_dataset
D-DIM	Field	is_dimension	whether a field is offered as a dimension	inferred from keys, relationships and aggregated columns
D-MEASURE	Metric	measure	the name of the metric's measure	generated from the aggregate
D-WINDOW	Metric	window	period-over-period, rolling, cumulative, rank and value windows	the plain base aggregate
D-LOD	Field / Metric	lod, derive type lod	calculate at one grain, display at another	a field is a stored column
D-PARAM	Metric / Field	params	window widths and filter literals bound at query time	every slot is the literal it declares
D-DERIVE	Metric	derive	a metric built from other metrics (filter, compose)	a self-contained metric
D-CONFORM	Relationship	cardinality	align two facts on a shared dimension without joining them	many_to_one — the edge is a join
D-FORMAT	time Field	time	how a time column stores dates (20260907, 202609)	a native DATE / TIMESTAMP is assumed
dosi info lists every key the engine reads, with the version that introduced it and what ignoring it costs.

Why your model stays valid Ossie¶
Ossie's schema is strict (additionalProperties: false everywhere), so we cannot add bare fields like relationship.join_type. But the spec provides a first-class escape hatch on every major object:


custom_extensions:
  - vendor_name: <string>
    data: <JSON string>
custom_extensions is present on SemanticModel, Dataset, Field, Relationship, and Metric. The upstream ossie validator accepts it, so a document carrying Datus extensions passes upstream validation unchanged. A consumer that doesn't understand vendor_name: DATUS simply skips it — the model still means exactly what pure Ossie says it means. The Datus extensions only refine engine behavior at points Ossie leaves undefined; they never change what a metric or dimension is.

Writing an extension¶
A Datus extension is one custom_extensions entry:


custom_extensions:
  - vendor_name: DATUS
    data: '{"v": "1.1", "<option>": <value>}'
vendor_name: canonically the uppercase constant "DATUS" (matching the Ossie spec's uppercase enum style, e.g. ANSI_SQL). Matching is case-insensitive; the historical lowercase datus keeps working.
data: a JSON string (per the Ossie spec data is a string, not an object) decoding to a single JSON object — the payload. At most one DATUS entry per object. Two envelope keys are reserved: v and requires.
Consumption rules (dosi-engine):

Absent → the documented Ossie default (overview). Extensions are purely additive; a model with none behaves exactly as today.
Unknown payload keys are ignored, not rejected — so a newer Datus agent can emit keys an older engine doesn't read without breaking it (forward-compat). The exception is requires, below.
Unknown vendor_name (anything but DATUS, compared case-insensitively) is ignored.
A malformed DATUS payload (not JSON, or a known key with the wrong type) is a structured error (invalid_datus_extension), not a silent default — a Datus-authored document is held to the Datus contract. Compilation still continues with the default, so one pass reports every problem in the model.
v — the declared version¶
v is MAJOR.MINOR. The string form is canonical ("v": "1.1"); a JSON number is accepted for convenience ("v": 1.1, and the bare integer "v": 1 meaning 1.0).

A JSON number can only carry minors 0–9. 1.10 and 1.1 are the same f64, so a number cannot express minor 10; a number with two or more fractional digits is rejected outright with a message telling you to use the string form. Once the current minor reaches 10, write "v" as a string always.

v is optional and omitting it is always safe — an unversioned payload behaves exactly as it did before the version gate existed, and produces no diagnostic. The version is declared per custom_extensions entry, not per document: a model-level v does not become a default for its datasets, fields, relationships, or metrics.

What stamping a version buys you, given the engine's own version:

Situation	Engine's response	Code
v absent or null	silent — behaves as before the gate	—
v malformed (bad string, wrong JSON type, number with ≥2 decimals)	error	invalid_datus_extension
v below 1.0, or a major newer than the engine's	error	unsupported_datus_ext_version
minor newer than the engine's	warning naming the keys that were dropped; everything the engine knows still applies	datus_ext_version_ahead
a key newer than the v you declared	warning; the key is honored anyway	datus_ext_key_newer_than_declared
minor older than the engine's, keys all within it	silent	—
A future major is the one blanket rejection: a major bump means at least one key changed meaning, and an engine that predates the change has no way to know which — reading the payload anyway would produce quietly wrong numbers. Minors are additive by construction, so they only ever warn.

requires — keys that must not be dropped¶

data: '{"v": "1.4", "requires": ["semi_additive"], "semi_additive": {"reduce": "last"}}'
requires is an array of key names the producer declares must not be silently ignored. A consumer that does not implement a listed key raises datus_ext_key_required naming it, instead of dropping it.

This exists because a consumer cannot judge the danger of a key it has never heard of. "Unknown keys are ignored" is the right default for a key like join_type, where ignoring it falls back to a documented behavior — but the wrong default for a key like semi_additive, where ignoring it means summing an end-of-period balance and returning a number that is simply wrong. The producer knows which is which; requires is how it says so.

Emit requires for every key whose ignore-impact is silent (see the versioning policy) and for nothing else: letting an older engine degrade is better than refusing to serve it. dosi info / GET /v1/capabilities report each key's impact.

In Datus mode, a non-null silent key omitted from requires emits datus_ext_requires_missing, even when v is absent. The model still compiles; add the named key to requires to protect older consumers.

When a version is bumped, what each minor added, and which keys are still to come are recorded on Extension versions: the versioning policy, the change log and the roadmap.

Simple extensions¶
Each of these is one key with a small value, and its whole contract fits in its section.

D-JOIN — relationship join type¶
Where: a Relationship's custom_extensions. Payload key: join_type, enum "left" | "inner". Default (absent): "left".

Say you measure revenue on line_items and want it broken down by orders.status. Some line items point to an order that isn't in your orders table — a late-arriving row, a soft-deleted order, a data-quality gap. join_type decides what happens to their revenue when a fan-out branch follows this relationship (many→one) to join the "one" side in, reaching a dimension or an off-base measure:

"left" (default): keep rows on the many side that have no match — orphan fact rows survive and land in the NULL group of any dimension drawn from the missing side. Reconciliation / audit semantics: every dollar is accounted for.
"inner": drop unmatched many-side rows. Attribution semantics — a measure is only counted where the entity it attributes to actually exists.

relationships:
  - name: line_items_to_orders
    from: line_items
    to: orders
    from_columns: [order_id]
    to_columns: [order_id]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.0", "join_type": "inner"}'
Same query, orphan revenue of 50:

join_type	status	revenue
left (default)	paid	300
(blank)	50 ← orphans kept
inner	paid	300
(orphan row dropped)
This makes the INNER-vs-LEFT choice an explicit property of the relationship instead of a hard-coded generator default (S-JOIN in the semantics contract), so every query that follows the join gets the same answer. dosi query --explain shows each join's kind, so you can confirm the extension took effect.

D-FILL — metric null-fill for missing merge groups¶
Where: a Metric's custom_extensions. Payload key: fill_nulls_with, a JSON number. Default (absent): engine default — a bare COUNT metric fills 0 (S-FAN-6 in the semantics contract); every other metric keeps NULL.

You ask for order_count and signups by city, and a city had signups but no orders this period. After the multi-branch merge, a group present in only some branches projects NULL for the metrics evaluated in the missing branch. A plain count already reads 0 there — a count of nothing is zero — but a SUM stays blank, because "the sum of no rows" could mean 0 or unknown. fill_nulls_with: <n> decides per metric: it COALESCEs this metric's output to <n> (typically 0) in both merge and single-branch plans, for any metric kind (SUM, ratio, expression), and takes precedence over the count default.


metrics:
  - name: revenue
    expression:
      dialects:
        - dialect: ANSI_SQL
          expression: SUM(orders.amount)
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.0", "fill_nulls_with": 0}'
revenue and signups by city, where Denver has signups but no orders:

city	revenue (default)	revenue (fill_nulls_with: 0)
Austin	900	900
Denver	(blank)	0
Scope guard (unchanged from S-FAN-6): fill is applied to the metric's output, never to a measure embedded in a ratio/expression — a count denominator must not become a literal 0 divisor. For a ratio, fill the ratio, and the engine still evaluates it null-safely.

Under a window stage the rule is the same: the merged measure column stays NULL, and the count default is applied where the bare count metric is projected as an output. A count that a compose or ratio reads as a member — or that sits inside a window.base series — keeps NULL, so a count denominator never becomes a literal 0 divisor. Adding a window metric to a query never moves a selected count from 0 to NULL.

D-TIME — primary (aggregation) time dimension¶
Where: a Dataset's or a Metric's custom_extensions. Payload key: time_dimension, a string name reference — on a dataset, the name of one of its own is_time fields; on a metric, "field" or "dataset.field". Default (absent): a dataset with exactly one is_time field gets it as implicit primary time (this inference reads no extension, so it applies in basic mode too); zero or several is_time fields → no primary time.

A query says "revenue and inventory moves, by month, last quarter" — but which column is time? orders has order_date and ship_date; the inventory table has move_date. time_dimension declares which time column is a table's or a metric's business time axis — the column the engine uses when a query asks for a time range or grain without naming a column. Resolution per metric (S-TIME-5):

the metric's own time_dimension (highest precedence);
else the unique primary time among the metric's datasets (explicit dataset time_dimension, else the single-is_time inference).
A metric-level bare "field" resolves against the metric's own datasets first (time-field names like etl_dt recur across datasets), widening to the whole model only if none of them has the field.


datasets:
  - name: orders
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.1", "time_dimension": "order_date"}'
    fields:
      - name: order_date
        dimension: { is_time: true }
      - name: ship_date
        dimension: { is_time: true }
metrics:
  - name: shipped_revenue          # same SUM, but on the shipping axis
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: SUM(orders.amount) }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.1", "time_dimension": "ship_date"}'
Queries consume this through the reserved name metric_time (--group-by metric_time:month, --time-dimension metric_time):


dosi query --metrics revenue,moves --group-by metric_time:month
Each metric is truncated on its own table's primary time column and the results align on the shared metric_time__month output. A time range with no named dimension and no time item in the group-by filters each metric's primary time — per aggregation branch, on that branch's own column. A metric with no resolvable primary time yields no_primary_time_dimension. Metrics on one base with different primary times are not an error: each time axis gets its own branch over that base, and the branches merge on the group keys, so each metric returns what it returns alone.

The reference must exist and have an effective is_time: true (explicit or from the core datatype default), else invalid_datus_extension.

D-GRAIN — native time granularity¶
Where: a Field's custom_extensions (the field must be dimension.is_time: true, explicitly or from the core datatype default). Payload key: time_granularity, enum "day" | "week" | "month" | "quarter" | "year". Default (absent): inferred from time.format or core datatype: Date (day); otherwise unknown — any query-time grain may be requested.

Declares the grain the column is stored at (a monthly snapshot's etl_dt, say). Requesting a strictly finer grain (:day on a month column, via an explicit group-by item or via metric_time) is a structured grain_too_fine error instead of silently wrong data. Truncation to the native grain or coarser is unchanged. Interim vehicle for the upstream RFC's dimension.time.granularity. A column that stores dates as strings or integers declares its grain together with its layout — see D-FORMAT.


fields:
  - name: etl_dt
    dimension: { is_time: true }
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.1", "time_granularity": "month"}'
D-DATASET — metric home dataset¶
Where: a Metric's custom_extensions. Payload key: dataset, a string naming a dataset of the model. Default (absent): attribution is derived from the metric SQL alone.

A COUNT(*) names no column, so in a multi-dataset model with no other aggregate pinning a dataset, Ossie leaves its attribution undefined and the engine errors (count_star_needs_dataset). The hint resolves exactly that tie. It never overrides SQL-derived attribution — a metric whose aggregates name columns keeps their datasets regardless of the hint (extensions never redefine what the SQL says). An unknown dataset name is invalid_datus_extension.


metrics:
  - name: chat_message_count
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "COUNT(*)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.1", "dataset": "chat_record"}'
D-DIM — which fields are offered as dimensions¶
Where: a Field's custom_extensions. Payload key: is_dimension, a boolean. Default (absent): the engine infers the field's role (see below).

Ossie types every non-time field as a regular dimension, so a model cannot say that a column is a measurement (unit_cost), an identifier (c_customer_id), free text, or PII. Grouping by such a column is not a structural error — the SQL runs, the numbers are right, and the analysis is meaningless. is_dimension is how the model says so once, at modelling time, instead of every caller re-deciding it at every call.

It is a listing signal, not a permission:

Written	Effect
absent	the engine infers (below)
false	the field is not recommended: list_dimensions marks it is_dimension: false, and attribution does not offer it as a candidate. Naming it in group_by still works.
true	the field is recommended even where inference would exclude it — the way to group a numeric column into buckets
Because it never reaches the planner, ignoring the key changes no number; it only costs a caller a worse shortlist (documented impact).


datasets:
  - name: store_sales
    fields:
      - name: ss_quantity            # a measurement no metric aggregates
        expression: { dialects: [{ dialect: ANSI_SQL, expression: ss_quantity }] }
        custom_extensions:
          - vendor_name: DATUS
            data: '{"v": "1.7", "is_dimension": false}'
      - name: i_current_price        # numeric, but analyzed in price bands
        expression: { dialects: [{ dialect: ANSI_SQL, expression: i_current_price }] }
        custom_extensions:
          - vendor_name: DATUS
            data: '{"v": "1.7", "is_dimension": true}'
Inference when the key is absent. The engine uses only model structure — it never probes the warehouse (a COUNT(DISTINCT …) on a high-cardinality column is not a cost the engine may impose). A field is not offered when it is a primary-key member, a single-column unique key of the metric's own home dataset, a column named in a many-to-one relationship's join pair (a D-CONFORM pairing is not a join; its columns are the dimension itself), or a column some metric measures — the value read by a SUM/AVG/STDDEV*/VAR*. Everything else is offered, and list_dimensions reports which rule applied, as source.

Three aggregates that mention a column without measuring it, and therefore leave it a dimension:

COUNT — COUNT(DISTINCT region) ("how many active regions") is an ordinary metric over an ordinary dimension. This holds for the metric being listed too: the answer for a field does not depend on which metric you ask about.
MIN / MAX — they order strings and dates perfectly well; MIN(activity_code) picks one identifier, it does not measure it.
A condition inside an aggregate — SUM(CASE WHEN is_instant THEN 1 ELSE 0 END) counts rows where a flag holds; the flag stays a dimension. Only the THEN/ELSE values are measured. A D-DERIVE filter folds into exactly this shape, so it behaves the same way.
Inference cannot see a business key that is neither a declared key nor a relationship column, a numeric column no metric reads, free text, or PII — those are exactly the fields worth declaring.

D-MEASURE — explicit measure name¶
Where: a Metric's custom_extensions. Payload key: measure, {"name": <identifier>}. Default (absent): the sanitizer-generated stem (S-MEASURE-1).

Names the metric's single synthesized measure explicitly. Measure names are otherwise derived from the aggregate argument (S-MEASURE-1): letters and digits of any script stay, so CASE literals such as the Greek 'νέο' / 'παλιό' ("new" / "old") keep the two names apart, and aggregates that differ only in an operator get a signature tag (…_sum_1f3a9c0e, S-MEASURE-2). Such names can be long and are quoted in the generated SQL when they are not plain ASCII; an explicit name gives the measure a short one. The explicit name is a label: dedup still keys on the aggregate signature (an undeclared textual twin shares the declared name), the name enters the same collision checks (two explicit names on one signature, or one name on two signatures, still collide loudly), and it applies only to single-aggregate metrics. Because measure names surface in result columns and metadata, ignoring this key silently renames them — emit it under requires so older engines refuse instead (the first silent-impact key since requires shipped).


metrics:
  - name: new_product_revenue
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "SUM(CASE WHEN orders.category = 'νέο' THEN orders.amount END)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.4", "requires": ["measure"], "measure": {"name": "orders_new_product_amount_sum"}}'
measure, like derive, is normally written by the modeling tool rather than by hand.

Larger extensions¶
These change what a metric or a field computes, or how two tables meet, and each is specified on a page of its own. The sections below say what each one is for and where to read on.

D-WINDOW — window metrics¶
The window key on a metric declares period-over-period comparison, rolling and cumulative aggregates, ranks and first/last/nth values over the metric's own aggregate, without raw window SQL in the expression (which the engine rejects as window_in_metric). The time axis comes from D-TIME. Start with the quick start; the rest of Window metrics is the full specification.

D-LOD — levels of detail¶
A field carries lod: {expr: ...}, a metric carries derive: {type: "lod", expr: ...}: group the records one way, then use the result in a query grouped another way — Tableau's FIXED, INCLUDE and EXCLUDE. Understand LOD explains the idea and runs a first model, the commerce walkthrough builds a complete one, and the authoring reference has every rule:

where a filter applies, and how a fixed time key keeps its storage grain
reading a fixed value next to detail rows
why the core expression must match the declaration
include and exclude metrics, and fixed on a metric
aligning two facts on one axis
a value from another dataset's rows
which combinations need separate queries
D-PARAM — query-time parameters¶
The params key turns a window width, an offset count, a rank bucket count or a literal in a filter into a typed parameter the query binds: one moving_avg(n) instead of r7_revenue / r30_revenue / r90_revenue. Parameters fill typed holes only, never SQL text. Parameterized metrics is the specification, including a field's own parameters, a level-of-detail field's base and a derived subtype's thresholds.

D-DERIVE — derived metrics¶
The derive key records how a metric derives from other metrics of the same model: a filter variant of a base metric, a compose of several, or a window over another metric's series. The engine checks that the metric's expression matches the declaration, and exposes the structure to dosi list, lineage and attribution. It is normally written by the modeling tool. See Derived metrics.

D-CONFORM — conformed relationships between facts¶
cardinality: many_to_many on a relationship pairs two fact tables on the dimensions they share — a date, a platform — without joining their rows: each fact aggregates alone and the results merge. Its impact if ignored is silent, so list it in requires. See Cross-fact alignment.

D-FORMAT — time storage format¶
The time key on a time field declares how dates are stored when the column is not a native date — '20260907', 202609, '2026年09月07日' — so the engine can group the column by month or year and still filter it with partition-friendly bare comparisons. Its impact if ignored is silent, so list it in requires. Time storage format covers when to configure it, a quick start, writing filters, the supported formats, grouping and range boundaries, validation and known limits and the SQL it produces.

Validation & engine modes¶
Upstream Ossie validation (dosi validate --osi-basic, see the CLI reference): unaffected — custom_extensions is valid Ossie, so a model carrying Datus extensions still passes it.
Datus self-validation (engine in --osi-datus mode, the default, when it reads a DATUS entry): the payload must be a JSON object; known keys must match their declared type/enum; otherwise invalid_datus_extension. Unknown keys pass (forward-compat) unless named in requires. The envelope version gate (above) runs here, once per payload — including on the SemanticModel carrier, which defines no keys of its own but is still held to the envelope.
Basic mode (--osi-basic): the engine treats DATUS like any unknown vendor — the extension is inert (spec-compliant passthrough) and the compile collects one ignored_vendor_extension warning per carrying element — SemanticModel, Dataset, Field, Relationship, and Metric alike — naming the default that applies instead (D-JOIN → LEFT, D-FILL → no fill beyond S-FAN-6, D-TIME → single-is_time inference only, D-GRAIN → any grain, D-DATASET → SQL-derived attribution only, D-WINDOW → the plain base aggregate, D-FORMAT → the column is assumed to be a native date, which for a string or integer column means a warehouse type error or silently wrong rows, D-LOD → the field is dropped, D-CONFORM → the edge is an ordinary many→one join, so the two facts are joined row to row and duplicate-sensitive aggregates across it fan out). D-LOD is the one that cannot degrade to a value: without the declaration the field's expression is a bare aggregate, which is not a legal row-level attribute, so the field is dropped with an aggregate_field_dropped warning and any metric or query referencing it fails closed with unknown_field / unknown_column. The alternative would be AVG(COUNT(DISTINCT …)) reaching the warehouse. The payload is not parsed, so a malformed payload is also just ignored-with-warning — and no version diagnostic can occur in basic mode at all. The same model file is valid in both modes; only the semantics differ. (The single-is_time primary-time inference is not an extension — it applies in basic mode too, so metric_time still works wherever pure Ossie determines a unique time axis.)
Non-goals / boundaries¶
Datus extensions never redefine an Ossie object's identity or a metric's SQL — only engine behavior at Ossie-undefined seams.
They are not a place to smuggle vendor SQL. Metric expressions stay Ossie expression.dialects; the engine still infers metric semantics from that SQL.
Semantics that belong in Ossie core (granularity, time spine, window/cumulative metrics) may eventually be standardized upstream; if/when adopted there, the corresponding Datus extension (D-GRAIN, D-WINDOW) is retired in favor of the core field.

Skip to content
Dosi
Window metrics



 
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
Extensions
Extensions
Overview
Window metrics
Table of contents
Goals
Quick start
Table of Contents
Carrier and envelope
Enumerations
Granularity
Calculation (offset family)
Function (frame family)
The window payload
General form — offset
Schema
General form — frame
Schema
NULL placement (order.nulls, 1.9)
Whole-partition frames (scope: "partition", 1.9)
Shares of a total
General form — rank
Schema
General form — value
Schema
Shared modifiers — order / partition
Sugar — pop
Schema
Desugaring
Sugar — rolling
Schema
Desugaring
Sugar — cumulative
Schema
Desugaring
Time-axis resolution
Compilation semantics
Offset family — shifted-key self-join
Frame family — window function
Rollup contract
Querying window metrics
Validation
Restrictions
Extensibility model and roadmap
The extension mechanism
Frozen degrees of freedom (and the modifiers that open them)
Family roadmap
Function registry
Boundaries
Version history
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
Window metrics (D-WINDOW)¶
Extension datus-ext/1. Executed-verified on DuckDB and on live warehouses; what a dialect cannot run is listed under Limitations on its connector page.

Version: 1.9 (datus-ext)

Goals¶
Declarative window metrics: express period-over-period comparison, rolling aggregates, and cumulative (running / period-to-date) values on top of an existing Apache Ossie metric definition — without raw window SQL in metric expressions (which the engine rejects, see Boundaries).
100% Ossie-valid documents: the extension rides in a Metric's custom_extensions under the DATUS vendor envelope. A standard-Ossie consumer sees a plain aggregate metric; dosi --osi-basic ignores the extension with a warning.
Four canonical families, three sugars: every window metric normalizes to an offset (calendar-shifted comparison, backward or forward), a frame (ordered re-aggregation — time- or value-ordered, ROWS or RANGE), a rank (SQL:2003 ranking of the metric value), or a value (SQL:2011 first/last/nth navigation). The pop / rolling / cumulative sugars are pure authoring conveniences that desugar to the same internal form.
Recompute semantics: a window metric is a derived value. When the query's dimensions or time grain change, the engine recomputes the base aggregate at the new grain and re-evaluates the window — it never re-aggregates window outputs.
Quick start¶
"How does this month compare to last month?" "What's the 3-month moving average?" "Revenue year-to-date?" — none of these is expressible as one Ossie aggregate, and writing LAG(...) OVER (...) inside a metric expression is rejected outright (window_in_metric): raw window SQL can't be validated, recomputed across groupings, or re-ranged safely.

Declare the derivation on the metric with the window key; the metric's own expression stays the plain base aggregate:

To run this example, copy $DOSI_EXAMPLES/orders/model.yaml to window-model.yaml, append the three metric entries below to its existing metrics list, and seed orders.duckdb with $DOSI_EXAMPLES/orders/seed.sql using the DuckDB CLI. The YAML below is a fragment, not a standalone model.


metrics:
  - name: revenue_mom_growth          # month-over-month growth
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "SUM(orders.amount)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.2", "window": {"type": "pop", "offset": "1 month"}}'

  - name: revenue_3m_avg              # 3-month moving average
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "SUM(orders.amount)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.2", "window": {"type": "rolling", "function": "avg", "periods": 3}}'

  - name: revenue_ytd                 # year-to-date total
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "SUM(orders.amount)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.2", "window": {"type": "cumulative", "function": "sum", "reset": "year"}}'
Query them like any metric — group by the time axis with a grain (orders.order_date:month or metric_time:month); every other group-by dimension partitions the window (MoM within each region, for example):


dosi query --model window-model.yaml --metrics revenue,revenue_mom_growth,revenue_ytd \
  --group-by customers.region --group-by metric_time:month \
  --start-time 2024-01-01 --end-time 2024-04-01 --execute --db orders.duckdb
Period-over-period compiles to a calendar-correct self-join (a month with no prior month reads NULL — never "the previous existing row"), and the engine automatically loads the lookback: a May start fetches April, a YTD start mid-year fetches back to January 1, and the output is trimmed back to what you asked for.

One query can mix all of the above — MoM next to a rolling average next to YTD and QTD — with a single caveat: a never-resetting running total can't share a query with lookback/reset metrics when a start time is set (the widened scan would change what it accumulates; the engine tells you so). Rolling metrics also take "require_full_window": true to return NULL for the first buckets instead of averaging a partial frame.

Two things trip people up:

The time axis must be in the group-by with a grain — forgetting it is a structured error whose retry hint names the exact item to add.
Give offsets a --start-time: without one the first period in your data simply has no predecessor and reads NULL (correctly).
If you find yourself declaring revenue_1m_avg, revenue_3m_avg and revenue_6m_avg, declare the width as a query-time parameter instead: add a params entry and put {"param": "n"} where the number was.


  - name: revenue_moving_avg          # n-month moving average
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.5", "params": [{"name": "n", "type": "int", "default": 3, "min": 1, "max": 12}], "window": {"type": "rolling", "function": "avg", "periods": {"param": "n"}}}'

dosi query --metrics revenue,revenue_moving_avg \
  --group-by metric_time:month --param n=1,3,6
One metric, three columns (revenue_moving_avg__n_1 and friends). A query that binds nothing gets the default, so the metric behaves exactly like the fixed one it replaces. The declaration rules, the domain and the output contract are in Parameterized metrics.

The rest of this page is the full specification: the general offset / frame / rank / value forms the sugars desugar to, the value|delta|percent_change|ratio calculations, reset semantics, and the restrictions.

Table of Contents¶
Carrier and envelope
Enumerations
The window payload
General form — offset
General form — frame
General form — rank
General form — value
Shared modifiers — order / partition
Sugar — pop
Sugar — rolling
Sugar — cumulative
Time-axis resolution
Compilation semantics
Querying window metrics
Validation
Restrictions
Extensibility model and roadmap
Boundaries
Version history
Carrier and envelope¶
The extension is carried on a Metric in custom_extensions, vendor DATUS, as the payload key window. The metric's own expression is the base aggregate that the window derives from — the SQL is written once, and the window layers on it.


- name: revenue_mom_growth
  description: Month-over-month revenue growth rate
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: SUM(orders.amount)
  custom_extensions:
    - vendor_name: DATUS
      data: '{"v": 1, "window": {"type": "pop", "offset": "1 month"}}'
Default (key absent): the metric is its plain base aggregate — no derivation.

Without base, the metric's own expression MUST infer as a single plain aggregate: the window reads a measure column, and a ratio has none.

With base (1.9) that restriction moves to the base, which is where it belongs — the authored expression is then only the basic-mode approximation, and the series comes from the named metric. A base may be a leaf, a filter metric, a ratio, or a compose, whether its measures sit in one dataset or several; the planner projects its arithmetic as one more column of the base CTE (of the facts' merge, over several) and the window reads that. So ranking a rate, or taking its period-over-period, is written by naming the rate:


{"window": {"base": "close_rate", "rank": {"function": "rank"}}}
A compose over two facts takes a period-over-period the same way. An offset needs one time axis, so the two facts' dates are paired through a D-CONFORM many_to_many relationship, and each fact then reads its own date (Restrictions 2). Net revenue's quarter-over-quarter change, with net_revenue the compose revenue - refund_total over orders and refunds:


relationships:
  - name: orders_refunds_conformed
    from: orders
    to: refunds
    from_columns: [order_date]
    to_columns: [refund_date]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.8", "requires": ["cardinality"], "cardinality": "many_to_many"}'
metrics:
  - name: net_revenue_qoq
    expression:
      dialects:
        - dialect: ANSI_SQL
          expression: "COALESCE(SUM(orders.amount), 0) - COALESCE(SUM(refunds.amount), 0)"
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.9", "window": {"base": "net_revenue", "type": "pop", "offset": "1 quarter"}}'
A query groups it by metric_time:quarter, or by either date at a quarter grain, and a time_range lands on both facts.

See Validation for what stays rejected (naming a window metric directly, and nesting past three levels).

Enumerations¶
Granularity¶
Time granularities, shared with D-TIME / D-GRAIN.

Value	Notes
day	
week	Week start follows the execution dialect's DATE_TRUNC('week', …) convention (ISO Monday on DuckDB).
month	
quarter	
year	
Calculation (offset family)¶
How the current value relates to the offset (reference) value. Let cur = current bucket's base value, prev = the value count × granularity earlier.

Value	Result	NULL behavior
value	prev	NULL when no bucket exists at the shifted key
delta	cur - prev	NULL when prev is NULL
percent_change	(cur - prev) / prev, null-safe	NULL when prev is NULL or 0
ratio	cur / prev, null-safe	NULL when prev is NULL or 0
Division is always non-integer (CAST(… AS DOUBLE)), zero denominators yield NULL (NULLIF(prev, 0)). A NULL window output is semantically load-bearing ("no comparable prior period") — the engine's D-FILL fill_nulls_with is not applied to window outputs.

Function (frame family)¶
Re-aggregation applied over the ordered frame of per-bucket base values. This is registry level W1 — the portable core, not the ceiling; see the function registry for the SQL:2003-aligned levels (statistical aggregates, ranking, value/navigation) and what each unlocks.

Value	Level	Notes
sum	W1	
avg	W1	
min	W1	
max	W1	
count	W1	Counts frame rows (COUNT(*) over the frame; under units: range, tied rows are peers and share the count). The base value is not an argument.
stddev_pop / stddev_samp	W2 (1.3)	Explicit _pop/_samp forms only — bare STDDEV/VARIANCE mean different things across engines and are rejected.
var_pop / var_samp	W2 (1.3)	
covar_pop / covar_samp / corr	W3 (1.3)	Two-argument: require second (general frame form only). Rendered CORR(second, primary). Not available as window functions on Redshift, MySQL or TiDB — structured dialect_unsupported_window_function.
Metric-series semantics: the frame aggregates the per-bucket metric values, not the underlying source rows. A 3-month rolling sum of a distinct count is the sum of three monthly distinct counts — an entity active in two months counts twice. Frame-level re-deduplication (source_rows evaluation) is out of scope for v1.

The window payload¶
window is a JSON object. Its shape is selected by the optional type discriminator:

type	Meaning	Normalizes to
(absent)	General form — exactly one of offset / frame	itself
pop	Period-over-period sugar	offset family
rolling	Trailing-N-buckets sugar	frame family
cumulative	Running / period-to-date sugar	frame family
Top-level schema (all shapes):

Field	Type	Required	Description
type	string	No	pop | rolling | cumulative; absent = general form
base	string	No	The metric whose series the window computes over, instead of this metric's own expression (1.9): a leaf, filter, ratio or compose metric, or a compose over another window. Any shape. See Carrier and envelope
offset	object/string	shape-dependent	Offset spec (general form + pop)
calculation	string	shape-dependent	Calculation (general offset form + pop)
frame	object	shape-dependent	Frame spec (general form only)
rank	object	shape-dependent	Rank spec (general form only, 1.3)
value	object	shape-dependent	Value-navigation spec (general form only, 1.3)
function	string	shape-dependent	Function (rolling / cumulative)
periods	integer	shape-dependent	Frame width in buckets (rolling)
reset	string	shape-dependent	Accumulation reset boundary (cumulative)
The general form sets exactly one of offset / frame / rank / value.

Forward compatibility is fail-closed by construction:

Unknown keys at the top level of window are ignored — a future family key (say rank) is safe to ignore on a v1 engine, because the document then has neither offset nor frame and fails the exactly-one-family rule with a structured error instead of computing wrong values. An unknown type value is rejected outright.
Unknown keys inside offset / frame are rejected — a future modifier key (say partition or order) changes the semantics of an existing family, and silently ignoring it would return wrong numbers. Rejection forces "this engine is too old for this document" to surface as invalid_datus_extension.
Keys that belong to a different shape than the one selected are likewise rejected — e.g. periods with type: pop, or a top-level function in the general form.

General form — offset¶
Compare each time bucket with the bucket count × granularity earlier.


{"window": {"offset": {"count": 1, "granularity": "month"},
            "calculation": "percent_change"}}
Schema¶
Field	Type	Required	Description
offset.count	integer ≥ 1 | {"param": …}	Yes	Number of granularity units to shift. Parameterizable in the object form (D-PARAM slot offset.count); the string sugar takes no parameter.
offset.granularity	string	Yes	Granularity of the shift
offset.direction	string	No	"back" (default — prior period, LAG) | "forward" (next period, LEAD; 1.3). Object form only: the "1 month" string sugar is always back. A forward offset widens the time range's end bound instead of its start, and the output is trimmed back likewise.
calculation	string	Yes	Calculation; required in the general form (defaults belong to sugars)
The shift is calendar-correct under gaps: the reference value is the bucket at exactly the shifted calendar key, or NULL if that bucket has no data — never "the previous existing row". granularity may differ from the query grain as long as the shift lands on another bucket of it: YoY over monthly buckets (DATE_TRUNC('month', t) - INTERVAL 1 YEAR is the prior-year month), or 3 month over quarterly ones. A shift that does not land — 1 month over quarter or week buckets, 1 year over week buckets (a year ago is rarely a Monday) — would look up a bucket the query never builds and read NULL on every row, so it is refused with window_offset_misaligned, naming the grains the shift does land on. Day buckets take any shift.

General form — frame¶
Re-aggregate the base values over a frame: either the ordered trailing frame ending at the current bucket (preceding), or the whole partition (scope, 1.9). Exactly one of the two.


{"window": {"frame": {"function": "sum", "preceding": "unbounded", "reset": "year"}}}
{"window": {"frame": {"function": "count", "scope": "partition",
            "partition": {"mode": "query_dimensions_except",
                          "exclude": ["kpi_cell.market"]}}}}
Schema¶
Field	Type	Required	Description
frame.function	string	Yes	Function
frame.preceding	integer ≥ 0 | "unbounded" | {"param": …}	Exactly one of preceding / scope	Frame start: n = n PRECEDING; "unbounded" = UNBOUNDED PRECEDING. The frame always ends at CURRENT ROW. Parameterizable (D-PARAM slot frame.preceding).
frame.scope	string	Exactly one of preceding / scope	(1.9) "partition" — the frame is the whole partition, order-independent: COUNT(*) OVER (PARTITION BY …). Mutually exclusive with preceding, reset, require_full_window, units and order, each of which only describes an ordered frame with an end bound; writing one is rejected rather than ignored.
frame.reset	string	No	week | month | quarter | year — restart accumulation at each boundary of this granularity. Only meaningful with "unbounded"; also accepted with numeric preceding (the frame is then clipped at the boundary). Absent = never reset.
frame.require_full_window	boolean	No (default false)	NULL out buckets whose frame holds fewer than preceding + 1 rows, instead of aggregating a partial head frame. Finite preceding only — with "unbounded" and true the payload is rejected (an unbounded frame is never partial). Rejected under units: "range" (frame-row counting is undefined over tie peers). In-family 1.2 addition: an engine predating it rejects the key (invalid_datus_extension), never drops it.
frame.order	object	No	Shared modifier (1.3). Default {"by": "time", "direction": "asc"} — the v1 behavior. order.by: "value" orders by the metric's own per-bucket value and forbids reset (a time partition without time order is incoherent). order.nulls (1.9) pins where NULLs sort.
frame.partition	object	No	Shared modifier (1.3). Default mode query_dimensions — the v1 behavior.
frame.units	string	No	"rows" (default) | "range" (1.3). RANGE requires preceding: "unbounded" and order.by: "value": a value-ordered cumulative frame where tied values are peers and aggregate together (a time-ordered series has at most one row per bucket, where RANGE equals ROWS).
frame.second	string	Iff two-argument	(1.3) The second input series for covar_pop / covar_samp / corr, named as a plain single-aggregate metric of the same model (no window, not ratio/expression). Resolved at compile time; the base stage computes both series. General frame form only.
The frame is row-based over the aggregated series (ROWS BETWEEN … AND CURRENT ROW), where the series has at most one row per partition × time bucket by construction. Missing buckets are not densified in v1: a "3 preceding" frame spans the three previous existing buckets. Offsets do not have this caveat.

reset: day is rejected: day is the finest grain, so a day reset could never be coarser than any query grain.

NULL placement (order.nulls, 1.9)¶
order: {"by": …, "direction": …, "nulls": "first" | "last"} pins where NULLs sort within the ordering. Left unset, no clause is emitted and each warehouse applies its own default — and those disagree: DuckDB and PostgreSQL sort NULLs last ascending, MySQL-family engines and StarRocks sort them first.

That difference is invisible until the ordering key can be NULL, which a ratio base makes routine — a zero denominator yields NULL, and in an ascending rank a NULL sorted first takes the best position. The same model then 

Skip to content
Dosi
Understand LOD — calculate at one grain, display at another



 
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
Extensions
Extensions
Overview
Window metrics
LOD
LOD
Understand and start using LOD
Table of contents
What is LOD?
When do you need LOD?
Choose among four uses
Check the limits before designing a query
Run a minimal model
Filters decide which records enter the calculation
Turn a business question into a model
Next steps
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
Understand LOD — calculate at one grain, display at another¶
LOD handles calculations that first group records one way, then use those results in a query grouped another way. It lets you define an intermediate calculation separately from the final presentation.

This page explains the concepts and choices, summarizes the key limits, and runs a first query with a complete model before explaining filter scope. When you write a model, look up individual rules in the LOD authoring reference. A complete runnable case is the commerce walkthrough.

What is LOD?¶
LOD stands for Level of Detail. For a metric calculation, it describes what you group by and how detailed that grouping is. This is the calculation's grain.

Consider a small question: how much has each paying user acquired through ads spent on average? In the walkthrough's data through May 31, 2030, that channel has two buyers:

Buyer	Orders	Lifetime spending
U01	4	CNY 500
U04	1	CNY 300
The answer is (500 + 300) / 2 = 400. First total spending per buyer, then average those two totals. U01 must not contribute more times to the average simply because they placed more orders.

There are two groupings here: calculate per buyer, then display per acquisition channel. The first is the calculation grain; the second is the query grain. Grain: what one row means places LOD beside the other grains and cross-fact alignment. Grouping fields are also called dimensions. The query's group_by controls the display; an LOD declaration controls the intermediate calculation that needs its own grouping.

In Dosi, you declare LOD in the semantic model. Queries still select metrics and grouping fields; Dosi generates SQL for the calculation stages. LOD is a DATUS extension to Apache Ossie (formerly OSI). The database does not need a function named LOD.

When do you need LOD?¶
Ordinary aggregation usually suffices when you only need a sum, count or average at the query's grouping. Consider LOD for these requirements:

Classify or filter by an aggregate. Count purchases per buyer before classifying buyers as once-only or repeat buyers.
Aggregate intermediate results. Total spending per buyer before averaging buyers within each acquisition channel.
Compare results at different groupings or scopes. Display daily app sales alongside daily site sales or the app's monthly total.
Before choosing an expression, state which records enter the calculation and how many times each entity contributes. Returning a number is not sufficient to establish that it answers the business question.

Choose among four uses¶
First decide whether you need a reusable field or a metric returned by the query. A field can be classified, filtered, projected into detail rows or aggregated again. A metric is a statistical result selected by a query.

What you need	Use	Declaration in the model (commerce walkthrough)
An aggregate per entity that you can use as a field	fixed field	fixed(paid_total, orders.user_id): each buyer's lifetime payments
A finer calculation within each display group, followed by an average, maximum or other aggregate	include metric	AVG(include(order_count, orders.user_id)): count orders per buyer, then average
A coarser subtotal within the selected records	exclude metric	exclude(paid_total, orders.paid_date): remove date to get the channel's period total
A baseline grouped by specified fields, unaffected by ordinary display filters	fixed metric	fixed(paid_total, orders.paid_date): daily whole-site sales
The first argument of a declaration is an existing metric, called the base metric: here paid_total is the payment total and order_count is the order count. The remaining arguments are grouping fields.

fixed appears twice because the result has different uses. Buyer lifetime spending needs to be a field that another metric averages. Daily site sales can be a metric displayed next to daily app sales. Decide how you will use the result before choosing a fixed field or a fixed metric.

A fixed field hosted on a dimension, such as each customer's order count on customers, can also group or filter any metric whose dataset reaches that dimension along a many-to-one path: order count or margin by big and small customers. The value is computed from the dimension's own rows under the query's context and time range, then read beside each fact row. A dimension row with no rows on the aggregated side reads 0 for a count and NULL otherwise. A fact row with no dimension row, because its key matches none or is NULL, reads NULL, as it does for the dimension's other fields.

include means calculate at a finer grouping first: to show average purchase frequency by channel, count orders per buyer within each channel, then average. exclude means remove some grouping: when displaying channel and date, remove date to obtain the channel total for the selected period.

These definitions can share one model. They cannot necessarily share one query; see the combination limits.

Check the limits before designing a query¶
LOD definitions can coexist in one model even when a query cannot combine them. Keep these limits in mind before choosing report columns:

No nested LOD. A base metric and its grouping keys cannot depend on another LOD value, even indirectly.
Include metrics are computed apart from their own fact's ordinary metrics. In one query Dosi computes them as separate parts aligned on the grouping, and does the same for include metrics with different base datasets or dimension lists. Exclude/fixed metrics and another fact's metrics share the include's own pass. On a dialect that shape is not enabled for, query them separately.
Re-aggregating fixed fields changes the counting unit. These metrics are computed apart from raw-row metrics, and from metrics re-aggregating another fixed-key family, as separate parts. Using a fixed field only for grouping or filtering still permits ordinary metrics.
A keyed fixed metric requires its exact keys in group_by. A time grain such as :day is not the same key. To aggregate fixed results to a coarser grouping, use a fixed field and an ordinary aggregate.
Window metrics and time_ranges are computed apart from LOD. A window metric beside a LOD metric is a separate part, and a LOD metric under time_ranges is computed once per period. A window metric can be grouped or filtered by a LOD field unless its stage reads rows before the time range (an offset's lookback or a reset under a time_range.start): the fixed value would be computed over those rows too. Exclude/fixed metrics cannot share a query with LOD fields the query groups or filters by. Metrics of another fact table can share a query with any LOD metric: each fact aggregates on its own and the results align on the grouping fields.
Use datus mode. Basic mode does not execute an equivalent LOD calculation. LOD also does not automatically fill missing dates or entities.
The complete limits and alternatives distinguish unsupported combinations from supported ones.

Run a minimal model¶
Start with two buyers and three orders. Save this SQL as lod-minimal-seed.sql:


CREATE TABLE orders (
  order_id VARCHAR PRIMARY KEY,
  user_id VARCHAR,
  paid_amount DECIMAL(12,2)
);
INSERT INTO orders VALUES
  ('O01', 'U01', 100),
  ('O02', 'U01', 300),
  ('O03', 'U02', 200);
Save the following complete model as lod-minimal.yaml. It needs no other model files. paid_total sums payments; buyer_lifetime_paid fixes that sum by buyer; mean_lifetime_paid averages the resulting buyer values.


version: 0.2.0.dev0
name: lod_minimal
datasets:
- name: orders
  source: orders
  primary_key:
  - order_id
  fields:
  - name: order_id
    expression:
      dialects:
      - dialect: ANSI_SQL
        expression: order_id
    datatype: String
    custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","is_dimension":false}'
  - name: user_id
    expression:
      dialects:
      - dialect: ANSI_SQL
        expression: user_id
    datatype: String
    custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","is_dimension":false}'
  - name: paid_amount
    expression:
      dialects:
      - dialect: ANSI_SQL
        expression: paid_amount
    datatype: Decimal
    custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","is_dimension":false}'
  - name: buyer_lifetime_paid
    expression:
      dialects:
      - dialect: ANSI_SQL
        expression: SUM(orders.paid_amount)
    datatype: Decimal
    custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","is_dimension":false,"lod":{"expr":"fixed(paid_total, orders.user_id)"}}'
metrics:
- name: paid_total
  expression:
    dialects:
    - dialect: ANSI_SQL
      expression: SUM(orders.paid_amount)
- name: mean_lifetime_paid
  expression:
    dialects:
    - dialect: ANSI_SQL
      expression: AVG(orders.buyer_lifetime_paid)
With Dosi and the DuckDB CLI installed, run these commands in the directory containing both files. Use a new database file:


duckdb lod-minimal.duckdb < lod-minimal-seed.sql
dosi validate --model lod-minimal.yaml --strict
dosi query --model lod-minimal.yaml --metrics mean_lifetime_paid \
  --execute --db lod-minimal.duckdb --format json
The returned mean_lifetime_paid is 300: U01 contributes 400 once and U02 contributes 200 once, so (400 + 200) / 2 = 300. Averaging the three raw order amounts would return 200 and answer a different question. Even though the fixed value is available on each order, its re-aggregation counts each buyer once.

The fixed declaration belongs to the field's custom_extensions; its first argument names the existing base metric. The field's ordinary expression repeats that base metric's complete SQL. data is a JSON-encoded string.

For include, exclude and fixed metrics, put the declaration under derive: {type: "lod", expr: ...} in the metric's DATUS JSON. Include's core SQL wraps the base aggregate in the declared outer aggregate; exclude/fixed metrics use a SUM wrapper, even if their base is COUNT or MAX. For example, fixed(paid_total, orders.paid_date) uses SUM(SUM(orders.paid_amount)). Over a compose, ratio or expression base the wrapper holds the base's whole expression, as in SUM(COALESCE(SUM(orders.amount), 0) - COALESCE(SUM(refunds.amount), 0)) for exclude(net_revenue). This nested SQL represents calculation stages for Dosi to validate and lower; it is not SQL to execute directly against the database. See the expression rules and the commerce model for complete examples of these forms.

Filters decide which records enter the calculation¶
Grouping determines how records are aggregated; filters determine which records participate. Fixed means a fixed grouping, not unconditional access to all history.

A query has two kinds of conditions:

Context (context_filter, CLI --context-filter) and the time range (time_range) restrict the calculation input, including fixed calculations. Each calculation reads them on the dataset it aggregates: the time range lands on that dataset's own time field, and a dataset without one is read whole, with a warning. A context_filter condition applies where that dataset can read it through a declared relationship or key pairing; one it cannot read is skipped and reported in the result's warnings. Define a period that differs from the query's in its base metric (cross-table rules).
Ordinary filters (where_sql, CLI --where) select which rows are displayed. They narrow ordinary metrics and include/exclude inputs, but not the input of fixed.
In the app sales report, April 2, 2030 has CNY 160 of app sales and CNY 500 of site sales. App sales for all of April total CNY 570. One report row needs these three scopes:

Result	Records it needs	Use
Daily app sales: 160	App orders on April 2	Ordinary payment metric displayed by date and channel
Daily site sales: 500	All channels' orders on April 2	Metric fixed by date, preserving the whole-site scope
App monthly total: 570	App orders throughout April	Exclude date while retaining channel
The query supplies April as its time range and app as an ordinary display filter. The time range scopes all three calculations. The app filter scopes the daily numerator and channel subtotal, but not the fixed daily site baseline. App's share of that day's site sales is therefore 160 / 500 = 0.32. The walkthrough's question 4 runs this query and shows its results.

Choose a filter location by its purpose:

Purpose	Location	Effect on a fixed calculation
Restrict the records or time used for calculation	context_filter, time_range	Restricts the input; the time range lands on each dataset's own time field, a condition only where that dataset can read it
Select ordinary field values to display	where_sql	Does not narrow the fixed input; does narrow include/exclude inputs
Select by a computed fixed value or selected metric	where_sql	Runs after the corresponding value has been computed
The last two rows both use where_sql and need no extra marking. Dosi decides when a condition runs from the fields it references: a condition on ordinary fields selects display records before calculation; a condition on a fixed field or a selected metric runs after those values exist. For example, orders.channel = 'app' is the first kind and orders.buyer_order_count >= 2 is the second.

Restrictions in a dataset's source always apply. Define independent periods through parameters. See detailed filter rules for mixed AND/OR predicates, related dimensions and time encodings.

Turn a business question into a model¶
Write down these five decisions in ordinary language before choosing syntax:

What does one source row represent? Declare actual primary keys. A repeating identifier must not be declared unique merely to pass validation.
Which entities belong in the result? Distinguish no activity from being outside the population. Decide whether missing records mean zero or NULL.
What do you calculate first, and what do you display? Prefer a fixed field when the intermediate value will be classified, filtered or aggregated.
How often should each entity contribute? Per-person, per-record and per-day averages have different denominators. Define each side of a ratio.
Which conditions affect calculation, and which select results? Define separate observation windows and explicit key mappings for other datasets.
Choose the simplest expression that preserves these decisions. A total divided by an entity count equals an average over entities only when populations and weights agree. Check the model against independent raw-data calculations, then change the data and query parameters and check again.

Next steps¶
Run the commerce walkthrough: one model, five questions, with expected results to compare against.
While writing a model, use the LOD authoring reference: base-metric requirements, declaration format, parameterized windows, cross-dataset calculations, which query combinations must be split, and validation.
 Back to top
Next
Commerce modeling walkthrough
Copyright © Datus · Licensed under the Elastic License 2.0
Skip to content
Dosi
Five commerce questions with one LOD model



 
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
Extensions
Extensions
Overview
Window metrics
LOD
LOD
Understand and start using LOD
Commerce modeling walkthrough
Table of contents
Run the example
Data and business definitions
How the model separates calculation grains
1. How much do repeat buyers contribute?
2. Which acquisition channels bring higher-spending buyers?
3. How often do buyers purchase within a channel?
4. What does the daily app sales report show?
5. Who bought in April but did not return in May?
What this example proves
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
Five commerce questions with one LOD model¶
Run the example¶
Install Dosi and the DuckDB CLI. Use the model and seed shipped in the release; no Python or regression script is required.


export DOSI_EXAMPLES=~/.local/share/dosi/examples
cp -R "$DOSI_EXAMPLES/lod-commerce" ./lod-commerce
cd lod-commerce
duckdb lod_commerce.duckdb < seed.sql
dosi validate --model model.yaml --strict
Use a new database file. For manual archives, first point DOSI_EXAMPLES at that archive. The seed comes from testkit data; queries.json, expected.json, independent SQL and verification scripts are not shipped.

Data and business definitions¶
The current seed contains 10 customers, 41 orders and 9 refund applications. Model dataset users maps to physical table customers; paid_total includes amounts only for completed orders. Data includes nulls, unmatched relationships and non-completed orders, rather than the older 2030 demonstration dataset. The queries use 2026 intervals, inclusive at the start and exclusive at the end.

How the model separates calculation grains¶
Read the bundled model.yaml instead of copying a second potentially stale definition. Fixed fields calculate by customer; include aggregates at a finer grain first; exclude removes a grouping field; independent parameters control separate time windows. The forms are exercised in separate queries below.

1. How much do repeat buyers contribute?¶

dosi query --model model.yaml \
  --metrics buyers,paid_total \
  --group-by orders.buyer_group \
  --context-filter "orders.paid_date < DATE '2026-06-01'" \
  --order buyer_group \
  --execute --db lod_commerce.duckdb --format json
buyer_group	buyers	paid_total
once	1	47.49
repeat	6	19873.0
2. Which acquisition channels bring higher-spending buyers?¶

dosi query --model model.yaml \
  --metrics mean_lifetime_paid \
  --group-by users.acquisition_channel \
  --context-filter "orders.paid_date < DATE '2026-06-01'" \
  --order acquisition_channel \
  --execute --db lod_commerce.duckdb --format json
acquisition_channel	mean_lifetime_paid
ads	2191.5
organic	3102.398
NULL	25.5
3. How often do buyers purchase within a channel?¶

dosi query --model model.yaml \
  --metrics mean_orders,max_orders \
  --group-by orders.channel \
  --start-time 2026-04-01 \
  --end-time 2026-05-01 \
  --time-dimension orders.paid_date \
  --execute --db lod_commerce.duckdb --format json
channel	mean_orders	max_orders
app	1.0	1
4. What does the daily app sales report show?¶

dosi query --model model.yaml \
  --metrics paid_total,site_amount,daily_site_share,app_month_amount,app_month_share \
  --group-by orders.channel,orders.paid_date \
  --where "orders.channel = 'app'" \
  --start-time 2026-04-01 \
  --end-time 2026-05-01 \
  --time-dimension orders.paid_date \
  --order paid_date \
  --execute --db lod_commerce.duckdb --format json
channel	paid_date	paid_total	site_amount	daily_site_share	app_month_amount	app_month_share
app	2026-04-05	842.0	842.0	1.0	1707.0	0.4932630345635618
app	2026-04-15	865.0	865.0	1.0	1707.0	0.5067369654364382
--where selects displayed app rows; it is not a substitute for a context filter that changes every LOD denominator. Both dates have app-month amount 1707.

5. Who bought in April but did not return in May?¶

dosi select --model model.yaml \
  --from users \
  --fields users.user_id,users.region,users.reference_paid,users.observation_orders,users.window_refunded \
  --where "users.reference_orders > 0 AND users.observation_orders = 0" \
  --param reference_start=20260401 \
  --param reference_end=20260501 \
  --param observation_start=20260501 \
  --param observation_end=20260601 \
  --param refund_start=20260401 \
  --param refund_end=20260501 \
  --param refund_status=SUCCEEDED \
  --limit 10000 \
  --execute --db lod_commerce.duckdb --format json
user_id	region	reference_paid	observation_orders	window_refunded
1	east	842.0	0	50.0
3	east	865.0	0	0.0
The three parameter windows are independent. Do not add a global time range that truncates the other windows. NULL values remain NULL.

What this example proves¶
These commands were executed using the exported seed and model. Full regression cases and expected answers remain in testkit and are not installation prerequisites. See LOD and detail queries for combination limits.

 Back to top
Next
Authoring reference
Copyright © Datus · Licensed under the Elastic License 2.0

Skip to content
Dosi
LOD authoring reference



 
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
Extensions
Extensions
Overview
Window metrics
LOD
LOD
Understand and start using LOD
Commerce modeling walkthrough
Authoring reference
Table of contents
What can be a base metric?
Fixed fields: choose the grouping and reuse the value
LOD metrics: return the calculation as a metric
Where do declarations belong?
Why are both SQL and an LOD declaration required?
Different periods: define independent parameterized windows
Across tables: decide where the result belongs
Classify two facts independently, then align them
Detailed filter rules
Query the LOD definitions
Limits: which calculations need separate queries?
Validation, versions and compatibility
The document shape Dosi accepts
Basic mode does not perform the same LOD calculation
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
LOD authoring reference¶
This page organizes the rules by the questions that come up while writing a model or tracking down an error. If LOD is new to you, read Understand LOD first; it covers the concepts, how to choose among the four uses, and the basic filter distinction. Field and metric names in the examples come from the commerce walkthrough.

Terms used throughout this page:

Base metric: the existing metric named by an LOD declaration's first argument, such as paid_total in fixed(paid_total, orders.user_id).
Fixed field: a field declaring fixed; the result can be grouped, filtered and aggregated again.
LOD metric: a metric declaring include, exclude or fixed; the result is returned directly by the query.
Context: a query's context_filter and time_range, which restrict the source records used for calculation. Across datasets, only conditions readable through declared relationships or key pairings propagate; see across tables. In contrast, an ordinary filter (where_sql) selects display rows and does not narrow fixed inputs.
Host: the dataset on which a fixed field is defined. It decides which rows the result has; see across tables.
What can be a base metric?¶
A base must resolve to one aggregate over one dataset, or a filter derivation of that aggregate. Supported functions are SUM, COUNT, COUNT with DISTINCT, AVG, MIN and MAX. A bare COUNT(*) in a multiple-dataset model needs an explicit home through D-DATASET. Counting a declared non-null identifier also makes ownership explicit.

A LOD metric's base may also be built from several such aggregates: a compose, a ratio or an expression metric. The LOD evaluates each aggregate at its grain and applies the base's own arithmetic, fill included, so exclude(net_revenue) over revenue − refunds is the whole query's revenue minus its refunds, even over two facts; dividing by it gives a share of the total. The authored expression nests the base's whole expression inside the outer aggregate, for example SUM(COALESCE(SUM(orders.amount), 0) - COALESCE(SUM(refunds.amount), 0)). Each aggregate is bucketed on its member's time axis, as in the compose itself, and a fixed level of detail applies the query's time range on those axes; a compose over the level of detail keeps them. An include over aggregates of one dataset on two axes is refused, since it rolls each dataset up once.

An include over a base spanning several datasets computes each dataset's aggregates on its own rows, per display row and declared dimension, then merges them on those groupings before applying the base's arithmetic. A grouping that any of the datasets has is one inner row. An aggregate whose dataset has no rows there is NULL, as a missing member of a cross-fact compose is at the query grain, unless the base's fill replaces it. The outer aggregate skips NULLs. Without a fill, AVG(include(net_revenue, orders.customer_id)) leaves out a customer with orders but no refunds; with fill: 0 that customer counts as revenue − 0. Each declared dimension must be every dataset's own: a field of one of them, paired through D-CONFORM with a field of each other one. Over several datasets, exclude() and fixed() take no grouping fields and compute over the whole query; for a keyed one, put the LOD on each member and compose the results.

A fixed field's base is always a single aggregate. A window result, another LOD metric, or a compose with either as a member cannot be a base. Neither the base nor its grouping fields may depend on a LOD field, even indirectly through an ordinary calculated field. Changing declaration order does not permit nesting. An include base must not share its name with a field on its own dataset.

Fixed fields: choose the grouping and reuse the value¶
A fixed field uses fixed. Its first argument is the base metric name; subsequent arguments are fixed grouping fields, separated by commas. Those fields belong to the dataset containing the fixed field. Unqualified names resolve there too, but explicit dataset names make ownership clear. Several fields form one combined grouping, not separate calculations per field.

The walkthrough's fixed(paid_total, orders.user_id) totals payments per user. A same-dataset calculation may omit every grouping field to compute one value for its whole calculation population. A cross-dataset calculation may not. A field cannot use include/exclude or wrap its fixed declaration in an aggregate.

Queries can read, group by, filter on or sort the field; ordinary calculated fields can read it too. A calculated field that reads a fixed field can in turn be named by another field of the same dataset: premier written as customer_margin >= 3000 AND customer_orders >= 6 reads the calculated customer_margin field, not a stored column of that name, because a value that reads a LOD value is never stored. Any other name in a field expression is still a stored column, so a field named after the column it rescales (amount defined as amount * 0.01) keeps meaning the column wherever a sibling names it. Two such calculated fields that name each other are rejected with field_reference_cycle. When an ordinary metric aggregates a fixed field again, each fixed group contributes once within each display group. The ads-channel buyer mean in Understand LOD therefore averages two buyers, not five repeated order-level values. An entity appearing in several display groups contributes once in each one; those groups' entity counts need not be additive.

What the aggregate reads	What contributes once
A stored column or raw-row count	Each source record
A fixed field	Each distinct fixed group within the display group
A calculated field reading only LOD values and constants	The fixed groups of those LOD values
A calculated field also reading a stored column	Each source record
A filter metric whose condition reads a LOD value	What its base aggregates; a filtered row count still counts source records
Using a fixed field only for grouping does not change raw-row metrics' weights. Fields with different fixed groupings may be displayed or filtered together, but a query may reaggregate only values with the same fixed grouping fields. It cannot also request a metric weighted by source records. This holds even when two metrics spell one expression: a filter metric over a row count counts source records, while the same CASE WHEN written directly as a metric reads only LOD values and counts fixed groups.

A NULL key matches no fixed value: keys compare with = (S-FAN-5). A same-dataset calculation with no filter or context of its own is joined to its rows with an inner join, so rows whose key is NULL leave a query that reads it; otherwise they stay and read the value as NULL. Distinct counting excludes NULL as well. Matching a combined grouping preserves every component.

LOD metrics: return the calculation as a metric¶
A metric declaration also takes the base metric name first, followed by dimension fields. These fields must belong to the base metric's dataset and must be qualified with their dataset names.

Type	Declaration requirement	Grouping and calculation
include	At least one dimension; exactly one outer aggregate	Add the declared dimensions to the display grouping, compute the base, then apply the outer aggregate
exclude	Dimensions optional; no outer aggregate in the declaration	Remove the declared dimensions and recompute the base; omitting dimensions removes all display grouping
Metric fixed	Grouping fields optional; no outer aggregate in the declaration	Compute at exactly those fields; omitting them produces an overall baseline
The include outer function may be SUM, COUNT, AVG, MIN or MAX; an outer DISTINCT is not accepted. The walkthrough's AVG(include(order_count, orders.user_id)) computes buyer order counts, then averages them. Without group_by, include still computes the declared finer grouping before aggregating to an overall result. A display time grain applies inside that finer calculation. Ordinary filters and context both scope its input.

Exclude also honors ordinary filters and context. It recomputes at the coarser grouping instead of adding displayed values, so distinct counts must be recalculated there too. Every field a keyed exclude names must be in the query's group_by, at any time grain: excluding a field the query does not group by would remove nothing and leave the base, so the query is refused with lod_dimension_not_grouped. A keyless exclude needs no grouping; without group_by, it is the base over selected records.

Metric fixed honors context and ignores ordinary display filters. Every fixed field must appear unchanged in the query's group_by, without a time grain, or the query is refused with lod_dimension_not_grouped. The metric catalog lists the fields a keyed exclude or fixed names as required_dimensions, and the refusal lists the ones missing as missing_dimensions. A keyless overall baseline also works without grouping. To aggregate fixed results further, use a fixed field and an ordinary aggregate; wrapping a fixed metric declaration in another aggregate is unsupported.

Ordinary, exclude and fixed metrics on one fact may share a query, with compose for ratios or other arithmetic. Multiple include metrics may share a query when their base datasets and ordered dimension list agree; their bases and outer functions may differ. Beside ordinary metrics of a dataset its base reads, an include is computed as a separate part; it cannot share a query with LOD fields the query groups or filters by.

Where do declarations belong?¶
A field or metric retains its normal expression.dialects list with an ANSI_SQL expression. Metric SQL references declared semantic fields using dataset-qualified names, not undeclared physical columns.

Put the extension in the field or metric's custom_extensions, with vendor_name: DATUS. data is a JSON-encoded string. Its v member is the extension version string; 1.11 supports this page's rules. Optional requires lists capability keys: lod for fixed fields and derive for derived metrics.

Definition	Location inside the JSON object
Fixed field	lod.expr contains the fixed declaration
LOD metric	derive.type is lod; derive.expr contains include, exclude or fixed
Filtered base metric	derive.type is filter; derive.base names the base and derive.where contains the predicate
Arithmetic over metrics	derive.type is compose; derive.expr uses metric names in an arithmetic expression
This is the walkthrough's buyer lifetime-spending field, placed in the orders dataset's fields list. The full model also defines its paid_total base.


- name: buyer_lifetime_paid
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: SUM(orders.paid_amount)
  datatype: Decimal
  custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","is_dimension":false,"lod":{"expr":"fixed(paid_total, orders.user_id)"}}'
LOD metrics go in the model's metrics list. This is the average number of orders per buyer within each channel, based on order_count (COUNT(DISTINCT orders.order_id)):


- name: mean_orders
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: AVG(COUNT(DISTINCT orders.order_id))
  custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","derive":{"type":"lod","expr":"AVG(include(order_count, orders.user_id))"}}'
Why are both SQL and an LOD declaration required?¶
expression describes what is calculated; the DATUS declaration describes its grouping and stages. They must agree. Core SQL must be fully expanded, not a metric name, a constant placeholder or a fixed/include/exclude declaration.

Definition	Required core SQL
Fixed field	The base metric's complete aggregate SQL
Include metric	The declared outer function around the base metric's aggregate SQL
Exclude or fixed metric	SUM around the base metric's aggregate SQL, even when the base uses a different function
Ordinary reaggregation of a fixed field	An ordinary aggregate over the field reference
Compose metric	Each metric name replaced by that metric's complete core SQL
The lifetime-spending field above has core SQL SUM(orders.paid_amount). The ordinary buyer-mean metric uses AVG(orders.buyer_lifetime_paid); the include metric in the previous section is AVG(COUNT(DISTINCT orders.order_id)).

Exclude and fixed metrics are the most confusing case. This is the walkthrough's daily site sales:


- name: site_amount
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: SUM(SUM(orders.paid_amount))
  custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","derive":{"type":"lod","expr":"fixed(paid_total, orders.paid_date)"}}'
The inner SUM(orders.paid_amount) is the SQL of the base metric paid_total. The outer SUM is a fixed wrapper meaning "take the base metric's result once more at another grain". It is SUM even when the base uses COUNT or MAX: a fixed metric based on order_count is written SUM(COUNT(DISTINCT orders.order_id)). The outer SUM exists only so Dosi can check that the SQL agrees with the declaration; it does not add baseline values together. Nested aggregate SQL cannot run directly as ordinary SQL. Dosi uses the declaration to split it into valid stages. A mismatch returns derive_expression_mismatch.

A filtered base's core SQL expands its predicate with parameter defaults; it does not retain parameter markers. A filtered row-count COUNT uses a conditional expression returning 1 for a matching record and NULL otherwise. Do not replace this 1 with an identifier merely because the SQL is equivalent. For a sum or distinct count of values, return the aggregated value on a match and NULL otherwise, then apply the aggregate. A fixed field repeats that filtered aggregate SQL and references the filtered metric by name in lod.expr.

Metric names belong in derive.expr, not in place of core SQL. Keep casts and zero-denominator handling consistent between compose declarations and their fully expanded core expressions.

Different periods: define independent parameterized windows¶
When a fixed field needs its own period, define a filtered base metric with parameters for that period, then reference it from the field. Put params beside derive in the base metric's DATUS JSON, not on the fixed field.

These are the walkthrough's two definitions for reference-period payments. First the parameterized base metric, in the metrics list:


- name: reference_paid_amount
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: SUM(CASE WHEN orders.paid_date_key >= 0 AND orders.paid_date_key < 99991231 THEN orders.paid_amount END)
  custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11",
        "derive":{"type":"filter","base":"paid_total",
          "where":"orders.paid_date_key >= :reference_start AND orders.paid_date_key < :reference_end"},
        "params":[{"name":"reference_start","type":"int","default":0,"min":0,"max":99991231},
                  {"name":"reference_end","type":"int","default":99991231,"min":0,"max":99991231}]}'
Then the fixed field on users that references it, one value per user:


- name: reference_paid
  expression:
    dialects:
      - dialect: ANSI_SQL
        expression: SUM(CASE WHEN orders.paid_date_key >= 0 AND orders.paid_date_key < 99991231 THEN orders.paid_amount END)
  datatype: Decimal
  custom_extensions:
    - vendor_name: DATUS
      data: '{"v":"1.11","is_dimension":false,"lod":{"expr":"fixed(reference_paid_amount, users.user_id)"}}'
The 0 and 99991231 in the core SQL are the parameter defaults substituted in, as the previous section requires. A query selects April with --param reference_start=20300401 --param reference_end=20300501.

Each parameter declares name, type and default. Types int and float may set min/max; string may set allowed or leave its domain open. Allowed values and numeric bounds are mutually exclusive. With neither, datus_param_unbounded is a compilation warning, not by itself a validate --strict failure. Declare known business domains. Integer bounds do not validate date codes, and D-PARAM has no date type.

A predicate references a parameter with a colon followed by its name. Supply values through query params or CLI --param, not SQL concatenation. Unbound parameters use defaults.

One parameter value goes to every metric in the query that declares a parameter of that name: metrics the query selects directly, and base metrics behind the fixed fields the query reads. Two periods you want to control separately therefore need different names, such as the walkthrough's reference_start for the reference period and observation_start for the observation period; sharing one name forces both periods to be the same. The value must also satisfy the type and domain of every declaration of that name. A fixed field's calculation accepts one value per parameter, never a list.

The value reaches a fixed field's base however the query reads the field: through grouping, filtering, detail projection or a calculated field that depends on it. If no metric in the query declares the name, the query returns unknown_metric_param, usually because of a misspelling. A wrong type, a value outside the domain or a list for a fixed field's parameter returns param_out_of_domain.

When display records and a fixed field observe different periods on the same dataset, put the display date predicate in ordinary where_sql and the field's period in its parameterized base. A query time_range enters the fixed calculation too, intersecting the periods. A field's predicate adds constraints; it does not override source restrictions, time range or context.

For automatic query time-range routing, declare a native time dimension or use D-FORMAT for its encoding. Ordinary integer date-code fields may also be compared directly with integer parameters without being declared as time axes.

Across tables: decide where the result belongs¶
A fixed field may aggregate records from another dataset. The dataset containing the field is its host. It decides which records can appear in the result; entities present only in the aggregated dataset do not add host rows. Fixed grouping fields still belong to the host, and each needs an explicit correspondence to the other dataset.

Declared correspondence	How the aggregated dataset obtains a grouping field
Direct many-to-one relationship between the two, in either direction	

Skip to content
Dosi
Parameterized metrics



 
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
Table of contents
What can be parameterized
Declaring a parameter
Parameters on fields
A field's own parameters
A level-of-detail field's base
A derived subtype's thresholds
Binding at query time
Reading the result
Discovering what a metric takes
Limits and boundaries
Errors
See also
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
Parameterized metrics (D-PARAM)¶
revenue_1m_avg, revenue_3m_avg, revenue_6m_avg are one metric asked three ways. Modeled one per width they multiply: every new width is another metric to define, review, and keep in step with its siblings — and every dashboard that shows "the moving average" has to say which one it means.

A parameterized metric declares the width itself as a typed, query-time parameter. One moving_avg metric, and the query says how wide:

This reference example assumes a sales dataset with a primary time field, a region field, and an amount_total = SUM(sales.amount) metric, plus the moving_avg declaration below. The six east-region monthly totals are 10, 20, 30, 40, 50, and 60 from November 2024 through April 2025. model.yaml and sales.duckdb here are illustrative inputs, not bundled files. For a complete bundled-data walkthrough, start with Your first metric query.


dosi --model model.yaml query --metrics amount_total,moving_avg \
    --group-by metric_time:month --where "sales.region = 'east'" \
    --order metric_time__month --param n=1,3,6 --execute --db sales.duckdb
metric_time__month   amount_total  moving_avg__n_1  moving_avg__n_3  moving_avg__n_6
2024-11-01 00:00:00  10            10               10               10
2024-12-01 00:00:00  20            20               15               15
2025-01-01 00:00:00  30            30               20               20
2025-02-01 00:00:00  40            40               30               25
2025-03-01 00:00:00  50            50               40               30
2025-04-01 00:00:00  60            60               50               35
6 rows
A parameter fills a typed hole in the metric's declaration — never SQL text. It is declared with a type and a domain, so the model is valid for every value the parameter is allowed to take, and a binding can neither step outside that domain nor inject anything into the compiled SQL. The domain is where governance lives: it states which definitions this metric is permitted to take.

This page is the reference for the feature, and the specification of the D-PARAM extension key params: what can be parameterized, how to declare and bind it, and where the boundaries are. The DATUS entry it rides in, and the v / requires rules every key shares, are described with the envelope.

What can be parameterized¶
A parameter can fill exactly these seven holes, and nothing else:

Slot	What it controls	Declared on	Type
rolling.periods	the width of a rolling window	window.periods under type: rolling	int, domain ≥ 1
frame.preceding	how many buckets a general frame looks back	window.frame.preceding	int, domain ≥ 0
offset.count	how many periods back a comparison reaches	window.offset.count (object form; the "1 month" sugar takes no parameter)	int, domain ≥ 1
rank.buckets	the bucket count of an ntile	window.rank.buckets	int, domain ≥ 1
value.n	the position an nth_value reads	window.value.n	int, domain ≥ 1
filter.literal	a literal in a derived filter metric's predicate	the where of a derive of type: filter	int, float or string
field.literal	a literal in a field's expression	the field's own params (below)	int, float or string
The five window slots are counts, so they take integers whose domain has a declared lower bound at or above the slot's own minimum. A filter.literal or field.literal hole takes any of the three types (region = :r, amount > :t, datediff_pay < :n).

Everything else is fixed at modeling time. A parameter reference in a metric's expression, in a field with no params, in a query's where_sql, or in any other extension key is an error, never a silently rendered literal. Dimensions, grains, aggregate functions and dialects cannot be parameterized.

Declaring a parameter¶
Parameters are declared on the metric, in the params list of its DATUS extension entry, alongside the window or derive payload whose hole they fill. Reference the hole with {"param": "<name>"} in a window slot, or with :<name> in a filter predicate.


metrics:
  # A rolling width as a parameter: `moving_avg` at n=3 is the classic
  # 3-month moving average; n=[1,3,6] is three columns in one query.
  - name: moving_avg
    description: n-month rolling average (n is a query-time parameter)
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "SUM(sales.amount)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: |
          {"v": "1.5",
           "params": [{"name": "n", "type": "int", "default": 3, "min": 1, "max": 12,
                       "description": "trailing window width in buckets of the queried grain"}],
           "window": {"type": "rolling", "function": "avg", "periods": {"param": "n"}}}

  # A filter literal as a parameter. The authored expression is the default
  # (n = 7) instantiation — the two must agree.
  - name: paid_within_n
    description: Payments within the first n days after registration
    expression:
      dialects:
        - dialect: ANSI_SQL
          expression: "SUM(CASE WHEN base.datediff_pay < 7 THEN base.sum_pay_amount END)"
    custom_extensions:
      - vendor_name: DATUS
        data: |
          {"v": "1.5",
           "params": [{"name": "n", "type": "int", "default": 7,
                       "allowed": [1, 3, 7, 14, 30, 60, 90, 180],
                       "description": "days since registration, exclusive upper bound"}],
           "derive": {"type": "filter", "base": "pay_total", "where": "datediff_pay < :n"}}
Each entry in params carries:

Field	Required	Meaning
name	yes	[a-z][a-z0-9_]*, unique within the metric; this is what a query binds
type	yes	int, float or string
default	yes	the value used when a query binds nothing; must lie in the domain
allowed	either	an explicit list of legal values
min / max	or	a closed range, either side optional
description	no	shown in the catalog; write it for the agent that will read it
allowed and min/max are mutually exclusive. A string parameter takes no min/max. A parameter of any type may also declare no domain at all — a date spelled the way its column stores it, a code the model cannot enumerate — and compiles with a datus_param_unbounded warning. An open string still binds as a quoted literal, never as SQL text, so it widens what a query may ask, not what it may inject; prefer allowed whenever the values can be listed. A parameter no slot references warns with datus_param_unused.

One parameter may fill several slots, and one predicate may hold several parameters (datediff_pay < :n AND platid = :plat). A predicate made only of parameters (:n < 7) is rejected — a filter must still test a column.

A metric that declares parameters but is queried without any behaves exactly like an unparameterized metric at its defaults, down to the generated SQL: the compiled model is instantiated at every parameter's default. For a filter slot the authored expression must be that default instantiation (< 7 above), and the D-DERIVE equivalence check holds the two together. A bound value re-materializes the same measure a hand-written literal would produce — paid_within_n at n = 30 and a hand-written SUM(CASE WHEN datediff_pay < 30 …) share one column of the base CTE.

A metric derived from parameterized members inherits their parameters: if ltv = paid_within_n / new_user_cnt, then ltv takes n without declaring it.

Parameters on fields¶
A field's own parameters¶
Since 1.12 a Field may declare params of its own, filling :name holes in its expression — the field.literal slot. The field compiles at the defaults, so a query without params reads it exactly as a plain field; a query that binds a name the field declares reads the field, wherever it appears — a group-by, a filter, a select projection, a metric's measure — computed under that value, and the response echoes it under bindings as {dataset, field, params}.


- name: big_customer
  datatype: Boolean
  expression: {dialects: [{dialect: ANSI_SQL, expression: "COALESCE(customer_margin >= :margin_min, FALSE)"}]}
  custom_extensions:
  - vendor_name: DATUS
    data: '{"v": "1.12", "is_dimension": true, "params": [{"name": "margin_min", "type": "float", "default": 5000, "min": 0}]}'
Rules: every hole is declared, or the field is dropped (field_dropped_needs_params; this is also how basic mode degrades, since it reads no declaration); a declared name with no hole warns (datus_param_unused); a level-of-detail field declares none (its parameters are its base's, below). A field is one column, so a field's parameter binds one value — a list is param_out_of_domain, also when a metric reads the field. A metric whose measures read a parameterised field inherits its parameters: describe_metric lists them, and a bound output column carries the binding in its name (big_count__margin_min_7000). A field computed from a parameterised field reads it at the defaults — a binding applies to the field the query names.

A level-of-detail field's base¶
A level-of-detail field's parameters belong to its filter base. Queries reading the field bind them; field bases take single values, and independent windows use separate names. The core expression must agree with the defaults. See Independent windows and parameters for the complete rules.

dosi list dimensions, GET /v1/dimensions and MCP list_dimensions list them as params on the field's row, and on the row of any field that reads it, each with declared_on naming the metric that declares it.

A derived subtype's thresholds¶
A derived subtype's derived_by rule may hold :name placeholders. Their declarations live on the mapping document — the one ontology-side custom_extensions slot, since the upstream Concept schema takes no extra keys — under concept_params, keyed by concept, each an ordinary params array:


custom_extensions:
- vendor_name: DATUS
  data: '{"v": "1.12", "concept_params": {"PremierCustomer": [
    {"name": "margin_min", "type": "float", "default": 3000, "min": 0},
    {"name": "orders_min", "type": "int", "default": 6, "min": 1}]}}'
Lowering declares them on the subtype's flag column, so from there on they are field parameters: --param margin_min=4000 on select --from PremierCustomer, --group-by PremierCustomer or a metric that reads the flag is a what-if that changes no file. Placeholders and declarations must agree one-to-one (ontology_subtype_params). Basic mode ignores concept_params like every DATUS key: a rule with placeholders is then recorded unmapped (derived_by_needs_params) — describable, not queryable — while a literal rule works in both modes.

Binding at query time¶
Bindings are per query, by parameter name, and reach every queried metric that declares that name — directly or through its derive members.


CLI
REST / MCP / Python

dosi --model model.yaml query --metrics moving_avg \
  --group-by metric_time:month --param n=6
--param is repeatable (--param n=6 --param k=4). A comma-separated value is a list; quote a string value that itself contains a comma (--param city=NYC,"Paris, FR" is two values).


Omitted parameters take their declared defaults. Binding nothing is always legal and always means "the metric as modeled".

A list expands the metric into one column per value. Two lists take the cartesian product, first declared parameter outermost. The expansion is capped at 64 metric columns per query.

Types are strict. An int parameter rejects 7.5 and "7". The single implicit conversion is that a float parameter accepts a JSON integer.

Reading the result¶
A default binding leaves the column named after the metric — moving_avg. A non-default binding names it {metric}__{param}_{value}:


moving_avg__n_6            # n = 6
ltv__n_30                  # inherited parameter
rate__t_1_5                # float 1.5
region_share__r_o_neil     # strings are sanitized
by_channel__c_h0898d4b7    # …unless sanitizing loses the value
Several parameters append in declaration order, and under a list binding every column is suffixed — including the one at the default. The rule exists so two dashboards can never show the same metric name over different widths.

That last line is the one exception. The sanitizer keeps ASCII alphanumerics only, so it cannot tell an all-CJK allowed list apart — every value would suffix the same way — and the same goes for us-east next to us_east. Those values take an opaque token: h plus a hash of the value, stable across queries and across edits to the rest of the domain. A value whose sanitized stem is unique in its domain keeps it, so a readable ASCII domain is named exactly as it always was. Compiling a model with an opaque domain warns with datus_param_opaque_token and names the mapping; the CLI prints a legend under the result table, and outputs (below) always carries the real value:


dosi --model model.yaml query --metrics by_channel --group-by t.day \
    --param c=自营,经销 --execute --db repair.duckdb
day         by_channel__c_h06048f43  by_channel__c_h0898d4b7
2026-01-01  100                      50
by_channel__c_h06048f43 = c="自营"
by_channel__c_h0898d4b7 = c="经销"
The token is a function of the value and its declared domain and nothing else, so it does not move when a query binds a different set, when allowed is reordered, or when a value is added to it. A string parameter with no allowed has no other values to compare against, so its token has to be injective on the value alone: a value that already is its own stem (v1, us_east) names the column, and any other (us-east, 自营, o'neil) takes a 16-digit opaque token. There is no compile-time warning for these — the domain is not known until the query binds it.

A bound column name is at most 63 bytes, the identifier limit of PostgreSQL, GaussDB and DWS, which cut a longer name without an error. Two date parameters already pass it, each date being a 16-digit token, so a binding that would is spelled as one token for the whole binding: h and 16 hex digits of a hash of the full name (revenue_cur__h5e1698108f4d9c01 for cur_start and cur_end together). A metric name over 44 bytes is cut to leave the token room. The limit is the same on every warehouse, so a query's columns are named alike everywhere.

Every response also carries the machine-readable form, outputs — one entry per metric column with its metric and the parameters that produced it, defaults included:


dosi --model model.yaml query --metrics amount_total,moving_avg \
    --group-by metric_time:month --param n=3,6 --format json --execute --db sales.duckdb
{
  "dialect": "duckdb",
  "sql": "WITH base AS (…) SELECT … AVG(sales_amount_sum) OVER (ORDER BY metric_time__month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg__n_3, … AS moving_avg__n_6 FROM base",
  "outputs": [
    {"column": "amount_total",    "metric": "amount_total"},
    {"column": "moving_avg__n_3", "metric": "moving_avg", "params": {"n": 3}},
    {"column": "moving_avg__n_6", "metric": "moving_avg", "params": {"n": 6}}
  ],
  …
}
outputs appears in the CLI JSON, the REST and MCP query responses, the Python result dicts, and explain; over Arrow the same information rides as field metadata (dosi.metric, dosi.params). Read it instead of parsing column names.

One consequence for ordering: an order key names a metric or a group-by item, never a column the engine assigned — so you never have to reproduce a suffix rule to sort. A metric name resolves while that metric is a single column, suffix and all (--param n=6 --order -moving_avg orders by moving_avg__n_6). Under a list binding there is nothing to resolve it to, and the query is refused with unknown_order_key; bind the one value you want to rank by, which is what the error's suggested_retry hands back:


dosi ... --param n=1,3 --order -moving_avg
error: UnknownOrderKey: metric "moving_avg" is bound to 2 columns; an order key
       names a metric or a group-by item, not one of the bound columns
  candidates: {"n": 1}, {"n": 3}
  hint: bind a single value, then order by the metric name: "params": {"n": 1}
Discovering what a metric takes¶
dosi list metrics --format json and GET /v1/metrics carry each metric's params — the declaration plus the slots it fills, and declared_on when the parameter is inherited from a member:


dosi --model model.yaml list metrics --format json
{
  "name": "moving_avg",
  "params": [
    {"name": "n", "type": "int", "default": 3, "min": 1, "max": 12,
     "description": "trailing window width in buckets of the queried grain",
     "slots": ["rolling.periods"]}
  ],
  …
}
For agents, the MCP list_metrics tool lists only the parameter names, and describe_metric carries the declarations plus param_schema — a JSON Schema for the params map, with allowed rendered as enum and min/max as minimum/maximum. An agent reads the schema, then binds within it.

Whether an engine supports parameters at all is in dosi info --format json and GET /v1/capabilities, as the params key of the DATUS extension registry ("since": "1.5").

Limits and boundaries¶
Seven slots, listed above. Grains, dimensions, aggregate functions, and dialects are not parameterizable.
One binding set per query. params applies to every queried metric that declares the name; there is no per-metric override, so two metrics sharing a parameter name are asked at the same value.
64 metric columns per query after list expansion (param_expansion_too_large).
Attribution binds one value per parameter. dosi attribute --param n=30 is fine; a list is refused with a single-value retry. The resolved bindings come back in comparison_metadata.params. See attribution.md.
Basic mode ignores parameters. Under --osi-basic the D-WINDOW and D-DERIVE hosts are ignored, so every slot takes its declared default and a query-time params is rejected as unknown_metric_param.
Older engines. An engine below DATUS 1.5 rejects {"param": …} in a window slot structurally (the field is an integer there) and the "v": "1.5" envelope at the version gate — which is the guard that matters for :name in a filter, since a 1.4 engine would otherwise render the bind marker into SQL. A 1.4 server, however, tolerates unknown request fields: it silently drops a client's params and answers at the defaults; a current one refuses any key a query does not define with unknown_query_key. Check GET /v1/capabilities (or dosi info) before relying on binding remotely.
Errors¶
Every rejection is structured, names the metric, and carries a suggested_retry you can send back unchanged. Full catalog in errors.md.

Code	Raised when
unknown_metric_param	no queried metric declares the bound name; candidates lists the names that are declared
param_out_of_domain	the value is the wrong type or outside the domain — also an empty or repeated list, or a list where one value is required
param_expansion_too_large	list

Skip to content
Dosi
Derived metrics



 
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
Table of contents
Filter, compose and lod
Window bases and window members
Composes inside composes
Cycles and metadata
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
Derived metrics (D-DERIVE)¶
Where: a Metric's custom_extensions. Payload key: derive, a JSON object with a type discriminant — plus the window key's base field (M2, below). Default (absent): the metric is self-contained, no structure metadata. Supported forms: M1 ships filter + compose; M2 adds window.base and compose-with-window-members; 1.10 adds lod (D-LOD, documented with the rest of D-LOD). reagg and cross-dataset filter predicates are reserved and reject not_implemented with a hint naming their replacement or milestone. A metric re-aggregated at a coarser grain, such as a share of the total or of a parent group, is written today as a compose over a denominator: a fixed / exclude level of detail for any metric, or a whole-partition window for one that adds up across the grouping removed. See Shares of a total.

Declares how a metric derives from other metrics of the same model — structure Ossie expressions cannot carry. Two degradation tiers: on the flatten tier (filter, compose over leaves/filter members) the metric's expression is a legal, semantically equivalent Ossie aggregate — basic mode computes it and gets the same numbers, and the engine enforces the equivalence at compile time (derive_expression_mismatch); on the stage tier (window.base, compose with window members) no flat expression can express the semantics, so the expression is only the nearest approximation and basic mode legitimately diverges (with the ignored-extension warning). Either way, treat the expression as tool-maintained output, not something to edit by hand. When it does not match, the derive_expression_mismatch error carries the expansion as expected_expression (validate's JSON output), ready to be written in place by a tool — or pasted by hand.


metrics:
  - name: net_revenue
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "COALESCE(SUM(orders.amount), 0) - COALESCE(SUM(refunds.ref_amount), 0)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.4", "derive": {"type": "compose", "expr": "revenue - total_refunds", "fill": 0}}'
  # Reuses net_revenue; `expression` is the expansion, so a change to
  # net_revenue's declaration breaks this metric loudly.
  - name: net_revenue_per_order
    expression:
      dialects: [{ dialect: ANSI_SQL, expression: "(COALESCE(SUM(orders.amount), 0) - COALESCE(SUM(refunds.ref_amount), 0)) / COUNT(orders.order_id)" }]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.4", "derive": {"type": "compose", "expr": "net_revenue / order_count"}}'
Filter, compose and lod¶
filter — {"type":"filter","base":<metric>,"where":<predicate>}: a dimensional-filter variant of base. The predicate folds into the aggregate argument (CASE WHEN pred THEN arg END; COUNT(*) and a count of a primary-key column fold to THEN 1, while a count of any other column keeps it, since THEN 1 would also count the rows where it is NULL), never a WHERE — non-matching time buckets keep existing (value NULL), which is what makes the variant safe under ROWS-framed windows. The predicate reads the base's own dataset, or — since 1.12, S-ATTR-3′ — one dimension that dataset reaches by a unique many→one path (where: "customers.premier_customer = TRUE" on a metric over orders; a count of the primary key then keeps its column so the aggregate still names its home); when base is guarded already, the predicate may read that same dimension (the two guards nest) but not another; any other cross-dataset predicate is not_implemented until M4. base is a plain metric over one dataset — a single aggregate, a ratio or any expression over several aggregates, in which case the predicate folds into each of them — another filter, whose predicate it joins with AND in the same CASE, or a compose over such metrics, under whose arithmetic it folds. A base that is, or is built from, a window or a level of detail is refused, and so are bind parameters on a filter over several aggregates or over a parameterized filter. The engine records the subset relation (subset_of) in metric metadata.
compose — {"type":"compose","expr":<arithmetic>,"fill":<n>}: scalar arithmetic over metric names ("revenue - total_refunds"). Identifiers resolve strictly to metric names — a qualified or unknown name is an error with candidates, never a silent column read. Aggregate calls are rejected (composition never re-aggregates). fill replaces a member's NULL after branch alignment, before the arithmetic (the cross-fact "missing bucket poisons the sum" fix). A filled division of two members is cast to DOUBLE and its filled divisor wrapped in NULLIF(…, 0), so a missing denominator reads NULL rather than a division error; the plain authored arithmetic still passes the equivalence check, and the guarded form is what runs. A compose with no time_dimension of its own buckets each member by the member's: its declared time_dimension, else its dataset's primary time — never another dataset's date. The engine detects linear combinations (Σ cᵢ·metricᵢ), records members + coefficients + the leaf-measure decomposition, and fixes the conformed dimension set (dimensions every member branch reaches, under the unique-join-path rule, plus metric_time when the members share one time axis or each resolves its own — S-TIME-5) at compile time — dosi list exposes it, and a query grouping outside it gets unconformed_dimension naming the member that misses the dimension. Members sharing no dimension warn compose_no_conformed_dimensions at compile time. Attribution metadata is labeled exact only for linear composes over SUM/COUNT-tier measures (a filtered COUNT(DISTINCT) overlaps its complement — never exact).
lod — {"type":"lod","expr":<level of detail>} declares an include, exclude or fixed metric. Arithmetic over these metrics uses compose. The core expression carries the required nested aggregate representation; basic mode fails closed. See the unified LOD declarations and expression rules.
Window bases and window members¶
Since M2 a derived metric can also sit on, or read, a window stage:

window.base — the window key (D-WINDOW) accepts "base": <metric>: the window computes over the named metric's series instead of this metric's own expression. The base may be a leaf, a filter metric, a ratio (1.9), a compose — whether its measures sit in one dataset or several — or a compose over another window's output (1.9). A cross-fact base's series only becomes a value after the branch merge, so the stage sits on the merge instead of on one aggregate. Rejected: naming a window metric directly (the inner window would have no stage of its own to sit on — wrap it in a compose, which gives it one). Metadata records derive_family: "window", the derive_base chain and the inherited subset_of. Windows over a filter base inherit the CASE-WHEN bucket preservation: buckets with no matching rows still exist, so ROWS frames stay calendar-correct and cumulative sums skip the NULLs.
A ratio or compose base is what makes "rank the markets by close rate", "this rate versus last month's" and "am I above the average rate" expressible — none of which a single aggregate can state.

Nested windows (1.9) go one step further: a window reading a compose over another window's output is a second level, lowered as one more SQL stage. The scorecard shape needs it — rank the markets within a brand, then weight the scores within a market — because the two comparisons partition in different directions and one OVER has one PARTITION BY. This is not the re-aggregation I-1 bans: the row count is unchanged, each level recomputes with the query, and the weighted value lands on every row rather than folding them away. Levels are capped at 3, and past the first one partition must be declared explicitly — the query_dimensions default reads clearly on a single stage, but stacked, "which dims, minus whose exclusions" is no longer answerable from one declaration. Note what the window does and does not average: a whole-partition avg over a ratio base is the unweighted mean of the per-row rates, which is not the pooled ratio of sums. Both are legitimate and they differ; the window is the one that says "mean of the rates". - compose with window members — a compose whose expr references window metrics runs the arithmetic over the window output ("cumulative AOV" = revenue_cum / order_count_cum: numerator and denominator each accumulate, then divide). Members may mix window metrics with leaves; all members share one window stage, so they must share a time axis (window members over different facts do when the facts' dates are paired by D-CONFORM). Attribution is always approximate and there is no leaf-measure decomposition — window output cannot be served from materialized aggregates. - filter members in compose — a compose member may be a filter metric ("new_order_count / order_count"); it composes on the flatten tier exactly like a leaf.

Composes inside composes¶
Compose members that are themselves composes are expanded inline at compile time: the compiled artifact is flat, exactly as if the inner arithmetic had been written out. That is what lets a formula be stated once and reused — and, because the outer's expression is checked against the expansion of the inner's declaration, editing the inner and leaving a stale copy in a consumer is a derive_expression_mismatch rather than a number nobody questions.

Four consequences worth knowing before you nest:

The expression is the full expansion. Write out the whole flattened arithmetic, including the parentheses the nesting implies and the CAST(… AS DOUBLE) an inner bare-ratio member compiles to. It is tool-maintained output — regenerate it from the declaration rather than editing it by hand.
fill pushes down to the leaves. Each leaf reference takes the innermost fill declared on its path, so an outer fill: 0 still repairs a missing branch inside a member that declared none. The decision is per reference: one leaf reached through two members that fill it differently keeps both.
derive_members is the flattened leaf list on the flatten tier, with coefficients folded across the levels (quadrupled = doubled × 2 over doubled = revenue × 2 reports revenue at 4). Conformed dimensions intersect along the DAG, and unconformed_dimension therefore names the leaf that misses the dimension, not the intermediate compose. dosi lineage still draws the edge to the member you wrote.
On the stage tier a compose member stays a member. A window anywhere in the tree puts the metric on the stage tier, and there inlining would be wrong: each member is its own series out of the window stage, so dissolving an intermediate compose into leaves would lose arithmetic the stage never projects (cum_aov / revenue would lose the division and become two window members over a leaf). derive_members is therefore the DIRECT members there, the planner computes each nested compose as a hidden stage output, and substitution runs innermost-first at lowering. The tier decision reads the whole tree, so a window on an intermediate compose stages everything above it too.
Nesting is capped at 256 member references in one expansion — an inlined member is substituted at every occurrence, so referencing one twice doubles the expansion per level.
Cycles and metadata¶
Cycles among metric references are metric_reference_cycle with the full path. Metadata surfaces on every host (dosi list, REST, MCP, Python): derive_family, derive_base, derive_expr, subset_of, derive_members, leaf_measures, attribution, conformed_dimensions, and required_dimensions — the dims a query has to group by, gathered over the whole chain: a window's partition.exclude (window-extension.md#frozen-degrees) and the dims a keyed exclude or fixed names (D-LOD). requires_time_axis says whether the query also needs the time axis at a grain. A lod metric's derive_expr is its declaration in canonical spelling.

 Back to top
Next
Cross-fact alignment
Copyright © Datus · Licensed under the Elastic License 2.0


Skip to content
Dosi
Cross-fact alignment



 
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
Table of contents
How a conformed edge is queried
What the pairing makes possible
Validation
Why ignoring the key is silent
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
Cross-fact alignment (D-CONFORM)¶
Where: a Relationship's custom_extensions. Payload key: cardinality, enum "many_to_one" | "many_to_many". Default (absent): "many_to_one" — the ordinary foreign-key edge the join walk follows. Impact if ignored: silent — list it in requires.

Two fact tables often share dimensions — a date, a platform — without either being the many side of the other: a login row does not belong to a payment row. Strict OSI can relate them only through a shared dimension table, so a cross-fact metric like payers per active player by date has no date to group by (unconformed_dimension). many_to_many declares the pairing directly: each from_columns / to_columns pair is the same dimension spelled on each side. Grain: what one row means explains why facts are aligned rather than joined row to row.


relationships:
  - name: login_payment_conformed
    from: netbar_login
    to: netbar_payment
    from_columns: [stat_date, platform_id]
    to_columns:   [pay_date,  platform_id]
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.8", "requires": ["cardinality"], "cardinality": "many_to_many"}'
How a conformed edge is queried¶
A conformed edge is never joined row to row — that would fan every SUM / COUNT out. Each fact aggregates alone, projects its own spelling of the paired column under the query's output name, and the branches merge on that name exactly as S-FAN-3 merges any two branches:


{"metrics": ["pay_rate"], "group_by": ["netbar_login.stat_date"]}

WITH m0 AS (SELECT netbar_payment.pay_date AS stat_date, COUNT(DISTINCT netbar_payment.player_id) AS payers
            FROM main.netbar_payment GROUP BY netbar_payment.pay_date),
     m1 AS (SELECT netbar_login.stat_date AS stat_date, COUNT(DISTINCT netbar_login.player_id) AS users
            FROM main.netbar_login GROUP BY netbar_login.stat_date)
SELECT COALESCE(m0.stat_date, m1.stat_date) AS stat_date, CAST(m0.payers AS DOUBLE) / m1.users AS pay_rate
FROM m0 FULL OUTER JOIN m1 ON m0.stat_date = m1.stat_date OR m0.stat_date IS NULL AND m1.stat_date IS NULL
What the pairing makes possible¶
Either spelling works in group_by, where_sql and time_range.dimension, for metrics on either fact — netbar_payment.pay_date groups a login-only metric too; the output column keeps the spelling the query used.
metric_time resolves per branch when the facts' primary time dimensions are paired: each branch filters and groups its own date.
A grain rolls up per branch (stat_date:month): each fact truncates its own column and the merge key is the truncated value. Facts stored at different grains may be paired; the coarser side's D-GRAIN floor applies.
Transitive: payment ↔ login and login ↔ register pair payment with register too — three facts, one date dimension.
Only the paired columns are shared. Grouping a cross-fact metric by an unpaired column is unconformed_dimension; filtering on one is no_join_path, with a hint that the filter cannot be applied to one side alone (make the filtered member a D-DERIVE filter metric instead). A conformed edge cannot be named in a relationship path (invalid_relationship_path) — there is no row on the other side to reach.
Listed once. list_dimensions for a metric reports a conformed dimension as one row, under the spelling on the metric's own dataset (any spelling is still a legal group_by). When the paired columns are the metric's time axis, that row is metric_time.
S-FAN-6 still holds: a count inside a ratio reads NULL for a date that has rows on one side only; only a bare count metric fills 0.
A pair may be stored differently on each side — a native DATE on one fact, a D-FORMAT %Y%m%d string on the other. A pairing says the columns are one dimension, not one encoding, so every bound the planner moves across is folded into the target column's encoding: the single time range, each tagged range's CASE gate, and the scan predicate those gates OR together. The one thing that cannot cross is a where_sql comparison — D-FORMAT refuses an encoded column there (the literal would not be in its encoding), and naming the pair's native side is refused the same way rather than silently rewritten (unsupported_filter, naming both spellings). Use time_range with that dimension instead.
A window metric may be grouped by either spelling, and a pair of dates is one time axis. A window over one fact asked for under the other fact's name is that fact's own series, ordered by its own column, under the name the query used. A window whose own members span two paired facts — the period-over-period of revenue - refunds — orders the merged rows by the name the query used while each branch reads its own date; it needs no metric-level time_dimension, and a time range on either spelling (or metric_time) lands on both facts. Facts whose dates are not paired have no shared axis (no_primary_time_dimension, pointing at the pairing).
Validation¶
Declared once, checked once: paired columns must agree on dimension.is_time and, when both declare a datatype, on the type; a dataset contributes at most one column to a conformed class; join_type is invalid on a many_to_many edge (invalid_datus_extension). Model validation still warns relationship_target_not_unique on the edge — correct under strict OSI, and the one signal a consumer that ignores the key gets.

Why ignoring the key is silent¶
Why silent: an engine that drops the key sees an ordinary many→one edge and joins the facts row to row. A COUNT DISTINCT compose then returns plausible numbers (every login without a same-day payment falls into the NULL bucket); a SUM across the join is refused as fan_out_risk only when the dimension sits on the other fact, and double-counts otherwise. Listing cardinality in requires makes a datus-mode engine without it fail at the version gate instead. Basic mode is that degradation by design.

 Back to top
Next
Time storage format
Copyright © Datus · Licensed under the Elastic License 2.0


Skip to content
Dosi
Time storage format



 
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
Table of contents
When to configure it
Quick start
Writing filters
Configuration reference
Grouping and range boundaries
Validation and known limits
How it affects SQL
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
Time storage format (D-FORMAT)¶
Where: a time Field's custom_extensions. Payload key: time, an object with format, storage and granularity. Default (absent): the column is assumed to be a native DATE / TIMESTAMP. Impact if ignored: silent — list it in requires.

When to configure it¶
Use D-FORMAT when a time field stores dates as strings or integers, such as '20260907' or 202609. Native DATE/TIMESTAMP fields normally need no configuration: that is the default when time.format is absent.

time.format describes the stored encoding, not the output display format. Declare it in a field's DATUS custom_extensions. Its effective is_time must be true, either explicitly or from the core datatype default. The declaration describes the result of the field's expression, which may be a calculation rather than a source column.

Quick start¶
Within a dataset's fields, declare a string date and an integer month as follows. storage is inferred from datatype: String gives "string", Integer gives "int". Without datatype, it still defaults to "string". Include "time" in requires so older Datus-mode engines that do not support it reject the model instead of ignoring it.


fields:
  - name: dt
    expression:
      dialects: [{dialect: ANSI_SQL, expression: dt}]
    datatype: String
    dimension: {is_time: true}
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.6", "requires": ["time"], "time": {"format": "%Y%m%d"}}'
  - name: etl_month
    expression:
      dialects: [{dialect: ANSI_SQL, expression: etl_month}]
    datatype: Integer
    dimension: {is_time: true}
    custom_extensions:
      - vendor_name: DATUS
        data: '{"v": "1.6", "requires": ["time"], "time": {"format": "%Y%m", "storage": "int"}}'
For a dataset named events, a query's time-range fragment still uses YYYY-MM-DD, regardless of the stored encoding. The start is inclusive and the end exclusive:


time_range:
  dimension: events.dt
  start: "2026-01-01"
  end: "2026-04-01"
Writing filters¶
Use time_range to filter D-FORMAT encoded fields. Its bounds always use YYYY-MM-DD; the engine converts them to the declared storage encoding.

where_sql rejects any reference to a D-FORMAT encoded field with unsupported_filter, even when the literal uses the correct encoding. This also applies to computed dataset fields, references inside functions or CAST, IS NULL, IN, BETWEEN, and comparisons between fields. Remove the reference from where_sql and use time_range instead. Conditions that cannot be expressed by time_range, such as selecting isolated dates or NULL values, are not supported through where_sql for these fields.

Native temporal declarations (date, datetime, timestamp, timestamp_tz) remain allowed in where_sql, as do fields without a D-FORMAT encoding.

This restriction applies to query where_sql. D-DERIVE filter and filters inside metric expressions still require literals in the stored encoding; they do not convert ISO date literals automatically. For example, a %Y%m%d string field requires '20260401', not '2026-04-01'. Use dosi list dimensions to inspect time_format and time_storage.

Configuration reference¶
The time object accepts format, storage, and granularity. Besides the native type names below, format accepts a continuous, ordered, fixed-width prefix of %Y%m%d%H%M%S. %Y is a four-digit year; %m, %d, %H, %M, and %S are two-digit, zero-padded month, day, hour (00–23), minute, and second. Stop after any component, but do not skip, repeat, or reorder them.

Any fixed literal text may appear between components or as a prefix/suffix: %Y-%m, %Y年%m月%d日, %Y.%m.%dT%H:%M, and %Y%m%d%H%M%S are valid. Write %% for a literal percent sign. Unknown directives, variable-width modifiers, fractional seconds, and timezone directives are not supported. For example, %Y%d, %d/%m/%Y, %-m, and %Y%m%d%z are rejected.

These combinations extend the existing 1.6 contract without a version bump. Engines predating this enhancement reject layouts they do not recognize:


custom_extensions:
  - vendor_name: DATUS
    data: '{"v":"1.6","requires":["time"],"time":{"format":"%Y年%m月%d日"}}'
Every stored value must follow the declared layout, including literal text and padding. Bare-column range comparisons require a column collation that orders these fixed-width values chronologically; use an appropriate binary/code-point ordering when necessary. The declaration does not inspect or change collation.

The original six encodings remain supported:

format	Stored value	Inferred grain	Allowed storage
%Y%m%d	20260907	day	string, int
%Y-%m-%d	2026-09-07	day	string
%Y/%m/%d	2026/09/07	day	string
%Y%m	202609	month	string, int
%Y	2026	year	string, int
%Y-%m-%d %H:%M:%S	2026-09-07 12:30:00	Not inferred	string
Native temporal declarations do not parse or re-encode the field:

format	Meaning	Inferred grain
date	Native date	day
datetime, timestamp, timestamp_tz	Native date and time	Not inferred
Prefer datatype: Date, DateTime, or DateTimeTz for new native temporal fields; no time.format is needed. The legacy spellings remain supported: date must agree with Date, datetime and timestamp with DateTime, and timestamp_tz with DateTimeTz. Native temporal datatypes reject stored patterns such as %Y%m%d. Time has no supported D-FORMAT encoding.

Native declarations do not accept storage. For encoded fields, an explicit storage must agree with datatype (String → "string", Integer → "int"); other declared datatypes cannot use patterns. Omitted storage is inferred from the datatype, or defaults to "string" when datatype is absent. "int" accepts only uninterrupted component prefixes without literal text (including literal digits), such as %Y%m%d%H or %Y%m%d%H%M%S. The longest encoding has 14 digits: choose a sufficiently wide integer-valued column, such as BIGINT or DECIMAL(n,0), for a compact timestamp. A physical DECIMAL(n,0) column holding an integer date key may declare logical Integer; logical Decimal is not an integer encoding. storage cannot be declared without format. These checks use the field expression's result type, not the source column type. Basic mode ignores the Datus block with a warning, while retaining the core datatype time-role default.

granularity accepts day, week, month, quarter, or year. It may be omitted when the format implies it. time.granularity and time_granularity are equally supported spellings: neither is deprecated, but they must agree when both are present. A declared grain cannot be finer than the encoding. A time object may declare only granularity, but cannot be empty. Year/month/day prefixes infer year/month/day regardless of their literal text; prefixes including a clock component do not infer a grain. Hours, minutes, and seconds in storage do not enable sub-day grouping or date-time time_range bounds: those query features remain unsupported. Parsed output uses January/day one for omitted calendar components and zero for omitted clock components; for example, %Y-%m denotes the first day of that month, and %Y%m%d%H denotes the start of that hour.

Grouping and range boundaries¶
A %Y%m field can be grouped by month, quarter, or year, but not by day or week: its stored values contain no such detail. Likewise, %Y supports year grouping. Requesting a finer grain produces grain_too_fine.

Range boundaries must align with the stored periods. For %Y%m, end: "2026-03-15" produces grain_too_fine: use "2026-03-01" to exclude March or "2026-04-01" to include it. The error offers both aligned bounds; the engine does not choose whether to include a partial month. Year encodings require year boundaries. These checks apply to the query's bounds; scan expansion for window calculations is handled by the engine.

Selected time dimensions are parsed into date/time values, including when no grain is requested. Do not rely on results retaining the stored text layout. D-FORMAT does not provide an output display-format setting.

Validation and known limits¶
The declaration must match the data. Dosi checks supported formats and configuration combinations, and — where the field declares core Field.datatype — that the declared storage and native format agree with it. It does not inspect warehouse column types or values, so dosi validate cannot establish that %Y%m%d matches the actual field expression's output.

A wrong format or storage type can cause a warehouse error, NULL groups, or incorrect filtering without an error. For example, declaring %Y%m%d for values stored as '2026-09-07' also generates filter literals in the wrong layout. Check both the expression's type and representative stored values.

ClickHouse uses parseDateTime, whose DateTime result is limited to roughly 1970-01-01 through 2106-02-07. Earlier or later dates, including an SCD2 '99991231' end-date sentinel, are not supported reliably for grouping or parsed projections. Verified on ClickHouse 24.8.14: parseDateTime rejects 1960, 2107, and 9999 dates with CANNOT_PARSE_DATETIME. parseDateTime64 is unavailable in that version. The BestEffort alternatives are not a safe replacement: parseDateTimeBestEffort clamps out-of-range values, and parseDateTime64BestEffort changes 9999-12-31 to 2299-12-31. Bare encoded time_range comparisons do not use this parser, but that does not make an out-of-range grouping safe.

In basic mode, DATUS extensions are ignored with a warning, including requires. Use Datus mode for fields that depend on encoded-date handling. Epoch encodings such as epoch_seconds and epoch_millis are not supported.

How it affects SQL¶
Grouping parses an encoded field into a date/time value and truncates it when a coarser grain is requested. At the stored grain, or with no requested grain, the GROUP BY key can remain the stored column and the SELECT expression parses it.

For time_range, the engine instead converts the bounds into the stored encoding. The examples above generate predicates of these forms:


WHERE dt >= '20260101' AND dt < '20260401'
WHERE etl_month >= 202601 AND etl_month < 202604
Keeping parsing functions off the filter column helps warehouses use partition pruning. Actual pruning depends on the warehouse, field expression, and partition definition.

 Back to top
Next
Versions & compatibility
Copyright © Datus · Licensed under the Elastic License 2.0


Skip to content
Dosi
Versions & compatibility



 
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
Table of contents
Versioning policy
Change log
Not yet implemented (roadmap)
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
Extension versions and compatibility¶
Every DATUS payload may declare the version it was written against in v, and list in requires the keys an older engine must not drop — both are described with the envelope. Any engine reports the version it implements: dosi info, GET /v1/capabilities, or dosi_engine.DATUS_EXT_VERSION. This page is the policy behind that number and the record of what each version added.

Versioning policy¶
Every key this engine reads is registered with the version that introduced it and what it costs a consumer to ignore it. The registry is built into the engine and is the source of truth: the version gate, the basic-mode warning text, dosi info and GET /v1/capabilities all read it at runtime, so none of them can drift from it. The change log below is written by hand against that registry.

Ignore-impact levels, ordered by increasing danger — note that failing loudly ranks safer than quietly returning different numbers:

Impact	Meaning
inert	Presentation metadata the engine never consumed. Results are identical.
degraded	No wrong numbers — the request fails loudly instead (a structured error, or a query name that stops resolving).
documented	Numbers change, but predictably and in a documented direction (LEFT vs INNER, NULL vs 0).
silent	Numbers are wrong and the caller cannot tell. Must be listed under requires.
Three shipped keys are silent: measure (D-MEASURE, 1.4), time (D-FORMAT, 1.6) and cardinality (D-CONFORM, 1.8) — see the change log. Every other key is degraded or documented, so basic mode remains coherent for models that use none of them. A model that uses one must list it under requires, which is what stops a datus-mode engine that does not implement the key (an older Dosi, another vendor's reader of the envelope) from silently computing a different number: it fails at the version gate with datus_ext_key_required instead. Basic mode is a different case — it never reads the payload at all, requires included, and only warns ignored_vendor_extension; a model with a silent key should not be run in basic mode.

When to bump:

MINOR — purely additive: a new key, or a new accepted value for an existing key that no old payload could have used. Old payloads keep working unchanged; a new payload on an old engine gets datus_ext_version_ahead and loses only the new key. A new key whose impact is silent is still a minor bump, but the producer must list it under requires.
MAJOR — only when an existing key changes meaning, is renamed, or is withdrawn. Every payload declaring an older v is then rejected per changed key, with an instruction for updating the YAML.
Neither — engine-internal work that changes no key's contract. A version with no key behind it would make the registry's invariants vacuous.
Change log¶
Each released minor and the keys it added. The machine-readable form of this table is capabilities().history — dosi info --format json or GET /v1/capabilities.

Version	Date	Keys added	Impact if ignored
1.0	2026-07-11	join_type (D-JOIN), fill_nulls_with (D-FILL)	documented, documented
1.1	2026-08-02	time_dimension (D-TIME), time_granularity (D-GRAIN), dataset (D-DATASET)	degraded, documented, degraded
1.3	2026-08-09	(none — in-family growth of window)	(inherits window's documented)
1.4	2026-08-12	derive (D-DERIVE M1), measure (D-MEASURE)	degraded, silent
1.5	2026-08-26	params (D-PARAM M1)	degraded
1.6	2026-09-02	time (D-FORMAT)	silent
1.7	2026-09-11	is_dimension (D-DIM)	documented
1.8	2026-09-16	cardinality (D-CONFORM)	silent
1.9	2026-09-16	(none — in-family growth of window)	(inherits window's documented)
1.10	2026-09-23	lod (D-LOD; also the lod family of derive)	degraded
1.11	2026-10-07	(none — in-family growth of lod)	(inherits lod's degraded)
1.12	2026-10-25	concept_params (D-PARAM, on the mapping document); params grows onto the field carrier	degraded
1.0 (2026-07-11): initial — D-JOIN (relationship join_type, implemented: the join is emitted as INNER or LEFT accordingly), D-FILL (metric fill_nulls_with, implemented 2026-07-12: the metric output is COALESCEd; overrides the S-FAN-6 count default, any metric kind).
1.1 (2026-08-02): time & attribution — D-TIME (dataset/metric time_dimension, implemented: a dataset's primary time dimension and a metric's own, the reserved metric_time query name, per-branch time-range fallback), D-GRAIN (field time_granularity, implemented: finer-grain requests → grain_too_fine), D-DATASET (metric dataset, implemented: COUNT(*) home attribution). Basic-mode ignored_vendor_extension warnings now cover all five carrier objects (previously only Relationship and Metric).
1.1 (2026-08-04, no version bump): the envelope version gate itself — v is now read and enforced (envelope), requires is introduced, each key carries a declared ignore-impact, and the whole registry is published through dosi info, GET /v1/capabilities, and dosi_engine.DATUS_EXT. No key was added or changed, so the version stays at 1.1. Payloads without a v — which is what the Datus agent emits today — are unaffected.
1.2 (2026-08-06): window metrics — D-WINDOW (metric window, implemented: the WindowStage step in --explain; pop/rolling/cumulative sugars + general offset/frame form, cumulative reset with scan expansion + post-window trim; ignore-impact documented — the metric falls back to its plain base aggregate; spec in window-extension.md). Initially DuckDB-executed; the all-dialect enablement (per-dialect offset-shift lowerings, verified on live warehouses) landed as a follow-up. In-family additions (same minor, fail-closed on engines predating them): mixed families / mixed shifts / mixed resets-with-start relaxed, and require_full_window on finite frames (2026-08-06) — see window-extension.md#restrictions.
1.3 (2026-08-09): window families & functions — grows the window key in-family, no new registry row (capabilities().history still ends at 1.2; the version signals the payload dialect). Additions: the rank family (row_number / rank / dense_rank / ntile / percent_rank / cume_dist over the metric value), the value family (first_value / last_value / nth_value), offset direction: "forward" (LEAD — next-period reference), frame order / partition / units / second modifiers (value-ordered and RANGE frames, partition modes, two-argument inputs), and registry levels W2 (stddev_pop/samp, var_pop/samp) + W3 (covar_pop/samp, corr). Every addition fails closed on a 1.2 engine: unknown top-level family keys fall into the exactly-one-family error, unknown in-family keys are rejected, unknown function names fail enum parse — and a declared "v": "1.3" trips the version gate besides. Spec: window-extension.md (family sections + function registry).
1.4 (2026-08-12): derived metrics & measure naming — D-DERIVE M1 (derive key, implemented: filter + compose families, metric references resolved with cycle detection, the expression equivalence gate, compile-time conformed dimensions, and derive metadata on every host surface; reagg / cross-dataset filter predicates / window.base are reserved and reject not_implemented naming their milestone — note window.base previously fell into the silently-ignored unknown-key bucket and now fails closed) and D-MEASURE (measure key, implemented: explicit measure naming, signature-keyed dedup and collision checks unchanged — the fix for the non-ASCII stem collision, and the first silent-impact key: emit it under requires). M2 (2026-08-19, in-family — no new keys, no version bump): window.base flips from reserved to functional (windows over leaf / filter metrics), compose accepts window members (stage-tier lowering: arithmetic over window output) and filter members (flatten tier). A 1.4 payload using these shapes compiles on an M2 engine and fails closed (not_implemented / rejection) on the M1 engine — never a silent misread. Nested compose (in-family — no new keys, no version bump): a compose member may itself be a compose, expanded inline at compile time. An older engine rejects the shape with the same invalid_datus_extension it always did, so this too fails closed. The metric stays on the flatten tier, and derive_members becomes the transitive leaf list with folded coefficients; a window anywhere in the tree keeps the combination not_implemented.
1.6 (2026-09-02): time storage formats — D-FORMAT (field time block, implemented: the column is parsed per dialect on the grouping side, and filter bounds are folded into the stored format before the SQL is built, with the grain inferred from the layout). time.granularity joins the flat time_granularity as an equal spelling — no deprecation, but the two may not disagree. The second silent-impact key: emit it under requires, so a datus-mode engine that lacks it fails at the version gate (basic mode never reads requires; see the envelope). A consumer that drops it does not fail loudly; it assumes a native date column and, for a month- or year-stored one, returns an empty result with no diagnostic. See D-FORMAT for configuration and limitations. Scope: string and integer layouts plus the native temporal types; epoch_seconds / epoch_millis are deferred because from_unixtime reads the session timezone and that needs its own round.
In-family addition (2026-09-08, still 1.6): time.format accepts continuous ordered fixed-width calendar prefixes with arbitrary fixed literal text. Filtering still folds date bounds to stored constants; grouping extracts components into a canonical date/time before dialect parsing. Existing layouts remain compatible. Older Datus engines reject new layouts rather than silently interpret them. - 1.7 (2026-09-11): field roles — D-DIM (field is_dimension, implemented: read by list_dimensions and by attribution's candidate list). The model can finally say that a column is a measurement, an identifier, free text or PII, so it stops being offered as a grouping dimension. It is a recommendation signal only: an explicitly named group_by is honored either way and no compiled SQL changes, hence documented impact and no requires entry. Where the key is absent the engine infers the role from model structure alone — keys, relationship columns and numerically aggregated columns — and never probes the warehouse. See D-DIM. - 1.8 (2026-09-16): conformed fact-to-fact relationships — D-CONFORM (cardinality key on a Relationship, implemented: conformed classes in the join graph, per-branch resolution of group-by / filter / time-range columns and of metric_time to each fact's own spelling, merge on the output name). A many_to_many edge is never joined; only its paired columns are shared. The third silent-impact key: emit it under requires — without it the edge is a many→one join and duplicate-sensitive aggregates fan out. See D-CONFORM. - 1.9 (2026-09-16): a ranked scorecard — grows the window key in-family the way 1.3 did, so there is no new registry row. Four additions driven by one shape.

frame.scope: "partition" states the frame as the entire partition instead of an offset from the current row, lowering to <func>(…) OVER (PARTITION BY …) with no ORDER BY and no frame clause. It is the second window function beside a rank — "how many rows am I ranked against" — which a ranked scorecard needs and which no combination of the existing keys could express: count with preceding: "unbounded" is a running count, and units: "range" still stops at the current row's peer group. scope is mutually exclusive with preceding, reset, require_full_window, units and order, each of which only describes an ordered frame; writing one is rejected rather than ignored.

window.base widens: a ratio or a compose can now be the series a window reads, so ranking a rate or taking its period-over-period is expressible. A cross-fact compose's series only becomes a value after the branch merge, so the window stage sits on the merge — the branches aggregate separately and their CTEs ride beside the stage's base. Such a base has a shared time axis only when the facts' dates are paired (D-CONFORM); otherwise the families that need one refuse on conformance.

Fails closed on a 1.8 engine twice over: it rejects the new keys inside frame and order as unknown, and the payload's "v": "1.9" trips the version gate. See window-extension.md.

Nested windows and NULL placement complete the shape. order.nulls: "first" | "last" pins where NULLs sort in a window ordering — left unset, each warehouse applies its own default and they disagree (DuckDB and PostgreSQL last ascending, MySQL-family and StarRocks first), which a ratio base turns from theoretical into routine. The generator emits the clause only where it differs from the target's default, and where a dialect has no such clause at all (MySQL, TiDB) the placement lowers to a leading CASE WHEN … IS NULL sort key — so the rendering varies by dialect and the meaning does not. And window.base goes one step further than a ratio: it accepts a compose over another window's output, so a second window can read the first one's. Each level lowers to one more SQL stage; depth is capped at 3 and level 2+ must declare partition. The guards that matter run across levels, not within one: the output trim stays outside the outermost stage, and a partition-global metric at any level refuses to share a query with a scan-expanding one at any other. Also in-family: a stage-tier compose may have a compose member (it stays a member rather than being inlined, since on that tier each member is its own series). - 1.10 (2026-09-23): levels of detail — D-LOD, one grammar on two carriers: the lod key on a Field and the lod family of the existing derive key on a Metric (implemented).

On a field, fixed(base, dims…) makes a per-entity aggregate groupable, filterable and re-aggregatable through an ordinary query (the LodRollup step in --explain; a D-FORMAT rollup key keeps its storage encoding and is parsed once, outside). When the view needs more than the rollup — a dimension outside its keys, a row-level field reading the value, several key families — the broadcast shape (LodBroadcast) joins the detail rows with one derived table per key family; an aggregate over a pinned value counts each of its keys once per group. No keys is the table-scoped FIXED, broadcast onto every row.

On a metric, include groups the inner stage by the query's own group-by plus the declared dimensions and collapses it with the declared outer aggregate; the grain and any D-FORMAT parse apply inside. exclude attaches a coarser aggregation of the same fact to the view's rows — LEFT JOIN on the dimensions they share, CROSS JOIN on none — recomputed rather than windowed so a COUNT(DISTINCT) base is right too. fixed is the field's declaration hosted on a metric so it can attach to the view: a table-scoped or key-scoped total computed before the dimension filter, which is what Tableau's {SUM([Sales])} is as a denominator and what exclude() cannot mean (LOD metrics). A conformed pairing whose endpoints are two facts' own fixed fields aligns them on one axis: each fact rolls up for itself and the branches merge, the facts are never joined.

Filter placement follows Tableau's order of operations. The where clause is a dimension filter — applied to a fixed rollup's output when it names only the rollup's keys, to the detail rows otherwise, never inside the rollup — while time_range is the context filter that scopes every rollup. A row-level field that reads only D-LOD values and constants aggregates once per key and can filter a reaggregation at that key grain. A calculated classification such as customer_margin >= 5000 AND customer_orders >= 6 therefore filters the fixed values before they are reaggregated. If the predicate reads a different fixed key family, it is evaluated where that family is available, before the grid is reduced.

Basic mode drops a D-LOD field (aggregate_field_dropped) and fails closed on a D-LOD metric, whose expression is the nested aggregate its declaration describes (nested_aggregate), rather than computing one level of it. See Understand LOD. - 1.11 (2026-10-07): levels of detail across datasets — grows the lod key in-family, no new registry row. A field's fixed base may aggregate another dataset than the field's own when every key corresponds to a column of it through a declared relationship: a many→one edge between the two in either direction, a many→one path from the base to the host, or a D-CONFORM pairing (a rollup over a scan of its own). The rollup is LEFT JOINed onto the host's rows on plain equality and a count reads 0 where nothing matched, so > 0, = 0 and >= n are the semi-join, the anti-join and a threshold; the context layer reaches the rollup through the base's own edges or the pairing, and a conjunct it cannot read there is left out (across datasets). A 1.10 engine fails closed: it refuses the base as aggregating another dataset. - 1.12 (2026-10-25): parameters on fields and derived subtypes. params grows onto the field carrier — a field expression's :name holes, the field.literal slot, bound by patching the field for the whole query and echoed under bindings (field parameters) — and concept_params lands on the Ossie mapping document, declaring the placeholders of a derived subtype's derived_by rule (subtype thresholds). Alongside, the semantics contract gains S-ATTR-3′: a CASE WHEN <guard> THEN <arg> END aggregate may read, in its guard, one dataset the arg's dataset reaches many→one — which is what lets a D-DERIVE filter on a fact metric be written where: "Customer is PremierCustomer". An older engine fails closed: concept_params is an unknown key, a :name in a field is not a column, and "v": "1.12" trips the version gate.

Not yet implemented (roadmap)¶
The queued RFCs are all additive, so each lands as one minor bump with its own registry rows. Order may change; every entry occupies a minor of its own.

Minor	RFC	Keys	Carrier	Impact	requires?
≥1.12	rfc-semi-additive-metrics §4	semi_additive	Metric	silent — an end-of-period balance gets summed across snapshots	yes
≥1.12	rfc-minimal-declarations §3 D-DISTINCT-STATE	distinct_state	Field / Dataset	silent — a materialized bitmap column is aggregated as if it were an ordinary column	yes
≥1.7	rfc-derived-metrics M3-M4	derive reagg + the general cross-dataset filter (window.base and compose window/filter members shipped in-family at 1.4/M2; the one-dimension guard shipped at 1.12 as S-ATTR-3′)	Metric	degraded/error — reserved shapes reject not_implemented today, never silently compute	no
—	window-extension.md rank/share/streak families, W2+ functions	future window shapes (in-family additions, fail-closed on older engines)	Metric	degraded	no
Three silent keys have shipped: measure (D-MEASURE) at 1.4, time (D-FORMAT) at 1.6 and cardinality (D-CONFORM) at 1.8. If you author models against a Datus agent that emits silent-impact keys, keep an eye on the requires list it produces: that is what stops an older engine fr
