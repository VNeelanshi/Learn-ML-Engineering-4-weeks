# FastAPI in 4 Hours — ML Model Serving Tutorial

> **How to use this guide:** Each hour is self-contained. Read the explanation, then do the hands-on exercise before moving on. By the end of Hour 4 you'll have a production-ready API serving your containerized ML model with `/predict` and `/health` endpoints.
>
> **Prerequisites:** Basic Python, familiarity with functions and classes. Hour 4 assumes you have a saved sklearn or PyTorch model (a `.pkl` or `.pt` file). If not, the exercise includes a fallback.
>
> **Setup:** `pip install fastapi uvicorn pydantic` before starting Hour 1.

---

# Hour 1 — FastAPI Fundamentals

## Why FastAPI (and Not Flask)?

FastAPI is a Python web framework — a library that lets you define URLs your app responds to, handle incoming data, and send back responses. Compared to older Python web frameworks, it solves three things out of the box. If you've seen Flask before, FastAPI will feel familiar — but it solves three things Flask doesn't:

1. **Automatic validation.** You declare what shape your data should be, FastAPI rejects bad input before it touches your code. No manual `if request.json.get(...)` checks.
2. **Automatic docs.** FastAPI generates interactive API documentation at `/docs` the moment you run the server. No extra work.
3. **Built for async.** FastAPI is built on async Python from the ground up — critical for ML serving where you want to handle multiple requests without blocking (covered in Hour 2).

For ML serving specifically, FastAPI has become the industry default. It's what powers production inference endpoints at companies using Python ML stacks.

---

## Your First FastAPI App

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello from FastAPI"}
```

Run it:
```bash
uvicorn main:app --reload
```

- `main` = the filename (`main.py`)
- `app` = the FastAPI instance inside it
- `--reload` = auto-restart when you save changes (development only)

Open `http://localhost:8000` — you get your JSON response.  
Open `http://localhost:8000/docs` — you get a fully interactive API explorer. Free, automatic.

> **What is uvicorn?** FastAPI is a web *framework* — it defines routes and logic. Uvicorn is the *server* that actually listens for HTTP connections and hands them to FastAPI. You always need both. Think of it like FastAPI is your Python app and uvicorn is the socket listener in front of it.

---

## Routes and HTTP Methods

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items")           # GET — read data
def list_items():
    return [{"id": 1}, {"id": 2}]

@app.post("/items")          # POST — create / send data
def create_item():
    return {"created": True}

@app.get("/items/{item_id}") # Path parameter — part of the URL itself
def get_item(item_id: int):  # FastAPI converts the string "42" to int automatically
    return {"item_id": item_id}
```

The type annotation on `item_id: int` does real work — it's not just documentation. FastAPI uses it to validate and coerce the value. Send `GET /items/abc` and you get a 422 validation error back automatically.

---

## Path Parameters vs Query Parameters

```python
@app.get("/models/{model_name}")
def get_model(
    model_name: str,        # path param — comes from /models/resnet
    version: int = 1,       # query param — comes from ?version=2 (default: 1)
    verbose: bool = False   # query param — ?verbose=true
):
    return {
        "model": model_name,
        "version": version,
        "verbose": verbose
    }
```

- **Path parameters** (`{model_name}`) are part of the URL path. Required — the route won't match without them.
- **Query parameters** (`?version=2`) come after the `?`. Optional if they have a default value.

Call: `GET /models/iris?version=3&verbose=true`  
Response: `{"model": "iris", "version": 3, "verbose": true}`

FastAPI parsed and typed all of that for you.

---

## Pydantic Models — The Heart of FastAPI

Pydantic is the validation library FastAPI is built on. You define the *shape* of your data as a class, and FastAPI enforces it automatically on incoming requests.

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()

class PredictionRequest(BaseModel):
    sepal_length: float = Field(..., gt=0, description="Sepal length in cm")
    sepal_width: float  = Field(..., gt=0, description="Sepal width in cm")
    petal_length: float = Field(..., gt=0, description="Petal length in cm")
    petal_width: float  = Field(..., gt=0, description="Petal width in cm")

class PredictionResponse(BaseModel):
    prediction: str
    confidence: float
    model_version: str = "1.0"

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    # request.sepal_length is already a validated float
    # if it was missing or wrong type, FastAPI already rejected the request
    return PredictionResponse(
        prediction="setosa",
        confidence=0.97
    )
```

What Pydantic gives you for free:
- Missing fields → 422 error with a clear message
- Wrong types → 422 error with a clear message
- `gt=0` constraint violated → 422 error with a clear message
- `response_model=PredictionResponse` → output is also validated and serialized

> **Gotcha:** `Field(...)` — the `...` means the field is **required** (no default). It's not a typo. `Field(1.0)` would make it optional with a default of 1.0. This is a common source of confusion when first reading FastAPI code.

---

## Response Models and Status Codes

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    value: float

@app.get("/items/{item_id}", response_model=Item, status_code=200)
def get_item(item_id: int):
    if item_id == 0:
        raise HTTPException(status_code=404, detail="Item not found")
    return Item(name="widget", value=9.99)
```

`HTTPException` is how you return error responses. FastAPI turns it into the right HTTP status code and a JSON body automatically.

---

## 🛠️ Hands-On Exercise 1 — Build a Validated Prediction API Skeleton

**What you'll do:** Build the route structure for your ML API — no model yet, just the shapes, validation, and docs.

**Step 1: Create project folder**
```bash
mkdir fastapi-ml && cd fastapi-ml
```

**Step 2: Create `main.py`**

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI(
    title="ML Model API",
    description="Serving ML predictions via FastAPI",
    version="1.0.0"
)

# --- Request / Response models ---

class PredictionRequest(BaseModel):
    sepal_length: float = Field(..., gt=0, le=20, description="Sepal length in cm")
    sepal_width:  float = Field(..., gt=0, le=20, description="Sepal width in cm")
    petal_length: float = Field(..., gt=0, le=20, description="Petal length in cm")
    petal_width:  float = Field(..., gt=0, le=20, description="Petal width in cm")

    model_config = {
        "json_schema_extra": {
            "examples": [
                {"sepal_length": 5.1, "sepal_width": 3.5,
                 "petal_length": 1.4, "petal_width": 0.2}
            ]
        }
    }

class PredictionResponse(BaseModel):
    prediction: str
    confidence: float
    model_version: str

class HealthResponse(BaseModel):
    status: str
    model_loaded: bool

# --- Routes ---

@app.get("/health", response_model=HealthResponse)
def health():
    return HealthResponse(status="ok", model_loaded=False)

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    # Stub — we'll load a real model in Hour 3
    return PredictionResponse(
        prediction="setosa",
        confidence=0.99,
        model_version="stub-1.0"
    )
```

**Step 3: Run it**
```bash
uvicorn main:app --reload
```

**Step 4: Explore**

- Visit `http://localhost:8000/docs`
- Click `POST /predict` → "Try it out" → the example is pre-filled → Execute
- Try sending a request with `sepal_length: -1` — watch the validation error
- Try sending a request with `sepal_length: "banana"` — same
- Visit `http://localhost:8000/health`

**✅ Checkpoint:** You have a running API with automatic docs, type validation, and proper error responses — and you've written almost no boilerplate to get there.

---

# Hour 2 — Async Patterns

## Why Async Matters for ML Serving

Imagine your API gets two requests at the same time. Request A needs to query a database (fast I/O). Request B needs to run model inference (slow compute). 

In a synchronous server, Request B blocks the entire server while it's running — Request A waits in line even though it only needs a quick database call. In an async server, while Request B is doing compute, the server can handle other requests in between.

More precisely, async helps when your code is **waiting on I/O**: network calls, database queries, reading files. For pure CPU computation (like numpy operations), async alone doesn't help — that's what thread pools are for (covered below).

---

## async/await Basics

```python
import asyncio

# Synchronous — blocks while sleeping
def slow_function():
    import time
    time.sleep(2)          # nothing else can run during these 2 seconds
    return "done"

# Asynchronous — yields control while waiting
async def async_function():
    await asyncio.sleep(2) # other coroutines can run during these 2 seconds
    return "done"
```

The `async` keyword says "this function can be paused." The `await` keyword says "pause here and let other things run until this resolves."

---

## Sync vs Async Endpoints in FastAPI

FastAPI supports both. The rule of thumb:

```python
from fastapi import FastAPI
import asyncio
import httpx

app = FastAPI()

# Use async def when your function does I/O:
# — calls an external API
# — queries a database with an async driver
# — reads/writes files with aiofiles
@app.get("/external-data")
async def get_external_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()

# Use def (sync) for CPU-bound work — FastAPI runs it in a thread pool
# automatically so it doesn't block the event loop
@app.post("/predict")
def predict(request: dict):
    result = run_heavy_model_inference(request)  # CPU-bound numpy/torch ops
    return result
```

> **The most important gotcha in this hour:** Don't use `async def` for CPU-bound work (numpy, PyTorch inference) and then run it directly — you'll block the async event loop and make performance *worse*, not better. Use regular `def` for CPU work; FastAPI automatically runs sync endpoints in a thread pool. Only use `async def` when you're doing real I/O with an async library.

---

## The Event Loop Mental Model

```
Event Loop (single thread)
│
├── Request A arrives → starts async DB query → yields (awaits)
│   └── while waiting for DB...
├── Request B arrives → starts processing → awaits external API call → yields
│   └── while waiting for API...
├── Request A's DB query returns → resumes → sends response
├── Request B's API call returns → resumes → sends response
└── (all of this happens concurrently in ONE thread)
```

No threads needed for I/O concurrency — that's the efficiency gain. But CPU work doesn't yield, so:

```
Event Loop
├── Request A arrives → starts numpy matrix multiply → BLOCKS
│   └── Request B is stuck waiting even though it just needs a DB call
```

That's why CPU work goes in thread pools.

---

## Running CPU-Bound Work Without Blocking

For ML inference (which is CPU/GPU bound), use FastAPI's built-in thread pool via `run_in_executor`:

```python
from fastapi import FastAPI
from concurrent.futures import ThreadPoolExecutor
import asyncio

app = FastAPI()
executor = ThreadPoolExecutor(max_workers=4)

def heavy_inference(features):
    # This runs in a thread, not the event loop
    import time
    time.sleep(2)  # simulate slow model inference
    return {"prediction": "setosa"}

@app.post("/predict")
async def predict(request: dict):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, heavy_inference, request)
    return result
```

In practice for simple ML APIs, a plain `def` endpoint (not `async def`) is often the right call — FastAPI handles the thread pool for you automatically.

---

## Background Tasks

FastAPI has a built-in `BackgroundTasks` for work you want to do *after* returning a response — like logging, sending a notification, or updating a counter:

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def log_prediction(input_data: dict, prediction: str):
    # Runs after the response is sent — doesn't slow down the API
    with open("predictions.log", "a") as f:
        f.write(f"{input_data} -> {prediction}\n")

@app.post("/predict")
def predict(request: dict, background_tasks: BackgroundTasks):
    prediction = "setosa"
    background_tasks.add_task(log_prediction, request, prediction)
    return {"prediction": prediction}  # returned immediately; logging happens after
```

---

## 🛠️ Hands-On Exercise 2 — Async Health Check with External Call

**What you'll do:** Add an async endpoint that calls an external service, and observe the difference between blocking and non-blocking behavior.

**Step 1: Install httpx** (async HTTP client)
```bash
pip install httpx
```

**Step 2: Update `main.py`** — add these imports and endpoints

```python
import asyncio
import httpx
from fastapi import BackgroundTasks
import time

# Add to your existing app...

# Async endpoint — correct use of async (I/O bound)
@app.get("/health/deep")
async def deep_health():
    """Checks our own /health endpoint — simulates checking a downstream service."""
    async with httpx.AsyncClient() as client:
        response = await client.get("http://localhost:8000/health")
    return {
        "self_check": response.json(),
        "latency_ms": response.elapsed.total_seconds() * 1000
    }

# Sync endpoint — correct for CPU-bound (FastAPI runs in thread pool)
@app.post("/predict/slow")
def predict_slow(request: PredictionRequest, background_tasks: BackgroundTasks):
    """Simulates slow model inference — uses sync def intentionally."""
    time.sleep(1)  # simulate inference
    result = {"prediction": "setosa", "confidence": 0.97}
    
    # Log in background after response is sent
    background_tasks.add_task(
        lambda: print(f"[LOG] Prediction made: {result}")
    )
    return result
```

**Step 3: Test it**
```bash
# In terminal 1: keep uvicorn running
# In terminal 2:
curl http://localhost:8000/health/deep
curl -X POST http://localhost:8000/predict/slow \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

Watch the server terminal — the background task log line prints *after* the response is returned.

**✅ Checkpoint:** You understand when to use `async def` vs `def`, you've used `BackgroundTasks`, and you know why using `async def` for numpy work is actually wrong.

---

# Hour 3 — Model Serving Patterns

## The Core Problem: Where Does the Model Live?

Naive approach — load the model inside the route handler:

```python
@app.post("/predict")
def predict(request: PredictionRequest):
    model = pickle.load(open("model.pkl", "rb"))  # ❌ loads from disk on EVERY request
    return model.predict(...)
```

This is a disaster. Loading a model from disk takes hundreds of milliseconds to seconds. At 100 requests/second, you're reloading the model 100 times per second. The model needs to be loaded **once** at startup and reused for every request.

---

## Loading Models at Startup with Lifespan

FastAPI's `lifespan` context manager is the modern way to run setup/teardown code:

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import pickle

# Global model store
models = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # === STARTUP: runs before the first request ===
    print("Loading model...")
    with open("model/iris_model.pkl", "rb") as f:
        models["iris"] = pickle.load(f)
    print("Model loaded.")
    
    yield  # app runs here (handles requests)
    
    # === SHUTDOWN: runs when server stops ===
    models.clear()
    print("Model unloaded.")

app = FastAPI(lifespan=lifespan)

@app.post("/predict")
def predict(request: PredictionRequest):
    model = models["iris"]  # ✅ already in memory, instant
    features = [[
        request.sepal_length, request.sepal_width,
        request.petal_length, request.petal_width
    ]]
    prediction = model.predict(features)[0]
    return {"prediction": prediction}
```

> **Gotcha:** The old way was `@app.on_event("startup")` — you'll see this in older tutorials and StackOverflow answers. It still works but is deprecated. Use `lifespan` for new projects.

---

## Dependency Injection

Dependency Injection (DI) sounds complex but the FastAPI version is simple: it's a way to share state (like a loaded model) across routes without using ugly global variables.

```python
from fastapi import FastAPI, Depends
from contextlib import asynccontextmanager
import pickle

# Model container class — cleaner than a raw dict
class ModelStore:
    def __init__(self):
        self.model = None
        self.version = None
    
    def load(self, path: str, version: str):
        with open(path, "rb") as f:
            self.model = pickle.load(f)
        self.version = version
    
    def predict(self, features):
        if self.model is None:
            raise RuntimeError("Model not loaded")
        return self.model.predict([features])[0]

model_store = ModelStore()

@asynccontextmanager
async def lifespan(app: FastAPI):
    model_store.load("model/iris_model.pkl", version="1.0.0")
    yield

app = FastAPI(lifespan=lifespan)

# Dependency function — FastAPI calls this and injects the result
def get_model() -> ModelStore:
    return model_store

# Route uses Depends() to receive the model store
@app.post("/predict")
def predict(request: PredictionRequest, store: ModelStore = Depends(get_model)):
    pred = store.predict([
        request.sepal_length, request.sepal_width,
        request.petal_length, request.petal_width
    ])
    return {"prediction": pred, "model_version": store.version}
```

Why bother over a global variable? When testing, you can swap `get_model` for a dependency that returns a mock model — without changing any route code.

```python
# In tests — override the dependency cleanly
def get_mock_model():
    class MockModel:
        version = "mock"
        def predict(self, features): return "setosa"
    return MockModel()

app.dependency_overrides[get_model] = get_mock_model
```

---

## Model Versioning

A common pattern for serving multiple model versions:

```python
from fastapi import FastAPI, Depends, HTTPException, Query

models = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    models["v1"] = load_model("model/iris_v1.pkl")
    models["v2"] = load_model("model/iris_v2.pkl")
    models["latest"] = models["v2"]
    yield

app = FastAPI(lifespan=lifespan)

def get_versioned_model(version: str = Query("latest", enum=["v1", "v2", "latest"])):
    if version not in models:
        raise HTTPException(status_code=404, detail=f"Model version {version} not found")
    return models[version]

@app.post("/predict")
def predict(
    request: PredictionRequest,
    model = Depends(get_versioned_model)
):
    ...
```

Call: `POST /predict?version=v1` or `POST /predict?version=latest`

---

## 🛠️ Hands-On Exercise 3 — Real Model Loaded at Startup

**What you'll do:** Wire up your actual saved sklearn model (from the Docker tutorial) into the FastAPI app with proper startup loading and DI.

**Step 1: Train and save a model if you don't have one**

```python
# run_once/train.py — run this once outside the API
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
import pickle, os

os.makedirs("model", exist_ok=True)
iris = load_iris()
clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(iris.data, iris.target)

with open("model/iris_model.pkl", "wb") as f:
    pickle.dump({"model": clf, "classes": iris.target_names.tolist()}, f)
print("Saved.")
```

```bash
python run_once/train.py
```

**Step 2: Rewrite `main.py` with everything from Hours 1–3**

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel, Field
from contextlib import asynccontextmanager
from typing import Optional
import pickle

# --- Model store ---

class ModelStore:
    def __init__(self):
        self.model = None
        self.classes = None
        self.version = "unloaded"
    
    def load(self, path: str, version: str):
        with open(path, "rb") as f:
            data = pickle.load(f)
        self.model   = data["model"]
        self.classes = data["classes"]
        self.version = version
        print(f"[startup] Model {version} loaded from {path}")
    
    def predict(self, features: list[float]) -> tuple[str, float]:
        probs = self.model.predict_proba([features])[0]
        idx   = probs.argmax()
        return self.classes[idx], float(probs[idx])

model_store = ModelStore()

# --- Lifespan ---

@asynccontextmanager
async def lifespan(app: FastAPI):
    model_store.load("model/iris_model.pkl", version="1.0.0")
    yield
    print("[shutdown] Cleaning up.")

# --- App ---

app = FastAPI(title="Iris ML API", version="1.0.0", lifespan=lifespan)

# --- Schemas ---

class PredictionRequest(BaseModel):
    sepal_length: float = Field(..., gt=0, le=20)
    sepal_width:  float = Field(..., gt=0, le=20)
    petal_length: float = Field(..., gt=0, le=20)
    petal_width:  float = Field(..., gt=0, le=20)

class PredictionResponse(BaseModel):
    prediction:    str
    confidence:    float
    model_version: str

class HealthResponse(BaseModel):
    status:       str
    model_loaded: bool
    model_version: str

# --- Dependency ---

def get_model() -> ModelStore:
    if model_store.model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    return model_store

# --- Routes ---

@app.get("/health", response_model=HealthResponse)
def health():
    return HealthResponse(
        status="ok",
        model_loaded=model_store.model is not None,
        model_version=model_store.version
    )

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest, store: ModelStore = Depends(get_model)):
    features = [request.sepal_length, request.sepal_width,
                request.petal_length, request.petal_width]
    prediction, confidence = store.predict(features)
    return PredictionResponse(
        prediction=prediction,
        confidence=confidence,
        model_version=store.version
    )
```

**Step 3: Run and test**
```bash
uvicorn main:app --reload

# In another terminal:
curl http://localhost:8000/health
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

**✅ Checkpoint:** Your model loads once at startup. The dependency injects it into your route. `/health` tells you if the model is ready. You're now 90% of the way to a production-grade serving API.

---

# Hour 4 — Build: Complete ML Serving API

This hour assembles everything into a deployment-ready API for your containerized model, with `/predict`, `/health`, proper error handling, and a Dockerfile.

## Final Project Structure

```
fastapi-ml/
├── model/
│   └── iris_model.pkl        # your saved model
├── run_once/
│   └── train.py              # run locally to generate the model
├── main.py                   # FastAPI application
├── requirements.txt
├── Dockerfile
└── .dockerignore
```

---

## Step 1: requirements.txt

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.8.2
scikit-learn==1.5.2
```

---

## Step 2: The Complete `main.py`

This is production-ready — error handling, logging, versioned response, ready to containerize.

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field
from contextlib import asynccontextmanager
import pickle
import logging
import time
import os

# --- Logging ---
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# --- Model Store ---

class ModelStore:
    def __init__(self):
        self.model   = None
        self.classes = None
        self.version = "unloaded"
    
    def load(self, path: str, version: str):
        if not os.path.exists(path):
            raise FileNotFoundError(f"Model file not found: {path}")
        with open(path, "rb") as f:
            data = pickle.load(f)
        self.model   = data["model"]
        self.classes = data["classes"]
        self.version = version
        logger.info(f"Model {version} loaded from {path}")
    
    def predict(self, features: list[float]) -> tuple[str, float, list[dict]]:
        probs      = self.model.predict_proba([features])[0]
        idx        = probs.argmax()
        prediction = self.classes[idx]
        confidence = float(probs[idx])
        all_probs  = [
            {"class": cls, "probability": float(p)}
            for cls, p in zip(self.classes, probs)
        ]
        return prediction, confidence, all_probs

model_store = ModelStore()

# --- Lifespan ---

MODEL_PATH    = os.getenv("MODEL_PATH", "model/iris_model.pkl")
MODEL_VERSION = os.getenv("MODEL_VERSION", "1.0.0")

@asynccontextmanager
async def lifespan(app: FastAPI):
    try:
        model_store.load(MODEL_PATH, MODEL_VERSION)
    except FileNotFoundError as e:
        logger.error(f"Startup failed: {e}")
        # App still starts — /health will report model_loaded: false
    yield
    logger.info("Shutting down.")

# --- App ---

app = FastAPI(
    title="Iris Classifier API",
    description="FastAPI ML serving example — Random Forest on Iris dataset",
    version="1.0.0",
    lifespan=lifespan
)

# --- Request timing middleware ---

@app.middleware("http")
async def add_timing_header(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration_ms = (time.time() - start) * 1000
    response.headers["X-Process-Time-Ms"] = f"{duration_ms:.2f}"
    return response

# --- Schemas ---

class PredictionRequest(BaseModel):
    sepal_length: float = Field(..., gt=0, le=20, description="Sepal length in cm")
    sepal_width:  float = Field(..., gt=0, le=20, description="Sepal width in cm")
    petal_length: float = Field(..., gt=0, le=20, description="Petal length in cm")
    petal_width:  float = Field(..., gt=0, le=20, description="Petal width in cm")

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "sepal_length": 5.1, "sepal_width": 3.5,
                "petal_length": 1.4, "petal_width": 0.2
            }]
        }
    }

class ClassProbability(BaseModel):
    class_name: str = Field(alias="class")
    probability: float
    model_config = {"populate_by_name": True}

class PredictionResponse(BaseModel):
    prediction:    str
    confidence:    float
    probabilities: list[dict]
    model_version: str

class HealthResponse(BaseModel):
    status:        str
    model_loaded:  bool
    model_version: str
    model_path:    str

# --- Dependency ---

def get_model() -> ModelStore:
    if model_store.model is None:
        raise HTTPException(
            status_code=503,
            detail="Model not loaded. Check /health for details."
        )
    return model_store

# --- Routes ---

@app.get("/health", response_model=HealthResponse, tags=["ops"])
def health():
    """
    Health check — use this in your Docker HEALTHCHECK and load balancer.
    Returns model_loaded: false if the model failed to load at startup.
    """
    return HealthResponse(
        status="ok" if model_store.model is not None else "degraded",
        model_loaded=model_store.model is not None,
        model_version=model_store.version,
        model_path=MODEL_PATH
    )

@app.post("/predict", response_model=PredictionResponse, tags=["inference"])
def predict(request: PredictionRequest, store: ModelStore = Depends(get_model)):
    """
    Run inference on a single sample.
    Returns the predicted class, confidence, and full probability distribution.
    """
    features = [
        request.sepal_length, request.sepal_width,
        request.petal_length, request.petal_width
    ]
    
    try:
        prediction, confidence, probabilities = store.predict(features)
    except Exception as e:
        logger.error(f"Inference error: {e}")
        raise HTTPException(status_code=500, detail="Inference failed")
    
    logger.info(f"Predicted: {prediction} ({confidence:.2%})")
    
    return PredictionResponse(
        prediction=prediction,
        confidence=confidence,
        probabilities=probabilities,
        model_version=store.version
    )
```

---

## Step 3: Dockerfile

Multi-stage build from Hour 2, tuned for FastAPI:

```dockerfile
# ============ Stage 1: Builder ============
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt


# ============ Stage 2: Runtime ============
FROM python:3.12-slim

WORKDIR /app

# Copy installed packages
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy model and app code
COPY model/iris_model.pkl model/iris_model.pkl
COPY main.py .

# Environment defaults (overridable at runtime)
ENV MODEL_PATH=model/iris_model.pkl
ENV MODEL_VERSION=1.0.0

# Non-root user
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000

# Health check — Docker will call this and mark container unhealthy if it fails
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

> **Gotcha — `--host 0.0.0.0`:** By default uvicorn binds to `127.0.0.1` (localhost inside the container). That means port mapping (`-p 8000:8000`) won't work — requests from outside the container can't reach it. Always use `--host 0.0.0.0` in Docker so uvicorn listens on all interfaces.

---

## Step 4: .dockerignore

```
__pycache__/
*.pyc
.venv/
venv/
run_once/
.env
.git/
*.ipynb
```

---

## Step 5: Build and Run

```bash
# Build
docker build -t iris-api .

# Run
docker run -p 8000:8000 iris-api

# Test from another terminal
curl http://localhost:8000/health

curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'

# Setosa: 5.1, 3.5, 1.4, 0.2
# Versicolor: 6.4, 3.2, 4.5, 1.5
# Virginica: 6.3, 2.5, 5.0, 1.9
```

---

## Step 6: Override Config at Runtime

Because model path and version are environment variables, you can swap models without rebuilding:

```bash
docker run -p 8000:8000 \
  -e MODEL_VERSION=2.0.0 \
  -v $(pwd)/model_v2:/app/model \
  iris-api
```

---

## Step 7: docker-compose.yml for Development

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./main.py:/app/main.py          # live reload your code
      - ./model:/app/model              # swap models without rebuilding
    environment:
      - MODEL_VERSION=1.0.0
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

```bash
docker compose up
# Edit main.py → server reloads automatically
```

---

## Step 8: Verify Everything

```bash
# Check image size
docker images iris-api

# Inspect layers
docker history iris-api

# Check the HEALTHCHECK is passing
docker ps  # look for "(healthy)" in the STATUS column after ~30 seconds

# Open interactive docs
open http://localhost:8000/docs
```

**✅ Final Checkpoint:** You have a containerized FastAPI ML API with:
- Pydantic-validated input/output
- Model loaded once at startup via lifespan
- Clean dependency injection
- `/health` endpoint for monitoring
- `/predict` endpoint with full probability output
- Request timing middleware
- Multi-stage Docker build with non-root user
- HEALTHCHECK in the Dockerfile
- Runtime-configurable model path and version

---

# Gotchas Summary

| Gotcha | Problem | Fix |
|---|---|---|
| Loading model in route handler | Model reloaded on every request — seconds of latency | Load in `lifespan`, store globally, inject via `Depends` |
| `@app.on_event("startup")` | Deprecated in newer FastAPI | Use `lifespan` context manager instead |
| `async def` for numpy/sklearn inference | Blocks the event loop — worse than sync | Use plain `def`; FastAPI runs it in a thread pool automatically |
| uvicorn defaulting to `127.0.0.1` | Port mapping doesn't work in Docker | Always add `--host 0.0.0.0` in Docker CMD |
| Missing `Field(...)` vs `Field(default)` | `...` means required; forgetting it makes fields optional silently | Use `Field(...)` for required, `Field(default_value)` for optional |
| No `/health` endpoint | Load balancers and Docker HEALTHCHECK have nothing to probe | Always add `/health`; return `model_loaded` status |
| Hardcoding model path | Can't swap models without rebuilding the image | Use `os.getenv("MODEL_PATH", "default")` and pass via `-e` |
| Forgetting `response_model=` | Output not validated or documented in /docs | Always declare `response_model` on routes |

---

# Quick Reference Card

```bash
# Run locally
uvicorn main:app --reload                          # development (auto-reload)
uvicorn main:app --host 0.0.0.0 --port 8000       # production / Docker

# Test endpoints
curl http://localhost:8000/health
curl http://localhost:8000/docs                    # interactive browser UI

curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'

# Docker
docker build -t iris-api .
docker run -p 8000:8000 iris-api
docker run -p 8000:8000 -e MODEL_VERSION=2.0 iris-api   # override env var
docker ps   # check (healthy) status after HEALTHCHECK kicks in

# Compose
docker compose up --build
docker compose logs -f api
```

```python
# FastAPI patterns at a glance

# Path param
@app.get("/items/{id}")
def get(id: int): ...

# Query param (optional, with default)
@app.get("/items")
def list(skip: int = 0, limit: int = 10): ...

# Request body (Pydantic)
@app.post("/items")
def create(item: ItemRequest): ...

# Dependency injection
def get_db(): return db_connection
@app.get("/items")
def list(db = Depends(get_db)): ...

# HTTP errors
raise HTTPException(status_code=404, detail="Not found")

# Background task
@app.post("/items")
def create(bg: BackgroundTasks):
    bg.add_task(log_something)
    return {"ok": True}

# Startup/shutdown
@asynccontextmanager
async def lifespan(app):
    # startup
    yield
    # shutdown
app = FastAPI(lifespan=lifespan)
```

---

*Natural next steps: add authentication (FastAPI OAuth2/JWT), swap the sklearn model for a PyTorch model, add request batching for throughput, and deploy behind nginx with Docker Compose.*
