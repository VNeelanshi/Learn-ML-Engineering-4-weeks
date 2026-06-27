# GPU Containers, Optimization & Registries — A Beginner's 4-Hour Tutorial

> **Who this is for:** You know basic Docker (`docker build`, `docker run`, what an image and a container are) but you've never touched GPUs, image optimization, or cloud registries. By the end you'll have a PyTorch model running on a GPU inside a container, slimmed down, scanned for vulnerabilities, and pushed to AWS.
>
> **How to use this file:** Work top to bottom. Each hour is ~70% reading (the *why*) and ends with a hands-on exercise (the *how*). Don't skip the exercises — the concepts only stick once you've typed the commands.
>
> **What you need before starting:**
> - A Linux machine (Ubuntu assumed) with an **NVIDIA GPU**
> - Docker installed (`docker --version`)
> - For Hour 4: an AWS account + the AWS CLI

---

## Hour 1 — GPU Containers

### The problem we're solving

Containers were built to be *isolated* from the host machine. That isolation is the whole point — a container shouldn't be able to reach into your host and grab arbitrary hardware or files. But GPUs are physical hardware attached to the host, and deep-learning code needs direct access to them. So we have a tension: isolation by default vs. a workload that specifically needs to break through to a piece of hardware.

The naive fix would be to install NVIDIA drivers *inside* every image. This is a trap, and understanding why is the key insight of this hour:

- **Drivers are tied to the host kernel.** A driver compiled for one kernel version may not work on another. If you bake the driver into the image, that image only runs on machines with a matching kernel — you've destroyed portability, which is the reason you containerized in the first place.
- **Images would balloon in size.** Drivers and the full CUDA stack are large.
- **You'd have to rebuild every image** whenever the host driver updated.

### The solution: keep the driver on the host, inject it at runtime

NVIDIA's answer is the **NVIDIA Container Toolkit**. The mental model is simple and worth internalizing:

> The **driver lives on the host**. The **CUDA libraries live in the image**. At the moment a container starts, the toolkit mounts the host's driver and GPU device files *into* the container.

This is the best of both worlds: your image stays portable and reasonably small (it carries CUDA libraries but not the driver), and the container still gets real, direct GPU access — there's no emulation layer, so performance is essentially identical to running on bare metal.

> **Naming gotcha:** You'll find tons of old tutorials referencing **`nvidia-docker`** or **`nvidia-docker2`** and a special `--runtime=nvidia` flag. That's the *old* way. It's been replaced by the **NVIDIA Container Toolkit**, and the modern interface is the `--gpus` flag on plain `docker run`. If a guide tells you to install `nvidia-docker2`, it's out of date. Use the Toolkit.

### The three layers that must line up

GPU containers fail most often because of **version mismatches**. There are three layers stacked on top of each other:

1. **Host GPU driver** (e.g., driver 535) — installed on the host, *not* in the container.
2. **NVIDIA Container Toolkit** — the bridge that injects layer 1 into the container.
3. **CUDA libraries inside the image** (e.g., the `nvidia/cuda:12.x` base image).

**The cardinal rule:** the CUDA version inside your container must be **less than or equal to** what the host driver supports. A newer container CUDA on an older host driver will refuse to start. You can check what your host driver supports with `nvidia-smi` (look at the "CUDA Version" it reports — that's the *maximum* CUDA the driver can handle, not necessarily what's installed).

> **Gotcha — the host comes first:** If `nvidia-smi` fails *on the host*, no container will ever see the GPU. The container can only expose what the host already has working. Always confirm `nvidia-smi` runs on the host before debugging anything container-side.

### Understanding the CUDA base images

NVIDIA publishes official CUDA base images on Docker Hub (`nvidia/cuda`). They come in flavors — the tag tells you what's inside:

- **`base`** — minimal CUDA runtime. Smallest. Good when your app brings its own libraries.
- **`runtime`** — base + CUDA math libraries (cuBLAS, etc.). What most ML *inference* needs.
- **`devel`** — runtime + compilers and headers (`nvcc`). Needed if you *compile* CUDA code; bigger.

A tag like `12.3.0-runtime-ubuntu22.04` reads as: CUDA 12.3.0, the runtime flavor, on an Ubuntu 22.04 base.

> **Gotcha — image tags expire:** NVIDIA periodically removes old CUDA tags. If a build that worked last year suddenly can't pull its base image, the tag may have been retired. Check the current tag list on Docker Hub and pin to a tag that still exists.

> **Why pin a tag at all?** Never use `:latest` for a base image in real work. `latest` silently changes underneath you, so a build that worked yesterday can break today with no change on your part. Pin an explicit version so your builds are reproducible.

### How GPU passthrough actually looks

The flag is `--gpus`. A few forms you'll use:

- `--gpus all` — give the container every GPU on the host.
- `--gpus 1` — give it one GPU (any one).
- `--gpus '"device=0,2"'` — give it specific GPUs by index (0 and 2 here).

> **Gotcha — the quoting on `device=`:** When you target specific devices, the value contains a comma, so it **must** be wrapped in single quotes *around* double quotes: `'"device=0,2"'`. Get this wrong and Docker misparses it. The simple `--gpus all` and `--gpus 1` forms don't need this dance.

The GPU index numbers come from the leftmost column of `nvidia-smi` output.

### 🛠️ Hands-on Exercise 1 (Hour 1)

**Goal:** Prove your whole GPU-container stack works, end to end.

**Step 1 — Confirm the host sees the GPU.** This must succeed before anything else:
```bash
nvidia-smi
```
You should see a table of your GPU(s) and a "CUDA Version" in the top-right. If this fails, stop and fix your host NVIDIA driver first (`sudo ubuntu-drivers install`, then reboot).

**Step 2 — Install the NVIDIA Container Toolkit** (if not already installed). Follow the official install steps for your distro from the NVIDIA Container Toolkit docs, then restart Docker:
```bash
# Verify it's present
dpkg -l | grep nvidia-container
sudo systemctl restart docker
```

**Step 3 — Run `nvidia-smi` *inside* a container.** This is the moment of truth — it proves the toolkit is injecting the host driver into a container:
```bash
docker run --rm --gpus all nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi
```
If you see the same GPU table you saw on the host, **passthrough works**. 🎉

**Step 4 — Prove isolation is real.** Run the *same image without* the flag:
```bash
docker run --rm nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi
```
This should **fail** (no GPU visible). That failure is the lesson: GPU access only happens when you explicitly ask for it with `--gpus`.

---

## Hour 2 — Image Optimization

### The problem we're solving

GPU images are *huge* — a CUDA `devel` image plus PyTorch can easily exceed several gigabytes. Size isn't just an aesthetic concern; it costs you real things:

- **Slow pushes and pulls.** Every deploy, every CI run, every new machine has to move those gigabytes over the network.
- **Slow startup and scaling.** When you need to spin up a new container fast (a traffic spike, a node failure), a multi-GB image is a multi-minute wait.
- **A bigger attack surface.** Every package in the image is something that can have a vulnerability. More stuff = more risk.

So this hour is about two related goals: making images **smaller** and making them **safer**.

### Lever 1: `.dockerignore` — stop shipping junk into the build

When you run `docker build .`, Docker sends the *entire* directory (the "build context") to the daemon before building. If that directory contains your `.git` history, datasets, virtual environments, model checkpoints, or notebooks, all of it gets sucked in — slowing the build and often ending up baked into the image.

A `.dockerignore` file (same syntax as `.gitignore`) excludes these. It's the cheapest optimization you can make.

> **Gotcha — `.dockerignore` lives next to the Dockerfile, in the build context root**, not in some config folder. If it's in the wrong place it silently does nothing.

A sensible starter for a PyTorch project:
```
.git
__pycache__/
*.pyc
.venv/
venv/
data/
datasets/
*.ckpt
*.pth.tmp
.ipynb_checkpoints/
*.log
```

> **Gotcha — don't accidentally exclude what you need.** A broad pattern like `*.pth` would exclude your *trained model weights* too. Be specific about what's disposable vs. what the image actually needs at runtime.

### Lever 2: Multi-stage builds — build heavy, ship light

This is the single biggest size win for GPU images, and it directly applies the `devel` vs. `runtime` distinction from Hour 1.

The idea: use a **big `devel` image to build/compile**, then **copy only the finished artifacts into a small `runtime` image** for the final product. The compilers, headers, and build tools never make it into what you ship.

```dockerfile
# ---- Stage 1: builder (big, has compilers) ----
FROM nvidia/cuda:12.3.0-devel-ubuntu22.04 AS builder
RUN apt-get update && apt-get install -y --no-install-recommends python3-pip
COPY requirements.txt .
RUN pip3 install --no-cache-dir --prefix=/install -r requirements.txt

# ---- Stage 2: final (small, runtime only) ----
FROM nvidia/cuda:12.3.0-runtime-ubuntu22.04
COPY --from=builder /install /usr/local
COPY . /app
WORKDIR /app
CMD ["python3", "infer.py"]
```
Only the second stage becomes your image. Everything in `builder` is discarded.

### Lever 3: Smaller habits that add up

- **`--no-install-recommends`** on `apt-get install` skips optional packages you didn't ask for.
- **Clean apt caches in the same `RUN`:** `rm -rf /var/lib/apt/lists/*`.
  > **Gotcha — it must be the *same* `RUN` line.** Each `RUN` creates a new image layer, and layers are immutable. If you install in one `RUN` and clean up in the next, the files still exist in the earlier layer and your image isn't any smaller. Chain them with `&&`.
- **`pip install --no-cache-dir`** stops pip from keeping a download cache inside the image.
- **Order layers from least- to most-frequently-changed.** Copy `requirements.txt` and install deps *before* copying your code. That way changing your code doesn't invalidate the (slow) dependency-install layer, and rebuilds stay fast thanks to Docker's layer cache.

### Lever 4: Security scanning with Trivy

Smaller images are safer, but "fewer packages" isn't the same as "no known vulnerabilities." **Trivy** is a free scanner that inspects an image and reports known CVEs in its OS packages and language dependencies. The *why* here is simple: you want to find out about a critical vulnerability from a scanner on your laptop, not from an attacker in production.

Basic usage:
```bash
trivy image my-pytorch-app:latest
```
Trivy groups findings by severity (LOW/MEDIUM/HIGH/CRITICAL). In a real pipeline you'd fail the build on HIGH/CRITICAL:
```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 my-pytorch-app:latest
```

> **Gotcha — not everything reported is fixable by you.** Many CVEs come from the base image's OS packages. The realistic fixes are usually: (1) update to a newer base image tag, (2) use a slimmer base so the vulnerable package isn't there at all. Chasing individual unfixable CVEs in someone else's base image is wasted effort.

> **Gotcha — first scan is slow.** Trivy downloads a vulnerability database on first run. Don't be surprised by the initial delay; subsequent scans are fast.

### 🛠️ Hands-on Exercise 2 (Hour 2)

**Goal:** Take a build and make it smaller and scanned.

**Step 1 — Add a `.dockerignore`.** Create the file shown above in your project root.

**Step 2 — Convert a single-stage Dockerfile to multi-stage** using the template above. Build it:
```bash
docker build -t my-pytorch-app:slim .
```

**Step 3 — Compare sizes.** If you have an older single-stage build, compare:
```bash
docker images | grep my-pytorch-app
```
Note the difference between the `devel`-based and `runtime`-based images.

**Step 4 — Scan it:**
```bash
trivy image --severity HIGH,CRITICAL my-pytorch-app:slim
```
Read the report. Identify which findings come from the base OS (probably not yours to fix) vs. your own dependencies (yours to fix by bumping versions).

---

## Hour 3 — Container Registries

### The problem we're solving

So far your image only exists on your laptop. A registry is the **shared storage and distribution system for images** — it's to Docker images what GitHub is to code. You push an image once, and any authorized machine (a colleague's laptop, a CI runner, a production server, a GPU cloud instance) can pull it. Without a registry, you can't deploy anywhere but the machine you built on.

### Two registries you'll meet

- **Docker Hub** — the default public registry. When you `docker pull nginx`, it comes from here. Great for public/open-source images. Has rate limits on anonymous pulls and isn't where you'd keep private company images by default.
- **AWS ECR (Elastic Container Registry)** — AWS's private registry. This is what you use when your images are private and your workloads run on AWS. It integrates with AWS permissions (IAM), which is both its strength and the source of its one big gotcha (auth — covered below).

### How image names encode their registry

An image reference is really `registry/repository:tag`. When the registry part is omitted, Docker assumes Docker Hub. That's why `nginx:latest` and `docker.io/library/nginx:latest` are the same thing.

For ECR, the registry part is your account-and-region-specific URL:
```
<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>
```
So tagging an image for ECR means *renaming* it to include that full address.

### Tagging strategy — the part beginners get wrong

It's tempting to tag everything `:latest`. Don't rely on it. `latest` is just a label that moves; it tells you nothing about *which* version is running, and two machines pulling `latest` at different times can get different images. When something breaks in production, "it's running latest" is useless for debugging.

A practical strategy:
- **Tag with something immutable and traceable** — a git commit SHA (`:a1b2c3d`) or a version number (`:1.4.2`). This is what you deploy and what you roll back to.
- **Also** move a `:latest` (or `:stable`) tag to the newest good build *for convenience*, but never deploy *by* it.
- Consider enabling **tag immutability** on the ECR repo so a tag, once pushed, can't be overwritten — this guarantees `:1.4.2` always means the exact same image.

> **Gotcha — `:latest` is a lie in disguise.** It's not "the newest"; it's "whatever was last tagged latest." Treat it as a convenience alias, never as a source of truth.

### The ECR authentication flow (the #1 thing that trips people up)

Here's the *why*: Docker's CLI doesn't natively understand AWS IAM permissions. So you can't just point Docker at ECR and have it work. Instead, you ask AWS for a **temporary password** and feed that to `docker login`. The flow is always:

1. **Create the repository** (one time per image name):
   ```bash
   aws ecr create-repository --repository-name my-pytorch-app --region us-east-1
   ```
2. **Get a temporary token and log Docker in:**
   ```bash
   aws ecr get-login-password --region us-east-1 \
     | docker login --username AWS --password-stdin \
       <account-id>.dkr.ecr.us-east-1.amazonaws.com
   ```
3. **Tag** your local image with the full ECR address:
   ```bash
   docker tag my-pytorch-app:slim \
     <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-pytorch-app:v1
   ```
4. **Push:**
   ```bash
   docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-pytorch-app:v1
   ```

> **Gotcha — the login URL is the *registry*, not the repository.** In the `docker login` line, use `<account-id>.dkr.ecr.<region>.amazonaws.com` with **no** `/my-pytorch-app` on the end. Appending the repo name is the most common cause of the dreaded `no basic auth credentials` error on push.

> **Gotcha — the token expires after 12 hours.** That `get-login-password` token is temporary. If a push fails with auth errors hours later, you probably just need to re-run the login command. This is by design (it's more secure than a permanent password).

> **Gotcha — region must match everywhere.** The region in `get-login-password`, in the login URL, and in the repo's region must all agree. A mismatch produces confusing auth failures rather than a clear "wrong region" message.

> **Gotcha — IAM permissions.** Your AWS identity needs `ecr:GetAuthorizationToken` (to log in) plus push/pull permissions. If login itself fails, it's usually a missing permission, not a typo.

### 🛠️ Hands-on Exercise 3 (Hour 3)

**Goal:** Get your image off your laptop and into ECR.

**Step 1 — Confirm AWS CLI is configured:**
```bash
aws sts get-caller-identity
```
This shows your account ID (you'll need it) and proves your credentials work.

**Step 2 — Create the repo:**
```bash
aws ecr create-repository --repository-name my-pytorch-app --region us-east-1
```

**Step 3 — Log in** (substitute your real account ID and region):
```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.us-east-1.amazonaws.com
```
Expect `Login Succeeded`.

**Step 4 — Tag with a real version (not `latest`) and push:**
```bash
docker tag my-pytorch-app:slim \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-pytorch-app:v1

docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-pytorch-app:v1
```

**Step 5 — Verify it landed:**
```bash
aws ecr list-images --repository-name my-pytorch-app --region us-east-1
```

---

## Hour 4 — Practice: Containerize a PyTorch Model with GPU Support and Push to ECR

This hour has no new concepts — it's where the previous three hours come together into one workflow. Use one of your own PyTorch projects (or the tiny example below). The *why* of this exercise: this end-to-end loop — build for GPU, verify GPU, slim, scan, push — is the actual day-to-day workflow you'll repeat for every model you deploy.

### A minimal PyTorch app to containerize

`infer.py`:
```python
import torch

def main():
    print(f"PyTorch version: {torch.__version__}")
    print(f"CUDA available: {torch.cuda.is_available()}")
    if torch.cuda.is_available():
        print(f"GPU: {torch.cuda.get_device_name(0)}")
        x = torch.rand(1000, 1000, device="cuda")
        y = x @ x          # a real matmul on the GPU
        print(f"Result sum (on GPU): {y.sum().item():.2f}")
    else:
        print("No GPU visible to PyTorch — check your --gpus flag.")

if __name__ == "__main__":
    main()
```

`requirements.txt`:
```
torch
```
> **Gotcha — match the PyTorch build to your CUDA.** A plain `pip install torch` may pull a build expecting a particular CUDA version. If `torch.cuda.is_available()` returns `False` despite a working `--gpus all`, the usual cause is a PyTorch/CUDA mismatch — install the PyTorch build matching your base image's CUDA version (see PyTorch's install selector for the right index URL).

`Dockerfile` (multi-stage, applying Hour 2):
```dockerfile
# ---- builder ----
FROM nvidia/cuda:12.3.0-devel-ubuntu22.04 AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3-pip && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip3 install --no-cache-dir --prefix=/install -r requirements.txt

# ---- final ----
FROM nvidia/cuda:12.3.0-runtime-ubuntu22.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /install /usr/local
COPY infer.py /app/infer.py
WORKDIR /app
CMD ["python3", "infer.py"]
```

`.dockerignore`:
```
.git
__pycache__/
*.pyc
.venv/
data/
*.pth.tmp
```

### The full workflow

**1 — Build:**
```bash
docker build -t pytorch-gpu-demo:slim .
```

**2 — Run with GPU and verify PyTorch sees it** (this combines Hour 1 + your app):
```bash
docker run --rm --gpus all pytorch-gpu-demo:slim
```
Success looks like `CUDA available: True`, a GPU name, and a result sum. If it says `False`, revisit the PyTorch/CUDA gotcha above before going further.

**3 — Scan it** (Hour 2):
```bash
trivy image --severity HIGH,CRITICAL pytorch-gpu-demo:slim
```

**4 — Create repo, log in, tag, push** (Hour 3):
```bash
aws ecr create-repository --repository-name pytorch-gpu-demo --region us-east-1

aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.us-east-1.amazonaws.com

docker tag pytorch-gpu-demo:slim \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com/pytorch-gpu-demo:v1

docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/pytorch-gpu-demo:v1
```

**5 — Prove the loop closes — pull it back fresh:**
```bash
docker rmi <account-id>.dkr.ecr.us-east-1.amazonaws.com/pytorch-gpu-demo:v1
docker run --rm --gpus all \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com/pytorch-gpu-demo:v1
```
If a freshly-pulled image runs on the GPU, you've completed the entire pipeline a real ML team uses.

### 🛠️ Final challenge (optional)

Pick one and do it for real on your own project:
- Swap in an actual model + weights and run inference on a sample input inside the container.
- Compare your single-stage vs. multi-stage image size and write down the number.
- Add the Trivy scan as a step that fails on CRITICAL (`--exit-code 1`) — the start of a CI gate.

---

## 📋 Gotchas Summary Table

| # | Gotcha | Why it bites | Fix |
|---|--------|--------------|-----|
| 1 | Using `nvidia-docker` / `nvidia-docker2` / `--runtime=nvidia` | Old tutorials; deprecated | Use NVIDIA Container Toolkit + `--gpus` flag |
| 2 | `nvidia-smi` fails on the host | Container can't expose what host lacks | Fix host driver first, then containers |
| 3 | Container CUDA newer than host driver supports | Container refuses to start | Container CUDA ≤ host driver's CUDA (check `nvidia-smi`) |
| 4 | Wrong quoting on `--gpus '"device=0,2"'` | Comma misparsed | Single-quote-wrap the double-quoted value |
| 5 | Base image tag retired by NVIDIA | Old build can't pull base | Check Docker Hub for live tags; pin a current one |
| 6 | Using `:latest` for base images | Silent changes break reproducibility | Pin explicit versions |
| 7 | `.dockerignore` in wrong location | Silently does nothing | Put it in the build-context root, next to Dockerfile |
| 8 | Over-broad ignore pattern (e.g. `*.pth`) | Excludes needed weights | Be specific about disposable vs. needed files |
| 9 | Install + cleanup in separate `RUN` lines | Files persist in earlier layer; no size win | Chain in one `RUN` with `&&` |
| 10 | Treating all Trivy CVEs as your job | Many are unfixable base-OS CVEs | Update/slim the base; focus on your deps |
| 11 | Appending repo name to `docker login` URL | `no basic auth credentials` on push | Log in to the *registry* URL only |
| 12 | ECR token expired (12h) | Auth fails later | Re-run `get-login-password` login |
| 13 | Region mismatch across ECR commands | Confusing auth errors | Same region in token, URL, and repo |
| 14 | Missing IAM `ecr:GetAuthorizationToken` etc. | Login/push denied | Grant ECR push/pull + auth-token permissions |
| 15 | Deploying by `:latest` | Can't tell or roll back versions | Deploy immutable tags (SHA / version) |
| 16 | `torch.cuda.is_available()` is `False` | PyTorch/CUDA build mismatch | Install PyTorch build matching base-image CUDA |

---

## 🗂️ Quick Reference Card

**Verify GPU stack**
```bash
nvidia-smi                                   # host sees GPU?
docker run --rm --gpus all \
  nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi   # container sees GPU?
```

**GPU passthrough flags**
```bash
--gpus all                 # all GPUs
--gpus 1                   # one GPU
--gpus '"device=0,2"'      # specific GPUs (mind the quotes)
```

**CUDA base image flavors** — `base` (smallest) · `runtime` (math libs, inference) · `devel` (compilers, building)
Tag format: `12.3.0-runtime-ubuntu22.04`

**Optimize**
```bash
# .dockerignore in context root; multi-stage build:
FROM ...devel... AS builder      # build heavy
FROM ...runtime...               # ship light
COPY --from=builder /install /usr/local
# same-RUN cleanup:
RUN apt-get update && apt-get install -y --no-install-recommends pkg \
    && rm -rf /var/lib/apt/lists/*
pip install --no-cache-dir ...
```

**Scan**
```bash
trivy image my-app:tag
trivy image --severity HIGH,CRITICAL --exit-code 1 my-app:tag
```

**ECR push (full flow)**
```bash
aws sts get-caller-identity                            # who am I / account id
aws ecr create-repository --repository-name NAME --region REGION
aws ecr get-login-password --region REGION \
  | docker login --username AWS --password-stdin \
    ACCOUNT.dkr.ecr.REGION.amazonaws.com               # registry URL only!
docker tag NAME:slim ACCOUNT.dkr.ecr.REGION.amazonaws.com/NAME:v1
docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/NAME:v1
aws ecr list-images --repository-name NAME --region REGION
```

**Tagging rule** — deploy immutable tags (`:v1`, `:gitSHA`); never deploy by `:latest`.

**Golden order of operations:** host GPU works → toolkit injects it → build multi-stage → run with `--gpus all` & verify → scan → tag with version → login (registry URL) → push → pull-back test.

---

### Reference material used
- NVIDIA Container Toolkit docs & Docker "GPU access" docs (passthrough, `--gpus`, device quoting, CUDA/driver version rule)
- `nvidia/cuda` Docker Hub page (image flavors, tag lifetime)
- Trivy docs / Snyk container best practices (scanning, slimming, layer hygiene)
- AWS ECR docs (`create-repository`, `get-login-password`, login-URL-vs-repo gotcha, 12-hour token)
- PyTorch install guidance (CUDA-matched builds)
