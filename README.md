#Workflow Engine 
A small backend project implementing a simplified LangGraph-style workflow engine. It exposes a minimal graph engine over FastAPI, with a Mini‑Agent workflow, background execution, WebSocket log streaming, and structured logging.​

#Overview
The engine lets you define a workflow as a set of nodes (Python functions), edges (which node runs next), and a shared state dictionary that flows between nodes.​
The provided sample workflow (Option A from the assignment) reviews Python code by extracting functions, checking complexity, detecting basic issues, and suggesting improvements in a loop until a quality threshold is reached.​

#Features
Minimal workflow graph engine:

Nodes as async Python functions reading and mutating shared state

Edges as a Dict[str, str] mapping from_node -> to_node

Simple looping based on quality_score and a max iteration limit

Code Review Mini‑Agent workflow:

extract_functions → check_complexity → detect_issues → suggest_improvements (loop)

#FastAPI backend:

POST /graph/create – create a graph

POST /graph/{graph_id}/run – start a workflow run

GET /graph/run/{run_id}/state – inspect run status and state

Async/background execution using FastAPI background tasks

WebSocket endpoint to stream node‑by‑node logs in real time

Structured logging to logs.json plus console logs

#Project Structure
.
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI app and routes
│   ├── config.py          # Application settings
│   ├── logging_config.py  # Loguru logging configuration
│   ├── models.py          # Pydantic models (requests/responses)
│   ├── graph_engine.py    # Core workflow engine and in-memory stores
│   └── tools/
│       ├── __init__.py    # Tool registry
│       └── code_review.py # Code Review Mini-Agent node implementations
├── requirements.txt
└── README.md

#Prerequisites
Python 3.9+

pip / venv

#Installation
From the project root:

bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install "uvicorn[standard]"
pip install -r requirements.txt
#Run the Server
bash
uvicorn app.main:app --reload
API root: http://127.0.0.1:8000

Interactive docs (Swagger): http://127.0.0.1:8000/docs

On startup, a default graph with ID code_review is registered automatically.​

#Usage
1. Run the Code Review Workflow (via Swagger UI)
Open http://127.0.0.1:8000/docs in your browser.

Find POST /graph/{graph_id}/run and click Try it out.

Set the graph_id path parameter to:

text
code_review
In the request body, provide some Python code:

json
{
  "code": "def foo():\n    print('debug')\n    # TODO: fix this"
}
Click Execute.
The response will contain a run_id and a status like:

json
{
  "run_id": "some-uuid",
  "status": "queued"
}
Scroll to GET /graph/run/{run_id}/state, click Try it out, paste the run_id, and Execute to see:

status (queued → running → completed)

execution log

final state (functions, issues, improvements, quality_score, iteration)

2. Run the Code Review Workflow (via curl)
Start a run:

bash
curl -X POST "http://127.0.0.1:8000/graph/code_review/run" \
  -H "Content-Type: application/json" \
  -d "{\"code\": \"def foo():\n    print('debug')\n    # TODO: fix\"}"
Response example:

json
{
  "run_id": "6685064e-12b9-4dd9-a486-a8a4d87a10d1",
  "status": "queued"
}
Check the state:

bash
curl "http://127.0.0.1:8000/graph/run/6685064e-12b9-4dd9-a486-a8a4d87a10d1/state"
Example result:

json
{
  "run_id": "6685...",
  "status": "completed",
  "log": [
    "... Executing node 'extract_functions' ...",
    "... Finished node 'suggest_improvements' ...",
    "... Workflow completed"
  ],
  "state": {
    "code": "def foo():\n    print('debug')\n    # TODO: fix this",
    "iteration": 5,
    "functions": ["foo"],
    "quality_score": 0.8,
    "complexity_score": 0.995,
    "issues": [
      "Debug prints present",
      "Contains TODO comments"
    ],
    "improvements": [
      "Consider addressing: Debug prints present",
      "Consider addressing: Contains TODO comments"
    ]
  }
}
API Endpoints
POST /graph/create
Create a new workflow graph dynamically.

Request body:

json
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
Response:

json
{
  "graph_id": "graph_2"
}
POST /graph/{graph_id}/run
Start a new run for a given graph.

Path parameter: graph_id (e.g. code_review or any ID returned from /graph/create)

Body: initial state (for this workflow, at least a code string)

json
{
  "code": "def foo():\n    print('debug')\n    # TODO: fix this"
}
Returns:

json
{
  "run_id": "some-uuid",
  "status": "queued"
}
GET /graph/run/{run_id}/state
Get the current status, execution log, and state of a run.

Returns:

json
{
  "run_id": "some-uuid",
  "status": "completed",
  "log": ["...", "..."],
  "state": {
    "...": "..."
  }
}
WebSocket /ws/run/{run_id}
Connect a WebSocket client to stream logs step‑by‑step.

URL: ws://127.0.0.1:8000/ws/run/<RUN_ID>

Messages include:

event: "node_start" | "node_end" | "completed"

node, log, and optionally state fragments

#Implementation Details
Nodes: Implemented as async functions in app/tools/code_review.py that accept the current state as keyword arguments and return a partial state update.

Shared State: A Dict[str, Any] that is passed into each node and updated with each node’s return value.

Graph Definition:

nodes: map from node ID to tool name in the registry

edges: map from node ID to the next node ID

max_iterations: safety guard to avoid infinite loops

Execution:

The engine starts from extract_functions for the code review workflow.

It loops through nodes following the edges map.

It stops when quality_score >= 0.9 or when max_iterations is reached.

Storage:

Graphs and runs are stored in global in‑memory dictionaries (graphs and runs), which can be replaced by a database later.​

Logging
Console logs:

Standard FastAPI / Uvicorn logs

Node execution logs (Executing node ..., Finished node ..., Workflow completed)

File logs:

logs.json (JSON Lines format), with rotation at 1 MB and 7‑day retention

Powered by Loguru, configured in logging_config.py

Example to tail logs:

bash
# Linux / macOS
tail -f logs.json

# Windows PowerShell
Get-Content .\logs.json -Tail 20 -Wait
Future Improvements
Persist graphs and runs in SQLite or Postgres instead of in‑memory storage.​

Add richer branching/conditional routing based on state values (not just linear edges).​

Expose a tool registration endpoint to dynamically add tools at runtime.

Use Pydantic models for strongly‑typed state validation per graph.

Add metrics (e.g. Prometheus) for run duration, node failures, and throughput.
