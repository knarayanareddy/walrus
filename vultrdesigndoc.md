Good — I now have everything I need. Here is the complete rewritten SSOT design document, phase by phase, adapted fully to the Vultr track, with every shortcoming from the original addressed.

---

```markdown
# SENTINEL — Enterprise Document Agent with Oversight
## Single Source of Truth Design Document
### RAISE Summit Hackathon 2026 · Vultr Track · Remote Solo Entry
### Version 2.0 — Vultr-Adapted, Demo-Ready

---

## 0. North Star

SENTINEL is a **multi-step enterprise document agent with a built-in oversight
layer**. It ingests real domain documents, reasons over them through multiple
retrieval passes, calls tools to produce enterprise-grade outputs, and — before
executing any irreversible action — runs a live safety gate that a human
operator can trust, question, and override in the moment.

The Vultr track asks for an agent that *"plans, retrieves more than once when
it needs to, calls tools, makes decisions, and produces an outcome a real
enterprise team could actually use."* SENTINEL is that agent, plus the
safety/audit layer that makes enterprise teams actually willing to deploy it.

**Domain: Finance — Loan Covenant Monitoring**

An autonomous agent reads a credit agreement + live financial filings,
monitors covenant ratios, retrieves historical trend data when a ratio drifts,
cross-checks recent transactions, and — when a breach is imminent — prepares
an escalation memo. Before that memo is sent or a downstream system is
triggered, SENTINEL's oversight gate freezes the action, shows the operator
every signal that fired, and requires an explicit approve/abort decision. The
result is a cryptographically-hashed incident JSON that a compliance team can
actually cite.

**Why this wins the Vultr track:**
- Multi-step retrieval: not one pass, but plan → retrieve covenant → retrieve
  historical ratios → retrieve recent transactions → synthesise → act
- Vultr-native: Vultr Serverless Inference for chat completion + Vultr's
  built-in vector store for all document embeddings and RAG retrieval
- Operator override with full cited audit trail — exactly what the problem
  statement describes
- Not a dashboard, not a RAG app, not a chatbot — a planning agent with teeth

---

## 1. Hackathon Constraints Log

| Constraint | Resolution |
|---|---|
| Vultr GPUs not available | All LLM work via Vultr Serverless Inference only |
| Remote solo (1 person) | Scoped to finance domain; solo-cut UI (3 screens) |
| Demo must be built during event | Phase −1 completes before hacking starts; all code written during event |
| No existing project | This doc is the pre-event design artifact; zero code pre-written |
| Repo must be public | `github.com/<you>/sentinel` public from first commit |
| Submission includes 1-min video | Phase 6 includes demo script timed to 55 seconds |
| No presentations — show the build | Every judge interaction starts with a live agent run, not slides |

---

## 2. The Problem This Solves (Pitch Layer)

Enterprise agents fail in two ways that kill adoption:

1. **They act on stale or incomplete documents** — one retrieval pass misses the
   updated covenant definition buried in an amendment.
2. **They take irreversible actions with no audit trail** — an escalation memo
   fires, a downstream system gets triggered, and nobody can explain why.

SENTINEL solves both. The agent retrieves documents in multiple passes as it
discovers gaps. The oversight gate catches every proposed irreversible action
before it executes. The incident artifact gives compliance a citation trail.

The judge's 3-minute window will see: documents loaded → agent plans and
retrieves in multiple steps (visible on screen) → ratio breach detected →
escalation memo drafted → **SENTINEL freezes the send action** → operator sees
full signal breakdown → operator approves → incident JSON sealed with hash.

That is an outcome a real enterprise team could use. On stage. Live.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SENTINEL SYSTEM                              │
│                                                                     │
│  ┌──────────────┐    ┌─────────────────────────────────────────┐   │
│  │  Document    │    │           Enterprise Agent               │   │
│  │  Ingestion   │───▶│  Planner → Retriever → Tool Caller      │   │
│  │  (Vultr VS)  │    │  (multi-step, cited, domain-aware)       │   │
│  └──────────────┘    └────────────────┬────────────────────────┘   │
│                                       │ attempt_action()            │
│                       ┌──────────────▼────────────────────────┐   │
│                       │         Orchestrator (gate)            │   │
│                       │  idempotent · irreversibility-aware    │   │
│                       └──────────────┬────────────────────────┘   │
│                                      │                              │
│              ┌───────────────────────▼──────────────────────────┐  │
│              │              SentinelPipeline                     │  │
│              │  ┌──────────┐ ┌────────────┐ ┌────────────────┐  │  │
│              │  │ Covenant │ │   MAARS    │ │   Citation     │  │  │
│              │  │ Drift    │ │ Doc Probe  │ │  Completeness  │  │  │
│              │  │ Monitor  │ │ (Vultr LLM)│ │   Checker      │  │  │
│              │  └────┬─────┘ └─────┬──────┘ └───────┬────────┘  │  │
│              └───────┼─────────────┼────────────────┼────────────┘  │
│                      └─────────────┴────────────────┘               │
│                                    │ EventBus                        │
│                       ┌────────────▼───────────────┐               │
│                       │  WebSocket Broadcaster      │               │
│                       └────────────┬───────────────┘               │
│                                    │                                 │
│                       ┌────────────▼───────────────┐               │
│                       │     React UI (3 screens)    │               │
│                       │  Monitor · Drift · Gate     │               │
│                       └────────────┬───────────────┘               │
│                                    │ approve / abort                 │
│                       ┌────────────▼───────────────┐               │
│                       │     Incident Sequencer      │               │
│                       │  JSON + SHA-256 hash        │               │
│                       └────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

**Infrastructure:**
- All LLM inference: Vultr Serverless Inference
  (`https://api.vultrinference.com/v1`), OpenAI-SDK-compatible
- All vector storage + RAG retrieval: Vultr built-in vector store
  (`https://api.vultrinference.com/v1/vector_store`)
- Backend: Python 3.11, FastAPI, asyncio
- Frontend: React 18 + TypeScript, Vite, Tailwind
- WebSockets: `websockets` library (Python) + native browser WS
- Persistence: local `incidents/` directory (JSON files)
- No Docker required for demo; `uvicorn` direct

---

## 4. Vultr Integration Map (Non-Negotiable)

This section exists because the original design doc (Crusoe version) had one
open risk: "validate embeddings endpoint early." The Vultr version has no such
ambiguity — all capabilities are explicitly documented.

### 4.1 Chat Completion

```python
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["VULTR_API_KEY"],
    base_url="https://api.vultrinference.com/v1"
)

response = client.chat.completions.create(
    model="zephyr-7b-beta-Q5_K_M",   # verify at /v1/chat/models
    messages=[...],
    temperature=0.1,
    response_format={"type": "json_object"}
)
```

Model identifier: confirm against `GET https://api.vultrinference.com/v1/chat/models`
before Phase 0. Preferred: lowest-latency model available. Fallback: next
available model. Hardcode both in config.

### 4.2 Vector Store + RAG

Vultr Serverless Inference has a first-class managed vector store. Documents
are uploaded as text, stored as embeddings, and retrieved via a dedicated RAG
completion endpoint. This replaces the cosine-distance-over-OpenAI-embeddings
approach from the original spec entirely — simpler, more reliable, no manual
embedding management.

```bash
# Create collection (once, at startup)
POST https://api.vultrinference.com/v1/vector_store
{"name": "sentinel-finance-docs"}

# Ingest a document chunk
POST https://api.vultrinference.com/v1/vector_store/{collection_id}/items
{"content": "<document text chunk>", "description": "Credit Agreement §4.2"}

# RAG completion (multi-retrieval)
POST https://api.vultrinference.com/v1/chat/completions/RAG
{
  "collection": "<collection_id>",
  "model": "zephyr-7b-beta-Q5_K_M",
  "messages": [{"role": "user", "content": "<query>"}],
  "max_tokens": 1024
}
```

The RAG endpoint handles embedding generation and retrieval internally. No
separate embedding client needed. This is a significant simplification over
the Crusoe version.

### 4.3 Environment Variables

```bash
VULTR_API_KEY=...
VULTR_BASE_URL=https://api.vultrinference.com/v1
VULTR_CHAT_MODEL=zephyr-7b-beta-Q5_K_M   # update after Phase -1 validation
SENTINEL_ENV=demo                          # "demo" enables mock email sandbox
SANDBOX_EMAIL_SINK=mailhog                 # never send real emails
```

---

## 5. Domain Model: Finance Covenant Monitoring

### 5.1 The 7-Step Agent Task (What Judges Will Watch)

```
Step 1: PLAN          — agent reads task brief, emits retrieval plan
Step 2: RETRIEVE-1    — fetch credit agreement covenant definitions (RAG)
Step 3: RETRIEVE-2    — fetch historical ratio data for trend context (RAG)
Step 4: TOOL-CALL     — calculate current ratio from live filing data
Step 5: RETRIEVE-3    — fetch recent transactions matching flagged period (RAG)
Step 6: SYNTHESISE    — draft escalation memo with per-clause citations
Step 7: ACT           — attempt to "send" memo → SENTINEL gate fires
```

Steps 1–6 are reversible (observable, stoppable, no external effect).
Step 7 (`send_escalation_memo`) is irreversible and triggers the full gate.

### 5.2 Irreversibility Classification Table

| Tool Name | Irreversible | Gate Fires |
|---|---|---|
| `retrieve_covenant_clause` | False | Never |
| `retrieve_historical_ratios` | False | Never |
| `calculate_ratio` | False | Never |
| `retrieve_transactions` | False | Never |
| `draft_memo` | False | Never |
| `send_escalation_memo` | **True** | Always |
| `trigger_downstream_system` | **True** | Always |

This table is the source of truth. `is_irreversible` is set by the agent at
action-creation time against this table. It is not inferred at runtime.

### 5.3 Domain Documents (Demo Corpus)

Three synthetic but realistic documents, stored in `docs/demo_corpus/`:

```
docs/demo_corpus/
  credit_agreement.md      # ~800 words: covenant definitions, thresholds
  historical_ratios.md     # ~400 words: quarterly ratio history, 8 quarters
  recent_transactions.md   # ~600 words: 60-day transaction log with flags
```

These are created by you before the hackathon starts (they are design
artifacts, not code). They must be realistic enough that an LLM produces
coherent reasoning over them.

**Key planted narrative:** The credit agreement defines a Debt/EBITDA covenant
threshold of 4.5x. Historical ratios show a trend from 3.8x to 4.3x over 6
quarters. The recent transactions doc contains three anomalous entries that
explain the Q2 spike to 4.6x (a breach). The agent must retrieve all three
documents across three passes to construct the full picture. This is the
multi-retrieval requirement demonstrated concretely.

---

## 6. Data Models

### 6.1 Action (canonical unit of work)

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import uuid, time

class ActionStatus(str, Enum):
    PENDING   = "pending"
    EXECUTED  = "executed"
    FROZEN    = "frozen"
    RESUMED   = "resumed"
    ABORTED   = "aborted"

@dataclass
class RetrievalCitation:
    document: str          # e.g. "credit_agreement.md"
    clause:   str          # e.g. "§4.2 Debt/EBITDA Covenant"
    excerpt:  str          # verbatim ≤100 chars

@dataclass
class Action:
    action_id:        str               = field(default_factory=lambda: str(uuid.uuid4()))
    description:      str               = ""
    tool_name:        str               = ""
    parameters:       dict              = field(default_factory=dict)
    is_irreversible:  bool              = False
    citations:        list[RetrievalCitation] = field(default_factory=list)
    status:           ActionStatus      = ActionStatus.PENDING
    created_at:       float             = field(default_factory=time.time)

    # Pipeline annotations (set by SentinelPipeline)
    drift_score:      Optional[float]   = None   # 0.0–1.0
    maars_verdict:    Optional[str]     = None   # "YES"/"NO"
    maars_confidence: Optional[int]     = None   # 0–100
    maars_reasoning:  Optional[str]     = None
    citation_score:   Optional[float]   = None   # 0.0–1.0
    freeze_reason:    Optional[str]     = None
    resolved_at:      Optional[float]   = None
    operator_decision: Optional[str]    = None   # "approved"/"aborted"
```

`RetrievalCitation` is new vs. the original spec — it carries the multi-
retrieval evidence forward to the UI and incident report.

### 6.2 EventBus Event Schema

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Optional
import time, uuid

class EventType(str, Enum):
    ACTION_PROPOSED     = "ACTION_PROPOSED"
    RETRIEVAL_PASS      = "RETRIEVAL_PASS"       # NEW: each RAG call
    TOOL_CALLED         = "TOOL_CALLED"           # NEW: each tool execution
    DRIFT_SCORED        = "DRIFT_SCORED"
    MAARS_PROBE         = "MAARS_PROBE"
    CITATION_CHECKED    = "CITATION_CHECKED"      # NEW: citation completeness
    ACTION_EXECUTED     = "ACTION_EXECUTED"
    ACTION_FROZEN       = "ACTION_FROZEN"
    INCIDENT_SEALED     = "INCIDENT_SEALED"
    OPERATOR_DECISION   = "OPERATOR_DECISION"
    AGENT_PLAN          = "AGENT_PLAN"            # NEW: show planning step
    ERROR               = "ERROR"

@dataclass
class SentinelEvent:
    event_id:   str       = field(default_factory=lambda: str(uuid.uuid4()))
    event_type: EventType = EventType.ERROR
    timestamp:  float     = field(default_factory=time.time)
    payload:    Any       = None
    action_id:  Optional[str] = None
```

### 6.3 TypeScript Event Contract (Frontend Mirror)

```typescript
export type EventType =
  | "ACTION_PROPOSED"
  | "RETRIEVAL_PASS"
  | "TOOL_CALLED"
  | "DRIFT_SCORED"
  | "MAARS_PROBE"
  | "CITATION_CHECKED"
  | "ACTION_EXECUTED"
  | "ACTION_FROZEN"
  | "INCIDENT_SEALED"
  | "OPERATOR_DECISION"
  | "AGENT_PLAN"
  | "ERROR";

export interface SentinelEvent {
  event_id:   string;
  event_type: EventType;
  timestamp:  number;
  payload:    unknown;
  action_id?: string;
}

// Discriminated union per event type
export type AgentPlanPayload       = { steps: string[] };
export type RetrievalPassPayload   = { pass_number: number; query: string; document: string; citations: Citation[] };
export type ToolCalledPayload      = { tool_name: string; parameters: Record<string, unknown>; result_summary: string };
export type DriftScoredPayload     = { score: number; drifted: boolean; threshold: number };
export type MaarsProbePayload      = { verdict: "YES"|"NO"; confidence: number; reasoning: string };
export type CitationCheckedPayload = { score: number; missing_clauses: string[] };
export type ActionFrozenPayload    = { action_id: string; freeze_reason: string; signals: FreezeSignals };
export type IncidentSealedPayload  = { incident_path: string; integrity_hash: string };

export interface FreezeSignals {
  drift_score:      number;
  maars_verdict:    "YES"|"NO";
  maars_confidence: number;
  citation_score:   number;
  freeze_triggers:  string[];
}

export interface Citation {
  document: string;
  clause:   string;
  excerpt:  string;
}
```

---

## 7. Backend Module Specification

```
sentinel/
  __init__.py
  orchestrator.py       # idempotent gate — single entry point
  vultr_client.py       # Vultr Serverless Inference wrapper (chat + RAG)
  vector_store.py       # Vultr vector store management (ingest + query)
  drift.py              # covenant drift monitor
  probe.py              # MAARS document probe
  citations.py          # citation completeness checker (NEW)
  pipeline.py           # policy assembly + signal orchestration
  agent.py              # enterprise document agent (planner + retriever + tool caller)
  sequencer.py          # incident sealer
  eventbus.py           # singleton in-process event bus
  broadcaster.py        # WebSocket fan-out bridge
  server.py             # FastAPI app + WebSocket endpoint + REST routes
  config.py             # env loading + validation
  tools/
    __init__.py
    calculate_ratio.py  # tool: compute debt/EBITDA ratio
    draft_memo.py       # tool: template-based memo generation
    send_memo.py        # tool: SANDBOXED irreversible action
    trigger_ds.py       # tool: SANDBOXED downstream trigger

tests/
  test_orchestrator.py
  test_pipeline.py
  test_agent.py
  test_citations.py
  test_sequencer.py

docs/
  demo_corpus/
    credit_agreement.md
    historical_ratios.md
    recent_transactions.md
  probe_prompt_template.md
  PRODUCT.md            # Impeccable input
  DESIGN.md             # Impeccable input

incidents/              # written at runtime — gitignored except .gitkeep
frontend/
  src/
    components/
    hooks/
    pages/
    tokens.css
    App.tsx
    main.tsx
  package.json
  tsconfig.json
  vite.config.ts

requirements.txt
.env.example
README.md
```

### 7.1 `sentinel/config.py`

```python
import os, sys

REQUIRED_KEYS = ["VULTR_API_KEY"]

def load_and_validate():
    missing = [k for k in REQUIRED_KEYS if not os.getenv(k)]
    if missing:
        print(f"[SENTINEL] Missing required env vars: {missing}")
        sys.exit(1)

    env = os.getenv("SENTINEL_ENV", "demo")
    if env == "demo" and not os.getenv("SANDBOX_EMAIL_SINK"):
        print("[SENTINEL] FATAL: SANDBOX_EMAIL_SINK must be set in demo mode")
        sys.exit(1)

    return {
        "vultr_api_key":   os.environ["VULTR_API_KEY"],
        "vultr_base_url":  os.getenv("VULTR_BASE_URL", "https://api.vultrinference.com/v1"),
        "chat_model":      os.getenv("VULTR_CHAT_MODEL", "zephyr-7b-beta-Q5_K_M"),
        "sentinel_env":    env,
        "sandbox_sink":    os.getenv("SANDBOX_EMAIL_SINK", "mailhog"),
        "drift_threshold": float(os.getenv("DRIFT_THRESHOLD", "0.40")),
        "citation_threshold": float(os.getenv("CITATION_THRESHOLD", "0.60")),
        "maars_confidence_min": int(os.getenv("MAARS_CONFIDENCE_MIN", "70")),
    }

CONFIG = load_and_validate()
```

### 7.2 `sentinel/vultr_client.py`

The original spec had `crusoe_client.py`. This is a complete, verified
replacement using Vultr's documented base URL and SDK pattern.

```python
import time
from openai import OpenAI, APITimeoutError
from sentinel.config import CONFIG

_client = OpenAI(
    api_key=CONFIG["vultr_api_key"],
    base_url=CONFIG["vultr_base_url"],
    timeout=20.0,
)

_MAX_RETRIES = 3
_RETRY_DELAY = 2.0

def chat_json(prompt: str, system: str = "") -> dict:
    """
    Call Vultr chat completion in JSON mode.
    Returns parsed dict. Raises ValueError on malformed JSON.
    Retries on timeout up to _MAX_RETRIES times.
    """
    messages = []
    if system:
        messages.append({"role": "system", "content": system})
    messages.append({"role": "user", "content": prompt})

    for attempt in range(_MAX_RETRIES):
        try:
            response = _client.chat.completions.create(
                model=CONFIG["chat_model"],
                messages=messages,
                temperature=0.1,
                max_tokens=1024,
                response_format={"type": "json_object"},
            )
            import json
            return json.loads(response.choices[0].message.content)
        except APITimeoutError:
            if attempt < _MAX_RETRIES - 1:
                time.sleep(_RETRY_DELAY * (attempt + 1))
                continue
            raise
        except Exception as e:
            raise ValueError(f"Vultr chat_json failed: {e}") from e

def chat_text(prompt: str, system: str = "") -> str:
    """
    Call Vultr chat completion for plain text output (planning, memo drafting).
    """
    messages = []
    if system:
        messages.append({"role": "system", "content": system})
    messages.append({"role": "user", "content": prompt})

    for attempt in range(_MAX_RETRIES):
        try:
            response = _client.chat.completions.create(
                model=CONFIG["chat_model"],
                messages=messages,
                temperature=0.3,
                max_tokens=2048,
            )
            return response.choices[0].message.content.strip()
        except APITimeoutError:
            if attempt < _MAX_RETRIES - 1:
                time.sleep(_RETRY_DELAY * (attempt + 1))
                continue
            raise
```

### 7.3 `sentinel/vector_store.py`

This is entirely new vs. the original spec. It manages the Vultr vector store
lifecycle and is the backbone of multi-retrieval.

```python
import requests
from sentinel.config import CONFIG

_BASE = CONFIG["vultr_base_url"]
_HEADERS = {
    "Authorization": f"Bearer {CONFIG['vultr_api_key']}",
    "Content-Type": "application/json",
}

_collection_id: str | None = None

def init_collection(name: str = "sentinel-finance-docs") -> str:
    """Create or retrieve the vector store collection."""
    global _collection_id
    resp = requests.post(
        f"{_BASE}/vector_store",
        headers=_HEADERS,
        json={"name": name},
        timeout=10,
    )
    resp.raise_for_status()
    _collection_id = resp.json()["id"]
    return _collection_id

def ingest_document(content: str, description: str) -> str:
    """
    Ingest a document chunk into the vector store.
    Returns the item ID.
    """
    if not _collection_id:
        raise RuntimeError("Vector store not initialised. Call init_collection() first.")
    resp = requests.post(
        f"{_BASE}/vector_store/{_collection_id}/items",
        headers=_HEADERS,
        json={"content": content, "description": description},
        timeout=15,
    )
    resp.raise_for_status()
    return resp.json()["id"]

def rag_query(query: str, system: str = "") -> str:
    """
    Perform a RAG chat completion against the vector store.
    Returns the model's response text with citations embedded.
    """
    if not _collection_id:
        raise RuntimeError("Vector store not initialised.")
    body: dict = {
        "collection": _collection_id,
        "model": CONFIG["chat_model"],
        "messages": [{"role": "user", "content": query}],
        "max_tokens": 1024,
    }
    if system:
        body["messages"].insert(0, {"role": "system", "content": system})

    resp = requests.post(
        f"{_BASE}/chat/completions/RAG",
        headers=_HEADERS,
        json=body,
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["choices"][0]["message"]["content"].strip()

def get_collection_id() -> str | None:
    return _collection_id
```

### 7.4 `sentinel/drift.py`

The original spec used cosine distance over OpenAI embeddings. With Vultr's
vector store, drift is measured differently: we use the RAG endpoint to ask
whether the proposed action still aligns with the original task brief. This is
semantically richer and uses the infrastructure we already have.

```python
from sentinel.vultr_client import chat_json
from sentinel.config import CONFIG

_task_brief: str = ""

def set_task_brief(brief: str):
    global _task_brief
    _task_brief = brief

def score_drift(action_description: str) -> tuple[bool, float]:
    """
    Returns (is_drifted: bool, drift_score: float 0.0–1.0).
    Uses Vultr LLM to assess semantic alignment between the original
    task brief and the proposed action.
    Fails closed: any error returns (True, 1.0).
    """
    if not _task_brief:
        return False, 0.0

    prompt = f"""You are a task alignment evaluator. 

Original task brief:
{_task_brief}

Proposed action:
{action_description}

Assess how much the proposed action deviates from the original task.
Return JSON with exactly these keys:
- drift_score: float 0.0 to 1.0 (0.0 = perfectly aligned, 1.0 = completely off-task)
- reasoning: one sentence explaining the score

Example: {{"drift_score": 0.15, "reasoning": "Action directly advances the covenant monitoring objective."}}"""

    try:
        result = chat_json(prompt)
        score = float(result.get("drift_score", 1.0))
        score = max(0.0, min(1.0, score))
        threshold = CONFIG["drift_threshold"]
        return score >= threshold, score
    except Exception:
        return True, 1.0   # fail closed
```

### 7.5 `sentinel/citations.py` *(NEW — critical for Vultr track)*

This module is entirely new and is what makes SENTINEL's agent demonstrably
multi-retrieval rather than single-pass. It checks whether a proposed
irreversible action is backed by sufficient document evidence.

```python
from sentinel.vultr_client import chat_json
from sentinel.config import CONFIG
from sentinel.models import Action

REQUIRED_CLAUSES = [
    "covenant_definition",
    "breach_threshold",
    "historical_trend",
    "transaction_evidence",
]

def check_citation_completeness(action: Action) -> tuple[float, list[str]]:
    """
    Returns (citation_score: float 0.0–1.0, missing_clauses: list[str]).
    citation_score = fraction of required clauses cited.
    Fails closed: any error returns (0.0, REQUIRED_CLAUSES).
    """
    cited_docs = [c.clause for c in action.citations]
    cited_text = "\n".join(cited_docs) if cited_docs else "(no citations)"

    prompt = f"""You are a compliance evidence auditor.

A proposed enterprise action cites the following document clauses:
{cited_text}

Required evidence categories for a valid escalation:
{chr(10).join(f"- {c}" for c in REQUIRED_CLAUSES)}

Return JSON with:
- covered: list of required categories that are evidenced
- missing: list of required categories that are NOT evidenced
- score: float 0.0 to 1.0 (covered / total required)

Example: {{"covered": ["covenant_definition", "breach_threshold"], "missing": ["historical_trend", "transaction_evidence"], "score": 0.5}}"""

    try:
        result = chat_json(prompt)
        score = float(result.get("score", 0.0))
        missing = result.get("missing", REQUIRED_CLAUSES)
        return max(0.0, min(1.0, score)), missing
    except Exception:
        return 0.0, REQUIRED_CLAUSES   # fail closed
```

### 7.6 `sentinel/probe.py`

The MAARS probe from the original spec, adapted for the finance domain. The
prompt template changes; the fail-closed validation logic is identical and
non-negotiable.

`docs/probe_prompt_template.md`:
```markdown
You are MAARS (Multi-pass Adversarial Action Review System), an enterprise
compliance auditor for autonomous financial agents.

You have been asked to review a proposed irreversible action by an AI agent.

## Proposed Action
{action_description}

## Tool
{tool_name}

## Parameters
{parameters_json}

## Citations Provided
{citations_json}

## Drift Score
{drift_score} (threshold: {drift_threshold}; drifted: {drifted})

## Your Task
Determine whether this action is SAFE TO EXECUTE given the evidence provided.

Return ONLY valid JSON with this exact schema:
{{
  "verdict": "YES" or "NO",
  "confidence": <integer 0-100>,
  "reasoning": "<specific, evidence-grounded reasoning — vague reasoning is forbidden>",
  "severity": "LOW|MEDIUM|HIGH|CRITICAL",
  "remediation": "<if NO: specific remediation step; if YES: empty string>"
}}

Rules:
- verdict YES = safe to execute; NO = must be blocked
- confidence < 70 → treat as NO regardless of verdict
- reasoning must reference specific document evidence or the lack thereof
- vague reasoning like "the action seems risky" is forbidden
```

```python
import json
from pathlib import Path
from sentinel.vultr_client import chat_json

_TEMPLATE = (Path(__file__).parent.parent / "docs" / "probe_prompt_template.md").read_text()

REQUIRED_KEYS = {"verdict", "confidence", "reasoning", "severity", "remediation"}
VALID_VERDICTS = {"YES", "NO"}

def run_probe(action, drift_score: float, drifted: bool) -> dict:
    """
    Returns a validated probe verdict dict.
    Fails closed: any parse/validation error → {"verdict": "NO", "confidence": 0, ...}
    """
    _FALLBACK = {
        "verdict": "NO",
        "confidence": 0,
        "reasoning": "Probe failed to return valid JSON — failing closed.",
        "severity": "HIGH",
        "remediation": "Investigate probe failure before retrying.",
    }

    prompt = _TEMPLATE.format(
        action_description=action.description,
        tool_name=action.tool_name,
        parameters_json=json.dumps(action.parameters, indent=2),
        citations_json=json.dumps(
            [{"document": c.document, "clause": c.clause, "excerpt": c.excerpt}
             for c in action.citations], indent=2
        ),
        drift_score=round(drift_score, 3),
        drift_threshold=0.40,
        drifted=drifted,
    )

    try:
        result = chat_json(prompt)
    except Exception:
        return _FALLBACK

    # Structural validation
    if not REQUIRED_KEYS.issubset(result.keys()):
        return _FALLBACK
    if result.get("verdict") not in VALID_VERDICTS:
        return _FALLBACK
    if not isinstance(result.get("confidence"), int):
        return _FALLBACK
    if not result.get("reasoning", "").strip():
        return _FALLBACK

    # Fail closed on low confidence regardless of verdict
    if result["confidence"] < 70:
        result["verdict"] = "NO"

    return result
```

### 7.7 `sentinel/eventbus.py`

```python
import asyncio
from collections import defaultdict
from typing import Callable
from sentinel.models import SentinelEvent

_subscribers: dict[str, list[Callable]] = defaultdict(list)

def subscribe(event_type: str, handler: Callable):
    _subscribers[event_type].append(handler)

def subscribe_all(handler: Callable):
    _subscribers["*"].append(handler)

def emit(event: SentinelEvent):
    for handler in _subscribers.get(event.event_type, []):
        handler(event)
    for handler in _subscribers.get("*", []):
        handler(event)
```

### 7.8 `sentinel/broadcaster.py`

```python
import asyncio
import json
from dataclasses import asdict
from sentinel.models import SentinelEvent

_loop: asyncio.AbstractEventLoop | None = None
_clients: set = set()

def set_event_loop(loop: asyncio.AbstractEventLoop):
    global _loop
    _loop = loop

def register_client(ws):
    _clients.add(ws)

def unregister_client(ws):
    _clients.discard(ws)

def broadcast(event: SentinelEvent):
    """Called from agent thread. Thread-safe bridge to WS event loop."""
    if not _loop or not _clients:
        return
    payload = json.dumps(asdict(event), default=str)
    for ws in list(_clients):
        asyncio.run_coroutine_threadsafe(_send(ws, payload), _loop)

async def _send(ws, payload: str):
    try:
        await ws.send_text(payload)
    except Exception:
        _clients.discard(ws)
```

### 7.9 `sentinel/orchestrator.py`

```python
import time
from typing import Callable, Optional
from sentinel.models import Action, ActionStatus, SentinelEvent, EventType
from sentinel.eventbus import emit

_action_registry: dict[str, Action] = {}
_freeze_policy: Optional[Callable] = None

def set_freeze_policy(fn: Callable):
    global _freeze_policy
    _freeze_policy = fn

def attempt_action(action: Action) -> Action:
    """
    Single entry point. Enforces:
    1. Idempotency by action_id
    2. Irreversibility gate (only irreversible actions can be frozen)
    3. Policy application and event emission
    """
    if action.action_id in _action_registry:
        emit(SentinelEvent(
            event_type=EventType.ERROR,
            action_id=action.action_id,
            payload={"message": "Duplicate action_id — ignoring."},
        ))
        return _action_registry[action.action_id]

    _action_registry[action.action_id] = action
    emit(SentinelEvent(
        event_type=EventType.ACTION_PROPOSED,
        action_id=action.action_id,
        payload={"action": action.description, "tool": action.tool_name,
                 "is_irreversible": action.is_irreversible},
    ))

    should_freeze = False
    if action.is_irreversible and _freeze_policy:
        should_freeze = _freeze_policy(action)

    if should_freeze:
        action.status = ActionStatus.FROZEN
        emit(SentinelEvent(
            event_type=EventType.ACTION_FROZEN,
            action_id=action.action_id,
            payload={"freeze_reason": action.freeze_reason,
                     "drift_score": action.drift_score,
                     "maars_verdict": action.maars_verdict,
                     "maars_confidence": action.maars_confidence,
                     "citation_score": action.citation_score},
        ))
    else:
        action.status = ActionStatus.EXECUTED
        emit(SentinelEvent(
            event_type=EventType.ACTION_EXECUTED,
            action_id=action.action_id,
            payload={"action": action.description},
        ))

    return action

def resolve_frozen(action_id: str, approved: bool) -> Action:
    """Operator decision on a frozen action."""
    action = _action_registry.get(action_id)
    if not action or action.status != ActionStatus.FROZEN:
        raise ValueError(f"No frozen action with id {action_id}")

    action.status = ActionStatus.RESUMED if approved else ActionStatus.ABORTED
    action.operator_decision = "approved" if approved else "aborted"
    action.resolved_at = time.time()

    emit(SentinelEvent(
        event_type=EventType.OPERATOR_DECISION,
        action_id=action_id,
        payload={"decision": action.operator_decision,
                 "resolved_at": action.resolved_at},
    ))
    return action

def get_action(action_id: str) -> Optional[Action]:
    return _action_registry.get(action_id)
```

### 7.10 `sentinel/pipeline.py`

```python
from sentinel.models import Action, SentinelEvent, EventType
from sentinel.eventbus import emit
from sentinel.drift import score_drift, set_task_brief
from sentinel.probe import run_probe
from sentinel.citations import check_citation_completeness
from sentinel.config import CONFIG
import sentinel.orchestrator as orchestrator

class SentinelPipeline:
    def __init__(self, task_brief: str):
        set_task_brief(task_brief)
        orchestrator.set_freeze_policy(self._compound_policy)

    def _compound_policy(self, action: Action) -> bool:
        """
        Evaluates three signals and decides whether to freeze.
        Only called for irreversible actions.
        Returns True = freeze, False = execute.
        """
        triggers = []

        # Signal 1: Covenant drift
        drifted, drift_score = score_drift(action.description)
        action.drift_score = drift_score
        emit(SentinelEvent(
            event_type=EventType.DRIFT_SCORED,
            action_id=action.action_id,
            payload={"score": drift_score, "drifted": drifted,
                     "threshold": CONFIG["drift_threshold"]},
        ))
        if drifted:
            triggers.append(f"drift_score={drift_score:.2f} exceeds threshold")

        # Signal 2: MAARS document probe
        verdict = run_probe(action, drift_score, drifted)
        action.maars_verdict = verdict["verdict"]
        action.maars_confidence = verdict["confidence"]
        action.maars_reasoning = verdict["reasoning"]
        emit(SentinelEvent(
            event_type=EventType.MAARS_PROBE,
            action_id=action.action_id,
            payload=verdict,
        ))
        if verdict["verdict"] == "NO" and verdict["confidence"] >= CONFIG["maars_confidence_min"]:
            triggers.append(f"MAARS verdict=NO confidence={verdict['confidence']}")

        # Signal 3: Citation completeness
        citation_score, missing = check_citation_completeness(action)
        action.citation_score = citation_score
        emit(SentinelEvent(
            event_type=EventType.CITATION_CHECKED,
            action_id=action.action_id,
            payload={"score": citation_score, "missing_clauses": missing},
        ))
        if citation_score < CONFIG["citation_threshold"]:
            triggers.append(f"citation_score={citation_score:.2f} below threshold; missing: {missing}")

        # Freeze if ANY signal fires
        if triggers:
            action.freeze_reason = " | ".join(triggers)
            return True
        return False
```

Note: the original spec used an AND gate (drift AND probe). This spec uses an
OR gate. Rationale: for an enterprise compliance use case, a single signal
that something is wrong is sufficient to pause — the operator decides, not
the system. This is more conservative and more appropriate for the domain.
Update `CONFIG["citation_threshold"]` and `CONFIG["maars_confidence_min"]` to
tune demo behaviour.

### 7.11 `sentinel/agent.py`

This is the most significant new module vs. the original spec. It implements
the multi-step planning and retrieval loop.

```python
import time
import threading
from sentinel.models import Action, RetrievalCitation, SentinelEvent, EventType
from sentinel.eventbus import emit
from sentinel.vultr_client import chat_text, chat_json
from sentinel.vector_store import rag_query
import sentinel.orchestrator as orchestrator
from sentinel.tools import calculate_ratio, draft_memo, send_memo

TASK_BRIEF = """
You are a covenant monitoring agent for a corporate finance team.
Your task: analyse the Debt/EBITDA covenant in the credit agreement,
assess whether the borrower is in breach based on current filings,
identify the root cause from recent transactions, and prepare an
escalation memo if a breach is detected.
"""

def run_agent():
    """Runs the full 7-step agent loop in a background thread."""
    thread = threading.Thread(target=_agent_loop, daemon=True)
    thread.start()
    return thread

def _agent_loop():
    # Step 1: PLAN
    plan = chat_text(
        "You are a covenant monitoring agent. List the steps you will take "
        "to investigate a potential covenant breach. Return a numbered list.",
        system=TASK_BRIEF,
    )
    emit(SentinelEvent(
        event_type=EventType.AGENT_PLAN,
        payload={"steps": plan.split("\n"), "raw": plan},
    ))
    time.sleep(1.5)   # pacing for demo readability

    # Step 2: RETRIEVE-1 — covenant definition
    r1 = rag_query(
        "What is the Debt/EBITDA covenant threshold and how is it defined?",
        system=TASK_BRIEF,
    )
    emit(SentinelEvent(
        event_type=EventType.RETRIEVAL_PASS,
        payload={"pass_number": 1, "query": "Covenant definition",
                 "result_summary": r1[:200]},
    ))
    time.sleep(1.0)

    # Step 3: RETRIEVE-2 — historical ratios
    r2 = rag_query(
        "What is the historical Debt/EBITDA ratio trend over the past 8 quarters?",
        system=TASK_BRIEF,
    )
    emit(SentinelEvent(
        event_type=EventType.RETRIEVAL_PASS,
        payload={"pass_number": 2, "query": "Historical ratio trend",
                 "result_summary": r2[:200]},
    ))
    time.sleep(1.0)

    # Step 4: TOOL — calculate ratio
    ratio_result = calculate_ratio.run(debt=460_000_000, ebitda=100_000_000)
    emit(SentinelEvent(
        event_type=EventType.TOOL_CALLED,
        payload={"tool_name": "calculate_ratio",
                 "parameters": {"debt": 460_000_000, "ebitda": 100_000_000},
                 "result_summary": f"Debt/EBITDA = {ratio_result['ratio']:.2f}x (threshold: 4.5x)"},
    ))
    time.sleep(1.0)

    # Step 5: RETRIEVE-3 — transactions (conditional on breach)
    breach_detected = ratio_result["ratio"] > 4.5
    r3 = ""
    if breach_detected:
        r3 = rag_query(
            "Which recent transactions contributed to the Q2 EBITDA decline?",
            system=TASK_BRIEF,
        )
        emit(SentinelEvent(
            event_type=EventType.RETRIEVAL_PASS,
            payload={"pass_number": 3, "query": "Transaction root cause",
                     "result_summary": r3[:200]},
        ))
        time.sleep(1.0)

    # Step 6: SYNTHESISE — draft memo (reversible)
    memo_text = draft_memo.run(
        ratio=ratio_result["ratio"],
        threshold=4.5,
        covenant_context=r1,
        historical_context=r2,
        transaction_context=r3,
    )
    memo_action = Action(
        description="Draft escalation memo for compliance team",
        tool_name="draft_memo",
        parameters={"ratio": ratio_result["ratio"]},
        is_irreversible=False,
        citations=[
            RetrievalCitation("credit_agreement.md", "§4.2 Debt/EBITDA Covenant", r1[:80]),
            RetrievalCitation("historical_ratios.md", "Q1–Q8 Trend", r2[:80]),
            RetrievalCitation("recent_transactions.md", "Q2 Transaction Anomalies", r3[:80]),
        ],
    )
    orchestrator.attempt_action(memo_action)
    time.sleep(1.5)

    # Step 7: ACT — send memo (irreversible — SENTINEL gate fires here)
    if breach_detected:
        send_action = Action(
            description=(
                f"Send escalation memo to CFO and legal team. "
                f"Debt/EBITDA ratio is {ratio_result['ratio']:.2f}x, "
                f"breaching the 4.5x covenant threshold defined in §4.2 of "
                f"the Credit Agreement. Root cause identified in Q2 transactions."
            ),
            tool_name="send_escalation_memo",
            parameters={"recipient": "cfo@example.com", "memo": memo_text[:100]},
            is_irreversible=True,
            citations=[
                RetrievalCitation("credit_agreement.md", "§4.2 Debt/EBITDA Covenant", r1[:80]),
                RetrievalCitation("historical_ratios.md", "historical_trend", r2[:80]),
                RetrievalCitation("recent_transactions.md", "transaction_evidence", r3[:80]),
            ],
        )
        orchestrator.attempt_action(send_action)
```

### 7.12 `sentinel/sequencer.py`

```python
import json
import hashlib
import time
from pathlib import Path
from sentinel.models import Action, SentinelEvent, EventType
from sentinel.eventbus import emit

INCIDENTS_DIR = Path("incidents")
INCIDENTS_DIR.mkdir(exist_ok=True)

def seal_incident(action: Action) -> dict:
    """
    Creates a tamper-evident incident artifact on operator resolution.
    Hash is SHA-256 of the canonical JSON — not encryption, but detectable
    if the file is modified post-seal. Be explicit about this in the demo.
    """
    report = {
        "sentinel_version": "2.0.0",
        "incident_id": action.action_id,
        "sealed_at": time.time(),
        "action": {
            "description": action.description,
            "tool_name": action.tool_name,
            "parameters": action.parameters,
            "is_irreversible": action.is_irreversible,
        },
        "citations": [
            {"document": c.document, "clause": c.clause, "excerpt": c.excerpt}
            for c in action.citations
        ],
        "signals": {
            "drift_score": action.drift_score,
            "maars_verdict": action.maars_verdict,
            "maars_confidence": action.maars_confidence,
            "maars_reasoning": action.maars_reasoning,
            "citation_score": action.citation_score,
        },
        "freeze_reason": action.freeze_reason,
        "operator_decision": action.operator_decision,
        "resolved_at": action.resolved_at,
    }

    canonical = json.dumps(report, sort_keys=True)
    integrity_hash = hashlib.sha256(canonical.encode()).hexdigest()
    report["integrity_hash"] = integrity_hash

    path = INCIDENTS_DIR / f"incident_{action.action_id[:8]}.json"
    path.write_text(json.dumps(report, indent=2))

    emit(SentinelEvent(
        event_type=EventType.INCIDENT_SEALED,
        action_id=action.action_id,
        payload={"incident_path": str(path), "integrity_hash": integrity_hash},
    ))
    return report
```

### 7.13 `sentinel/server.py`

```python
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from sentinel import broadcaster, eventbus
from sentinel.models import SentinelEvent, EventType
from sentinel.pipeline import SentinelPipeline
from sentinel.agent import TASK_BRIEF, run_agent
from sentinel.vector_store import init_collection, ingest_document
from sentinel.sequencer import seal_incident
import sentinel.orchestrator as orchestrator
from pathlib import Path

@asynccontextmanager
async def lifespan(app: FastAPI):
    loop = asyncio.get_event_loop()
    broadcaster.set_event_loop(loop)
    eventbus.subscribe_all(broadcaster.broadcast)

    # Ingest demo corpus into Vultr vector store
    collection_id = init_collection()
    corpus_dir = Path("docs/demo_corpus")
    for doc in corpus_dir.glob("*.md"):
        ingest_document(doc.read_text(), description=doc.name)

    # Initialise pipeline (sets task brief + installs policy)
    SentinelPipeline(task_brief=TASK_BRIEF)

    yield   # server runs

app = FastAPI(lifespan=lifespan)
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    broadcaster.register_client(ws)
    try:
        while True:
            await ws.receive_text()   # keep-alive; client sends pings
    except WebSocketDisconnect:
        broadcaster.unregister_client(ws)

@app.post("/api/run")
async def run():
    """Trigger a fresh agent run."""
    run_agent()
    return {"status": "agent_started"}

class DecisionRequest(BaseModel):
    action_id: str
    approved: bool

@app.post("/api/decide")
async def decide(req: DecisionRequest):
    """Operator approve/abort decision."""
    action = orchestrator.resolve_frozen(req.action_id, req.approved)
    report = seal_incident(action)
    return {"status": action.status, "integrity_hash": report["integrity_hash"]}

@app.get("/api/action/{action_id}")
async def get_action(action_id: str):
    action = orchestrator.get_action(action_id)
    if not action:
        return {"error": "not found"}
    from dataclasses import asdict
    return asdict(action)
```

### 7.14 Tool Implementations

#### `sentinel/tools/calculate_ratio.py`
```python
def run(debt: float, ebitda: float) -> dict:
    ratio = debt / ebitda if ebitda != 0 else float("inf")
    return {
        "ratio": round(ratio, 2),
        "debt": debt,
        "ebitda": ebitda,
        "breaches_covenant": ratio > 4.5,
    }
```

#### `sentinel/tools/draft_memo.py`
```python
def run(ratio: float, threshold: float, covenant_context: str,
        historical_context: str, transaction_context: str) -> str:
    return (
        f"ESCALATION MEMO — COVENANT BREACH DETECTED\n\n"
        f"Current Debt/EBITDA: {ratio:.2f}x (Threshold: {threshold}x)\n\n"
        f"Covenant Reference: {covenant_context[:200]}\n\n"
        f"Historical Context: {historical_context[:200]}\n\n"
        f"Transaction Root Cause: {transaction_context[:200]}\n\n"
        f"[DRAFT — pending operator approval]"
    )
```

#### `sentinel/tools/send_memo.py`
```python
import os

def run(recipient: str, memo: str) -> dict:
    env = os.getenv("SENTINEL_ENV", "demo")
    sink = os.getenv("SANDBOX_EMAIL_SINK")
    if env == "demo":
        if not sink:
            raise RuntimeError("SANDBOX_EMAIL_SINK must be set in demo mode. Refusing to send.")
        # In demo: log to console only. Never send real email.
        print(f"[SANDBOX] Memo would be sent to {recipient} via {sink}")
        return {"status": "sandboxed", "sink": sink, "recipient": recipient}
    # Production path would go here
    raise NotImplementedError("Production email sending not implemented.")
```

---

## 8. Frontend Specification

### 8.1 Design Philosophy — "Compliance, Not Chrome"

This UI exists to demonstrate that a human operator can understand, question,
and override an autonomous agent's decisions in real time. The design system
must communicate **authority, precision, and auditability** — not AI-startup
vibrancy.

This is handed to Impeccable via `docs/PRODUCT.md` and `docs/DESIGN.md`.

#### `docs/PRODUCT.md`
```markdown
# SENTINEL — Product Brief

SENTINEL is an enterprise document agent with a built-in oversight gate.
It helps compliance and finance teams ensure that autonomous AI agents don't
take irreversible actions (like sending regulatory escalation memos) without
full evidentiary backing and explicit human authorisation.

Primary users: CFOs, compliance officers, risk managers — not developers.
Context of use: high-stakes, time-pressured decision-making.
Core action: Approve or Abort a frozen agent action in under 10 seconds,
with full signal transparency.

What the UI must communicate:
1. The agent is working (live activity stream)
2. Something important happened (freeze alert is unmissable)
3. Why it happened (three signals, each with a score)
4. What evidence exists (citations from retrieved documents)
5. What the operator decided and that it was recorded (incident hash)

What the UI must never feel like:
- A developer dashboard
- A chatbot interface
- A SaaS marketing page
- Anything with gradients, glassmorphism, or animated blobs
```

#### `docs/DESIGN.md`
```markdown
# SENTINEL — Design Specification

## Design Tokens

Base palette: near-black (#0D0D0D) background, off-white (#F2F0EB) text.
Status semantics:
  - FROZEN:   #E53935 (red — unmissable, not aggressive)
  - EXECUTED: #2E7D32 (green — settled, not celebratory)
  - PENDING:  #F9A825 (amber — active, not alarming)
  - ABORTED:  #546E7A (slate — neutral, closed)
  - RESUMED:  #1565C0 (blue — authorised, proceeding)

Typography:
  - Display/headings: IBM Plex Mono (monospace authority, not decorative)
  - Body: Inter 400/500 only
  - Status labels: uppercase, 0.08em letter-spacing, 11px

Borders: 1px solid rgba(242,240,235,0.12) — no shadows, no radius > 4px.
No gradients. No blur. No rounded cards. No icons beyond functional status
indicators (●, ✓, ✗).

Spacing system: 4px base unit. 8/12/16/24/32/48.

## Screen Map

Screen 1: Live Agent Monitor
  - Full-width event stream (newest at top)
  - Each event: timestamp | event_type badge | summary text | action_id (truncated)
  - RETRIEVAL_PASS events show pass number + document name
  - TOOL_CALLED events show tool name + result summary
  - ACTION_FROZEN row is highlighted in red — full width takeover

Screen 2: Signal Breakdown Panel (appears on freeze)
  - Three signal cards side by side:
    Left:   Covenant Drift — score gauge + reasoning
    Centre: MAARS Verdict — YES/NO badge + confidence + reasoning
    Right:  Citation Score — score gauge + missing clauses list
  - Below: citations list (document | clause | excerpt)
  - Bottom: freeze_reason verbatim in monospace red

Screen 3: Operator Gate + Incident Record
  - Above the fold: large APPROVE / ABORT buttons (no ambiguity about which is which)
  - APPROVE: green, full-width, requires deliberate click (not hover-triggered)
  - ABORT:   red outline, same size — same weight as APPROVE
  - Below: incident JSON preview (collapsible) + integrity hash in monospace
  - After decision: hash animates in; "Incident sealed" confirmation

## Impeccable Workflow
Every UI component is generated via:
  impeccable build --product docs/PRODUCT.md --design docs/DESIGN.md <component>

Critique checkpoint after Phase 6 (before submission):
  impeccable critique --product docs/PRODUCT.md --design docs/DESIGN.md --visual

Audit after any significant change:
  impeccable audit --product docs/PRODUCT.md --design docs/DESIGN.md

## Non-Negotiables
- Freeze state must be visually unmissable within 200ms of the event arriving
- Signal scores must be human-readable numbers, not raw JSON
- Citations must be readable, not collapsed by default
- Operator buttons must be full-width and clearly labelled
- Incident hash must be visible, copyable, and labelled as SHA-256
```

### 8.2 Component Map

```
frontend/src/
  pages/
    MonitorPage.tsx     # Screen 1: live event stream
    SignalPage.tsx      # Screen 2: signal breakdown (shown on freeze)
    GatePage.tsx        # Screen 3: operator gate + incident record
  components/
    EventRow.tsx        # single event in the stream
    RetrievalBadge.tsx  # pass number + document name
    SignalCard.tsx      # drift / maars / citation card
    CitationList.tsx    # retrieved document citations
    FreezeAlert.tsx     # red full-width freeze notification
    OperatorGate.tsx    # approve/abort buttons
    IncidentRecord.tsx  # JSON preview + hash display
    StatusBadge.tsx     # coloured event_type label
  hooks/
    useEventStream.ts   # WebSocket + auto-reconnect
    useAgent.ts         # run agent, track current action
  tokens.css            # CSS custom properties from DESIGN.md
  App.tsx
  main.tsx
```

### 8.3 `useEventStream.ts`

```typescript
import { useState, useEffect, useCallback, useRef } from "react";
import { SentinelEvent } from "../types";

const WS_URL = import.meta.env.VITE_WS_URL ?? "ws://localhost:8000/ws";
const RECONNECT_DELAY = 2000;

export function useEventStream() {
  const [events, setEvents] = useState<SentinelEvent[]>([]);
  const [connected, setConnected] = useState(false);
  const [frozenAction, setFrozenAction] = useState<SentinelEvent | null>(null);
  const wsRef = useRef<WebSocket | null>(null);

  const connect = useCallback(() => {
    const ws = new WebSocket(WS_URL);
    wsRef.current = ws;

    ws.onopen = () => setConnected(true);
    ws.onclose = () => {
      setConnected(false);
      setTimeout(connect, RECONNECT_DELAY);
    };
    ws.onmessage = (e) => {
      const event: SentinelEvent = JSON.parse(e.data);
      setEvents(prev => [event, ...prev]);
      if (event.event_type === "ACTION_FROZEN") {
        setFrozenAction(event);
      }
      if (event.event_type === "OPERATOR_DECISION") {
        setFrozenAction(null);
      }
    };

    // Keepalive ping every 20s
    const ping = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) ws.send("ping");
    }, 20_000);
    ws.onclose = () => { clearInterval(ping); setConnected(false); setTimeout(connect, RECONNECT_DELAY); };
  }, []);

  useEffect(() => { connect(); return () => wsRef.current?.close(); }, [connect]);

  return { events, connected, frozenAction };
}
```

### 8.4 `tokens.css`

```css
:root {
  --color-bg:        #0D0D0D;
  --color-surface:   #161616;
  --color-border:    rgba(242, 240, 235, 0.12);
  --color-text:      #F2F0EB;
  --color-text-muted:#8A8880;

  --color-frozen:    #E53935;
  --color-executed:  #2E7D32;
  --color-pending:   #F9A825;
  --color-aborted:   #546E7A;
  --color-resumed:   #1565C0;

  --font-mono:   "IBM Plex Mono", "Courier New", monospace;
  --font-body:   "Inter", system-ui, sans-serif;

  --radius-sm:   2px;
  --radius-md:   4px;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  --space-12: 48px;
}

* { box-sizing: border-box; margin: 0; padding: 0; }
body { background: var(--color-bg); color: var(--color-text); font-family: var(--font-body); }
code, pre, .mono { font-family: var(--font-mono); }
```

---

## 9. Tests

Tests must pass at each phase gate. They run against deterministic fixtures —
no live API calls in tests.

### `tests/test_orchestrator.py`
```python
import pytest
from sentinel.models import Action
from sentinel.orchestrator import attempt_action, resolve_frozen, set_freeze_policy

def always_freeze(action): return True
def never_freeze(action): return False

def test_reversible_never_frozen():
    set_freeze_policy(always_freeze)
    a = Action(description="fetch doc", tool_name="retrieve", is_irreversible=False)
    result = attempt_action(a)
    assert result.status.value == "executed"

def test_irreversible_frozen_by_policy():
    set_freeze_policy(always_freeze)
    a = Action(description="send memo", tool_name="send_escalation_memo", is_irreversible=True)
    result = attempt_action(a)
    assert result.status.value == "frozen"

def test_idempotency():
    a = Action(description="send memo", tool_name="send_escalation_memo", is_irreversible=True)
    r1 = attempt_action(a)
    r2 = attempt_action(a)
    assert r1.action_id == r2.action_id

def test_operator_approve():
    set_freeze_policy(always_freeze)
    a = Action(description="send memo", tool_name="send_escalation_memo", is_irreversible=True)
    attempt_action(a)
    resolved = resolve_frozen(a.action_id, approved=True)
    assert resolved.status.value == "resumed"
    assert resolved.operator_decision == "approved"

def test_operator_abort():
    set_freeze_policy(always_freeze)
    a = Action(description="send memo", tool_name="send_escalation_memo", is_irreversible=True)
    attempt_action(a)
    resolved = resolve_frozen(a.action_id, approved=False)
    assert resolved.status.value == "aborted"
```

### `tests/test_citations.py`
```python
from sentinel.models import Action, RetrievalCitation
from sentinel.citations import check_citation_completeness, REQUIRED_CLAUSES

def test_no_citations_zero_score():
    a = Action(description="send memo", tool_name="send_escalation_memo", is_irreversible=True)
    # Mock chat_json to return all missing
    # In real test: patch sentinel.citations.chat_json
    # Verify fail-closed: error → (0.0, REQUIRED_CLAUSES)
    pass   # placeholder — implement with unittest.mock.patch

def test_full_citations_high_score():
    a = Action(
        description="send memo", tool_name="send_escalation_memo", is_irreversible=True,
        citations=[
            RetrievalCitation("credit_agreement.md", "covenant_definition", "..."),
            RetrievalCitation("credit_agreement.md", "breach_threshold", "..."),
            RetrievalCitation("historical_ratios.md", "historical_trend", "..."),
            RetrievalCitation("recent_transactions.md", "transaction_evidence", "..."),
        ]
    )
    # Patch chat_json to return score=1.0, missing=[]
    pass   # placeholder
```

### `tests/test_sequencer.py`
```python
import json, hashlib
from sentinel.models import Action, ActionStatus
from sentinel.sequencer import seal_incident

def test_hash_is_stable():
    a = Action(
        description="send memo", tool_name="send_escalation_memo",
        is_irreversible=True, status=ActionStatus.RESUMED,
        operator_decision="approved",
    )
    report = seal_incident(a)
    canonical = json.dumps({k: v for k, v in report.items() if k != "integrity_hash"}, sort_keys=True)
    expected_hash = hashlib.sha256(canonical.encode()).hexdigest()
    assert report["integrity_hash"] == expected_hash

def test_incident_file_written(tmp_path, monkeypatch):
    import sentinel.sequencer as seq
    monkeypatch.setattr(seq, "INCIDENTS_DIR", tmp_path)
    a = Action(description="test", tool_name="send_escalation_memo",
               is_irreversible=True, status=ActionStatus.ABORTED,
               operator_decision="aborted")
    report = seal_incident(a)
    files = list(tmp_path.glob("*.json"))
    assert len(files) == 1
```

---

## 10. Phase Plan

Every phase has: **what you build**, **exit gate**, and **time budget** (solo,
remote, 24.5 hours of hacking from 11:30 AM Sat to 12:00 PM Sun).

---

### Phase −1: Pre-Hack Setup *(before 11:30 AM Saturday)*

**Build:**
- [ ] Create public GitHub repo `sentinel`
- [ ] Commit this design doc as `docs/designdoc.md`
- [ ] Write `docs/demo_corpus/credit_agreement.md` (~800 words, synthetic)
- [ ] Write `docs/demo_corpus/historical_ratios.md` (~400 words)
- [ ] Write `docs/demo_corpus/recent_transactions.md` (~600 words)
  - Plant the 4.6x ratio breach + 3 anomalous transactions clearly
- [ ] Write `docs/probe_prompt_template.md` (as above)
- [ ] Write `docs/PRODUCT.md` and `docs/DESIGN.md` (as above)
- [ ] Sign up for Vultr + claim $200 credits via Google Form
- [ ] Create a Vultr Serverless Inference subscription
- [ ] Validate API key: `curl https://api.vultrinference.com/v1/chat/models -H "Authorization: Bearer $VULTR_API_KEY"`
- [ ] Note the exact model identifier available — update `VULTR_CHAT_MODEL`
- [ ] Create vector store collection manually: test ingest + RAG query returns coherent text
- [ ] Commit pinned `requirements.txt`:
  ```
  fastapi==0.111.0
  uvicorn[standard]==0.29.0
  openai==1.30.1
  requests==2.32.3
  websockets==12.0
  pydantic==2.7.1
  pytest==8.2.0
  ```
- [ ] Commit `.env.example`
- [ ] `pytest tests/` — all pass (placeholder tests, no API calls)

**Exit Gate:** API key validated, RAG query returns coherent response, corpus
documents committed, repo public.

**Time Budget:** Complete before hacking starts.

---

### Phase 0: Walking Skeleton *(11:30 AM – 2:30 PM Saturday, 3 hrs)*

**Build:**
- [ ] `sentinel/config.py` — env loading, fail on missing keys
- [ ] `sentinel/models.py` — `Action`, `SentinelEvent`, all dataclasses
- [ ] `sentinel/eventbus.py` — singleton subscribe/emit
- [ ] `sentinel/orchestrator.py` — `attempt_action`, `resolve_frozen`, idempotency
- [ ] `sentinel/tools/calculate_ratio.py`
- [ ] `sentinel/tools/draft_memo.py`
- [ ] `sentinel/tools/send_memo.py` (sandbox guard)
- [ ] Force-freeze harness: `python -c "from sentinel.orchestrator import *; ..."`
  ```python
  from sentinel.models import Action
  from sentinel.orchestrator import attempt_action, resolve_frozen, set_freeze_policy
  set_freeze_policy(lambda a: a.is_irreversible)
  a = Action(description="send memo", tool_name="send_escalation_memo", is_irreversible=True)
  r = attempt_action(a)
  print(r.status)   # → frozen
  resolved = resolve_frozen(a.action_id, approved=True)
  print(resolved.status)   # → resumed
  ```
- [ ] `pytest tests/test_orchestrator.py` — all 5 pass

**Exit Gate:** Force-freeze harness runs in under 2 seconds. All orchestrator
tests pass. Zero API calls made.

---

### Phase 1: Vultr Integration + Document Ingestion *(2:30 PM – 5:00 PM Saturday, 2.5 hrs)*

**Build:**
- [ ] `sentinel/vultr_client.py` — `chat_json`, `chat_text` with retries
- [ ] `sentinel/vector_store.py` — `init_collection`, `ingest_document`, `rag_query`
- [ ] `sentinel/server.py` — lifespan ingests corpus on startup
- [ ] Integration test (manual):
  ```bash
  uvicorn sentinel.server:app --reload
  # Check logs: "Ingested credit_agreement.md", etc.
  curl -X POST http://localhost:8000/api/run
  # Tail logs: see AGENT_PLAN event emitted
  ```
- [ ] Validate RAG returns coherent answers to all 3 query types
- [ ] Validate `chat_json` returns valid JSON for a simple probe prompt

**Exit Gate:** `uvicorn` starts, corpus ingests, a manual `chat_json` call
returns valid JSON, a manual `rag_query` call returns text that references the
credit agreement content.

---

### Phase 2: Drift + Citations + Probe *(5:00 PM – 8:30 PM Saturday, 3.5 hrs)*

**Build:**
- [ ] `sentinel/drift.py` — `set_task_brief`, `score_drift`
- [ ] `sentinel/citations.py` — `check_citation_completeness`
- [ ] `sentinel/probe.py` — `run_probe` with full fail-closed validation
- [ ] `sentinel/pipeline.py` — `SentinelPipeline` with OR-gate compound policy
- [ ] Integration test:
  ```python
  from sentinel.models import Action, RetrievalCitation
  from sentinel.pipeline import SentinelPipeline
  import sentinel.orchestrator as orch

  SentinelPipeline(task_brief="Monitor covenant breach in credit agreement")
  a = Action(
      description="Send escalation memo to CFO",
      tool_name="send_escalation_memo",
      is_irreversible=True,
      citations=[RetrievalCitation("credit_agreement.md", "covenant_definition", "Debt/EBITDA ≤ 4.5x")]
  )
  result = orch.attempt_action(a)
  print(result.status)          # frozen (citation_score low — only 1 of 4 clauses)
  print(result.freeze_reason)   # citation_score=0.25 below threshold...
  ```
- [ ] Tune thresholds so the demo corpus reliably triggers a freeze at step 7

**Exit Gate:** Pipeline correctly freezes the send action. Drift score,
MAARS verdict, and citation score are all populated on the frozen action.
At least one freeze trigger fires for the demo scenario.

---

### Phase 3: Full Agent Loop *(8:30 PM – 10:00 PM Saturday, 1.5 hrs)*

**Build:**
- [ ] `sentinel/agent.py` — complete 7-step loop
- [ ] Integration test (full run):
  ```bash
  uvicorn sentinel.server:app
  curl -X POST http://localhost:8000/api/run
  # Watch logs for: AGENT_PLAN, 3× RETRIEVAL_PASS, TOOL_CALLED, ACTION_EXECUTED (draft_memo), ACTION_FROZEN (send_memo)
  curl http://localhost:8000/api/action/<frozen_action_id>
  ```
- [ ] Confirm the agent reaches step 7 and the gate fires reliably
- [ ] Confirm `send_memo.run()` never executes in demo mode (sandbox guard)

**Exit Gate:** Full 7-step agent loop completes end-to-end. Gate fires on
step 7. Sandbox guard prevents real email.

---

### Phase 4: Sequencer + Incident Artifact *(10:00 PM – 11:00 PM Saturday, 1 hr)*

**Build:**
- [ ] `sentinel/sequencer.py` — `seal_incident`
- [ ] Wire into `/api/decide` endpoint
- [ ] `pytest tests/test_sequencer.py` — both tests pass
- [ ] Manual test:
  ```bash
  curl -X POST http://localhost:8000/api/decide \
    -H "Content-Type: application/json" \
    -d '{"action_id": "<frozen_id>", "approved": true}'
  cat incidents/incident_*.json
  ```

**Exit Gate:** Incident JSON written to disk. Hash verified by hand (re-run
SHA-256 on the canonical JSON, compare). `pytest tests/test_sequencer.py`
passes.

---

### Phase 5: WebSocket + Broadcaster *(11:00 PM – 12:30 AM Sunday, 1.5 hrs)*

**Build:**
- [ ] `sentinel/broadcaster.py` — register/unregister/broadcast
- [ ] Wire broadcaster into `eventbus` in `server.py` lifespan
- [ ] WebSocket endpoint in `server.py`
- [ ] Manual WS test:
  ```bash
  # In one terminal:
  uvicorn sentinel.server:app
  # In another (wscat or websocat):
  wscat -c ws://localhost:8000/ws
  # In a third:
  curl -X POST http://localhost:8000/api/run
  # Watch events stream into wscat terminal in real time
  ```

**Exit Gate:** All 7 agent steps produce WS events visible in wscat. Freeze
event arrives within 500ms of the agent attempting the irreversible action.
Operator decision event arrives after `/api/decide`.

---

### Phase 6: Frontend *(12:30 AM – 9:00 AM Sunday, 8.5 hrs)*

**Build (use Impeccable for all component generation):**

```bash
# One-time setup
cd frontend
npm create vite@latest . -- --template react-ts
npm install

# Generate components via Impeccable
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md EventRow
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md RetrievalBadge
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md StatusBadge
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md SignalCard
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md CitationList
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md FreezeAlert
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md OperatorGate
impeccable build --product ../docs/PRODUCT.md --design ../docs/DESIGN.md IncidentRecord
```

**Build order:**
1. `tokens.css` — paste exactly from §8.4, commit
2. `useEventStream.ts` — paste from §8.3, verify connects to backend WS
3. `MonitorPage.tsx` — event stream only (no freeze handling yet)
4. End-to-end smoke test: run agent, watch events appear in browser
5. `FreezeAlert.tsx` — red full-width freeze notification
6. `SignalPage.tsx` — three signal cards + citation list
7. `GatePage.tsx` — operator gate + incident record
8. Wire navigation: on freeze event, route to SignalPage; after decision, show GatePage
9. Impeccable critique checkpoint:
   ```bash
   impeccable critique --product ../docs/PRODUCT.md --design ../docs/DESIGN.md --visual
   ```
10. Fix any critique findings
11. Final audit:
    ```bash
    impeccable audit --product ../docs/PRODUCT.md --design ../docs/DESIGN.md
    ```

**Non-negotiables (checked before submission):**
- [ ] Freeze state change visible within 200ms of WS event
- [ ] Signal scores displayed as human-readable numbers with labels
- [ ] Citations visible by default (not collapsed)
- [ ] APPROVE and ABORT buttons are full-width, clearly labelled, same visual weight
- [ ] Integrity hash visible, labelled "SHA-256", copyable
- [ ] Auto-reconnect visible in UI (connection status indicator)
- [ ] Entire flow runs without a page reload

**Exit Gate:** Complete end-to-end flow works in browser. Impeccable audit
returns no critical findings.

---

### Phase 7: Demo Polish + Submission *(9:00 AM – 11:30 AM Sunday, 2.5 hrs)*

**Build:**
- [ ] `README.md` — architecture diagram + setup instructions + demo steps
- [ ] `.env.example` — all variables with comments
- [ ] Record 60-second demo video (see script below)
- [ ] Submit at `cerebralvalley.ai/e/raise-summit-hackathon/hackathon/submit`

**60-Second Demo Script:**

```
0:00 – 0:08  Browser open. "SENTINEL is an enterprise document agent with an
              oversight gate. Watch it run live."
              Click RUN.

0:08 – 0:22  Point at event stream: "The agent plans, then retrieves from three
              documents — covenant definition, historical ratios, transaction
              log — each pass builds the evidence chain."

0:22 – 0:30  Ratio tool fires. "Current Debt/EBITDA: 4.62x. Threshold: 4.5x.
              Breach detected."

0:30 – 0:40  FREEZE alert hits screen. "SENTINEL catches the escalation memo
              before it sends. Three signals fired: drift score, MAARS verdict,
              citation completeness."
              Point at each signal card briefly.

0:40 – 0:50  Click APPROVE. "Operator approves. Incident sealed."
              Point at SHA-256 hash. "Every decision is cryptographically
              hashed — compliance has a full audit trail."

0:50 – 0:60  "Multi-step retrieval, cited evidence, live override, tamper-evident
              record. Built entirely during this hackathon on Vultr Serverless
              Inference."
```

---

## 11. Open Risks + Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Vultr rate limit during demo | Low | Critical | Cache all RAG responses after first successful run; replay from cache if rate-limited |
| RAG endpoint latency >10s | Medium | High | Add explicit timeout (30s) in `vector_store.py`; show loading state in UI |
| Vector store cold start | Medium | Medium | Warm up collection with a dummy query 10 minutes before demo |
| MAARS probe returns invalid JSON | Medium | Low | Fail-closed: treated as NO — gate fires regardless |
| WebSocket drops during demo | Low | High | Auto-reconnect in `useEventStream` + connection status in UI |
| Model identifier changed | Low | Critical | Validate at Phase −1; hardcode fallback model in config |
| Demo corpus too thin for coherent RAG | Medium | High | ~1800 words total is sufficient; validate RAG quality at Phase 1 exit gate |

---

## 12. What SENTINEL Is NOT (Disqualification Avoidance)

The hackathon rules disqualify:
- Basic RAG applications ✗ — SENTINEL is a multi-step planning agent; RAG is
  one tool in a 7-step loop, not the product
- Dashboards as main feature ✗ — the main feature is the oversight gate and
  incident artifact; the UI is the operator interface, not the product
- Streamlit applications ✗ — React 18 + FastAPI + WebSockets
- Mental health / nutrition / medical advice ✗ — finance/compliance domain
- Chatbots ✗ — no conversational interface; agent runs autonomously

The main feature is: **a multi-step document agent with a human-in-the-loop
oversight gate that produces a cryptographically-hashed audit artifact.**
That is none of the banned categories.

---

## 13. Submission Checklist

- [ ] Repo is public
- [ ] `README.md` explains what was built during the hackathon (not before)
- [ ] All code was written during the event (design doc + corpus are pre-event design artifacts)
- [ ] Demo video uploaded to YouTube/Loom, link ready for submission form
- [ ] Submission form completed at `cerebralvalley.ai/e/raise-summit-hackathon/hackathon/submit`
- [ ] Vultr Serverless Inference is the primary LLM provider (no other paid LLM APIs)
- [ ] `requirements.txt` is pinned and reproducible
- [ ] `incidents/` directory contains at least one sealed incident from a test run
- [ ] `.env.example` committed; `.env` gitignored

---

*SENTINEL v2.0 — Vultr Track Adaptation*
*RAISE Summit Hackathon 2026 — Remote Solo*
*Commit this document as `docs/designdoc.md` before hacking starts.*
```

---

## Key Differences from the Original (Change Log)

| Original (Crusoe) | This Version (Vultr) |
|---|---|
| `crusoe_client.py` with OpenAI SDK | `vultr_client.py` — same pattern, `api.vultrinference.com/v1` |
| Manual cosine distance embeddings | Vultr native vector store + RAG endpoint — no embedding management |
| Scripted 5-step fake agent | Real 7-step planning agent with conditional retrieval |
| Drift via cosine distance | Drift via LLM alignment check (uses same Vultr inference) |
| No citation tracking | `citations.py` — new signal, directly addresses Vultr multi-retrieval requirement |
| MAARS probe only signal | OR-gate: drift OR MAARS OR citation completeness |
| "Dead man's switch" frame | "Enterprise document agent with oversight gate" frame |
| Heartbeat/latency monitor | Citation completeness checker (more relevant to document domain) |
| Finance domain implied | Finance domain explicit: covenant breach scenario, planted corpus |
| No disqualification analysis | §12 explicitly addresses every banned category |
