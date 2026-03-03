# Workflow Scribe: AI Workflow Recording & Replay Agent
## Implementation Plan

### 1. System Overview
Workflow Scribe is an AI-powered system that records, understands, narrates, and replays computer workflows. It combines real-time screen capture, vision-language models, workflow memory, and autonomous agent replay.

### 2. Architecture (4 Layers)

#### Layer 1: Screen Capture & Ingestion
- **Library**: Python `mss` (30-60 FPS cross-platform) or `DXcam` (240Hz+ on Windows)
- **Input capture**: `pynput` for mouse/keyboard event logging with timestamps
- **Frame pipeline**: Capture at 2-5 FPS for VLM analysis (cost-optimized), full FPS for video storage
- **Storage**: Raw frames saved to local disk, compressed via FFmpeg to MP4 for archival
- **Streaming**: Frames sent to processing pipeline via async queue (Python `asyncio` + `aiofiles`)

#### Layer 2: Vision Understanding & Narration
- **Primary VLM**: GPT-4o or Claude 3.5 Sonnet for frame analysis and natural language narration
- **Screen parsing**: Microsoft OmniParser v2 for structured UI element extraction (uses fine-tuned YOLOv8 for interactable region detection + PaddleOCR + fine-tuned Florence model for icon descriptions)
- **Action recognition**: Each frame pair analyzed for state changes; actions classified as click, type, scroll, navigate, etc.
- **Narration engine**: VLM receives parsed screen elements + raw screenshot + previous context window, outputs natural language description of what the user is doing
- **Temporal context**: Sliding window of last 10 frames + narrations maintained for coherent multi-step understanding

#### Layer 3: Workflow Memory & Learning
- **Workflow graph**: Actions stored as state transition diagrams (nodes = screen states, edges = actions) following the AgentRR paradigm
- **Experience levels**: Three tiers of abstraction - (1) Raw action traces, (2) Step-level procedures, (3) Task-level workflows
- **Vector storage**: Workflow embeddings stored in Pinecone/ChromaDB for semantic similarity search
- **Embedding model**: OpenAI text-embedding-3-large or Cohere embed-v3 for workflow chunk embeddings
- **Knowledge graph**: Neo4j for storing relationships between workflows, sub-tasks, and application contexts
- **Learning loop**: After each recording session, summarize traces into reusable experiences; rank by success rate and frequency

#### Layer 4: Agent Replay & Autonomous Execution
- **Replay engine**: Based on AgentRR's record-replay paradigm - large model records, small model replays
- **Action execution**: PyAutoGUI or Anthropic Computer Use API for mouse/keyboard control
- **Screen grounding**: OmniParser for element detection + VLM for action-to-coordinate mapping (GUI-Actor approach for coordinate-free grounding)
- **Condition checking**: Before/after each action, capture screenshot and verify expected state via VLM
- **Self-recovery**: If state doesn't match expectation, agent re-plans from current state using workflow graph
- **Safety**: Human-in-the-loop confirmation for destructive actions; sandboxed execution via Docker/VM

### 3. Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.11+ |
| Screen Capture | `mss` (cross-platform), `DXcam` (Windows) |
| Input Recording | `pynput` |
| Video Processing | OpenCV, FFmpeg |
| Vision-Language Model | GPT-4o API / Claude 3.5 Sonnet API |
| Screen Parsing | OmniParser v2 (YOLOv8 + Florence + PaddleOCR) |
| Workflow Storage | PostgreSQL (metadata) + ChromaDB (vectors) + Neo4j (graph) |
| Agent Framework | LangGraph for multi-step orchestration |
| Agent Execution | PyAutoGUI / Anthropic Computer Use API |
| Backend API | FastAPI |
| Frontend Dashboard | Next.js + React |
| Deployment | Docker Compose, Fly.io or Railway |
| CI/CD | GitHub Actions |

### 4. Data Models

```python
@dataclass
class ScreenFrame:
    timestamp: float
    image_path: str
    resolution: tuple[int, int]
    mouse_position: tuple[int, int]
    active_window: str

@dataclass  
class Action:
    timestamp: float
    action_type: str  # click, type, scroll, hotkey
    target_element: str  # OmniParser element ID
    parameters: dict  # coordinates, text, key combo
    before_frame: str
    after_frame: str

@dataclass
class WorkflowStep:
    step_id: str
    description: str  # Natural language narration
    actions: list[Action]
    preconditions: list[str]
    postconditions: list[str]
    application_context: str

@dataclass
class Workflow:
    workflow_id: str
    name: str
    description: str
    steps: list[WorkflowStep]
    embedding: list[float]
    success_count: int
    total_attempts: int
```

### 5. Recording Pipeline
1. Start screen capture at 2-5 FPS (VLM) + full FPS (archival)
2. Log all mouse/keyboard events via pynput
3. Every N frames, send to OmniParser for UI element extraction
4. Send parsed elements + screenshot to VLM for narration
5. Detect action boundaries (state changes between frames)
6. Build action trace as sequence of (state, action, state) tuples
7. On session end, summarize into WorkflowStep and Workflow objects
8. Generate embeddings, store in vector DB and knowledge graph

### 6. Replay Pipeline
1. User describes task in natural language
2. Retrieve most similar workflow from vector DB
3. Load workflow graph and step sequence
4. For each step: capture current screen -> parse with OmniParser -> map expected action to current UI -> execute via PyAutoGUI
5. After each action, verify state matches expected postcondition
6. If mismatch, re-plan from current state or escalate to human
7. Log replay trace for experience refinement

### 7. API Endpoints
- `POST /record/start` - Begin recording session
- `POST /record/stop` - End recording, trigger summarization
- `GET /workflows` - List all learned workflows
- `GET /workflows/{id}` - Get workflow details with steps
- `POST /replay` - Start autonomous replay of a workflow
- `GET /replay/{id}/status` - Check replay progress
- `POST /search` - Semantic search across workflows
- `GET /narration/stream` - SSE stream of real-time narration

### 8. Deployment Plan
1. **Phase 1 (MVP)**: Screen capture + VLM narration + basic storage. Single user, local execution.
2. **Phase 2**: Add OmniParser integration, workflow graph storage, vector search.
3. **Phase 3**: Agent replay with condition checking and self-recovery.
4. **Phase 4**: Multi-user support, web dashboard, cloud deployment.

### 9. Key Research References
- **OmniParser** (Microsoft): Pure vision-based UI screen parsing with YOLOv8 + Florence + OCR
- **AgentRR**: Record & Replay paradigm for LLM agents with multi-level experience abstraction
- **GUI-Actor**: Coordinate-free visual grounding for GUI agents using attention-based action heads
- **SkillForge**: Screen recording to agent-ready replayable skills
- **Anthropic Computer Use**: Claude-based screen observation and autonomous computer control
- **OpenAI CUA**: Computer-Using Agent for multi-step web/desktop automation
- **LangGraph**: Graph-based multi-agent workflow orchestration

### 10. Repository Structure
```
workflow-scribe/
  PLAN.md
  README.md
  requirements.txt
  src/
    capture/       # Screen capture and input recording
    vision/        # VLM integration and OmniParser
    narration/     # Real-time narration engine
    memory/        # Workflow storage, embeddings, graph
    replay/        # Autonomous replay engine
    api/           # FastAPI endpoints
  frontend/        # Next.js dashboard
  docker-compose.yml
  .github/workflows/  # CI/CD
```
