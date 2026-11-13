# 04a · Theory Interview Questions (with answers)

## Basics
**1. What is Microsoft Fabric?**
A unified SaaS analytics platform that combines data integration (Data Factory), data engineering (Spark), data warehousing, data science, real-time intelligence and Power BI on a single lake (OneLake), with shared security, governance and capacity-based billing.

**2. What problem does Fabric solve?**
Tool sprawl and data duplication. Instead of ADF + ADLS + Synapse + Databricks + Power BI each with their own copies, security and bills, Fabric gives one copy of data, one security model and one capacity.

**3. What is OneLake?**
A single, tenant-wide logical data lake built on ADLS Gen2, auto-provisioned, storing tabular data as Delta/Parquet. Every Fabric engine reads/writes it. Often called "OneDrive for data".

**4. How many OneLakes does an organisation have?**
Exactly one per tenant.

**5. What is the hierarchy in Fabric?**
Tenant → Capacity → (Domain) → Workspace → Items.

**6. What is a workspace?**
A collaboration container for items, with roles (Admin, Member, Contributor, Viewer), assigned to a capacity, mapped to a folder in OneLake.

**7. What are the workspace roles and what can each do?**
- **Admin** – everything incl. manage access & delete workspace.
- **Member** – add Contributors/Viewers, share items, create/edit.
- **Contributor** – create/edit/delete items, cannot manage access.
- **Viewer** – read/view only (can query SQL endpoint, view reports).

**8. What is a Lakehouse?**
An item combining files (`/Files`, any format) and managed Delta tables (`/Tables`) in OneLake, with Spark for processing and an auto-generated read-only SQL analytics endpoint.

**9. What is the SQL analytics endpoint?**
An auto-created, read-only T-SQL interface over a lakehouse's Delta tables. You can create views, functions and apply SQL security, but not INSERT/UPDATE/DELETE into tables.

**10. Difference between Lakehouse and Warehouse?**
Lakehouse = Spark-first, supports unstructured data, T-SQL read-only. Warehouse = T-SQL-first, full DML, multi-table transactions, structured data only. Both store Delta in OneLake.

**11. What is Delta Lake and why does Fabric use it?**
Open table format = Parquet files + transaction log (`_delta_log`). Gives ACID transactions, time travel, schema enforcement/evolution, upserts. It's open, so Spark, SQL, KQL and Power BI can all read the same copy.

**12. What is V-Order?**
A write-time optimisation of Parquet files (special sorting, row-group distribution, encoding/compression) that speeds up reads by Power BI (VertiPaq), SQL and Spark. Files remain standard Parquet.

**13. What are shortcuts?**
Virtual pointers in OneLake to data stored elsewhere (another lakehouse/workspace, ADLS Gen2, Amazon S3, GCS, S3-compatible, Dataverse…). No data copy; data appears as files/tables.

**14. What is mirroring?**
Continuous, near-real-time replication of an external operational database (Azure SQL DB/MI, Cosmos DB, Snowflake, Databricks, PostgreSQL, open mirroring) into OneLake as Delta tables, with a SQL endpoint for analytics.

**15. Shortcut vs Mirroring?**
Shortcut = no copy, reads source directly, best when source already in an open format. Mirroring = replicates (CDC) operational DB into Delta in OneLake.

**16. What is Direct Lake mode?**
A Power BI semantic-model storage mode that loads Delta/Parquet columns from OneLake directly into memory on demand — no import refresh and no DirectQuery translation. Gives near-Import performance with near-real-time data.

**17. What is "framing" in Direct Lake?**
The refresh operation that points the model to the latest Delta table version (metadata only, very fast). Can be automatic when data changes.

**18. When does Direct Lake fall back to DirectQuery?**
(For Direct Lake on SQL endpoint) when guardrails are exceeded (rows/files/row groups/model size per SKU), when querying SQL views, or when SQL-endpoint security (RLS/OLS) must be enforced. Can be controlled with `DirectLakeBehavior` (Automatic / DirectLakeOnly / DirectQueryOnly).

**19. What is a Fabric capacity and a CU?**
Capacity = dedicated pool of compute bought as an F-SKU. Capacity Unit (CU) = the unit of compute all workloads consume. F64 = 64 CUs.

**20. Explain bursting and smoothing.**
Bursting lets a job use more CUs than the SKU momentarily to finish faster. Smoothing spreads that consumption over future time (≈5 min for interactive, 24 h for background) so you don't need to size for peaks.

**21. What is throttling?**
When smoothed future usage exceeds limits: first interactive requests are delayed, then interactive requests rejected, then background jobs rejected.

**22. What is the minimum SKU for free users to view Power BI content?**
F64 (or P1). Below that, viewers need Pro or PPU licences.

## Data Factory
**23. Difference between pipeline and Dataflow Gen2?**
Pipeline = orchestration and high-volume copy with control flow. Dataflow Gen2 = Power Query-based low-code transformations with output destinations.

**24. What is a Copy job?**
A simplified item for bulk or incremental copies from sources to destinations without authoring a full pipeline; supports incremental (watermark/CDC) modes.

**25. How do you connect to on-premises data?**
On-premises data gateway (or VNet data gateway for Azure private networks).

**26. Name common pipeline activities.**
Copy data, Notebook, Dataflow, Stored procedure, Script, Lookup, Get Metadata, ForEach, If Condition, Switch, Until, Wait, Web, Set/Append variable, Invoke pipeline, Office 365 Outlook/Teams, Fail.

**27. How do you parameterise pipelines?**
Pipeline parameters & variables, dynamic content expressions (`@pipeline().parameters.x`, `@utcnow()`), and variable libraries for environment-specific values.

## Data Engineering / Spark
**28. Starter pool vs custom pool?**
Starter pools are pre-warmed, start in seconds, medium nodes. Custom pools let you choose node size, autoscale and dynamic allocation but take longer to start.

**29. What is an Environment?**
An item that bundles Spark runtime version, compute settings and libraries (public/custom), attached to notebooks/SJDs or set as workspace default.

**30. What is high-concurrency mode?**
Multiple notebooks share one Spark session (same user, same environment) – faster start, lower cost; also used in pipelines.

**31. What is the Native Execution Engine?**
A vectorised C++ execution engine (based on Velox/Gluten) that runs Spark SQL/DataFrame operations natively for better performance, with no code change.

**32. What is notebookutils?**
Fabric utility library (successor to mssparkutils) – file system ops, run/exit notebooks, credentials/secrets from Key Vault, lakehouse ops, runtime context.

**33. Managed vs unmanaged (external) tables in Lakehouse?**
Managed: Spark manages metadata and data under `/Tables`; dropping deletes data. External: metadata only, data elsewhere (e.g. `/Files` path); dropping keeps data.

**34. OPTIMIZE vs VACUUM?**
OPTIMIZE compacts small files into larger ones (optionally Z-ORDER/V-ORDER). VACUUM removes files no longer referenced by the Delta log older than the retention period (default 7 days).

**35. How do you handle schema changes?**
Delta schema evolution: `.option("mergeSchema", "true")` or `ALTER TABLE ADD COLUMNS`; schema enforcement blocks incompatible writes.

## Warehouse
**36. Does Fabric Warehouse need distribution keys or indexes?**
No — distribution, scaling and much of the tuning are automatic. You still design good schemas, statistics, and use clustering where available.

**37. Can you query a lakehouse from a warehouse?**
Yes — cross-database queries using three-part names (`lakehouse.dbo.table`) within the same workspace; shortcuts for other workspaces.

**38. What security options exist in Warehouse?**
GRANT/DENY object-level, row-level security (security predicates), column-level security, dynamic data masking, plus workspace/item permissions.

**39. How do you load data into Warehouse?**
`COPY INTO`, pipelines/copy job, Dataflow Gen2, `INSERT…SELECT` / CTAS from lakehouse tables, T-SQL notebooks.

## Real-Time Intelligence
**40. What is an Eventhouse?**
A container of KQL databases optimised for time-series/log/streaming data, queried with KQL; data can be exposed in OneLake (OneLake availability).

**41. What is an Eventstream?**
No-code item to capture, transform and route real-time events from sources (Event Hubs, IoT Hub, Kafka, CDC, custom apps) to destinations (Eventhouse, Lakehouse, Activator…).

**42. What is Activator?**
A no-code detection engine that watches data (streams, reports) for conditions and triggers actions — email, Teams message, pipeline/notebook run, Power Automate.

## Governance / ALM
**43. How does Git integration work?**
A workspace is connected to a branch in Azure DevOps or GitHub; supported items are serialised as files; you commit and update from the workspace UI or APIs.

**44. What are deployment pipelines?**
Fabric's Dev → Test → Prod promotion tool, with deployment rules to change data sources/parameters per stage.

**45. What is endorsement?**
Marking items as **Promoted** or **Certified** (by authorised users) to signal trustworthiness; also "Master data".

**46. What is OneLake security?**
Role-based data access (folders/tables, with row- and column-level rules) defined once in OneLake and enforced across Spark, SQL endpoint and Direct Lake.

**47. What is a domain?**
A logical grouping of workspaces by business area for decentralised (data mesh) governance, with domain admins and settings.

**48. How is Purview integrated?**
Purview hub in Fabric, automatic metadata scanning, sensitivity labels inherited downstream, DLP policies, audit logs.

## AI
**49. What can Copilot do in Fabric?**
Generate/explain code in notebooks, SQL in warehouse, KQL queries, Dataflow transformations, pipeline expressions, Power BI reports and DAX.

**50. What is a Fabric data agent?**
A configurable conversational agent that answers natural-language questions grounded on your lakehouses, warehouses, semantic models and KQL databases, respecting user permissions.
