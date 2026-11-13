# 02 · Important Topics (Priority List)

Study in this order. ⭐ = asked in almost every interview.

## A. Must-know (⭐⭐⭐)
1. **OneLake** – one lake per tenant, ADLS Gen2 underneath, Delta format, "one copy" principle.
2. **Lakehouse vs Warehouse** – when to use which (see table below).
3. **SQL analytics endpoint** – auto-created, *read-only* T-SQL over lakehouse Delta tables.
4. **Shortcuts vs Mirroring vs Copy** – zero-copy reference vs replicated CDC vs physical copy.
5. **Direct Lake mode** – how it works, framing, fallback to DirectQuery, guardrails.
6. **Delta Lake** – ACID, `_delta_log`, time travel, schema evolution, OPTIMIZE, VACUUM, V-Order.
7. **Medallion architecture** (Bronze/Silver/Gold) in Fabric.
8. **Capacity & CUs** – F-SKUs, bursting, smoothing, throttling, Capacity Metrics app.
9. **Workspace roles & security** – Admin/Member/Contributor/Viewer, item sharing, OneLake security, RLS/CLS.
10. **Pipelines vs Dataflow Gen2 vs Notebooks vs Copy job** – choosing the right ingestion/transform tool.

## B. Very important (⭐⭐)
11. Incremental loading patterns – watermark column, CDC, `MERGE`, Delta `MERGE INTO`.
12. Slowly Changing Dimensions (SCD Type 1 / Type 2) in Spark and T-SQL.
13. Spark in Fabric – starter vs custom pools, environments, high-concurrency, Native Execution Engine, `notebookutils`.
14. Real-Time Intelligence – Eventstream → Eventhouse (KQL DB) → Real-Time Dashboard → Activator.
15. Basic **KQL** – `where`, `summarize`, `bin()`, `project`, `extend`, `join`, `render`.
16. Git integration & deployment pipelines (CI/CD).
17. Monitoring – Monitoring hub, capacity metrics, Spark UI, query insights (`queryinsights` views in Warehouse).
18. Semantic models – star schema, relationships, DAX basics, RLS.
19. Fabric vs Azure Synapse vs Databricks (positioning questions).
20. Data governance – Purview, sensitivity labels, lineage, endorsement, domains.

## C. Good to know (⭐)
21. SQL database in Fabric (OLTP + auto-mirroring).
22. Data agents & Copilot in Fabric.
23. Semantic Link (`sempy`) – read Power BI models from notebooks.
24. Network security – Private Links, managed VNets, managed private endpoints, trusted workspace access, workspace identity.
25. Fabric REST APIs, Fabric CLI, `fabric-cicd`, Terraform.
26. Variable libraries, notebook parameters, pipeline parameters/expressions.
27. Data Science – MLflow, `PREDICT`, AutoML.
28. Licensing – F64 threshold for free viewers, Pro/PPU, trial.

---

## Key comparison tables

### Lakehouse vs Warehouse
| | **Lakehouse** | **Warehouse** |
|---|---|---|
| Primary language | PySpark / Spark SQL (T-SQL read-only via SQL endpoint) | T-SQL (full DDL + DML) |
| Data types | Structured, semi-structured, unstructured (`/Files`) | Structured only |
| Write via T-SQL | ❌ (read-only endpoint) | ✅ |
| Multi-table transactions | ❌ | ✅ |
| Best for | Data engineers, big data, ML, raw/silver layers | SQL developers, BI, gold layer, stored procedures |
| Storage | Delta in OneLake | Delta in OneLake |

### Shortcut vs Mirroring vs Copy
| | Shortcut | Mirroring | Copy (pipeline/copy job) |
|---|---|---|---|
| Data moved? | No (virtual) | Yes, replicated continuously | Yes, on schedule |
| Latency | Live (source) | Near real-time | Batch |
| Transformation | None | None | Optional |
| Use when | Data already in Delta/Parquet elsewhere | Operational DB you want in OneLake | Need transformation or unsupported source |

### Pipeline vs Dataflow Gen2 vs Notebook
| | Pipeline | Dataflow Gen2 | Notebook |
|---|---|---|---|
| Style | Orchestration, low-code | Low-code Power Query | Code (PySpark/SQL) |
| Best for | Copying large volumes, control flow, scheduling | Business-user transforms, many connectors | Complex/large-scale transforms, ML |
| Scale | High (copy) | Small–medium | Very high |

### Power BI storage modes
| Mode | Data location | Freshness | Speed |
|---|---|---|---|
| Import | Copied into model | Needs refresh | Fastest |
| DirectQuery | Stays in source | Real-time | Slower |
| **Direct Lake** | Delta files in OneLake loaded on demand | Near real-time (reframing) | Near Import |

### Capacity throttling stages
| Overage (future smoothed usage) | Effect |
|---|---|
| ≤ 10 min | Overage protection – no throttling |
| 10 – 60 min | Interactive requests delayed 20 s |
| 60 min – 24 h | Interactive requests rejected |
| > 24 h | Background jobs rejected too |

---

## Code snippets worth memorising

**PySpark – write Delta table to Lakehouse**
```python
df = spark.read.option("header", True).csv("Files/raw/sales.csv")
df.write.mode("overwrite").format("delta").saveAsTable("silver_sales")
```

**Delta MERGE (upsert / SCD1)**
```sql
MERGE INTO silver_customer t
USING staging_customer s
ON t.customer_id = s.customer_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

**Maintenance**
```sql
OPTIMIZE silver_sales VORDER;
VACUUM silver_sales RETAIN 168 HOURS;
DESCRIBE HISTORY silver_sales;
SELECT * FROM silver_sales VERSION AS OF 3;  -- time travel
```

**KQL basics**
```kql
Telemetry
| where Timestamp > ago(1h)
| summarize avg(Temperature) by DeviceId, bin(Timestamp, 5m)
| render timechart
```

**notebookutils**
```python
notebookutils.fs.ls("Files/")
notebookutils.notebook.run("child_notebook", 600, {"run_date": "2026-11-13"})
notebookutils.credentials.getSecret("https://<vault>.vault.azure.net/", "secret-name")
```
