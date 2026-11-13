# 04b · Scenario-Based Interview Questions (with model answers)

> Tip: Structure every answer as **Requirement → Fabric items chosen → Why → Trade-offs/monitoring.**

---

### S1. Design an end-to-end solution: daily sales CSVs from SFTP + an Azure SQL ERP + a Power BI report for executives.
- **Ingest:** Pipeline Copy activity (SFTP → Bronze lakehouse `/Files`); **Mirroring** for Azure SQL ERP (near-real-time, no ETL).
- **Transform:** Notebooks (PySpark) Bronze → Silver (clean, dedupe) → Gold (star schema) using Delta `MERGE`.
- **Serve:** Gold lakehouse/warehouse → **Direct Lake** semantic model → Power BI report/app.
- **Orchestrate:** One parent pipeline with schedule, failure alerts (Teams/Outlook activity).
- **Govern:** Workspaces per layer/environment, Git + deployment pipelines, RLS in semantic model, certified dataset.

### S2. Your Power BI report on Direct Lake is suddenly slow. What do you check?
1. Has it **fallen back to DirectQuery**? (Performance analyzer / DAX query view / trace) — causes: guardrails exceeded, SQL views, SQL-endpoint RLS.
2. **Small files / too many row groups** → run `OPTIMIZE ... VORDER`.
3. V-Order disabled on write?
4. Capacity **throttling** (Capacity Metrics app).
5. Cold cache after reframing; model design (high-cardinality columns, bi-directional relationships, complex DAX).

### S3. Incremental load of a 500 M-row table from on-prem SQL Server every hour.
- On-premises data gateway → pipeline Copy activity or **Copy job** in incremental mode.
- **Watermark** column (`ModifiedDate`/rowversion) stored in a control table; Lookup old watermark → copy `WHERE ModifiedDate > @old` → update watermark.
- Land into Bronze, then `MERGE` into Silver Delta table.
- If source has CDC enabled, use CDC to capture deletes too.
- Partition/parallel copy for initial full load.

### S4. Implement SCD Type 2 for a Customer dimension in Fabric.
- Silver staging table with latest records; Gold `dim_customer` with `effective_from`, `effective_to`, `is_current`, surrogate key.
- Delta `MERGE`: when matched AND attributes changed → update old row `is_current=false, effective_to=now()`; insert new version (union trick or two-step merge). When not matched → insert.
- In Warehouse, the same with T-SQL `MERGE` or UPDATE + INSERT inside a transaction.

### S5. Data lives in ADLS Gen2 in another subscription and also in Amazon S3. The business doesn't want duplicates.
- Create **shortcuts** (ADLS Gen2 & S3) in a lakehouse. Use **workspace identity/trusted workspace access** for firewall-protected ADLS. Optionally enable **shortcut caching** for S3 to reduce egress cost.
- Query via Spark/SQL endpoint; build Direct Lake model if data is Delta.

### S6. Capacity is being throttled every morning at 9 AM.
- Open **Capacity Metrics app** → find top items by CU (often heavy Dataflow Gen2 refresh, Spark jobs, or interactive report queries).
- Move heavy background jobs to off-peak, optimise them (notebooks instead of Dataflows, OPTIMIZE tables, incremental instead of full loads).
- Split workloads across capacities (prod vs dev), scale SKU up temporarily, or use surge protection settings.

### S7. Different regions should see only their own data in reports, and analysts querying the lakehouse SQL endpoint should be restricted too.
- Use **OneLake security** roles with row-level filters so restrictions apply across Spark, SQL and Direct Lake, or
- RLS in the semantic model (`[Region] = USERPRINCIPALNAME()`-mapped table) + RLS on SQL endpoint/warehouse.
- Give analysts Viewer/item-level ReadData, not workspace Contributor.

### S8. Migrate an Azure Synapse dedicated SQL pool (5 TB) to Fabric.
- Assess objects (Migration Assistant / DACPAC), list unsupported T-SQL features.
- Export data with **CETAS → Parquet** in ADLS → shortcut or `COPY INTO` Fabric Warehouse.
- Remove distribution/index syntax, fix code, migrate stored procs.
- Re-point pipelines and Power BI (Direct Lake), validate row counts/aggregates, run in parallel, cut over.

### S9. IoT devices send 50k events/sec; ops wants a live dashboard and an alert when temperature > 80 °C.
- Event Hubs/IoT Hub → **Eventstream** → **Eventhouse (KQL DB)**.
- **Real-Time Dashboard** with KQL tiles (auto-refresh).
- **Activator** rule on the stream: temperature > 80 for 2 min → Teams/email + trigger pipeline.
- Enable OneLake availability to join history with lakehouse data for batch analytics.

### S10. A notebook that ran in 10 minutes now takes 1 hour.
- Spark UI / monitoring hub: skew, spill, shuffle partitions, too many small files.
- Run `OPTIMIZE`, check data growth, use partition pruning/predicate pushdown, broadcast small joins, cache wisely, enable **Native Execution Engine**, choose a larger custom pool or autoscale.

### S11. Set up CI/CD for Dev, Test and Prod.
- Three workspaces; Dev workspace connected to Git (feature branches → PR → main).
- **Deployment pipelines** (or `fabric-cicd` in Azure DevOps/GitHub Actions) to promote to Test/Prod.
- **Deployment rules / variable libraries** to switch connections & lakehouse IDs per stage. Service principal for automation.

### S12. Business users want to clean an Excel file weekly without writing code.
- **Dataflow Gen2** (Power Query) with output destination = Lakehouse table, scheduled refresh; or SharePoint source. Keep it in a separate workspace to monitor CU usage.

### S13. Pipeline fails intermittently on a REST API source because of rate limits.
- Retry policy & timeout on activity, `Until` loop with `Wait`, pagination rules, smaller batch sizes, ForEach with limited concurrency (batch count), log failures to a control table, alert on final failure.

### S14. Data scientists need features from the gold layer and want to track experiments.
- Notebooks reading Gold Delta tables (or Semantic Link to read Power BI measures).
- MLflow experiments + model registry in Fabric; batch scoring with `PREDICT` writing back to a lakehouse table → Power BI.

### S15. Should this team choose Lakehouse or Warehouse?
- Team mostly SQL developers, needs stored procedures, multi-table transactions → **Warehouse**.
- Team is Python/Spark, semi-structured JSON, ML → **Lakehouse**.
- Common pattern: Lakehouse for Bronze/Silver, Warehouse for Gold — both share OneLake so no duplication.

### S16. How do you delete a customer's data for a GDPR request in Delta tables?
- `DELETE FROM table WHERE customer_id = ...` in every layer, then `VACUUM` (after retention) so old Parquet files with the data are physically removed; also clear any shortcuts' sources and semantic model caches.

### S17. A shortcut to ADLS Gen2 stops working after the storage firewall was enabled.
- Enable **trusted workspace access**: create a workspace identity, add resource instance rule on the storage account, grant the identity Storage Blob Data Reader, recreate connection using workspace identity.

### S18. Report numbers don't match the source system.
- Check pipeline run history and row counts per layer, watermark logic (late-arriving data, timezone), duplicates in MERGE keys, Direct Lake framing (old version?), filters/RLS in semantic model, DAX measure logic.

### S19. Leadership asks: "Why move to Fabric instead of staying on Synapse + Power BI?"
- One copy of data (OneLake + Direct Lake), less admin (SaaS), single capacity billing, unified governance, Real-Time Intelligence & Copilot. Caveats: migration effort, capacity planning, some features still maturing — recommend phased adoption starting with shortcuts.

### S20. You need sub-second lookup of a single record for an app plus analytics on the same data.
- Use **SQL database in Fabric** (OLTP) for the app; it automatically mirrors to OneLake, so analytics/Power BI use the Delta copy without impacting the OLTP workload.
