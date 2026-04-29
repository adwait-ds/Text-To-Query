# Marketing AI — Project Context

## Project goal
Build a flexible text-to-SQL system for marketing analytics using a fine-tuned LLaMA model.
- User asks a natural language question
- LLaMA (trained on schema + NL→SQL pairs) generates SQL
- SQL executes on the real database
- Results render as visualization (chart or table)
- LLaMA never sees raw data — only the schema is exposed at inference time
- System is flexible — works with any client schema, not just the reference 7-table schema

---

## Finalized decisions

### Model
- Base model: LLaMA 3.1 8B
- Fine-tuning method: QLoRA (4-bit quantization, LoRA adapters)
- Library: Unsloth + HuggingFace
- Training platform: Google Colab (free T4 GPU)
- Experiment tracking: Weights & Biases (free tier)

### Data sources
- BigQuery (production)
- CSV/Excel via DuckDB (lightweight/local)
- MySQL — skipped for now, add later
- Multi-channel attribution — skipped for now, add later

### SQL dialect handling
- BigQuery SQL for BigQuery source
- DuckDB SQL for CSV/Excel source
- Dialect passed as part of prompt context at inference time
- Prompt format: Schema + Dialect + Question → SQL

### System flexibility
- Flexible from day one — works with any client schema
- Reference 7-table schema used for training and validation
- Schema stored as JSON config file (auto-generated from client data source)
- No code changes needed when client has different tables or columns
- Config structure: tables → columns → relationships → common_filters

### Domain scope
- B2C ecom marketing only (v1)
- 4 domains in scope:
  1. Ecom funnel (behavioral events)
  2. Campaign performance
  3. Customer / user data
  4. Orders & revenue

### Conversation memory
- Sliding window — last 5 exchanges kept in prompt
- Session stored in Redis (24hr expiry)
- Cross-session memory not in v1, add later

### Caching layers
- Schema cache: loaded once at startup, lives in memory
- Result cache: Redis with TTL (5min real-time / 1hr daily / 24hr historical)
- BigQuery query cache: automatic, handled by BigQuery natively

### Inference pipeline
User question
  → check result cache (Redis)
  → embed question
  → RAG finds relevant tables (2-3 tables)
  → inject schema context into prompt
  → inject last 5 conversation exchanges
  → LLaMA generates SQL
  → validate SQL syntax
  → execute on BigQuery or DuckDB
  → detect result shape
  → render chart or table
  → cache result

### Visualization — auto-detection rules
- Time series data     → Line chart
- Comparisons         → Bar chart
- Part of whole       → Pie or donut chart
- Funnel steps        → Funnel chart
- Rankings            → Horizontal bar chart
- Two metrics         → Dual axis chart
- Raw data            → Sortable table
- Single number       → Metric card with trend indicator
- User can override chart type

### Deployment (planned)
- API: FastAPI
- Container: Docker
- Cloud: GCP Cloud Run (pairs well with BigQuery)
- Cost optimization: spot instances, quantization, auto-scaling

---

## 6-week plan

| Week | Focus | Deliverable | Status |
|---|---|---|---|
| 1 | Schema + data foundation | 7 tables designed, DDL written, synthetic data generated | In progress |
| 2 | Training data creation | 300-500 NL→SQL pairs, schema-aware prompt template | Not started |
| 3 | Fine-tuning | QLoRA fine-tune LLaMA 3.1 8B on Colab, W&B tracking | Not started |
| 4 | Inference pipeline | Schema injection, SQL generation, validation, execution, caching | Not started |
| 5 | Visualization layer | Result shape detection, auto chart selection, frontend | Not started |
| 6 | Deployment + polish | FastAPI, Docker, GCP Cloud Run, cost optimization | Not started |

### Risk flags
- Week 2 is the most important — poor NL→SQL pairs = poor model in Week 3
- Week 3 and 4 are the hardest — budget extra time here
- Fine-tuning runs take time even on Colab — use weekends

---

## Current status

- Week: 1 (in progress)
- Completed: schema design, ERD, DDL queries (BigQuery + DuckDB), synthetic data script
- Next: run synthetic data script locally, verify data, then move to Week 2

---

## Schema (finalized)

### Table creation order (respect dependencies)
1. products
2. campaigns
3. customers
4. campaign_daily_stats
5. orders
6. order_items
7. events

### Table summaries

**products** — product catalog, no foreign keys
- Key columns: product_id (PK), category, subcategory, brand, base_price, current_price, cost_price, is_active
- No partition (small table)

**campaigns** — campaign metadata, no foreign keys
- Key columns: campaign_id (PK), campaign_name, campaign_type, channel, platform, budget, status, utm_source, utm_medium
- No partition (small table)

**customers** — user profiles and acquisition data
- Key columns: user_id (PK), acquisition_channel, acquisition_campaign_id (FK→campaigns), customer_segment, total_orders, total_revenue, first_purchase_at
- No partition, cluster by user_id + customer_segment

**campaign_daily_stats** — daily performance per campaign
- Key columns: stat_id (PK), campaign_id (FK→campaigns), stat_date, impressions, clicks, spend, revenue, conversions
- PARTITION BY stat_date, CLUSTER BY campaign_id
- Derived metrics (calculated at query time): CTR, CPC, CPM, CPA, ROAS, ROI, CVR
- Always use NULLIF(x, 0) for division to avoid division by zero

**orders** — transaction headers
- Key columns: order_id (PK), user_id (FK→customers), campaign_id (FK→campaigns), order_date, status, total_amount, discount_amount, final_amount, payment_status, is_first_order
- PARTITION BY DATE(order_date), CLUSTER BY user_id + status
- Always filter: payment_status = 'paid' AND status NOT IN ('cancelled','refunded')
- Always use final_amount for revenue (not total_amount)

**order_items** — product lines within orders
- Key columns: item_id (PK), order_id (FK→orders), product_id (FK→products), quantity, unit_price (snapshotted at purchase time), final_price
- PARTITION BY DATE(created_at), CLUSTER BY order_id + product_id

**events** — behavioral event stream
- Key columns: event_id (PK), event_type, event_timestamp, user_id (FK→customers), session_id, campaign_id (FK→campaigns), product_id (FK→products), order_id (FK→orders), event_properties (JSON)
- PARTITION BY DATE(event_timestamp), CLUSTER BY event_type + user_id
- event_type values: page_view, product_view, add_to_cart, checkout_start, checkout_complete, purchase, refund (extensible)
- product_id and order_id are nullable — only populated for relevant event types

### Relationships
```
customers.user_id               ← events.user_id
customers.user_id               ← orders.user_id
customers.acquisition_campaign_id → campaigns.campaign_id
campaigns.campaign_id           ← campaign_daily_stats.campaign_id
campaigns.campaign_id           ← events.campaign_id
campaigns.campaign_id           ← orders.campaign_id
products.product_id             ← events.product_id
products.product_id             ← order_items.product_id
orders.order_id                 ← order_items.order_id
orders.order_id                 ← events.order_id
```

### BigQuery vs DuckDB type mapping
| BigQuery | DuckDB |
|---|---|
| STRING | VARCHAR |
| FLOAT64 | DOUBLE |
| INT64 | INTEGER |
| BOOL | BOOLEAN |
| PARTITION BY / CLUSTER BY | Not used in DuckDB |

### Important SQL patterns the model must learn
- NULLIF(x, 0) for all division operations
- final_amount not total_amount for revenue
- COUNT(DISTINCT user_id) not COUNT(*) for funnel steps
- LEFT JOIN not INNER JOIN when nullable FKs involved
- DATE_TRUNC(CURRENT_DATE(), MONTH) for BigQuery month filters
- DATE_TRUNC('month', col) for DuckDB month filters
- CASE WHEN COUNT DISTINCT pattern for funnel queries
- LAG() window function for day-over-day trend queries

---

## Synthetic data

### Volume targets
| Table | Rows |
|---|---|
| products | 50 |
| campaigns | 30 |
| customers | 10,000 |
| campaign_daily_stats | ~3,600 |
| orders | ~8,000 |
| order_items | ~20,000 |
| events | ~150,000 |

### Realistic patterns built into synthetic data
- Funnel drop-off: only ~3% of page_views become purchases
- Campaign quality variance: poor (0.8x ROAS) to excellent (9x ROAS)
- 80% of customers have made at least one purchase
- 65% mobile / 30% desktop / 5% tablet split
- 30% of products on sale (current_price < base_price)
- ~30% of traffic is organic (no campaign_id)
- is_first_order correctly flagged on first purchase per customer
- unit_price snapshotted at time of purchase (not current_price)

### Dependencies
```bash
pip install faker pandas numpy google-cloud-bigquery duckdb
```

### Output files
- marketing.duckdb — local DuckDB database with all 7 tables
- BigQuery dataset: your_project.marketing (all 7 tables)

---

## Training data (Week 2 — not started)

### Target distribution
- 1 table examples  → 20%
- 2 table examples  → 40%
- 3 table examples  → 30%
- 4+ table examples → 10%

### Query categories to cover
- Simple lookups and aggregations
- Date range filters (day, week, month, quarter, year)
- Funnel analysis (CASE WHEN COUNT DISTINCT pattern)
- Campaign performance (ROAS, ROI, CTR, CPA, CPC)
- Customer segmentation
- Revenue analysis
- Product performance
- Trend analysis (LAG, DATE_TRUNC)
- Data quality checks (IS NULL, COUNT vs COUNT(col))
- Partial schema examples (1-2 table subsets)

### Training pair format
```
### Schema:
Table: [table_name]
Columns: [col1 (TYPE), col2 (TYPE FK→table.col), ...]

### Relationships:
[table1.col] → [table2.col]

### Dialect: BigQuery | DuckDB

### Question:
[natural language question]

### SQL:
[correct SQL answer]
```

- Total pairs target: 300-500
- Synthetic expansion: use Claude API to generate from seed examples
- Total pairs: 0 (not started)

---

## Open questions

- Frontend framework for visualization layer (Week 5) — React vs plain HTML
- Schema onboarding UX — how does client upload/connect their schema?
- Error retry strategy — how many times to retry if LLaMA generates wrong SQL?
- Evaluation metric for SQL correctness (Week 3) — exact match vs execution match

---

## How to use this file

- Start every new session by pasting this file at the top of your first message
- End every session: "Update my project_context.md based on what we decided today"
- Keep stored in GitHub for version control
- Never paste full code into this file — decisions, schema, and status only
