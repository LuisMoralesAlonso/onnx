Skip to content
Dosi
REST API



 
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
Reference
Reference
CLI
SQL policy providers
REST API
Table of contents
Starting the server
Endpoints
The query body
The detail body
Arrow IPC streaming
Errors
Concurrency model
MCP server
Errors
Python API
Semantics contract
Benchmarks
Release notes
REST API¶
dosi-server serves an Apache Ossie (formerly OSI) semantic model over HTTP: validate models, browse the compiled semantic layer, compile metric queries to dialect SQL, and execute them against a warehouse. JSON only — the same machine contract as dosi --format json. For the shell equivalent of every endpoint, see the CLI reference.

Interactive docs: every running server hosts its own OpenAPI 3.1 spec at /openapi.json and a Swagger UI at /docs (generated with utoipa; the UI assets are vendored, so both work offline). This page is the narrative companion; the spec is the authoritative schema.

Starting the server¶

# compile-only, in-memory DuckDB for execution
export DOSI_EXAMPLES=~/.local/share/dosi/examples
dosi-server --model $DOSI_EXAMPLES/orders/model.yaml

# with warehouse connectors and a connections file (a `datasources:` YAML
# in Datus agent.yml vocabulary, or a full agent.yml)
dosi-server \
  --model model.yaml --connections dosi-connections.yaml --bind 0.0.0.0:8080
That is where the install script puts the bundled example models; any Ossie model file works just as well.

Flag	Env	Default	Meaning
--model	DOSI_MODEL	required	Ossie model file (.yaml/.json)
--connections	DOSI_CONNECTIONS	./dosi-connections.yaml → ~/.config/dosi/connections.yaml (legacy osi- paths still discovered)	warehouse profiles — Datus agent.yml datasources: vocabulary (connectors.md)
--bind	DOSI_BIND	127.0.0.1:8081	listen address
--db	—	in-memory	DuckDB file for connection-less executes
--max-concurrent-executions	DOSI_MAX_EXECUTIONS	16	execute concurrency cap
--pool-size	DOSI_POOL_SIZE	8	connection-pool cap per profile
--execute-timeout-secs	—	60	budget per warehouse execution
--request-timeout-secs	—	30	budget for non-execute requests
--auth-token	DOSI_SERVER_TOKEN	off	require Authorization: Bearer <token> on /v1/*
--disable-execute	—	off	compile-only deployment (execute → 403)
--osi-datus / --osi-basic	—	--osi-datus	engine mode, server-wide: datus honors DATUS custom_extensions; basic is strict standard Ossie — extensions ignored, logged as warnings at startup and reported by /v1/validate (cli.md)
The model is loaded, validated, and compiled once at startup — the process exits on an invalid model — and the compiled IR is shared immutably across requests. Warehouse executors are built once per profile, so MySQL-family and Postgres connections are pooled and reused.

Endpoints¶
Method	Path	Purpose
GET	/health	liveness — never requires auth
GET	/ready	readiness
GET	/docs, /openapi.json	Swagger UI / OpenAPI spec — never require auth
GET	/v1/model	model name, path, engine mode, object counts, datus_ext_version
GET	/v1/capabilities	engine / Ossie spec / datus-ext versions and every DATUS extension key this engine reads, with the version that introduced it and what it costs to ignore it (datus-extensions.md)
GET	/v1/datasets	datasets with source, keys, field counts
GET	/v1/datasets/{name}	one dataset in full: fields with their expressions and fixed-field declarations, keys, time axis, relationships (MCP describe_dataset)
GET	/v1/metrics	metrics with inferred kind and datasets
GET	/v1/metrics/{name}	one metric in full: expr, the datasets it can be grouped by, every parameter a query can bind for it, time grains (MCP describe_metric)
GET	/v1/dimensions[?metric=<name>]	dimensions (dataset.field) with time flags, is_dimension and source; scoped to one metric's accepted group-bys when metric is given
GET	/v1/connections	{name, dialect, available} — never endpoints or secrets
POST	/v1/validate	validate inline model text (in the server's engine mode; response includes warnings)
POST	/v1/query/compile	metric query → SQL
POST	/v1/query/explain	metric query → logical plan (text)
POST	/v1/query/execute	metric query → rows
POST	/v1/query/attribute	metric + dimensions + two windows → per-dimension decomposition of the change (attribution)
POST	/v1/select/compile	detail query → SQL (detail queries)
POST	/v1/select/explain	detail query → logical plan (text)
POST	/v1/select/execute	detail query → rows
The query body¶
compile, explain, and execute share one body: a MetricQuery plus dialect (default duckdb), pretty, and — execute only — connection (a profile name; omitted = local DuckDB).


{
  "metrics": ["revenue", "order_count"],          // required
  "group_by": [
    {"field": "customers.region"},                // dataset.field
    {"field": "orders.order_date", "grain": "month"}  // day|week|month|quarter|year
  ],
  "where_sql": "status = 'completed' AND revenue > 100",  // a selected metric: result rows; else input rows (semantics §8)
  "context_filter": "customers.region <> 'test'", // always pre-aggregation (the calculation population)
  "time_range": {"start": "2024-01-01", "end": "2025-01-01",
                 "dimension": "orders.order_date"},   // half-open [start, end)
  "order_by": [{"key": "order_date__month", "desc": true}],  // NULLs last; "nulls": "first" to lead with them
  "limit": 12,
  "params": {"n": [7, 30]},                       // D-PARAM bindings: value or list per name
  "dialect": "postgres",
  "connection": "warehouse-prod"                  // execute only
}

curl -s localhost:8081/v1/query/compile -H 'content-type: application/json' \
    -d '{"metrics":["revenue"],"group_by":[{"field":"orders.order_date","grain":"month"}],"dialect":"postgres"}'
{"dialect":"postgres","sql":"SELECT DATE_TRUNC('MONTH', orders.order_date) AS order_date__month, ...",
 "outputs":[{"column":"revenue","metric":"revenue"}],"warnings":[]}

curl -s localhost:8081/v1/query/execute -H 'content-type: application/json' \
    -d '{"metrics":["revenue"],"group_by":[{"field":"customers.region"}],"connection":"pg"}'
{"dialect":"postgres","sql":"...","columns":["region","revenue"],
 "rows":[{"region":"east","revenue":340},{"region":"west","revenue":110}],"row_count":2}

curl -s localhost:8081/v1/validate \
    -d "$(jq -Rs '{model: .}' < model.yaml)" -H 'content-type: application/json'
{"valid":true,"issues":[],"compile_errors":[]}
The detail body¶
The other query plane returns records rather than aggregates. Its body is the DetailQuery wire type, identical on the CLI, MCP and Python surfaces:


{
  "from": "orders",              // result grain: one row per order
  "fields": [                    // "field" | "dataset.field" | "rel.rel.field",
    "orders.order_id",           //   each with an optional ":grain"
    "orders.amount",
    "customers.name"
  ],
  "where_sql": "customers.tier = 'Enterprise'",
  "time_range": {"start": "2026-01-01", "end": "2026-02-01"},
  "order_by": [{"key": "amount", "desc": true}],  // projected columns only
  "limit": 100,                  // default 100, clamped to 10000
  "dialect": "postgres",         // execute/compile envelope, as above
  "connection": "warehouse-prod"
}

curl -s localhost:8080/v1/select/execute \
    -H 'content-type: application/json' \
    -d '{"from":"orders","fields":["orders.order_id","customers.name"],"limit":10}'
Datasets other than from are joined automatically when a field, filter or the time range needs them; the result grain stays from. A dataset reachable only against the join direction returns detail_fanout with the root to use instead, and a field several paths can reach returns ambiguous_join_path with the relationship prefixes that name each one.

Arrow IPC streaming¶
POST /v1/query/execute with Accept: application/vnd.apache.arrow.stream returns the result as a streamed Arrow IPC body instead of JSON: record batches flow from the warehouse adapter through the IPC writer into the response with no row materialization and no per-value JSON encoding. Columnar consumers (Polars, pandas/pyarrow, DataFusion, another dosi) read it zero-parse; benefit scales with result size. The JSON response stays byte-identical for clients that don't opt in.


curl -s localhost:8081/v1/query/execute -H 'content-type: application/json' \
    -H 'accept: application/vnd.apache.arrow.stream' \
    -d '{"metrics":["revenue"],"group_by":[{"field":"customers.region"}]}' \
    | python3 -c 'import pyarrow.ipc,sys; print(pyarrow.ipc.open_stream(sys.stdin.buffer).read_all())'
Semantics:

Pre-flight errors keep the JSON envelope: compile/profile/warehouse failures are detected before the response commits, so you still get the structured {"error": {code, message, hint}} with the proper HTTP status.
Mid-stream failures terminate the body (standard for streaming APIs) — the IPC stream ends without the end-of-stream marker; re-request.
--execute-timeout-secs bounds time-to-first-byte on this path, not total stream time; the concurrency permit is held until the stream finishes.
The CLI equivalent is dosi query --execute --format arrow (IPC on stdout).
Availability: on by default (server feature arrow); a Flight SQL server endpoint is future work — this REST body is the current columnar egress.
Errors¶
Error bodies wrap the engine's structured errors verbatim, with the same stable machine codes as dosi --format json:


{"error": {"code": "unknown_metric",
           "message": "unknown metric \"revenu\"",
           "candidates": ["revenue", "order_count", "..."]}}
HTTP	When
400	planner rejections (unknown_metric, ambiguous_dimension, no_join_path, ...), bad JSON, unknown dialect, profile config errors
401	missing/wrong bearer token (only when --auth-token is set)
403	/v1/query/execute with --disable-execute
429	all execution slots busy (Retry-After: 1)
501	planner not_implemented
502	warehouse unreachable / credentials rejected / SQL rejected (connection, auth, sql_rejected, driver)
504	warehouse timeout or the server's own execute timeout
Concurrency model¶
Compile/explain/list are pure CPU over the shared in-memory IR and run inline on the async workers — no locks, no I/O. A release build sustains thousands of compile requests per second at single-digit-millisecond p50.
Execute is blocking warehouse I/O: a semaphore (--max-concurrent-executions) bounds it, spawn_blocking keeps it off the async workers, and --execute-timeout-secs abandons the response. Known v1 limit: an abandoned call still runs to completion in the background, holding its permit until it finishes.
MySQL-family and Postgres profiles pool connections (--pool-size per profile); ClickHouse/Trino reuse HTTP keep-alive; DuckDB spawns a subprocess per call.
 Back to top
Next
MCP server
Copyright © Datus · Licensed under the Elastic License 2.0
