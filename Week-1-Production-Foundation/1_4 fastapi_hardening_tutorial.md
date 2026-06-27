# FastAPI Hardening Tutorial
## Background Tasks · Auth & Middleware · Testing · Production Hardening

> **Level:** Beginner-friendly (no prior FastAPI hardening experience assumed)
> **Goal:** By the end you'll have a production-ready FastAPI service with background jobs, authentication, tests, logging, and error handling.

---

## Setup Checklist

Before you start, make sure you have:

```bash
pip install fastapi uvicorn[standard] celery redis pytest httpx python-jose[cryptography] passlib[bcrypt] slowapi
```

You'll also need **Redis** running locally (used by Celery):

```bash
# macOS
brew install redis && brew services start redis

# Ubuntu/Debian
sudo apt install redis-server && sudo systemctl start redis

# Docker (any platform)
docker run -d -p 6379:6379 redis:alpine
```

We'll build on a simple ML-serving API throughout. If you don't have one, use this starter:

```python
# main.py
from fastapi import FastAPI

app = FastAPI(title="My Serving API")

@app.get("/predict")
def predict(text: str):
    # Placeholder — swap in your real model
    return {"input": text, "prediction": "positive", "score": 0.91}
```

---

## Redis Primer — What It Is and Why Celery Needs It

If you've never used Redis before, read this before Hour 1. You don't need to become a Redis expert — you just need to understand the role it plays so the Celery setup makes sense.

### What Redis is

Redis is an in-memory key-value store — think of it as a super-fast shared notepad that multiple processes can read and write. It's not a database for your app's data; here it's acting as a **message broker**: the middleman that holds Celery tasks in a queue until a worker picks them up.

### Why Celery needs a broker

Your FastAPI app and the Celery worker are **separate processes** — they can't talk to each other directly. Redis sits in the middle so they don't have to know about each other at all:

```
Your FastAPI app          Redis                  Celery worker
      │                     │                         │
      │  "run this task"    │                         │
      │ ──────────────────► │                         │
      │                     │   "here's a task"       │
      │                     │ ───────────────────────►│
      │                     │                         │ (does the work)
      │                     │    "done, result=..."   │
      │                     │ ◄───────────────────────│
```

### The two things Redis does in this setup

In `celery_app.py` you'll see two Redis URLs that look the same:

```python
celery = Celery(
    "tasks",
    broker="redis://localhost:6379/0",   # inbox: stores pending tasks
    backend="redis://localhost:6379/0",  # outbox: stores finished results
)
```

- **Broker** — where FastAPI *sends* tasks ("please run this")
- **Backend** — where workers *store results* so you can retrieve them later with `AsyncResult(task_id)`

They can point to the same Redis instance (fine for local dev) or separate ones (common in production).

### The `/0` at the end

Redis has numbered databases (0–15) within a single instance — like tabs in a spreadsheet. `/0` just means "use database 0." You rarely need to change this.

### Checking Redis is actually running

Before starting your app or Celery worker, confirm Redis is up:

```bash
redis-cli ping
# Should print: PONG
```

If you get `Connection refused`, Redis isn't running — go back to the install commands in the Setup Checklist above for your platform.

### The one thing you don't need to configure

Redis manages the queue automatically. You never write any Redis code directly — Celery handles all of that. Your only job is to make sure Redis is running and the connection URL is correct.

---

---

# Hour 1 — Background Tasks & Queues

## Why This Matters

Imagine your `/predict` endpoint takes 10 seconds to run. If 50 users hit it simultaneously, they all wait. Worse: if it times out, the job is lost entirely.

**Background tasks** solve this by decoupling *accepting* a request from *doing the work*. You respond immediately ("job queued!") and do the heavy lifting separately.

There are two levels of solution:

| Tool | Best For | Persistence |
|---|---|---|
| `BackgroundTasks` (built into FastAPI) | Short, lightweight tasks (send an email, write a log) | No — lost if the process dies |
| Celery | Long jobs, retries, scheduling, multiple workers | Yes — tasks survive restarts |

---

## 1.1 FastAPI `BackgroundTasks`

FastAPI's `BackgroundTasks` runs a function *after* the response is sent, inside the same process.

### Why the "after the response" part matters

Without background tasks, your endpoint can't return until all its code finishes. `BackgroundTasks` lets FastAPI send the HTTP response first, then run your function — so the caller isn't waiting.

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_log(message: str):
    # Simulates slow work (e.g., writing to a remote logging service)
    with open("jobs.log", "a") as f:
        f.write(message + "\n")

@app.post("/predict-async")
def predict_async(text: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(write_log, f"Processed: {text}")
    return {"status": "accepted", "message": "Job is running in background"}
```

> ⚠️ **Gotcha:** `BackgroundTasks` runs in the *same process and event loop* as your app. If your background function crashes or takes too long, it can affect other requests. Use it only for genuinely lightweight work (logging, sending a notification). For anything CPU-intensive or long-running, use Celery.

---

## 1.2 Celery — The Full Queue

Celery is a distributed task queue. Your FastAPI app *publishes* tasks to a **broker** (Redis), and one or more **worker** processes pick them up and run them independently.

```
FastAPI app  →  Redis (broker)  →  Celery worker
   (sends)         (stores)           (runs task)
```

### Setting up Celery

```python
# celery_app.py
from celery import Celery

celery = Celery(
    "tasks",
    broker="redis://localhost:6379/0",   # where tasks are sent
    backend="redis://localhost:6379/0",  # where results are stored
)

celery.conf.update(task_track_started=True)
```

### Defining a task

```python
# tasks.py
from celery_app import celery
import time

@celery.task(bind=True)
def run_heavy_prediction(self, text: str):
    # 'self' gives you access to task metadata (id, retry, etc.)
    time.sleep(5)  # Simulates a slow model
    return {"input": text, "prediction": "positive", "score": 0.91}
```

> ⚠️ **Gotcha:** The `bind=True` argument gives you `self` so you can call `self.retry()` on failure or check `self.request.id`. Without it you lose access to task context.

### Calling the task from FastAPI

```python
# main.py (updated)
from fastapi import FastAPI
from tasks import run_heavy_prediction

app = FastAPI()

@app.post("/predict-celery")
def predict_celery(text: str):
    task = run_heavy_prediction.delay(text)   # .delay() sends to queue immediately
    return {"task_id": task.id, "status": "queued"}

@app.get("/result/{task_id}")
def get_result(task_id: str):
    from celery_app import celery
    task = celery.AsyncResult(task_id)
    return {"task_id": task_id, "status": task.status, "result": task.result}
```

### Running the worker

Open a *second terminal* and run:

```bash
celery -A celery_app worker --loglevel=info
```

Now your FastAPI app and the Celery worker are separate processes. The API stays responsive even if a job takes minutes.

> ⚠️ **Gotcha:** Never import your FastAPI `app` object inside a Celery task. It creates circular imports and can accidentally spin up extra app instances in your worker process.

---

## ✏️ Hour 1 Exercise

**Goal:** Wire up a Celery task to your serving API.

1. Create `celery_app.py` and `tasks.py` as shown above.
2. Add a `/predict-celery` endpoint to your `main.py`.
3. Start the Celery worker in a second terminal.
4. Send a POST to `/predict-celery?text=hello` using `curl` or the FastAPI docs UI (`/docs`).
5. Copy the `task_id` from the response and hit `/result/{task_id}` — first immediately (you should see `PENDING`), then after 5 seconds (you should see `SUCCESS` with the result).

**Stretch:** Add a `@celery.task(bind=True, max_retries=3)` decorator and call `self.retry(exc=e, countdown=2)` inside a try/except — this makes the task retry up to 3 times on failure, waiting 2 seconds between attempts.

---

---

# Hour 2 — Auth & Middleware

## Why This Matters

An unprotected API is a liability. Anyone who finds your URL can call your (potentially expensive) model, exhaust your resources, or extract your data. Authentication and middleware are your first lines of defense.

---

## 2.1 API Key Authentication

The simplest form of auth: the caller sends a secret key, you check it's valid.

### The problem with putting the key in the URL

`/predict?api_key=secret123` — this gets logged in server logs, browser history, and load balancer access logs. **Always put secrets in a header.**

```python
from fastapi import FastAPI, HTTPException, Security
from fastapi.security import APIKeyHeader

app = FastAPI()

API_KEY = "my-secret-key-123"  # In production: load from environment variable
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def verify_api_key(key: str = Security(api_key_header)):
    if key != API_KEY:
        raise HTTPException(status_code=403, detail="Invalid or missing API key")
    return key

@app.get("/predict", dependencies=[Depends(verify_api_key)])
def predict(text: str):
    return {"prediction": "positive"}
```

> ⚠️ **Gotcha:** `auto_error=False` tells FastAPI *not* to automatically return a 403 when the header is missing — it passes `None` to your function instead, letting you write a custom error message. Without it you get a generic, less helpful error.

---

## 2.2 JWT Authentication

JWT (JSON Web Token) is more sophisticated: users log in and receive a *signed token* they include in all future requests. You verify the signature — no database lookup needed.

A JWT has three parts: `header.payload.signature`. The signature is what you verify.

```python
from datetime import datetime, timedelta
from jose import jwt, JWTError
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

SECRET_KEY = "your-very-secret-key"   # Use a long random string in production
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return username
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/token")
def login(username: str, password: str):
    # In production: verify username/password against your database
    if username == "admin" and password == "secret":
        token = create_access_token({"sub": username})
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Bad credentials")

@app.get("/protected-predict")
def protected_predict(text: str, user: str = Depends(get_current_user)):
    return {"prediction": "positive", "requested_by": user}
```

> ⚠️ **Gotcha:** JWTs are *signed*, not *encrypted*. Anyone can base64-decode the payload and read its contents. Never put passwords, credit card numbers, or other sensitive data inside a JWT.

---

## 2.3 CORS Middleware

CORS (Cross-Origin Resource Sharing) controls which domains can call your API from a browser. Without it, a JavaScript app on `https://myapp.com` can't call your API on `https://api.myapp.com`.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],   # Never use ["*"] in production
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)
```

> ⚠️ **Gotcha:** `allow_origins=["*"]` with `allow_credentials=True` is an invalid combination that browsers will reject. You must list specific origins if you need credentials (cookies/auth headers).

---

## 2.4 Rate Limiting

Rate limiting prevents any single client from flooding your API.

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/predict")
@limiter.limit("10/minute")   # Max 10 requests per minute per IP
def predict(request: Request, text: str):
    return {"prediction": "positive"}
```

> ⚠️ **Gotcha:** `slowapi`'s `@limiter.limit` decorator requires the endpoint to accept a `Request` parameter — even if you don't use it. Forgetting this causes a runtime error.

---

## ✏️ Hour 2 Exercise

**Goal:** Protect your serving API.

1. Add the API key middleware to your `/predict` endpoint.
2. Test that calling without the header returns a `403`.
3. Test that calling *with* `X-API-Key: my-secret-key-123` returns a `200`.
4. Add CORS middleware — allow only `http://localhost:3000` (simulating a local frontend).
5. Add rate limiting of `5/minute` to `/predict`.

**Stretch:** Implement the `/token` + `/protected-predict` JWT flow. Get a token from `/token`, then use it in the `Authorization: Bearer <token>` header to call `/protected-predict`.

---

---

# Hour 3 — Testing

## Why This Matters

Without tests, every change to your API is a gamble. A test suite lets you refactor, add features, or upgrade dependencies with confidence. FastAPI makes testing unusually easy because it ships with a built-in test client.

---

## 3.1 The TestClient

FastAPI's `TestClient` wraps your app in a way that lets you make real HTTP requests *without* starting a server. It's synchronous (even if your app uses `async`), which makes tests simple to write.

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app   # import your FastAPI app

client = TestClient(app)

def test_predict_returns_200():
    response = client.get("/predict", params={"text": "hello"})
    assert response.status_code == 200

def test_predict_has_prediction_key():
    response = client.get("/predict", params={"text": "hello"})
    data = response.json()
    assert "prediction" in data
```

Run with:

```bash
pytest test_main.py -v
```

> ⚠️ **Gotcha:** Import `app` from your module, not `from fastapi import FastAPI`. If you accidentally create a *new* FastAPI instance in your test file, your routes won't be registered on it and all tests will return 404.

---

## 3.2 Testing Authenticated Endpoints

For API key auth, just add the header:

```python
def test_predict_with_valid_key():
    response = client.get(
        "/predict",
        params={"text": "hello"},
        headers={"X-API-Key": "my-secret-key-123"}
    )
    assert response.status_code == 200

def test_predict_with_invalid_key():
    response = client.get(
        "/predict",
        params={"text": "hello"},
        headers={"X-API-Key": "wrong-key"}
    )
    assert response.status_code == 403
```

---

## 3.3 Mocking — Testing Without Side Effects

Your app probably calls external services (a database, a model, an email API). Tests shouldn't hit real services — they should be fast, offline, and deterministic.

`unittest.mock.patch` replaces a real object with a fake one for the duration of a test.

```python
from unittest.mock import patch, MagicMock

def test_celery_task_is_called():
    # We don't want to actually run the Celery task in tests
    with patch("main.run_heavy_prediction") as mock_task:
        mock_task.delay.return_value = MagicMock(id="fake-task-id-123")
        
        response = client.post("/predict-celery", params={"text": "hello"})
        
        assert response.status_code == 200
        assert response.json()["task_id"] == "fake-task-id-123"
        mock_task.delay.assert_called_once_with("hello")
```

> ⚠️ **Gotcha:** `patch("main.run_heavy_prediction")` patches the name *where it's used* (in `main.py`), not where it's defined (in `tasks.py`). This is one of the most common mocking mistakes. The rule: **patch the name in the namespace that uses it.**

---

## 3.4 Fixtures and Shared Setup

pytest **fixtures** let you define reusable setup code. The `client` fixture below creates a fresh `TestClient` for every test function.

```python
# conftest.py  ← pytest automatically loads this file
import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def auth_headers():
    return {"X-API-Key": "my-secret-key-123"}
```

Now your tests become cleaner:

```python
# test_main.py
def test_predict(client, auth_headers):
    response = client.get("/predict", params={"text": "hi"}, headers=auth_headers)
    assert response.status_code == 200
```

---

## 3.5 Test Coverage

Coverage tells you *which lines of your code are never executed by tests* — those are your blind spots.

```bash
pip install pytest-cov

pytest --cov=main --cov-report=term-missing
```

The `term-missing` option prints which line numbers aren't covered. Aim for >80% on critical paths (auth, error handling).

> ⚠️ **Gotcha:** 100% coverage doesn't mean your code is correct — it just means every line ran. A test that calls a function but doesn't assert anything about the output contributes to coverage but catches nothing.

---

## ✏️ Hour 3 Exercise

**Goal:** Write a test suite for your API.

1. Create `conftest.py` with `client` and `auth_headers` fixtures.
2. Write tests for:
   - `GET /predict` without auth → expect `403`
   - `GET /predict` with valid auth → expect `200` and a `prediction` key in the response
   - `POST /predict-celery` with auth → mock `run_heavy_prediction.delay` and assert it was called with the right argument
3. Run `pytest -v` and confirm all tests pass.
4. Run `pytest --cov=main --cov-report=term-missing` and check your coverage percentage.

**Stretch:** Write a test for the rate limiter — send 6 requests in a loop and assert the 6th returns `429 Too Many Requests`.

---

---

# Hour 4 — Harden the API

## Why This Matters

Auth and tests get you 60% of the way. The remaining 40% is what separates a toy API from something you'd trust in production: **structured logging** so you can debug issues, **global error handlers** so clients always get a useful response, and **wiring everything together** consistently.

---

## 4.1 Structured Logging

`print()` statements are not logging. They have no severity level, no timestamps, and can't be filtered or aggregated.

Python's `logging` module gives you levels (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`), timestamps, and the ability to route logs to files, external services, or stdout depending on the environment.

```python
import logging
import sys

# Configure once, at startup
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    handlers=[
        logging.StreamHandler(sys.stdout),           # Always log to stdout
        logging.FileHandler("api.log", mode="a"),    # Also log to file
    ]
)

logger = logging.getLogger(__name__)
```

Use it in your endpoints:

```python
@app.get("/predict")
def predict(text: str, api_key: str = Depends(verify_api_key)):
    logger.info(f"Prediction request received | text_length={len(text)}")
    result = {"prediction": "positive", "score": 0.91}
    logger.info(f"Prediction complete | result={result['prediction']}")
    return result
```

> ⚠️ **Gotcha:** Call `logging.basicConfig(...)` exactly *once*, before any other logging calls. If you call it multiple times (e.g., once in `main.py` and once in `celery_app.py`), only the first call takes effect. Use a dedicated `logging_config.py` module and import it everywhere.

---

## 4.2 Request Logging Middleware

Rather than adding `logger.info(...)` to every endpoint, write a middleware that logs *every* request and response automatically.

```python
import time
from fastapi import Request

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    logger.info(f"Request started | method={request.method} path={request.url.path}")
    
    response = await call_next(request)  # Run the actual endpoint
    
    duration = round((time.time() - start) * 1000, 2)
    logger.info(
        f"Request complete | method={request.method} path={request.url.path} "
        f"status={response.status_code} duration_ms={duration}"
    )
    return response
```

This gives you a log line for every request without touching any endpoint code.

---

## 4.3 Global Exception Handlers

By default, unhandled exceptions return a `500 Internal Server Error` with a generic message. Your users can't do anything with that, and you lose context about what went wrong.

Global exception handlers let you:
- Return a consistent, structured error format to callers
- Log the full traceback for debugging
- Handle specific error types differently

```python
from fastapi import Request
from fastapi.responses import JSONResponse

# Handle unexpected errors (catches everything not handled below)
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(
        f"Unhandled exception | path={request.url.path} error={str(exc)}",
        exc_info=True   # This logs the full traceback
    )
    return JSONResponse(
        status_code=500,
        content={"detail": "An internal error occurred. Please try again later."}
    )

# Handle your own custom errors more gracefully
class ModelError(Exception):
    def __init__(self, message: str):
        self.message = message

@app.exception_handler(ModelError)
async def model_error_handler(request: Request, exc: ModelError):
    logger.warning(f"Model error | path={request.url.path} message={exc.message}")
    return JSONResponse(
        status_code=422,
        content={"detail": f"Model error: {exc.message}"}
    )
```

> ⚠️ **Gotcha:** `exc_info=True` in `logger.error(...)` is what includes the full stack trace in your log output. Without it, you only get the error message — which is often not enough to diagnose the problem.

---

## 4.4 Putting It All Together

Here's a complete, hardened `main.py` combining everything from all four hours:

```python
# main.py — hardened version
import logging
import sys
import time
from datetime import datetime, timedelta

from fastapi import Depends, FastAPI, HTTPException, Request, Security
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from fastapi.security import APIKeyHeader, OAuth2PasswordBearer
from jose import JWTError, jwt
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address

from tasks import run_heavy_prediction

# ── Logging ──────────────────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    handlers=[logging.StreamHandler(sys.stdout), logging.FileHandler("api.log")],
)
logger = logging.getLogger(__name__)

# ── App & middleware ──────────────────────────────────────────────────────────
limiter = Limiter(key_func=get_remote_address)
app = FastAPI(title="Hardened Serving API")
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

# ── Auth ──────────────────────────────────────────────────────────────────────
API_KEY = "my-secret-key-123"
SECRET_KEY = "jwt-secret-change-in-production"
ALGORITHM = "HS256"

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def verify_api_key(key: str = Security(api_key_header)):
    if key != API_KEY:
        raise HTTPException(status_code=403, detail="Invalid or missing API key")
    return key

def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user = payload.get("sub")
        if not user:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# ── Request logging middleware ────────────────────────────────────────────────
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    logger.info(f"→ {request.method} {request.url.path}")
    response = await call_next(request)
    ms = round((time.time() - start) * 1000, 2)
    logger.info(f"← {response.status_code} {request.url.path} ({ms}ms)")
    return response

# ── Global error handlers ─────────────────────────────────────────────────────
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled error on {request.url.path}", exc_info=True)
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})

# ── Endpoints ─────────────────────────────────────────────────────────────────
@app.get("/predict")
@limiter.limit("10/minute")
def predict(request: Request, text: str, _key: str = Depends(verify_api_key)):
    logger.info(f"Prediction | text_length={len(text)}")
    return {"input": text, "prediction": "positive", "score": 0.91}

@app.post("/predict-celery")
def predict_celery(text: str, _key: str = Depends(verify_api_key)):
    task = run_heavy_prediction.delay(text)
    logger.info(f"Task queued | task_id={task.id}")
    return {"task_id": task.id, "status": "queued"}

@app.get("/result/{task_id}")
def get_result(task_id: str, _key: str = Depends(verify_api_key)):
    from celery_app import celery
    task = celery.AsyncResult(task_id)
    return {"task_id": task_id, "status": task.status, "result": task.result}

@app.post("/token")
def login(username: str, password: str):
    if username == "admin" and password == "secret":
        expire = datetime.utcnow() + timedelta(minutes=30)
        token = jwt.encode({"sub": username, "exp": expire}, SECRET_KEY, ALGORITHM)
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Bad credentials")
```

---

## ✏️ Hour 4 Exercise

**Goal:** Fully harden your serving API.

1. Replace your `main.py` with the hardened version above (or apply each section to your existing file).
2. Start your app and make a few requests. Open `api.log` and verify you see timestamped log lines for each request.
3. Deliberately cause an error: add `raise Exception("test error")` to `/predict`, call it, and confirm:
   - The caller gets `{"detail": "Internal server error"}` (not a stack trace)
   - `api.log` contains the full traceback
4. Remove the `raise` line, run your full test suite (`pytest -v`) — everything should still pass.

**Stretch:** Add a `GET /health` endpoint that returns `{"status": "ok", "timestamp": ...}` — with *no* auth required and *not* rate-limited. Add a test for it.

---

---

# Gotchas Summary Table

| # | Gotcha | Where It Bites | Fix |
|---|---|---|---|
| 1 | `BackgroundTasks` runs in-process | Hr 1 | Use Celery for anything CPU-intensive or long |
| 2 | Importing `app` in Celery tasks | Hr 1 | Keep Celery tasks independent of the FastAPI app object |
| 3 | `bind=True` missing on Celery task | Hr 1 | Add it if you need `self.retry()` or task metadata |
| 4 | API key in URL query param | Hr 2 | Always use a request header (`X-API-Key`) |
| 5 | `auto_error=False` missing on `APIKeyHeader` | Hr 2 | Add it to control your own error message |
| 6 | JWTs are signed, not encrypted | Hr 2 | Never put sensitive data in the JWT payload |
| 7 | `allow_origins=["*"]` + `allow_credentials=True` | Hr 2 | List specific origins when credentials are needed |
| 8 | `Request` param missing on rate-limited endpoint | Hr 2 | `slowapi` requires it — add even if unused |
| 9 | Importing a new `app` instance in test file | Hr 3 | Import the same `app` object from `main.py` |
| 10 | Patching the wrong namespace | Hr 3 | Patch where the name is *used*, not where it's defined |
| 11 | 100% coverage ≠ correct code | Hr 3 | Write assertions, not just calls |
| 12 | `logging.basicConfig()` called multiple times | Hr 4 | Configure logging once in one place |
| 13 | `exc_info=True` missing from `logger.error()` | Hr 4 | Add it or you lose the stack trace |

---

---

# Quick Reference Card

## Commands

```bash
# Start the API
uvicorn main:app --reload

# Start Celery worker
celery -A celery_app worker --loglevel=info

# Run tests
pytest -v

# Run tests with coverage
pytest --cov=main --cov-report=term-missing
```

## Key Patterns

```python
# Background task (lightweight)
background_tasks.add_task(my_function, arg1, arg2)

# Celery task (heavy/persistent)
task = my_task.delay(arg1)          # send
result = celery.AsyncResult(task.id) # check

# API key dependency
Depends(verify_api_key)

# JWT dependency
Depends(get_current_user)

# Rate limit decorator
@limiter.limit("10/minute")  # requires Request param in endpoint

# Log with traceback
logger.error("message", exc_info=True)

# Mock in tests
with patch("main.my_function") as mock:
    mock.return_value = "fake"
```

## HTTP Status Codes Used

| Code | Meaning | When |
|---|---|---|
| 200 | OK | Successful request |
| 401 | Unauthorized | Missing/invalid JWT |
| 403 | Forbidden | Invalid API key |
| 422 | Unprocessable Entity | Validation / model error |
| 429 | Too Many Requests | Rate limit hit |
| 500 | Internal Server Error | Unhandled exception |

## File Structure

```
project/
├── main.py           # FastAPI app, routes, middleware, auth
├── celery_app.py     # Celery configuration
├── tasks.py          # Celery task definitions
├── conftest.py       # pytest fixtures
├── test_main.py      # Test suite
└── api.log           # Log output (auto-created)
```
