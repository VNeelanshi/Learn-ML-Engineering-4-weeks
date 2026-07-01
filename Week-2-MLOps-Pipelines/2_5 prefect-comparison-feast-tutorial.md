# Prefect, Orchestrator Trade-offs & Feast — A 4-Hour Guide

*Hands-on, with your Airflow knowledge as the starting point. New tools and terms are defined the first time they appear.*

## What this assumes, and what's new

You already know Apache Airflow well — DAGs, tasks, operators, the scheduler/executors, XCom, retries, deployment. This guide uses that as an anchor (the plan itself frames Prefect as an "Airflow alternative"), so it explains the new tools *by contrast* with what you know rather than from scratch.

New here, and defined as we go: **Prefect** (a different orchestration model), how **Prefect / Dagster / Kubeflow** differ from Airflow and from each other, and **Feast** (a feature store — a new category of tool entirely).

### One-time setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install prefect          # Hour 1
pip install feast scikit-learn pandas   # Hours 3-4
```
(Dagster and Kubeflow in Hour 2 are discussed, not installed — Kubeflow in particular needs a Kubernetes cluster, covered there.)

---

# Hour 1 — Prefect basics

**Goal:** understand Prefect's model and why it's shaped differently from Airflow.

## 1.1 The core philosophical difference

Airflow's model is *structure first*: you declare a DAG, wire tasks with `>>`, and the scheduler runs that fixed graph. The shape of the workflow is fixed before anything runs.

**Prefect** inverts this. You write **ordinary Python** — including `if`, `for`, and function calls — and Prefect observes it as it runs. There's no separate DAG object you build up front; the workflow's shape can depend on runtime data. Prefect describes itself as adding orchestration to Python "as-is": you decorate functions and keep your normal control flow.

Why this matters: workflows whose shape is only known at runtime (loop over however many files arrived today, branch on a value you just computed) are awkward in a fixed-DAG model and natural in Prefect. The trade-off is that Prefect gives you less of a fixed, inspectable graph up front.

## 1.2 Flows and tasks (the two building blocks)

- A **flow** is a Python function decorated with `@flow`. It's the top-level unit Prefect orchestrates — roughly the counterpart to an Airflow DAG, but it's a *function you call*, not a graph you declare. Decorating a function with `@flow` gives it state tracking, retries, timeouts, logging, and observability. A flow can call other flows (those are **subflows**).

- A **task** is a Python function decorated with `@task`. It's a unit of work inside a flow — the counterpart to an Airflow task. Tasks add retries, caching, and concurrency to a single piece of work, and Prefect tracks dependencies between them automatically.

A key difference from Airflow: **tasks are optional.** A flow can be useful with no tasks at all. You add `@task` when you want a piece of work to have its own retries/caching/visibility.

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=5)   # note: retry_delay_seconds, not Airflow's retry_delay
def extract():
    print("Extracting...")
    return [1, 2, 3, 4]

@task
def transform(data):
    return [x * 10 for x in data]

@flow(log_prints=True)   # log_prints=True sends print() output to Prefect's logs
def my_pipeline():
    raw = extract()
    result = transform(raw)        # passing the result creates the dependency
    print(f"Result: {result}")
    return result

if __name__ == "__main__":
    my_pipeline()                  # just call it — it runs immediately
```

Run it with `python my_pipeline.py`. Notice what you did **not** do: no `>>`, no DAG object, no scheduler required to test it. Dependencies are inferred exactly like the TaskFlow API you know — by passing one task's return value into another.

> **Gotcha — calling a flow runs it now.** `my_pipeline()` executes immediately, like any Python function. That's great for development but means there's no schedule yet. Scheduling is a separate step (1.4).

> **Gotcha — call tasks from inside a flow.** A `@task` function is meant to be invoked within a flow run. Calling it bare, outside any flow, won't get you Prefect's orchestration features.

## 1.3 Running tasks concurrently

By default, when you call a task and use its return value, it runs and you wait for the result — sequential. To run tasks **concurrently**, use `.submit()`, which returns a *future* (a placeholder for a result that isn't ready yet):

```python
@flow
def parallel_pipeline():
    futures = [process.submit(i) for i in range(5)]   # 5 tasks start concurrently
    results = [f.result() for f in futures]            # wait for all
    return results
```

This is Prefect's equivalent of fanning out work — but expressed in plain Python rather than DAG edges.

## 1.4 Scheduling and deployment (where work pools and workers come in)

So far you've run flows by hand. To run them on a schedule or trigger them remotely, you create a **deployment** — the act of telling Prefect *where, when, and how* a flow should run, turning it from a function you call into an API-managed entity. (Same concept as deploying an Airflow DAG, different mechanics.)

The simplest path is `.serve()`, which is enough for most scheduling:

```python
if __name__ == "__main__":
    my_pipeline.serve(name="daily-pipeline", cron="0 6 * * *")
```

This starts a long-running process that schedules and runs the flow on the cron you gave — no extra infrastructure.

For production across varied infrastructure, Prefect uses **work pools** and **workers** (this is what replaced the old "agents"):

- A **work pool** is a named channel that holds scheduled runs and the configuration for how/where they execute. Think of it like a queue topic that connects "work that needs running" to "things that run work." It decouples your flow code from the infrastructure it runs on — switch a deployment's work pool to move it from local to Docker to Kubernetes without changing the flow.
- A **worker** is a lightweight process that polls a work pool, picks up scheduled runs, and executes them on the matching infrastructure. A worker's *type* (process, Docker, Kubernetes…) determines where the flow actually runs.

```bash
prefect work-pool create my-pool --type process     # create the pool
prefect worker start --pool my-pool                 # start a worker that polls it
prefect deploy                                       # register a deployment to that pool
```

> **Gotcha — "agents" are gone.** Old tutorials say `prefect agent start`. In Prefect 3 that's deprecated; the replacement is `prefect worker start --pool <name>`. If you copy an agent-based example it won't reflect the current model.

> **Gotcha — pausing a work pool doesn't pause schedules.** If you pause a pool, the deployment keeps *scheduling* runs; they just won't execute until you resume. They queue up and then all run — similar in spirit to Airflow's catchup surprise.

## 1.5 The backend: Prefect Server vs Prefect Cloud

Prefect needs a backend (an API + database + UI) to track runs and serve the dashboard:

- **Prefect Server** — the self-hosted, open-source backend. Start it with `prefect server start` (UI at `http://127.0.0.1:4200`). Like running your own Airflow webserver + metadata DB.
- **Prefect Cloud** — the managed, hosted backend, with extra features (automations, user management, audit trails). Like Astronomer/MWAA, but for Prefect.

> **Gotcha — no backend means no tracking.** If `PREFECT_API_URL` isn't pointed at a running server (or Cloud), flows still *run* but their history isn't recorded in a UI. For development, run `prefect server start` and set `PREFECT_API_URL=http://127.0.0.1:4200/api`.

## ✅ Hour 1 Exercise

1. Write `etl_flow.py` with three tasks — `extract`, `transform`, `load` — wired by passing return values, inside a `@flow`. Give `extract` `retries=2, retry_delay_seconds=5`.
2. Run it directly (`python etl_flow.py`) and read the run output.
3. Start a backend (`prefect server start` in another terminal; set `PREFECT_API_URL`) and re-run; find the flow run in the UI at `:4200`.
4. Add a fourth task that runs five sub-jobs **concurrently** with `.submit()` and collects their results.
5. Replace the manual run with `etl_flow.serve(name="etl", cron="*/5 * * * *")` and watch it fire on schedule, then stop it.

---

## 🎤 Hour 1 — Interview Questions & Answers

**Q1 (conceptual). How does Prefect's execution model differ from Airflow's, and when does that difference matter?**
Airflow declares a fixed DAG up front and the scheduler runs that static graph. Prefect runs ordinary Python and observes execution at runtime, so the workflow's shape can depend on runtime data (loops over dynamic inputs, branches on computed values) using native Python control flow. It matters most for dynamic or unpredictable workflows; Airflow's fixed graph is better when the structure is known and stable.

**Q2 (conceptual). What's the difference between a flow and a task in Prefect, and are tasks required?**
A flow (`@flow`) is the top-level orchestrated unit — like a DAG but invoked as a function; a task (`@task`) is a unit of work inside a flow with its own retries/caching/concurrency. Tasks are optional: a flow can run useful work with no tasks. You add tasks when a step needs independent retry/caching/observability.

**Q3 (gotcha). A candidate's tutorial uses `prefect agent start`. What's wrong and what's current?**
Agents are deprecated in Prefect 3. The current model is work pools plus workers: `prefect worker start --pool <name>`. A worker polls a work pool for scheduled runs and executes them on infrastructure matching the worker's type.

**Q4 (practical). How do you make tasks run concurrently in Prefect, and what does `.submit()` return?**
Call the task with `.submit()`, which schedules it to run concurrently and returns a future (a placeholder for a not-yet-ready result). Collect results with `.result()`. Calling a task normally and using its return value runs it sequentially.

**Q5 (practical). What's the minimal way to put a Prefect flow on a schedule, and when would you reach for work pools instead?**
`flow.serve(name=..., cron=...)` starts a process that schedules and runs the flow — enough for most cases with no extra infra. Reach for work pools and workers when you need to run across configurable/remote infrastructure (Docker, Kubernetes, serverless) or to decouple flow code from where it executes.

---

# Hour 2 — Orchestrator comparison & decision matrix

**Goal:** place Airflow, Prefect, Dagster, and Kubeflow on a map and know which to reach for.

## 2.1 The one axis that explains most of the differences

The biggest conceptual split is **what the tool treats as the central thing**:

- **Task-centric** (Airflow, Prefect): you model *steps* and the order they run in. The pipeline is "do this, then that." Whether data was produced correctly is your concern, not the orchestrator's.
- **Asset-centric** (Dagster): you model the *data products* (datasets, models, tables) the pipeline produces, and declare what each asset depends on. The orchestrator then knows how to (re)build any asset and tracks lineage automatically.

A second axis is **where it runs**: most are infrastructure-flexible, but **Kubeflow is Kubernetes-native** — it only makes sense on a Kubernetes cluster.

## 2.2 The four tools, briefly

**Apache Airflow** *(your baseline)* — the most-deployed orchestrator, with the largest ecosystem (1,000+ provider packages) and managed options (MWAA, Cloud Composer, Astronomer). Task-centric DAGs. Heaviest operational footprint of the four. Airflow 3 added asset-aware scheduling, narrowing the gap with Dagster. Strongest where you need broad integrations, mature tooling, and have platform support.

**Prefect** *(Hour 1)* — Python-first, task-centric but with dynamic, runtime-shaped workflows and the lightest ops model. Strong failure-recovery story and fast script-to-production. Best for teams that want orchestration without much infrastructure overhead and for unpredictable/dynamic flows.

**Dagster** — asset-centric. You define **software-defined assets**: a Python function decorated to declare "this function produces *this* data asset, which depends on *those* assets." Dagster builds a lineage graph of your data, knows which assets are stale and what to run to refresh them, and treats data-quality checks as first-class. Excellent local development/testing. Best when *lineage and data quality* are central — data platforms and ML pipelines where you care about the datasets/models, not just step success.

```python
# Dagster's mental model (illustrative)
from dagster import asset

@asset
def raw_data():
    return fetch()

@asset
def clean_data(raw_data):       # depends on the raw_data asset
    return clean(raw_data)
# Dagster now knows clean_data is built from raw_data — lineage for free.
```

**Kubeflow Pipelines (KFP)** — Kubernetes-native ML pipelines. Each step is a **component**: a self-contained piece of code packaged as a *container* that runs in its own Kubernetes pod. You author components and pipelines with the KFP Python SDK (`@dsl.component`, `@dsl.pipeline`), compile to a YAML specification, and run it on a cluster. It's part of the broader Kubeflow ML platform (which also includes hyperparameter tuning and model serving). Best when you're *already on Kubernetes*, need per-step container isolation, reproducibility, and elastic scaling, and want an ML-specific tool.

```python
# Kubeflow's mental model (illustrative)
from kfp import dsl

@dsl.component
def preprocess(data: str) -> str: ...

@dsl.component
def train(data: str) -> float: ...

@dsl.pipeline(name="training")
def pipeline(data: str):
    p = preprocess(data=data)
    train(data=p.output)        # each step runs in its own container/pod
```

## 2.3 Decision matrix

| Factor | Airflow | Prefect | Dagster | Kubeflow Pipelines |
|---|---|---|---|---|
| Central concept | Tasks (DAG) | Tasks (dynamic flows) | **Assets** (data products) | Containerized ML steps |
| Authoring | Python DAGs | Plain Python + decorators | Python assets/ops | Python SDK → YAML, runs on K8s |
| Lineage / data quality | Add-on / improving | Add-on | **Built-in, first-class** | Artifact + metadata tracking |
| Ops burden | Heaviest | Lightest | Moderate | Heavy (needs a K8s cluster) |
| Infra requirement | VM/containers/managed | Anything (local→serverless) | Anything (hybrid control plane) | **Kubernetes required** |
| Ecosystem / integrations | **Largest (1,000+)** | Growing | Strong (esp. dbt) | ML-focused |
| Dynamic/runtime-shaped flows | Awkward | **Natural** | Supported | Supported |
| Sweet spot | Broad scheduled ETL, big teams | Pythonic, dynamic, small ops | Data platforms, lineage, ML assets | ML on existing Kubernetes |

## 2.4 How to actually choose (for ML / feature work)

- You already run Airflow and need more integrations or scheduled ETL → **stay on Airflow** (managed, if you can).
- You want the fastest path from a Python script to a scheduled pipeline with minimal infrastructure → **Prefect**.
- You're building a data/ML platform where dataset and model *lineage and quality* are the point → **Dagster**.
- You already operate Kubernetes and want container-isolated, reproducible, scalable ML steps → **Kubeflow Pipelines**.

> **Gotcha — don't choose Kubeflow without Kubernetes expertise.** KFP's power comes from running on Kubernetes, which is also its cost: you need a cluster and the skills to run it. For a small team without K8s, it's usually the wrong tool regardless of its ML features.

> **Gotcha — "newer = better" is a trap.** Prefect and Dagster have nicer developer experiences, but Airflow's ecosystem, community, and managed options are still unmatched. The right answer depends on team skills and existing infrastructure, not novelty.

> **Gotcha — task-centric vs asset-centric is hard to retrofit.** Choosing Dagster's asset model (or not) is a fundamental modeling decision. Migrating a large task-centric pipeline to an asset model — or vice versa — is real work, so decide deliberately rather than by default.

## ✅ Hour 2 Exercise

For each scenario, pick a tool and justify in two sentences (state your assumptions):

1. A 12-person startup wants nightly ETL and a couple of ML retraining jobs, with as little infrastructure to babysit as possible.
2. A data platform team needs full lineage of every dataset and model, plus data-quality gates, integrated tightly with dbt.
3. An ML org already running everything on a Kubernetes cluster wants reproducible, container-isolated training steps that scale elastically.
4. An enterprise with an existing platform team, hundreds of pipelines, and a need to connect to dozens of SaaS systems and clouds.

**Stretch:** for scenario 1, sketch the same three-step pipeline (ingest → train → register) in both Prefect (`@flow`/`@task`) and Dagster (`@asset`) pseudocode, and note which reads more naturally to you and why.

---

## 🎤 Hour 2 — Interview Questions & Answers

**Q1 (conceptual). Explain task-centric vs asset-centric orchestration with an example.**
Task-centric tools (Airflow, Prefect) model steps and their order — "extract, then transform, then load" — and don't inherently track the data produced. Asset-centric tools (Dagster) model the data products themselves: you declare that `clean_data` is built from `raw_data`, and the orchestrator tracks lineage, knows which assets are stale, and can rebuild exactly what's needed. The first asks "did the steps run?"; the second asks "is this dataset up to date, and what produced it?"

**Q2 (practical). When would you recommend Kubeflow Pipelines over Airflow or Prefect, and what's the prerequisite?**
When the team already runs on Kubernetes and wants ML steps that are containerized, isolated per step, reproducible, and elastically scalable — KFP runs each component in its own pod. The prerequisite is a Kubernetes cluster and the expertise to operate it; without that, KFP's overhead outweighs its benefits.

**Q3 (conceptual). What does Dagster give you that a task-centric orchestrator doesn't, out of the box?**
First-class data assets with automatic lineage, the ability to ask "what do I need to run to refresh this asset?", and built-in data-quality checks — plus strong local development and testing. Task-centric tools can approximate lineage with metadata add-ons but don't treat assets as the primary object.

**Q4 (gotcha). A team says "Airflow is old, we'll switch to Prefect for everything." What pushback is fair?**
Airflow's maturity, 1,000+ providers, large community, and managed offerings (MWAA, Composer, Astronomer) remain unmatched, especially for broad integrations and scheduled ETL at scale. Prefect's advantages (dynamic flows, lighter ops) are real but don't automatically beat Airflow; the choice should hinge on workflow type, team skills, and infrastructure, not age.

**Q5 (practical). For an ML feature pipeline specifically, what would tilt you toward Dagster vs Prefect?**
Dagster if you want to model and track the *features/datasets/models* as versioned assets with lineage and quality checks (helpful when many models share features). Prefect if you want the simplest Pythonic path to scheduled training with minimal infrastructure and don't need built-in asset lineage. Either can orchestrate the work; the difference is whether asset tracking is a first-class need.

---

# Hour 3 — Feast concepts

**Goal:** understand what a feature store is and the Feast data model. (This is a different category from the orchestrators — they *run* pipelines; Feast *manages the features* those pipelines produce and consume.)

## 3.1 Why a feature store exists

In ML, a **feature** is an input variable a model learns from (e.g., a driver's average trips per day). Two hard problems show up once you go to production:

1. **Training–serving skew.** You compute features one way in a training script (batch, over historical data) and another way in the live service (real-time, per request). If those two computations drift apart, the model sees different inputs in production than it trained on, and quietly degrades.
2. **Point-in-time correctness (avoiding data leakage).** When you build a training set, each example must use only the feature values that were known *at that example's moment in time*. If a feature value computed *after* the prediction time leaks into training, the model effectively "sees the future," scores great offline, and fails in production.

A **feature store** is a system that solves both: it manages features consistently for *training* (historical, batch) and *serving* (real-time, low-latency), and it does the time-correct joins for you. **Feast** is an open-source feature store: a Python SDK plus storage abstractions, a registry, an optional serving server, a CLI, and a UI. (Note: Feast is *not* an ETL tool — it doesn't compute your features; you compute them upstream and point Feast at where they live.)

## 3.2 The two stores

Feast manages two kinds of storage, and the distinction is the heart of the tool:

- **Offline store** — holds large amounts of *historical* feature data, used to build training datasets and for batch scoring. Locally this is files (Parquet); in production it's a warehouse like BigQuery, Snowflake, or Redshift. Optimized for big, time-ranged reads.
- **Online store** — holds the *latest* feature value per entity, used for *real-time* inference where you need a feature in milliseconds. Locally this is SQLite; in production it's a low-latency store like Redis or DynamoDB. Optimized for fast key lookups.

The same feature exists in both: history in the offline store for training, latest value in the online store for serving. Keeping them consistent is what kills training–serving skew.

## 3.3 The Feast data model (define each term)

- **Entity** — the object that features describe, identified by a **join key**. For a ride-share model, the entity is `driver`, with join key `driver_id`. The join key is how Feast knows which rows belong to which driver.

- **Data source** — where the raw feature data physically lives (e.g., a Parquet file, a warehouse table). Each data source has an **event timestamp** column recording *when* each feature value was observed — this is what makes point-in-time joins possible.

- **Feature view** — a logical group of related features coming from one data source, tied to an entity. It declares the feature names and types (its **schema**), the entity it's keyed on, the source, and a **TTL** (see below). It's the unit you define and register. Roughly: "these features, about this entity, from this source."

- **Field** — one feature inside a feature view: a name plus a data type (e.g., `Field(name="avg_daily_trips", dtype=Int64)`).

- **Feature service** — a named bundle of features (often drawn from several feature views) that a *specific model* needs. It decouples "all the features that exist" from "the features model X uses," so you can version a model's feature set.

- **Registry** — the catalog of all your definitions (entities, data sources, feature views, feature services). Locally it's a file; it's the single source of truth for what exists. You populate it by running `feast apply`.

- **TTL (time-to-live)** — how far back in time Feast is allowed to look when joining a feature value to a point in time. Crucially, TTL is measured *relative to each entity row's timestamp*, not relative to "now." Set it too short and valid-but-older feature values won't join (you'll get nulls).

## 3.4 Point-in-time joins (the killer feature)

To build a training set you give Feast an **entity dataframe**: a table with the entity keys, an `event_timestamp` for each row (the moment you want to "reproduce the world" for), and usually the label you're predicting. Feast then performs a **point-in-time join**: for each row, it finds the *latest feature value at or before that row's timestamp* (within the TTL window) and attaches it.

That "at or before" is the whole point — it guarantees no feature value generated *after* the prediction moment can leak into the row, which is what prevents the inflated-offline-metrics trap. Feast calls this a "last known good value" join.

```
entity row:  driver_id=1001, event_timestamp=2021-04-12 10:00
feature data: 1001 had avg_daily_trips=11 recorded at 09:45, and =14 at 10:30
result:       Feast joins 11 (the 09:45 value) — the 10:30 value is "in the future" and excluded
```

## 3.5 The Feast lifecycle (preview of Hour 4)

Four steps, in order:

1. **Define** entities, data sources, and feature views in Python.
2. **`feast apply`** — register those definitions in the registry and set up the online store tables.
3. **`feast materialize`** (or `materialize-incremental`) — load the latest feature values from the offline store into the online store, so real-time serving has something to read.
4. **Retrieve** — `get_historical_features` for training (point-in-time join over the offline store) and `get_online_features` for inference (fast lookup from the online store).

> **Gotcha — the online store is empty until you materialize.** A frequent first-timer surprise: you run `get_online_features` right after `apply` and get nothing. `apply` only registers definitions; you must `materialize` to actually load values into the online store before online reads return data.

> **Gotcha — your entity dataframe must include `event_timestamp`.** Without a timestamp per row, Feast can't do a point-in-time join. The column is required for `get_historical_features`.

> **Gotcha — Feast computes nothing.** It doesn't transform or aggregate your raw data into features. You produce features with your own pipeline (perhaps one of the orchestrators from Hour 2) and register the *result* with Feast.

## ✅ Hour 3 Exercise

No code yet — build the mental model:

1. For a churn-prediction model on customers, name the **entity** and its likely **join key**.
2. List three plausible **features** and which **feature view** they'd live in.
3. Explain, in your own words, why the **entity dataframe** needs a timestamp column and what would go wrong without it.
4. A teammate sets `ttl=timedelta(hours=1)` and most rows come back null in their training set, even though plenty of feature data exists. Give the most likely cause.
5. State which store (online vs offline) serves *training* and which serves *real-time inference*, and why each is suited to its job.

---

## 🎤 Hour 3 — Interview Questions & Answers

**Q1 (conceptual). What two problems does a feature store solve, and how?**
Training–serving skew (features computed inconsistently between training and serving) and point-in-time correctness/data leakage. A feature store solves the first by managing the same features for both offline training and online serving through one definition, and the second by performing point-in-time joins that attach only feature values known at or before each example's timestamp.

**Q2 (conceptual). Explain the difference between the online and offline stores.**
The offline store holds large historical feature data for building training sets and batch scoring (files locally; a warehouse in production), optimized for big time-ranged reads. The online store holds the latest value per entity for low-latency real-time inference (SQLite locally; Redis/DynamoDB in production), optimized for fast key lookups. The same feature lives in both — history offline, latest online.

**Q3 (conceptual). What is a point-in-time join and why is it essential?**
For each entity row with a timestamp, Feast joins the latest feature value recorded at or before that timestamp (within TTL). It's essential because using any value generated after the prediction time would leak future information into training, producing inflated offline metrics that collapse in production. It reproduces the state of the world as of each example's moment.

**Q4 (practical). Walk through the Feast objects you'd define for a driver model and how they relate.**
Define an entity (`driver`, join key `driver_id`); a data source pointing at the feature data with its event-timestamp column; one or more feature views grouping related features (each with schema Fields, the entity, the source, and a TTL); optionally a feature service bundling the features a specific model uses. Running `feast apply` registers all of these in the registry.

**Q5 (gotcha). Right after `feast apply`, `get_online_features` returns nothing. What's the cause and fix?**
`apply` only registers definitions and sets up online-store tables; it doesn't load data. The online store is empty until you run `feast materialize` (or `materialize-incremental`) to push the latest values from the offline store into it. After materializing, online reads return values.

---

# Hour 4 — Feast hands-on

**Goal:** stand up a local Feast store, register and load features, and retrieve them for training and for inference.

## 4.1 Scaffold a project

Feast ships a working example you can learn from:

```bash
feast init my_feature_repo
cd my_feature_repo/feature_repo
```

This creates a ready-to-run project. Look at the three things that matter:

- **`feature_store.yaml`** — the config. For a local project it sets `provider: local`, an **online store** of type `sqlite`, and a local **registry** file. This is where you'd later point at Redis/BigQuery/etc.
- **`example_repo.py`** — the Python definitions (entity, data source, feature view, feature service) for a sample `driver_hourly_stats` dataset.
- **`data/`** — a Parquet file of sample feature data acting as the offline store.

> **Gotcha — run commands from the folder with `feature_store.yaml`.** Feast finds your project via that file. Running `feast apply` from the wrong directory either errors or operates on the wrong project. (`FeatureStore(repo_path=".")` follows the same rule.)

## 4.2 The definitions (what's in `example_repo.py`)

The sample defines roughly this — the shape you'll reuse for any project:

```python
from datetime import timedelta
from feast import Entity, FeatureView, Field, FileSource, FeatureService
from feast.types import Float32, Int64

# 1) the object features describe
driver = Entity(name="driver", join_keys=["driver_id"])

# 2) where raw feature data lives + its event-timestamp column
driver_stats_source = FileSource(
    path="data/driver_stats.parquet",
    timestamp_field="event_timestamp",
)

# 3) a logical group of features about the driver, from that source
driver_stats_fv = FeatureView(
    name="driver_hourly_stats",
    entities=[driver],
    ttl=timedelta(days=1),                 # how far back point-in-time joins may look
    schema=[
        Field(name="conv_rate", dtype=Float32),
        Field(name="acc_rate", dtype=Float32),
        Field(name="avg_daily_trips", dtype=Int64),
    ],
    source=driver_stats_source,
    online=True,                            # make available to the online store
)

# 4) the bundle a specific model consumes
driver_activity = FeatureService(
    name="driver_activity_v1",
    features=[driver_stats_fv],
)
```

## 4.3 Register and load

```bash
feast apply
```
This reads your definitions, writes them to the registry, and creates the online-store tables. (Run it again any time you change definitions.)

```bash
# load the latest feature values into the online store, up to "now"
feast materialize-incremental $(date -u +"%Y-%m-%dT%H:%M:%S")
```
`materialize-incremental` loads everything new since the last materialize (or, first time, back by the feature view's TTL). This is the step that makes online serving possible.

> **Gotcha — re-run `feast apply` after editing definitions.** Changing `example_repo.py` doesn't take effect until you re-apply. Forgetting this is why "my new feature isn't showing up."

## 4.4 Retrieve for training (offline, point-in-time)

Create `retrieve_training.py` in the same folder:

```python
from datetime import datetime
import pandas as pd
from feast import FeatureStore

store = FeatureStore(repo_path=".")

# the entity dataframe: who, at what moment, and (optionally) the label
entity_df = pd.DataFrame.from_dict({
    "driver_id": [1001, 1002, 1003],
    "event_timestamp": [
        datetime(2021, 4, 12, 10, 59, 42),
        datetime(2021, 4, 12, 8, 12, 10),
        datetime(2021, 4, 12, 16, 40, 26),
    ],
    "label": [1, 0, 1],     # what you're predicting; Feast just carries it through
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "driver_hourly_stats:conv_rate",
        "driver_hourly_stats:acc_rate",
        "driver_hourly_stats:avg_daily_trips",
    ],
).to_df()

print(training_df)
```

Run `python retrieve_training.py`. Each row gets the feature values that were current *at or before* its `event_timestamp` — a point-in-time join, leakage-free. You'd then train a model on `training_df` (features as inputs, `label` as target) exactly as in your earlier MLflow work.

> **Gotcha — rows can come back with nulls.** If an entity row's timestamp is older than any feature data, or older than the TTL window relative to that row, there's nothing valid to join and you get nulls. That's correct behavior (no leakage), not a bug — check your timestamps and TTL.

## 4.5 Retrieve for inference (online, latest value)

Create `retrieve_online.py`:

```python
from feast import FeatureStore

store = FeatureStore(repo_path=".")

features = store.get_online_features(
    features=[
        "driver_hourly_stats:conv_rate",
        "driver_hourly_stats:acc_rate",
        "driver_hourly_stats:avg_daily_trips",
    ],
    entity_rows=[{"driver_id": 1001}],   # who we're scoring right now
).to_dict()

print(features)
```

Run `python retrieve_online.py`. This is the *latest* value per feature for driver 1001, read fast from the online store — what a live prediction service would do at request time. (If it's empty, you skipped `materialize` — see Hour 3's gotcha.)

## 4.6 Inspect it

```bash
feast ui      # browse entities, feature views, and feature services in a web UI
```

## ✅ Hour 4 Exercise

Working in `my_feature_repo/feature_repo`:

1. Run the full lifecycle: `feast apply` → `feast materialize-incremental <now>` → both retrieval scripts. Confirm training retrieval returns a joined dataframe and online retrieval returns the latest values.
2. **Add a feature.** Edit the feature view to add one more `Field` (use a column that exists in the sample data, or add a column to the Parquet first). Re-run `feast apply` and re-materialize, then retrieve it.
3. **Prove point-in-time correctness.** In the entity dataframe, set one row's `event_timestamp` to *before* any feature data exists. Re-run training retrieval and confirm that row's features are null — explain why in a comment.
4. **Train on it.** Extend `retrieve_training.py` to fit a simple scikit-learn classifier on the retrieved features and `label`, printing accuracy — connecting Feast (features) to a model end-to-end.
5. **Reflect (one sentence each):** which store did step 1's training retrieval read from, which did online retrieval read from, and what would change in `feature_store.yaml` to move the online store to Redis in production.

---

## 🎤 Hour 4 — Interview Questions & Answers

**Q1 (practical). Walk through the Feast workflow from definitions to a served feature.**
Define entities, data sources, and feature views in Python; run `feast apply` to register them and create online-store tables; run `feast materialize`/`materialize-incremental` to load the latest values into the online store; then retrieve — `get_historical_features` (point-in-time join over the offline store) for training, and `get_online_features` (fast lookup) for real-time inference.

**Q2 (practical). What goes in the entity dataframe for `get_historical_features`, and what does Feast do with it?**
The entity keys, an `event_timestamp` per row, and optionally the label. For each row, Feast performs a point-in-time join, attaching the latest feature values recorded at or before that row's timestamp within the TTL window — producing a leakage-free training set.

**Q3 (gotcha). Online retrieval returns empty values right after `feast apply`. Why?**
`apply` only registers definitions and creates online-store tables; it doesn't load data. You must run `materialize`/`materialize-incremental` to push the latest values into the online store before online reads return anything.

**Q4 (gotcha). Some training rows come back with null features even though plenty of data exists. Give two possible causes.**
(1) The row's `event_timestamp` is earlier than any available feature value, so there's nothing valid to join. (2) The valid value is older than the feature view's TTL relative to that row's timestamp, so it falls outside the look-back window. Both are correct, leakage-preventing behavior; fix by adjusting timestamps or TTL.

**Q5 (conceptual). How does Feast relate to an orchestrator like Airflow or Prefect — does it replace one?**
No. Feast manages features (storage, point-in-time retrieval, online/offline consistency) but computes nothing itself. An orchestrator runs the pipeline that produces features and calls Feast (`materialize`, retrieval) on a schedule. They're complementary: the orchestrator runs the work; Feast manages the feature data that work produces and consumes.

---

# 📋 Gotchas summary table

| # | Area | Gotcha | Fix |
|---|------|--------|-----|
| 1 | Prefect | "Agents" are deprecated | Use work pools + workers (`prefect worker start --pool`) |
| 2 | Prefect | `retry_delay` (Airflow's name) | Prefect uses `retry_delay_seconds` |
| 3 | Prefect | Calling a flow runs it immediately | Use `flow.serve()` / deployments to schedule |
| 4 | Prefect | No backend = no run tracking | Run `prefect server start`, set `PREFECT_API_URL` |
| 5 | Prefect | Pausing a work pool doesn't pause schedules | Runs queue and execute on resume; pause the schedule too |
| 6 | Choosing | Picking Kubeflow without Kubernetes/skills | KFP needs a cluster; pick a lighter tool otherwise |
| 7 | Choosing | "Newer = better" | Weigh ecosystem, team skills, infra — not novelty |
| 8 | Choosing | Asset vs task model retrofit | Decide the modeling philosophy up front; migration is costly |
| 9 | Feast | Online store empty until materialized | `feast materialize-incremental <now>` before online reads |
| 10 | Feast | Entity dataframe missing `event_timestamp` | Required for point-in-time joins; include it |
| 11 | Feast | Edits not taking effect | Re-run `feast apply` after changing definitions |
| 12 | Feast | TTL too short → null features | TTL is relative to each row's timestamp; widen it if valid data is excluded |
| 13 | Feast | Expecting Feast to compute features | Feast stores/serves features; compute them upstream |
| 14 | Feast | Running CLI from wrong directory | Run where `feature_store.yaml` lives |

---

# 🗂️ Quick reference card

### Prefect — core
```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=5)
def step(x): return x * 2

@flow(log_prints=True)
def pipeline():
    return step(10)

pipeline()                                   # run now
# pipeline.serve(name="daily", cron="0 6 * * *")   # schedule
# [step.submit(i) for i in range(5)]               # concurrent
```
```bash
prefect server start                         # self-hosted backend, UI :4200
prefect work-pool create my-pool --type process
prefect worker start --pool my-pool
prefect deploy
```

### Orchestrator one-liners
| Tool | Reach for it when… |
|------|--------------------|
| Airflow | Broad integrations, scheduled ETL, big team + platform support |
| Prefect | Pythonic, dynamic flows, minimal ops |
| Dagster | Asset lineage + data quality are central (data/ML platforms) |
| Kubeflow | Already on Kubernetes; container-isolated, scalable ML steps |

### Feast — lifecycle
```bash
feast init my_repo && cd my_repo/feature_repo
feast apply                                  # register definitions
feast materialize-incremental $(date -u +"%Y-%m-%dT%H:%M:%S")
feast ui                                     # browse
```
```python
from feast import FeatureStore
store = FeatureStore(repo_path=".")

# training (offline, point-in-time): entity_df needs entity keys + event_timestamp
train_df = store.get_historical_features(entity_df=entity_df, features=[
    "driver_hourly_stats:conv_rate", "driver_hourly_stats:avg_daily_trips",
]).to_df()

# inference (online, latest value): materialize first!
feats = store.get_online_features(
    features=["driver_hourly_stats:conv_rate"],
    entity_rows=[{"driver_id": 1001}],
).to_dict()
```

### Feast — definition skeleton
```python
from datetime import timedelta
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float32, Int64

driver = Entity(name="driver", join_keys=["driver_id"])
src = FileSource(path="data/driver_stats.parquet", timestamp_field="event_timestamp")
fv = FeatureView(name="driver_hourly_stats", entities=[driver],
                 ttl=timedelta(days=1), source=src, online=True,
                 schema=[Field(name="conv_rate", dtype=Float32),
                         Field(name="avg_daily_trips", dtype=Int64)])
```

### Feast vocabulary
Entity = object features describe (has a join key) · Data source = where raw feature data lives (has an event timestamp) · Feature view = group of features about an entity, from a source · Field = one feature (name + type) · Feature service = the feature bundle a model uses · Online store = latest values, fast, for serving · Offline store = history, for training · Registry = catalog of definitions · TTL = how far back a point-in-time join may look (relative to each row's timestamp).

---

*Feast and the orchestrators are complementary: an orchestrator (Airflow, Prefect, Dagster, or Kubeflow) runs the pipeline; Feast manages the features it produces and serves. Re-run each hour's exercise until the new models feel as familiar as the Airflow one you started with.*
