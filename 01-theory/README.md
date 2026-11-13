# 01 · Microsoft Fabric — Theory

## 1. What is Microsoft Fabric?
Microsoft Fabric is an **end-to-end, SaaS (Software-as-a-Service) analytics platform** that brings data integration, data engineering, data warehousing, data science, real-time analytics and business intelligence (Power BI) into **one product, one storage layer (OneLake), one security model and one billing model (capacity)**.

- **Public preview:** May 2023 (Microsoft Build)
- **General availability:** November 2023 (Microsoft Ignite)
- **Delivery model:** SaaS — no clusters/servers to provision; you buy *capacity* and create *items* in *workspaces*.
- **Open format:** All tabular data is stored as **Delta Lake (Parquet)** in OneLake, so every engine (Spark, T-SQL, KQL, Power BI) reads the same copy.

> One-line definition for interviews: *"Fabric is a unified SaaS analytics platform built on a single data lake (OneLake) in open Delta format, where every workload—ingestion, Spark, SQL warehouse, real-time, ML and Power BI—shares the same data, security and capacity."*

---

## 2. Architecture (layer by layer)

```
[ Copilot · Data agents · AI ]
                 |
[ Data Factory | Data Engineering | Data Warehouse | Data Science | Real-Time Intelligence | Power BI | Databases ]
                 |
[ OneLake — one logical lake per tenant, Delta/Parquet ]
                 |
[ Governance & security — Entra ID, Purview, OneLake security, labels ]
                 |
[ Capacity — F-SKU, Capacity Units (CU) ]
```

### Organisational hierarchy
`Tenant → Capacity → Domain (optional) → Workspace → Items (Lakehouse, Warehouse, Notebook, Pipeline, Report…)`

- **Tenant** – your organisation (Microsoft Entra ID). Has exactly **one OneLake**.
- **Capacity** – pool of compute (Capacity Units, CU) that all workloads consume. SKUs F2 → F2048.
- **Domain** – logical grouping of workspaces by business area (data mesh).
- **Workspace** – collaboration container; maps to a folder in OneLake; has roles.
- **Item** – a Fabric artifact (Lakehouse, Warehouse, Eventhouse, Notebook, Pipeline, Semantic model, Report…).

---

## 3. OneLake — "the OneDrive for data"
- Single, tenant-wide, logical data lake built on **ADLS Gen2**.
- Created automatically — no provisioning.
- Exposes ADLS Gen2-compatible APIs (`abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<item>/...`).
- **One copy** of data: Spark writes Delta → SQL endpoint, Power BI (Direct Lake) and KQL can read it without copying.
- **Shortcuts** – virtual pointers (zero-copy) to data in other OneLake locations, ADLS Gen2, Amazon S3, S3-compatible storage, Google Cloud Storage, Dataverse, etc.
- **Mirroring** – near-real-time replication of operational databases (Azure SQL DB, Azure SQL MI, Cosmos DB, Snowflake, Azure Databricks Unity Catalog, PostgreSQL, "open mirroring" for others) into OneLake as Delta.
- **OneLake catalog** – discover/explore/govern items across the tenant.

---

## 4. Workloads (experiences)

### 4.1 Data Factory (Data Integration)
| Item | What it does |
|---|---|
| **Data pipeline** | Orchestration (Copy activity, Notebook, Stored procedure, ForEach, If, Lookup, Web, etc.). Successor of ADF/Synapse pipelines. |
| **Dataflow Gen2** | Low-code Power Query transformations, 150+ connectors, output destinations (Lakehouse, Warehouse, SQL DB, ADLS…). |
| **Copy job** | Simplified bulk/incremental copy without building a pipeline. |
| **Gateways** | On-premises data gateway and VNet data gateway for private sources. |

### 4.2 Data Engineering
- **Lakehouse** – combines a data lake (`/Files` – any format) with Delta tables (`/Tables`). Every lakehouse gets an auto-generated **SQL analytics endpoint** (read-only T-SQL).
- **Notebooks** – PySpark, Spark SQL, Scala, SparkR; `notebookutils` (formerly `mssparkutils`).
- **Spark Job Definition** – submit batch/streaming Spark jobs.
- **Environments** – reusable Spark settings + libraries.
- **Starter pools** (fast session start) and **custom pools**; **high-concurrency mode**; **Native Execution Engine** (vectorised C++ engine for Spark).

### 4.3 Data Warehouse
- **Fabric Warehouse** – fully transactional T-SQL warehouse (DDL + DML, multi-table ACID transactions) storing data as Delta in OneLake.
- Serverless, auto-scaling distributed engine — **no distribution keys, indexes or DWUs to manage**.
- Cross-database queries across warehouses and lakehouse SQL endpoints in the same workspace.
- Security: object-level, row-level (RLS), column-level (CLS), dynamic data masking.

### 4.4 Data Science
- Notebooks + **MLflow** experiments and model registry, SynapseML, AutoML (FLAML), `PREDICT` for batch scoring, Semantic Link (`sempy`) to read Power BI models from notebooks.

### 4.5 Real-Time Intelligence (RTI)
| Item | Role |
|---|---|
| **Real-Time hub** | Catalog of all streaming sources/events in the tenant |
| **Eventstream** | No-code ingestion & routing of streaming data (Event Hubs, IoT Hub, Kafka, CDC…) |
| **Eventhouse / KQL Database** | Time-series & log analytics store queried with **KQL** |
| **KQL Queryset** | Saved KQL queries |
| **Real-Time Dashboard** | Low-latency dashboards on KQL |
| **Activator** | Detect patterns/thresholds and trigger actions (email, Teams, pipelines) |

### 4.6 Power BI
- Semantic models, reports, dashboards, apps.
- **Direct Lake mode** – reads Delta/Parquet files from OneLake directly into the VertiPaq engine → *Import-like speed with DirectQuery-like freshness*. Can fall back to DirectQuery (Direct Lake on SQL endpoint) when guardrails are exceeded or for unsupported features.

### 4.7 Databases
- **SQL database in Fabric** – OLTP database (Azure SQL engine) whose data is auto-mirrored to OneLake for analytics.

### 4.8 AI & Copilot
- Copilot in notebooks, pipelines, Dataflows, Warehouse, Power BI and KQL.
- **Data agents** – conversational Q&A agents grounded on lakehouses, warehouses, semantic models and KQL DBs.

---

## 5. Capacity & licensing
- **F SKUs** (F2, F4 … F2048) purchased in Azure; pay-as-you-go (can pause) or reserved.
- **Capacity Unit (CU)** – unit of compute consumed by every workload.
- **Bursting** – a job may temporarily use more CUs than the SKU provides.
- **Smoothing** – consumption is averaged over time (interactive ≈ 5 min, background ≈ 24 h) to avoid spikes.
- **Throttling** – when future capacity is over-consumed: interactive delay → interactive rejection → background rejection.
- Monitor with the **Fabric Capacity Metrics app**.
- Power BI: F64 and above lets free-licence users *view* content; below that, viewers need Pro/PPU.
- **Fabric trial** – 60 days of trial capacity.

---

## 6. Storage format & optimisation
- **Delta Lake** = Parquet files + `_delta_log` transaction log → ACID, time travel, schema enforcement/evolution.
- **V-Order** – Fabric's write-time Parquet optimisation (sorting/encoding) for faster reads by Power BI/SQL.
- **OPTIMIZE** (bin-compaction of small files), **Z-ORDER**, **VACUUM** (remove old files), **Liquid clustering**.
- **Table maintenance** can be run from the Lakehouse UI or via notebooks/API.

---

## 7. Security & governance
- **Authentication:** Microsoft Entra ID.
- **Workspace roles:** Admin, Member, Contributor, Viewer.
- **Item permissions:** share an item (Read, ReadData, ReadAll, Build, Write…).
- **OneLake security (data access roles):** folder/table-level, plus row/column security applied across engines.
- **Engine-level security:** RLS/CLS/OLS/DDM in Warehouse & SQL endpoint; RLS/OLS in semantic models.
- **Network:** Private Links (tenant & workspace level), Managed VNets for Spark, Managed private endpoints, Trusted workspace access, Workspace identity, Customer-managed keys.
- **Governance:** Microsoft Purview integration, sensitivity labels, endorsement (Promoted/Certified), lineage view, domains, audit logs.

---

## 8. Application lifecycle management (CI/CD)
- **Git integration** – connect a workspace to Azure DevOps or GitHub; items serialised as source files.
- **Deployment pipelines** – Dev → Test → Prod promotion with deployment rules.
- **REST APIs**, Fabric CLI, `fabric-cicd` Python library, Terraform provider.
- **Variable libraries** for environment-specific values.

---

## 9. Medallion architecture in Fabric
| Layer | Purpose | Typical Fabric item |
|---|---|---|
| **Bronze** | Raw, as-is ingested data | Lakehouse (Files + raw Delta) |
| **Silver** | Cleaned, deduplicated, conformed | Lakehouse Delta tables |
| **Gold** | Business-ready star schemas/aggregates | Lakehouse or Warehouse → Direct Lake semantic model |

Best practice: separate lakehouses (or workspaces) per layer, orchestrate with pipelines, use shortcuts to avoid copying.

---

## 10. Certifications mapped to Fabric
| Exam | Role | Focus |
|---|---|---|
| **DP-600** | Fabric Analytics Engineer Associate | Lakehouse/Warehouse, semantic models, Power BI, DAX, Direct Lake |
| **DP-700** | Fabric Data Engineer Associate | Ingestion, Spark, pipelines, RTI/KQL, security, monitoring (replaced DP-203) |
| **DP-900** | Data Fundamentals | Basics |
