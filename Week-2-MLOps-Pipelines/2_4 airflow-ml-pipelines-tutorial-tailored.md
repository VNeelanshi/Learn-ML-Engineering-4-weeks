# ML Pipelines with Airflow + MLflow

*A 4-hour, hands-on continuation. This assumes you've completed a general Airflow course and skips what you already know.*

## What this tutorial assumes you already know

You've got the fundamentals, so this guide does **not** re-explain any of the following — if any feel shaky, go back to your base tutorial:

- DAGs, tasks, operators, task instances, DAG runs, the logical date
- The scheduler, the executors (Local / Celery / Kubernetes), the API server, the DAG processor, the metadata database
- XCom — push/pull, the "small values only" rule, and JSON-serialization limits
- The TaskFlow API (`@dag` / `@task`), dependency operators (`>>`, fan-out `>> [a, b]`, fan-in `[a, b] >>`)
- `schedule`, `default_args`, `catchup=False`, the UTC default, the top-level-code gotcha, calling the `@dag` function
- Connections (and never hard-coding credentials), sensors (FileSensor/S3KeySensor, poke vs reschedule), hooks, Jinja templates
- Running Airflow locally with the Astro CLI (`astro dev start`, `astro dev restart`, requirements.txt rebuilds)

## What's new here

1. **The ML pipeline pattern** — why ML workflows have a specific shape and what each stage is for.
2. **Branching and trigger rules** — making a DAG choose a path, and the skip-cascade trap.
3. **Robust error handling** — retry strategy, idempotency, and failure callbacks (deeper than basic `retries`).
4. **Production deployment** — beyond local: self-hosted, Amazon MWAA, and Astronomer's managed cloud.
5. **MLflow** — tracking experiments, params, metrics, models, and the registry.

---

## First: imports and MLflow setup

### Add MLflow to your environment

Since you're on the Astro CLI, add these to `requirements.txt` and rebuild:

```text
mlflow
scikit-learn
pandas
```
```bash
astro dev restart      # rebuilds the image so the new packages are installed
```

You'll also run an MLflow **tracking server** (the service that stores and displays your ML run records). The simplest path for the course is to run it on your host machine in a separate terminal:

```bash
pip install mlflow
mlflow server --host 0.0.0.0 --port 5000      # MLflow UI at http://localhost:5000
```

> **Gotcha — `localhost` from inside a container isn't your host.** Because Astro runs Airflow inside Docker, a task that calls `mlflow.set_tracking_uri("http://localhost:5000")` will look for MLflow *inside the container* and fail to reach the server you started on your host. On Docker Desktop (Mac/Windows), use `http://host.docker.internal:5000` from inside the container instead. (On Linux, either run MLflow as another service in the same Docker network or pass the host gateway.) Keep this in mind for all the MLflow code below — swap the URL accordingly for your setup.

---

# Hour 1 — ML DAG patterns

**Goal:** understand the *shape* of an ML pipeline and why it's built that way. (You already know how to write a DAG — this hour is about what an ML DAG should contain.)

## 1.1 Why ML pipelines have a fixed shape

A one-off model on your laptop is a script. A *production* model needs to be retrained as new data arrives, scored fairly, version-tracked, and deployable — repeatedly and reliably. That's exactly the orchestration problem you already understand, applied to machine learning. Nearly every ML pipeline therefore follows the same backbone:

```
ingest → preprocess → train → evaluate → register
```

Each stage exists to solve a specific problem:

- **Ingest** — *fetch the raw data* from wherever it lives (an API, a database, a file in cloud storage). Separated out so a flaky data source can be retried on its own without redoing everything downstream.
- **Preprocess** — *clean and reshape the data* (handle missing values, fix types, scale features). Models learn poorly from messy data, so this always comes before training.
- **Train** — *fit the model* on the prepared data.
- **Evaluate** — *score the model on data it never saw during training.* This is the stage beginners skip and regret. A model can memorize its training data and look perfect while being useless on new data (this is called **overfitting**). Scoring on a held-out slice of data is the only honest measure of quality.
- **Register** — *save and catalog the approved model*, tagged with its version and its scores, so it can be deployed and audited. In this course, MLflow's **model registry** is where this happens.

Drawn as a graph it's usually a straight line, occasionally with a branch (Hour 2) to decide whether a freshly trained model is good enough to register.

> **Why split into stages instead of one big `@task`?** The same three reasons you already know from general Airflow — granular retries, per-stage visibility, and reuse/parallelism — but they matter *more* in ML because stages are slow and expensive. Re-running a 40-minute training step just because the final "register" call had a typo is painful; separate tasks let you retry only what failed.

## 1.2 The two ML-specific habits that aren't in a general Airflow course

You know XCom is for small values only. In ML this rule is non-negotiable and has a standard pattern:

**1. Pass artifacts by reference, never by value.** Datasets and trained models are large. Write them to shared storage (local disk for the course, cloud storage like S3 in production) and pass only the *path* through XCom — exactly the technique you learned, but now it's the backbone of every ML DAG.

```python
@task
def preprocess(raw_path: str) -> str:
    df = pd.read_parquet(raw_path)
    # ...clean...
    out = "/tmp/clean.parquet"
    df.to_parquet(out)
    return out        # pass the path, not the DataFrame
```

**2. Make every task idempotent and reproducible.** **Idempotent** means running a task twice produces the same result as running it once (critical because retries re-run tasks — covered in Hour 2). **Reproducible** means the same inputs always give the same model — which in practice means *setting a random seed*. Most ML libraries make random choices (how data is split, how a model initializes); without a fixed seed you get a slightly different model every run and can't compare experiments fairly.

```python
train_test_split(X, y, test_size=0.2, random_state=42)   # random_state = the seed
RandomForestClassifier(n_estimators=100, random_state=42)
```

## ✅ Hour 1 Exercise

Write `dags/iris_skeleton.py` using the TaskFlow style you already know, modeling the full backbone with five `@task` functions: `ingest`, `preprocess`, `train`, `evaluate`, `register`. Requirements:

1. Use the **Airflow 3 imports** (`from airflow.sdk import dag, task`).
2. Each stage passes a *path or small value* to the next (practice the by-reference habit — even if it's a fake path string for now).
3. `train` and any data split use a fixed `random_state`.
4. `evaluate` returns an accuracy number; `register` prints it.
5. Trigger it and confirm the five tasks run in order.

**Stretch:** add a `start_date`, `schedule=None`, `catchup=False`, and a tag — and confirm you remembered to call `iris_skeleton()` at the bottom.

---

## 🎤 Hour 1 — Interview Questions & Answers

**Q1 (conceptual). Why is the evaluate stage kept separate from train, and what failure does it guard against?**
Evaluate scores the model on held-out data the model never saw during training. It guards against *overfitting* — a model that memorizes its training data scores perfectly on it but fails on new data. Only by measuring on unseen data do you get an honest quality signal, which is also what a downstream branch uses to decide whether to register the model.

**Q2 (practical). In an ML DAG, how do you move a trained model or a large dataset from one task to the next, and why not via XCom directly?**
Write the artifact to shared storage (disk or cloud like S3) and pass only the path/URI through XCom. XCom stores values in the metadata database and is meant for small values; pushing a large dataset or pickled model through it bloats the database and degrades scheduler performance.

**Q3 (conceptual). What does it mean for a training task to be reproducible, and how do you achieve it?**
Reproducible means the same inputs yield the same model every run. You achieve it by fixing the random seed (`random_state`/`seed`) everywhere randomness enters — data splitting, model initialization, shuffling — so runs are comparable and results can be reproduced or debugged.

**Q4 (gotcha). You combined ingest, preprocess, and train into one big task to "keep it simple." Training fails on the last line. What did you lose?**
Granular retries and visibility. A retry now re-runs ingest and preprocess too — wasting time and possibly re-hitting rate-limited sources — and the UI only shows "the big task failed," not *which* stage. Splitting stages lets you retry just `train` and see exactly where the pipeline broke.

**Q5 (practical). When would you add a branch to the standard backbone, and between which stages?**
Most commonly between evaluate and register: branch on the evaluation metric so you only register the model if it clears a quality threshold, otherwise route to an alert/skip path. (Branching mechanics are Hour 2.) A second common branch is after preprocess, gating on a data-quality check before spending compute on training.

---

# Hour 2 — Branching & error handling

**Goal:** make pipelines that choose paths and survive failures. You know basic `retries`/`default_args`; this hour goes deeper and adds the parts your base course didn't cover — branching, trigger rules, retry *strategy*, and callbacks.

## 2.1 Branching: choosing which tasks run

**Branching** lets a task inspect a condition and decide which downstream task(s) run; the unchosen ones are marked **skipped**. The modern tool is the `@task.branch` decorator — it's a TaskFlow task whose return value is the **task id** (the name) of whichever task should run next.

```python
from datetime import datetime
from airflow.sdk import dag, task

@dag(start_date=datetime(2024, 1, 1), schedule=None, catchup=False, tags=["hour2"])
def branching_demo():

    @task
    def evaluate() -> float:
        return 0.72                       # pretend this is the model's accuracy

    @task.branch
    def gate(accuracy: float) -> str:
        # return the ID (name) of the task to run next
        return "register" if accuracy >= 0.85 else "alert_low_quality"

    @task
    def register():
        print("Good model — registering.")

    @task
    def alert_low_quality():
        print("Accuracy below bar — not registering; alerting the team.")

    acc = evaluate()
    decision = gate(acc)
    decision >> [register(), alert_low_quality()]   # branch must point at BOTH options

branching_demo()
```

Run it: `gate` returns `"alert_low_quality"` (0.72 < 0.85), so `register` shows as **skipped**.

> **Gotcha — branch targets must be *direct* downstream tasks.** The id you return has to be a task immediately downstream of the branch. Returning a task id from further down the graph won't work.

> **Gotcha — returning `None` skips *everything* downstream.** If you want an explicit "do nothing" path, create a real placeholder task (e.g., an `EmptyOperator`) and return its id, rather than relying on `None`.

## 2.2 Trigger rules: the part that catches everyone

By default a task runs only when **all** its upstream tasks **succeeded** — that rule is named **`all_success`**. A **trigger rule** is simply the condition under which a task is allowed to start, based on its upstreams' states. This default collides with branching, because **branching produces skipped tasks**.

Picture a join task after a branch:

```
          ┌─> register ───┐
gate ─>───┤               ├─> finalize
          └─> alert ──────┘
```

If `gate` picks `register`, then `alert` is **skipped**. Now `finalize` has one succeeded parent and one skipped parent. Under the default `all_success`, a skipped parent means "not all succeeded," so **`finalize` skips too** — even though the meaningful work succeeded. Worse, skips *cascade*: that skip propagates to anything downstream of `finalize` as well.

The fix is to give the join a forgiving trigger rule. The right one here is **`none_failed_min_one_success`** — run as long as nothing failed and at least one parent succeeded; skips don't count as failures.

```python
@task(trigger_rule="none_failed_min_one_success")
def finalize():
    print("Runs regardless of which branch was taken.")
```

Trigger rules you'll actually reach for:

| Rule | Runs when… | Typical use |
|---|---|---|
| `all_success` *(default)* | every upstream succeeded | normal linear flow |
| `none_failed_min_one_success` | nothing failed + ≥1 succeeded | **join after a branch** |
| `all_done` | every upstream finished, any result | cleanup that must always run |
| `one_success` | any upstream succeeded | first-to-finish wins |
| `one_failed` | any upstream failed | "alert if anything broke" task |

> **Gotcha — any time you branch, decide the join's trigger rule on purpose.** This is the single most common branching bug. The moment you add a branch, ask "what runs after the branches rejoin?" and set that task's `trigger_rule` (almost always `none_failed_min_one_success`).

## 2.3 Retry *strategy* (beyond `retries=2`)

You know `retries` and `retry_delay`. Two additions matter for real pipelines:

- **`retry_exponential_backoff=True`** — grows the wait after each failure (1, 2, 4, 8 minutes…). Use this when failures come from an external system that might be overloaded or rate-limiting you; backing off gives it room to recover instead of hammering it.
- **Idempotency is a *precondition* for retries.** A retry re-runs the whole task. If the task *appends* a row to a table, a retry duplicates data. Before adding retries, make the task safe to repeat — *overwrite* outputs rather than *append*, use deterministic file paths/keys.

```python
from datetime import timedelta
from airflow.sdk import task

@task(retries=3, retry_delay=timedelta(minutes=2), retry_exponential_backoff=True)
def call_flaky_source():
    ...
```

> **Gotcha — retries are for *transient* failures only.** A network blip deserves a retry; a genuine bug does not. Retrying broken code just delays the error and floods your logs with identical failures.

## 2.4 Callbacks: getting told when things break

A **callback** is a function Airflow runs automatically on an event. The key one is **`on_failure_callback`**, which fires when a task finally fails (after retries are exhausted). It's how teams send alerts. Airflow hands your function a `context` dictionary describing what failed.

```python
def alert_on_failure(context):
    try:
        ti = context["task_instance"]
        print(f"ALERT: '{ti.task_id}' failed: {context.get('exception')}")
        # real life: send to Slack / email / PagerDuty here
    except Exception as e:
        print(f"alert callback itself errored: {e}")

@task(on_failure_callback=alert_on_failure)
def risky():
    raise ValueError("boom")
```

Attach it per task (precise) or to the whole DAG via `@dag(..., on_failure_callback=...)` (one catch-all on DAG-run failure). Related callbacks: `on_retry_callback` (fires on each retry) and `on_success_callback`.

> **Gotcha — `on_failure_callback` does NOT fire on a retry that later succeeds.** It fires only after the *final* attempt fails. If you want to know about every failed attempt, that's `on_retry_callback`.

> **Gotcha — keep callbacks tiny and wrapped in try/except.** A callback runs in a constrained context. If your alerting code itself throws, you may get *no* notification — the opposite of what you wanted. Do one small thing, fast, defensively.

## ✅ Hour 2 Exercise

Extend your Hour 1 skeleton into `dags/iris_branching.py`:

1. After `evaluate`, add a `@task.branch` called `quality_gate` returning `"register"` if accuracy ≥ 0.85, else `"alert_low_quality"`.
2. Add the `alert_low_quality` task.
3. Add a `finalize` task after both branches with `trigger_rule="none_failed_min_one_success"`; confirm it runs even when `register` is skipped.
4. Give `train` `retries=2`, a 30-second `retry_delay`, and `retry_exponential_backoff=True`.
5. Attach `on_failure_callback` to `train`, temporarily `raise ValueError("test")` inside it, and confirm: it retries twice (watch the growing delay), then the callback fires. Remove the `raise`.

---

## 🎤 Hour 2 — Interview Questions & Answers

**Q1 (gotcha). After a branch, your join task keeps getting skipped even though one branch succeeded. Why, and what's the fix?**
The unchosen branch is *skipped*, and under the default `all_success` rule a skipped upstream means "not all succeeded," so the join skips too — and skips cascade further downstream. Fix it by setting the join's `trigger_rule="none_failed_min_one_success"`, which ignores skips and runs as long as nothing failed and at least one upstream succeeded.

**Q2 (conceptual). Why must a task be idempotent before you enable retries?**
A retry re-executes the task from the start. If it isn't idempotent — e.g., it appends rather than overwrites — the retry corrupts results (duplicate rows, double side-effects). Making tasks idempotent (overwrite outputs, deterministic keys) ensures retries are safe.

**Q3 (practical). When would you turn on exponential backoff?**
When failures stem from an external system that may be overloaded or rate-limiting. Exponential backoff increases the gap between attempts, giving the system time to recover, instead of a fixed-interval hammering that can worsen the outage or get you throttled.

**Q4 (gotcha). A task has `retries=3` and an `on_failure_callback`. Attempts 1 and 2 fail, attempt 3 succeeds. Does the failure callback fire?**
No. `on_failure_callback` fires only after the *final* attempt fails. Intermediate failed attempts that are later rescued by a successful retry trigger `on_retry_callback` (if set), not the failure callback.

**Q5 (conceptual). What does `@task.branch` return, and what happens to the branches it doesn't return?**
It returns the task id (or list of ids) of the direct-downstream task(s) that should run. Every other direct-downstream task is marked *skipped*. Because skips interact with trigger rules, any join that must run regardless needs a forgiving rule like `none_failed_min_one_success`.

---

# Hour 3 — Production Airflow

**Goal:** run Airflow for real. You already know the executors, the components, and the Astro CLI for *local* dev — so this hour focuses on the deployment options and configuration that take you from your laptop to production.

## 3.1 The deployment spectrum

Local `astro dev start` is for development. Production means: a real metadata database (PostgreSQL/MySQL, not SQLite), an executor that parallelizes (Local/Celery/Kubernetes — which you already know), and something that keeps it all running and recoverable. There are three broad ways to get there, trading control for convenience.

### Option A — Self-hosted (you run everything)

You operate every component yourself, typically in **containers** (packaged, isolated app copies). Two common shapes:

- **Docker Compose** on a single machine — describe all components in one file and run them together. Fine for small teams.
- **Kubernetes** via the official **Helm chart** (a packaged, configurable installer for Kubernetes) — for large, resilient, auto-scaling deployments, usually paired with the CeleryExecutor or KubernetesExecutor you already know.

*Trade-off:* maximum control and lowest licensing cost, but you own upgrades, patching, backups, scaling, and outages.

### Option B — Amazon MWAA (AWS manages it)

**MWAA** = **Amazon Managed Workflows for Apache Airflow**, an AWS service that runs Airflow's components for you. As of mid-2026 it supports Airflow 3.2 (and 3.0). How it works in practice:

- You drop your DAG files, `requirements.txt`, and plugins into an **Amazon S3 bucket** (cloud file storage); MWAA picks them up.
- AWS runs the scheduler, workers, database, and web server — you don't manage servers.
- It comes in more than one shape:
  - **Provisioned** — you pick an environment size (from a tiny "micro" up to large) and get fixed, tunable, environment-level resources.
  - **Serverless** — a newer option that runs workflows on demand with automatic scaling and no Airflow configuration to manage.

> **Gotcha — MWAA `requirements.txt` must be constrained.** For recent Airflow versions, MWAA requires your requirements to pin compatible versions via a `--constraint` line. Unconstrained or mismatched requirements are the most common cause of a failed environment update — and a failed update triggers a slow rollback. Test requirements before deploying.

> **Gotcha — major version upgrades aren't in-place on MWAA.** You can move between *minor* versions in place, but a *major* jump (e.g., the 2.x line to the 3.x line) requires creating a *new* environment and migrating to it. Plan that as a project, not a toggle.

### Option C — Astronomer's managed cloud (Astro)

You already use the **Astro CLI** locally. The same company offers **Astro**, a managed Airflow platform, and the CLI doubles as your deploy tool:

```bash
astro dev start     # local (you know this)
astro deploy        # push the same project to Astronomer's managed cloud
```

*Trade-off:* the smoothest path from local to production and managed operations, in exchange for vendor lock-in and cost.

**Choosing, briefly:** self-host for maximum control / lowest licensing cost if you have platform engineers; MWAA if you're on AWS and want managed infra; Astronomer if you want the smoothest developer-to-production experience.

## 3.2 Configuration in production

Three configuration mechanisms matter; one is new relative to your base course.

1. **Settings via environment variables.** Airflow's config lives in `airflow.cfg`, but production almost always overrides it with environment variables named `AIRFLOW__<SECTION>__<SETTING>` (double underscores). Example — choose the executor:
   ```bash
   export AIRFLOW__CORE__EXECUTOR=LocalExecutor
   ```

2. **Connections** *(you know these)* — named, stored credentials for external systems, referenced by id from DAGs/hooks. Still the right home for database/API/cloud credentials.

3. **Variables** *(likely new)* — a **Variable** is a named value you store in Airflow and read at runtime, ideal for things that change between environments (a threshold, a folder path, a feature flag). Set them in *Admin → Variables* or via CLI, read them in code:
   ```python
   from airflow.sdk import Variable
   threshold = float(Variable.get("quality_threshold"))
   ```
   The difference from a Connection: Connections hold *credentials*; Variables hold *configuration values*. Both keep changeable details out of your code.

> **Gotcha — `Variable.get()` at the top level hurts the scheduler.** You already know top-level code runs on every parse. `Variable.get()` hits the metadata database, so calling it at module level means a DB query on every parse of every DAG. Read Variables *inside* tasks (or use Jinja templating, which defers the lookup to run time).

> **Gotcha — for real secrets, graduate beyond Connections/Variables.** In production, back credentials with a dedicated **secrets manager** (a secure external credential store such as AWS Secrets Manager or HashiCorp Vault), which Airflow can integrate with. The principle you learned holds: secrets never live in the `.py` file.

## ✅ Hour 3 Exercise

No new DAG — practice production thinking and the new config piece:

1. **Decide and justify.** For a daily ML-training pipeline that must alert on failure and run a couple of tasks in parallel, write down which deployment option you'd choose and why, and which executor that implies.
2. **Add a Variable.** In the UI, create `quality_threshold = 0.85`.
3. **Wire it in.** Change your Hour 2 `quality_gate` to read the threshold from the Variable *inside the task* instead of hard-coding `0.85`. Flip the Variable to `0.5`, re-run, and watch the branch decision change without editing code.
4. **Map the upgrade path.** In one or two sentences, describe what it would take to move this pipeline from local Astro to either MWAA or Astronomer's managed cloud — and what the MWAA major-version gotcha would mean if you were currently on Airflow 2.

---

## 🎤 Hour 3 — Interview Questions & Answers

**Q1 (conceptual). Compare self-hosting Airflow with a managed service like MWAA or Astronomer.**
Self-hosting (Docker Compose or Kubernetes+Helm) gives maximum control and lowest licensing cost but makes you responsible for upgrades, patching, scaling, backups, and outages. Managed services run the infrastructure for you — MWAA on AWS (you supply DAGs via S3; provisioned or serverless), Astronomer via the same Astro CLI you use locally — trading money and some low-level control for far less operational burden.

**Q2 (practical). What's the difference between an Airflow Connection and an Airflow Variable, and when do you use each?**
A Connection stores *credentials* for an external system (host, login, password, extras), referenced by id from hooks/operators. A Variable stores a *configuration value* (a threshold, path, flag) read with `Variable.get()`. Use Connections for anything authentication-related; use Variables for environment-specific settings you want to change without editing code.

**Q3 (gotcha). On MWAA an engineer tries to upgrade from Airflow 2.x to 3.x by changing the version on the existing environment, and it fails. Why?**
That's a *major* version jump. MWAA supports in-place changes only between *minor* versions; a major upgrade requires creating a *new* environment and migrating DAGs, requirements, and metadata. Also, requirements must be constrained to the target version or the update can fail and roll back.

**Q4 (gotcha). Why is calling `Variable.get()` at the top level of a DAG file a problem in production?**
Top-level code runs on every DAG parse, which is frequent. `Variable.get()` queries the metadata database, so a top-level call means a DB hit on every parse of that file — multiplied across DAGs and parse cycles it loads the database and slows the scheduler. Read Variables inside tasks or via Jinja so the lookup happens at run time.

**Q5 (practical). You're moving a working local pipeline to production and need it to run several tasks in parallel. What changes, and where?**
The executor must change from the local default to one that parallelizes — LocalExecutor (single machine) or Celery/Kubernetes (distributed) — configured via environment variables/`airflow.cfg`, not in the DAG. You also need a real metadata database (PostgreSQL/MySQL) instead of SQLite, and a deployment that keeps the components running (self-hosted infra, MWAA, or Astronomer).

---

# Hour 4 — Build: a training DAG that logs to MLflow

**Goal:** combine everything into one realistic pipeline — **ingest → preprocess → train → (branch) → log to MLflow** — with retries, a failure callback, a trigger rule, and a Variable.

## 4.1 What MLflow is and why we add it

So far tasks just `print` results. Real ML needs to *remember* every run: the settings used, the resulting scores, and the model file itself — so runs are comparable, reproducible, and deployable. **MLflow** is a free, open-source tool for **experiment tracking** that does exactly this. Its vocabulary:

- **Tracking server** — the MLflow service that stores records and serves a web UI.
- **Tracking URI** — the address records are sent to (e.g., `http://localhost:5000`, or `http://host.docker.internal:5000` from inside a container — see the earlier gotcha).
- **Experiment** — a named group of related runs (e.g., all "iris-classifier" runs).
- **Run** — one execution of training, with its settings and results attached.
- **Parameter** — an input setting you chose (e.g., `n_estimators=100`).
- **Metric** — a measured result number (e.g., `accuracy=0.93`).
- **Artifact** — an output file from the run, most importantly the trained model.
- **Model registry** — the catalog of models promoted from runs, for versioning and deployment. This is the "register" stage from Hour 1.

> **Airflow orchestrates; MLflow records.** They're independent services. Your Airflow task simply *calls* the MLflow library to log things. Keep both running; neither depends on the other to function.

## 4.2 Logging to MLflow from inside a task

Inside a `@task`, point MLflow at the server, pick an experiment, open a run, and log. The `with mlflow.start_run():` block auto-closes the run even if an error occurs inside it.

```python
import mlflow

mlflow.set_tracking_uri("http://host.docker.internal:5000")  # inside the task!
mlflow.set_experiment("iris-classifier")
with mlflow.start_run() as run:
    mlflow.log_param("n_estimators", 100)
    mlflow.log_metric("accuracy", 0.93)
    mlflow.sklearn.log_model(model, name="model")   # see version gotcha below
    run_id = run.info.run_id
```

> **Gotcha — set the tracking URI *inside* the task, not at module level.** Same reason you already know: top-level code runs on every parse. Configuration and logging belong inside the `@task`. (Cleaner still: set the `MLFLOW_TRACKING_URI` environment variable, which MLflow reads automatically.)

> **Gotcha — `log_model`'s signature changed across MLflow versions.** Older MLflow used `mlflow.sklearn.log_model(model, artifact_path="model")`; recent 3.x prefers `name="model"`. If one errors on your installed version, use the other. `log_param`/`log_metric` are stable; model logging is the part that shifted — the code below tries both.

## 4.3 The complete pipeline

```python
# dags/iris_training_pipeline.py
from datetime import datetime, timedelta
from airflow.sdk import dag, task, Variable

MLFLOW_URI = "http://host.docker.internal:5000"   # use http://localhost:5000 if not in Docker

def alert_on_failure(context):
    try:
        ti = context["task_instance"]
        print(f"ALERT: '{ti.task_id}' failed: {context.get('exception')}")
        # real life: send to Slack / email here
    except Exception as e:
        print(f"alert callback errored: {e}")

default_args = {
    "retries": 2,
    "retry_delay": timedelta(seconds=30),
    "retry_exponential_backoff": True,
    "on_failure_callback": alert_on_failure,
}

@dag(
    start_date=datetime(2024, 1, 1),
    schedule=None,
    catchup=False,
    default_args=default_args,
    tags=["course", "hour4", "ml"],
)
def iris_training_pipeline():

    @task
    def ingest() -> str:
        from sklearn.datasets import load_iris
        df = load_iris(as_frame=True).frame
        path = "/tmp/iris_raw.parquet"
        df.to_parquet(path)
        print(f"Ingested {len(df)} rows -> {path}")
        return path

    @task
    def preprocess(raw_path: str) -> dict:
        import pandas as pd
        df = pd.read_parquet(raw_path).dropna()
        clean = "/tmp/iris_clean.parquet"
        df.to_parquet(clean)
        quality = len(df) / 150.0
        print(f"Preprocessed -> {clean} (quality={quality:.2f})")
        return {"path": clean, "quality": quality}

    @task
    def train(prep: dict) -> dict:
        import mlflow, pandas as pd
        from sklearn.ensemble import RandomForestClassifier
        from sklearn.model_selection import train_test_split
        from sklearn.metrics import accuracy_score

        df = pd.read_parquet(prep["path"])
        X, y = df.drop(columns=["target"]), df["target"]
        X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)

        n_estimators = 100
        model = RandomForestClassifier(n_estimators=n_estimators, random_state=42)
        model.fit(X_tr, y_tr)
        acc = accuracy_score(y_te, model.predict(X_te))
        print(f"Trained, accuracy={acc:.3f}")

        mlflow.set_tracking_uri(MLFLOW_URI)
        mlflow.set_experiment("iris-classifier")
        with mlflow.start_run() as run:
            mlflow.log_param("n_estimators", n_estimators)
            mlflow.log_param("model_type", "RandomForestClassifier")
            mlflow.log_metric("accuracy", acc)
            mlflow.log_metric("train_rows", len(X_tr))
            try:
                mlflow.sklearn.log_model(model, name="model")
            except TypeError:
                mlflow.sklearn.log_model(model, artifact_path="model")
            run_id = run.info.run_id
        return {"accuracy": acc, "run_id": run_id}

    # branch (Hour 2): only register if the model clears the bar
    @task.branch
    def quality_gate(result: dict) -> str:
        threshold = float(Variable.get("quality_threshold", default=0.85))  # Hour 3
        return "register" if result["accuracy"] >= threshold else "alert_low_quality"

    @task
    def register(result: dict):
        import mlflow
        mlflow.set_tracking_uri(MLFLOW_URI)
        model_uri = f"runs:/{result['run_id']}/model"
        mlflow.register_model(model_uri, "iris-classifier-prod")
        print(f"Registered {model_uri}")

    @task
    def alert_low_quality(result: dict):
        print(f"Accuracy {result['accuracy']:.3f} below bar — not registering.")

    # join: runs either way (Hour 2 trigger rule)
    @task(trigger_rule="none_failed_min_one_success")
    def finalize():
        print(f"Done. See MLflow at {MLFLOW_URI}")

    # cleanup: always runs, even on failure (Hour 2 trigger rule)
    @task(trigger_rule="all_done")
    def cleanup():
        import os, glob
        for f in glob.glob("/tmp/iris_*.parquet"):
            os.remove(f)
        print("Temp files removed.")

    prep = preprocess(ingest())
    result = train(prep)
    decision = quality_gate(result)

    reg = register(result)
    alert = alert_low_quality(result)

    decision >> [reg, alert]
    [reg, alert] >> finalize() >> cleanup()

iris_training_pipeline()
```

## 4.4 Run it and read the results

1. Ensure your local Airflow (`astro dev start`) and `mlflow server` are both running, and the Variable `quality_threshold` exists (Hour 3).
2. Drop the file in `dags/`; wait for it to appear.
3. Trigger it. Watch: `ingest → preprocess → train → quality_gate`, then one of `register`/`alert_low_quality` runs and the other shows **skipped**, then `finalize` runs (not skipped — your trigger rule), then `cleanup` runs.
4. Open MLflow at the tracking URI, click the **iris-classifier** experiment, and inspect the run's params, metrics, and model artifact. If accuracy cleared the threshold, check the **Models** section for `iris-classifier-prod`.

## ✅ Hour 4 Exercise

1. **Log a third metric** in `train` (e.g., a per-class score) and confirm it shows up in MLflow's run comparison.
2. **Lower the bar** by setting `quality_threshold` to `0.99`, re-run, and confirm the branch routes to `alert_low_quality` and `register` is skipped — while `finalize` and `cleanup` still run.
3. **Prove resilience** by temporarily adding `raise RuntimeError("test")` inside `train`: confirm two retries (with growing delay), then the failure callback fires, then `cleanup` *still* runs because of its `all_done` rule. Remove the `raise`.
4. **Reflect:** write one sentence on what would change to point this at a *remote* MLflow server and a *cloud* storage path for artifacts in production.

---

## 🎤 Hour 4 — Interview Questions & Answers

**Q1 (conceptual). What does MLflow give you that Airflow doesn't?**
Airflow orchestrates — runs steps in order, on schedule, with retries and visibility. MLflow tracks experiments — records each run's parameters, metrics, and model artifacts, and catalogs promoted models in a registry. Airflow runs the training; MLflow remembers and versions what each run produced. They're complementary.

**Q2 (practical). Where in an Airflow task should `mlflow.set_tracking_uri()` go, and why?**
Inside the task function, never at module level — top-level code runs on every DAG parse, so configuration there executes pointlessly and repeatedly. Inside the `@task` it runs only at execution. Setting `MLFLOW_TRACKING_URI` as an environment variable is an even cleaner alternative.

**Q3 (gotcha). Your Astro-hosted task logs to `http://localhost:5000` but can't reach the MLflow server you started on your laptop. Why, and what's the fix?**
The task runs inside a Docker container, where `localhost` refers to the container, not your host. The MLflow server is on the host. On Docker Desktop, use `http://host.docker.internal:5000` from inside the container (or run MLflow as a service in the same Docker network).

**Q4 (gotcha). In the capstone, `register` is skipped when the model is below threshold, yet `finalize` still runs. What makes that work, and what would break it?**
`finalize` has `trigger_rule="none_failed_min_one_success"`, which ignores the skipped branch and runs as long as nothing failed and at least one upstream succeeded. Leaving it on the default `all_success` would break it: the skipped `register` would cascade and skip `finalize` too.

**Q5 (conceptual). How does the pipeline connect a trained model to the registry, and what's the role of the run id?**
`train` opens an MLflow run, logs the model as an artifact under that run, and returns the `run_id`. `register` builds the model URI `runs:/<run_id>/model` and calls `mlflow.register_model(...)`. The run id is the link between the logged artifact and the registry entry, giving you lineage from a registered model back to the exact run, params, and metrics that produced it.

---

# 📋 Gotchas summary table (new material only)

*Your base tutorial's gotchas table already covers logical date, catchup, UTC, XCom size/serialization, top-level code, calling `@dag`, sensor poke mode, FileSensor-in-Docker, pip rebuilds, hardcoded creds, Bash quirks, and Docker RAM. These are the additional ones from this course.*

| # | Gotcha | Where it bites | Fix |
|---|--------|----------------|-----|
| 1 | Copying imports from older (2.x) tutorials or blogs | Import errors / deprecation warnings | Use `airflow.sdk` and `airflow.providers.standard.*` |
| 2 | Join task skipped after a branch | Default `all_success` cascades skips | Set join's `trigger_rule="none_failed_min_one_success"` |
| 3 | Branch returns a non-direct / blank target | Branch errors or skips everything | Return a *direct* downstream task id; use an explicit placeholder for "do nothing" |
| 4 | Retries on a non-idempotent task | Duplicate data / double side-effects | Make tasks idempotent (overwrite, don't append) first |
| 5 | Exponential backoff omitted on rate-limited sources | Hammering worsens the outage | `retry_exponential_backoff=True` |
| 6 | `on_failure_callback` expected on every retry | Confusion about alerts | It fires only after the *final* attempt; use `on_retry_callback` for retries |
| 7 | Heavy/throwing callback code | Missed alerts | Keep callbacks tiny, fast, wrapped in try/except |
| 8 | `Variable.get()` at module level | Scheduler slows (DB hit per parse) | Read Variables inside tasks, or via Jinja |
| 9 | MWAA `requirements.txt` unconstrained | Failed environment update + slow rollback | Pin with a `--constraint` for the target version |
| 10 | In-place MWAA major upgrade | Upgrade "doesn't work" | Major upgrades = new environment + migration; only minor versions are in-place |
| 11 | `localhost` for MLflow from inside a container | Task can't reach host MLflow server | Use `http://host.docker.internal:5000` (Docker Desktop) or same-network service |
| 12 | `mlflow...log_model` signature mismatch | Errors on model logging | `name=` (newer) vs `artifact_path=` (older); try/except both |
| 13 | No fixed random seed in training | Non-reproducible, incomparable runs | Set `random_state`/seed everywhere randomness enters |
| 14 | `set_tracking_uri` at module level | Runs on every parse | Call it inside the task (or set `MLFLOW_TRACKING_URI`) |

---

# 🗂️ Quick reference card

### Airflow 3 imports (the ones you'll use most)
```python
from airflow.sdk import dag, task, DAG, Variable, chain, Label
from airflow.providers.standard.operators.python import PythonOperator, BranchPythonOperator
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.providers.standard.sensors.filesystem import FileSensor
```

### Branching
```python
@task.branch
def gate(metric):
    return "task_yes" if metric >= 0.85 else "task_no"   # return the downstream task id

gate(metric) >> [task_yes(), task_no()]
```

### Trigger rules (most used)
| Rule | Runs when… |
|------|-----------|
| `all_success` *(default)* | every upstream succeeded |
| `none_failed_min_one_success` | nothing failed + ≥1 succeeded — **use after branches** |
| `all_done` | every upstream finished, any result — cleanup tasks |
| `one_success` | any upstream succeeded |
| `one_failed` | any upstream failed — alert tasks |

### Retry strategy + failure alert
```python
@task(retries=3, retry_delay=timedelta(minutes=2),
      retry_exponential_backoff=True,
      on_failure_callback=alert_fn)
def step(): ...

def alert_fn(context):
    print(context["task_instance"].task_id, context.get("exception"))
```

### Variables (config, not credentials)
```python
from airflow.sdk import Variable
threshold = float(Variable.get("quality_threshold", default=0.85))  # inside a task
```

### MLflow from a task
```python
import mlflow
mlflow.set_tracking_uri("http://host.docker.internal:5000")  # inside the task
mlflow.set_experiment("my-experiment")
with mlflow.start_run() as run:
    mlflow.log_param("n_estimators", 100)
    mlflow.log_metric("accuracy", 0.93)
    mlflow.sklearn.log_model(model, name="model")   # or artifact_path="model"
    run_id = run.info.run_id
mlflow.register_model(f"runs:/{run_id}/model", "my-model-prod")
```

### Deployment options at a glance
| Option | Who runs it | Best when |
|--------|-------------|-----------|
| Self-hosted (Docker / Kubernetes+Helm) | You | Max control, lowest licensing cost, platform engineers on hand |
| Amazon MWAA (provisioned or serverless) | AWS | On AWS, want managed infra; DAGs go to S3 |
| Astronomer (Astro) | Astronomer | Smoothest local-to-prod (`astro deploy`); vendor-managed |

### The ML pipeline backbone
```
ingest → preprocess → train → evaluate → register
```
Ingest = fetch raw · Preprocess = clean · Train = fit (fixed seed!) · Evaluate = score on held-out data · Register = catalog the approved model in MLflow.

---

*This builds directly on your existing Airflow knowledge — re-run the exercises until the new pieces (branching, trigger rules, callbacks, MLflow) feel as natural as the basics you already have.*
