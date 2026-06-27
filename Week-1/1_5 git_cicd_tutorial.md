# Git & CI/CD for ML Tutorial
## Advanced Git · GitHub Actions · ML Pipelines · Full CI/CD Build

> **Level:** Beginner-friendly (basic Git knowledge assumed — you know `commit`, `push`, `pull`)
> **Goal:** By the end you'll have a full CI/CD pipeline on your ML model-serving repo: automated tests, linting, Docker image builds, and pushes on every commit.

---

## Setup Checklist

Make sure you have:

```bash
# Git (should already be installed)
git --version   # need 2.20+

# Docker (for Hour 3 & 4)
docker --version

# A GitHub account and a repo to work in
# (your ML serving repo from the FastAPI tutorial works perfectly)
```

You'll also need:
- A **GitHub account** — free tier is fine
- A repo pushed to GitHub (public or private)
- **Docker Hub account** (free) for image pushing in Hour 4 — sign up at hub.docker.com

---

## What Is CI/CD and Why Does It Matter for ML?

**CI (Continuous Integration)** means every time you push code, an automated system runs your tests, linter, and build — so you find out immediately if something is broken, instead of discovering it weeks later.

**CD (Continuous Delivery/Deployment)** means the pipeline also packages and ships your code automatically — building a Docker image and pushing it to a registry, ready to deploy.

For ML specifically, this matters because:
- Models are retrained and swapped out regularly — you need confidence a new model version didn't break the API
- ML code often has subtle bugs (wrong preprocessing, shape mismatches) that only tests catch
- Docker images ensure your serving environment is identical from development to production

---

---

# Hour 1 — Advanced Git

## Why This Matters

You already know the basics: `commit`, `push`, `pull`, `branch`. But real project work hits messier situations — a bug that appeared "somewhere in the last 50 commits," a fix you made on the wrong branch, a local experiment you want to run without touching your main checkout. The commands in this hour are the ones that turn Git from a save-button into a genuine time machine.

---

## 1.1 Rebase — Rewriting History

### The problem rebase solves

You branch off `main`, make three commits, and meanwhile a colleague pushes two commits to `main`. Now your branch's history has diverged. A regular `merge` creates a "merge commit" that says "these two branches were joined here" — over time your history fills up with these and becomes hard to read.

**Rebase** replays your commits *on top of* the updated `main`, as if you had branched off it today. The result is a clean, linear history.

```
Before rebase:
main:       A → B → C → D
your branch:      └→ X → Y

After rebase:
main:       A → B → C → D
your branch:              └→ X' → Y'
(X and Y are replayed on top of D)
```

```bash
git checkout my-feature-branch
git rebase main
```

If Git hits a conflict during replay, it pauses and tells you which file to fix. After fixing:

```bash
git add <conflicted-file>
git rebase --continue   # move to the next commit
# or
git rebase --abort      # bail out entirely and go back to where you started
```

> ⚠️ **Gotcha:** Never rebase a branch that other people are working on. Rebase rewrites commit hashes — if a colleague has your old commits, their history will conflict with your rewritten one. The rule: **rebase your own private branches; merge shared branches.**

---

## 1.2 Cherry-pick — Taking One Commit From Anywhere

### The problem cherry-pick solves

You made a critical bug fix on your feature branch, but `main` needs it *right now* — you don't want to merge the whole feature branch yet.

`cherry-pick` copies a single commit (or a range) from one branch and applies it to another.

```bash
# First, find the commit hash you want
git log my-feature-branch --oneline
# abc1234 Fix null pointer in preprocessor
# def5678 WIP: new model architecture

# Now apply just that one commit to main
git checkout main
git cherry-pick abc1234
```

> ⚠️ **Gotcha:** Cherry-pick *duplicates* a commit — it creates a new commit with the same changes but a different hash. The original commit still exists on the feature branch. If you later merge the feature branch, Git may apply those changes twice, causing conflicts. Use cherry-pick sparingly, for genuine hotfixes.

---

## 1.3 Bisect — Finding the Commit That Broke Things

### The problem bisect solves

Your model API returns wrong predictions. It worked fine 3 weeks ago. There have been 60 commits since then. You don't want to check each one manually.

`git bisect` does a **binary search** through your commit history. You tell it "good" and "bad" endpoints; it checks out the midpoint and asks you to test. Each answer eliminates half the remaining commits. 60 commits → found in ~6 steps.

```bash
git bisect start
git bisect bad                  # current commit is broken
git bisect good v1.2.0          # this tag/commit was working

# Git checks out a commit halfway between the two.
# Test your code (run the API, check predictions, run a test, etc.)
# Then tell Git the result:
git bisect good    # this commit is fine → bug is in the newer half
git bisect bad     # this commit is broken → bug is in the older half

# Repeat until Git says:
# "abc1234 is the first bad commit"

git bisect reset   # return to HEAD when done
```

You can also automate the testing step:

```bash
git bisect run pytest tests/test_predictions.py
# Git runs the test suite at each step and determines good/bad automatically
```

> ⚠️ **Gotcha:** `git bisect reset` is not optional — if you forget it, you'll be left on a detached HEAD pointing at some random historical commit, not your branch tip. Always reset when done.

---

## 1.4 Reflog — The Undo Button for Everything

### The problem reflog solves

You accidentally deleted a branch, reset to the wrong commit, or did a rebase that went badly. The commits seem gone. They're not — Git almost never actually deletes anything for about 90 days.

The **reflog** (reference log) records every time HEAD moved — every checkout, commit, reset, rebase, merge. It's your complete undo history.

```bash
git reflog
# Output:
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: Add preprocessing step
# 789abcd HEAD@{2}: commit: Initial model serving endpoint
# ...
```

To recover:

```bash
# Go back to where you were before the bad reset
git checkout HEAD@{1}

# Or recover a deleted branch
git checkout -b recovered-branch HEAD@{1}
```

> ⚠️ **Gotcha:** Reflog is **local only** — it's not pushed to GitHub. If you clone a fresh copy of a repo, the reflog starts empty. This is why it's a safety net for local mistakes, not a backup strategy.

---

## 1.5 Worktrees — Multiple Checkouts at Once

### The problem worktrees solve

You're in the middle of a big feature. A production bug comes in that needs an urgent fix on `main`. Normally you'd stash your work, switch branches, fix it, come back, unstash — and hope nothing conflicts. With worktrees, you check out a second branch *in a separate folder* without touching your current work.

```bash
# Create a second checkout of the repo in a new folder
git worktree add ../hotfix-branch main

# Now you have two working directories:
# /my-project         ← your feature branch, untouched
# /hotfix-branch      ← clean checkout of main

cd ../hotfix-branch
# fix the bug, commit, push

# When done, clean up
git worktree remove ../hotfix-branch
```

> ⚠️ **Gotcha:** You can't check out the same branch in two worktrees simultaneously — Git prevents this to avoid conflicts. If you try, you'll get an error like "already checked out." Check out a new branch in the worktree instead.

---

## ✏️ Hour 1 Exercise

**Goal:** Practice the history-manipulation commands on a safe test repo.

1. Create a throwaway repo: `mkdir git-practice && cd git-practice && git init`
2. Make 5 commits: `echo "step $i" >> log.txt && git add . && git commit -m "commit $i"` (in a loop or one at a time)
3. **Bisect:** Mark the first commit as good, the latest as bad, and run `git bisect start` — practice navigating to "the bad commit" manually.
4. **Reflog:** Run `git reset --hard HEAD~2` (this "loses" 2 commits). Then use `git reflog` to find and recover them with `git checkout -b recovered HEAD@{1}`.
5. **Rebase:** Create a branch, add 2 commits, go back to `main`, add 1 more commit, then rebase your branch onto main.

**Stretch:** Try `git bisect run` with an automated test — create a file `test.sh` that exits 0 or 1 depending on content in `log.txt`, and let bisect find the "bad" commit automatically.

---

---

# Hour 2 — GitHub Actions Basics

## Why This Matters

GitHub Actions is the automation layer built into GitHub. Every push, PR, tag, or schedule can trigger a **workflow** — a script that runs on GitHub's servers. This is how you turn "tests pass on my machine" into "tests pass on every commit, for everyone, automatically."

---

## 2.1 Core Concepts

Before writing any YAML, understand the four-level hierarchy:

```
Workflow          ← one .yml file in .github/workflows/
  └── Job         ← a group of steps that runs on one machine
        └── Step  ← a single command or action
              └── Action  ← a reusable unit of work (from the Marketplace or your own)
```

- **Workflow** — triggered by an event (a push, a PR, a schedule). Defined in `.github/workflows/*.yml`.
- **Job** — runs on a **runner** (a fresh virtual machine, usually Ubuntu). Jobs run in parallel by default; you can make them sequential with `needs:`.
- **Step** — either a shell command (`run:`) or an action (`uses:`).
- **Action** — a packaged, reusable step. `actions/checkout@v4` is the most common — it clones your repo into the runner.

---

## 2.2 Your First Workflow

Create this file in your repo:

```yaml
# .github/workflows/hello.yml
name: Hello World

on:
  push:                    # trigger on every push
    branches: [main]       # but only to main

jobs:
  greet:
    runs-on: ubuntu-latest   # use GitHub's Ubuntu runner

    steps:
      - name: Check out the code
        uses: actions/checkout@v4    # clones your repo onto the runner

      - name: Say hello
        run: echo "Hello from GitHub Actions! Branch is ${{ github.ref_name }}"

      - name: Show repo structure
        run: ls -la
```

Push this file to GitHub. Go to your repo → **Actions** tab → you'll see the workflow run. Click it to see logs for each step.

> ⚠️ **Gotcha:** The file must be in `.github/workflows/` (note the dot). A common mistake is putting it in `github/workflows/` without the dot — GitHub won't find it and no error is shown.

---

## 2.3 Runners

A **runner** is the machine that executes your workflow. GitHub provides:

| Runner | Label | Notes |
|---|---|---|
| Ubuntu | `ubuntu-latest` | Most common, fastest startup |
| Windows | `windows-latest` | Use only if you need Windows-specific behavior |
| macOS | `macos-latest` | Slowest and uses more of your free minutes |

Each job gets a **fresh, clean machine** — nothing carries over between jobs or workflow runs. This is why you need `actions/checkout@v4` in every job that touches your code.

> ⚠️ **Gotcha:** Because runners are ephemeral, any file you create in one job is gone in the next job — unless you upload it as an artifact (`actions/upload-artifact`) or pass it via a shared cache. Don't assume files persist between jobs.

---

## 2.4 Secrets and Environments

You'll need to pass credentials (Docker Hub password, API keys, cloud tokens) to your workflows without hardcoding them.

**Secrets** are encrypted values stored in GitHub:

1. Go to your repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name it (e.g. `DOCKER_PASSWORD`) and paste the value

Use it in your workflow:

```yaml
steps:
  - name: Log in to Docker Hub
    run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u myusername --password-stdin
```

**Environments** (Settings → Environments) let you group secrets per deployment target (staging, production) and add approval gates — so pushing to `main` can require a human to approve before deploying to production.

> ⚠️ **Gotcha:** Secrets are masked in logs (GitHub replaces them with `***`), but they're still passed as plain text to your runner. Never `echo` a secret directly in a run command without piping it — it may appear in error output. Use `--password-stdin` patterns instead of `--password ${{ secrets.X }}`.

---

## 2.5 Conditional Steps and Matrix Builds

Run a step only under certain conditions:

```yaml
- name: Deploy to production
  if: github.ref == 'refs/heads/main'   # only on main branch
  run: ./deploy.sh
```

**Matrix builds** run the same job with different combinations of variables — useful for testing across Python versions:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pytest
```

This creates three parallel jobs, one for each Python version.

---

## ✏️ Hour 2 Exercise

**Goal:** Get a real workflow running on your repo.

1. Create `.github/workflows/ci.yml` in your repo with:
   - Trigger: push to any branch
   - One job on `ubuntu-latest`
   - Steps: checkout, set up Python 3.11, install dependencies from `requirements.txt`, run `pytest`
2. Push the file. Go to the Actions tab and watch the run.
3. Deliberately break a test, push again, and confirm the workflow goes red.
4. Fix the test, push, confirm it goes green.

**Stretch:** Add a matrix build testing Python 3.10, 3.11, and 3.12. Add a conditional step that only runs on pushes to `main` (e.g. `echo "This is main — ready to deploy"`).

---

---

# Hour 3 — CI/CD for ML

## Why This Matters

A generic CI pipeline runs tests and lints code. An **ML CI pipeline** has extra concerns: Does the model load without errors? Does preprocessing produce the right shapes? Does the API return sensible outputs for known inputs? Does the Docker image build and run correctly? This hour adds those ML-specific layers.

---

## 3.1 The ML CI Pipeline — What to Check

A good ML CI pipeline validates four things in order, stopping if any step fails:

```
1. Code quality  →  2. Unit tests  →  3. Integration tests  →  4. Build & package
    (lint, format)     (fast, no model)   (with model loaded)      (Docker image)
```

Each step is a job. Earlier jobs are cheap and fast; later ones are slower. Fail fast at the cheap stages.

---

## 3.2 Linting and Formatting

**Linting** catches errors and style issues before code runs. **Formatting** ensures consistency.

```yaml
# .github/workflows/ci.yml
name: ML CI

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install linting tools
        run: pip install ruff black

      - name: Lint with ruff
        run: ruff check .          # catches unused imports, undefined names, etc.

      - name: Check formatting with black
        run: black --check .       # fails if any file isn't formatted
```

> ⚠️ **Gotcha:** `black --check` doesn't modify files — it just exits with a non-zero code if anything would change. This is what you want in CI. Run `black .` locally to actually fix formatting before pushing.

---

## 3.3 Unit Tests (Fast, No Model)

Unit tests should run in seconds. Don't load your actual model in unit tests — mock it.

```yaml
  unit-tests:
    runs-on: ubuntu-latest
    needs: lint          # only run if lint passes

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run unit tests
        run: pytest tests/unit/ -v --cov=app --cov-report=xml

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.xml
```

---

## 3.4 Integration Tests (With Model)

Integration tests load the real model and check end-to-end behavior. These are slower but catch a different class of bugs — shape mismatches, serialization errors, wrong output formats.

```yaml
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests    # only run after unit tests pass

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Download model artifact
        # If your model is stored in GitHub releases or S3:
        run: |
          curl -L https://github.com/yourname/repo/releases/download/v1.0/model.pkl \
               -o models/model.pkl

      - name: Run integration tests
        run: pytest tests/integration/ -v
        env:
          MODEL_PATH: models/model.pkl
```

Your integration test file might look like:

```python
# tests/integration/test_api_with_model.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_predict_returns_valid_score():
    response = client.get("/predict", params={"text": "hello world"})
    assert response.status_code == 200
    data = response.json()
    assert "score" in data
    assert 0.0 <= data["score"] <= 1.0   # score must be a valid probability

def test_predict_handles_empty_input():
    response = client.get("/predict", params={"text": ""})
    # Should return 422 (validation error) or a graceful response — not a 500
    assert response.status_code != 500
```

> ⚠️ **Gotcha:** Don't commit model files to Git — they're typically hundreds of MB and will bloat your repo permanently (Git history is forever). Store models in GitHub Releases, S3, GCS, or a model registry, and download them in CI as shown above.

---

## 3.5 Building and Pushing a Docker Image

The final stage of CI packages your app into a Docker image and pushes it to Docker Hub (or another registry). This image is what gets deployed.

First, make sure you have a `Dockerfile` in your repo:

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Then add a build job:

```yaml
  build-and-push:
    runs-on: ubuntu-latest
    needs: integration-tests    # only build if all tests pass
    if: github.ref == 'refs/heads/main'   # only push images from main

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            yourname/model-serving:latest
            yourname/model-serving:${{ github.sha }}
```

Tagging with `github.sha` (the commit hash) means every image is traceable back to the exact commit that built it.

> ⚠️ **Gotcha:** The `docker/build-push-action` with `push: true` will fail if the `docker/login-action` step didn't succeed. Always check that your secrets (`DOCKER_USERNAME`, `DOCKER_PASSWORD`) are correctly set in your repo's Settings before pushing this workflow.

---

## ✏️ Hour 3 Exercise

**Goal:** Build a multi-stage CI pipeline for your ML API.

1. Create `tests/unit/` and `tests/integration/` directories. Move any existing tests into `unit/`.
2. Write one integration test that actually calls your `/predict` endpoint and validates the response shape.
3. Create the full `ci.yml` with four jobs: `lint` → `unit-tests` → `integration-tests` → `build-and-push`.
4. Add `DOCKER_USERNAME` and `DOCKER_PASSWORD` secrets to your repo settings.
5. Push to a feature branch — confirm `lint`, `unit-tests`, and `integration-tests` run but `build-and-push` is **skipped** (because you're not on `main`).
6. Merge to `main` — confirm all four jobs run and an image appears in your Docker Hub account.

**Stretch:** Add a `ruff check` and `black --check` step to the lint job. Intentionally break formatting locally and push — confirm the pipeline fails at the lint stage before even running tests.

---

---

# Hour 4 — Build the Full CI/CD Pipeline

## Why This Matters

In Hour 3 you built each piece separately. This hour assembles them into a production-grade pipeline on your actual model-serving repo — with caching for faster runs, notifications when things break, and a clear promotion flow from commit to deployed image.

---

## 4.1 Dependency Caching

By default, every workflow run installs your Python dependencies from scratch. For a project with 30 packages this takes 60–90 seconds per job. Caching brings this down to 5–10 seconds on cache hits.

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: actions/setup-python@v5
    with:
      python-version: "3.11"

  - name: Cache pip dependencies
    uses: actions/cache@v4
    with:
      path: ~/.cache/pip
      key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
      restore-keys: |
        ${{ runner.os }}-pip-

  - name: Install dependencies
    run: pip install -r requirements.txt
```

The cache key includes a hash of `requirements.txt` — if you add or change a dependency, the hash changes, the old cache is skipped, and a fresh install runs. Otherwise the cache is reused.

> ⚠️ **Gotcha:** If you cache but forget to include `requirements.txt` in the hash key, you can end up with stale cached dependencies after an update — your CI passes locally but with old package versions. Always hash your dependency file in the cache key.

---

## 4.2 The Complete Pipeline

Here is the full production pipeline to add to your repo. Read through it in full before pasting — each section maps to something from Hours 2 and 3.

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOCKER_IMAGE: ${{ secrets.DOCKER_USERNAME }}/model-serving
  PYTHON_VERSION: "3.11"

jobs:
  # ── Stage 1: Code Quality ─────────────────────────────────────────────────
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
      - run: pip install ruff black
      - run: ruff check .
      - run: black --check .

  # ── Stage 2: Unit Tests ───────────────────────────────────────────────────
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
      - run: pip install -r requirements.txt
      - run: pytest tests/unit/ -v --cov=app --cov-report=xml
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage.xml

  # ── Stage 3: Integration Tests ────────────────────────────────────────────
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
      - run: pip install -r requirements.txt
      - run: pytest tests/integration/ -v
        env:
          MODEL_PATH: models/model.pkl

  # ── Stage 4: Build & Push Docker Image ───────────────────────────────────
  build-and-push:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3   # enables layer caching for Docker

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.DOCKER_IMAGE }}:latest
            ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
          cache-from: type=gha      # use GitHub Actions cache for Docker layers
          cache-to: type=gha,mode=max

      - name: Output image digest
        run: echo "Image pushed with SHA ${{ github.sha }}"
```

---

## 4.3 Browsing the Actions Marketplace

The **GitHub Actions Marketplace** (marketplace.github.com/actions) has thousands of pre-built actions. Rather than writing shell scripts for common tasks, search for an action first.

Useful ones for ML projects:

| Action | What It Does |
|---|---|
| `actions/checkout` | Clone your repo onto the runner |
| `actions/setup-python` | Install a specific Python version |
| `actions/cache` | Cache dependencies between runs |
| `actions/upload-artifact` | Save files between jobs (coverage reports, model files) |
| `docker/login-action` | Authenticate with any Docker registry |
| `docker/build-push-action` | Build and push Docker images with layer caching |
| `docker/setup-buildx-action` | Enable advanced Docker build features |
| `codecov/codecov-action` | Upload coverage reports to codecov.io |

When evaluating a Marketplace action, check: the number of stars, when it was last updated, and whether it's from a verified creator. Prefer actions from `actions/` (GitHub's own) and well-known vendors (`docker/`, `aws-actions/`).

> ⚠️ **Gotcha:** Always pin actions to a specific version tag (`@v4`, `@v3`) rather than a branch (`@main`). Using `@main` means a malicious or accidental update to that action's repo immediately affects all your workflows. Version tags are immutable.

---

## 4.4 Notifications on Failure

Knowing when CI breaks matters. Add a notification step that only runs on failure:

```yaml
  # Add this as a final job in your pipeline
  notify-on-failure:
    name: Notify on Failure
    runs-on: ubuntu-latest
    needs: [lint, unit-tests, integration-tests, build-and-push]
    if: failure()    # only runs if any previous job failed

    steps:
      - name: Send failure notification
        run: |
          echo "Pipeline failed on branch ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "See: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          # In practice: replace this with a curl to Slack/Teams webhook,
          # or use a Marketplace action like slackapi/slack-github-action
```

---

## 4.5 Protecting `main` with Branch Rules

The pipeline is only useful if people can't bypass it by pushing directly to `main`. Set this up in GitHub:

1. Repo → **Settings** → **Branches** → **Add branch protection rule**
2. Branch name pattern: `main`
3. Enable: **Require status checks to pass before merging**
4. Select your workflow jobs (`lint`, `unit-tests`, `integration-tests`) as required checks
5. Enable: **Require a pull request before merging**

Now nobody — including you — can push to `main` without passing CI.

---

## ✏️ Hour 4 Exercise

**Goal:** Get the complete pipeline running end-to-end on your ML serving repo.

1. Copy the full `ci-cd.yml` from section 4.2 into your repo.
2. Add the four required secrets to your repo settings: `DOCKER_USERNAME`, `DOCKER_PASSWORD`.
3. Create a `develop` branch and push it — confirm stages 1–3 run but `build-and-push` is skipped.
4. Open a Pull Request from `develop` to `main`. Confirm all checks run and appear on the PR.
5. Merge the PR. Confirm `build-and-push` runs and the new image appears on Docker Hub with both the `latest` tag and the commit SHA tag.
6. Set up branch protection rules on `main` as described in section 4.5.

**Stretch:** Add the `notify-on-failure` job. Test it by deliberately breaking a test, pushing to `develop`, and opening a PR — confirm the failure notification step runs. Replace the `echo` with a real Slack webhook if you have one.

---

---

# Gotchas Summary Table

| # | Gotcha | Where It Bites | Fix |
|---|---|---|---|
| 1 | Rebasing a shared branch | Hr 1 | Only rebase your own private branches; merge shared ones |
| 2 | Forgetting `git bisect reset` | Hr 1 | Always reset after bisect or you're left on detached HEAD |
| 3 | Cherry-pick causes duplicate conflicts on merge | Hr 1 | Use sparingly; expect and resolve the conflict when merging the full branch |
| 4 | Reflog is local only | Hr 1 | It can't recover commits you never made locally |
| 5 | Worktree can't check out same branch twice | Hr 1 | Create a new branch in the worktree instead |
| 6 | `.github/workflows/` vs `github/workflows/` | Hr 2 | Must have the leading dot — GitHub silently ignores the wrong path |
| 7 | Files don't persist between jobs | Hr 2 | Use `upload-artifact` / `download-artifact` to pass files between jobs |
| 8 | Secrets visible in error output | Hr 2 | Use `--password-stdin` patterns instead of inline `--password ${{ secrets.X }}` |
| 9 | `black --check` vs `black .` | Hr 3 | `--check` is for CI (read-only); without it, black silently reformats files on the runner |
| 10 | Committing model files to Git | Hr 3 | Use GitHub Releases, S3, or a model registry — download in CI |
| 11 | `build-and-push` fails without prior login | Hr 3 | Ensure `docker/login-action` step succeeds and secrets are set |
| 12 | Stale cache after dependency change | Hr 4 | Always include `hashFiles('requirements.txt')` in the cache key |
| 13 | Pinning actions to `@main` instead of `@v4` | Hr 4 | Always pin to a version tag — branch refs are mutable and a security risk |

---

---

# Quick Reference Card

## Commands

```bash
# Rebase your branch onto main
git rebase main

# Cherry-pick a single commit
git cherry-pick <commit-hash>

# Start bisecting
git bisect start
git bisect good <known-good-ref>
git bisect bad                      # mark current as bad
git bisect run pytest               # automate
git bisect reset                    # always run this when done

# See full local history (your undo log)
git reflog

# Create a second working directory
git worktree add ../other-branch <branch-name>
git worktree remove ../other-branch

# Trigger CI manually (re-run without a new commit)
# GitHub UI: Actions tab → select run → Re-run jobs
```

## GitHub Actions YAML Structure

```yaml
name: My Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  MY_VAR: value          # available to all jobs

jobs:
  my-job:
    runs-on: ubuntu-latest
    needs: previous-job   # sequential dependency

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Run something
        run: echo "hello"
        env:
          SECRET_VALUE: ${{ secrets.MY_SECRET }}
      - name: Only on main
        if: github.ref == 'refs/heads/main'
        run: ./deploy.sh
```

## Key GitHub Context Variables

| Variable | Value |
|---|---|
| `github.ref` | Full ref, e.g. `refs/heads/main` |
| `github.ref_name` | Short name, e.g. `main` |
| `github.sha` | Full commit hash |
| `github.event_name` | `push`, `pull_request`, etc. |
| `github.repository` | `owner/repo-name` |
| `runner.os` | `Linux`, `Windows`, `macOS` |

## Pipeline Stage Order

```
lint  →  unit-tests  →  integration-tests  →  build-and-push
 (fast)     (fast)           (slower)             (main only)
```

## File Structure

```
repo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml         # your pipeline
├── tests/
│   ├── unit/                 # fast tests, no model
│   └── integration/          # full API tests with model
├── Dockerfile
├── main.py
├── requirements.txt
└── models/                   # gitignored — downloaded in CI
```
