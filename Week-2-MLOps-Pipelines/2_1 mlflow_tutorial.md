# MLflow: A Complete Beginner's Tutorial
### From Zero to Confident — 4 Hours, Hands-On

---

> **How to use this guide**
> Work through each hour in order. Don't skip the exercises — they cement the concepts. The interview questions at the end of each hour are real-world prep. The reference card and gotchas table at the very end are yours to keep.

---

## Prerequisites

Before you start, make sure you have:

- Python 3.8 or higher installed (`python --version` to check)
- `pip` available (`pip --version` to check)
- A terminal / command prompt you're comfortable opening

That's it. No prior ML or MLflow knowledge assumed.

---

# Hour 1 — MLflow Concepts: What Is It and Why Does It Exist?

## 1.1 The Problem MLflow Solves

Imagine you're baking bread. You experiment with different amounts of flour, yeast, and water. Some loaves come out great, some terrible. If you don't write anything down, you quickly lose track of which combination worked best. A week later, you can't reproduce your best loaf.

Machine learning has exactly this problem — but at scale. A data scientist might train dozens of versions of the same model, each with slightly different settings, on slightly different data, producing different results. Without a system to record all of this:

- You can't reproduce a result that worked
- You can't compare this week's model to last week's
- You can't hand a model off to a colleague reliably
- Deploying a model to production becomes a manual, error-prone process

**MLflow** is an open-source platform that solves this. It gives you a structured, systematic way to:

1. **Track** experiments — record what you tried and what happened
2. **Package** your code — make experiments reproducible anywhere
3. **Manage models** — standardise how models are saved and loaded
4. **Register models** — maintain a catalogue of models ready for production

> **Term: open-source** — Free to use, and the code itself is publicly available. Anyone can read, use, or contribute to it.

## 1.2 Installing MLflow

Open your terminal and run:

```bash
pip install mlflow
```

Verify the installation:

```bash
mlflow --version
```

You should see something like `mlflow, version 2.x.x`. If you do, you're ready.

---

## 1.3 The Four Components of MLflow

MLflow is made up of four distinct components. Think of them as four departments in a lab:

```
┌─────────────────────────────────────────────────────────────────┐
│                          MLflow                                 │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  ┌────────┐  │
│  │   Tracking   │  │   Projects   │  │  Models  │  │Registry│  │
│  │              │  │              │  │          │  │        │  │
│  │ "What did    │  │ "How do I    │  │ "How do  │  │ "Which │  │
│  │ I try?"      │  │ run this     │  │ I share  │  │ model  │  │
│  │              │  │ elsewhere?"  │  │ a model?"│  │is live?"│  │
│  └──────────────┘  └──────────────┘  └──────────┘  └────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

Let's understand each one.

---

### Component 1: MLflow Tracking

**What problem does it solve?**
When you train a model, there are many things you want to remember: the settings you used (called **parameters**), how well the model performed (called **metrics**), and any files the training produced like charts or saved model files (called **artifacts**). MLflow Tracking is a logging system for all of these.

> **Term: parameter** — A setting you choose before training begins. For example, "train for 100 rounds" or "use a learning rate of 0.01". You control parameters.

> **Term: metric** — A measurement of how well your model performed. For example, "accuracy = 92%" or "error rate = 0.08". Metrics come out of training.

> **Term: artifact** — Any file produced during or after training that you want to keep. Examples: a saved model file, a chart image, a CSV of predictions.

> **Term: run** — One complete execution of your training script. Each time you train, that's one run. Each run gets its own record in Tracking.

> **Term: experiment** — A named group of related runs. For example, "price-prediction-v2" might be an experiment containing 20 different runs where you tried different settings.

MLflow Tracking stores all of this in a place called the **tracking server** (which by default is just a folder on your computer called `mlruns`).

---

### Component 2: MLflow Projects

**What problem does it solve?**
You train a model on your laptop. It works great. Then you try to run the same code on a cloud server — but it fails because a library is missing, or the Python version is different. MLflow Projects is a standard format for packaging your code so that anyone (or any machine) can run it reliably.

> **Term: dependency** — A library or package your code needs to work. If your code uses `pandas` and someone else doesn't have it installed, your code breaks on their machine.

An MLflow Project is a directory (folder) that contains:
- Your code
- A file called `MLproject` that describes how to run the code and what it needs

> **Note for this tutorial:** MLflow Projects is the least commonly used component day-to-day. We introduce it here for completeness but won't spend deep time on it. Hours 2–4 cover the components you'll use constantly.

---

### Component 3: MLflow Models

**What problem does it solve?**
Different ML libraries save models in different ways. A model from one library can't easily be loaded by code written for another library. This makes serving (using) models in production painful.

> **Term: serving** — Making a model available so that other software can send it data and get predictions back. Like setting up a restaurant: the model is the chef, and other systems are customers placing orders.

MLflow Models defines a standard format (called a **flavor**) for saving models so they can be loaded and served in a consistent way — regardless of which library was used to train them.

> **Term: flavor** — In MLflow, a flavor is a specific way to use a model. For example, a model might have a `python_function` flavor (use it in Python) and a `sklearn` flavor (use it with scikit-learn). Having multiple flavors means the model can be used in different contexts.

---

### Component 4: MLflow Model Registry

**What problem does it solve?**
As your team produces more and more models, you need a central place to track which model version is in testing, which is in production, and which has been retired. The Model Registry is that central catalogue.

> **Term: model version** — Every time you register a new model (or an update to an existing one), it gets a version number: 1, 2, 3, and so on.

> **Term: stage** — A label that tells you the lifecycle status of a model version. Common stages are: Staging (being tested), Production (live and in use), and Archived (no longer active).

> **Term: alias** — A human-readable nickname for a specific model version. Instead of saying "use version 7", you can say "use the model aliased 'champion'".

---

## 1.4 How the Components Relate

Here's the typical journey a model takes through MLflow:

```
1. You run experiments → MLflow Tracking records every run
2. The best run produces a model → saved in MLflow Models format
3. You register that model → it appears in the Model Registry
4. Team reviews it → it's promoted through stages to Production
5. Your serving system pulls the Production model → end users get predictions
```

Each component feeds into the next.

---

## 1.5 The MLflow UI

MLflow includes a built-in web interface that lets you browse experiments, compare runs, and view the registry — all in a browser. To launch it:

```bash
mlflow ui
```

Then open your browser to: `http://127.0.0.1:5000`

> **⚠️ Gotcha:** The MLflow UI must be running in a separate terminal while you work. It doesn't run in the background by default. Open a second terminal tab and leave `mlflow ui` running there.

---

## 🏋️ Hour 1 Exercise: Orientation

**Goal:** Get familiar with the MLflow environment before writing any training code.

**Steps:**

1. Install MLflow if you haven't yet: `pip install mlflow`

2. In a new terminal, start the UI: `mlflow ui`

3. Open `http://127.0.0.1:5000` in your browser. You'll see an empty interface — that's normal.

4. In your working directory, check that an `mlruns` folder was created (it appears after first use). It won't exist yet — note where it will appear.

5. Read the MLflow documentation overview page at: https://mlflow.org/docs/latest/index.html
   - Find the four components listed there. Can you match them to what you just learned?

6. In your own words (write them down), answer: "What is the difference between a parameter and a metric?"

---

## 📝 Hour 1 Interview Questions

---

**Q1 (Conceptual): What is MLflow and what core problem does it solve?**

**A:** MLflow is an open-source platform for managing the machine learning lifecycle. It solves the problem of reproducibility and organisation in ML experimentation: without it, it's difficult to track which model settings produced which results, share models reliably, or manage which model version is deployed in production. MLflow provides structured tools for tracking experiments, packaging code, standardising model formats, and cataloguing models through their lifecycle.

---

**Q2 (Conceptual): Explain the difference between a parameter, a metric, and an artifact in MLflow.**

**A:**
- A **parameter** is an input to your training process — a setting you choose before training, like `learning_rate=0.01` or `max_depth=5`.
- A **metric** is a measured output of training — a number that tells you how well the model performed, like `accuracy=0.92` or `loss=0.04`.
- An **artifact** is any file produced by a run that you want to preserve: a saved model, a confusion matrix image, a CSV of test predictions.

Parameters go *in*, metrics come *out*, artifacts are *files*.

---

**Q3 (Practical): When would you use the Model Registry vs. just MLflow Tracking?**

**A:** Use MLflow Tracking for every experiment run — it records the history of everything you tried. Use the Model Registry when a specific model version is ready to be considered for production use. The Registry adds lifecycle management (staging, promotion, aliasing) that Tracking doesn't provide. In a solo research project, you might only use Tracking. In a team production environment, you'd use both: Tracking to find the best model, Registry to manage its path to deployment.

---

**Q4 (Gotcha): A colleague says "I have an MLflow experiment with 50 runs." What does that mean exactly — are they saying they trained 50 models?**

**A:** Not necessarily 50 complete models, but 50 *training executions*. Each run is one execution of the training script with potentially different parameters. Some runs might have failed (logged as failed runs), some might have been quick tests, and some might be serious candidates. An experiment is just a container grouping related runs. The colleague has tried 50 different configurations within a theme — how many produced usable models depends on the run statuses.

---

**Q5 (Gotcha): If you delete the `mlruns` folder from your project directory, what happens to all your experiment data?**

**A:** It's gone — unless you had configured a remote tracking server. By default, MLflow stores all tracking data (runs, parameters, metrics, artifacts) in a local `mlruns` directory. There is no cloud backup unless you explicitly set one up. This is a critical reason why production teams configure a central remote tracking server (covered in Hour 2) rather than relying on the default local storage.

---

---

# Hour 2 — MLflow Tracking: Recording Your Experiments

## 2.1 Why You Need Tracking

Suppose you're training a model to predict house prices. You try many different settings — maybe you adjust how many training rounds to run, or how aggressively the model corrects its mistakes. Each combination gives a different accuracy.

Without tracking, after 20 experiments you're left staring at 20 scripts and your memory. With MLflow Tracking, every run is recorded with its settings and results, and you can browse, compare, and filter them in a UI.

## 2.2 Setting Up the Tracking Server

MLflow Tracking has two parts:

1. **The tracking client** — the code in your Python script that logs data
2. **The tracking server** — the place where logged data is stored

By default, the server is just a local folder. But MLflow also supports running a proper server that can be accessed by a whole team over a network.

### Option A: Local (Default — Good for Learning)

You don't need to do anything special. MLflow automatically creates an `mlruns` folder wherever you run your script.

### Option B: Local Server (Better for Seeing the UI in Action)

Start a local tracking server with:

```bash
mlflow server --host 127.0.0.1 --port 5000
```

Then, in your Python code, tell MLflow where to send data:

```python
import mlflow
mlflow.set_tracking_uri("http://127.0.0.1:5000")
```

> **⚠️ Gotcha:** If you forget to call `set_tracking_uri()`, your code still works — it just writes to the local `mlruns` folder instead of the server. This is a silent difference. Always verify where your runs are being stored.

### Option C: Remote Server (For Teams — Mentioned for Awareness)

In production, teams run the MLflow server on a shared machine or cloud service, and everyone's code points to the same URI. This is how teams share experiments. Setup is beyond this tutorial's scope, but the concept is the same: just change the URI to point at the shared server.

---

## 2.3 Your First Tracked Run

Here's the anatomy of a tracked run:

```python
import mlflow

# 1. Name the experiment (creates it if it doesn't exist)
mlflow.set_experiment("my-first-experiment")

# 2. Start a run (everything inside this block is one run)
with mlflow.start_run():

    # 3. Log parameters (inputs)
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("max_epochs", 100)

    # --- Imagine training happens here ---
    # For now, we'll simulate results:
    accuracy = 0.88
    loss = 0.12

    # 4. Log metrics (outputs)
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("loss", loss)

    # 5. Log an artifact (any file)
    with open("notes.txt", "w") as f:
        f.write("This run used a small learning rate.")
    mlflow.log_artifact("notes.txt")
```

> **Term: `with mlflow.start_run():`** — This is a Python *context manager*. Everything indented under this line is grouped into one run. When the indented block finishes (or if it crashes), the run is automatically closed and saved. You don't have to manually open and close the run.

> **⚠️ Gotcha:** If you call `mlflow.log_param()` outside of a `with mlflow.start_run():` block, MLflow will automatically create a new "active run" — but it won't be cleaned up properly. Always use the `with` block.

---

## 2.4 Logging Metrics Over Time

Some metrics change during training — for example, the loss value usually decreases after each training round. MLflow can record this as a *time series* using the `step` parameter:

```python
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)

    # Simulate training for 5 epochs
    for epoch in range(1, 6):
        # In a real scenario, you'd train here and compute real loss
        simulated_loss = 1.0 / epoch  # loss decreasing over time

        # Log the metric at each step
        mlflow.log_metric("loss", simulated_loss, step=epoch)
```

> **Term: epoch** — One complete pass through your training data. Training usually involves many epochs, with the model improving each time.

In the MLflow UI, if you open this run and click on the `loss` metric, you'll see a line chart showing loss decreasing across epochs. This is extremely useful for spotting when a model stopped improving or when something went wrong.

---

## 2.5 Autologging: Let MLflow Log for You

Manually logging every parameter and metric is tedious. For the most popular ML libraries (like scikit-learn, TensorFlow, PyTorch, and XGBoost), MLflow can do this automatically with one line:

```python
import mlflow
import mlflow.sklearn  # import the autolog module for the library you're using

mlflow.sklearn.autolog()  # turn on automatic logging

# Now just train normally — MLflow logs everything automatically
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris

X, y = load_iris(return_X_y=True)

with mlflow.start_run():
    model = LogisticRegression(max_iter=200)
    model.fit(X, y)
    # MLflow automatically logged: parameters, metrics, and the model itself
```

> **⚠️ Gotcha:** Autologging logs *everything* the library exposes, which can be a lot of data. For complex models, the run might have dozens of logged values. This is usually fine, but be aware your run records can get large.

> **⚠️ Gotcha:** Autologging must be called *before* training begins. Calling `mlflow.sklearn.autolog()` after `model.fit(X, y)` does nothing.

---

## 2.6 Searching and Comparing Runs

After many runs, you want to find the best one. You can do this in the UI, or programmatically:

```python
import mlflow

# Search for all runs in an experiment where accuracy > 0.85
runs = mlflow.search_runs(
    experiment_names=["my-first-experiment"],
    filter_string="metrics.accuracy > 0.85",
    order_by=["metrics.accuracy DESC"]
)

print(runs[["run_id", "params.learning_rate", "metrics.accuracy"]])
```

`search_runs` returns a **pandas DataFrame** — a table where each row is a run and each column is a parameter or metric.

> **Term: pandas DataFrame** — A table structure in Python, like a spreadsheet in code. Rows and columns. If you're not familiar with pandas, just know that `runs[["column1", "column2"]]` selects specific columns from the table.

---

## 2.7 Run Tags: Adding Context

Beyond parameters and metrics, you can attach free-form labels called **tags** to a run. Tags are useful for things that aren't numeric — like who ran the experiment, which dataset version was used, or whether it's a baseline run.

```python
with mlflow.start_run():
    mlflow.set_tag("dataset_version", "v3.2")
    mlflow.set_tag("owner", "alice")
    mlflow.set_tag("model_type", "logistic_regression")

    mlflow.log_param("max_iter", 200)
    mlflow.log_metric("accuracy", 0.91)
```

---

## 🏋️ Hour 2 Exercise: Track a Real Experiment

**Goal:** Log a real training run with parameters, metrics (per epoch), an artifact, and tags. Then find your best run programmatically.

**Setup:**

```bash
pip install scikit-learn pandas
```

**Exercise Script — save as `exercise_2.py`:**

```python
import mlflow
import mlflow.sklearn
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import json

# Point at local server or use default (comment out if not running a server)
# mlflow.set_tracking_uri("http://127.0.0.1:5000")

mlflow.set_experiment("iris-classification")

# Try 3 different values for max_iter
for max_iter in [50, 100, 200]:

    with mlflow.start_run(run_name=f"logreg-maxiter-{max_iter}"):

        # Log parameters
        mlflow.log_param("max_iter", max_iter)
        mlflow.log_param("solver", "lbfgs")
        mlflow.set_tag("model_type", "logistic_regression")

        # Train
        X, y = load_iris(return_X_y=True)
        X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

        model = LogisticRegression(max_iter=max_iter)
        model.fit(X_train, y_train)

        # Evaluate
        preds = model.predict(X_test)
        acc = accuracy_score(y_test, preds)

        # Log metric
        mlflow.log_metric("accuracy", acc)

        # Log an artifact: save results to a JSON file
        results = {"max_iter": max_iter, "accuracy": acc}
        with open("results.json", "w") as f:
            json.dump(results, f)
        mlflow.log_artifact("results.json")

        print(f"max_iter={max_iter} → accuracy={acc:.4f}")

# Find the best run
print("\n--- Finding best run ---")
runs = mlflow.search_runs(
    experiment_names=["iris-classification"],
    order_by=["metrics.accuracy DESC"]
)
best = runs.iloc[0]
print(f"Best run: {best['run_id']}, accuracy={best['metrics.accuracy']:.4f}, max_iter={best['params.max_iter']}")
```

**Run it:**

```bash
python exercise_2.py
```

**Then open the UI** (`mlflow ui` in another terminal, go to `http://127.0.0.1:5000`) and:
- Find the `iris-classification` experiment
- Click "Compare" to compare all 3 runs side by side
- Click on any run and navigate to its Artifacts tab to see `results.json`

---

## 📝 Hour 2 Interview Questions

---

**Q1 (Conceptual): What is the difference between `log_param` and `log_metric`?**

**A:** `log_param` records a setting that was chosen *before* training — it's an input, logged once per run. `log_metric` records a measurement produced *during or after* training — it's an output. Metrics can also be logged multiple times in the same run using the `step` parameter, which creates a time series chart (useful for tracking progress across epochs). You can't do this with params — each param name maps to exactly one value per run.

---

**Q2 (Practical): Your team has 500 runs across several experiments. You want to find all runs where accuracy exceeded 90% and the learning rate was less than 0.05. How would you do this?**

**A:** Using `mlflow.search_runs()` with a filter string:

```python
mlflow.search_runs(
    experiment_names=["your-experiment"],
    filter_string="metrics.accuracy > 0.9 AND params.learning_rate < '0.05'",
    order_by=["metrics.accuracy DESC"]
)
```

Note that parameter values are stored as strings, so the comparison for params uses string comparison. For numeric filtering, metrics are better — they're stored as numbers.

---

**Q3 (Gotcha): You call `mlflow.sklearn.autolog()` and then train your model, but nothing appears in the MLflow UI. What are the most likely causes?**

**A:** Several possibilities:
1. `autolog()` was called inside the `with mlflow.start_run():` block instead of before it — autolog must be called before training starts.
2. The tracking URI is pointing to a different location than where the UI is reading from (e.g., the UI is reading `mlruns` locally but the code is writing to a server, or vice versa).
3. The MLflow UI is not running, or you're looking at the wrong experiment in the UI.
4. The library version doesn't support autolog (unlikely with major libraries, but possible).

---

**Q4 (Conceptual): What is a "run name" and is it required?**

**A:** A run name is an optional human-readable label for a run, set with the `run_name` parameter in `mlflow.start_run(run_name="my-run")`. It's not required — MLflow auto-generates a random name if you don't provide one (like "graceful-crane-412"). Run names are useful when you have many runs and want to quickly identify what each one represents without clicking into it. They appear in the UI run list. Importantly, run names are *not* unique — two runs can have the same name.

---

**Q5 (Gotcha): You log `mlflow.log_param("learning_rate", 0.01)` in one run and `mlflow.log_param("learning_rate", "0.01")` in another. When you search runs using `params.learning_rate = '0.01'`, will both runs appear?**

**A:** Yes, because all parameter values are stored as strings internally, regardless of what Python type you pass. `0.01` (float) and `"0.01"` (string) both get stored as the string `"0.01"`. This is important for search: always use string comparisons for params in filter strings. It also means you can't do numeric range filtering on params (e.g., `params.learning_rate < 0.05` won't work as expected) — log numeric values you want to filter on as *metrics* instead of *params*.

---

---

# Hour 3 — MLflow Models: Packaging and Serving Models

## 3.1 The Problem with "Just Save the Model"

After training a model in Python, the most naive approach to saving it is to use whatever the library provides. For example, scikit-learn models are often saved with Python's `pickle` module. This works — but it creates problems:

- The person loading the model needs to know it's a scikit-learn model and use scikit-learn to load it
- The model can only be loaded in Python
- Different code is needed to serve (use) a scikit-learn model vs. a TensorFlow model
- There's no standard metadata: what data was it trained on? What input format does it expect?

MLflow Models solves this by defining a **standard way to save any model** — including a wrapper that lets you use the model without knowing what library created it.

---

## 3.2 The MLflow Model Format

When you save a model using MLflow, it creates a directory (folder) with a specific structure:

```
my_model/
├── MLmodel          ← The main descriptor file (YAML format)
├── model.pkl        ← The actual model file (format varies by library)
├── conda.yaml       ← Python environment dependencies
├── python_env.yaml  ← Alternative environment spec
└── requirements.txt ← pip dependencies
```

> **Term: YAML** — A human-readable text format for configuration files. Think of it as a structured way to write settings that both humans and programs can read.

The most important file is `MLmodel`. Here's what one looks like:

```yaml
artifact_path: model
flavors:
  python_function:
    env:
      conda: conda.yaml
      virtualenv: python_env.yaml
    loader_module: mlflow.sklearn
    model_path: model.pkl
    python_version: 3.9.7
  sklearn:
    code: null
    pickled_model: model.pkl
    serialization_format: cloudpickle
    sklearn_version: 1.0.2
mlflow_version: 2.0.0
```

Notice the `flavors` section. This model has two flavors:
- `python_function` (often abbreviated `pyfunc`) — a generic Python callable interface
- `sklearn` — the native scikit-learn interface

---

## 3.3 Model Flavors

A **flavor** is a specific way to save or load a model. Each ML library has its own flavor. The most important one is `python_function` (pyfunc), because it works for *any* model regardless of library.

| Flavor | Library | How to log | How to load |
|---|---|---|---|
| `sklearn` | scikit-learn | `mlflow.sklearn.log_model()` | `mlflow.sklearn.load_model()` |
| `tensorflow` | TensorFlow/Keras | `mlflow.tensorflow.log_model()` | `mlflow.tensorflow.load_model()` |
| `pytorch` | PyTorch | `mlflow.pytorch.log_model()` | `mlflow.pytorch.load_model()` |
| `xgboost` | XGBoost | `mlflow.xgboost.log_model()` | `mlflow.xgboost.load_model()` |
| `pyfunc` | Any (generic) | `mlflow.pyfunc.log_model()` | `mlflow.pyfunc.load_model()` |

The `pyfunc` flavor is what powers serving. When you deploy a model as an API, MLflow uses the `pyfunc` flavor — which means the serving code doesn't need to know whether the model is scikit-learn or TensorFlow.

> **⚠️ Gotcha:** Not every model automatically gets a `pyfunc` flavor. Library-specific flavors like `sklearn` and `tensorflow` automatically include a `pyfunc` flavor too. But custom models you define yourself need you to explicitly implement the `pyfunc` interface.

---

## 3.4 Logging a Model

There are two ways to log a model:

**Method 1: Inside a run (recommended)**

```python
import mlflow
import mlflow.sklearn
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

mlflow.set_experiment("iris-model-demo")

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

with mlflow.start_run():
    model = LogisticRegression(max_iter=200)
    model.fit(X_train, y_train)

    # Log the model as an artifact of this run
    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="iris-model"  # subfolder name within the run's artifacts
    )

    print("Model logged!")
```

After running this, go to the MLflow UI → find the run → click Artifacts → you'll see the `iris-model` folder with all the model files.

**Method 2: Save to disk (outside a run)**

```python
import mlflow.sklearn

mlflow.sklearn.save_model(model, path="./saved_model")
```

This is less common — you lose the connection to the experiment run. Prefer Method 1.

---

## 3.5 Model Signatures: Documenting What Your Model Expects

A **model signature** describes the input and output format your model expects. Think of it as a contract: "give me data in this exact shape, and I'll give you predictions in this exact shape."

> **Why does this matter?** Without a signature, anyone loading your model has to guess what format to pass in. With a signature, MLflow can validate inputs automatically and catch errors early.

```python
from mlflow.models import infer_signature
import mlflow.sklearn
import mlflow
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

with mlflow.start_run():
    model = LogisticRegression(max_iter=200)
    model.fit(X_train, y_train)

    # Infer the signature from training data and predictions
    predictions = model.predict(X_train)
    signature = infer_signature(X_train, predictions)

    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="iris-model",
        signature=signature  # attach the signature
    )
```

`infer_signature(inputs, outputs)` automatically figures out the data types and column shapes. You can also add an `input_example`:

```python
mlflow.sklearn.log_model(
    sk_model=model,
    artifact_path="iris-model",
    signature=signature,
    input_example=X_train[:3]  # 3 sample rows showing what input looks like
)
```

> **⚠️ Gotcha:** Signatures are recorded but not always enforced at load time — they depend on the serving method. When using `mlflow models serve`, signatures *are* validated. When loading with `mlflow.sklearn.load_model()` directly, you bypass validation.

---

## 3.6 Loading a Model

To load and use a logged model, you need its run ID and artifact path. You can find the run ID in the UI or programmatically.

**Load using the library-specific flavor:**

```python
import mlflow.sklearn

run_id = "your-run-id-here"  # copy from the UI
model = mlflow.sklearn.load_model(f"runs:/{run_id}/iris-model")

# Use it like a normal scikit-learn model
predictions = model.predict(X_test)
```

**Load using the generic pyfunc flavor (works for any library):**

```python
import mlflow.pyfunc

model = mlflow.pyfunc.load_model(f"runs:/{run_id}/iris-model")

# pyfunc models always have a .predict() method
predictions = model.predict(X_test)
```

> **Term: `runs:/`** — A special URI scheme MLflow uses to reference artifacts stored in a run. Format: `runs:/{run_id}/{artifact_path}`.

---

## 3.7 Serving a Model as a REST API

MLflow can turn any saved model into a REST API server with a single command. This means other programs can send data to it over HTTP and get predictions back — no Python needed on the client side.

> **Term: REST API** — A standard way for software to communicate over the internet. The "client" sends an HTTP request (a message), the "server" processes it and sends back a response. Used everywhere on the web.

**Step 1: Find your model's URI**

In the MLflow UI, click your run, then click the model artifact. You'll see a path like:
`runs:/abc123def456/iris-model`

**Step 2: Start the serving server**

```bash
mlflow models serve \
  --model-uri "runs:/YOUR_RUN_ID/iris-model" \
  --port 5001 \
  --no-conda
```

> **`--no-conda`** tells MLflow to serve using the current Python environment rather than creating a new conda environment. For learning purposes, this is simpler.

> **⚠️ Gotcha:** `mlflow models serve` requires the model to have been logged with a signature, or at least be in pyfunc-compatible format. If you get errors about the model format, ensure you logged it with `mlflow.sklearn.log_model()` (not just `save_model()`).

**Step 3: Send a prediction request**

In another terminal, test it with `curl` (a command-line tool for HTTP requests):

```bash
curl -X POST http://127.0.0.1:5001/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_split": {"columns": ["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"], "data": [[5.1, 3.5, 1.4, 0.2]]}}'
```

Or in Python:

```python
import requests
import json

data = {
    "dataframe_split": {
        "columns": ["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"],
        "data": [[5.1, 3.5, 1.4, 0.2]]
    }
}

response = requests.post(
    "http://127.0.0.1:5001/invocations",
    headers={"Content-Type": "application/json"},
    data=json.dumps(data)
)
print(response.json())
```

> **Term: `/invocations`** — The standard endpoint MLflow exposes for predictions. Every model served with `mlflow models serve` is reached at this path.

> **⚠️ Gotcha:** The JSON format for predictions has changed across MLflow versions. In MLflow 2.x, use `dataframe_split` format (as shown above). Older examples online may use `instances` or `inputs` format, which may not work.

---

## 🏋️ Hour 3 Exercise: Log, Sign, and Serve

**Goal:** Log a model with a signature, then serve it and make a prediction.

**Exercise Script — save as `exercise_3.py`:**

```python
import mlflow
import mlflow.sklearn
from mlflow.models import infer_signature
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

mlflow.set_experiment("iris-serving-demo")

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

with mlflow.start_run() as run:
    model = LogisticRegression(max_iter=200)
    model.fit(X_train, y_train)

    predictions = model.predict(X_train)
    signature = infer_signature(X_train, predictions)

    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="iris-classifier",
        signature=signature,
        input_example=X_train[:2]
    )

    run_id = run.info.run_id
    print(f"\nRun ID: {run_id}")
    print(f"Model URI: runs:/{run_id}/iris-classifier")
    print("\nNow run this in a new terminal:")
    print(f'mlflow models serve --model-uri "runs:/{run_id}/iris-classifier" --port 5001 --no-conda')
```

**Steps:**

1. Run `python exercise_3.py` — note the printed model URI
2. In a second terminal, run the `mlflow models serve` command it printed
3. Wait for the message `Listening at: http://127.0.0.1:5001`
4. In a third terminal or a Python script, send a prediction request using the `requests` example from section 3.7
5. Confirm you get a prediction (a class label: 0, 1, or 2 for Iris species)

**Bonus challenge:** Open the MLflow UI, find this run's artifacts, and read the `MLmodel` file. Identify the flavors listed.

---

## 📝 Hour 3 Interview Questions

---

**Q1 (Conceptual): What is a model flavor in MLflow, and why is the `pyfunc` flavor important?**

**A:** A flavor is a specific way to save and load a model, typically corresponding to the library that created it. Each major ML library has its own flavor (sklearn, tensorflow, pytorch, etc.). The `pyfunc` (Python function) flavor is special because it provides a universal interface: any model with a `pyfunc` flavor can be loaded and used with the same code, regardless of what library trained it. This is what makes MLflow's serving infrastructure library-agnostic — the serving layer only needs to know about `pyfunc`, not about every possible ML library.

---

**Q2 (Practical): You've trained a model and logged it. Six months later, you want to reload it in a completely fresh Python environment. What do you need?**

**A:** You need:
1. MLflow installed in the new environment
2. The model URI (e.g., `runs:/abc123/my-model`) or the path to the saved model directory
3. Access to the tracking server (or the `mlruns` folder) where the model was saved

If you load using `mlflow.pyfunc.load_model()`, MLflow will use the `conda.yaml` or `requirements.txt` stored with the model to tell you (or automatically set up) the right environment. The model's dependencies are baked into the saved artifact — that's one of the key advantages of the MLflow model format.

---

**Q3 (Gotcha): What is the difference between `mlflow.sklearn.log_model()` and `mlflow.sklearn.save_model()`?**

**A:** `log_model()` saves the model *as an artifact of the current MLflow run* — it links the model to your experiment history, making it traceable. `save_model()` saves the model to a local path on disk with no connection to any run or experiment. In practice, always prefer `log_model()` unless you have a specific reason to save outside of a run, because the traceability is exactly what MLflow is designed to provide.

---

**Q4 (Gotcha): A junior team member says "I'll skip the signature — it's optional anyway." What risks does this introduce?**

**A:** Without a signature:
1. No automatic input validation — bad data format might cause cryptic errors instead of clear messages
2. No self-documentation — other users loading the model have no way to know what data format it expects without reading the training code
3. The model can't be served with the MLflow UI's built-in "Try this model" feature
4. When using MLflow model serving with strict validation, signature-less models may fail

Signatures are technically optional but strongly recommended for any model that will be used by others or deployed.

---

**Q5 (Conceptual): You're serving a model with `mlflow models serve`. Explain what happens end-to-end when a client sends a POST request to `/invocations`.**

**A:**
1. The client sends an HTTP POST request with input data in JSON format to `http://.../invocations`
2. MLflow's serving layer receives the request and parses the JSON payload
3. If the model has a signature, the input is validated against it (data types, column names)
4. The data is converted to the format the model expects (e.g., a NumPy array or pandas DataFrame)
5. The model's `predict()` method is called with this data
6. The predictions are serialised back to JSON
7. The JSON response is returned to the client

The model is loaded into memory when the server starts — not on each request — so subsequent requests are fast.

---

---

# Hour 4 — Model Registry: Managing Models Across Their Lifecycle

## 4.1 The Problem Without a Registry

Imagine your team has trained 200 models over the past year. The best model from 3 months ago is running in production. A newer, better model is being tested. And there are experimental models in development.

With only MLflow Tracking, you'd need to:
- Remember which run ID corresponds to the production model
- Communicate to your deployment team which run to pull from
- Manually track whether the new model has been reviewed and approved
- Hope no one accidentally swaps in an untested model

The **Model Registry** solves this with a structured lifecycle system: models are registered, versioned, reviewed, promoted, and eventually archived — all with a clear audit trail.

---

## 4.2 Registering a Model

To add a model to the registry, you use `mlflow.register_model()`:

```python
import mlflow
import mlflow.sklearn
from mlflow.models import infer_signature
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

mlflow.set_experiment("iris-registry-demo")

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Train and log the model
with mlflow.start_run() as run:
    model = LogisticRegression(max_iter=200)
    model.fit(X_train, y_train)

    signature = infer_signature(X_train, model.predict(X_train))

    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="iris-model",
        signature=signature
    )

    run_id = run.info.run_id

# Register the model from this run into the Registry
model_uri = f"runs:/{run_id}/iris-model"

registered = mlflow.register_model(
    model_uri=model_uri,
    name="IrisClassifier"  # The registry name — like a product name
)

print(f"Registered model version: {registered.version}")
```

> **Important distinction:** The `name` in the registry ("IrisClassifier") is the *product name* — it groups all versions of this model together. The `artifact_path` ("iris-model") is just the folder name inside the run's artifacts. They can be the same or different — they serve different purposes.

After registering, check the MLflow UI → "Models" tab (in the left sidebar). You'll see "IrisClassifier" listed.

---

## 4.3 Model Versions

Every time you register a model (or update an existing registered model), it gets a new **version number** — automatically assigned, starting at 1.

```python
# Register again (perhaps after retraining)
mlflow.register_model(
    model_uri=f"runs:/{another_run_id}/iris-model",
    name="IrisClassifier"
)
# This creates version 2
```

You can always see all versions of a model in the UI by clicking on the model name in the Models tab.

---

## 4.4 Model Stages (Legacy) and Aliases (Current)

MLflow originally used **stages** to track where a model version was in its lifecycle. Stages are still supported but are considered legacy as of MLflow 2.x. The recommended modern approach is **aliases**.

### Understanding Stages (You'll Still See These)

Stages are fixed labels that move with a model version:

| Stage | Meaning |
|---|---|
| `None` | Default — just registered, not evaluated yet |
| `Staging` | Being tested, not in production |
| `Production` | Deployed and in live use |
| `Archived` | Retired, kept for history |

A model version can only be in *one* stage at a time. Moving a version to `Production` automatically moves the previous `Production` version to `Archived`.

```python
from mlflow import MlflowClient

client = MlflowClient()

# Transition version 1 to Staging
client.transition_model_version_stage(
    name="IrisClassifier",
    version=1,
    stage="Staging"
)

# Transition version 1 to Production
client.transition_model_version_stage(
    name="IrisClassifier",
    version=1,
    stage="Production"
)
```

> **⚠️ Gotcha:** Stage transitions in the registry require the MLflow tracking server to be running (not just local file-based tracking). If you're using the default `mlruns` folder and haven't started a server, registry operations may fail with a "Registry not available" error.

### Understanding Aliases (Recommended Modern Approach)

An **alias** is a custom nickname you assign to a specific model version. Unlike stages (which are a fixed set), you can create any alias name you want.

```python
client = MlflowClient()

# Assign the alias "champion" to version 2
client.set_registered_model_alias(
    name="IrisClassifier",
    alias="champion",
    version=2
)

# Assign "challenger" to version 3 (a new model being tested)
client.set_registered_model_alias(
    name="IrisClassifier",
    alias="challenger",
    version=3
)
```

Then, to load a model by alias:

```python
model = mlflow.pyfunc.load_model("models:/IrisClassifier@champion")
```

> **Term: `models:/`** — A URI scheme for referencing registered models. Format: `models:/{registered_name}/{version_or_stage_or_alias}`.

Examples:
- `models:/IrisClassifier/1` — load version 1 specifically
- `models:/IrisClassifier/Production` — load whatever version is in Production
- `models:/IrisClassifier@champion` — load whatever version has the "champion" alias

> **⚠️ Gotcha:** The `@` symbol for aliases was introduced in MLflow 2.x. If you're reading older tutorials or working with older code, they use `models:/IrisClassifier/Production` syntax (stage-based). Both work, but aliases are more flexible.

---

## 4.5 The MlflowClient: Programmatic Registry Control

The `MlflowClient` is the Python class that gives you full control over both tracking and the registry programmatically. You've already seen it for stage transitions. Here are other useful operations:

```python
from mlflow import MlflowClient

client = MlflowClient()

# List all registered models
for model in client.search_registered_models():
    print(f"Model: {model.name}")

# Get all versions of a model
versions = client.search_model_versions(f"name='IrisClassifier'")
for v in versions:
    print(f"  Version {v.version}: stage={v.current_stage}, run_id={v.run_id}")

# Add a description to a model version
client.update_model_version(
    name="IrisClassifier",
    version=1,
    description="Baseline logistic regression trained on full Iris dataset"
)

# Add a description to the registered model itself
client.update_registered_model(
    name="IrisClassifier",
    description="Classifies Iris flower species from sepal/petal measurements"
)

# Delete an alias
client.delete_registered_model_alias(name="IrisClassifier", alias="challenger")

# Delete a specific version (use carefully!)
client.delete_model_version(name="IrisClassifier", version=1)
```

> **⚠️ Gotcha:** Deleting a model version is permanent and cannot be undone. The version number is also never reused — if you delete version 1 and re-register, the next version will be 2 (or whatever comes next), not 1 again.

---

## 4.6 Loading a Registered Model for Inference

Once a model is in the registry, your production code can load it by name and alias — without needing to know the run ID:

```python
import mlflow.pyfunc
import pandas as pd

# Load the "champion" model
model = mlflow.pyfunc.load_model("models:/IrisClassifier@champion")

# Prepare input data
new_data = pd.DataFrame({
    "sepal length (cm)": [5.1],
    "sepal width (cm)": [3.5],
    "petal length (cm)": [1.4],
    "petal width (cm)": [0.2]
})

prediction = model.predict(new_data)
print(f"Predicted class: {prediction}")
```

This is powerful: when you promote a new model to the "champion" alias, your production code automatically uses the new model *without any code changes* — just re-run or restart the service.

---

## 4.7 A Typical Production Workflow

Here's how the registry fits into a real team's process:

```
1. Data scientist trains Model A → logged in Tracking
2. Model A registered as "IrisClassifier" version 3
3. Description and notes added via MlflowClient
4. ML engineer reviews version 3 → assigns alias "staging"
5. Integration tests run against models:/IrisClassifier@staging
6. Tests pass → alias "champion" moved to version 3
7. Previous "champion" (version 2) optionally gets alias "previous-champion"
8. Production service (which loads @champion) automatically uses version 3
```

This entire workflow is auditable: the registry records who created each version, when aliases were assigned, and the full version history.

---

## 🏋️ Hour 4 Exercise: Full Registry Workflow

**Goal:** Register a model, promote it through aliases, and load it by alias.

**Exercise Script — save as `exercise_4.py`:**

```python
import mlflow
import mlflow.sklearn
from mlflow import MlflowClient
from mlflow.models import infer_signature
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import pandas as pd

# Start the MLflow server first:
# mlflow server --host 127.0.0.1 --port 5000
# And set URI:
mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("iris-registry-workflow")

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

def train_and_register(max_iter, run_number):
    """Train a model and register it."""
    with mlflow.start_run(run_name=f"run-{run_number}") as run:
        model = LogisticRegression(max_iter=max_iter)
        model.fit(X_train, y_train)

        acc = accuracy_score(y_test, model.predict(X_test))
        mlflow.log_param("max_iter", max_iter)
        mlflow.log_metric("accuracy", acc)

        sig = infer_signature(X_train, model.predict(X_train))
        mlflow.sklearn.log_model(
            sk_model=model,
            artifact_path="model",
            signature=sig
        )
        run_id = run.info.run_id

    # Register
    result = mlflow.register_model(f"runs:/{run_id}/model", "IrisClassifierV2")
    print(f"Registered version {result.version} (max_iter={max_iter}, accuracy={acc:.4f})")
    return result.version

client = MlflowClient()

# Register two versions
v1 = train_and_register(50, 1)
v2 = train_and_register(200, 2)

# Assign aliases
print(f"\nAssigning 'challenger' to version {v1}")
client.set_registered_model_alias("IrisClassifierV2", "challenger", v1)

print(f"Assigning 'champion' to version {v2}")
client.set_registered_model_alias("IrisClassifierV2", "champion", v2)

# Add descriptions
client.update_model_version(
    name="IrisClassifierV2",
    version=v2,
    description=f"Best model: max_iter=200"
)

# List all versions
print("\n--- All versions ---")
for v in client.search_model_versions("name='IrisClassifierV2'"):
    print(f"Version {v.version}: aliases={v.aliases}, stage={v.current_stage}")

# Load champion and predict
print("\n--- Loading champion model ---")
champion = mlflow.pyfunc.load_model("models:/IrisClassifierV2@champion")

sample = pd.DataFrame(
    [[5.1, 3.5, 1.4, 0.2]],
    columns=["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"]
)
pred = champion.predict(sample)
print(f"Champion prediction for sample: class {pred[0]}")
print("\nDone! Check the MLflow UI → Models tab to see IrisClassifierV2")
```

> **Before running**, make sure your MLflow server is running:
> ```bash
> mlflow server --host 127.0.0.1 --port 5000
> ```

**After running, in the UI:**
- Go to Models → IrisClassifierV2
- See both versions listed
- Check the aliases on each version
- Click on a version to see its description and linked run

---

## 📝 Hour 4 Interview Questions

---

**Q1 (Conceptual): What is the difference between an MLflow experiment run and a registered model version?**

**A:** An experiment run is a record of one training execution — it stores parameters, metrics, and artifacts, including the model file. It lives in MLflow Tracking. A registered model version is a *promotion* of that run's model into the Registry — it gets a version number, lifecycle metadata (stage or alias), and a description. Not every run needs to be registered; you only register models that are candidates for production use. The run is the raw history; the registry entry is the curated catalogue.

---

**Q2 (Practical): Your production service loads a model with `mlflow.pyfunc.load_model("models:/MyModel@champion")`. A new, better model is trained and you want production to use it. What steps do you take, and do you need to redeploy the service?**

**A:** Steps:
1. Train the new model and log it (creates a new run)
2. Register the model: `mlflow.register_model(run_uri, "MyModel")` — this creates e.g. version 5
3. Test version 5 thoroughly (optionally assign it a "challenger" alias first)
4. Move the "champion" alias to version 5: `client.set_registered_model_alias("MyModel", "champion", 5)`

The production service does **not** need to be redeployed — the next time it calls `load_model("models:/MyModel@champion")`, it will load version 5 automatically. This is exactly why alias-based loading is preferred over version-based (`models:/MyModel/4`) in production code.

---

**Q3 (Gotcha): Can two model versions of the same registered model both be in the "Production" stage simultaneously?**

**A:** With the legacy stage system: no. When you transition a version to "Production", the previously "Production" version is automatically moved to "Archived". Only one version can hold each stage at a time. This is why the modern **alias** system is more flexible — you can have as many aliases as you want, and multiple versions can hold different aliases simultaneously (e.g., "champion" on v5 and "previous-champion" on v4 and "canary" on v6, all at the same time).

---

**Q4 (Gotcha): You delete model version 3. What version number will the next registered model get?**

**A:** Version 4 (or whatever the next sequential number is). Version numbers in MLflow are never reused — they're always monotonically increasing. Deleting a version creates a gap in the sequence. This is intentional: version numbers should be considered permanent identifiers, not a compact sequence. After deletion, the version history in the UI will show a gap (versions 1, 2, 4... — no 3).

---

**Q5 (Gotcha): Your team uses `mlflow.register_model()` in the training script itself. A data scientist accidentally runs the training script twice. What happens?**

**A:** Two versions of the model get registered — say, versions 7 and 8. If no one notices, there are now two nearly identical versions in the registry with no meaningful differentiation. This is a real operational risk. Best practices to prevent it:
1. Register models in a separate, gated step (not automatically in the training script)
2. Add a CI/CD review step before registration
3. At minimum, always add a description to registered versions so it's clear why they exist
4. Use the run metadata (params, metrics) attached to each version to distinguish them

---

---

# Appendix: Gotchas Summary Table

| # | Gotcha | What Goes Wrong | The Fix |
|---|---|---|---|
| 1 | Running UI and tracking server on same port | UI and server clash, one fails to start | Use port 5000 for the server, a different port if needed for other things. By default `mlflow ui` starts on 5000 and `mlflow server` also defaults to 5000 — don't run both simultaneously on the same port |
| 2 | `set_tracking_uri()` not called | Data goes to local `mlruns`, not your server | Always call `mlflow.set_tracking_uri(...)` before logging anything, or set the env variable `MLFLOW_TRACKING_URI` |
| 3 | `autolog()` called inside `start_run()` | Autologging may not work or be incomplete | Call `autolog()` *before* `start_run()`, at the top of your script |
| 4 | Logging params outside `start_run()` | MLflow creates an implicit "active run" that doesn't close cleanly | Always use the `with mlflow.start_run():` context manager |
| 5 | Parameter values stored as strings | Numeric range filters on params don't work as expected | Log numeric values you need to filter on as *metrics* (`log_metric`), not params |
| 6 | Serving JSON format changed in MLflow 2.x | Old `instances` format requests return errors | Use `dataframe_split` or `dataframe_records` format in MLflow 2.x |
| 7 | Deleting `mlruns` folder | All experiment history is permanently lost | Use a remote tracking server for anything you care about; treat local `mlruns` as temporary |
| 8 | Skipping model signatures | No input validation; model is hard to serve and use | Always use `infer_signature()` and log signatures with `log_model()` |
| 9 | Registry requires a server | `mlflow.register_model()` fails with local file tracking | Start a proper MLflow server (`mlflow server ...`) before using the Registry |
| 10 | Deleting model versions creates gaps | Version sequence is non-contiguous | This is expected and permanent; treat version numbers as opaque IDs, not counters |
| 11 | `models:/Name/Production` vs `@alias` | Stage-based URIs only work if stages are set; alias-based URIs only work in MLflow 2.x+ | Know your MLflow version; prefer aliases for new projects |
| 12 | `log_model()` vs `save_model()` | `save_model()` loses traceability to a run | Use `log_model()` inside a `start_run()` block |
| 13 | Same alias assigned to multiple versions | An alias always points to exactly one version; assigning it again moves it | When you reassign an alias, the previous version silently loses it |
| 14 | `--no-conda` flag for serving | Without it, `mlflow models serve` tries to create a conda environment, which may not exist | Use `--no-conda` in development; set up conda properly in production |

---

# Quick Reference Card

## Setup

```bash
pip install mlflow                        # Install
mlflow --version                          # Verify
mlflow ui                                 # Start UI (http://127.0.0.1:5000)
mlflow server --host 127.0.0.1 --port 5000  # Start tracking server
```

## Tracking — Core Logging

```python
import mlflow

mlflow.set_tracking_uri("http://127.0.0.1:5000")   # Point at server
mlflow.set_experiment("experiment-name")            # Create/select experiment

with mlflow.start_run(run_name="optional-name"):
    mlflow.log_param("key", value)                  # Log one parameter
    mlflow.log_params({"key1": v1, "key2": v2})     # Log many at once
    mlflow.log_metric("key", value)                 # Log one metric
    mlflow.log_metric("key", value, step=epoch)     # Log metric over time
    mlflow.log_metrics({"acc": 0.9, "loss": 0.1})  # Log many at once
    mlflow.set_tag("key", "value")                  # Log a tag
    mlflow.log_artifact("path/to/file.csv")         # Log a file
    mlflow.log_artifacts("path/to/folder/")         # Log a whole folder
```

## Tracking — Autologging

```python
mlflow.sklearn.autolog()      # Before training
mlflow.tensorflow.autolog()
mlflow.pytorch.autolog()
mlflow.xgboost.autolog()
```

## Tracking — Search

```python
runs = mlflow.search_runs(
    experiment_names=["my-experiment"],
    filter_string="metrics.accuracy > 0.9 AND params.solver = 'lbfgs'",
    order_by=["metrics.accuracy DESC"],
    max_results=10
)
# Returns a pandas DataFrame
```

## Models — Log and Load

```python
from mlflow.models import infer_signature

signature = infer_signature(X_train, model.predict(X_train))

# Log (inside start_run)
mlflow.sklearn.log_model(model, "artifact-folder", signature=signature, input_example=X_train[:3])
mlflow.tensorflow.log_model(model, "artifact-folder", signature=signature)
mlflow.pyfunc.log_model("artifact-folder", python_model=custom_model)

# Load
model = mlflow.sklearn.load_model(f"runs:/{run_id}/artifact-folder")
model = mlflow.pyfunc.load_model(f"runs:/{run_id}/artifact-folder")  # generic
model = mlflow.pyfunc.load_model("models:/ModelName@champion")        # from registry
model = mlflow.pyfunc.load_model("models:/ModelName/Production")      # by stage (legacy)
model = mlflow.pyfunc.load_model("models:/ModelName/3")               # by version
```

## Models — Serving

```bash
# Serve a model
mlflow models serve \
  --model-uri "runs:/RUN_ID/artifact-folder" \
  --port 5001 \
  --no-conda

# Predict (curl)
curl -X POST http://127.0.0.1:5001/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_split": {"columns": ["col1","col2"], "data": [[1.0, 2.0]]}}'
```

```python
# Predict (Python)
import requests, json
response = requests.post(
    "http://127.0.0.1:5001/invocations",
    headers={"Content-Type": "application/json"},
    data=json.dumps({"dataframe_split": {"columns": [...], "data": [[...]]}})
)
```

## Registry — Register and Manage

```python
import mlflow
from mlflow import MlflowClient

# Register
result = mlflow.register_model(f"runs:/{run_id}/artifact-folder", "RegisteredModelName")
print(result.version)  # auto-assigned version number

client = MlflowClient()

# Aliases (recommended)
client.set_registered_model_alias("ModelName", "champion", version=3)
client.delete_registered_model_alias("ModelName", "champion")

# Stages (legacy)
client.transition_model_version_stage("ModelName", version=2, stage="Production")
# Stages: "None", "Staging", "Production", "Archived"

# Descriptions
client.update_model_version("ModelName", version=1, description="Baseline model")
client.update_registered_model("ModelName", description="Predicts X from Y")

# List
client.search_registered_models()
client.search_model_versions("name='ModelName'")

# Delete (permanent!)
client.delete_model_version("ModelName", version=1)
client.delete_registered_model("ModelName")  # deletes ALL versions
```

## URI Schemes

| URI | Meaning |
|---|---|
| `runs:/{run_id}/{artifact_path}` | Artifact from a specific run |
| `models:/{name}/{version}` | Specific version number from registry |
| `models:/{name}/{stage}` | Current version in a stage (legacy) |
| `models:/{name}@{alias}` | Current version with an alias (MLflow 2.x+) |

## Key Concepts Cheat Sheet

| Term | One-line definition |
|---|---|
| Run | One execution of a training script |
| Experiment | A named group of related runs |
| Parameter | An input setting logged before training |
| Metric | A measured output of training |
| Artifact | A file produced and saved during a run |
| Flavor | A specific save/load interface for a model type |
| pyfunc | The universal, library-agnostic model flavor |
| Signature | A spec describing a model's expected inputs and outputs |
| Model Registry | A catalogue of versioned, lifecycle-managed models |
| Model Version | One entry in the registry (tied to a specific run) |
| Stage | A fixed lifecycle label (None/Staging/Production/Archived) — legacy |
| Alias | A custom nickname pointing to a specific version — modern |
| MlflowClient | The Python class for programmatic control of tracking + registry |

---

*Tutorial complete. Estimated total working time: 4–6 hours including exercises.*
*Reference: https://mlflow.org/docs/latest/index.html*
