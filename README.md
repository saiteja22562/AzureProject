# Azure Data Platform — Spotify Data Engineering

> A hands-on Azure data engineering project built to explore how ingestion, distributed transformation, data quality, and curated data products fit together in a modern lakehouse architecture.

This project started as a pipeline exercise, but I used it to go deeper into the engineering decisions behind the pipeline — **where data should land, where transformations should happen, how layers should be separated, and what needs to change before calling a solution production-ready.**

---

## What problem am I solving?

The goal is to take source data through multiple stages of an Azure data platform:

```text
Source
  ↓
Ingestion
  ↓
Raw storage
  ↓
Distributed transformation
  ↓
Data quality / refinement
  ↓
Curated data
  ↓
Analytics
```

The implementation uses a **Medallion Architecture** to keep these stages independently understandable and easier to evolve.

---

## Architecture

```text
                         ┌─────────────────┐
                         │   Spotify Data  │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │     Azure Data Factory  │
                    │     Ingestion / Control │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │         BRONZE          │
                    │     Raw landed data     │
                    │       ADLS Gen2         │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       Databricks        │
                    │         PySpark         │
                    │ Distributed processing  │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │         SILVER          │
                    │ Cleaned / standardized  │
                    │    enriched datasets    │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Lakeflow Declarative     │
                    │ Pipelines               │
                    │ (formerly DLT)          │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │          GOLD           │
                    │ Curated analytical data │
                    └─────────────────────────┘
```

---

## Why did I split the architecture this way?

I don't want every service in the platform doing everything.

Each component has a clear responsibility.

| Component                          | Responsibility                              |
| ---------------------------------- | ------------------------------------------- |
| **Azure Data Factory**             | Orchestration and ingestion                 |
| **ADLS Gen2**                      | Durable data-lake storage                   |
| **Bronze**                         | Source-oriented landed data                 |
| **Databricks / PySpark**           | Distributed transformation                  |
| **Silver**                         | Cleansed and reusable datasets              |
| **Lakeflow Declarative Pipelines** | Declarative pipeline / transformation layer |
| **Gold**                           | Curated analytical datasets                 |

The important design principle is:

> **Separate control, storage, compute, and consumption responsibilities instead of coupling everything into one pipeline.**

---

# Data lifecycle

## 1. Bronze — preserve the source

The Bronze layer is intentionally close to the source representation.

The objective is not to make the data beautiful.

The objective is to create a reliable landing boundary.

### Responsibilities

* Land source data
* Preserve source fidelity
* Minimize transformation during ingestion
* Provide a stable input for downstream processing
* Enable reprocessing when downstream logic changes

```text
Source
  ↓
ADF
  ↓
Bronze
```

---

## 2. Silver — make the data trustworthy

Databricks and PySpark handle the heavier transformation work.

Typical transformations include:

* Data-type standardization
* Null handling
* Deduplication
* Cleansing
* Joins
* Enrichment
* Transformation of source structures into reusable datasets

The Silver layer should represent **trusted engineering data**, not dashboard-specific output.

```text
Bronze
   ↓
Clean
   ↓
Standardize
   ↓
Deduplicate
   ↓
Enrich
   ↓
Silver
```

---

## 3. Gold — make the data useful

The Gold layer is where the data becomes oriented toward consumers.

This can include:

* Curated datasets
* Business-oriented structures
* Aggregations
* Reporting-ready outputs
* Analytical data products

The goal is not simply:

```text
Silver → another folder
```

The goal is:

```text
Engineering data
       ↓
Business meaning
       ↓
Consumer-ready data
```

---

# The engineering decisions

## Why Azure Data Factory?

ADF is used as the orchestration and ingestion layer.

I deliberately keep orchestration separate from Spark transformation logic.

That gives me a clearer boundary:

```text
ADF
 └── "When and in what order should this run?"

Databricks
 └── "How should this data be transformed?"
```

This separation also makes it easier to replace or evolve the processing layer without rewriting the entire orchestration model.

---

## Why Databricks?

The transformation layer is a good fit for distributed processing.

Databricks/PySpark provides:

* Distributed execution
* DataFrame-based transformations
* Large-scale joins and aggregations
* Spark-native processing
* Integration with lakehouse workloads

I would not choose Spark simply because the data platform uses Azure.

The choice should be driven by **workload characteristics**.

---

## Why Medallion Architecture?

The three layers establish clear data boundaries:

```text
Bronze
  │
  │ source representation
  ▼
Silver
  │
  │ engineering representation
  ▼
Gold
  │
  │ business representation
  ▼
Consumers
```

This makes it easier to reason about:

* Data lineage
* Reprocessing
* Data quality
* Ownership
* Failure isolation
* Downstream reuse

---

# Reliability

A pipeline working once is not the same thing as a reliable pipeline.

These are the questions I would ask before putting this design into a production workload.

## Idempotency

If a pipeline is retried, I don't want the retry to blindly create duplicate business records.

A production implementation should establish an appropriate combination of:

* Business keys
* Ingestion identifiers
* Watermarks
* Merge/upsert logic
* Processing state

The exact approach depends on the source-system guarantees.

---

## Replayability

The Bronze layer gives downstream processing a durable input boundary.

If Silver transformation logic changes:

```text
Existing Bronze data
        ↓
New transformation logic
        ↓
New Silver output
```

rather than requiring the source system to be queried again.

This is one of the reasons I prefer separating raw landing from transformation.

---

## Failure isolation

Consider:

```text
ADF ingestion
      ↓
Bronze
      ↓
Databricks
      X
```

If Databricks fails after Bronze has successfully landed the data, I should be able to recover the transformation without automatically re-running the source extraction.

That reduces coupling between stages.

---

# Data quality

Data quality should be checked at the boundaries where assumptions change.

```text
Bronze
 │
 ├── Schema validation
 ├── Completeness checks
 └── Ingestion validation
 │
 ▼
Silver
 │
 ├── Null checks
 ├── Duplicate detection
 ├── Data-type validation
 ├── Referential checks
 └── Business rules
 │
 ▼
Gold
 │
 ├── Reconciliation
 ├── Aggregation checks
 └── Consumer-level validation
```

The important distinction is between:

**"I know this should be checked"**

and

**"I have actually implemented the check."**

This repository documents both separately rather than pretending every production control already exists.

---

# Performance thinking

I don't want to claim that a Spark pipeline is "high performance" simply because it uses Databricks.

For a larger workload, I would investigate:

### Spark

* Shuffle volume
* Data skew
* Join strategy
* Partition count
* Partition pruning
* Predicate pushdown
* Small-file accumulation
* Unnecessary caching
* Repeated scans
* Adaptive Query Execution

### Storage

* File format
* File size
* Partition layout
* Read/write patterns

### Pipeline

* Unnecessary data movement
* Serial dependencies
* Over-parallelization
* Compute utilization

The rule is simple:

> **Measure first. Optimize second.**

---

# What happens at 10× the current workload?

This is where the architecture becomes interesting.

At a much larger scale, I would revisit:

1. Metadata-driven ingestion
2. Incremental processing
3. CDC
4. Partition strategy
5. File-size management
6. Spark join strategy
7. Data-quality automation
8. Centralized observability
9. Cost controls
10. CI/CD
11. Governance
12. Disaster recovery

I would not automatically add more Azure services.

I would first identify the actual bottleneck.

---

# Deployment

The Databricks portion includes deployment-oriented configuration such as:

```text
databricks.yml
pyproject.toml
```

A mature deployment flow would look like:

```text
Developer branch
      ↓
Pull Request
      ↓
Validation
      ↓
Automated tests
      ↓
Build
      ↓
DEV
      ↓
Approval
      ↓
PROD
```

The presence of deployment configuration should not be confused with a complete enterprise CI/CD implementation.

That distinction is intentional.

---

# Security

A production implementation should never depend on credentials committed into source control.

Depending on the environment, I would use:

* Managed Identity
* Azure Key Vault
* RBAC
* Unity Catalog permissions
* Environment-specific configuration
* Secret references
* Least-privilege access

Security should be enforced at the platform boundary rather than scattered throughout transformation code.

---

# Repository structure

```text
.
├── Bronze Layer- Azure Data Factory/
│   ├── dataset/
│   ├── factory/
│   ├── linkedService/
│   └── pipeline/
│
├── Silver Layer - Databricks/
│   └── spotifynotebook/
│       ├── src/
│       ├── utils/
│       ├── resources/
│       ├── databricks.yml
│       └── pyproject.toml
│
├── Gold Layer - Databricks/
│   └── dlt/
│       ├── explorations/
│       ├── transformations/
│       └── utilities/
│
└── spotifyload.sql
```

---

# What is implemented vs. what I would harden

This is deliberately explicit.

| Area                                       | Current position     |
| ------------------------------------------ | -------------------- |
| Azure Data Factory ingestion/orchestration | Implemented          |
| Databricks/PySpark transformation          | Implemented          |
| Bronze/Silver/Gold structure               | Implemented          |
| Databricks deployment configuration        | Present              |
| Incremental framework                      | Production extension |
| CDC                                        | Production extension |
| Automated data-quality framework           | Production extension |
| Centralized observability                  | Production extension |
| CI/CD promotion workflow                   | Production extension |
| Infrastructure as Code                     | Production extension |
| Enterprise DR/SLO design                   | Production extension |

I prefer this distinction over calling a portfolio project "production-ready" when it has not actually operated under production conditions.

---

# What I learned from the project

The biggest lesson wasn't how to connect ADF to Databricks.

It was understanding that a data platform is really a collection of **boundaries**.

```text
Where does data become durable?
Where does transformation happen?
Where is quality enforced?
What can be replayed?
What happens when a stage fails?
What is the recovery boundary?
Which component owns each responsibility?
```

Those questions matter more than the number of Azure services in the architecture.

---

# Technology stack

**Cloud**

* Microsoft Azure
* Azure Data Factory
* Azure Data Lake Storage Gen2
* Azure Databricks

**Data Engineering**

* Apache Spark
* PySpark
* SQL
* Delta-oriented lakehouse processing

**Pipeline**

* Lakeflow Declarative Pipelines
* formerly Delta Live Tables (DLT)

**Development**

* Python
* Git
* GitHub

---

# Next engineering iterations

If I continue evolving this project, I would prioritize:

```text
1. Metadata-driven ingestion
2. Incremental processing / CDC
3. Data-quality framework
4. Centralized monitoring
5. CI/CD with GitHub Actions
6. Infrastructure as Code
7. Governance and lineage
8. Cost/performance benchmarking
9. Failure-injection testing
10. SLA / SLO definition
```

The order matters.

I would rather make the existing pipeline **observable, testable and recoverable** before adding another cloud service.

---

## Engineering principle

> **Use the simplest architecture that satisfies the requirements.**

More services do not automatically make a better platform.

Clear ownership, explicit failure boundaries, measurable performance, and defensible trade-offs do.

---

### Scope

This repository is a portfolio/reference implementation using representative Spotify data.

[1](https://github.com/saiteja22562/AzureProject)
