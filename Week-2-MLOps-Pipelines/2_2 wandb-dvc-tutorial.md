# Weights & Biases + DVC: A Complete Beginner's Tutorial

> **What you'll build:** A tracked, versioned, reproducible ML project from scratch

---

## How This Tutorial Is Organized

Each "hour" block follows the same pattern: we explain the *problem* first, then introduce the tool that solves it, walk through the core concepts with examples, and finish with a hands-on exercise. At the end of each block you'll find five interview-style questions with detailed answers.

---

# Hour 1 — Introduction to Weights & Biases (W&B)

## 1.1 The Problem: "Which Training Run Was That?"

Imagine you're training a machine learning model — a program that learns patterns from data. You try different settings (called **hyperparameters** — knobs you turn before training starts, like learning rate or batch size). After a dozen attempts, you get a great result... but you can't remember which combination of settings produced it.

You might try keeping notes in a spreadsheet, but that breaks down fast:

- You forget to log a run.
- You can't easily compare charts side by side.
- Sharing results with a teammate means emailing screenshots.

This is the **experiment tracking** problem, and it's what Weights & Biases (W&B) solves.

## 1.2 What Is W&B?

**Weights & Biases** (often written "W&B" or "wandb") is a cloud-hosted platform that automatically records everything about your ML experiments — the settings you used, the metrics your model produced at every step, and even the code you ran. You add a few lines of Python to your training script, and W&B handles the rest.

Think of it as a lab notebook that fills itself in and comes with built-in charts.

### Key Vocabulary

Before we go further, let's define terms you'll see constantly:

| Term | What It Means |
|------|---------------|
| **Run** | A single execution of your training script. Every time you click "run" (or type `python train.py`), W&B creates one run. Each run records its own settings and metrics. |
| **Project** | A folder that groups related runs together. For example, all runs for your cat-vs-dog classifier live in one project. |
| **Config** | The dictionary of hyperparameters for a run (learning rate, number of epochs, etc.). W&B stores this so you can compare configs across runs. |
| **Metric** | A number your model produces that tells you how well it's doing — for example, `accuracy` or `loss`. W&B logs these over time so you can see charts. |
| **Epoch** | One complete pass through your entire training dataset. If you train for 10 epochs, your model sees every example 10 times. |
| **Sweep** | An automated search over different hyperparameter combinations. Instead of manually trying learning_rate=0.01, then 0.001, then 0.0001, a sweep does this for you. |
| **Report** | A shareable document (like a Google Doc) that lives inside W&B, where you can embed charts from your runs and add notes. Great for sharing results with teammates or advisors. |

## 1.3 The W&B Workflow (Conceptual)

Here's what happens when you use W&B in a project, step by step:

1. **Install & authenticate.** You install the `wandb` Python package and log in with an API key (a secret password that links your computer to your W&B account).
2. **Initialize a run.** At the top of your training script, you call `wandb.init()`. This tells W&B "I'm starting a new experiment."
3. **Set your config.** You pass your hyperparameters to `wandb.init()` so they're recorded.
4. **Log metrics.** Inside your training loop, you call `wandb.log()` each time you want to record a number (like loss after each epoch).
5. **View results.** You open your browser, go to wandb.ai, and see interactive charts showing how every run performed.

That's the entire flow. Five steps.

## 1.4 Runs in Depth

A **run** is the atomic unit of W&B. When you call `wandb.init()`, W&B does three things behind the scenes:

1. Creates a unique ID for this run (like `run-3kx7f2`).
2. Starts recording any metrics you log.
3. Uploads everything to the W&B cloud so you can view it from any browser.

Each run gets a randomly generated name (like "dazzling-cloud-7") so you can tell them apart in the dashboard. You can also set your own name if you prefer.

Here's the simplest possible run:

```python
import wandb

# Start a run, grouped under the project "my-first-project"
run = wandb.init(project="my-first-project")

# Log a single metric
run.log({"accuracy": 0.85})

# Mark the run as finished
run.finish()
```

> **⚠️ Gotcha:** If you forget to call `run.finish()` (or use the `with` statement shown later), W&B may mark your run as "crashed." Always close your runs. The safest pattern is using a `with` block:
> ```python
> with wandb.init(project="my-project") as run:
>     run.log({"accuracy": 0.85})
> # run.finish() is called automatically when you exit the with block
> ```

## 1.5 Config: Recording Your Settings

When you start a run, you typically pass a **config** — a Python dictionary describing the hyperparameters for that experiment:

```python
config = {
    "learning_rate": 0.001,
    "epochs": 10,
    "batch_size": 32,
    "optimizer": "adam",
}

with wandb.init(project="my-first-project", config=config) as run:
    # Your training code here
    pass
```

Why bother? Because two weeks from now, when you look at your dashboard and see that run #7 had the best accuracy, you can instantly see *exactly* which settings produced it. No more guessing.

You can also access config values inside your script via `run.config`:

```python
with wandb.init(project="demo", config={"lr": 0.01}) as run:
    learning_rate = run.config["lr"]  # 0.01
```

## 1.6 Logging Metrics Over Time

The real power of W&B is watching metrics change during training. The `run.log()` method accepts a dictionary of metric names and values:

```python
for epoch in range(10):
    train_loss = train_one_epoch(model)  # your training logic
    val_acc = evaluate(model)            # your evaluation logic

    run.log({
        "train/loss": train_loss,
        "val/accuracy": val_acc,
        "epoch": epoch,
    })
```

W&B automatically creates line charts for every metric you log. The `"train/loss"` name uses a slash — this groups metrics into sections in the dashboard (a "train" section and a "val" section), keeping things tidy.

> **⚠️ Gotcha:** `wandb.log()` assigns an internal "step" counter that auto-increments with each call. If you call `run.log()` twice in the same loop iteration (once for training metrics, once for validation metrics), they'll end up on *different* steps and your charts won't align. Solution: log everything in a single `run.log()` call per step.

## 1.7 Sweeps: Automated Hyperparameter Search

Manually trying different hyperparameters is tedious. **Sweeps** automate this. You define a search space (which parameters to try, and what values), a search strategy (how to explore the space), and an optimization goal (what metric to optimize). W&B handles the rest.

### Search Strategies

W&B offers three search strategies:

- **Grid search:** Tries every possible combination. Thorough but slow — if you have 5 learning rates × 4 batch sizes × 3 optimizers, that's 60 runs.
- **Random search:** Picks combinations at random from the ranges you specify. Surprisingly effective and much faster than grid search for most problems.
- **Bayesian search:** Uses results from previous runs to make smarter guesses about which combinations to try next. Best when runs are expensive.

### Defining a Sweep

You describe your sweep in a Python dictionary (or a YAML file). Here's an example:

```python
sweep_config = {
    "method": "random",          # search strategy
    "metric": {
        "name": "val/accuracy",  # what we're optimizing
        "goal": "maximize",      # we want accuracy to go UP
    },
    "parameters": {
        "learning_rate": {
            "min": 0.0001,
            "max": 0.01,
        },
        "batch_size": {
            "values": [16, 32, 64],  # pick from this list
        },
        "epochs": {
            "value": 10,  # fixed — not swept
        },
    },
}
```

### Running a Sweep (Three Steps)

```python
# Step 1: Create the sweep on W&B's servers
sweep_id = wandb.sweep(sweep_config, project="my-project")

# Step 2: Define a training function
def train():
    with wandb.init() as run:
        lr = run.config.learning_rate
        bs = run.config.batch_size
        # ... train your model using lr and bs ...
        run.log({"val/accuracy": final_accuracy})

# Step 3: Launch the sweep agent — it calls train() repeatedly
wandb.agent(sweep_id, function=train, count=20)  # run 20 experiments
```

The **sweep agent** is a worker that asks the W&B server "what should I try next?", gets a set of hyperparameters, runs your training function with those hyperparameters, and repeats.

> **⚠️ Gotcha:** Inside a sweep, do **not** pass `config=` to `wandb.init()`. The sweep controller injects the config automatically. If you hardcode a config, it will override the sweep's choices and every run will use the same values.

## 1.8 Reports: Sharing Your Findings

**Reports** are collaborative documents inside W&B. You can drag in charts from any of your runs, add Markdown text, and share a link with your team. Think of them as a living lab report that's always connected to your actual experiment data.

Common uses for reports:

- Summarizing a round of experiments for your manager or advisor.
- Documenting what you tried, what worked, and what didn't (a "decision log").
- Creating a reproducible record that you can revisit months later.

You create reports from the W&B web dashboard — there's no code involved.

## 1.9 Hands-On Exercise: Your First W&B Run

**Goal:** Install W&B, create an account, and log a simulated training run.

### Steps

1. **Create an account** at [wandb.ai/site](https://wandb.ai/site) (free tier is sufficient).

2. **Install the library:**
   ```bash
   pip install wandb
   ```

3. **Log in from your terminal:**
   ```bash
   wandb login
   ```
   Paste your API key when prompted (find it at wandb.ai → User Settings → API Keys).

4. **Create this script** as `first_run.py`:

   ```python
   import wandb
   import random

   # Configuration for this experiment
   config = {
       "learning_rate": 0.01,
       "epochs": 10,
       "architecture": "simple_nn",
   }

   # Start the run
   with wandb.init(project="wandb-tutorial", config=config) as run:
       offset = random.random() / 5

       for epoch in range(1, config["epochs"] + 1):
           # Simulate training metrics (in a real project, these come from your model)
           acc = 1 - 2 ** -epoch - random.random() / epoch - offset
           loss = 2 ** -epoch + random.random() / epoch + offset

           # Log metrics for this epoch
           run.log({
               "train/accuracy": acc,
               "train/loss": loss,
               "epoch": epoch,
           })

           print(f"Epoch {epoch}: accuracy={acc:.4f}, loss={loss:.4f}")

   print("Done! Visit https://wandb.ai/home to see your run.")
   ```

5. **Run it:**
   ```bash
   python first_run.py
   ```

6. **View your results** at wandb.ai. Click into your "wandb-tutorial" project and explore the charts.

7. **Bonus:** Run the script 2–3 more times. Notice how each run appears separately in the dashboard, and you can overlay their charts.

---

### Hour 1 — Interview Questions

**Q1 (Conceptual): What is a "run" in W&B, and what information does it capture?**

A run is a single execution of your training script. It captures the hyperparameters you configured (via `wandb.config`), the metrics you logged over time (via `wandb.log()`), system information (GPU usage, Python version), and metadata like start time and duration. Runs are the atomic unit of experiment tracking — every comparison, chart, or report in W&B is built from runs.

**Q2 (Practical): You ran 50 experiments last week and found a great result, but you can't remember which settings produced it. How does W&B help?**

Every run stores its full config (hyperparameters) alongside its metrics. In the W&B dashboard, you can sort runs by any metric (e.g., sort by validation accuracy descending), click on the best run, and immediately see its config. You can also use the parallel coordinates plot or the hyperparameter importance panel to see which settings correlate with good performance.

**Q3 (Gotcha): A colleague tells you their sweep is running, but every run produces identical results. What's the most likely mistake?**

They're probably passing a hardcoded `config=` dictionary to `wandb.init()` inside the sweep's training function. This overrides the hyperparameters the sweep controller is trying to inject. The fix is to remove the `config=` argument from `wandb.init()` when running inside a sweep, and instead read hyperparameters from `run.config`, which the sweep populates automatically.

**Q4 (Conceptual): Explain the difference between grid search, random search, and Bayesian search in W&B Sweeps.**

Grid search exhaustively tries every combination of the hyperparameter values you specify — thorough but expensive. Random search samples combinations randomly from the defined ranges — often finds good results faster than grid because it explores more of the space per run. Bayesian search builds a probabilistic model of how hyperparameters affect the metric, then prioritizes combinations that are likely to improve the result — best when each run is expensive and you want to minimize total runs.

**Q5 (Practical): You're logging training loss and validation accuracy in two separate `wandb.log()` calls within the same training step. Your charts look misaligned. Why?**

Each call to `wandb.log()` increments an internal step counter. Two separate calls in the same loop iteration put the metrics on different steps, so the x-axes don't match. The fix is to combine both metrics into a single `wandb.log()` call: `run.log({"train/loss": loss, "val/accuracy": acc})`.

---

# Hour 2 — W&B Hands-On: Instrumenting a PyTorch Training Script

## 2.1 The Problem: "I Have a Training Script — Now What?"

You've seen the basics. Now imagine you have an actual PyTorch training script — a neural network that classifies images, predicts prices, whatever. How do you add W&B tracking to it without rewriting the whole thing?

The answer: you add roughly 5–10 lines of code. The rest of your script stays the same.

## 2.2 Key Vocabulary for This Section

| Term | What It Means |
|------|---------------|
| **PyTorch** | A popular open-source library for building and training neural networks in Python. |
| **DataLoader** | A PyTorch utility that feeds data to your model in small chunks (batches) during training. |
| **Loss function** | A mathematical formula that measures how wrong your model's predictions are. Lower is better. |
| **Optimizer** | An algorithm that adjusts your model's internal weights to reduce the loss. Common ones: SGD, Adam. |
| **Gradient** | A number that tells the optimizer which direction to adjust each weight. Computed automatically by PyTorch. |
| **`wandb.watch()`** | A W&B function that automatically logs your model's gradients and parameter values over time. |
| **Artifact** | A versioned file or directory stored in W&B — used for datasets, models, or any file you want to track. |

## 2.3 The Integration Pattern

Every PyTorch + W&B integration follows the same pattern:

```
1. Import wandb
2. Initialize a run with wandb.init()
3. (Optional) Call wandb.watch(model) to log gradients
4. Inside your training loop, call run.log() with your metrics
5. (Optional) Save your model as a W&B Artifact
6. Finish the run
```

Let's see this applied to a real training script.

## 2.4 A Complete Example: MNIST Digit Classifier

Below is a full, working PyTorch script that trains a simple neural network to recognize handwritten digits (the MNIST dataset). The lines marked with `# <-- W&B` are the only additions needed for full experiment tracking.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader
import wandb                                           # <-- W&B

# ── Hyperparameters ──────────────────────────────────
config = {
    "learning_rate": 1e-3,
    "epochs": 5,
    "batch_size": 64,
    "hidden_size": 128,
    "optimizer": "adam",
}

# ── Initialize W&B ───────────────────────────────────
run = wandb.init(                                      # <-- W&B
    project="mnist-pytorch",                           # <-- W&B
    config=config,                                     # <-- W&B
)                                                      # <-- W&B

# ── Data ─────────────────────────────────────────────
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,)),
])

train_dataset = datasets.MNIST("./data", train=True, download=True, transform=transform)
test_dataset  = datasets.MNIST("./data", train=False, transform=transform)

train_loader = DataLoader(train_dataset, batch_size=config["batch_size"], shuffle=True)
test_loader  = DataLoader(test_dataset,  batch_size=1000, shuffle=False)

# ── Model ────────────────────────────────────────────
class SimpleNN(nn.Module):
    def __init__(self, hidden_size):
        super().__init__()
        self.flatten = nn.Flatten()
        self.net = nn.Sequential(
            nn.Linear(28 * 28, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, 10),
        )

    def forward(self, x):
        return self.net(self.flatten(x))

model = SimpleNN(config["hidden_size"])
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=config["learning_rate"])

wandb.watch(model, log="all", log_freq=100)            # <-- W&B

# ── Training Loop ────────────────────────────────────
for epoch in range(1, config["epochs"] + 1):
    model.train()
    running_loss = 0.0

    for batch_idx, (data, target) in enumerate(train_loader):
        optimizer.zero_grad()
        output = model(data)
        loss = criterion(output, target)
        loss.backward()
        optimizer.step()

        running_loss += loss.item()

    avg_loss = running_loss / len(train_loader)

    # ── Evaluation ───────────────────────────────────
    model.eval()
    correct = 0
    with torch.no_grad():
        for data, target in test_loader:
            output = model(data)
            pred = output.argmax(dim=1)
            correct += pred.eq(target).sum().item()

    accuracy = correct / len(test_dataset)

    # ── Log to W&B ───────────────────────────────────
    run.log({                                          # <-- W&B
        "epoch": epoch,                                # <-- W&B
        "train/loss": avg_loss,                        # <-- W&B
        "val/accuracy": accuracy,                      # <-- W&B
    })                                                 # <-- W&B

    print(f"Epoch {epoch}: loss={avg_loss:.4f}, accuracy={accuracy:.4f}")

# ── Finish ───────────────────────────────────────────
run.finish()                                           # <-- W&B
```

Notice: the PyTorch code (model definition, training loop, evaluation) is completely standard. W&B is layered on top without altering any training logic.

## 2.5 Deep Dive: `wandb.watch()`

The `wandb.watch(model)` call tells W&B to automatically record:

- **Gradients:** The direction and magnitude of weight updates. Useful for debugging "vanishing gradients" (when gradients shrink to near-zero and training stalls) or "exploding gradients" (when they grow uncontrollably).
- **Parameters:** The actual weight values in your model over time.

The `log_freq=100` argument means "log every 100 batches." Setting this too low will slow your training; setting it too high means you miss details.

> **⚠️ Gotcha:** `wandb.watch()` only works with PyTorch models (nn.Module). If you're using a different framework (like TensorFlow), use the framework's specific W&B callback instead. Also, calling `wandb.watch()` more than once on the same model can cause duplicate logging.

## 2.6 Logging Rich Media

W&B can log more than just numbers. Here are common extras:

```python
# Log an image (e.g., a sample prediction)
run.log({"examples": wandb.Image(image_tensor, caption="Predicted: 7")})

# Log a table of predictions
table = wandb.Table(columns=["image", "predicted", "actual"])
table.add_data(wandb.Image(img), pred_label, true_label)
run.log({"predictions": table})

# Log a histogram of model weights
run.log({"weight_distribution": wandb.Histogram(model.fc1.weight.data.numpy())})
```

## 2.7 Saving Models as Artifacts

An **artifact** in W&B is a versioned file. You can save your trained model so that anyone on your team can download the exact model from a specific run:

```python
# After training is done
artifact = wandb.Artifact("mnist-model", type="model")
torch.save(model.state_dict(), "model.pth")
artifact.add_file("model.pth")
run.log_artifact(artifact)
```

Later, someone can retrieve it:

```python
run = wandb.init(project="mnist-pytorch")
artifact = run.use_artifact("mnist-model:latest")
artifact_dir = artifact.download()
# Load the model from artifact_dir/model.pth
```

## 2.8 Putting Sweeps Into Practice

Let's combine what we learned in Hour 1 (sweeps) with our PyTorch script. Here's how you'd sweep over learning rate and hidden size:

```python
import wandb

sweep_config = {
    "method": "bayes",
    "metric": {"name": "val/accuracy", "goal": "maximize"},
    "parameters": {
        "learning_rate": {"min": 1e-4, "max": 1e-2},
        "hidden_size": {"values": [64, 128, 256]},
        "batch_size": {"values": [32, 64]},
        "epochs": {"value": 5},
    },
}

sweep_id = wandb.sweep(sweep_config, project="mnist-sweep")

def train_sweep():
    with wandb.init() as run:
        cfg = run.config  # Sweep injects these automatically!

        # ... Build model with cfg.hidden_size ...
        # ... Create optimizer with cfg.learning_rate ...
        # ... Train for cfg.epochs ...
        # ... Log metrics with run.log() ...

wandb.agent(sweep_id, function=train_sweep, count=15)
```

> **⚠️ Gotcha:** When using sweeps, always access hyperparameters via `run.config` (not a local variable), because the sweep controller sets different values for each run.

## 2.9 Hands-On Exercise: Instrument Your Own Script

**Goal:** Take the MNIST training script from Section 2.4, run it, and then modify it for a sweep.

### Part A — Run the Base Script

1. Save the script from Section 2.4 as `train_mnist.py`.
2. Install dependencies:
   ```bash
   pip install torch torchvision wandb
   ```
3. Run it:
   ```bash
   python train_mnist.py
   ```
4. Go to wandb.ai, open the "mnist-pytorch" project, and explore:
   - The loss curve (should go down)
   - The accuracy curve (should go up)
   - The config tab (shows your hyperparameters)
   - The system tab (shows CPU/GPU usage)

### Part B — Add a Sweep

1. Create a new file `sweep_mnist.py`.
2. Move your training code into a function called `train_sweep()`.
3. Define a sweep config that searches over `learning_rate` (min 0.0001, max 0.01) and `hidden_size` (values: 64, 128, 256).
4. Use `wandb.sweep()` and `wandb.agent()` to run 10 experiments.
5. In the W&B dashboard, open the Sweeps panel and look at:
   - The parallel coordinates plot (which combos worked best?)
   - The hyperparameter importance chart (which parameter mattered most?)

---

### Hour 2 — Interview Questions

**Q1 (Practical): You have an existing PyTorch training script. What are the minimum changes needed to add W&B tracking?**

Four changes: (1) `import wandb`, (2) call `wandb.init(project=..., config=...)` before training, (3) add `run.log({...})` inside the training loop to record metrics each epoch, and (4) call `run.finish()` when done (or wrap everything in a `with wandb.init() as run:` block). Everything else in your PyTorch code stays the same.

**Q2 (Conceptual): What does `wandb.watch(model)` do, and when would you use it?**

It hooks into the model's backward pass to automatically log gradient and parameter values at a specified frequency. You'd use it when debugging training issues — for example, if your model isn't learning, you might check whether gradients are vanishing (shrinking toward zero) or exploding (growing extremely large). You wouldn't use it in production or when gradients aren't informative, since it adds overhead.

**Q3 (Gotcha): Your training script works fine without W&B, but after adding `wandb.init()`, it becomes very slow. What might be wrong?**

Several possibilities: (1) `wandb.watch()` with `log_freq=1` logs gradients for every single batch, which is extremely expensive — increase `log_freq` to 100 or more; (2) You're calling `run.log()` with large media objects (images, tables) every batch instead of every epoch; (3) Your network connection is slow and W&B is trying to sync in real time — you can set `WANDB_MODE=offline` to log locally and sync later with `wandb sync`.

**Q4 (Practical): How would you use W&B Artifacts to ensure your teammate can reproduce your exact model?**

After training, save the model as a W&B Artifact with `run.log_artifact()`. This versions the model file and links it to the specific run (and therefore the specific config and code) that produced it. Your teammate can then call `run.use_artifact("model-name:latest")` (or a specific version like `:v3`) to download the exact model file and see which run created it, including all hyperparameters.

**Q5 (Gotcha): You log `train/loss` in one `run.log()` call and `val/accuracy` in a separate `run.log()` call at the end of each epoch. The loss chart has twice as many data points as the accuracy chart. Explain.**

Every call to `run.log()` increments the global step counter. Two calls per epoch means two steps per epoch. The loss is logged at step 1, accuracy at step 2, loss at step 3, accuracy at step 4, and so on. The fix is to combine them: `run.log({"train/loss": loss, "val/accuracy": accuracy})`.

---

# Hour 3 — DVC Basics: Data Versioning

## 3.1 The Problem: "Git Can't Handle My Data"

You're building a model. Your code lives in Git (a tool that tracks changes to your files). But your training dataset is 2 GB. If you try to put it in Git:

- Git stores *every version* of every file internally. After 5 changes to your dataset, you're storing 10 GB.
- Cloning the repo takes forever.
- GitHub rejects files over 100 MB.

So people end up putting data in Google Drive or a shared folder, and writing things like `# download data from Google Drive link: ...` in their README. This creates problems:

- Which version of the data did you use for experiment #47?
- Your teammate has a slightly different version of the data. Your results don't match. You spend days debugging before realizing it's a data mismatch.

This is the **data versioning** problem.

## 3.2 What Is DVC?

**DVC** (Data Version Control) is a command-line tool that works alongside Git to version large files — datasets, trained models, or any binary file that's too big for Git. It works like this:

1. Your large data file stays **outside** of Git.
2. DVC creates a tiny **metadata file** (a `.dvc` file) that acts as a pointer to the actual data.
3. The `.dvc` file goes **into** Git. It's only a few lines of text.
4. The actual data is stored in a **remote storage** location you choose — an Amazon S3 bucket, Google Cloud Storage, a network drive, or even a local folder.

The result: `git log` shows you the history of your *data* (via the `.dvc` files), even though the data itself isn't in Git.

### Key Vocabulary

| Term | What It Means |
|------|---------------|
| **`.dvc` file** | A small text file that DVC creates to represent a large data file. It contains a hash (a unique fingerprint) of the data's contents. This file is tracked by Git. |
| **DVC cache** | A hidden local folder (`.dvc/cache`) where DVC stores copies of your data files. It uses the content hash to avoid duplicating identical files. |
| **DVC remote** | A storage location (like an S3 bucket) where you push your data so teammates can access it. Similar to how GitHub is a remote for your code. |
| **Hash / checksum** | A unique string (like `22a1a2931c8370d3aeedd7183606fd7f`) computed from a file's contents. If even one byte changes, the hash changes. DVC uses this to detect changes and track versions. |
| **`dvc add`** | The command that tells DVC to start tracking a file or folder. |
| **`dvc push`** | Uploads your tracked data to the remote storage. |
| **`dvc pull`** | Downloads tracked data from the remote storage to your local machine. |
| **`dvc checkout`** | Restores data files to match the version described by the current `.dvc` files (similar to `git checkout` for code). |

## 3.3 How DVC and Git Work Together

Here's the mental model. Think of two parallel tracks:

```
Git tracks:               DVC tracks:
├── train.py              ├── data/training_images/  (2 GB)
├── model.py              └── models/best_model.pkl  (500 MB)
├── requirements.txt
├── data/training_images.dvc   ← tiny pointer file
└── models/best_model.pkl.dvc  ← tiny pointer file
```

When you make a Git commit, the `.dvc` files are included. They record which *version* of the data corresponds to that commit. Later, you (or a teammate) can check out any Git commit and run `dvc checkout` to get the matching data.

## 3.4 Installation and Setup

```bash
# Install DVC (the base package)
pip install dvc

# If you'll use Amazon S3 as remote storage, also install:
pip install dvc-s3

# Other options: dvc-gdrive (Google Drive), dvc-gs (Google Cloud), dvc-azure
```

Then, inside your Git repository:

```bash
# Initialize DVC in an existing Git repo
cd my-project
dvc init
```

This creates a `.dvc/` directory (similar to `.git/`) that holds DVC's internal configuration.

> **⚠️ Gotcha:** `dvc init` must be run inside a Git repo. If you run it in a folder that hasn't been `git init`'d, you'll get an error. Always set up Git first, then DVC.

After initializing, commit the DVC setup files:

```bash
git add .dvc .dvcignore
git commit -m "Initialize DVC"
```

## 3.5 Tracking Your First File

Let's say you have a dataset file at `data/dataset.csv`:

```bash
dvc add data/dataset.csv
```

What happens:

1. DVC computes a hash of `dataset.csv`.
2. DVC moves the file into the local cache (`.dvc/cache`), then creates a link back to the original location so your code still works.
3. DVC creates `data/dataset.csv.dvc` — a tiny metadata file containing the hash.
4. DVC adds `data/dataset.csv` to a `.gitignore` file so Git won't try to track the big file.

Now you commit the *pointer file* to Git:

```bash
git add data/dataset.csv.dvc data/.gitignore
git commit -m "Track dataset with DVC"
```

What does a `.dvc` file look like inside? Something like this:

```yaml
outs:
- md5: 22a1a2931c8370d3aeedd7183606fd7f
  size: 37891850
  path: dataset.csv
```

That `md5` hash is the unique fingerprint. If the dataset changes, the hash changes, and the `.dvc` file changes — which Git will notice.

> **⚠️ Gotcha:** After running `dvc add`, always check that the data file is in `.gitignore` and that the `.dvc` file is staged for Git. Forgetting the `git add` step means your data version isn't actually recorded in Git history.

## 3.6 Remote Storage

The DVC cache is local — it's on your machine. To share data with teammates (or back it up), you configure a **DVC remote**.

### Configuring an S3 Remote

```bash
dvc remote add -d myremote s3://my-bucket/dvc-storage
```

The `-d` flag sets this as the **default** remote. Now:

```bash
# Upload data to the remote
dvc push

# Download data from the remote (on a different machine)
dvc pull
```

### Other Remote Types

```bash
# Google Drive
dvc remote add -d myremote gdrive://folder-id

# Google Cloud Storage
dvc remote add -d myremote gs://my-bucket/dvc-storage

# Local folder (for learning/testing — no cloud needed)
dvc remote add -d myremote /tmp/dvc-remote
```

> **⚠️ Gotcha:** For S3, make sure your AWS credentials are configured (via `aws configure` or environment variables). DVC doesn't have its own authentication — it uses whatever credentials your system already has for that storage service.

## 3.7 Versioning Data: The Full Cycle

Here's the typical workflow when your data changes:

```bash
# 1. Modify your data (add new samples, clean bad rows, etc.)
#    ... edit data/dataset.csv ...

# 2. Tell DVC about the change
dvc add data/dataset.csv

# 3. Commit the updated pointer to Git
git add data/dataset.csv.dvc
git commit -m "Update dataset: added 500 new samples"

# 4. (Optional) Tag the version for easy reference
git tag -a "dataset-v2" -m "Dataset with 500 new samples"

# 5. Push data to remote storage
dvc push
```

### Going Back to a Previous Version

```bash
# Switch Git to an older commit (or tag)
git checkout dataset-v1

# Tell DVC to restore the data files that match that commit
dvc checkout

# Now your data directory contains the v1 version
```

This is powerful: your code and data are synchronized through Git commits.

## 3.8 Hands-On Exercise: Version a Dataset with DVC

**Goal:** Create a project, track a dummy dataset with DVC, modify it, and switch between versions.

### Steps

1. **Set up the project:**
   ```bash
   mkdir dvc-tutorial && cd dvc-tutorial
   git init
   dvc init
   git add .dvc .dvcignore
   git commit -m "Initialize DVC"
   ```

2. **Create a dummy dataset:**
   ```bash
   mkdir data
   echo "id,name,score" > data/students.csv
   echo "1,Alice,85" >> data/students.csv
   echo "2,Bob,92" >> data/students.csv
   echo "3,Carol,78" >> data/students.csv
   ```

3. **Track it with DVC:**
   ```bash
   dvc add data/students.csv
   git add data/students.csv.dvc data/.gitignore
   git commit -m "Add initial dataset"
   git tag v1
   ```

4. **Configure a local remote (for practice — no cloud needed):**
   ```bash
   mkdir /tmp/dvc-remote
   dvc remote add -d localremote /tmp/dvc-remote
   git add .dvc/config
   git commit -m "Configure local DVC remote"
   dvc push
   ```

5. **Modify the data:**
   ```bash
   echo "4,Dave,95" >> data/students.csv
   echo "5,Eve,88" >> data/students.csv
   dvc add data/students.csv
   git add data/students.csv.dvc
   git commit -m "Add Dave and Eve to dataset"
   git tag v2
   dvc push
   ```

6. **Switch back to v1:**
   ```bash
   git checkout v1
   dvc checkout
   cat data/students.csv   # Should show only Alice, Bob, Carol
   ```

7. **Return to the latest version:**
   ```bash
   git checkout main
   dvc checkout
   cat data/students.csv   # Should show all 5 students
   ```

You've just versioned data the same way you version code.

---

### Hour 3 — Interview Questions

**Q1 (Conceptual): Why can't you just put large datasets directly into Git? What problem does DVC solve?**

Git stores every version of every file in its history. For large files, this makes the repository enormous — cloning becomes slow or impossible, and services like GitHub reject files over 100 MB. DVC solves this by keeping the actual data outside Git and instead storing small pointer files (`.dvc` files) in Git. These pointers contain a hash of the data, so Git tracks *which version* of the data existed at each commit without storing the data itself. The actual data lives in a separate storage location (local cache or a remote like S3).

**Q2 (Practical): A teammate clones your repo and runs the training script, but it fails because the data files are missing. What did they forget to do?**

They need to run `dvc pull` after cloning. When you clone a Git repo that uses DVC, you get the code and the `.dvc` pointer files, but not the actual data. `dvc pull` reads the pointers and downloads the matching data from the configured DVC remote. They'll also need the appropriate credentials for whatever remote storage is configured (e.g., AWS credentials for an S3 remote).

**Q3 (Gotcha): You ran `dvc add data/images/` but forgot to run `git add data/images.dvc` and `git commit`. What's the consequence?**

The data is tracked locally by DVC (it's in the cache), but the pointer file isn't recorded in Git history. This means: (1) There's no Git record of this data version — you can't check it out later, (2) If you modify the data and run `dvc add` again, the previous version is lost from Git's perspective, (3) Your teammates can't `dvc pull` this version because they won't have the `.dvc` file. Always pair `dvc add` with `git add` and `git commit` for the `.dvc` file.

**Q4 (Conceptual): Explain what the `.dvc` file contains and how DVC uses it.**

A `.dvc` file is a small YAML text file that contains: the MD5 hash (a unique fingerprint) of the tracked data, the file size, and the file path. DVC uses the hash to locate the correct version of the data in its cache or remote storage. When you run `dvc checkout`, DVC reads the hash from the `.dvc` file and restores the matching data from the cache. When the data changes, the hash changes, the `.dvc` file changes, and Git records that change — giving you version history.

**Q5 (Practical): You have 100 GB of image data and your team uses S3. Walk through the setup steps.**

(1) Install DVC with S3 support: `pip install dvc dvc-s3`. (2) Initialize DVC in your Git repo: `dvc init`. (3) Configure the S3 remote: `dvc remote add -d myremote s3://my-bucket/dvc-storage`. (4) Ensure AWS credentials are configured (`aws configure` or environment variables). (5) Track the data: `dvc add data/images/`. (6) Commit the pointer: `git add data/images.dvc data/.gitignore && git commit -m "Track images"`. (7) Push data to S3: `dvc push`. Teammates then clone the repo and run `dvc pull` to get the images.

---

# Hour 4 — DVC Pipelines: Reproducible ML Workflows

## 4.1 The Problem: "Run This, Then That, Then This Other Thing"

A real ML project isn't just one script. It's a series of steps:

1. **Prepare** — Download or clean raw data.
2. **Featurize** — Transform raw data into features the model can use.
3. **Train** — Train the model on the features.
4. **Evaluate** — Test the model and compute metrics.

You run these steps in order, and each step depends on the output of the previous one. When you change something — say, a parameter in the training step — you need to re-run training *and* evaluation, but *not* preparation or featurization (since those didn't change).

Doing this manually is error-prone. You might forget to re-run evaluation after retraining. Or you might re-run everything from scratch even when only one step changed, wasting time.

This is the **pipeline reproducibility** problem.

## 4.2 What Is a DVC Pipeline?

A **pipeline** in DVC is a formal definition of your multi-step workflow. You describe each step (called a **stage**) in a file called `dvc.yaml`, specifying:

- What **command** to run (e.g., `python train.py`).
- What **dependencies** the stage needs (input files, scripts, parameters).
- What **outputs** the stage produces (processed data, models, metrics).

DVC uses this information to build a **dependency graph** — a map of which stages depend on which. When you change something and run `dvc repro` (short for "reproduce"), DVC figures out which stages are affected and re-runs *only those stages*, in the right order. Stages that weren't affected are skipped entirely.

### Key Vocabulary

| Term | What It Means |
|------|---------------|
| **Stage** | A single step in your pipeline, defined in `dvc.yaml`. Each stage has a command, dependencies, and outputs. |
| **`dvc.yaml`** | The file where you define all your pipeline stages. Think of it as a recipe for your entire project. |
| **`dvc.lock`** | An auto-generated file that records the exact hashes of every dependency and output for each stage. This is how DVC knows whether something has changed. You should never edit this file manually. |
| **`params.yaml`** | A file where you store tunable parameters (like learning rate, train/test split ratio). Stages can declare dependencies on specific parameters, so DVC knows to re-run a stage when those parameters change. |
| **Dependency (dep)** | An input that a stage needs — a data file, a script, or a parameter. If any dependency changes, the stage is "invalidated" and must be re-run. |
| **Output (out)** | A file or directory that a stage produces. DVC tracks and caches these automatically. |
| **`dvc repro`** | The command that reproduces your pipeline. It checks what has changed since the last run and re-executes only the affected stages. |
| **DAG** | Directed Acyclic Graph — a fancy name for the dependency graph. "Directed" means the connections have a direction (prepare → featurize → train). "Acyclic" means there are no loops (a stage can't depend on its own output). |

## 4.3 Anatomy of `dvc.yaml`

Here's a complete example of a `dvc.yaml` file for a simple ML project:

```yaml
stages:
  prepare:
    cmd: python src/prepare.py
    deps:
      - src/prepare.py
      - data/raw.csv
    params:
      - prepare.split
      - prepare.seed
    outs:
      - data/prepared/

  featurize:
    cmd: python src/featurize.py
    deps:
      - src/featurize.py
      - data/prepared/
    params:
      - featurize.max_features
    outs:
      - data/features/

  train:
    cmd: python src/train.py
    deps:
      - src/train.py
      - data/features/
    params:
      - train.learning_rate
      - train.epochs
    outs:
      - models/model.pkl

  evaluate:
    cmd: python src/evaluate.py
    deps:
      - src/evaluate.py
      - models/model.pkl
      - data/features/
    metrics:
      - metrics.json:
          cache: false
```

Let's break this down stage by stage:

**`prepare` stage:** Runs `prepare.py`, which reads `data/raw.csv` and produces cleaned data in `data/prepared/`. It also depends on two parameters from `params.yaml` (the train/test split ratio and a random seed).

**`featurize` stage:** Depends on the *output* of `prepare` (`data/prepared/`). This is what creates the chain — if `prepare` re-runs and produces different output, `featurize` must re-run too.

**`train` stage:** Depends on features and training parameters. If you change `train.learning_rate` in `params.yaml`, DVC knows to re-run training (and evaluation, since it depends on the model).

**`evaluate` stage:** Notice `metrics:` instead of `outs:`. This tells DVC that `metrics.json` is a special metrics file. The `cache: false` means DVC won't cache this file (since it's small enough to store directly in Git).

## 4.4 The `params.yaml` File

Parameters live in a separate file, usually `params.yaml`:

```yaml
prepare:
  split: 0.2
  seed: 42

featurize:
  max_features: 100

train:
  learning_rate: 0.001
  epochs: 10
```

Your Python scripts read from this file:

```python
import yaml

with open("params.yaml") as f:
    params = yaml.safe_load(f)

split_ratio = params["prepare"]["split"]
seed = params["prepare"]["seed"]
```

Why use a separate params file? Because DVC can track *individual parameter values*. If you change `train.learning_rate` from 0.001 to 0.01, DVC knows that only the `train` and `evaluate` stages need to re-run — `prepare` and `featurize` are unaffected.

> **⚠️ Gotcha:** The parameter names in `dvc.yaml` (e.g., `train.learning_rate`) use dot notation to navigate the YAML structure. The dot separates nested keys: `train.learning_rate` means "the `learning_rate` key inside the `train` section of `params.yaml`."

## 4.5 Creating Stages with `dvc stage add`

You can write `dvc.yaml` by hand, or use the `dvc stage add` command:

```bash
dvc stage add -n prepare \
    -d src/prepare.py \
    -d data/raw.csv \
    -p prepare.split,prepare.seed \
    -o data/prepared/ \
    python src/prepare.py
```

The flags:
- `-n prepare` — name of the stage
- `-d` — dependency (repeat for each)
- `-p` — parameter dependency (comma-separated)
- `-o` — output (repeat for each)
- The last argument is the command to run

Both approaches (hand-editing `dvc.yaml` vs. using `dvc stage add`) produce the same result. For beginners, hand-editing is often easier to understand.

## 4.6 Reproducing the Pipeline

Once your `dvc.yaml` is defined:

```bash
# Run the entire pipeline
dvc repro
```

DVC reads `dvc.yaml`, builds the dependency graph, checks which stages are outdated (by comparing current file hashes against those in `dvc.lock`), and runs only what's needed.

After a successful run, DVC updates `dvc.lock` with the new hashes. You should commit both files:

```bash
git add dvc.yaml dvc.lock params.yaml
git commit -m "Run pipeline with initial parameters"
```

### Selective Reproduction

```bash
# Re-run only the train stage (and anything downstream of it)
dvc repro train
```

### Forced Re-run

```bash
# Re-run a stage even if DVC thinks nothing has changed
dvc repro --force train
```

> **⚠️ Gotcha:** `dvc repro` compares the *current state* of files against `dvc.lock`. If you modified a script but didn't save the file, DVC won't detect the change. If you modified `params.yaml` but the stage doesn't list that parameter in its `params:` section, DVC won't notice either.

## 4.7 Visualizing the Pipeline

DVC can show you the dependency graph right in your terminal:

```bash
dvc dag
```

Output:
```
+---------+
| prepare |
+---------+
      *
      *
      *
+-----------+
| featurize |
+-----------+
      *
      *
      *
+-------+
| train |
+-------+
      *
      *
      *
+----------+
| evaluate |
+----------+
```

This confirms that the stages run in the right order and that the dependencies are correct.

## 4.8 Comparing Parameters and Metrics

After making changes and re-running the pipeline, DVC can show you exactly what changed:

```bash
# Show which parameters changed between the current version and the last commit
dvc params diff

# Show how metrics changed
dvc metrics diff
```

Example output of `dvc metrics diff`:
```
Path           Metric    Old     New
metrics.json   accuracy  0.89    0.92
metrics.json   loss      0.31    0.25
```

This makes it trivial to see whether your change improved the model.

## 4.9 The `dvc.lock` File

When you run `dvc repro`, DVC creates (or updates) a `dvc.lock` file. This file records the exact state of every dependency and output for each stage — including file hashes, parameter values, and output hashes.

You should:
- **Commit** `dvc.lock` to Git (it's your "snapshot" of the pipeline's state).
- **Never edit** `dvc.lock` by hand — DVC manages it automatically.

The lock file is what makes reproduction possible: someone else checks out your commit, runs `dvc repro`, and DVC can verify whether their outputs match the recorded hashes.

> **⚠️ Gotcha:** If your pipeline involves randomness (e.g., random train/test splits, dropout), the outputs won't be byte-identical across runs even with the same inputs. Set random seeds in your code *and* track them as parameters so that reproduction is deterministic.

## 4.10 Hands-On Exercise: Build a DVC Pipeline

**Goal:** Create a three-stage pipeline (prepare → train → evaluate) for a toy project.

### Step 1 — Project Setup

```bash
mkdir pipeline-tutorial && cd pipeline-tutorial
git init
dvc init
git add .dvc .dvcignore
git commit -m "Initialize project"
```

### Step 2 — Create the Parameter File

Create `params.yaml`:
```yaml
prepare:
  split: 0.2
  seed: 42

train:
  learning_rate: 0.01
  epochs: 5
```

### Step 3 — Create the Scripts

Create `src/prepare.py`:
```python
"""Split raw data into train and test sets."""
import yaml
import os
import random

with open("params.yaml") as f:
    params = yaml.safe_load(f)

split = params["prepare"]["split"]
seed = params["prepare"]["seed"]

random.seed(seed)

# Generate some dummy data
data = [{"x": random.random(), "y": random.random()} for _ in range(100)]
random.shuffle(data)

split_idx = int(len(data) * (1 - split))
train_data = data[:split_idx]
test_data = data[split_idx:]

os.makedirs("data/prepared", exist_ok=True)

with open("data/prepared/train.txt", "w") as f:
    for row in train_data:
        f.write(f"{row['x']},{row['y']}\n")

with open("data/prepared/test.txt", "w") as f:
    for row in test_data:
        f.write(f"{row['x']},{row['y']}\n")

print(f"Prepared {len(train_data)} train, {len(test_data)} test samples")
```

Create `src/train.py`:
```python
"""Train a toy model (just computes average — placeholder for real training)."""
import yaml
import json

with open("params.yaml") as f:
    params = yaml.safe_load(f)

lr = params["train"]["learning_rate"]
epochs = params["train"]["epochs"]

# Read training data
with open("data/prepared/train.txt") as f:
    lines = f.readlines()
    values = [float(line.strip().split(",")[1]) for line in lines]

# "Train" — just compute the mean as our "model"
model = {"mean": sum(values) / len(values), "lr": lr, "epochs": epochs}

import os
os.makedirs("models", exist_ok=True)
with open("models/model.json", "w") as f:
    json.dump(model, f)

print(f"Trained model: mean={model['mean']:.4f}")
```

Create `src/evaluate.py`:
```python
"""Evaluate the model on test data."""
import json

with open("models/model.json") as f:
    model = json.load(f)

with open("data/prepared/test.txt") as f:
    lines = f.readlines()
    values = [float(line.strip().split(",")[1]) for line in lines]

# Compute mean absolute error
errors = [abs(v - model["mean"]) for v in values]
mae = sum(errors) / len(errors)

metrics = {"mae": round(mae, 4), "n_test": len(values)}

with open("metrics.json", "w") as f:
    json.dump(metrics, f, indent=2)

print(f"Evaluation: MAE={mae:.4f}")
```

### Step 4 — Define the Pipeline

Create `dvc.yaml`:
```yaml
stages:
  prepare:
    cmd: python src/prepare.py
    deps:
      - src/prepare.py
    params:
      - prepare.split
      - prepare.seed
    outs:
      - data/prepared/

  train:
    cmd: python src/train.py
    deps:
      - src/train.py
      - data/prepared/
    params:
      - train.learning_rate
      - train.epochs
    outs:
      - models/model.json

  evaluate:
    cmd: python src/evaluate.py
    deps:
      - src/evaluate.py
      - models/model.json
      - data/prepared/
    metrics:
      - metrics.json:
          cache: false
```

### Step 5 — Run the Pipeline

```bash
dvc repro
```

You should see all three stages execute in order.

### Step 6 — Commit Everything

```bash
git add dvc.yaml dvc.lock params.yaml src/ metrics.json
git commit -m "Initial pipeline run"
git tag v1
```

### Step 7 — Change a Parameter and Re-Run

Edit `params.yaml`: change `train.learning_rate` from `0.01` to `0.1`.

```bash
dvc repro
```

Notice that DVC **skips** the `prepare` stage (nothing changed for it) and only re-runs `train` and `evaluate`.

```bash
dvc params diff
dvc metrics diff
git add dvc.lock params.yaml metrics.json
git commit -m "Increase learning rate to 0.1"
git tag v2
```

### Step 8 — Visualize

```bash
dvc dag
```

You've built a reproducible, parameter-tracked ML pipeline.

---

### Hour 4 — Interview Questions

**Q1 (Conceptual): What is a DVC pipeline, and what problems does it solve?**

A DVC pipeline is a series of stages defined in a `dvc.yaml` file, where each stage specifies a command, its dependencies (input files, scripts, parameters), and its outputs. It solves two main problems: *automation* (DVC determines which stages need re-running when something changes and executes them in the correct order) and *reproducibility* (the `dvc.yaml` and `dvc.lock` files are committed to Git, so anyone can check out a commit and reproduce the exact pipeline results).

**Q2 (Practical): You changed a parameter in `params.yaml` but `dvc repro` says "Stage 'train' didn't change." What went wrong?**

The most likely cause is that the `train` stage's `params:` section in `dvc.yaml` doesn't list the specific parameter you changed. DVC only watches the parameters you explicitly declare. For example, if you added a new parameter `train.weight_decay` to `params.yaml` but the `train` stage only lists `- train.learning_rate` and `- train.epochs`, DVC won't detect the change. The fix is to add `- train.weight_decay` to the stage's `params:` list.

**Q3 (Gotcha): What is `dvc.lock` and why should you never edit it manually?**

`dvc.lock` is an auto-generated file that records the exact hash of every dependency, parameter value, and output for each stage after a successful pipeline run. DVC uses it to determine what has changed since the last run. Editing it manually could corrupt the hashes, causing DVC to either skip stages that should run or re-run stages unnecessarily. It could also break reproducibility, since the lock file would no longer accurately represent the pipeline state.

**Q4 (Conceptual): Explain the difference between `deps`, `params`, `outs`, and `metrics` in a `dvc.yaml` stage.**

`deps` (dependencies) are input files or directories the stage reads — if they change, the stage re-runs. `params` are key-value pairs from a parameters file (usually `params.yaml`) — they're a special kind of dependency that lets DVC track individual parameter values rather than whole files. `outs` (outputs) are files the stage produces — DVC tracks and caches them automatically. `metrics` are a special type of output for small measurement files (JSON, YAML, CSV) — DVC provides commands like `dvc metrics diff` to compare them across runs.

**Q5 (Practical): Your colleague runs `dvc repro` on your project and gets different results than you. Both of you are on the same Git commit. What could cause this?**

Several possibilities: (1) Non-deterministic code — randomness in the pipeline without fixed seeds means different runs produce different outputs. (2) Different software versions — a different version of Python, scikit-learn, or another library could produce slightly different results. (3) They haven't run `dvc pull` — if they don't have the same data files you had, the pipeline will produce different outputs. (4) Environment differences — things like floating-point behavior can vary across operating systems or CPU architectures. The fix is to pin random seeds (tracked as parameters), use a `requirements.txt` or lock file, and make sure everyone runs `dvc pull` before `dvc repro`.

---

# Appendix A — Gotchas Summary Table

| # | Tool | Gotcha | Consequence | Fix |
|---|------|--------|-------------|-----|
| 1 | W&B | Forgetting `run.finish()` or the `with` block | Run marked as "crashed" in dashboard | Always use `with wandb.init() as run:` |
| 2 | W&B | Two `run.log()` calls per step | Charts misalign; metrics on different x-axis steps | Combine all metrics into one `run.log()` call |
| 3 | W&B | Passing `config=` inside a sweep function | Sweep hyperparameters are overridden; all runs identical | Remove `config=` from `wandb.init()` in sweeps; read from `run.config` |
| 4 | W&B | `wandb.watch()` with low `log_freq` | Training becomes very slow | Set `log_freq=100` or higher |
| 5 | W&B | Calling `wandb.watch()` multiple times | Duplicate logging; inflated metrics | Call it once, after model creation |
| 6 | DVC | Running `dvc init` outside a Git repo | Error — DVC requires Git | Run `git init` first, then `dvc init` |
| 7 | DVC | Forgetting `git add` after `dvc add` | Data version not recorded in Git history | Always run `git add <file>.dvc` after `dvc add` |
| 8 | DVC | Missing AWS/cloud credentials for remote | `dvc push`/`dvc pull` fails | Configure credentials via `aws configure` or env vars |
| 9 | DVC | Parameter not listed in stage's `params:` | `dvc repro` doesn't detect the change | Add the parameter to the stage's `params:` list |
| 10 | DVC | Editing `dvc.lock` manually | Corrupted hashes; broken reproducibility | Never edit `dvc.lock` — let `dvc repro` manage it |
| 11 | DVC | Non-deterministic code without fixed seeds | Different outputs on each `dvc repro` | Set random seeds and track them as parameters |
| 12 | DVC | Forgetting to `dvc push` after `dvc add` | Teammates can't `dvc pull` the data | Run `dvc push` after committing |

---

# Appendix B — Quick Reference Card

## W&B Commands

| What You Want | Code |
|---------------|------|
| Install | `pip install wandb` |
| Log in | `wandb login` |
| Start a run | `run = wandb.init(project="name", config={...})` |
| Start a run (safe pattern) | `with wandb.init(project="name", config={...}) as run:` |
| Log metrics | `run.log({"loss": 0.5, "accuracy": 0.9})` |
| Log gradients | `wandb.watch(model, log="all", log_freq=100)` |
| Log an image | `run.log({"img": wandb.Image(tensor)})` |
| Save a model artifact | `artifact = wandb.Artifact("name", type="model")` → `artifact.add_file("model.pth")` → `run.log_artifact(artifact)` |
| Define a sweep | `sweep_id = wandb.sweep(sweep_config, project="name")` |
| Run a sweep | `wandb.agent(sweep_id, function=train_fn, count=20)` |
| Offline mode | `WANDB_MODE=offline python train.py` then `wandb sync` |
| End a run | `run.finish()` |

## DVC Commands

| What You Want | Command |
|---------------|---------|
| Install | `pip install dvc` (add `dvc-s3`, `dvc-gdrive`, etc. for remotes) |
| Initialize in a Git repo | `dvc init` |
| Track a file | `dvc add data/file.csv` |
| Add a remote | `dvc remote add -d myremote s3://bucket/path` |
| Upload data to remote | `dvc push` |
| Download data from remote | `dvc pull` |
| Restore data for current commit | `dvc checkout` |
| Run the pipeline | `dvc repro` |
| Run one stage only | `dvc repro <stage_name>` |
| Force re-run | `dvc repro --force` |
| View the pipeline graph | `dvc dag` |
| Compare parameters | `dvc params diff` |
| Compare metrics | `dvc metrics diff` |
| Check pipeline status | `dvc status` |
| Add a pipeline stage | `dvc stage add -n name -d dep -o out -p param cmd` |

## File Reference

| File | Tool | Tracked By | Purpose |
|------|------|-----------|---------|
| `data/file.dvc` | DVC | Git | Pointer to a large data file (contains hash, size, path) |
| `.dvc/config` | DVC | Git | DVC remote configuration |
| `.dvc/cache/` | DVC | Neither (local only) | Local cache of data file versions |
| `dvc.yaml` | DVC | Git | Pipeline definition (stages, deps, outputs, params) |
| `dvc.lock` | DVC | Git | Recorded state of last pipeline run (auto-generated) |
| `params.yaml` | DVC | Git | Hyperparameters and settings for pipeline stages |
| `metrics.json` | DVC | Git | Model evaluation metrics (special output type) |

---

# Appendix C — How W&B and DVC Work Together

W&B and DVC are complementary tools that solve different parts of the ML lifecycle:

| Concern | W&B Handles It | DVC Handles It |
|---------|---------------|----------------|
| Tracking training metrics (loss, accuracy) over time | Yes | No |
| Comparing hyperparameters across experiments | Yes | Limited (via `params diff`) |
| Interactive dashboards and charts | Yes | No |
| Automated hyperparameter search (sweeps) | Yes | No |
| Versioning large datasets | No | Yes |
| Versioning trained models as files | Partially (Artifacts) | Yes |
| Defining multi-step pipelines | No | Yes |
| Smart re-execution (only re-run what changed) | No | Yes |
| Sharing data with teammates via cloud storage | No | Yes |

A common setup in production teams: DVC manages the data and pipeline, and W&B is called from within the pipeline scripts to track metrics and hyperparameters. This gives you the best of both worlds — reproducible, versioned pipelines with rich experiment tracking and visualization.

---

*Tutorial complete. Work through the exercises at your own pace, and refer back to the gotchas table and quick reference card as you build real projects.*
