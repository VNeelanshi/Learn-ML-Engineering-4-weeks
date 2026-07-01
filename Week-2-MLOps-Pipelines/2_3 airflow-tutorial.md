# Apache Airflow: A Complete Beginner's Tutorial

> **What you'll build:** A fully functioning multi-task DAG with sensors, hooks, and cross-task communication

> **Version note:** This tutorial targets **Apache Airflow 3** (3.0 went GA in 2025; the series is on 3.2 as of mid-2026). Airflow 2 reached end-of-life in April 2026. Airflow 3 kept the concepts you'll learn here identical but changed several **import paths** and a few component names, so all code below uses the current Airflow 3 conventions — if you copy snippets from older tutorials and hit import errors, a 2.x→3.x path change is usually why.

---

## How This Tutorial Is Organized

Each "hour" block follows the same pattern: we explain the *problem* first, then introduce the Airflow concept that solves it, walk through the details with examples, and finish with a hands-on exercise. At the end of each block you'll find five interview-style questions with detailed answers.

---

# Hour 1 — Airflow Concepts

## 1.1 The Problem: "Run This After That, Every Day, and Tell Me if It Breaks"

Imagine you work at a company that processes data. Every morning at 6 AM, you need to:

1. Download yesterday's sales data from an API.
2. Clean and transform that data.
3. Load it into a database.
4. Send a summary email to your manager.

These steps must happen **in order** — you can't load data you haven't cleaned yet. They must happen **on a schedule** — every day, automatically. And when something fails (the API is down, the database is full), you need to know about it and be able to retry.

You could write a cron job (a Unix scheduling tool) and a long Python script, but that falls apart quickly:

- How do you retry just the failed step without re-running everything?
- How do you see which step is running right now?
- How do you handle the situation where Monday's job is still running when Tuesday's is supposed to start?
- How do you tell step 3 where step 2 saved its output?

This is the **workflow orchestration** problem — coordinating multi-step processes that run on schedules, depend on each other, and need monitoring. Apache Airflow solves it.

## 1.2 What Is Apache Airflow?

**Apache Airflow** is an open-source platform for writing, scheduling, and monitoring workflows. You define your workflow as Python code, Airflow runs it on a schedule, and you get a web dashboard to monitor everything.

The key idea: **workflows as code**. Instead of clicking through a GUI to wire steps together, you write a Python file that describes what to do, in what order, and when. This means your workflows are version-controlled, testable, and reviewable — just like any other code.

Airflow was created at Airbnb in 2014 and became an Apache Software Foundation top-level project in 2019. It's used by thousands of organizations for data pipelines, ML workflows, infrastructure automation, and more. The current major version is **Airflow 3**, a significant release that reworked the internal architecture (a new API server and a stable authoring interface called the Task SDK) while keeping day-to-day DAG authoring familiar.

## 1.3 The Core Vocabulary

Airflow has a specific vocabulary. Every term below will appear repeatedly throughout this tutorial, so let's define them all upfront.

### DAG (Directed Acyclic Graph)

A **DAG** is Airflow's word for a workflow. It's a collection of tasks organized with dependencies that define the order they run in.

The name comes from graph theory. "Directed" means the connections between tasks have a direction (task A runs *before* task B, not the other way around). "Acyclic" means there are no loops — task A can't depend on task B while task B also depends on task A.

In practical terms, a DAG is a single Python file that defines:

- **What** tasks to run (download data, clean data, etc.).
- **When** to run them (every day at 6 AM, every hour, etc.).
- **In what order** (download before clean, clean before load).

Here's the smallest possible DAG:

```python
from airflow.sdk import DAG
from airflow.providers.standard.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id="my_first_dag",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
) as dag:
    task1 = BashOperator(
        task_id="say_hello",
        bash_command="echo Hello, Airflow!",
    )
```

> **Airflow 3 imports:** Notice `from airflow.sdk import DAG`. In Airflow 3, the core authoring objects (`DAG`, the `@dag`/`@task` decorators, `Variable`, `BaseHook`, etc.) come from the **`airflow.sdk`** package — a stable interface meant to outlast internal refactors. Built-in operators and sensors now live under **`airflow.providers.standard.*`** (e.g. `airflow.providers.standard.operators.bash`). In Airflow 2 these were `from airflow import DAG` and `from airflow.operators.bash import BashOperator`; the old paths are deprecated. Everything else — `schedule`, `>>`, XCom, `default_args` — works exactly the same.

An **operator** is a template for a task — it defines *what kind of work* to do. Think of an operator as a blueprint, and a task as a specific building made from that blueprint.

Airflow provides many built-in operators:

- **BashOperator** — runs a bash shell command (like `echo hello` or `python script.py`).
- **PythonOperator** — runs a Python function you define.
- **EmailOperator** — sends an email.
- **S3ToRedshiftOperator** — copies data from Amazon S3 into a Redshift database.

There are hundreds more, organized into "provider packages" for different services (AWS, Google Cloud, Slack, databases, etc.).

When you write `BashOperator(task_id="say_hello", bash_command="echo hi")`, you're creating a **task** — a specific instance of the BashOperator with concrete parameters.

### Task and Task Instance

A **task** is a single unit of work in a DAG — one node in the graph. It's what you get when you instantiate an operator with specific arguments.

A **task instance** is a specific execution of a task for a specific date. If your DAG runs daily, the task "download_data" generates one task instance per day: `download_data` for January 1st, `download_data` for January 2nd, and so on.

Task instances have states: `queued`, `running`, `success`, `failed`, `up_for_retry`, `skipped`, and others. The Airflow UI shows these as colored circles so you can see at a glance what's happening.

### DAG Run

A **DAG run** is a single execution of your entire DAG for a specific date. If your DAG runs daily, you get one DAG run per day. Each DAG run contains one task instance for every task in the DAG.

DAG runs have a **logical date** (called the "execution date" in Airflow 2). This can be confusing: the logical date is the date the DAG run *represents*, not necessarily when it actually executes. A daily DAG processing yesterday's data will have yesterday's date as its logical date, even though it runs today.

> **⚠️ Airflow 3 change:** The term `execution_date` and the `execution_date` context/template variable were **removed in Airflow 3** — use `logical_date` instead. Several related context keys (`prev_ds`, `next_ds`, `tomorrow_ds`, `yesterday_ds`, etc.) were also removed. The everyday ones you'll use — `{{ ds }}`, `ti`, `dag_run` — are unchanged.

> **⚠️ Gotcha:** The logical date / execution date confusion trips up almost every Airflow beginner. A DAG scheduled with `schedule="@daily"` and `start_date=datetime(2024, 1, 1)` will first run on January 2nd, processing data for January 1st. The logical date will be January 1st. This is because Airflow waits for the interval to *end* before running. The DAG for "January 1st" runs at the *start* of January 2nd.

### Scheduler

The **scheduler** is the brain of Airflow. It's a long-running process that:

1. Reads all DAG files from a designated folder.
2. Determines which DAGs need to run based on their schedules.
3. Creates DAG runs and task instances at the appropriate times.
4. Sends tasks to the executor to be run.
5. Monitors running tasks and handles retries.

If the scheduler isn't running, nothing happens — DAGs may appear in the UI, but no tasks will execute.

> **Airflow 3 change:** In Airflow 3, parsing the DAG files (step 1 above) was split out into a **separate component called the DAG processor**, rather than running inside the scheduler. You don't have to manage it directly, but you'll see it listed as its own service when Airflow starts.

### Executor

The **executor** determines *how* tasks are actually run. The scheduler decides *what* should run; the executor handles the mechanics.

Common executors:

- **LocalExecutor** — runs tasks as separate processes on the same machine; can run several in parallel. In Airflow 3 this is the simplest executor and it works even with a SQLite database, so it's the usual choice for local development. (In Airflow 2, fresh installs defaulted to the **SequentialExecutor**, which ran one task at a time with SQLite; the SequentialExecutor was **removed in Airflow 3** and LocalExecutor absorbed its role.)
- **CeleryExecutor** — distributes tasks across multiple worker machines using a message queue (like Redis or RabbitMQ). Used for large-scale production deployments.
- **KubernetesExecutor** — spins up a new Kubernetes pod for each task. Provides strong isolation and dynamic resource allocation.

For local development, you'll typically use the LocalExecutor.

### API server (the web UI)

The **API server** — called the **webserver** in Airflow 2 — serves the Airflow web UI and the REST API: the dashboard where you see your DAGs, trigger runs, view logs, and monitor task states, plus the programmatic interface other systems use. In Airflow 3 this component was rebuilt as a FastAPI service behind a new React UI. It doesn't execute tasks; it provides visibility and control.

### Metadata Database

Airflow stores all of its state — DAG definitions, task instance statuses, schedules, XCom values, connection credentials, and more — in a relational database. This is typically PostgreSQL or MySQL (SQLite for development only).

Every component (scheduler, API server, DAG processor, workers) relies on this database — it's the shared source of truth. (One Airflow 3 detail: task code no longer reads/writes this database directly; it goes through a Task Execution API, which improves security. This rarely matters when you're starting out.)

### XCom (Cross-Communication)

**XCom** (short for "cross-communication") is Airflow's mechanism for passing small pieces of data between tasks. When one task needs to tell another task something — like "I saved the file at this path" or "I processed 1,247 rows" — it uses XCom.

XComs work through push and pull:

- A task **pushes** an XCom value (or simply returns a value from a PythonOperator, which is auto-pushed).
- A downstream task **pulls** that value using the source task's ID.

```python
# Task A pushes
def extract(**context):
    filepath = "/tmp/data_2024_01_01.csv"
    # do some work...
    return filepath  # This value is automatically pushed as an XCom

# Task B pulls
def transform(**context):
    ti = context["ti"]
    filepath = ti.xcom_pull(task_ids="extract_task")
    # Now filepath == "/tmp/data_2024_01_01.csv"
```

> **⚠️ Gotcha:** XComs are stored in the metadata database. They are designed for **small** values — a file path, a row count, a status flag. Do **not** push large objects (DataFrames, large JSON blobs, entire files) through XCom. This will bloat your database and slow everything down. Instead, write large data to a shared storage location (like S3) and pass only the *path* through XCom.

## 1.4 How the Pieces Fit Together

Here's the flow from writing a DAG to seeing results:

```
You write a Python DAG file
        ↓
Place it in the DAGs folder
        ↓
The SCHEDULER scans the folder, parses the file
        ↓
The scheduler creates DAG RUNS on schedule
        ↓
For each DAG run, the scheduler creates TASK INSTANCES
        ↓
The scheduler sends task instances to the EXECUTOR
        ↓
The executor runs the tasks (on the local machine, on workers, in pods...)
        ↓
Results, states, and XComs are stored in the METADATA DATABASE
        ↓
The API SERVER reads from the database and shows you everything in the UI
```

## 1.5 How Dependencies Work

In a DAG, you define task dependencies using the `>>` and `<<` operators (or `.set_downstream()` / `.set_upstream()` methods):

```python
# task_a runs before task_b
task_a >> task_b

# Equivalent:
task_a.set_downstream(task_b)

# Fan-out: task_a runs before both task_b and task_c
task_a >> [task_b, task_c]

# Fan-in: both task_b and task_c must complete before task_d
[task_b, task_c] >> task_d

# Full chain:
task_a >> [task_b, task_c] >> task_d
```

This creates a graph:

```
    task_a
    /    \
task_b  task_c
    \    /
    task_d
```

Airflow guarantees that a task only runs after all of its upstream tasks have succeeded (by default).

## 1.6 Schedule Expressions

The `schedule` parameter on a DAG controls when it runs. Common options:

| Expression | Meaning |
|-----------|---------|
| `"@once"` | Run exactly once |
| `"@hourly"` | Every hour |
| `"@daily"` | Every day at midnight |
| `"@weekly"` | Every Sunday at midnight |
| `"@monthly"` | First day of each month at midnight |
| `"0 6 * * *"` | Cron expression: every day at 6:00 AM |
| `None` | Never scheduled — only manual triggers |
| `timedelta(hours=2)` | Every 2 hours |

> **⚠️ Gotcha:** All times in Airflow are in **UTC** by default, not your local timezone. If you set `schedule="0 6 * * *"`, your DAG runs at 6 AM UTC. If you're in New York (UTC-5), that's 1 AM local time. You can configure a timezone on the DAG, but be aware of this default.

## 1.7 `default_args`: Shared Task Settings

Instead of repeating the same settings on every task, you can define `default_args` — a dictionary of parameters applied to all tasks in the DAG:

```python
default_args = {
    "owner": "data-team",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "email_on_failure": True,
    "email": ["team@company.com"],
}

with DAG(
    dag_id="my_dag",
    default_args=default_args,
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
) as dag:
    # All tasks inherit retries=3, retry_delay=5min, etc.
    ...
```

The `catchup` parameter controls whether Airflow backfills all the DAG runs between the `start_date` and today. **Airflow 3 changed the default to `False`** (in Airflow 2 it defaulted to `True`), so a fresh Airflow 3 DAG won't flood you with historical runs. It's still best practice to set `catchup=False` explicitly so your intent is clear and the DAG behaves the same on any environment. If you *do* set `catchup=True` with a `start_date` a year ago, Airflow will try to create ~365 DAG runs as soon as you unpause.

> **⚠️ Gotcha:** In Airflow 2, forgetting `catchup` meant it defaulted to `True` — you'd deploy a DAG with `start_date=datetime(2023, 1, 1)`, unpause it, and suddenly Airflow queued hundreds of "catch up" runs. Airflow 3 flipped the default to `False`, which removes that footgun, but set `catchup=False` explicitly anyway so the behavior is unmistakable and portable to older environments.

## 1.8 Hands-On Exercise: Map the Concepts

No code yet — this exercise is about building your mental model.

**Scenario:** You're building a daily pipeline that:

1. At 7 AM UTC, checks if a CSV file exists in an S3 bucket.
2. Once found, downloads the file.
3. Runs a Python script to clean the data.
4. Loads the clean data into a PostgreSQL database.
5. Sends a Slack notification that the pipeline completed.

**Tasks:**

1. Draw the DAG on paper. Label each task and the dependencies between them.
2. For each task, identify which operator type you'd use (you can look at the table above or think about what each task does).
3. Identify which pieces of information would need to be passed between tasks via XCom (hint: file paths).
4. What should happen if step 3 (cleaning) fails? What `default_args` would you set?
5. If the CSV is always named `sales_YYYY-MM-DD.csv`, which part of the filename would come from the DAG run's logical date?

**Expected answers:**

1. Linear chain: check_s3 → download → clean → load → notify.
2. check_s3: S3KeySensor (a sensor, not a regular operator); download: PythonOperator or S3Hook; clean: PythonOperator; load: PythonOperator (using a PostgreSQL hook); notify: SlackOperator.
3. The download path needs to be passed from download → clean. The row count might be passed from clean → load → notify.
4. Set `retries=3` and `retry_delay=timedelta(minutes=5)` in `default_args`. Also set `email_on_failure=True` with a team email.
5. The `YYYY-MM-DD` part would come from the logical date template: `{{ ds }}` (a Jinja template variable Airflow provides that expands to the logical date in `YYYY-MM-DD` format).

---

### Hour 1 — Interview Questions

**Q1 (Conceptual): What is a DAG in Airflow, and why is it "acyclic"?**

A DAG (Directed Acyclic Graph) is a collection of tasks organized with dependencies that define their execution order. "Directed" means dependencies have a direction — task A runs before task B. "Acyclic" means there are no circular dependencies — you can't have task A depend on task B while task B also depends on task A. This constraint is essential because circular dependencies would create infinite loops, making it impossible for the scheduler to determine what to run first. Airflow validates this at parse time and will reject any DAG containing cycles.

**Q2 (Practical): Explain the difference between an operator, a task, and a task instance.**

An operator is a class that serves as a template for a piece of work — like BashOperator or PythonOperator. A task is a specific instance of an operator with concrete parameters — when you write `BashOperator(task_id="say_hello", bash_command="echo hi")`, that's a task. A task instance is a specific execution of that task for a specific DAG run (date). The task "say_hello" might have hundreds of task instances, one for each date the DAG has run. Task instances are what carry state (running, success, failed) and what you see in the Airflow UI.

**Q3 (Gotcha): You set `start_date=datetime(2023, 1, 1)` and `schedule="@daily"` on a new DAG. When you unpause it, hundreds of runs appear. What happened and how do you fix it?**

Airflow is performing "catchup" — creating DAG runs for every interval between the `start_date` and now — which means `catchup` was set to `True` (in Airflow 2 that was the *default*; Airflow 3 defaults to `False`). The fix is `catchup=False` on the DAG, so Airflow only creates runs from the current time forward. For DAGs already backfilling, mark the unwanted runs as `success` in the UI or clear them.

**Q4 (Conceptual): What are the roles of the scheduler, the executor, and the API server (web UI)? Could Airflow work without it?**

The scheduler monitors DAG files, creates DAG runs on schedule, creates task instances, and sends them to the executor. The executor determines *how* tasks actually run — locally as processes (LocalExecutor), on remote workers (CeleryExecutor), or in Kubernetes pods (KubernetesExecutor). The API server (the web server, rebuilt on FastAPI in Airflow 3) serves the UI and REST API for monitoring and manual control. Yes, Airflow can run tasks without it — the scheduler and executor work independently — you'd just lose the UI and rely on the CLI. The API server is for visibility, not execution.

**Q5 (Gotcha): Your colleague pushed a 50 MB JSON object through XCom and now the Airflow UI is slow. What went wrong?**

XCom values are stored in the Airflow metadata database. Pushing 50 MB into the database bloats it, slowing down every query the scheduler and API server make — not just XCom reads. The metadata database is designed for small operational data, not bulk storage. The fix is to write the large data to external storage (S3, GCS, a shared filesystem) and pass only the *path* or *reference* through XCom. For future prevention, consider setting up a custom XCom backend that uses object storage, or simply enforcing a team policy to never push large values.

---

# Hour 2 — Local Airflow Setup

## 2.1 The Problem: "How Do I Actually Run This Thing?"

You understand the concepts. Now you need a running Airflow instance on your machine where you can write DAGs, see them in the UI, trigger runs, and debug failures. Airflow is a multi-component system (scheduler, API server, DAG processor, database), so getting everything running used to be painful. Fortunately, modern tooling has made this much easier.

We'll cover two approaches: the **Astro CLI** (recommended — fastest and simplest) and **Docker Compose** (the official Apache approach — more manual but gives you full control).

## 2.2 Option A: Astro CLI (Recommended for Beginners)

The **Astro CLI** is a free, open-source command-line tool maintained by Astronomer (a company that offers a managed Airflow platform). It wraps Docker Compose behind a simple interface, so you can go from zero to a running Airflow environment in about three commands.

### Step 1 — Install the Astro CLI

**macOS (Homebrew):**
```bash
brew install astro
```

**Windows (winget):**
```bash
winget install -e --id Astronomer.Astro
```

**Linux (curl):**
```bash
curl -sSL install.astronomer.io | sudo bash -s
```

### Step 2 — Create a Project

```bash
mkdir airflow-tutorial && cd airflow-tutorial
astro dev init
```

This creates a project directory with the following structure:

```
airflow-tutorial/
├── dags/             ← Your DAG files go here
│   └── example_astronauts.py
├── include/          ← Supporting files (SQL, data, etc.)
├── plugins/          ← Custom Airflow plugins
├── tests/            ← Tests for your DAGs
├── Dockerfile        ← Defines the Airflow Docker image
├── packages.txt      ← OS-level packages to install
└── requirements.txt  ← Python packages to install
```

### Step 3 — Start Airflow

```bash
astro dev start
```

This command builds a Docker image and starts containers for the API server, scheduler, DAG processor, triggerer, and a PostgreSQL database. It takes 1–3 minutes the first time.

When it's done, you'll see output like:

```
Airflow is starting up!
Project is running! All components are healthy.
Airflow UI: http://localhost:8080
Postgres Database: localhost:5432/postgres
The default Airflow UI credentials are: admin:admin
```

Open **http://localhost:8080** in your browser and log in with `admin` / `admin`.

### Useful Astro CLI Commands

| Command | What It Does |
|---------|-------------|
| `astro dev start` | Start the Airflow environment |
| `astro dev stop` | Stop without deleting data |
| `astro dev restart` | Rebuild image and restart (use after changing `requirements.txt`) |
| `astro dev kill` | Stop and delete all data (full reset) |
| `astro dev bash` | Open a shell inside the scheduler container |
| `astro dev run <command>` | Run an Airflow CLI command (e.g., `astro dev run dags list`) |

> **⚠️ Gotcha:** If you add a new Python package to `requirements.txt`, running `astro dev stop` followed by `astro dev start` is not enough — the image isn't rebuilt. Use `astro dev restart`, which rebuilds the Docker image with the new packages.

## 2.3 Option B: Docker Compose (Official Apache Method)

If you prefer the raw Apache Airflow approach (no Astronomer), you can use the official `docker-compose.yaml`:

### Step 1 — Download the Compose File

```bash
mkdir airflow-docker && cd airflow-docker
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'
```

### Step 2 — Create Required Directories

```bash
mkdir -p ./dags ./logs ./plugins ./config
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

### Step 3 — Initialize the Database

```bash
docker compose up airflow-init
```

### Step 4 — Start Everything

```bash
docker compose up -d
```

The UI will be at **http://localhost:8080** with default credentials `airflow` / `airflow`.

> **⚠️ Gotcha:** The official Docker Compose setup requires a good amount of system resources — at least 4 GB of RAM allocated to Docker. If you see containers restarting or the API server returning 503 errors, increase Docker's memory limit in Docker Desktop → Settings → Resources.

## 2.4 Touring the Web UI

With Airflow running, let's explore the UI. Open **http://localhost:8080**.

### The DAGs List Page

This is the home page. You'll see a table of all DAGs Airflow has detected. Key columns:

- **Toggle (on/off):** DAGs are paused by default. You must unpause (toggle on) a DAG before Airflow will schedule it. This is a safety feature — you don't want a new DAG immediately backfilling a year of runs.
- **DAG name:** The `dag_id` from your code.
- **Owner:** From `default_args`.
- **Runs:** Colored circles showing recent DAG run states (green = success, red = failed, yellow = running).
- **Schedule:** The schedule interval.
- **Last Run / Next Run:** Timing information.

### The Grid View (formerly Tree View)

Click on a DAG name to enter its detail page. The **Grid** view shows a matrix: columns are DAG runs (dates), and rows are tasks. Each cell is a task instance, colored by state. This is your primary debugging view.

### The Graph View

Shows the DAG structure as a visual graph — boxes for tasks, arrows for dependencies. Useful for verifying that your dependencies are correct.

### Task Instance Details

Click on any cell (task instance) in the Grid view to see:

- **Log:** The full stdout/stderr output from the task's execution. This is where you debug failures.
- **XCom:** Any XCom values pushed by this task instance.
- **Rendered Template:** Shows what template variables (like `{{ ds }}`) expanded to for this run.
- **Actions:** Buttons to clear (re-run), mark as success/failed, etc.

### The Admin Menu

Under Admin, you'll find:

- **Connections:** Configure credentials for external systems (databases, S3, APIs). This is how Airflow stores connection details without hardcoding passwords in your DAG code.
- **Variables:** Key-value pairs for configuration that shouldn't be in code (API endpoints, feature flags).
- **XComs:** Browse all XCom values currently stored.

## 2.5 Creating a Connection (You'll Need This Later)

Many operators and sensors need to connect to external systems. Airflow manages these connections through the UI (or CLI or environment variables).

To create one:

1. Go to Admin → Connections.
2. Click the "+" button.
3. Fill in:
   - **Connection Id:** A name you'll reference in your DAG code (e.g., `my_postgres`).
   - **Connection Type:** The type of system (Postgres, S3, HTTP, etc.).
   - **Host, Port, Login, Password, Schema:** The connection details.
4. Save.

In your DAG code, you'd then reference this connection by its ID:

```python
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

SQLExecuteQueryOperator(
    task_id="load_data",
    conn_id="my_postgres",   # matches the Connection Id above
    sql="INSERT INTO sales ...",
)
```

> **⚠️ Airflow 3 change:** The old database-specific operators like `PostgresOperator` (and `MySqlOperator`, etc.) were **deprecated and removed** in favor of one unified `SQLExecuteQueryOperator` (from `airflow.providers.common.sql.operators.sql`). It takes `conn_id` and `sql`, and works against any SQL database via the matching Connection. The connection-by-ID pattern itself is unchanged.

> **⚠️ Gotcha:** Never hardcode credentials in DAG files. DAG files are Python files in a shared folder — they may be version-controlled, visible to other developers, or even served through the Airflow UI. Always use Connections or Variables for sensitive values.

## 2.6 Hands-On Exercise: Set Up and Explore

**Goal:** Get a local Airflow instance running and familiarize yourself with the UI.

### Steps

1. **Install and start** Airflow using either the Astro CLI or Docker Compose (follow Section 2.2 or 2.3).

2. **Open the UI** at http://localhost:8080 and log in.

3. **Find the example DAG(s)** that came pre-installed (with Astro CLI, it's `example_astronauts`; with Docker Compose, there are many example DAGs — look for them in the toggle list).

4. **Unpause one example DAG** by clicking its toggle. Watch the DAG runs appear.

5. **Click into a DAG** and explore:
   - Switch between Grid view and Graph view.
   - Click on a task instance and read its logs.
   - Find the XCom tab for a task instance.

6. **Create a test Connection:**
   - Go to Admin → Connections.
   - Click "+".
   - Set Connection Id to `fs_default`, Connection Type to "File (path)", and leave the Extra field as `{"path": "/tmp"}`.
   - Save. (You'll use this in Hour 4.)

7. **Create a test Variable:**
   - Go to Admin → Variables.
   - Click "+".
   - Set Key to `my_name`, Value to your name.
   - Save.

---

### Hour 2 — Interview Questions

**Q1 (Practical): What is the Astro CLI, and how does it differ from running Airflow with the official Docker Compose file?**

The Astro CLI is a free, open-source command-line tool by Astronomer that wraps Docker Compose behind simple commands like `astro dev start` and `astro dev stop`. It generates a project structure, manages the Docker image, and handles configuration automatically. The official Docker Compose approach requires downloading a compose file, creating directories, setting environment variables, and running initialization commands manually. Both produce the same result (a local multi-container Airflow environment), but the Astro CLI is significantly faster and more beginner-friendly.

**Q2 (Conceptual): Why are DAGs paused by default when first deployed to Airflow?**

This is a safety mechanism. When a new DAG appears, Airflow doesn't know the author's intent — and if the DAG has a `start_date` in the past and `catchup=True`, automatically scheduling it could create many backfill runs, consuming resources and potentially writing data to production systems. (Airflow 3 defaults `catchup` to `False`, which softens this risk, but pausing-by-default remains the safety net.) By requiring an explicit unpause, Airflow ensures DAGs only run when someone has deliberately reviewed and activated them.

**Q3 (Gotcha): You added `pandas` to `requirements.txt` and restarted Airflow with `astro dev stop` then `astro dev start`, but your DAG still throws `ModuleNotFoundError: No module named 'pandas'`. Why?**

`astro dev stop` followed by `astro dev start` reuses the existing Docker image. Since `requirements.txt` changes are baked into the image at build time, you need to rebuild the image. The correct command is `astro dev restart`, which stops the environment, rebuilds the Docker image (installing the new package), and starts everything back up. Alternatively, `astro dev stop` followed by `astro dev start --no-cache` forces a rebuild.

**Q4 (Practical): Where should you store database credentials for use in your DAGs?**

In Airflow Connections (Admin → Connections in the UI). Each connection has an ID (like `my_postgres`), and your DAG code references that ID instead of containing passwords. Alternatively, you can set connections as environment variables using the format `AIRFLOW_CONN_<CONN_ID>=<connection_uri>`. For production, many teams use a secrets backend (like HashiCorp Vault or AWS Secrets Manager) which Airflow can integrate with. The critical rule: never hardcode credentials in DAG files, because those files are typically version-controlled and visible to all users.

**Q5 (Conceptual): What components does a local Airflow installation run, and what does each do?**

At minimum: (1) The **API server** (the web server, rebuilt on FastAPI in Airflow 3), which serves the UI and REST API on port 8080. (2) The **scheduler**, which decides what runs and when and sends tasks to the executor. (3) The **DAG processor**, which parses DAG files — a separate component in Airflow 3 (in Airflow 2 this lived inside the scheduler). (4) A **metadata database** (PostgreSQL typically), storing all state — DAG definitions, task instance statuses, connections, variables, XComs. (5) Optionally, a **triggerer**, which handles deferred tasks asynchronously (used by sensors in deferrable mode). In a local setup the executor is usually a LocalExecutor running inside the scheduler process, so there's no separate worker component.

---

# Hour 3 — Writing Your First DAG

## 3.1 The Problem: "I Understand the Concepts — Now How Do I Actually Write One?"

You know what DAGs, operators, and tasks are. You have Airflow running locally. Now it's time to write a real multi-task DAG from scratch, deploy it, trigger it, and see it execute.

## 3.2 DAG File Anatomy

Every DAG file follows this structure:

```python
# 1. Imports
from airflow.sdk import DAG
from airflow.providers.standard.operators.bash import BashOperator
from airflow.providers.standard.operators.python import PythonOperator
from datetime import datetime, timedelta

# 2. Default arguments
default_args = {
    "owner": "tutorial",
    "retries": 2,
    "retry_delay": timedelta(minutes=1),
}

# 3. DAG definition (using context manager)
with DAG(
    dag_id="my_dag_name",          # unique identifier
    default_args=default_args,
    description="What this DAG does",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
    tags=["tutorial"],             # for filtering in the UI
) as dag:

    # 4. Task definitions
    task_a = BashOperator(...)
    task_b = PythonOperator(...)
    task_c = BashOperator(...)

    # 5. Dependencies
    task_a >> task_b >> task_c
```

Important rules about DAG files:

- The file must be placed in the `dags/` folder.
- Airflow's scheduler periodically scans this folder and imports every Python file, looking for DAG objects.
- The file name doesn't need to match the `dag_id`, but it's good practice.
- Anything at the top level of the file runs every time the scheduler parses it (every 30 seconds by default). So don't put slow operations at the top level — no API calls, no database queries, no file reads. Only DAG/task definitions.

> **⚠️ Gotcha:** The scheduler imports your DAG file frequently to check for changes. If your DAG file contains expensive operations at the module level (like reading a large config file or making an HTTP request), this will slow down the entire scheduler. Keep top-level code minimal — just imports, default_args, and DAG/task definitions. Put actual work inside operator functions.

## 3.3 The BashOperator in Detail

The **BashOperator** runs a bash shell command. It's one of the most-used operators because it can run anything you'd type in a terminal.

```python
from airflow.providers.standard.operators.bash import BashOperator

# Simple command
greet = BashOperator(
    task_id="greet",
    bash_command="echo 'Hello from Airflow! Today is {{ ds }}'",
)

# Run a script
run_script = BashOperator(
    task_id="run_etl",
    bash_command="python /opt/airflow/scripts/etl.py --date {{ ds }}",
)

# Multi-line command
setup = BashOperator(
    task_id="setup_dirs",
    bash_command="""
        mkdir -p /tmp/airflow_data/raw
        mkdir -p /tmp/airflow_data/clean
        echo 'Directories created'
    """,
)
```

Notice the `{{ ds }}` — this is a **Jinja template** variable. Airflow replaces it at runtime with the logical date in `YYYY-MM-DD` format. Other useful template variables:

| Variable | Example Output | Meaning |
|----------|---------------|---------|
| `{{ ds }}` | `2024-01-15` | Logical date (YYYY-MM-DD) |
| `{{ ds_nodash }}` | `20240115` | Logical date without dashes |
| `{{ ts }}` | `2024-01-15T00:00:00+00:00` | Full timestamp |
| `{{ data_interval_start }}` | datetime object | Start of the data interval |
| `{{ data_interval_end }}` | datetime object | End of the data interval |

> **⚠️ Gotcha:** BashOperator commands that end with a trailing space before a newline can fail silently on some systems. If you're using a multi-line `bash_command` string, make sure there's no trailing whitespace after the last line. Also, if your bash command returns a non-zero exit code, Airflow marks the task as failed — even if the command "kind of worked."

## 3.4 The PythonOperator in Detail

The **PythonOperator** runs a Python function. This is where most of your actual business logic lives.

```python
from airflow.providers.standard.operators.python import PythonOperator

def extract_data(**kwargs):
    """Download and save raw data."""
    logical_date = kwargs["ds"]  # YYYY-MM-DD string
    print(f"Extracting data for {logical_date}")

    # Simulate downloading data
    data = {"date": logical_date, "sales": 42, "returns": 3}

    # Return value is automatically pushed as XCom
    return data

extract_task = PythonOperator(
    task_id="extract_data",
    python_callable=extract_data,
)
```

The `**kwargs` parameter receives a dictionary of context values that Airflow provides, including template variables, the task instance object, and more. The most useful keys:

- `kwargs["ds"]` — logical date as string
- `kwargs["ti"]` — the task instance (for XCom operations)
- `kwargs["dag_run"]` — the DAG run object
- `kwargs["params"]` — user-defined parameters

The return value of the `python_callable` is automatically stored as an XCom with the key `return_value`.

### Passing Arguments

You can pass extra arguments to your function:

```python
def greet(name, greeting="Hello"):
    print(f"{greeting}, {name}!")

greet_task = PythonOperator(
    task_id="greet",
    python_callable=greet,
    op_kwargs={"name": "Alice", "greeting": "Good morning"},
)
```

## 3.5 The TaskFlow API (Modern Alternative)

Airflow 2.0+ introduced the **TaskFlow API**, which uses Python decorators to make DAGs more concise. Instead of creating PythonOperator instances explicitly, you decorate functions with `@task`:

```python
from airflow.sdk import dag, task
from datetime import datetime

@dag(
    dag_id="taskflow_example",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
)
def my_pipeline():

    @task
    def extract():
        return {"sales": 100, "returns": 5}

    @task
    def transform(raw_data):
        return {"net_sales": raw_data["sales"] - raw_data["returns"]}

    @task
    def load(clean_data):
        print(f"Loading net_sales={clean_data['net_sales']} into database")

    # Dependencies are inferred from function arguments!
    raw = extract()
    clean = transform(raw)
    load(clean)

my_pipeline()  # Don't forget to call the DAG function!
```

The big advantage: XCom passing is **automatic**. When `transform(raw)` takes the output of `extract()` as its argument, Airflow automatically handles the XCom push and pull. No `xcom_pull()` calls needed.

> **⚠️ Gotcha:** When using the `@dag` decorator, you must call the decorated function at the bottom of the file (the `my_pipeline()` line). If you forget this, Airflow won't detect the DAG. This is different from the `with DAG(...)` context manager approach, which doesn't need a function call.

## 3.6 Mixing Operators: A Complete Multi-Task DAG

Here's a complete DAG that combines BashOperator and PythonOperator:

```python
from airflow.sdk import DAG
from airflow.providers.standard.operators.bash import BashOperator
from airflow.providers.standard.operators.python import PythonOperator
from datetime import datetime, timedelta
import json
import os

default_args = {
    "owner": "tutorial",
    "retries": 2,
    "retry_delay": timedelta(minutes=1),
}

def generate_data(**kwargs):
    """Simulate generating a data file."""
    logical_date = kwargs["ds"]
    filepath = f"/tmp/airflow_data/sales_{logical_date}.json"
    data = {
        "date": logical_date,
        "items_sold": 142,
        "revenue": 8520.50,
        "returns": 7,
    }
    os.makedirs("/tmp/airflow_data", exist_ok=True)
    with open(filepath, "w") as f:
        json.dump(data, f)
    print(f"Generated data at {filepath}")
    return filepath  # Auto-pushed as XCom

def process_data(**kwargs):
    """Read the data file, compute net revenue, and save a summary."""
    ti = kwargs["ti"]
    filepath = ti.xcom_pull(task_ids="generate_data")
    print(f"Reading from {filepath}")

    with open(filepath) as f:
        data = json.load(f)

    summary = {
        "date": data["date"],
        "net_revenue": data["revenue"] - (data["returns"] * 15),
        "items_sold": data["items_sold"],
    }

    summary_path = filepath.replace("sales_", "summary_")
    with open(summary_path, "w") as f:
        json.dump(summary, f)

    print(f"Summary: {summary}")
    return summary_path

def report_results(**kwargs):
    """Print a final report."""
    ti = kwargs["ti"]
    summary_path = ti.xcom_pull(task_ids="process_data")

    with open(summary_path) as f:
        summary = json.load(f)

    print("=" * 40)
    print(f"  Daily Report: {summary['date']}")
    print(f"  Items Sold: {summary['items_sold']}")
    print(f"  Net Revenue: ${summary['net_revenue']:.2f}")
    print("=" * 40)

with DAG(
    dag_id="daily_sales_pipeline",
    default_args=default_args,
    description="A multi-task DAG demonstrating Bash and Python operators",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
    tags=["tutorial"],
) as dag:

    # Task 1: Create necessary directories (BashOperator)
    setup = BashOperator(
        task_id="setup_directories",
        bash_command="mkdir -p /tmp/airflow_data && echo 'Directories ready'",
    )

    # Task 2: Generate data (PythonOperator)
    generate = PythonOperator(
        task_id="generate_data",
        python_callable=generate_data,
    )

    # Task 3: Process data (PythonOperator)
    process = PythonOperator(
        task_id="process_data",
        python_callable=process_data,
    )

    # Task 4: Print report (PythonOperator)
    report = PythonOperator(
        task_id="report_results",
        python_callable=report_results,
    )

    # Task 5: Cleanup (BashOperator)
    cleanup = BashOperator(
        task_id="cleanup",
        bash_command="echo 'Pipeline complete for {{ ds }}'",
    )

    # Dependencies
    setup >> generate >> process >> report >> cleanup
```

## 3.7 Hands-On Exercise: Build and Run Your First DAG

**Goal:** Deploy the multi-task DAG from Section 3.6, run it, and trace the data flow through XComs.

### Steps

1. **Save the DAG file** from Section 3.6 as `dags/daily_sales_pipeline.py` in your Airflow project.

2. **Wait 15–30 seconds** for the scheduler to detect the file. Refresh the Airflow UI — you should see "daily_sales_pipeline" in the DAG list.

3. **Unpause the DAG** by clicking its toggle. Since `catchup=False`, it will either create one immediate run or wait for the next scheduled interval.

4. **Manually trigger a run** by clicking the "Play" button (▶) on the DAG page and confirming. This is faster than waiting for the schedule.

5. **Watch the run execute** in the Grid view. You should see tasks turning green (success) in order: setup → generate → process → report → cleanup.

6. **Read the logs:**
   - Click on the `generate_data` task instance → Log tab. You should see "Generated data at /tmp/airflow_data/sales_YYYY-MM-DD.json".
   - Click on the `report_results` task instance → Log tab. You should see the daily report printout.

7. **Check XComs:**
   - Click on the `generate_data` task instance → XCom tab. You'll see a `return_value` entry containing the filepath.
   - Click on the `process_data` task instance → XCom tab. You'll see its `return_value` containing the summary filepath.

8. **Make a change:** Edit the DAG to add a `revenue_tax` calculation in `process_data`. Save the file, wait for the scheduler to re-parse, and trigger a new run.

9. **Bonus — Add parallelism:** Modify the DAG so that after `generate`, two tasks run in parallel: `process_data` and a new `count_records` task. Both should complete before `report_results`. The dependency should look like:
   ```
   setup >> generate >> [process, count] >> report >> cleanup
   ```

---

### Hour 3 — Interview Questions

**Q1 (Practical): Walk through the steps to deploy a new DAG to a local Airflow environment.**

Write a Python file defining the DAG and save it in the `dags/` folder. Wait for the scheduler to scan the folder and parse the file (usually 15–30 seconds). Check the Airflow UI — the DAG should appear in the list, paused by default. Toggle the DAG to unpause it. If you want to run it immediately rather than waiting for the schedule, click the trigger button. Check the Grid view and task logs to verify it ran correctly.

**Q2 (Conceptual): What's the difference between the `with DAG(...)` context manager pattern and the `@dag` decorator pattern?**

Both create DAGs, but they differ in syntax and XCom handling. The `with DAG(...)` pattern uses a context manager — any operator instantiated inside the `with` block is automatically associated with that DAG. You define dependencies explicitly with `>>`. The `@dag` decorator pattern uses function decorators: you decorate a function with `@dag` and inner functions with `@task`. The key advantage of the decorator pattern is that XCom passing is implicit — when one task function takes another's return value as an argument, Airflow handles the push/pull automatically. With the context manager pattern, you must use `xcom_pull()` explicitly. One gotcha: with `@dag`, you must call the decorated function at the bottom of the file.

**Q3 (Gotcha): You put a `requests.get()` call at the top level of your DAG file to fetch a configuration value. The Airflow scheduler becomes slow. Why?**

The scheduler imports every DAG file periodically (every 30 seconds by default) to check for changes. Code at the top level of the file — outside of operator functions — runs during every parse. An HTTP request that takes even 1 second will block the scheduler's parsing loop, and across many DAG files or many parse cycles, this adds up dramatically. The fix is to move the `requests.get()` call inside a PythonOperator's callable function, so it only runs when the task actually executes, not during parsing.

**Q4 (Practical): How would you pass data from a PythonOperator in one task to a BashOperator in a downstream task?**

The PythonOperator should return the value (which auto-pushes it as an XCom). The BashOperator can access it using Jinja templating: `bash_command="echo 'The file is at {{ ti.xcom_pull(task_ids='upstream_task') }}'"`. The `{{ }}` syntax tells Airflow to render the template before executing the bash command. The `ti.xcom_pull()` call retrieves the XCom value pushed by the specified upstream task. This works because BashOperator's `bash_command` field supports Jinja templates.

**Q5 (Gotcha): Your DAG file contains no syntax errors, but it doesn't appear in the Airflow UI. What could be wrong?**

Several possible causes: (1) The file isn't in the `dags/` folder, or there's a subdirectory issue. (2) The file has an import error for a package that isn't installed in the Airflow environment — this is a silent failure that prevents the DAG from loading. Check the scheduler logs for import errors. (3) The file doesn't actually create a DAG object at the module level — if using the `@dag` decorator, you forgot to call the decorated function. (4) The file name starts with a period or underscore, which some Airflow configurations ignore. (5) The `dag_id` conflicts with another existing DAG.

---

# Hour 4 — Sensors, Hooks, and XCom in Practice

## 4.1 The Problem: "My Pipeline Needs to Wait for Something External"

Not every task is "do something right now." Sometimes you need your pipeline to **wait** for a condition before proceeding:

- Wait for a file to appear in a directory (a partner uploads it overnight).
- Wait for a row to appear in a database table (another team's pipeline must finish first).
- Wait for an S3 object to exist (an upstream system writes it at an unpredictable time).

You *could* write a PythonOperator with a while loop that checks every 30 seconds, but that task would sit there consuming a worker slot the entire time. Airflow has a better abstraction for this: **sensors**.

## 4.2 What Is a Sensor?

A **sensor** is a special type of operator that does exactly one thing: wait for a condition to become true, then succeed. If the condition doesn't become true within a timeout period, the sensor fails.

Every sensor works through a method called `poke()`. Airflow calls `poke()` repeatedly at a configurable interval. Each call checks the condition and returns `True` (condition met, sensor succeeds) or `False` (not yet, check again later).

### Sensor Parameters

All sensors share these important parameters:

| Parameter | Default | What It Controls |
|-----------|---------|-----------------|
| `poke_interval` | 60 (seconds) | How often to check the condition |
| `timeout` | 604800 (7 days) | How long to wait before giving up and failing |
| `mode` | `"poke"` | How the sensor waits between checks (see below) |
| `soft_fail` | `False` | If `True`, timeout causes a "skip" instead of "fail" |

### Sensor Modes: `poke` vs `reschedule`

This is a critical concept:

- **`mode="poke"`** (default): The sensor occupies a worker slot for its entire duration. Between checks, it sleeps but still holds the slot. Simple, but wasteful — if you have many sensors poking every 60 seconds for hours, they're holding worker slots that could be running real tasks.

- **`mode="reschedule"`**: The sensor runs the check, and if the condition isn't met, it **releases the worker slot** and tells the scheduler "wake me up in `poke_interval` seconds." This is much more efficient for long waits.

**Rule of thumb:** If your sensor might wait more than a few minutes, use `mode="reschedule"`.

> **⚠️ Gotcha:** In `poke` mode, a sensor with `timeout=86400` (24 hours) and `poke_interval=60` will hold a worker slot for up to 24 hours. If you have a limited number of worker slots (e.g., 16 with LocalExecutor), just a few long-running sensors in poke mode can block your entire Airflow instance. Always use `mode="reschedule"` for sensors that might wait a long time.

## 4.3 FileSensor: Waiting for a Local File

The **FileSensor** waits for a file or directory to appear on the local filesystem.

```python
from airflow.providers.standard.sensors.filesystem import FileSensor

wait_for_file = FileSensor(
    task_id="wait_for_sales_file",
    filepath="/tmp/incoming/sales_{{ ds }}.csv",  # Jinja template!
    fs_conn_id="fs_default",                       # Connection with base path
    poke_interval=30,                              # Check every 30 seconds
    timeout=3600,                                  # Give up after 1 hour
    mode="reschedule",                             # Don't hog a worker slot
)
```

The `fs_conn_id` references an Airflow Connection of type "File (path)". The Connection's `extra` field can specify a base path, and the `filepath` parameter is relative to it. If you set up `fs_default` with `{"path": "/"}`, then `filepath="/tmp/incoming/..."` is absolute.

> **⚠️ Gotcha:** The FileSensor checks the filesystem of the **Airflow worker**, not your local machine. If Airflow is running in Docker, the worker's filesystem is inside the container. A file at `/tmp/incoming/` on your laptop won't be visible unless that directory is mounted into the container.

## 4.4 S3KeySensor: Waiting for an S3 Object

The **S3KeySensor** waits for a file (called a "key" in S3 terminology) to appear in an Amazon S3 bucket.

```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_for_s3 = S3KeySensor(
    task_id="wait_for_s3_file",
    bucket_name="my-data-bucket",
    bucket_key="incoming/sales_{{ ds }}.csv",
    aws_conn_id="aws_default",          # Airflow Connection with AWS credentials
    poke_interval=120,                   # Check every 2 minutes
    timeout=7200,                        # Give up after 2 hours
    mode="reschedule",
)
```

You can also use the full S3 URL format:

```python
S3KeySensor(
    task_id="wait_for_s3_file",
    bucket_key="s3://my-data-bucket/incoming/sales_{{ ds }}.csv",
    aws_conn_id="aws_default",
)
```

The `aws_conn_id` references an Airflow Connection that stores your AWS access key and secret key.

> **⚠️ Gotcha:** The S3KeySensor requires the `apache-airflow-providers-amazon` package. If you get an import error, add `apache-airflow-providers-amazon` to your `requirements.txt` and rebuild your environment.

## 4.5 Other Useful Sensors

| Sensor | What It Waits For |
|--------|-------------------|
| `HttpSensor` | An HTTP endpoint to return a success status |
| `SqlSensor` | A SQL query to return a truthy result |
| `ExternalTaskSensor` | A task in another DAG to complete |
| `DateTimeSensor` | A specific date/time to arrive |
| `PythonSensor` | A custom Python callable to return `True` |
| `BashSensor` | A bash command to return exit code 0 |

The `PythonSensor` is especially versatile — you can check *anything*:

```python
from airflow.providers.standard.sensors.python import PythonSensor

def check_api_ready():
    import requests
    try:
        response = requests.get("https://api.example.com/health")
        return response.status_code == 200
    except Exception:
        return False

wait_for_api = PythonSensor(
    task_id="wait_for_api",
    python_callable=check_api_ready,
    poke_interval=60,
    timeout=1800,
    mode="reschedule",
)
```

## 4.6 What Is a Hook?

A **hook** is a reusable interface to an external system — a database, an API, a cloud service. Hooks handle the connection details (authentication, session management, retries) so your task code doesn't have to.

Operators and sensors *use* hooks internally. For example, the S3KeySensor uses the `S3Hook` to connect to AWS. When you use a PythonOperator and need to interact with S3, you use the `S3Hook` directly:

```python
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

def download_from_s3(**kwargs):
    hook = S3Hook(aws_conn_id="aws_default")

    # Download a file
    local_path = hook.download_file(
        key="incoming/sales_2024-01-15.csv",
        bucket_name="my-data-bucket",
        local_path="/tmp/airflow_data/",
    )
    print(f"Downloaded to {local_path}")
    return local_path
```

The hook reads credentials from the Airflow Connection named `aws_default` — you never need to handle access keys in your code.

### Common Built-In Hooks

| Hook | What It Connects To |
|------|-------------------|
| `S3Hook` | Amazon S3 (upload, download, list, delete files) |
| `PostgresHook` | PostgreSQL databases (run queries, fetch results) |
| `HttpHook` | Any HTTP API (GET, POST, etc.) |
| `SlackHook` | Slack (send messages) |
| `FTPHook` / `SFTPHook` | FTP/SFTP servers |
| `MySqlHook` | MySQL databases |

### Writing a Custom Hook

If Airflow doesn't have a hook for your system, you can write one by extending `BaseHook`:

```python
from airflow.sdk import BaseHook
import requests

class MyApiHook(BaseHook):
    """Hook for the My Company internal API."""

    conn_name_attr = "my_api_conn_id"
    default_conn_name = "my_api_default"

    def __init__(self, my_api_conn_id="my_api_default"):
        super().__init__()
        self.my_api_conn_id = my_api_conn_id

    def get_conn(self):
        """Get connection details from Airflow Connection."""
        conn = self.get_connection(self.my_api_conn_id)
        return {
            "base_url": conn.host,
            "api_key": conn.password,
        }

    def fetch_data(self, endpoint):
        """Fetch data from the API."""
        config = self.get_conn()
        response = requests.get(
            f"{config['base_url']}/{endpoint}",
            headers={"Authorization": f"Bearer {config['api_key']}"},
        )
        response.raise_for_status()
        return response.json()
```

## 4.7 XCom in Practice: Passing Data Between Tasks

Let's see XCom used properly in a realistic pipeline:

```python
from airflow.sdk import dag, task
from datetime import datetime

@dag(
    dag_id="xcom_demo",
    start_date=datetime(2024, 1, 1),
    schedule=None,          # Manual trigger only
    catchup=False,
    tags=["tutorial"],
)
def xcom_demo():

    @task
    def extract():
        """Simulate extracting records from a source."""
        records = [
            {"id": 1, "name": "Widget A", "price": 9.99},
            {"id": 2, "name": "Widget B", "price": 14.99},
            {"id": 3, "name": "Widget C", "price": 4.99},
        ]
        print(f"Extracted {len(records)} records")
        return records  # Auto-pushed as XCom

    @task
    def transform(records):
        """Apply a discount and filter cheap items."""
        DISCOUNT = 0.1
        transformed = []
        for r in records:
            r["discounted_price"] = round(r["price"] * (1 - DISCOUNT), 2)
            if r["discounted_price"] >= 5.00:
                transformed.append(r)
        print(f"Kept {len(transformed)} of {len(records)} records after filtering")
        return transformed

    @task
    def load(records):
        """Simulate loading to a database."""
        for r in records:
            print(f"  Loaded: {r['name']} @ ${r['discounted_price']}")
        return len(records)

    @task
    def notify(record_count):
        """Send a notification with the summary."""
        print(f"Pipeline complete! Loaded {record_count} records.")

    # Chain: dependencies are inferred from function arguments
    raw = extract()
    clean = transform(raw)
    count = load(clean)
    notify(count)

xcom_demo()
```

With the TaskFlow API, the XCom push/pull is completely hidden — you just pass values between functions like normal Python.

### XCom with Traditional Operators

If you're using the classic `with DAG(...)` pattern:

```python
# Push: return a value from PythonOperator, or explicitly:
def push_example(**kwargs):
    kwargs["ti"].xcom_push(key="my_custom_key", value="my_value")

# Pull: in another task's callable:
def pull_example(**kwargs):
    value = kwargs["ti"].xcom_pull(task_ids="push_task", key="my_custom_key")
    # Or for the default return_value key:
    value = kwargs["ti"].xcom_pull(task_ids="push_task")

# Pull in a BashOperator using Jinja:
bash_task = BashOperator(
    task_id="use_xcom",
    bash_command="echo 'Value is: {{ ti.xcom_pull(task_ids=\"push_task\") }}'",
)
```

> **⚠️ Gotcha:** XCom values must be serializable (convertible to JSON by default). If you try to push a non-serializable object (like a database connection or a file handle), it will fail. Stick to basic types: strings, numbers, lists, dictionaries.

## 4.8 Hands-On Exercise: Build a Sensor + Hook + XCom Pipeline

**Goal:** Build a DAG that waits for a file to appear, reads it, processes it, and reports results — all with proper XCom communication.

### Step 1 — Create the DAG File

Save this as `dags/sensor_pipeline.py`:

```python
from airflow.sdk import DAG
from airflow.providers.standard.operators.bash import BashOperator
from airflow.providers.standard.operators.python import PythonOperator
from airflow.providers.standard.sensors.filesystem import FileSensor
from datetime import datetime, timedelta
import json
import os

default_args = {
    "owner": "tutorial",
    "retries": 1,
    "retry_delay": timedelta(minutes=1),
}

def process_file(**kwargs):
    """Read the file, compute stats, and return a summary."""
    ti = kwargs["ti"]
    logical_date = kwargs["ds"]
    filepath = f"/tmp/incoming/orders_{logical_date}.json"

    with open(filepath) as f:
        orders = json.load(f)

    total_revenue = sum(o["amount"] for o in orders)
    avg_order = total_revenue / len(orders) if orders else 0

    summary = {
        "date": logical_date,
        "order_count": len(orders),
        "total_revenue": round(total_revenue, 2),
        "avg_order_value": round(avg_order, 2),
    }
    print(f"Processed: {summary}")
    return summary

def generate_report(**kwargs):
    """Generate a text report from the summary."""
    ti = kwargs["ti"]
    summary = ti.xcom_pull(task_ids="process_file")

    report = (
        f"=== Daily Orders Report ({summary['date']}) ===\n"
        f"Orders: {summary['order_count']}\n"
        f"Revenue: ${summary['total_revenue']}\n"
        f"Avg Order: ${summary['avg_order_value']}\n"
    )
    print(report)

    report_path = f"/tmp/reports/report_{summary['date']}.txt"
    os.makedirs("/tmp/reports", exist_ok=True)
    with open(report_path, "w") as f:
        f.write(report)

    return report_path

with DAG(
    dag_id="sensor_pipeline",
    default_args=default_args,
    start_date=datetime(2024, 1, 1),
    schedule=None,          # Manual trigger only
    catchup=False,
    tags=["tutorial", "sensors"],
) as dag:

    # Task 1: Ensure directories exist
    setup = BashOperator(
        task_id="setup_dirs",
        bash_command="mkdir -p /tmp/incoming /tmp/reports",
    )

    # Task 2: Wait for the orders file to arrive
    wait_for_file = FileSensor(
        task_id="wait_for_orders",
        filepath="/tmp/incoming/orders_{{ ds }}.json",
        fs_conn_id="fs_default",
        poke_interval=10,           # Check every 10 seconds (fast for demo)
        timeout=300,                # Give up after 5 minutes
        mode="poke",                # OK for short waits in demo
    )

    # Task 3: Process the file
    process = PythonOperator(
        task_id="process_file",
        python_callable=process_file,
    )

    # Task 4: Generate report
    report = PythonOperator(
        task_id="generate_report",
        python_callable=generate_report,
    )

    # Task 5: Log completion
    done = BashOperator(
        task_id="log_completion",
        bash_command=(
            "echo 'Report saved to: "
            "{{ ti.xcom_pull(task_ids=\"generate_report\") }}'"
        ),
    )

    # Dependencies
    setup >> wait_for_file >> process >> report >> done
```

### Step 2 — Make Sure the Connection Exists

If you didn't create the `fs_default` connection in Hour 2's exercise, do it now:

- Go to Admin → Connections → "+".
- Connection Id: `fs_default`
- Connection Type: `File (path)`
- Extra: `{"path": "/"}`

### Step 3 — Trigger the DAG

1. Unpause and manually trigger the `sensor_pipeline` DAG.
2. In the Grid view, you'll see `setup_dirs` succeed, then `wait_for_orders` will start "sensing" (its state will show as "running" or "up_for_reschedule").

### Step 4 — Create the File (Simulating the External Event)

The sensor is waiting! Now simulate the file arriving. Open a terminal:

**If using Astro CLI:**
```bash
astro dev bash
# Now inside the container:
cat > /tmp/incoming/orders_$(date -u +%Y-%m-%d).json << 'EOF'
[
    {"order_id": 1, "product": "Laptop", "amount": 999.99},
    {"order_id": 2, "product": "Mouse", "amount": 29.99},
    {"order_id": 3, "product": "Keyboard", "amount": 79.99},
    {"order_id": 4, "product": "Monitor", "amount": 449.99}
]
EOF
exit
```

**If using Docker Compose:**
```bash
docker exec -it <scheduler_container_name> bash
# Then the same cat command as above
```

> **Note:** The `$(date -u +%Y-%m-%d)` generates today's date in UTC. If your DAG run's logical date differs (e.g., you triggered for a specific date), adjust accordingly.

### Step 5 — Watch It Complete

Within 10 seconds (the `poke_interval`), the sensor should detect the file and succeed. The remaining tasks will execute in sequence.

### Step 6 — Verify the Results

- Check the logs for `process_file` — you should see the summary stats.
- Check the logs for `generate_report` — you should see the formatted report.
- Check the XCom tab for `generate_report` — it should show the report file path.
- Check the logs for `log_completion` — it should echo the report path pulled from XCom.

---

### Hour 4 — Interview Questions

**Q1 (Conceptual): What is a sensor in Airflow, and how does it differ from a regular operator?**

A sensor is a special type of operator that waits for an external condition to become true before succeeding. While a regular operator executes work immediately (run a script, send an email, load data), a sensor's job is to repeatedly check a condition (via its `poke()` method) at a configurable interval and succeed only when the condition is met. If the condition isn't met within the timeout, the sensor fails. Sensors are useful for making pipelines event-driven — waiting for files to arrive, APIs to become available, or upstream systems to complete their work.

**Q2 (Gotcha): You have 20 sensors in `poke` mode, each waiting up to 6 hours with `poke_interval=60`. Your Airflow instance has 16 worker slots. Other DAGs stop running. What happened?**

In `poke` mode, each sensor occupies a worker slot for its entire duration, even while sleeping between checks. With 20 sensors, you need 20 slots — but you only have 16. The first 16 sensors claim all available slots, and the remaining 4 sensors (plus all other tasks from all other DAGs) are stuck in the queue with no slots available. This is called "sensor deadlock." The fix is to switch the sensors to `mode="reschedule"`, which releases the worker slot between checks. In reschedule mode, the sensor only occupies a slot for the brief moment it runs the actual check.

**Q3 (Practical): What is a hook, and why would you use one instead of writing connection logic directly in your PythonOperator?**

A hook is a reusable class that manages connections to external systems (databases, APIs, cloud services). You'd use one instead of inline connection logic for several reasons: (1) Credentials are managed through Airflow Connections, not hardcoded in DAG files. (2) Hooks handle connection pooling, retries, and session management. (3) They're reusable across multiple tasks and DAGs. (4) They provide a tested, community-maintained interface — you don't have to write your own S3 upload logic. For example, `S3Hook` handles AWS authentication, retries, and all the boto3 details, and you just call `hook.download_file()`.

**Q4 (Practical): A task returns a dictionary from its `python_callable`. How does the downstream task access that data? Show both the TaskFlow and classic approaches.**

TaskFlow approach: The downstream `@task` function simply takes the upstream's return value as a parameter. Airflow handles XCom push/pull automatically:
```python
@task
def upstream():
    return {"key": "value"}

@task
def downstream(data):
    print(data["key"])

downstream(upstream())
```

Classic approach: The upstream PythonOperator's return value is auto-pushed to XCom with key `return_value`. The downstream task pulls it:
```python
def downstream_fn(**kwargs):
    data = kwargs["ti"].xcom_pull(task_ids="upstream_task")
    print(data["key"])
```

**Q5 (Gotcha): Your FileSensor is configured with `filepath="/data/sales.csv"` and works in testing, but in production (Docker), it never finds the file even though the file exists on the host machine. Why?**

The FileSensor runs inside the Airflow worker container, not on the host machine. The container has its own filesystem, and `/data/sales.csv` on the host is not visible inside the container unless that directory is explicitly mounted as a Docker volume. The fix is to add a volume mount in your Docker Compose configuration (or `docker-compose.override.yml`) mapping the host directory to a container path, for example: `- /data:/data`. Then the file will be visible at the same path inside the container.

---

# Appendix A — Gotchas Summary Table

| # | Area | Gotcha | Consequence | Fix |
|---|------|--------|-------------|-----|
| 1 | DAG scheduling | Logical date ≠ actual run time | DAG for "Jan 1" runs on Jan 2 | Understand that Airflow runs at the *end* of each interval |
| 2 | DAG scheduling | `catchup=True` with an old `start_date` (was the Airflow 2 default; Airflow 3 defaults to `False`) | Hundreds of backfill runs queued at once | Set `catchup=False` explicitly unless you want backfilling |
| 3 | DAG scheduling | Times are in UTC by default | DAG runs at unexpected local times | Set timezone explicitly on the DAG, or adjust your cron expressions |
| 4 | XCom | Pushing large objects through XCom | Metadata database bloats; UI slows down | Push only small values (paths, counts); store bulk data externally |
| 5 | XCom | Non-serializable objects pushed | Task fails with serialization error | Only push JSON-serializable types (str, int, list, dict) |
| 6 | DAG file | Expensive code at module level | Scheduler slows down (re-runs every parse) | Move API calls, file reads, etc. inside operator callables |
| 7 | DAG file | `@dag`-decorated function not called | DAG doesn't appear in UI | Add `my_dag_function()` at bottom of file |
| 8 | Sensors | `mode="poke"` on long-wait sensors | Worker slots exhausted; deadlock | Use `mode="reschedule"` for waits over a few minutes |
| 9 | Sensors | FileSensor path is on host, not container | Sensor never finds the file in Docker | Mount the host directory as a Docker volume |
| 10 | Dependencies | Installing new pip packages | `ModuleNotFoundError` after restart | Use `astro dev restart` (not stop/start) to rebuild the image |
| 11 | Connections | Hardcoded credentials in DAG files | Security risk; credentials in version control | Use Airflow Connections or a secrets backend |
| 12 | BashOperator | Trailing whitespace in multi-line commands | Unexpected failures on some systems | Trim trailing spaces in bash_command strings |
| 13 | Operators | Non-zero exit code from BashOperator | Task marked as failed even if "mostly worked" | Ensure scripts return 0 on success; use `set -e` in scripts |
| 14 | Resources | Docker not allocated enough RAM | Containers restart; API server returns 503 | Allocate ≥4 GB RAM to Docker Desktop |

---

# Appendix B — Quick Reference Card

## Airflow CLI Commands

| What You Want | Command |
|---------------|---------|
| List all DAGs | `airflow dags list` |
| Trigger a DAG run | `airflow dags trigger <dag_id>` |
| Pause a DAG | `airflow dags pause <dag_id>` |
| Unpause a DAG | `airflow dags unpause <dag_id>` |
| List tasks in a DAG | `airflow tasks list <dag_id>` |
| Test a single task | `airflow tasks test <dag_id> <task_id> <date>` |
| Check DAG for errors | `airflow dags test <dag_id>` |
| View Airflow config | `airflow config list` |
| Create a user | `airflow users create --username admin --password admin --role Admin --email admin@example.com --firstname Admin --lastname User` |

> **⚠️ Airflow 3 change:** User and login management moved to a pluggable **auth manager**. The `airflow users ...` commands now belong to the **FAB (Flask-AppBuilder) auth manager**, which must be installed/enabled to use them; Airflow 3's lightweight default is the **SimpleAuthManager**. With the Astro CLI the default `admin`/`admin` login is preconfigured, so you usually don't need this command locally.

## Astro CLI Commands

| What You Want | Command |
|---------------|---------|
| Create a project | `astro dev init` |
| Start Airflow | `astro dev start` |
| Stop Airflow (keep data) | `astro dev stop` |
| Restart (rebuild image) | `astro dev restart` |
| Full reset (delete data) | `astro dev kill` |
| Open scheduler shell | `astro dev bash` |
| Run Airflow CLI command | `astro dev run <airflow_command>` |
| View running containers | `astro dev ps` |

## Common Operators

| Operator | Import | Purpose |
|----------|--------|---------|
| `BashOperator` | `from airflow.providers.standard.operators.bash import BashOperator` | Run bash commands |
| `PythonOperator` | `from airflow.providers.standard.operators.python import PythonOperator` | Run Python functions |
| `EmailOperator` | `from airflow.providers.smtp.operators.smtp import EmailOperator` | Send emails |
| `EmptyOperator` | `from airflow.providers.standard.operators.empty import EmptyOperator` | No-op placeholder (for branching/joining) |

## Common Sensors

| Sensor | Import | Waits For |
|--------|--------|-----------|
| `FileSensor` | `from airflow.providers.standard.sensors.filesystem import FileSensor` | File on filesystem |
| `S3KeySensor` | `from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor` | Object in S3 |
| `HttpSensor` | `from airflow.providers.http.sensors.http import HttpSensor` | HTTP endpoint success |
| `SqlSensor` | `from airflow.providers.common.sql.sensors.sql import SqlSensor` | SQL query returns truthy |
| `ExternalTaskSensor` | `from airflow.providers.standard.sensors.external_task import ExternalTaskSensor` | Task in another DAG |
| `PythonSensor` | `from airflow.providers.standard.sensors.python import PythonSensor` | Custom Python check |

## Common Hooks

| Hook | Import | Connects To |
|------|--------|-------------|
| `S3Hook` | `from airflow.providers.amazon.aws.hooks.s3 import S3Hook` | Amazon S3 |
| `PostgresHook` | `from airflow.providers.postgres.hooks.postgres import PostgresHook` | PostgreSQL |
| `HttpHook` | `from airflow.providers.http.hooks.http import HttpHook` | HTTP APIs |
| `MySqlHook` | `from airflow.providers.mysql.hooks.mysql import MySqlHook` | MySQL |
| `SlackHook` | `from airflow.providers.slack.hooks.slack import SlackHook` | Slack |

## Jinja Template Variables

| Variable | Example Value | Description |
|----------|-------------|-------------|
| `{{ ds }}` | `2024-01-15` | Logical date (YYYY-MM-DD) |
| `{{ ds_nodash }}` | `20240115` | Logical date without dashes |
| `{{ ts }}` | `2024-01-15T00:00:00+00:00` | Full ISO timestamp |
| `{{ data_interval_start }}` | datetime | Start of data interval |
| `{{ data_interval_end }}` | datetime | End of data interval |
| `{{ dag_run.logical_date }}` | datetime | Logical date as datetime object |
| `{{ params.my_param }}` | any | User-defined parameters |
| `{{ var.value.my_var }}` | string | Airflow Variable value |
| `{{ conn.my_conn.host }}` | string | Connection attribute |
| `{{ ti.xcom_pull(task_ids='...') }}` | any | XCom value from another task |

## DAG Definition Cheat Sheet

```python
# Minimal DAG — copy and customize
from airflow.sdk import DAG
from airflow.providers.standard.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    "owner": "your-name",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

def my_task_function(**kwargs):
    print(f"Running for {kwargs['ds']}")
    return "done"

with DAG(
    dag_id="my_dag",
    default_args=default_args,
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
    tags=["my-team"],
) as dag:

    task_a = PythonOperator(
        task_id="task_a",
        python_callable=my_task_function,
    )
```

## Dependency Syntax

```python
# Linear
a >> b >> c

# Fan-out
a >> [b, c]

# Fan-in
[b, c] >> d

# Full diamond
a >> [b, c] >> d

# Multiple chains on separate lines
a >> b
a >> c
[b, c] >> d
```

---

*Tutorial complete. Work through the exercises at your own pace, and refer back to the gotchas table and quick reference card as you build real DAGs.*
