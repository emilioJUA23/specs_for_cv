## PostgreSQL to Neo4j Sync Pipeline — Implementation Plan

**Architecture overview:**

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose Network                  │
│                                                             │
│  ┌──────────────┐    psycopg2     ┌────────────────────┐   │
│  │  PostgreSQL  │ ◄────────────── │   Python Worker    │   │
│  │  (source +   │                 │   (pipeline.py)    │   │
│  │  state store)│ ────────────► │   - scheduler      │   │
│  └──────────────┘  pipeline_state │   - extractor      │   │
│                                   │   - transformer    │   │
│                                   │   - loader         │   │
│                                   └────────┬───────────┘   │
│                                            │ Bolt           │
│                                            │ (neo4j-driver) │
│                                   ┌────────▼───────────┐   │
│                                   │      Neo4j 5.x     │   │
│                                   │  (graph replica)   │   │
│                                   └────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Tech stack:**

| Layer | Choice | Reason |
|---|---|---|
| Source DB | PostgreSQL 16 (official Docker image) | Target of the pipeline; PG16 chosen for COPY improvements (not used here but version-relevant) |
| Graph DB | Neo4j 5.x Community Edition | Current stable; free tier sufficient for Northwind scale |
| Pipeline runtime | Python 3.11 | Standard DE tooling; driver support is first-class in Neo4j and psycopg2 |
| PG driver | psycopg2-binary | Battle-tested; supports server-side cursors for large result sets |
| Neo4j driver | neo4j 5.x (official Python driver) | `execute_query()` API handles retry + connection pooling |
| Scheduling | APScheduler (in-process) | Zero-infra scheduler; sufficient for portfolio scope; swap for Airflow/Prefect in production |
| Logging | Python `logging` + JSON formatter | Structured logs; each run emits start, watermark, records_processed, duration, errors |
| Containerization | Docker Compose v2 | Isolated network; named volumes for state persistence |

**Components:**

- `docker-compose.yml` — orchestrates postgres, neo4j, worker services on shared bridge network
- `init/northwind.sql` — Northwind DDL + seed data; mounted into postgres container as init script
- `init/seed_split.sql` — inserts only orders with `order_date < '1997-01-01'` on first seed; remaining orders available for incremental demo
- `pipeline/pipeline.py` — main entrypoint; runs scheduler loop
- `pipeline/extractor.py` — all PostgreSQL read logic; watermark queries, dimension queries, state reads
- `pipeline/transformer.py` — row-to-node/edge mapping; pure functions, no I/O
- `pipeline/loader.py` — all Neo4j write logic; UNWIND batching, constraint setup, MERGE execution
- `pipeline/state.py` — reads and writes `pipeline_state` table in PostgreSQL
- `pipeline/models.py` — dataclasses for Node and Edge typed structures passed between transformer and loader
- `requirements.txt` — pinned dependencies

**Build steps:**

1. **Repo scaffold**
   - Create directory structure: `init/`, `pipeline/`, `tests/`
   - Write `docker-compose.yml` with three services (postgres, neo4j, worker) on `etl_net` bridge network
   - Configure named volumes: `pg_data`, `neo4j_data`
   - Worker depends_on postgres and neo4j with health checks

2. **PostgreSQL init**
   - Mount `init/northwind.sql` into `/docker-entrypoint-initdb.d/` — runs automatically on first start
   - Mount `init/pipeline_state.sql` — creates the state table:
     ```sql
     CREATE TABLE IF NOT EXISTS pipeline_state (
         key   VARCHAR(64) PRIMARY KEY,
         value TEXT NOT NULL,
         updated_at TIMESTAMPTZ DEFAULT now()
     );
     INSERT INTO pipeline_state (key, value)
     VALUES ('orders_last_synced_at', '1996-01-01 00:00:00')
     ON CONFLICT DO NOTHING;
     ```
   - Initial watermark set to `1996-01-01` so first run picks up the pre-1997 seed batch

3. **Neo4j constraints (run once on startup)**
   - Loader creates uniqueness constraints before any data is written:
     ```cypher
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Customer) REQUIRE n.customer_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Order) REQUIRE n.order_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Product) REQUIRE n.product_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Supplier) REQUIRE n.supplier_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Employee) REQUIRE n.employee_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Category) REQUIRE n.category_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Shipper) REQUIRE n.shipper_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Territory) REQUIRE n.territory_id IS UNIQUE;
     CREATE CONSTRAINT IF NOT EXISTS FOR (n:Region) REQUIRE n.region_id IS UNIQUE;
     ```
   - These are mandatory before any MERGE — without them, MERGE does a full label scan per row

4. **Extractor: dimension tables (full refresh with count guard)**
   - On each pipeline run, query PostgreSQL for current row count of each dimension table
   - Compare against count stored in `pipeline_state`; if different, extract full table
   - Tables: customers, products, suppliers, employees, categories, shippers, territories, regions
   - Use server-side cursor for safety (though all dimension tables are <100 rows)

5. **Extractor: fact table (watermark-based incremental)**
   - Query orders WHERE `order_date > last_synced_at` ORDER BY `order_date ASC`
   - For each new order, also extract its `order_details` rows (JOIN on order_id)
   - Extract `employee_territories` rows for any employee_ids present in new orders (or full refresh — small table)
   - Watermark advances to `MAX(order_date)` of processed batch only on successful Neo4j write

6. **Transformer: row → node/edge mapping**
   - Pure functions; no I/O; fully testable in isolation
   - Each table has a dedicated mapper: `map_customer_row(row) -> Node`, etc.
   - Junction table dissolution: `map_order_detail_row(row) -> Edge(Order, CONTAINS, Product, props={unit_price, quantity, discount})`
   - Self-referential: `map_employee_row(row) -> (Node, Optional[Edge])` — Edge only if `reports_to IS NOT NULL`

7. **Loader: dimension nodes (UNWIND + MERGE)**
   - Batch all dimension rows by label type; send each label in one UNWIND call
   - Example pattern (same structure for all dimension nodes):
     ```cypher
     UNWIND $rows AS row
     MERGE (n:Customer {customer_id: row.customer_id})
     SET n += row
     ```
   - Batch size: 500 rows (well within Neo4j sweet spot; all dimension tables are under 100 rows anyway)

8. **Loader: fact nodes and edges (two-pass, order matters)**
   - Pass 1 — Order nodes: UNWIND + MERGE on order_id
   - Pass 2 — Edges in dependency order:
     1. `PLACED` (Customer → Order) — both endpoints guaranteed present
     2. `PROCESSED` (Employee → Order) — Employee loaded in dimension pass
     3. `SHIPPED_BY` (Order → Shipper)
     4. `CONTAINS` (Order → Product, with properties) — Product loaded in dimension pass
     5. `ASSIGNED_TO` (Employee → Territory)
     6. `REPORTS_TO` (Employee → Employee) — both Employee nodes guaranteed present from dimension pass; safe to create in same run

9. **State update**
   - On successful completion of a pipeline run, update `pipeline_state`:
     - `orders_last_synced_at` → MAX(order_date) of batch just processed
     - `customers_last_count`, `products_last_count`, etc. → current counts for dimension change detection
   - State update is the last write; if pipeline fails before this point, next run reprocesses the same batch (idempotent via MERGE)

10. **Scheduling**
    - APScheduler `BlockingScheduler` with `IntervalTrigger`; interval configurable via `PIPELINE_INTERVAL_SECONDS` env var (default: 60)
    - On startup: run one immediate pipeline execution, then schedule
    - Scheduler runs in the foreground; Docker restarts the container on exit

11. **Incremental demo**
    - After initial load (orders pre-1997) is confirmed in Neo4j, run:
      ```sql
      INSERT INTO orders SELECT * FROM northwind_remaining_orders;
      ```
      (or equivalent — pre-staged in a second SQL file)
    - Next scheduled pipeline run detects new orders, syncs them
    - Verify in Neo4j: node count increases, new PLACED/CONTAINS/PROCESSED edges appear

12. **Structured logging**
    - Each run emits a JSON log line at start and end:
      ```json
      {"event": "run_start", "watermark": "1996-07-04T00:00:00", "ts": "..."}
      {"event": "run_complete", "nodes_written": 91, "edges_written": 830, "duration_ms": 412, "new_watermark": "1996-12-31T00:00:00", "ts": "..."}
      ```
    - Errors logged with `{"event": "run_error", "error": "...", "ts": "..."}`

13. **Verification queries (run in Neo4j Browser after full load)**
    ```cypher
    // Node counts
    MATCH (n) RETURN labels(n)[0] AS label, count(n) AS count ORDER BY count DESC;

    // Relationship counts
    MATCH ()-[r]->() RETURN type(r) AS rel, count(r) AS count ORDER BY count DESC;

    // Multi-hop: suppliers for German customers
    MATCH (c:Customer {country: 'Germany'})-[:PLACED]->(:Order)-[:CONTAINS]->(:Product)<-[:SUPPLIED_BY]-(s:Supplier)
    RETURN DISTINCT s.company_name;

    // Org hierarchy
    MATCH (boss:Employee {employee_id: 2})<-[:REPORTS_TO*]-(report:Employee)
    RETURN report.first_name, report.last_name;
    ```

**Open questions / decisions deferred:**

- **Scheduler swap for production framing:** APScheduler is correct for portfolio scope. If the implementation.md is later used to pitch this as a production-ready design, document the swap to Airflow/Prefect as a natural next step — not a gap.
- **Dimension change detection granularity:** Count-based change detection will miss in-place updates (e.g., a customer's phone number changes but count stays the same). For Northwind (static dataset) this is fine. For a real operational source, you'd need `updated_at` columns or CDC. Flag this as a known limitation in the README.
- **Neo4j Community Edition limits:** Community Edition has no role-based access control and no clustering. Fine for portfolio. Note it in the README.
- **orders.order_date vs. a dedicated updated_at:** Northwind has no `updated_at` on orders. `order_date` is used as the watermark proxy. This means backdated order inserts (order_date in the past) would be missed. Document this assumption explicitly — it's a real interview question.
- **Worker startup race:** postgres and neo4j may not be fully ready when the worker starts. Health checks in docker-compose handle this, but implement a retry loop with exponential backoff in the worker's connection setup regardless.
