# Docker in 4 Hours — Beginner's Complete Tutorial

> **How to use this guide:** Each hour is self-contained. Read the explanation, then do the hands-on exercises before moving on. Don't skip the exercises — Docker clicks through doing, not just reading.
>
> **Prerequisites:** Docker Desktop installed (https://docs.docker.com/get-docker/). After installing, run `docker run hello-world` — if you see a success message, you're ready.

---

# Hour 1 — Docker Basics

## Why Docker Exists (Start Here)

You've probably hit this problem: code works perfectly on your laptop, but breaks when a teammate runs it, or when you deploy it to a server. The culprit is almost always the *environment* — different OS, different Python version, slightly different library installed.

Docker's answer: **ship the environment with your code.**

You bundle your application *plus* the exact OS, Python version, and libraries it needs into one self-contained unit. Anyone who runs it gets identical behavior — on any machine, on any OS.

Think of it like this:
- A Python virtualenv isolates your packages. 
- Docker goes deeper — it isolates the entire operating system layer too.

---

## The Three Core Concepts

Before touching any commands, get these three concepts straight. Everything else in Docker flows from them.

### 1. Image
A **read-only snapshot** of a filesystem. It contains an OS, your code, your dependencies — everything needed to run your app. Think of it like a *class* in programming, or a cake recipe. It's a template. You build it once.

### 2. Container
A **running instance** of an image. Think of it like an *object* created from that class, or the actual baked cake. You can spin up ten containers from the same image and they'll each be isolated from one another.

The key insight: **a container is just an image plus a thin writable layer on top.** When the container stops, that writable layer disappears (unless you save it). The underlying image is untouched.

### 3. Dockerfile
A **text file with instructions** for building an image. Docker reads it top-to-bottom and executes each step.

```
Dockerfile  →  (docker build)  →  Image  →  (docker run)  →  Container
   recipe            oven            cake          serve          plate
```

---

## Dockerfile Syntax

Here's a minimal Dockerfile for a Python app with every common instruction explained:

```dockerfile
# FROM: every Dockerfile starts here. This is your base image —
# a starting point someone else already built. You build on top of it.
FROM python:3.12-slim

# WORKDIR: sets the working directory inside the container for all
# subsequent instructions. Like doing `cd /app` and staying there.
WORKDIR /app

# COPY: copies files from your machine (left) into the image (right).
# The dot means "current directory on your machine".
COPY requirements.txt .

# RUN: executes a shell command at BUILD TIME. The result is baked
# into the image. Used for installing dependencies, compiling code, etc.
RUN pip install -r requirements.txt

# COPY your actual code AFTER installing dependencies (explained below)
COPY . .

# CMD: the default command that runs when a container starts.
# Use JSON array format (not a string).
CMD ["python", "app.py"]
```

**Other instructions you'll see:**

| Instruction | Purpose | When it runs |
|---|---|---|
| `ENV NAME=value` | Set environment variable | Build + runtime |
| `EXPOSE 8080` | Document that the app uses port 8080 (doesn't actually open it) | Documentation only |
| `ARG NAME` | Build-time variable (not available at runtime) | Build time |
| `ENTRYPOINT` | Like CMD, but harder to override. Often used together with CMD | Runtime |

---

## Layers and Caching — The Most Important Concept in Hour 1

Every instruction in a Dockerfile creates a **layer**. Docker stores each layer separately and **caches them**.

When you rebuild an image, Docker walks your Dockerfile top-to-bottom. For each instruction, it checks: "Has this layer changed since last time?" If no → it reuses the cached layer instantly. The moment a layer changes, **every layer below it is rebuilt** — cache is busted from that point down.

This has a critical practical consequence:

```dockerfile
# ❌ BAD — every time you change a single line of code,
#    Docker re-installs ALL your dependencies from scratch
COPY . .
RUN pip install -r requirements.txt

# ✅ GOOD — copy requirements FIRST, install them, THEN copy code.
#    Changing your code only rebuilds the last COPY layer.
#    pip install is reused from cache unless requirements.txt changes.
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

This is the single most common performance mistake beginners make. Dependencies change rarely; code changes constantly. Keep slow steps high, fast-changing files low.

---

## Essential Commands

```bash
# --- IMAGES ---
docker images                          # list images you have locally
docker pull python:3.12-slim           # download an image without running it
docker rmi <image_name>                # delete an image
docker build -t my-app .               # build image from Dockerfile in current dir, tag it "my-app"
docker build -t my-app:v1 .            # same but with a version tag

# --- CONTAINERS ---
docker run my-app                      # create and start a container from an image
docker run -it my-app bash             # interactive mode — drops you into a shell inside
docker run -d my-app                   # detached mode — runs in background
docker run -p 8080:80 my-app           # map port 8080 on YOUR machine to port 80 in container
docker run --rm my-app                 # auto-delete container when it stops (useful for testing)

docker ps                              # list RUNNING containers
docker ps -a                           # list ALL containers (including stopped)
docker stop <container_id>             # gracefully stop a running container
docker rm <container_id>               # delete a stopped container
docker logs <container_id>             # see what the container printed to stdout
docker exec -it <container_id> bash    # open a shell in a RUNNING container
```

> **Gotcha:** `docker run` always creates a *new* container. It doesn't restart an old one. If you `docker run` ten times, you get ten containers. Use `docker ps -a` to see the pile building up, and `docker rm` to clean them.

---

## 🛠️ Hands-On Exercise 1 — Build Your First Image

**What you'll do:** Write a Dockerfile, build an image, run it, observe caching.

**Step 1: Create your project folder**

```bash
mkdir docker-practice && cd docker-practice
```

**Step 2: Create a simple Python script**

Create a file called `hello.py`:
```python
print("Hello from inside a Docker container!")
print("Python is running in an isolated environment.")
```

**Step 3: Create a `requirements.txt`** (empty for now, but good habit)

```
# no dependencies yet
```

**Step 4: Create your Dockerfile** (no extension, exactly this name)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "hello.py"]
```

**Step 5: Build it**

```bash
docker build -t hello-docker .
```

You'll see Docker pulling the base image (first time only) and executing each step. Watch for the `Step X/Y` output.

**Step 6: Run it**

```bash
docker run hello-docker
```

You should see your print statements.

**Step 7: Observe caching**

Build again immediately:
```bash
docker build -t hello-docker .
```

Every step should say `CACHED`. Now edit `hello.py` (change the text), and build again. Notice: the `COPY requirements.txt` and `RUN pip install` steps are still CACHED. Only the `COPY . .` step (and below) rebuilds. That's the caching lesson in action.

**Step 8: Poke around inside**

```bash
docker run -it hello-docker bash
# you're now inside the container
ls                  # see /app
cat hello.py        # your file is in there
python --version    # the Python version from the image
exit                # leave the container
```

**✅ Checkpoint:** You've built an image, run a container, seen caching work, and explored the container's filesystem. That's the full loop.

---

# Hour 2 — Docker for Python

## Choosing a Base Image

When you write `FROM python:3.12`, you get the official Python image — but it's ~900MB because it comes with the full Debian OS. For production, you want something leaner.

The three main options for Python:

| Tag | Size | What it is | When to use |
|---|---|---|---|
| `python:3.12` | ~900MB | Full Debian + Python | Development, debugging |
| `python:3.12-slim` | ~130MB | Minimal Debian + Python | **Most Python apps** ← start here |
| `python:3.12-alpine` | ~50MB | Alpine Linux + Python | Only if you really need small; C extensions often break |
| `gcr.io/distroless/python3` | ~50MB | No OS shell at all | High-security production |

> **Gotcha with Alpine:** Alpine uses `musl libc` instead of `glibc`. Many Python packages (numpy, pandas, scikit-learn) ship pre-compiled wheels for glibc. On Alpine they'll try to compile from source, often fail, and always take forever. **Use `slim` by default for data science work.**

---

## Multi-Stage Builds

Here's a problem: to install some Python packages, you need build tools (gcc, compilers). But you don't need those tools in your final running image — they're just bloat and a security risk.

**Multi-stage builds** solve this: use one stage to build/compile, then copy only the results into a clean final image. Docker throws away the build stage.

```dockerfile
# ============ STAGE 1: Builder ============
# We name this stage "builder" so we can reference it later
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build tools needed to compile some packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# Install packages into a specific folder (not system-wide)
RUN pip install --user --no-cache-dir -r requirements.txt


# ============ STAGE 2: Final image ============
# Start completely fresh — no gcc, no build tools
FROM python:3.12-slim

WORKDIR /app

# Copy ONLY the installed packages from the builder stage
COPY --from=builder /root/.local /root/.local

# Copy your application code
COPY . .

# Make sure the installed packages are findable
ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]
```

The final image contains none of the build tools — just Python and your installed packages. Often cuts image size by 40-60%.

---

## Best Practices for Python Dockerfiles

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# 1. Separate requirements from code (cache efficiency)
COPY requirements.txt .

# 2. --no-cache-dir reduces image size (pip's cache is useless inside an image)
# 3. --upgrade pip avoids version warnings
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

COPY . .

# 4. Don't run as root inside the container
RUN useradd --create-home appuser
USER appuser

CMD ["python", "app.py"]
```

> **Security gotcha:** By default, everything in a container runs as root. That's fine for local development but bad practice for production. Adding a non-root user is a quick win.

---

## .dockerignore — The .gitignore for Docker

Without this, `COPY . .` copies everything — including your virtual environment, `.git` folder, `__pycache__`, test data, etc. This bloats your image and can expose sensitive files.

Create a `.dockerignore` file in the same directory as your Dockerfile:

```
# Python artifacts
__pycache__/
*.py[cod]
*.egg-info/
.eggs/

# Virtual environments
venv/
.venv/
env/

# Git
.git/
.gitignore

# Tests
tests/
pytest_cache/

# Local config / secrets — never bake these into images
.env
*.env
config/local.py

# OS
.DS_Store
```

> **Gotcha:** If you don't have a `.dockerignore`, and you have a `venv/` folder with hundreds of packages, Docker will copy all of it into the image *and* then install packages again on top. Always add `.dockerignore` before running your first build.

---

## 🛠️ Hands-On Exercise 2 — Python App with slim Image

**What you'll do:** Build an actual Python app image with proper slim base and .dockerignore.

**Step 1: Add a dependency**

In your `docker-practice` folder, update `requirements.txt`:
```
requests==2.32.3
```

**Step 2: Update hello.py to use it**

```python
import requests

response = requests.get("https://httpbin.org/json")
data = response.json()
print("Got response from the internet inside a container:")
print(data)
```

**Step 3: Create .dockerignore**

```
__pycache__/
*.pyc
.venv/
venv/
.env
```

**Step 4: Update your Dockerfile**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "hello.py"]
```

**Step 5: Build and run**

```bash
docker build -t hello-docker .
docker run hello-docker
```

**Step 6: Check the image size**

```bash
docker images hello-docker
```

Note the size. Now try changing the base image to `python:3.12` (full, not slim), rebuild, and compare sizes.

```bash
# Edit FROM line, then:
docker build -t hello-docker-fat .
docker images | grep hello-docker
```

**✅ Checkpoint:** You can see the size difference between full and slim images, and you understand why `.dockerignore` matters.

---

# Hour 3 — Docker Compose

## The Problem Compose Solves

Real applications aren't a single container. A typical web app has:
- A web server (Python/Flask/FastAPI)
- A database (Postgres, MySQL)
- A cache (Redis)
- Maybe a background worker

You could start each with `docker run` manually, passing a dozen flags each time. That's painful and error-prone.

**Docker Compose** lets you define your entire multi-container app in one YAML file and start everything with one command: `docker compose up`.

---

## The `docker-compose.yml` File

```yaml
# docker-compose.yml

version: "3.9"   # Compose file format version

services:
  
  # Each key under "services" is a container
  web:
    build: .                        # build from Dockerfile in current directory
    ports:
      - "8000:8000"                 # host_port:container_port
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydb
    depends_on:
      - db                          # don't start web until db is running
    volumes:
      - .:/app                      # mount current directory into container (for dev)

  db:
    image: postgres:15              # use pre-built image from Docker Hub (no Dockerfile needed)
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data   # named volume (data persists)

  redis:
    image: redis:7-alpine


# Named volumes are declared here (Compose manages them)
volumes:
  postgres_data:
```

---

## Key Compose Concepts

### Networks
Compose automatically creates a network and puts all your services on it. This means your `web` container can reach your `db` container using the **service name as the hostname** — `db:5432` just works. You don't need to manage IP addresses.

### Volumes (two types)

**Bind mounts** — link a folder on your machine to a folder in the container. Changes on either side are reflected immediately. Used for development so you don't have to rebuild the image every time you change code.
```yaml
volumes:
  - ./src:/app/src     # your machine's ./src  →  container's /app/src
```

**Named volumes** — Docker manages the storage location. Used for data that needs to persist between container restarts (databases, uploaded files). If you delete and recreate the container, the data survives.
```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

> **Gotcha:** A common beginner mistake is storing database data in a bind mount or not using a volume at all. If you run `docker compose down` without volumes declared, your database data is **gone**. Always use a named volume for databases.

### Environment Variables and `.env` files

Hard-coding passwords in `docker-compose.yml` is bad — that file goes into git. Instead, use a `.env` file:

```bash
# .env  (add this to .gitignore!)
POSTGRES_PASSWORD=supersecret
SECRET_KEY=abc123
```

```yaml
# docker-compose.yml references them with ${VAR_NAME}
services:
  db:
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
```

Compose automatically reads `.env` in the same directory.

---

## Essential Compose Commands

```bash
docker compose up              # start all services (attached — you see logs)
docker compose up -d           # start all services in background (detached)
docker compose down            # stop and remove containers (data in named volumes survives)
docker compose down -v         # stop, remove containers AND named volumes (data gone!)
docker compose logs            # see logs from all services
docker compose logs web        # logs from a specific service
docker compose ps              # see status of all services
docker compose exec web bash   # open a shell in the running "web" container
docker compose build           # rebuild images without starting
docker compose restart web     # restart a specific service
```

> **Gotcha:** `docker compose down` vs `docker compose stop` — `stop` just pauses containers but keeps them around. `down` removes them (but not named volumes unless you add `-v`). Most of the time you want `down`.

---

## 🛠️ Hands-On Exercise 3 — Multi-Service App with Compose

**What you'll do:** Run a Python web app alongside Redis using Compose.

**Step 1: Create a new project folder**

```bash
mkdir compose-practice && cd compose-practice
```

**Step 2: Create `app.py`** — a tiny Flask app that counts visits using Redis

```python
from flask import Flask
import redis
import os

app = Flask(__name__)
r = redis.Redis(host='redis', port=6379)   # 'redis' is the service name from compose

@app.route('/')
def index():
    count = r.incr('hits')
    return f'This page has been visited {count} times.\n'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=True)
```

**Step 3: Create `requirements.txt`**

```
flask==3.0.3
redis==5.0.4
```

**Step 4: Create `Dockerfile`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Step 5: Create `.dockerignore`**

```
__pycache__/
*.pyc
.venv/
```

**Step 6: Create `docker-compose.yml`**

```yaml
version: "3.9"

services:
  web:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app          # live reload — code changes reflect without rebuilding
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
```

**Step 7: Start everything**

```bash
docker compose up
```

Watch both services start. Open your browser to `http://localhost:8000` and refresh a few times. The counter increments — that's the Flask app talking to Redis across Compose's internal network, using the service name `redis` as the hostname.

**Step 8: Experiment**

```bash
# In a new terminal:
docker compose ps                     # see both containers running
docker compose logs redis             # just Redis logs
docker compose exec web bash          # shell into the web container
# try: redis-cli -h redis ping        # (won't work — redis-cli not installed in web container)
exit
docker compose exec redis redis-cli   # shell directly into Redis
# try: KEYS *   and   GET hits
exit

docker compose down                   # stop everything
docker compose up -d                  # start again in background
docker compose down                   # clean up
```

**✅ Checkpoint:** You ran two containers that communicated with each other. You used service names as hostnames. You used bind mounts for live code reloading. This is the foundation of every real multi-service app.

---

# Hour 4 — Practice: Containerize an sklearn Model

This is where it all comes together. You'll build a realistic container for a machine learning model — an inference service that accepts input and returns predictions.

## Project Structure

```
sklearn-docker/
├── model/
│   └── train.py          # train and save the model
├── app.py                # prediction script / API
├── requirements.txt
├── Dockerfile
├── .dockerignore
└── docker-compose.yml    # optional but included for good practice
```

---

## Step 1: Set Up the Project

```bash
mkdir sklearn-docker && cd sklearn-docker
mkdir model
```

---

## Step 2: Train and Save a Model

Create `model/train.py` — this trains a simple Iris classifier and saves it to disk:

```python
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import pickle
import os

# Load data
iris = load_iris()
X, y = iris.data, iris.target

# Split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Train
clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X_train, y_train)

# Evaluate
acc = accuracy_score(y_test, clf.predict(X_test))
print(f"Model accuracy: {acc:.3f}")

# Save
os.makedirs("model", exist_ok=True)
with open("model/iris_model.pkl", "wb") as f:
    pickle.dump(clf, f)

print("Model saved to model/iris_model.pkl")
```

**Run this locally** (outside Docker, once) to generate the saved model:

```bash
pip install scikit-learn   # if not already installed
python model/train.py
```

You should now have `model/iris_model.pkl`.

---

## Step 3: Create the Prediction Script

Create `app.py` — loads the model and makes predictions:

```python
import pickle
import sys

# Iris class names for readable output
CLASS_NAMES = ["setosa", "versicolor", "virginica"]

def load_model():
    with open("model/iris_model.pkl", "rb") as f:
        return pickle.load(f)

def predict(features: list[float]) -> str:
    model = load_model()
    prediction = model.predict([features])[0]
    probabilities = model.predict_proba([features])[0]
    
    class_name = CLASS_NAMES[prediction]
    confidence = probabilities[prediction]
    
    return f"Prediction: {class_name} (confidence: {confidence:.1%})"

if __name__ == "__main__":
    # Default test input if no args provided
    # Iris features: sepal_length, sepal_width, petal_length, petal_width
    if len(sys.argv) == 5:
        features = [float(x) for x in sys.argv[1:]]
    else:
        print("Usage: python app.py <sepal_length> <sepal_width> <petal_length> <petal_width>")
        print("Using default test input: 5.1 3.5 1.4 0.2 (setosa)")
        features = [5.1, 3.5, 1.4, 0.2]
    
    result = predict(features)
    print(result)
```

Test it locally first:
```bash
python app.py 5.1 3.5 1.4 0.2   # should predict setosa
python app.py 6.3 2.5 5.0 1.9   # should predict virginica
```

---

## Step 4: requirements.txt

```
scikit-learn==1.5.2
```

---

## Step 5: The Dockerfile

This uses a multi-stage build — everything from Hour 2 applied for real:

```dockerfile
# ============ Stage 1: Build ============
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# Install into user directory, not system
RUN pip install --user --no-cache-dir -r requirements.txt


# ============ Stage 2: Runtime ============
FROM python:3.12-slim

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local

# Copy model file and app
COPY model/iris_model.pkl model/iris_model.pkl
COPY app.py .

# Make packages findable
ENV PATH=/root/.local/bin:$PATH

# Default prediction (can be overridden at runtime)
CMD ["python", "app.py"]
```

---

## Step 6: .dockerignore

```
__pycache__/
*.pyc
.venv/
venv/
model/train.py          # no need to include training code in the runtime image
.git/
.env
*.ipynb
```

> **Important gotcha:** We copy the *saved model file* (`iris_model.pkl`) into the image, but not the training script. The container only needs to run inference, not train. Training happens once on your machine; the artifact (the `.pkl` file) gets shipped.

---

## Step 7: Build and Run

```bash
# Build
docker build -t sklearn-predictor .

# Run with default input
docker run sklearn-predictor

# Run with custom input (override the CMD)
docker run sklearn-predictor python app.py 6.3 2.5 5.0 1.9

# Run interactively
docker run -it sklearn-predictor bash
```

---

## Step 8: docker-compose.yml (Bonus)

For completeness — in a real project you might add this for running in a team environment:

```yaml
version: "3.9"

services:
  predictor:
    build: .
    # Override command to show usage
    command: python app.py 5.1 3.5 1.4 0.2
```

```bash
docker compose run predictor               # run with default from compose
docker compose run predictor python app.py 6.3 2.5 5.0 1.9  # custom input
```

---

## Step 9: Inspect and Verify Your Image

```bash
# Check final image size (should be much smaller than full python:3.12)
docker images sklearn-predictor

# See all layers
docker history sklearn-predictor

# Run a quick sanity check
docker run sklearn-predictor python -c "import sklearn; print(sklearn.__version__)"
```

**✅ Final Checkpoint:** You have a fully containerized ML model. Anyone with Docker can run your model with zero environment setup. No "works on my machine" — it works in the container.

---

# Key Gotchas Summary

| Gotcha | The Problem | The Fix |
|---|---|---|
| Layer cache order | Slow rebuilds because code and dependencies are in the wrong order | Copy `requirements.txt` first, install, then `COPY . .` |
| No .dockerignore | `venv/` and `.git` get baked into your image | Always add `.dockerignore` |
| Running as root | Security risk in production | Add a non-root user with `useradd` and `USER` |
| Alpine + data science | scikit-learn/pandas won't install from wheels | Use `slim`, not `alpine` |
| Forgetting named volumes | Database data lost on `docker compose down` | Always declare named volumes for persistent data |
| `docker run` piles up | Ten runs = ten stopped containers eating disk | Use `--rm` flag for one-off runs, or `docker ps -a` + `docker rm` |
| Hardcoded secrets | Passwords in `docker-compose.yml` end up in git | Use `.env` files + add `.env` to `.gitignore` |
| Training vs inference | Training code and model artifacts both in the image | Only copy the saved model; exclude training scripts in `.dockerignore` |

---

# Quick Reference Card

```bash
# Build
docker build -t my-app .
docker build -t my-app:v1 --no-cache .   # force full rebuild, ignore cache

# Run
docker run my-app                          # foreground
docker run -d my-app                       # background (detached)
docker run -it my-app bash                 # interactive shell
docker run --rm my-app                     # delete container when done
docker run -p 8080:8000 my-app             # port mapping
docker run -e MY_VAR=value my-app          # set environment variable

# Inspect
docker ps                                  # running containers
docker ps -a                               # all containers
docker logs <id>                           # container output
docker exec -it <id> bash                  # shell into running container
docker images                              # local images
docker history my-app                      # image layers

# Clean up
docker rm <id>                             # remove container
docker rmi my-app                          # remove image
docker system prune                        # remove ALL unused containers + images

# Compose
docker compose up -d                       # start all services
docker compose down                        # stop + remove containers
docker compose logs -f                     # follow logs
docker compose exec web bash              # shell into service
docker compose build                       # rebuild without starting
```

---

*You now have a solid foundation in Docker. The natural next steps are: environment variables and secrets management, Docker in CI/CD pipelines, and Kubernetes basics if you need to run containers at scale.*
