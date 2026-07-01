# Capstone: Build an End-to-End ML Pipeline — A 6-Hour Project Guide

*You'll plan, build, and debug a complete training pipeline that ingests and validates data, engineers features through a feature store, trains and logs a model, and promotes it only if it clears a quality bar. This is the capstone that ties your earlier tutorials together.*

## What this assumes you already know

From the earlier tutorials, you're comfortable with:

- **Airflow 3** — `@dag`/`@task` (TaskFlow), `>>`, retries, `on_failure_callback`, **branching** (`@task.branch`) and **trigger rules** (`none_failed_min_one_success`, `all_done`), XCom (small values / pass file paths), **Connections** and **Variables**, the `airflow.sdk` / `airflow.providers.standard.*` import style.
- **Feast** — entities, feature views, online vs offline stores, `feast apply`, `feast materialize`, `get_historical_features` (point-in-time joins).
- **MLflow** — tracking server, experiments/runs, `log_param`/`log_metric`/`log_model`.

This guide does not re-explain those. It focuses on stitching them into one pipeline and on the new pieces below.

## What's new here

1. **Project planning** — how to scope an ML pipeline before writing code, and sketch the DAG.
2. **Great Expectations** — declaring and running data-quality checks (a major-version-1 tool, taught with its current Fluent API).
3. **S3** — saving artifacts to object storage from an Airflow task.
4. **Model signatures** — recording a model's input/output schema as a contract.
5. **Promotion with aliases & tags** — MLflow's current mechanism (replacing the old "stages") to mark a model as production-ready only when it passes.
6. **End-to-end debugging** — using the Airflow UI to fix a pipeline until it runs clean.

### One-time setup

Add to your Astro project's `requirements.txt` and rebuild (`astro dev restart`):

```text
apache-airflow-providers-amazon      # S3 access from Airflow (S3Hook)
great-expectations>=1.7              # data validation (1.x Fluent API)
feast                                # feature store
mlflow                               # experiment tracking + registry
scikit-learn
pandas
pyarrow                              # parquet
s3fs                                 # lets pandas read/write s3:// paths
```

Run an MLflow tracking server in a separate terminal (a database-backed store is required for the model **registry**, so we point it at a local SQLite file):

```bash
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlartifacts \
  --host 0.0.0.0 --port 5000
```

> **Gotcha — the registry needs a real backend store.** If you run `mlflow server` with the default file store (no `--backend-store-uri`), model **registration and aliases won't work** — you'll get errors when you try to register or alias a model. The `sqlite:///mlflow.db` backend above is the minimum that enables the registry. (Recall from earlier: from inside Astro's Docker, reach this server at `http://host.docker.internal:5000`, not `localhost`.)

---

# Hour 1 — Project planning

**Goal:** decide *what* you're building before *how*. A pipeline you can't describe in one paragraph is one you'll struggle to debug in Hour 6.

## 1.1 Why planning is a real step

It's tempting to start writing tasks immediately. But an ML pipeline has choices that are expensive to change later: the problem type dictates the model and metric; the dataset dictates the feature-store entity; the success metric dictates the promotion gate. Spending the first hour writing a short spec and sketching the DAG saves you from rebuilding tasks in Hour 6.

## 1.2 The five planning questions

Answer these before touching code:

1. **What's the problem, and what type is it?** Classification (predict a category) or regression (predict a number)? This sets your model family and your metric.
2. **What dataset, and where does it live?** Pick something clean and small enough to iterate on. Good sources: Kaggle datasets and the UCI Machine Learning Repository (a long-running public collection of datasets for ML).
3. **What's the entity?** The thing each row/prediction is about — this becomes your Feast entity and its join key (e.g., a customer, identified by `customer_id`).
4. **What's the target and the success metric, with a number?** Not "good accuracy" — a concrete bar like "ROC-AUC ≥ 0.80 on the holdout set." This becomes your promotion threshold in Hour 5.
5. **What can go wrong with the data?** List the data-quality rules that must hold (no null IDs, charges ≥ 0, known set of categories). These become your Great Expectations checks in Hour 2.

## 1.3 The worked example we'll build

To make the rest concrete, this guide uses one running example. (Swap in your own dataset by re-answering the five questions.)

- **Problem:** predict customer churn (will a customer leave?) — **binary classification**.
- **Dataset:** the *Telco Customer Churn* dataset (a single CSV of ~7,000 customers, widely available on Kaggle). A clean alternative on UCI is *Bank Marketing*.
- **Entity:** `customer`, join key `customer_id`.
- **Target & metric:** target `churn` (yes/no); promote only if **ROC-AUC ≥ 0.80** on the holdout. (ROC-AUC is a classification score from 0.5 = random to 1.0 = perfect; it's robust when classes are imbalanced, which churn usually is.)
- **Data risks:** null `customer_id`, negative `tenure` or `charges`, `contract` outside the known set of values.

> **Gotcha — static datasets have no timestamps, but feature stores need them.** Feast point-in-time joins require an event timestamp per row, and a downloaded snapshot like Telco churn has none. We'll **synthesize an `event_timestamp`** during ingestion (Hour 2). That's a normal teaching workaround; real feature data carries real timestamps from when each value was observed.

## 1.4 Sketch the end-to-end DAG

Before coding, draw the shape. Ours:

```
ingest_and_validate → materialize_features → train → evaluate ─▶ (branch)
                                                                   ├─ promote   ─┐
                                                                   └─ reject     ─┴─▶ finalize → cleanup
```

- **ingest_and_validate** — pull the raw data, run Great Expectations, add timestamps, save to S3. (Hour 2)
- **materialize_features** — register/refresh the Feast feature view so features are queryable. (Hour 3)
- **train** — build the training set from Feast (point-in-time), fit the model, log to MLflow with a signature, register a new version. (Hour 4)
- **evaluate** — score on holdout; **branch** on the threshold. (Hour 5)
- **promote / reject** — set the `champion` alias + tags, or tag as rejected. (Hour 5)
- **finalize / cleanup** — always-run wrap-up. (Hour 6)

Translate the sketch into a runnable skeleton now (empty tasks, real wiring) — you'll fill each task in later:

```python
# dags/churn_pipeline.py
from datetime import datetime
from airflow.sdk import dag, task

@dag(start_date=datetime(2024, 1, 1), schedule=None, catchup=False, tags=["capstone"])
def churn_pipeline():

    @task
    def ingest_and_validate() -> str: ...        # returns S3 path of validated data

    @task
    def materialize_features(data_path: str) -> str: ...   # returns feature-repo info

    @task
    def train(_: str) -> dict: ...               # returns {run_id, version, ...}

    @task
    def evaluate(train_info: dict) -> dict: ...   # returns {metric, passed, ...}

    @task.branch
    def gate(eval_info: dict) -> str:
        return "promote" if eval_info["passed"] else "reject"

    @task
    def promote(train_info: dict, eval_info: dict): ...

    @task
    def reject(train_info: dict, eval_info: dict): ...

    @task(trigger_rule="none_failed_min_one_success")
    def finalize(): ...

    @task(trigger_rule="all_done")
    def cleanup(): ...

    data = ingest_and_validate()
    feats = materialize_features(data)
    t = train(feats)
    e = evaluate(t)
    decision = gate(e)
    p, r = promote(t, e), reject(t, e)
    decision >> [p, r]
    [p, r] >> finalize() >> cleanup()

churn_pipeline()
```

Drop it in `dags/`, trigger it, confirm the empty tasks run and the graph matches your sketch. A correct skeleton now means Hour 6 is about logic bugs, not wiring bugs.

## ✅ Hour 1 Exercise

Write a one-paragraph spec for *your* pipeline by answering the five questions (Section 1.2), then commit a skeleton DAG like the one above with your task names and dependencies. Trigger it and confirm every empty task turns green in the order you intended. Keep the spec at the top of the DAG file as a docstring — your future self debugging in Hour 6 will thank you.

---

## 🎤 Hour 1 — Interview Questions & Answers

**Q1 (conceptual). Why decide the success metric and threshold during planning rather than after training?**
Because the threshold defines the pipeline's promotion gate and shapes the evaluate task, and the metric choice affects how you train and what you optimize. Deciding "ROC-AUC ≥ 0.80" up front gives an objective, pre-committed bar; deciding it after seeing results invites moving the goalposts to make a mediocre model look acceptable.

**Q2 (practical). How does the dataset choice influence the feature-store design?**
The thing each row is about becomes the entity and its join key (e.g., `customer` / `customer_id`), which every feature view is keyed on. If the dataset lacks event timestamps, you must add them, because point-in-time joins depend on them. So the dataset determines the entity, the join key, and whether timestamps need synthesizing.

**Q3 (conceptual). Why build a skeleton DAG with empty tasks before implementing any logic?**
It separates wiring bugs from logic bugs. Getting dependencies, branch targets, and trigger rules correct on empty tasks means that when tasks fail later, you know it's the task's logic, not the graph. It also validates your mental sketch against what Airflow actually parses.

**Q4 (practical). You're choosing between accuracy and ROC-AUC for a churn model. What guides the choice?**
Class balance and how the prediction is used. Churn data is usually imbalanced (most customers don't churn), where accuracy can look high by always predicting "no churn." ROC-AUC measures ranking quality across thresholds and is more informative under imbalance, so it's the better promotion metric here.

**Q5 (gotcha). What's the risk of skipping the "what can go wrong with the data" planning step?**
You'll discover data problems only when training produces a bad model or a task crashes deep in the pipeline. Listing data risks up front turns them into explicit validation checks (Hour 2) that fail fast at ingestion with a clear message, instead of corrupting everything downstream.

---

# Hour 2 — Data ingestion + validation

**Goal:** build the first real task — pull the data, **validate** it, add timestamps, and save it to S3 — so nothing bad flows downstream.

## 2.1 Why validate before anything else

A model is only as good as its data, and bad data fails without notice: a column that's suddenly all nulls, charges that went negative after a billing bug, a new category your encoder never saw. If you don't check, the pipeline runs "successfully" and produces a broken model. **Data validation** means asserting, in code, the properties your data must have — and failing the task loudly when they don't hold. That's the whole idea behind catching problems at the door.

## 2.2 What Great Expectations is

**Great Expectations (GX)** is an open-source Python library for data validation. You write **expectations** — declarative statements about your data, like "this column has no nulls" or "these values are between 0 and 100." A group of expectations is an **expectation suite**, and running a suite against a batch of data tells you which passed and which failed.

> **Version warning — GX changed a lot.** GX had a major rewrite at **version 1.0**; its API differs significantly from the 0.x examples you'll find in most blog posts (the old `DataContext` + YAML + `ValidationOperators` flow is gone). This guide uses the **GX 1.x Fluent (Python) API**. Pin `great-expectations>=1.7` and treat older tutorials with suspicion. If a GX snippet errors on method names, a version mismatch is almost always why.

## 2.3 Validating a DataFrame with GX (1.x)

The cleanest approach for a pipeline task is to validate a pandas DataFrame in memory. The 1.x flow: get a context, register the DataFrame as a data source/asset, define a batch, attach expectations, and validate.

```python
import great_expectations as gx

def validate_df(df) -> bool:
    context = gx.get_context()                       # ephemeral, in-memory context

    # tell GX about the data
    source = context.data_sources.add_pandas("churn_src")
    asset = source.add_dataframe_asset(name="churn_asset")
    batch_def = asset.add_batch_definition_whole_dataframe("churn_batch")

    # declare what we expect (from our Hour 1 data-risk list)
    suite = context.suites.add(gx.ExpectationSuite(name="churn_suite"))
    suite.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(column="customer_id"))
    suite.add_expectation(gx.expectations.ExpectColumnValuesToBeBetween(column="tenure", min_value=0, max_value=120))
    suite.add_expectation(gx.expectations.ExpectColumnValuesToBeBetween(column="MonthlyCharges", min_value=0))
    suite.add_expectation(gx.expectations.ExpectColumnValuesToBeInSet(
        column="Contract", value_set=["Month-to-month", "One year", "Two year"]))

    # run validation against this DataFrame
    batch = batch_def.get_batch(batch_parameters={"dataframe": df})
    result = batch.validate(suite)
    return bool(result.success)        # True only if every expectation passed
```

The `result` object also contains per-expectation detail (which failed, and the offending values) — useful for logging when a check trips.

> **Gotcha — production uses the GX Airflow operators.** For a real deployment, the `airflow-provider-great-expectations` package (v1.x) provides dedicated operators that run a GX **Checkpoint** as an Airflow task with proper result tracking and "Data Docs" (GX's HTML validation reports). We validate inline here because it's easier to learn and debug in one task; mention the operators in interviews as the production-grade option.

## 2.4 Saving to S3

**S3 (Amazon Simple Storage Service)** is a cloud service that stores files ("objects") in named buckets — the standard place to park datasets and artifacts in a pipeline so every task and machine can reach them. Airflow talks to S3 through a **Connection** (id `aws_default` by default) and the **S3Hook** (a hook is the reusable client for an external system).

```python
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

def save_to_s3(local_path: str, bucket: str, key: str):
    hook = S3Hook(aws_conn_id="aws_default")     # credentials live in the Connection, not code
    hook.load_file(filename=local_path, bucket_name=bucket, key=key, replace=True)
    return f"s3://{bucket}/{key}"
```

> **Gotcha — no AWS handy? Use a local stand-in.** If you don't have S3 credentials, you have two beginner-friendly options: (1) write to a local path (e.g., `include/data/churn.parquet`) and treat it as "S3" for the course, or (2) run **MinIO** or **LocalStack** (local, S3-compatible servers) and point `aws_default` at them. The pipeline logic is identical; only the storage target changes. The code below shows a local fallback so you can run today.

## 2.5 The ingestion task

Putting it together — pull, validate, add the timestamp Feast will need, and persist. If validation fails, raise so the task (and pipeline) stops loudly.

```python
@task(retries=2)
def ingest_and_validate() -> str:
    import pandas as pd
    from datetime import datetime, timezone

    # 1) pull raw data (point at your downloaded CSV, or a URL/S3 source)
    df = pd.read_csv("include/data/telco_churn.csv")
    df = df.rename(columns={"customerID": "customer_id"})

    # 2) validate — fail loudly if the data is wrong
    if not validate_df(df):
        raise ValueError("Data validation failed — see GX result detail in logs.")

    # 3) add the event_timestamp Feast needs (synthetic for this static dataset)
    now = datetime.now(timezone.utc)
    df["event_timestamp"] = now
    df["created"] = now

    # 4) persist as parquet (local stand-in for S3; swap in save_to_s3 for real S3)
    out = "include/data/churn_validated.parquet"
    df.to_parquet(out, index=False)
    # production: return save_to_s3(out, bucket="my-bucket", key="churn/validated.parquet")
    return out
```

## ✅ Hour 2 Exercise

1. Download your dataset into `include/data/` and implement `validate_df` with **at least four** expectations drawn from your Hour 1 data-risk list.
2. Implement `ingest_and_validate` to pull, validate, add `event_timestamp`, and save parquet. Trigger the DAG and confirm it produces the file.
3. **Make it fail on purpose:** temporarily inject a bad row (e.g., set one `customer_id` to null), re-run, and confirm the task fails with your validation error — then read the GX detail in the task log to see which expectation tripped. Remove the bad row.
4. **Stretch:** swap the local save for a real S3 write using an `aws_default` Connection (or MinIO/LocalStack), and confirm the object appears in the bucket.

---

## 🎤 Hour 2 — Interview Questions & Answers

**Q1 (conceptual). What problem does data validation solve that model evaluation doesn't?**
Validation catches bad *inputs* before training, failing fast with a specific reason (null IDs, out-of-range values, unknown categories). Model evaluation only tells you the *output* is bad, after you've spent compute training on corrupt data — and a model trained on bad data can still post a passable score while being wrong. Validation prevents garbage-in at the door.

**Q2 (practical). Walk through validating a DataFrame with Great Expectations 1.x.**
Get a context, register the DataFrame as a pandas data source and dataframe asset, define a whole-dataframe batch, create an expectation suite and add expectations (not-null, value ranges, value sets), then call `batch.validate(suite)` and check `result.success`. The result also carries per-expectation detail for logging which check failed and on what values.

**Q3 (conceptual). Why store the validated data in S3 instead of passing it through XCom?**
The dataset is large, and XCom is for small values stored in Airflow's metadata database; pushing a dataset through it bloats the DB and slows the scheduler. S3 (or any shared object store) holds the data, and the task passes only the path/URI through XCom — every downstream task and worker can then read it.

**Q4 (practical). Where do AWS credentials for the S3 write belong, and how does the task use them?**
In an Airflow Connection (e.g., `aws_default`), never hard-coded in the DAG. The task uses `S3Hook(aws_conn_id="aws_default")`, which pulls the credentials from that Connection. This keeps secrets out of version-controlled code and lets you change credentials per environment without editing the pipeline.

---

# Hour 3 — Feature engineering with Feast

**Goal:** turn the validated data into queryable, time-correct features by wiring a Feast feature view and materializing it.

## 3.1 Why route features through a feature store here

You could read the parquet and train directly — but then training and any future serving would compute features differently, risking training–serving skew, and you'd have to redo point-in-time correctness by hand. Routing through **Feast** (which you know from the earlier tutorial) gives you one definition of each feature, a correct point-in-time join for the training set, and a path to low-latency serving later. In this pipeline, Feast sits between ingestion and training.

## 3.2 Define the feature repo

In your project (e.g., `include/feature_repo/`), define the entity, the data source pointing at the validated parquet, and the feature view. (This is the same shape from the Feast tutorial, applied to churn.)

```python
# include/feature_repo/churn_features.py
from datetime import timedelta
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Int64, Float32, String

customer = Entity(name="customer", join_keys=["customer_id"])

churn_source = FileSource(
    path="data/churn_validated.parquet",     # the file written in Hour 2
    timestamp_field="event_timestamp",
    created_timestamp_column="created",
)

customer_features = FeatureView(
    name="customer_features",
    entities=[customer],
    ttl=timedelta(days=3650),                 # wide TTL since timestamps are synthetic
    schema=[
        Field(name="tenure", dtype=Int64),
        Field(name="MonthlyCharges", dtype=Float32),
        Field(name="Contract", dtype=String),
    ],
    source=churn_source,
    online=True,
)
```

> **Gotcha — TTL vs synthetic timestamps.** Because Hour 2 stamped every row with "now," a short TTL plus a training `event_timestamp` slightly in the past could exclude all rows (point-in-time join finds nothing within the window) and return nulls. A wide TTL avoids that for this teaching dataset. With real, varied timestamps you'd set TTL to a meaningful business window.

## 3.3 The feature-engineering task

The task applies the definitions and materializes the latest values so features are available offline (for training) and online (for future serving).

```python
@task
def materialize_features(data_path: str) -> str:
    import subprocess
    from datetime import datetime, timezone

    repo = "include/feature_repo"
    # register/refresh definitions in the registry + create online tables
    subprocess.run(["feast", "apply"], cwd=repo, check=True)
    # load latest values into the online store, up to now
    now = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S")
    subprocess.run(["feast", "materialize-incremental", now], cwd=repo, check=True)
    return repo
```

Using the Feast CLI via `subprocess` keeps the task simple and mirrors what you'd run by hand. (You can also drive Feast through its Python `FeatureStore` API; either is fine.)

> **Gotcha — re-apply after any definition change.** If you edit `churn_features.py`, the task must re-run `feast apply` for the change to register. Forgetting this is the classic "my new feature isn't showing up." Because the task runs `feast apply` every time, you're covered — just remember it when testing definitions by hand.

> **Gotcha — paths are relative to the repo.** `FileSource(path="data/churn_validated.parquet")` is resolved relative to the feature repo directory, and the Feast CLI must run from that directory (`cwd=repo`). A path mismatch here surfaces as "file not found" during `apply`/`materialize`.

## ✅ Hour 3 Exercise

1. Create your feature repo (`feature_store.yaml` via `feast init`, then your entity/source/feature view) pointing at the Hour 2 parquet. Include at least three features.
2. Implement `materialize_features` and run the DAG through this task; confirm `feast apply` and `materialize-incremental` succeed in the task logs.
3. **Verify retrieval works** (preview of Hour 4): in a scratch script, build a small entity dataframe (`customer_id` + `event_timestamp`) and call `get_historical_features` to confirm features come back joined.
4. **Stretch:** add a second feature view (e.g., a different feature group) and a feature service bundling the features your model will use.

---

## 🎤 Hour 3 — Interview Questions & Answers

**Q1 (conceptual). Why insert a feature store between ingestion and training instead of training straight from the file?**
To get one consistent feature definition for both training and future serving (avoiding training–serving skew), a correct point-in-time join for building the training set, and a ready path to low-latency online serving. Training straight from a file gives none of these and pushes point-in-time correctness onto you.

**Q2 (practical). What does `materialize-incremental` do, and why is it in this task?**
It loads the latest feature values from the offline store into the online store, up to a given timestamp (incrementally since the last run). It's here so that after features are (re)defined, the online store is populated for any real-time serving and the feature view is fully refreshed as part of the pipeline run.

**Q3 (gotcha). After editing a feature view, the change doesn't take effect. Why?**
Definitions only update in the registry when you run `feast apply`. Editing the Python file alone changes nothing until apply runs. The task re-applies every run, but manual testing needs an explicit `feast apply`.

**Q4 (gotcha). Your training retrieval returns all nulls for the features. Name two likely causes given this pipeline.**
(1) TTL too short relative to the gap between the feature `event_timestamp` and the entity dataframe's `event_timestamp`, so the point-in-time join finds nothing in range. (2) A path/timestamp-field mismatch in the `FileSource` so Feast isn't reading the data you think it is. Both yield empty joins rather than errors.

**Q5 (conceptual). How does Feast relate to the Airflow task that calls it — does Feast schedule anything?**
No. Feast manages feature definitions, storage, and retrieval but doesn't orchestrate. Airflow schedules and runs the task that calls `feast apply`/`materialize`; Feast just executes those operations and serves features. The orchestrator drives; the feature store stores and serves.

---

# Hour 4 — Training task with MLflow

**Goal:** build the training set from Feast, train the model, and log it to MLflow **with a signature**, registering a new model version.

## 4.1 Build the training set from Feast (point-in-time)

Training reads features through Feast so the join is time-correct. You supply an **entity dataframe** (entity keys + an `event_timestamp` + the label) and Feast attaches the right feature values.

```python
import pandas as pd
from feast import FeatureStore

def build_training_df(repo: str) -> pd.DataFrame:
    store = FeatureStore(repo_path=repo)

    # entity_df: who, at what moment, and the label to predict
    raw = pd.read_parquet("include/data/churn_validated.parquet")
    entity_df = raw[["customer_id", "event_timestamp"]].copy()
    entity_df["churn"] = (raw["Churn"] == "Yes").astype(int)   # label as 0/1

    training_df = store.get_historical_features(
        entity_df=entity_df,
        features=[
            "customer_features:tenure",
            "customer_features:MonthlyCharges",
            "customer_features:Contract",
        ],
    ).to_df()
    return training_df
```

## 4.2 What a model signature is and why log one

A **model signature** is a recorded description of the model's expected inputs and outputs — the column names and types it takes, and what it returns. Think of it as a contract: it documents how to call the model and lets MLflow (and serving tools) **catch schema mismatches** — wrong columns, wrong types — before they cause silent, wrong predictions in production. You don't write it by hand; MLflow infers it from example data.

```python
from mlflow.models import infer_signature
signature = infer_signature(X_train, model.predict(X_train))
```

## 4.3 The training task

Train, log params/metrics, log the model *with the signature*, and register a new version so Hour 5 can evaluate and (maybe) promote it.

```python
MLFLOW_URI = "http://host.docker.internal:5000"   # localhost from outside Docker
MODEL_NAME = "churn-classifier"

@task
def train(repo: str) -> dict:
    import mlflow, pandas as pd
    from mlflow.models import infer_signature
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import roc_auc_score

    df = build_training_df(repo)
    y = df["churn"]
    # simple encoding for the demo; a real pipeline would use a proper preprocessor
    X = pd.get_dummies(df[["tenure", "MonthlyCharges", "Contract"]], columns=["Contract"])

    X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

    n_estimators = 200
    model = RandomForestClassifier(n_estimators=n_estimators, random_state=42)
    model.fit(X_tr, y_tr)

    # holdout metric (used by Hour 5's gate)
    auc = roc_auc_score(y_te, model.predict_proba(X_te)[:, 1])

    mlflow.set_tracking_uri(MLFLOW_URI)
    mlflow.set_experiment("churn")
    with mlflow.start_run() as run:
        mlflow.log_param("n_estimators", n_estimators)
        mlflow.log_param("model_type", "RandomForestClassifier")
        mlflow.log_metric("roc_auc", auc)

        signature = infer_signature(X_tr, model.predict(X_tr))
        try:
            info = mlflow.sklearn.log_model(
                model, name="model", signature=signature,
                input_example=X_tr.head(3), registered_model_name=MODEL_NAME,
            )
        except TypeError:   # older MLflow uses artifact_path=
            info = mlflow.sklearn.log_model(
                model, artifact_path="model", signature=signature,
                input_example=X_tr.head(3), registered_model_name=MODEL_NAME,
            )
        run_id = run.info.run_id

    # find the version this run just registered
    from mlflow import MlflowClient
    client = MlflowClient(tracking_uri=MLFLOW_URI)
    version = client.get_latest_versions(MODEL_NAME)[-1].version
    return {"run_id": run_id, "version": version, "roc_auc": float(auc)}
```

> **Gotcha — `registered_model_name` auto-registers a new version every run.** Each pipeline run creates a *new* version of `churn-classifier`. That's intended (so Hour 5 can compare and promote), but it means versions accumulate — don't be surprised by version numbers climbing. Promotion (next hour) is what marks which version is actually "the one."

> **Gotcha — `log_model` signature differs by MLflow version.** Recent MLflow uses `name=`; older uses `artifact_path=`. The `try/except` above handles both. `log_param`/`log_metric` are stable.

## ✅ Hour 4 Exercise

1. Implement `build_training_df` and `train` for your dataset and model. Run the DAG through `train` and confirm a run appears in MLflow with your params, the `roc_auc` metric, the model artifact, **and a signature** (check the model's UI page).
2. Confirm a registered model with an incrementing **version** appears under the MLflow Models tab.
3. **Stretch:** add a second logged metric (e.g., accuracy or F1) and an `input_example`, and verify both show up in the run.

---

## 🎤 Hour 4 — Interview Questions & Answers

**Q1 (conceptual). What is a model signature and what does logging one protect against?**
It's a recorded schema of the model's inputs and outputs (column names/types in, prediction type out). Logging it creates a contract that lets MLflow and serving tools validate calls and catch schema mismatches — wrong or missing columns, wrong types — before they cause silent, incorrect predictions in production.

**Q2 (practical). How do you build the training set from Feast, and what guarantees does that give?**
Provide an entity dataframe of entity keys, an `event_timestamp` per row, and the label, then call `get_historical_features(...)`. Feast performs a point-in-time join, attaching only feature values known at or before each row's timestamp. This guarantees no future data leaks into training and that features match how they'd be computed at serving time.

**Q3 (practical). What does `registered_model_name` do in `log_model`, and what's the side effect across runs?**
It registers the logged model in the Model Registry under that name, creating a new model *version* each time. The side effect is that every pipeline run adds a version, so version numbers grow; which version is "production" is determined separately by aliases/tags, not by being the newest.

**Q4 (gotcha). Your `log_model` call fails with an unexpected keyword argument. What's the likely cause and fix?**
The MLflow version's `log_model` signature changed — recent versions use `name=` while older ones use `artifact_path=`. The fix is to use the form matching your installed version (or wrap both in try/except). Params/metrics logging is unaffected.

**Q5 (conceptual). Why log the metric to MLflow in the training task instead of just returning it via XCom?**
Returning it via XCom passes it to the next task, but logging it to MLflow attaches it durably to the run and model version — enabling comparison across runs, lineage from model to the metric that produced it, and an auditable record for promotion decisions. The pipeline can do both: return for the immediate gate, log for the permanent record.

---

# Hour 5 — Evaluation & promotion

**Goal:** decide whether the new model is good enough, and if so, promote it — using MLflow **aliases and tags**.

## 5.1 Why promotion is a gate, not an automatic step

Registering a model version doesn't make it production-ready; it just records it. **Promotion** is the deliberate decision that *this* version is now the one to serve. Gating it on a metric threshold (and ideally on beating the current production model) prevents a worse model from replacing a better one sneakily. This is the branch you sketched in Hour 1.

## 5.2 Aliases and tags (MLflow's current promotion model)

MLflow used to use fixed **stages** (`Staging`, `Production`) — these are **deprecated**. The current mechanism is two things:

- **Alias** — a movable, named pointer to a specific model version, e.g., `champion`. Your serving code loads `models:/churn-classifier@champion` and always gets whatever version currently holds that alias. Promotion = pointing `champion` at the new version. Rollback = pointing it back.
- **Tag** — a key/value label on a version, e.g., `validation_status=passed` or `rejected`. Tags record *why* a version is where it is.

```python
from mlflow import MlflowClient
client = MlflowClient(tracking_uri=MLFLOW_URI)
client.set_registered_model_alias("churn-classifier", "champion", version=3)   # promote v3
client.set_model_version_tag("churn-classifier", "3", "validation_status", "passed")
# serving later: mlflow.sklearn.load_model("models:/churn-classifier@champion")
```

## 5.3 The evaluate task (with a branch)

Evaluate compares the new version's metric to the threshold *and* to the current champion (challenger-beats-champion), then the branch routes to promote or reject.

```python
THRESHOLD = 0.80   # from Hour 1; or read from an Airflow Variable

@task
def evaluate(train_info: dict) -> dict:
    from mlflow import MlflowClient
    client = MlflowClient(tracking_uri=MLFLOW_URI)

    new_auc = train_info["roc_auc"]
    passes_bar = new_auc >= THRESHOLD

    # compare to current champion, if one exists
    beats_champion = True
    try:
        champ = client.get_model_version_by_alias(MODEL_NAME, "champion")
        champ_run = client.get_run(champ.run_id)
        champ_auc = champ_run.data.metrics.get("roc_auc", 0.0)
        beats_champion = new_auc > champ_auc
    except Exception:
        beats_champion = True   # no champion yet → first passing model wins

    passed = bool(passes_bar and beats_champion)
    return {**train_info, "passed": passed, "new_auc": new_auc}

@task.branch
def gate(eval_info: dict) -> str:
    return "promote" if eval_info["passed"] else "reject"

@task
def promote(eval_info: dict):
    from mlflow import MlflowClient
    client = MlflowClient(tracking_uri=MLFLOW_URI)
    v = str(eval_info["version"])
    client.set_registered_model_alias(MODEL_NAME, "champion", eval_info["version"])
    client.set_model_version_tag(MODEL_NAME, v, "validation_status", "passed")
    print(f"Promoted version {v} to @champion (roc_auc={eval_info['new_auc']:.3f}).")

@task
def reject(eval_info: dict):
    from mlflow import MlflowClient
    client = MlflowClient(tracking_uri=MLFLOW_URI)
    v = str(eval_info["version"])
    client.set_model_version_tag(MODEL_NAME, v, "validation_status", "rejected")
    print(f"Rejected version {v} (roc_auc={eval_info['new_auc']:.3f} below bar or champion).")
```

> **Gotcha — `@champion` is mutable; deploy by alias, not version number.** The point of an alias is that serving code references `models:/churn-classifier@champion` and never hard-codes a version. If you bake a version number into serving, promotion does nothing for production traffic. Always load by alias.

> **Gotcha — a rejected model is still registered.** Rejecting tags the version `rejected` but the version still exists in the registry. That's fine and intentional (you keep the history), but don't assume "rejected" means "deleted" — it just doesn't hold the `champion` alias.

> **Gotcha — branch joins need the right trigger rule.** `promote` and `reject` are branches; whichever doesn't run is *skipped*. The downstream `finalize` must use `trigger_rule="none_failed_min_one_success"` or it'll skip too. (You set this in the Hour 1 skeleton — this is where it matters.)

## ✅ Hour 5 Exercise

1. Implement `evaluate`, `gate`, `promote`, `reject` with your threshold. Run the DAG end-to-end; confirm the model is promoted (check the `champion` alias and `validation_status` tag in the MLflow UI).
2. **Force a rejection:** raise the threshold above your model's score, re-run, and confirm the new version is tagged `rejected`, the `champion` alias is unchanged, and `finalize` still runs.
3. **Challenger beats champion:** lower the threshold so models pass, run twice with different `n_estimators`, and confirm `champion` moves only when the new run's metric beats the current champion's.
4. **Stretch:** read `THRESHOLD` from an Airflow Variable so you can change the bar without editing code.

---

## 🎤 Hour 5 — Interview Questions & Answers

**Q1 (conceptual). What replaced MLflow model stages, and why?**
Aliases and tags. Stages (`Staging`/`Production`) were fixed and inflexible. Aliases are mutable named pointers (e.g., `champion`) to a specific version, decoupling deployment from version numbers; tags are key/value labels recording status (e.g., `validation_status=passed`). Together they're more flexible and auditable than the old four fixed stages.

**Q2 (practical). How does promotion via alias enable zero-touch rollback?**
Serving loads `models:/<name>@champion`, so it always uses whatever version the alias points to. Promotion reassigns the alias to the new version; rollback reassigns it back to a previous version. Neither requires changing or redeploying the serving code — the alias indirection decouples deployment from the application.

**Q3 (conceptual). Why gate promotion on beating the current champion, not just clearing a fixed threshold?**
A fixed threshold prevents obviously-bad models, but a new version can clear the bar while still being *worse* than what's in production. Comparing to the champion's metric ensures you only replace production when the challenger is actually an improvement, avoiding silent regressions.

**Q4 (gotcha). After promotion, production predictions don't change. What's a likely cause?**
The serving code references a hard-coded version (e.g., `models:/<name>/3`) instead of the alias (`models:/<name>@champion`). Promotion moves the alias, not the version number, so version-pinned serving ignores it. Fix by loading via the alias.

**Q5 (gotcha). In the DAG, when the model is rejected, the downstream `finalize` task gets skipped. Why, and how do you fix it?**
`promote` and `reject` are branch targets; the unchosen one is skipped, and under the default `all_success` rule that skip cascades to `finalize`. Set `finalize`'s `trigger_rule="none_failed_min_one_success"` so it runs as long as nothing failed and at least one upstream succeeded.

---

# Hour 6 — Iterate & debug

**Goal:** get the whole pipeline running clean end-to-end, using the Airflow UI to find and fix what breaks.

## 6.1 The debugging mindset

A multi-task pipeline rarely runs perfectly the first time, and that's expected — the skill is *locating* the failure fast. Airflow's UI is built for exactly this: it shows you which task failed, why, and lets you re-run just that task after a fix. You already know the views; here's how to use them as a debugging loop.

## 6.2 The debug loop in the Airflow UI

1. **Grid view** — find the red (failed) task instance. The color tells you *where* the pipeline broke without reading any logs.
2. **Logs** — click the failed task and read its log bottom-up; the actual exception and traceback are near the end. This tells you *why*.
3. **Fix** the task's code (or data, or a Connection/Variable).
4. **Clear and re-run just that task** — use the "Clear task" action so Airflow re-runs the failed task (and its downstream) without redoing the expensive upstream tasks that already succeeded. This is the core of fast iteration.
5. Repeat until the whole row is green.

> **Gotcha — fix-and-retrigger reruns everything; clear-task doesn't.** Re-triggering the whole DAG re-ingests, re-materializes, and re-trains from scratch — slow. Clearing only the failed task (and downstream) reuses the successful upstream work. For a pipeline with a multi-minute training step, this is the difference between a 20-second and a 4-minute debug cycle.

## 6.3 The failures you'll most likely hit (and where they show up)

Grounded in the exact pipeline you built:

- **`ingest_and_validate` fails** — usually a real validation failure (good!) or a wrong file path / missing CSV. The log shows the GX failure detail or a `FileNotFoundError`. Fix the data or the path.
- **`materialize_features` fails** — almost always a Feast path/timestamp mismatch (`FileSource` path relative to the repo, `cwd` wrong) or forgetting that `feast apply` must precede `materialize`. The subprocess error is in the log.
- **`train` fails to reach MLflow** — the `localhost` vs `host.docker.internal` gotcha, or the MLflow server isn't running, or it lacks a `--backend-store-uri` so registration fails. Symptom: connection refused, or a registry error on `registered_model_name`.
- **`train` returns nulls / poor metric** — Feast returned null features (TTL too short, or timestamp mismatch from Hour 3). Check the training dataframe in the log.
- **`finalize` unexpectedly skipped** — the branch trigger-rule gotcha from Hour 5. Set `none_failed_min_one_success`.
- **`promote` does nothing visible** — you're looking at version numbers, but promotion sets an alias/tag; check the model version's alias and tags in the UI.

## 6.4 The complete pipeline

With every task implemented, your DAG reads end-to-end like this (bodies abbreviated to the ones built in Hours 2–5):

```python
# dags/churn_pipeline.py  (final shape)
from datetime import datetime, timedelta
from airflow.sdk import dag, task

MLFLOW_URI = "http://host.docker.internal:5000"
MODEL_NAME = "churn-classifier"

def alert(context):
    try:
        print(f"ALERT: {context['task_instance'].task_id} failed: {context.get('exception')}")
    except Exception as e:
        print(f"alert callback errored: {e}")

@dag(
    start_date=datetime(2024, 1, 1), schedule=None, catchup=False,
    default_args={"retries": 1, "retry_delay": timedelta(seconds=30),
                  "on_failure_callback": alert},
    tags=["capstone", "ml"],
)
def churn_pipeline():

    @task(retries=2)
    def ingest_and_validate() -> str:
        ...      # Hour 2: pull → validate (GX) → add event_timestamp → save → return path

    @task
    def materialize_features(data_path: str) -> str:
        ...      # Hour 3: feast apply + materialize-incremental → return repo

    @task
    def train(repo: str) -> dict:
        ...      # Hour 4: Feast point-in-time → fit → log (signature) → register → return info

    @task
    def evaluate(train_info: dict) -> dict:
        ...      # Hour 5: threshold + champion comparison → return {..., passed}

    @task.branch
    def gate(eval_info: dict) -> str:
        return "promote" if eval_info["passed"] else "reject"

    @task
    def promote(eval_info: dict):
        ...      # Hour 5: set @champion alias + passed tag

    @task
    def reject(eval_info: dict):
        ...      # Hour 5: tag rejected

    @task(trigger_rule="none_failed_min_one_success")
    def finalize():
        print(f"Pipeline complete. Review at {MLFLOW_URI}")

    @task(trigger_rule="all_done")
    def cleanup():
        import glob, os
        for f in glob.glob("include/data/*_tmp*.parquet"):
            os.remove(f)

    data = ingest_and_validate()
    feats = materialize_features(data)
    info = train(feats)
    ev = evaluate(info)
    decision = gate(ev)
    p, r = promote(ev), reject(ev)
    decision >> [p, r]
    [p, r] >> finalize() >> cleanup()

churn_pipeline()
```

## ✅ Hour 6 Exercise

1. Run the full DAG. When something breaks (it will), use the Grid → Logs → fix → **clear-task** loop to fix one failure at a time without re-running the whole pipeline.
2. **Practice clear-task:** after a green run, deliberately break `evaluate` (e.g., a typo), re-run, then fix it and *clear only* `evaluate` — confirm `ingest`/`materialize`/`train` are **not** re-run.
3. Achieve one fully green end-to-end run that ends with a promoted (or deliberately rejected) model, and `finalize` + `cleanup` both run.
4. **Reflect:** write down the single hardest bug you hit and which UI view revealed it — that's your debugging playbook for next time.

---

## 🎤 Hour 6 — Interview Questions & Answers

**Q1 (practical). Walk through your loop for debugging a failed Airflow pipeline.**
Open the Grid view to find the red task (where it broke), read that task's log bottom-up for the exception (why), fix the code/data/config, then clear just the failed task so Airflow re-runs it and its downstream without redoing successful upstream work. Repeat until the run is green.

**Q2 (gotcha). Why prefer clearing a task over re-triggering the whole DAG when debugging?**
Re-triggering reruns every task from scratch — re-ingesting, re-materializing, re-training — which is slow and wasteful. Clearing the failed task reuses the already-successful upstream outputs and only re-runs from the fix point, dramatically shortening the iteration cycle for pipelines with expensive steps.

**Q3 (practical). `train` fails with a connection error to MLflow inside your Astro environment. What do you check first?**
That the task is using `http://host.docker.internal:5000` (not `localhost`, which points inside the container), that the MLflow server is actually running, and that it was started with a `--backend-store-uri` so the registry works. Connection-refused points to URL/server; a registry error on `registered_model_name` points to a missing backend store.

**Q4 (conceptual). The whole DAG shows green but the model wasn't promoted. Is the pipeline broken?**
Not necessarily — green means tasks succeeded, and a deliberate rejection is a *successful* outcome of the gate (the model didn't clear the bar, so `reject` ran and tagged it). Confirm by checking the gate's decision and the version's tags/alias. Green tasks describe execution success, not the business decision.

**Q5 (gotcha). After fixing a feature definition, you clear and re-run `train` but still get null features. Why might clearing `train` alone be insufficient?**
Because the fix lives upstream: feature definitions are applied in `materialize_features`, not `train`. Clearing only `train` re-reads features Feast already has; you must clear from `materialize_features` so `feast apply`/`materialize` re-run with the corrected definition. Clear from the task whose output actually changed.

---

# 📋 Gotchas summary table

| # | Stage | Gotcha | Fix |
|---|-------|--------|-----|
| 1 | Setup | MLflow registry/aliases fail | Start `mlflow server` with `--backend-store-uri` (e.g., sqlite) |
| 2 | Setup | Task can't reach MLflow on `localhost` | From Astro/Docker use `http://host.docker.internal:5000` |
| 3 | Planning | Vague success metric | Commit a concrete threshold (e.g., ROC-AUC ≥ 0.80) up front |
| 4 | Planning | Static dataset has no timestamps | Synthesize `event_timestamp` at ingestion for Feast |
| 5 | Validation | Copying GX 0.x tutorials | Use GX 1.x Fluent API; pin `great-expectations>=1.7` |
| 6 | Validation | Validation passes on bad data without alerts | Add explicit expectations; `raise` on failure to stop the DAG |
| 7 | Storage | Pushing the dataset through XCom | Save to S3/file; pass only the path |
| 8 | Storage | AWS creds in the DAG | Use an `aws_default` Connection + `S3Hook` |
| 9 | Feast | Edited feature view not taking effect | Re-run `feast apply` |
| 10 | Feast | Null features after retrieval | Widen TTL; check `FileSource` path/timestamp field; run from repo dir |
| 11 | Train | `log_model` keyword error | `name=` (newer) vs `artifact_path=` (older) — try both |
| 12 | Train | Version numbers keep climbing | Expected: `registered_model_name` adds a version each run; alias marks the real one |
| 13 | Train | Missing model signature | `infer_signature(X, model.predict(X))` + pass `signature=`/`input_example=` |
| 14 | Promote | Production unchanged after promotion | Serve by alias `models:/<name>@champion`, never a hard-coded version |
| 15 | Promote | Stages don't work | Stages are deprecated; use aliases + tags |
| 16 | Branch | `finalize` skipped after promote/reject branch | `trigger_rule="none_failed_min_one_success"` on the join |
| 17 | Debug | Re-triggering whole DAG to test a fix | Clear only the failed task (and downstream) |
| 18 | Debug | Clearing `train` doesn't fix null features | Clear from `materialize_features` — the fix is upstream |

---

# 🗂️ Quick reference card

### The pipeline shape
```
ingest_and_validate → materialize_features → train → evaluate ─▶ gate ─┬─ promote ─┐
                                                                       └─ reject  ─┴─▶ finalize → cleanup
```

### Great Expectations (1.x) — validate a DataFrame
```python
import great_expectations as gx
ctx = gx.get_context()
asset = ctx.data_sources.add_pandas("s").add_dataframe_asset("a")
bdef = asset.add_batch_definition_whole_dataframe("b")
suite = ctx.suites.add(gx.ExpectationSuite(name="suite"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(column="customer_id"))
ok = bdef.get_batch(batch_parameters={"dataframe": df}).validate(suite).success
```

### Save to S3 (via Airflow Connection)
```python
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
S3Hook(aws_conn_id="aws_default").load_file(
    filename=local_path, bucket_name="my-bucket", key="churn/validated.parquet", replace=True)
```

### Training set from Feast (point-in-time)
```python
from feast import FeatureStore
store = FeatureStore(repo_path="include/feature_repo")
train_df = store.get_historical_features(
    entity_df=entity_df,           # customer_id + event_timestamp + label
    features=["customer_features:tenure", "customer_features:MonthlyCharges"],
).to_df()
```

### MLflow — log with signature, register, promote
```python
import mlflow
from mlflow.models import infer_signature
from mlflow import MlflowClient

mlflow.set_tracking_uri("http://host.docker.internal:5000")
sig = infer_signature(X_tr, model.predict(X_tr))
with mlflow.start_run() as run:
    mlflow.log_metric("roc_auc", auc)
    mlflow.sklearn.log_model(model, name="model", signature=sig,
                             input_example=X_tr.head(3),
                             registered_model_name="churn-classifier")

client = MlflowClient(tracking_uri="http://host.docker.internal:5000")
client.set_registered_model_alias("churn-classifier", "champion", version=3)   # promote
client.set_model_version_tag("churn-classifier", "3", "validation_status", "passed")
# serve later: mlflow.sklearn.load_model("models:/churn-classifier@champion")
```

### Promotion model (replaces stages)
| Concept | What it is | Example |
|---------|-----------|---------|
| Alias | movable pointer to a version | `champion` → `models:/m@champion` |
| Tag | key/value label on a version | `validation_status=passed` |
| (Stages) | **deprecated** — don't use | ~~Staging/Production~~ |

### Airflow UI debug loop
Grid (find red) → Logs (read bottom-up for the exception) → fix → **Clear task** (re-run just that task + downstream) → repeat until green.

---

*You've built the whole arc: scope → validate → feature store → train+log → gated promotion → debug to green. Re-run the exercises with a different dataset to prove the pattern is yours, not just this example's.*
