# 03 · Relationship between Microsoft Fabric and Azure Synapse Analytics

## 1. The short story
- **Azure Synapse Analytics** (2019/2020) = Microsoft's **PaaS** analytics service that stitched together SQL pools, Spark pools, pipelines and Data Explorer in one *workspace*, on top of your own ADLS Gen2.
- **Microsoft Fabric** (2023) = the **SaaS evolution** of that idea. Microsoft took the Synapse engines (SQL, Spark, Data Explorer, pipelines), re-built them around **OneLake + Delta Lake**, and merged them with **Power BI** and **Data Factory**.
- Early Fabric literally named its workloads "Synapse Data Engineering", "Synapse Data Warehouse", "Synapse Data Science", "Synapse Real-Time Analytics" — the "Synapse" prefix was dropped in 2024.
- Fabric is widely described as the **natural successor** to Synapse, but there is **no automatic upgrade path** — moving is a migration project.
- Microsoft has said it has **no current plans to retire Azure Synapse Analytics**; existing customers can keep running it. New feature investment, however, is focused on Fabric.

> Interview one-liner: *"Synapse is PaaS — you manage pools, storage and scale. Fabric is SaaS — Synapse's engines rebuilt on OneLake with Power BI and Data Factory, billed by a single capacity. Fabric is the strategic successor, but Synapse is not retired."*

---

## 2. Component mapping (Synapse → Fabric)
| Azure Synapse Analytics | Microsoft Fabric equivalent | Key change |
|---|---|---|
| Synapse workspace | Fabric workspace | SaaS, no Azure resource per workspace |
| ADLS Gen2 (your storage account) | **OneLake** | One lake per tenant, auto-provisioned; shortcuts can still point to ADLS |
| **Dedicated SQL pool** (MPP, DWUs) | **Fabric Warehouse** | No distributions/indexes/DWUs; Delta storage; auto-scale |
| **Serverless SQL pool** | **SQL analytics endpoint** of Lakehouse | Auto-created, read-only T-SQL over Delta |
| **Apache Spark pool** | **Spark in Data Engineering** (starter/custom pools, notebooks, SJDs) | Starter pools start in seconds; Environments replace pool-level libs |
| **Synapse pipelines** / ADF | **Data Factory pipelines** + Dataflow Gen2 + Copy job | Mostly same activities; Dataflow Gen2 replaces mapping data flows |
| Mapping data flows (Spark-based) | Dataflow Gen2 (Power Query) or notebooks | No 1:1 equivalent — rewrite |
| **Data Explorer pool** | **Eventhouse / KQL Database** (Real-Time Intelligence) | Plus Eventstream, Activator, RT dashboards |
| Synapse Link (for SQL/Cosmos/Dataverse) | **Mirroring** / Dataverse shortcut ("Link to Fabric") | Lands as Delta in OneLake |
| Power BI (separate service, linked) | **Power BI built in** + **Direct Lake** | No import copy needed |
| Synapse Studio | Fabric portal (app.fabric.microsoft.com) | One UI for all personas |
| Azure RBAC + SQL permissions | Workspace roles + item perms + OneLake security | Entra ID based, unified |
| Pay per DWU / vCore / pipeline run | **Single capacity (F-SKU, CUs)** | One bill, shared by all workloads |
| Git (Synapse Studio) | Fabric Git integration + deployment pipelines | Works across all item types |

---

## 3. Side-by-side comparison
| Aspect | Azure Synapse Analytics | Microsoft Fabric |
|---|---|---|
| Service model | PaaS | SaaS |
| Storage | ADLS Gen2 (any format) | OneLake (Delta/Parquet open format) |
| Compute | Separate pools you size & pause | Shared capacity, serverless engines |
| Data copies | Often multiple (lake → DW → Power BI import) | "One copy" via OneLake & Direct Lake |
| Warehouse tuning | Distribution keys, indexes, partitions, resource classes, workload mgmt | Mostly automatic (statistics, V-Order, caching) |
| Real-time | Data Explorer pools (preview, limited) | Full Real-Time Intelligence workload |
| BI | External Power BI | Native Power BI, Direct Lake |
| AI | Azure ML integration | Copilot, data agents, MLflow, Semantic Link |
| Governance | Purview (separate) | Purview built in, OneLake catalog, labels |
| Networking | Mature VNet, private endpoints | Private Links, managed VNets, trusted access (maturing) |
| Admin effort | Higher | Lower |
| Best for | Existing heavy MPP DW workloads, deep network control | New projects, unified analytics, Power BI–centric orgs |

---

## 4. Migration paths
### 4.1 Dedicated SQL pool → Fabric Warehouse
- Microsoft provides a **Migration Assistant for Fabric Data Warehouse** that copies schema/metadata from a dedicated SQL pool (DACPAC) and helps fix incompatible T-SQL with Copilot assistance.
- Data movement: `COPY INTO` from ADLS (export with CETAS to Parquet first), pipelines/copy job, or shortcuts.
- Rework needed for: distribution/index hints, some unsupported T-SQL features and data types, workload management/resource classes, materialised views, etc. (check current docs — the gap is shrinking).

### 4.2 Spark pools → Fabric Spark
- Move notebooks (export/import `.ipynb`), Spark job definitions, and pool configs → **Environments**.
- Replace `mssparkutils` with `notebookutils` (backward compatible), update `abfss://` paths to OneLake or create **ADLS shortcuts** so code keeps working.
- Hive metastore tables → re-register as Lakehouse Delta tables (or shortcut existing Delta folders).

### 4.3 Pipelines → Data Factory in Fabric
- Most activities exist; linked services become **connections**; integration runtimes become **gateways**.
- Mapping data flows must be rewritten (Dataflow Gen2 or notebooks).
- You can also mount an existing **Azure Data Factory** in a Fabric workspace while migrating.

### 4.4 Data Explorer pools → Eventhouse
- Recreate databases/tables, re-point ingestion via Eventstream; KQL queries mostly portable.

### 4.5 Recommended approach
1. Inventory & assess Synapse assets.
2. Land data in OneLake early with **shortcuts** (zero copy) → quick wins in Power BI Direct Lake.
3. Migrate workload by workload (Spark → pipelines → DW), run in parallel, validate, cut over.
4. Re-design security (workspace roles, OneLake security) rather than lift-and-shift.
5. Size capacity with the Fabric capacity estimator and monitor with Capacity Metrics.

---

## 5. Common interview questions on this topic
1. Is Fabric a replacement for Synapse? → Successor/evolution, but Synapse not retired; no auto-upgrade.
2. What happens to distribution keys in Fabric Warehouse? → Managed automatically; no HASH/ROUND_ROBIN/REPLICATE choice.
3. What is the Fabric equivalent of serverless SQL pool? → SQL analytics endpoint.
4. How would you migrate a Synapse dedicated pool? → Migration assistant / DACPAC + CETAS→Parquet + COPY INTO, fix T-SQL, validate.
5. Can Fabric read data still in Synapse's ADLS? → Yes, via ADLS Gen2 shortcuts (no copy).
6. Cost model difference? → Synapse pays per pool/DWU/run; Fabric pays one capacity shared by all workloads (smoothing & bursting).
7. When would you keep Synapse? → Very large tuned MPP workloads, strict networking needs not yet met, no Power BI, or migration cost not justified yet.
