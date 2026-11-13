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

---

## Shortcuts — scenario deep-dive

### S21. What is a OneLake shortcut, and why would you use one instead of copying data?
- A shortcut is a **pointer (symbolic link) inside OneLake** to data stored somewhere else. It looks and behaves like a normal folder/table, but **no data is moved or duplicated**.
- **Why:** one copy of data (no ETL to keep in sync, no extra storage), data is always current, faster onboarding of existing data lakes, and every Fabric engine (Spark, SQL endpoint, Direct Lake, KQL) can read it through the same OneLake path.
- **Trade-offs:** performance and egress cost depend on the source location; you don't control the source's schema, retention or availability.

### S22. Which types of shortcuts exist?
- **Internal (OneLake) shortcuts:** point to another Fabric item — Lakehouse, Warehouse, KQL database, mirrored database, etc. — in the same or a different workspace.
- **External shortcuts:** Azure Data Lake Storage Gen2, Azure Blob Storage, Amazon S3, S3-compatible storage (e.g., MinIO, Cloudflare R2), Google Cloud Storage, Dataverse.
- **On-premises / network-restricted sources:** S3-compatible, GCS or other storage behind a firewall reached via the **on-premises data gateway**.
- Newer options (check current preview status): SharePoint/OneDrive shortcuts, Iceberg tables virtualised as Delta, and shortcut transformations (e.g., auto-convert CSV/Parquet/JSON in a shortcut to a Delta table).

### S23. Where can you create a shortcut, and what's the difference between Tables and Files?
- **Lakehouse** (most common) and **KQL database**. A Warehouse can't host shortcuts itself, but it can query lakehouse tables (including shortcut tables) through cross-database queries.
- **Tables section:** the shortcut must sit at the top level and point to a **Delta** folder → it is auto-discovered as a table and is usable by the SQL endpoint and Direct Lake.
- **Files section:** any format (CSV, Parquet, JSON, images), can be in subfolders → read with Spark (`spark.read...`), not exposed as SQL tables.
- Interview line: "Delta → Tables shortcut for BI; raw/non-Delta → Files shortcut, then transform with a notebook."

### S24. How do security and permissions work with shortcuts?
- **Internal shortcuts:** the **caller's identity is passed through** — the user needs permission on the target item, otherwise access is denied even if they can see the shortcut.
- **External shortcuts:** access uses the **credentials stored in the cloud connection** (account key, SAS, service principal, organisational account, workspace identity). Anyone with access to the shortcut reads with those credentials, so scope the connection identity with least privilege (e.g., Storage Blob Data Reader).
- Firewall-protected ADLS Gen2 → **trusted workspace access** with workspace identity (see S17).
- Layer **OneLake security roles** on top to restrict folders/rows/columns for consumers.

### S25. A user deletes a shortcut — is the source data deleted? What if they delete a file *inside* the shortcut?
- Deleting the **shortcut** only removes the pointer; source data is untouched.
- Deleting **files/folders inside** the shortcut deletes them **in the source** (if the identity has write/delete rights). Mitigate with read-only credentials on the connection and limited workspace roles.

### S26. When should you use a shortcut vs Mirroring vs a Copy (pipeline/Copy job)?
| Need | Choose |
|---|---|
| Data already in a lake/object store (ADLS, S3, GCS, other Fabric item), want zero copy | **Shortcut** |
| Operational database (Azure SQL, Cosmos DB, Snowflake, etc.) to be replicated near-real-time as Delta | **Mirroring** |
| Need to reshape, filter, historise, or the source is an API/file system that shortcuts don't support | **Copy activity / Copy job / Dataflow Gen2** |
| Cross-cloud data queried heavily, egress/latency is a problem | Copy it, or enable **shortcut caching** |

### S27. Data in Amazon S3 is queried every hour by reports and the AWS egress bill is rising.
- Enable **shortcut caching** at workspace level (supported for S3, S3-compatible, GCS and on-prem gateway shortcuts); files are cached in OneLake for a configurable retention period (1–28 days), so repeat reads don't hit S3. Very large files aren't cached.
- If data is not Delta, use a notebook to convert it into a Delta table once, then build Direct Lake on that.
- Aggregate/pre-compute in Gold so reports read smaller data.

### S28. A Finance domain owns curated Gold tables; Sales and HR workspaces need them without copies (data mesh).
- Finance keeps the **single source** in its lakehouse; Sales/HR create **internal OneLake shortcuts** into their own lakehouses.
- Grant consumers only **ReadData/item-level** permission (or OneLake security roles) on the Finance item — they can't modify the source.
- Endorse (certify) the Finance lakehouse; lineage view shows downstream consumers.

### S29. How do shortcuts behave in Dev → Test → Prod (Git & deployment pipelines)?
- Shortcut definitions are part of the lakehouse item definition, so they travel with **Git integration and deployment pipelines**.
- Problem: a Dev shortcut points to Dev data. Fix with **variable libraries** / deployment rules (or fabric-cicd parameter files) to repoint the target workspace/item/connection per stage.
- Validate after deployment that the connection used by external shortcuts exists in the target stage.

### S30. When should you *not* use a shortcut?
- Source is slow, far away (cross-region/cross-cloud) and queried heavily → copy or cache.
- You need your own history/snapshots — the source owner may VACUUM or overwrite, breaking time travel.
- Source schema changes without notice and breaks downstream Direct Lake models.
- Source isn't a supported storage type (REST API, SFTP, most databases) → use pipelines, Copy job or Mirroring.

---

## Dataflows Gen2 — scenario deep-dive

### S31. Why do we need Dataflows Gen2 when we already have pipelines and notebooks?
- **Low-code / no-code** transformation with **Power Query** (the same experience as Excel/Power BI) and 150+ connectors → business analysts and citizen developers can build ETL without Python or SQL.
- Pipelines **move and orchestrate**, notebooks need **code**; Dataflow Gen2 fills the gap for **visual transformation** (merge, pivot/unpivot, split columns, fuzzy match, type cleaning).
- Results land in a governed destination (Lakehouse/Warehouse) inside OneLake instead of personal Excel files, so IT can monitor, secure and reuse the data.

### S32. What changed from Dataflow Gen1 to Gen2 — why the upgrade?
- **Output destinations:** Gen2 writes to Lakehouse, Warehouse, KQL database, Azure SQL DB, SharePoint files, etc. Gen1 data lived in internal storage usable mainly by Power BI.
- **Staging + Fabric compute:** staged data in a lakehouse/warehouse lets heavy transformations run at scale (SQL engine), not only in the mashup engine.
- **Pipeline integration:** can run as a **Dataflow activity** inside pipelines with dependencies and alerts.
- **Monitoring hub & refresh history** with per-table details; **Fast Copy** for large ingestion; **incremental refresh**; **parameters** that pipelines can pass in (public parameters); **Git/CI-CD** support; **Copilot** to generate steps.
- Migration path: *Save as Dataflow Gen2* from a Gen1 dataflow, then set destinations.

### S33. When should you choose Dataflow Gen2 vs Copy job/Copy activity vs Notebook?
| Scenario | Best fit |
|---|---|
| Analysts, moderate volume, many cleaning steps, Excel/SharePoint/SaaS sources | **Dataflow Gen2** |
| Pure bulk movement, no transformation, very large volume | **Copy activity / Copy job** |
| Complex logic, big data, ML, MERGE/SCD, performance-critical | **Notebook (Spark)** |
| SQL-skilled team, transformations in Gold | **Warehouse T-SQL / stored procs** |
- Mention cost: Dataflow Gen2 is usually **more CU-hungry** than Copy or Spark for the same volume (see S6).

### S34. Every week the business uploads a messy Excel file to SharePoint; they want a clean Lakehouse table without IT writing code.
- Dataflow Gen2 → SharePoint folder connector → Power Query steps (promote headers, remove blank rows, fix types, unpivot months).
- **Output destination:** Lakehouse table, update method **Replace** (or Append with a load-date column).
- Schedule refresh (or trigger from a pipeline after file arrives); failure notification to the owner.
- Use **data validation** steps (e.g., keep errors → separate error table) so bad rows don't break the load.

### S35. A Dataflow Gen2 refresh of 50 M rows takes 2 hours. How do you speed it up?
1. Check **query folding** (step indicators) — push filters/joins back to the source; move non-folding steps to the end.
2. Enable **Fast Copy** for large source → destination loads.
3. Use **staging** for heavy merges/aggregations so the warehouse engine does the work; disable staging for small queries where it only adds overhead.
4. Use **incremental refresh** (DateTime column, refresh only recent buckets).
5. Split into multiple dataflows (ingest raw vs transform), or move the heavy part to a notebook/Copy job.
6. Check gateway sizing for on-prem sources and Capacity Metrics for throttling.

### S36. How do you implement incremental loads in Dataflow Gen2?
- Built-in **incremental refresh**: choose a DateTime column, the bucket size (day/month), how far back to extract, and which buckets to refresh; the destination must support it (e.g., Lakehouse/Warehouse) and the query should fold.
- Alternative pattern: pipeline passes a **watermark parameter** to the dataflow (public parameters), dataflow filters `ModifiedDate > @watermark` and **Appends** to the destination; pipeline updates the watermark table.

### S37. The dataflow needs data from an on-premises SQL Server and Oracle.
- Install/cluster the **on-premises data gateway**, create connections in *Manage connections and gateways*, and bind the dataflow queries to them.
- Keep the gateway close to the source, size its machine for mashup work, and use folding so the database filters rows before they cross the network.

### S38. How do you orchestrate and govern many Dataflow Gen2 items in production?
- Call them from a **pipeline Dataflow activity** (pass parameters, chain dependencies, retries, Teams/Outlook alerts on failure).
- Put them in **Git** and promote with deployment pipelines; parameterise connections/destinations per stage.
- Keep analyst-built dataflows in a **separate workspace/capacity** and watch CU in the Capacity Metrics app; endorse the destination tables, not the dataflow itself, as the reusable source.
