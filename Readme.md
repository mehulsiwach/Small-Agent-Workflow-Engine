Workflow Engine
A simplified LangGraph-style workflow engine which Implements the Mini-Agent workflow with nodes, edges, shared state, looping, WebSocket streaming, background tasks, and structured logging.​

Features
Workflow Graph Engine: Nodes (async functions), edges, shared state (dict), looping until quality_score >= 0.9

Code Review Mini-Agent: Extracts functions → checks complexity → detects issues → suggests improvements

FastAPI Endpoints: POST /graph/create, POST /graph/{graph_id}/run, GET /graph/run/{run_id}/state

Background Tasks: Async workflow execution (non-blocking)

WebSocket Streaming: Live logs via ws://localhost:8000/ws/run/{run_id}

Structured Logging: JSON logs to logs.json + console output

Tool Registry: Extensible tools (pre-registered code review functions)

 Project Structure
text
ai_workflow_engine/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app + routes
│   ├── config.py            # Pydantic settings
│   ├── logging_config.py    # Loguru JSON logging
│   ├── models.py            # Pydantic models
│   ├── graph_engine.py      # Core workflow logic
│   └── tools/
│       ├── __init__.py      # Tool registry
│       └── code_review.py   # Node implementations
├── requirements.txt
├── logs.json                # Generated logs
└── README.md
 Quick Start
1. Clone & Setup
bash
git clone <your-repo>
cd ai_workflow_engine
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate
pip install "uvicorn[standard]"
pip install -r requirements.txt
2. Run Server
bash
uvicorn app.main:app --reload
Server: http://127.0.0.1:8000
Docs: http://127.0.0.1:8000/docs

3. Test Code Review Workflow
Default graph code_review is auto-created on startup.

bash
curl -X POST "http://127.0.0.1:8000/graph/code_review/run" \
  -H "Content-Type: application/json" \
  -d "{\"code\": \"def foo():\n    print('debug')\n    # TODO: fix\"}"
Response:

json
{
  "run_id": "6685064e-12b9-4dd9-a486-a8a4d87a10d1",
  "status": "queued"
}
Check results:

bash
curl "http://127.0.0.1:8000/graph/run/<RUN_ID>/state"
Sample Output:

json
{
  "status": "completed",
  "state": {
    "code": "def foo():\n    print('debug')\n    # TODO: fix this",
    "functions": ["foo"],
    "issues": ["Debug prints present", "Contains TODO comments"],
    "quality_score": 0.8,
    "improvements": ["Consider addressing: Debug prints present"]
  }
}
 Workflow Flow
text
extract_functions → check_complexity → detect_issues → suggest_improvements (loop)
                           ↑
                   quality_score >= 0.9 ?
                           ↓
                        STOP
extract_functions: Regex finds def foo(): → {"functions": ["foo"]}

check_complexity: Lines + nesting → {"complexity_score": 0.995}

detect_issues: Smells (prints, TODOs) → {"issues": ["Debug prints present"]}

suggest_improvements: Generates fixes → {"quality_score": +0.1}

Loop: Repeats improvements until quality_score >= 0.9 or max 5 iterations

 API Reference
Endpoint	Method	Description
/graph/create	POST	Create custom graph (nodes + edges)
/graph/{graph_id}/run	POST	Start workflow run (returns run_id)
/graph/run/{run_id}/state	GET	Get run status + final state
/ws/run/{run_id}	WS	Stream live logs (node_start, node_end, completed)
 Logs
Console: Uvicorn + execution logs

File: logs.json (JSONL format, rotates at 1MB)

bash
# View logs
tail -f logs.json
# or open in VS Code
 Architecture Decisions
In-memory storage: graphs and runs dicts (SQLite optional)

Shared state: Mutable Dict[str, Any] passed between nodes

Async hygiene: All nodes + workflow are async

Tool registry: Global tools dict, nodes map to tool names

Looping: Simple while quality_score < 0.9 and iteration < max_iter

Error handling: Graceful WS disconnects, missing tools

 Improvements with More Time
Persistent Storage: SQLite/Postgres for graphs/runs

Dynamic Branching: Conditional edges based on state

Tool API: POST /tools/register

Validation: Pydantic state schemas per graph

Metrics: Prometheus + Grafana for workflow stats

Docker: Multi-stage builds for production

Rate Limiting: Prevent abuse on /graph/run