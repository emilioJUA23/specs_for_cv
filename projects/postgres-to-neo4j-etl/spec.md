## PostgreSQL to Neo4j Sync Pipeline — Northwind Dataset

**Target roles:** Data Engineering

**One-liner:** Scheduled incremental pipeline that continuously syncs operational PostgreSQL data into Neo4j to power graph analytics unavailable in the relational layer.

**Context:**
Operational databases (PostgreSQL) are optimized for transactional writes. Multi-hop analytical queries — "which suppliers provide products ordered by customers in a given region?", "who in the org chain processed the most high-value orders?" — require 4–5 joins in SQL and degrade with data growth. The goal is a continuously running pipeline that keeps a Neo4j graph replica in sync with PostgreSQL as new data arrives, enabling graph traversals on live operational data without touching the source DB for analytics.

The Northwind dataset (specialty foods trading company) is used as the operational source: 14 tables, ~3,362 rows across orders, customers, products, suppliers, employees, and territories. New orders are inserted into PostgreSQL over time; the pipeline detects and syncs them incrementally into Neo4j.

**What you did:**
- Designed a graph schema mapping 9 relational tables to node labels and dissolving join tables into typed relationships
- Built a **scheduled incremental pipeline**: on each run, detects new or changed records in PostgreSQL since the last successful sync (watermark-based on `orders.order_date` for fact data; count-based change detection for dimension tables)
- Persists pipeline state (last sync watermark) in a `pipeline_state` table in PostgreSQL — survives container restarts
- Handles load ordering: dimension nodes (Customer, Product, Supplier, Employee, etc.) are always loaded before fact edges (PLACED, CONTAINS, PROCESSED) to avoid ghost node creation
- Uses `MERGE` on all node and relationship creation — idempotent on rerun, safe against partial failures
- Handles self-referential FK (`employees.reports_to`) by loading all Employee nodes first, then creating `REPORTS_TO` edges in a second pass
- Dissolves M:M junction tables (`order_details` → `CONTAINS` edge with properties; `employee_territories` → `ASSIGNED_TO` edge)
- Scoped out two structurally present but empty tables (`customer_demographics`, `customer_customer_demo`) — decision documented, labels not created
- Built a fully containerized environment with Docker Compose: PostgreSQL (source + state), Neo4j (graph replica), Python worker (pipeline runner)
- Pipeline is observable: logs each run's start time, watermark, records processed, and any errors to stdout (structured JSON logs)

**Pipeline design:**

```
[PostgreSQL — operational source]
        ↓  (psycopg2, watermark query)
[Python worker — scheduled every N minutes]
    1. Read last_synced_at from pipeline_state table
    2. Extract new/changed records since watermark
    3. Load dimension nodes first (full refresh for small tables)
    4. Load fact nodes and edges (incremental, watermark-gated)
    5. Update pipeline_state on success
        ↓  (neo4j-python-driver, UNWIND + MERGE via Bolt)
[Neo4j — graph analytics replica]
```

**Incremental strategy by table type:**

| Table type | Strategy | Rationale |
|---|---|---|
| Dimension (customers, products, suppliers, etc.) | Count-based: full refresh if count differs | Small tables (<100 rows); no updated_at column |
| Fact (orders) | Watermark on `order_date` | Append-only; new orders have newer dates |
| Junction (order_details, employee_territories) | Follows parent fact table watermark | No independent timestamp |
| Self-referential (employees.reports_to) | Two-pass: nodes first, edges second | Prevents ghost node / dangling edge on first load |

**Graph Schema:**

Nodes (9 labels):
| Node Label | Source Table | Row Count |
|---|---|---|
| Customer | customers | 91 |
| Order | orders | 830 |
| Product | products | 77 |
| Supplier | suppliers | 29 |
| Employee | employees | 9 |
| Category | categories | 8 |
| Shipper | shippers | 6 |
| Territory | territories | 53 |
| Region | region | 4 |

Relationships (9 types):
| Relationship | From | To | Source |
|---|---|---|---|
| PLACED | Customer | Order | orders.customer_id |
| PROCESSED | Employee | Order | orders.employee_id |
| SHIPPED_BY | Order | Shipper | orders.shipper_id |
| CONTAINS | Order | Product | order_details (dissolved) |
| SUPPLIED_BY | Product | Supplier | products.supplier_id |
| BELONGS_TO | Product | Category | products.category_id |
| REPORTS_TO | Employee | Employee | employees.reports_to (self-FK) |
| ASSIGNED_TO | Employee | Territory | employee_territories (dissolved) |
| PART_OF | Territory | Region | territories.region_id |

Edge properties on `CONTAINS`: unit_price, quantity, discount

**Dev environment:**
- `docker-compose.yml`: three services on isolated Docker network — `postgres` (source + state store), `neo4j` (Community Edition 5.x), `worker` (Python pipeline runner)
- Worker runs on a schedule (configurable interval via env var); can also be triggered manually
- Neo4j ports: 7474 (Browser), 7687 (Bolt). PostgreSQL: 5432
- Data persisted via named Docker volumes; pipeline state survives container restarts
- New data simulated by seeding Northwind in two batches: initial load (orders before 1997-01-01), then incremental runs pick up remaining orders — demonstrates the pipeline detecting and syncing new records

**Impact:** [TO CONFIRM after build]
- Total nodes loaded: ~1,107 (sum of all node label counts)
- Total relationships loaded: ~3,354 (sum across all edge types — to confirm from Neo4j)
- Pipeline runs end-to-end in < X seconds on full initial load (measure and record)
- Incremental run (new orders only) completes in < Y seconds (measure and record)
- Graph query complexity: 4-join SQL → single Cypher pattern match (demonstrable, not latency-claimed)

**Keywords:** ETL, data pipeline, incremental load, watermark, change detection, PostgreSQL, Neo4j, graph database, data modeling, Cypher, Bolt protocol, relational-to-graph, schema design, graph schema, data transformation, Docker, Docker Compose, containerization, idempotency, MERGE, self-referential FK, M:M junction resolution, pipeline state, scheduled pipeline, structured logging

**CV bullet(s):**

*Primary (Data Engineering):*
- Built a scheduled incremental pipeline syncing PostgreSQL operational data into a Neo4j graph replica; designed 9-node, 9-relationship schema with watermark-based change detection, two-pass self-referential FK handling, and full idempotency via MERGE

*Variant — emphasizing graph modeling:*
- Designed and implemented a relational-to-graph sync pipeline (Postgres → Neo4j); dissolved M:M junction tables into typed edges, handled employee org hierarchy via two-pass load, enabling multi-hop graph traversals unavailable in the SQL layer

---

## Definition of Done

- [ ] `implementation.md` written
- [ ] Pipeline built: initial load + at least 2 incremental runs verified
- [ ] pipeline_state table persists watermark across container restarts (verified)
- [ ] Idempotency verified: two consecutive runs produce identical Neo4j state
- [ ] Ghost node prevention verified: no nodes with only an ID property after full load
- [ ] Node count and relationship count confirmed from Neo4j
- [ ] Structured logs captured for a sample run
- [ ] Impact metrics filled in (full load time, incremental run time)
- [ ] CV bullet finalized
- [ ] `implementation.md` exported to build repo
