# Workflow Engine 

A minimal LangGraph-style workflow engine built with FastAPI which implements a small-agent with nodes, edges, shared state, looping, WebSocket streaming, background tasks, and structured logging.

---

## Table of Contents

- [Features](#features)
- [Project Structure](#project-structure)
- [How to Run](#how-to-run)
- [Testing the Workflow](#testing-the-workflow)
- [What the Engine Supports](#what-the-engine-supports)
- [API Endpoints](#api-endpoints)
- [Future Improvements](#future-improvements)

---

## Features

✅ **Workflow Graph Engine**: Nodes (async functions), edges (dict mapping), shared state (mutable dict)  
✅ **Code Review Mini-Agent**: Extract functions → Check complexity → Detect issues → Suggest improvements (with loop)  
✅ **FastAPI APIs**: All required endpoints + WebSocket support  
✅ **Background Tasks**: Non-blocking async workflow execution  
✅ **WebSocket Streaming**: Live logs via `ws://localhost:8000/ws/run/{run_id}`  
✅ **Structured Logging**: JSON logs to `logs.json` + console output  
✅ **Tool Registry**: Extensible, pre-registered code review functions

---

## Project Structure

workflow-engine/
├── app/
│ ├── init.py
│ ├── main.py # FastAPI app + routes
│ ├── config.py # Settings (Pydantic)
│ ├── logging_config.py # Loguru logging setup
│ ├── models.py # Pydantic models
│ ├── graph_engine.py # Core workflow logic
│ └── tools/
│ ├── init.py # Tool registry
│ └── code_review.py # Node implementations
├── requirements.txt
└── README.md

text

---

## How to Run

### Step 1: Setup Environment

python -m venv venv

Windows
venv\Scripts\activate

macOS/Linux
source venv/bin/activate

text

### Step 2: Install Dependencies

pip install "uvicorn[standard]"
pip install -r requirements.txt

text

### Step 3: Start the Server

uvicorn app.main:app --reload

text

**Access:**
- API: `http://127.0.0.1:8000`
- Interactive Docs: `http://127.0.0.1:8000/docs`

A default `code_review` graph is auto-created on startup.

---

## Testing the Workflow

### Option 1: Using Swagger UI (Recommended)

1. Open `http://127.0.0.1:8000/docs`
2. Find **POST** `/graph/{graph_id}/run`
3. Click **Try it out**
4. Set `graph_id` to: `code_review`
5. Request body:
{
"code": "def foo():\n print('debug')\n # TODO: fix this"
}

text
6. Click **Execute** → Copy the `run_id` from response
7. Go to **GET** `/graph/run/{run_id}/state`
8. Paste `run_id` → **Execute**

### Option 2: Using curl

Start workflow
curl -X POST "http://127.0.0.1:8000/graph/code_review/run"
-H "Content-Type: application/json"
-d "{"code": "def foo():\n print('debug')\n # TODO: fix"}"

Check result (use run_id from previous response)
curl "http://127.0.0.1:8000/graph/run/<RUN_ID>/state"

text

**Expected Output:**
{
"status": "completed",
"state": {
"functions": ["foo"],
"issues": ["Debug prints present", "Contains TODO comments"],
"quality_score": 0.8,
"improvements": ["Consider addressing: Debug prints present"]
}
}

text

---

## What the Engine Supports

### Core Features

**1. Nodes**  
Async Python functions that read and modify shared state. Each node is a tool registered in the global `tools` dictionary.

**2. Edges**  
Simple `Dict[str, str]` mapping defining which node runs after which (e.g., `"extract_functions": "check_complexity"`).

**3. Shared State**  
Mutable `Dict[str, Any]` flows between nodes, accumulating data like `functions`, `issues`, `quality_score`.

**4. Looping**  
The workflow repeats nodes until `quality_score >= 0.9` or `max_iterations` (default: 5) is reached.

**5. Branching (Basic)**  
Conditional routing implemented via loop termination based on state values.

### Code Review Mini-Agent Workflow

extract_functions → check_complexity → detect_issues → suggest_improvements (loop)
↑ |
└───────────────────────────────────────────────────────┘
(until quality_score >= 0.9)

text

**Node Details:**
- `extract_functions`: Regex-based function name extraction
- `check_complexity`: Line count + nesting analysis
- `detect_issues`: Detects prints, TODOs, long files
- `suggest_improvements`: Rule-based suggestions, increments quality_score

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| **POST** | `/graph/create` | Create new workflow graph |
| **POST** | `/graph/{graph_id}/run` | Start workflow run (returns `run_id`) |
| **GET** | `/graph/run/{run_id}/state` | Get run status, logs, and final state |
| **WS** | `/ws/run/{run_id}` | Stream live logs (`node_start`, `node_end`, `completed`) |

### Example: Create Custom Graph

{
"nodes": {
"extract_functions": "extract_functions",
"check_complexity": "check_complexity",
"detect_issues": "detect_issues",
"suggest_improvements": "suggest_improvements"
},
"edges": {
"extract_functions": "check_complexity",
"check_complexity": "detect_issues",
"detect_issues": "suggest_improvements",
"suggest_improvements": "suggest_improvements"
},
"max_iterations": 5
}

text

---

## Future Improvements

### With More Time, I Would Add:

**1. Persistent Storage**  
Replace in-memory `graphs`/`runs` dicts with SQLite or Postgres for durability and history tracking.

**2. Dynamic Branching**  
Conditional edges based on state values (e.g., `if quality_score > 0.7: goto node_x else: goto node_y`).

**3. Tool Registration API**  
`POST /tools/register` endpoint to dynamically add new tools at runtime.

**4. State Validation**  
Pydantic schemas per graph to validate state structure and types.

**5. Observability**  
- Prometheus metrics (run duration, success rate, node timings)
- Distributed tracing (OpenTelemetry)
- Better error handling and retries

**6. Production Features**  
- Docker multi-stage builds
- Authentication & rate limiting
- Horizontal scaling with Redis for shared state

---
