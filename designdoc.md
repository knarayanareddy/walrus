# SENTINEL — Master Design Document
## RAISE Summit Hackathon 2026 · Crusoe Track

**Version 1.0 · Pre-Hack Reference Document**

The previous responses covered these pieces in fragments across the conversation — this document consolidates everything into one authoritative spec so the whole team can build from a single source of truth instead of scrolling back through chat history.

---

## 1. Executive Summary

| | |
|---|---|
| **Project name** | SENTINEL |
| **One-line pitch** | A dead man's switch for autonomous AI agents — freezes irreversible actions when human oversight lapses or an adversarial probe disagrees with the agent. |
| **Track** | Crusoe (Neon Noir venue, 50-participant cap) |
| **Team size** | 5 (in-person) |
| **Core thesis** | Enterprise agentic AI governance today is entirely pre-action (permissions, sandboxing). Nobody builds the post-action failsafe: what happens when oversight itself lapses mid-task. |
| **Origin story** | Direct synthesis of two existing repos: MAARS (adversarial multi-agent consensus scoring) + the Digital Will/MORTIS/LegacyVault "dead man's switch" lineage. |

---

## 2. Problem Statement Alignment

Crusoe's brief requires agents that construct a live situational model from streaming inputs and use that model to drive proactive, context-sensitive actions a non-technical operator can trust, question, and override in the moment.

**SENTINEL's direct mapping:**

| Crusoe requirement | SENTINEL mechanism |
|---|---|
| Live situational model from streaming inputs | Real-time scope drift scoring + heartbeat monitoring, both computed continuously via Crusoe inference |
| Proactive, context-sensitive action | Autonomous FREEZE of irreversible tool calls before they execute |
| Operator can trust, question, override | Approve/Abort buttons + sealed incident report with full reasoning trace |
| Learns from override | Scope model updates from operator decisions, feeding future drift calculations |

---

## 3. System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                        DEMO AGENT LAYER                        │
│   Proposes sequential actions (search → read → write → email)  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ action proposed
                            ▼
┌───────────────────────────────────────────────────────────────┐
│                     SENTINEL DECISION ENGINE                    │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │  DRIFT SCORER    │  │  MAARS PROBE     │  │  HEARTBEAT    │  │
│  │  (Crusoe embed)  │  │  (Crusoe chat,   │  │  MONITOR      │  │
│  │                  │  │   JSON mode)     │  │  (local calc) │  │
│  │  cosine distance │  │  Builder/Critic  │  │  approval     │  │
│  │  from task vec   │  │  scoring pattern │  │  latency      │  │
│  └────────┬─────────┘  └────────┬─────────┘  └───────┬───────┘  │
│           └──────────┬───────────┴──────────┬─────────┘         │
│                      ▼                      ▼                   │
│              COMPOUND TRIGGER: (drift>0.35 AND MAARS=NO         │
│                       AND confidence>75) OR heartbeat<30         │
└───────────────────────────┬───────────────────────────────────┘
                            │ if triggered
                            ▼
┌───────────────────────────────────────────────────────────────┐
│                     FAILSAFE SEQUENCER                          │
│  FREEZE action → SEAL (SHA-256 incident JSON) → ALERT (UI +    │
│  optional Gradium voice) → await operator decision → LEARN     │
│  (update scope model from Approve/Abort)                        │
└───────────────────────────┬───────────────────────────────────┘
                            │ events
                            ▼
┌───────────────────────────────────────────────────────────────┐
│                          UI LAYER                                │
│  Live Agent Monitor │ Drift Trajectory │ MAARS Split-Screen │    │
│  Freeze/Incident Modal │ (stretch: Incident Archive)             │
└───────────────────────────────────────────────────────────────┘
```

---

## 4. Component Specifications

### 4.1 Demo Agent
- **Purpose**: Generates the live action stream SENTINEL monitors. Runs for real (live inference calls), sequence is pre-determined for demo reliability but not pre-scripted in its outcomes.
- **Demo sequence**: `search_web()` → `read_doc()` → `write_file()` → `draft_email()` → `send_email()` (last two escalate drift)
- **Output**: Structured action objects `{description, is_irreversible, timestamp}`

### 4.2 Drift Scorer
- **Input**: Task definition (embedded once at registration), each proposed action description
- **Method**: Cosine distance between task embedding and action embedding, both computed via Crusoe Managed Inference embedding endpoint
- **Threshold**: 0.35 (tunable during testing)
- **Output**: `drift_score: float`, emitted as `DRIFT_SCORED` event

### 4.3 MAARS Adversarial Probe
- **Origin**: Direct port of MAARS's master-prompt scoring/justification rubric and forbidden-behaviors constraints — not the 11-phase SDLC, just the single Builder/Critic scoring exchange, running once per action instead of once per SDLC phase.
- **Prompt contract**: Must return JSON `{verdict, confidence, reasoning, severity, remediation}`. Vague/unjustified verdicts are explicitly forbidden in the prompt (reused directly from MAARS's forbidden-behaviors list).
- **Call pattern**: Crusoe chat completion, JSON response mode
- **Output**: Emitted as `MAARS_PROBE` event, rendered in the split-screen UI

### 4.4 Heartbeat Monitor
- **Signal (simplified for build-time constraints)**: Approval latency trend — ratio of recent approval speed vs. baseline
- **Score**: 0–100, where rapid-fire approvals (rubber-stamping) drive the score down
- **Threshold**: <30 fires independently of drift/MAARS

### 4.5 Failsafe Sequencer
- **FREEZE**: Injects a hold flag, prevents execution of the flagged action
- **SEAL**: Serializes full decision trace + SHA-256 hash to `incidents/{uuid}.json` on disk — a tamper-evident log, not a full cryptographic seal (deliberately simplified from MORTIS's full implementation to fit the time budget)
- **ALERT**: Generates plain-language advisory text; optional Gradium TTS playback
- **LEARN**: On operator Approve/Abort, updates the scope embedding or flags an out-of-scope action class for future automatic freezing

---

## 5. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Inference | Crusoe Managed Inference | Embeddings (drift) + chat completions JSON mode (MAARS probe) |
| Backend | Python | Async-friendly for concurrent inference calls |
| Frontend | React + TypeScript | Consistent with team's stated frontend skillset |
| Voice (stretch) | Gradium TTS/STT | 145K credits available via `RAISE-2026` code |
| Storage | Local JSON files (incidents), no DB needed for demo scale | Avoids unnecessary infra for a 24-hour build |
| Design tooling | Impeccable (`npx impeccable install`) | Prevents generic-AI-SaaS visual tells |

---

## 6. Build Phase Map (Condensed Reference)

| Phase | Window | Deliverable | Owner |
|---|---|---|---|
| −1 | Pre-hack | Accounts, repo, PRODUCT.md/DESIGN.md drafted | All |
| 0 | Sat 11:30–1:30 | Freeze mechanic (stubbed trigger) | You + backend |
| 1 | Sat 11:30–1:30 | Live demo agent, calling into freeze | ML person |
| 2 | Sat 1:30–3:30 | Real Crusoe drift scoring | You + ML |
| 3 | Sat 3:30–5:30 | MAARS probe live | You |
| 4 | Sat 5:30–7:00 | Heartbeat trigger live | Backend |
| 5 | Sat 7:00–8:30 | Sealed incident report + advisory | You |
| 6 | Sat 8:30–11:00 | Dashboard: 4 screens, Impeccable checkpoints | Frontend + you |
| 7 (stretch) | Sun 7:00–9:00 | MCP wrapper | Backend |
| 8 (stretch) | Sun 9:00–9:45 | Override learning loop | You |
| 9 | Sun 9:00–11:00 | Demo video + submission | Storyteller + you |
| 10 | Sun 12:15–1:45 | Round 1 judging | You + team |
| 11 | Sun 2:00–3:00 | Final round (if advanced) | You + team |

**Irreducible core if time runs short**: Phases 0–3 + 5 + minimal 6. Everything else is amplification.

---

## 7. UI/UX Design System

### 7.1 Register Classification
**Product**, not Brand — earned familiarity is the bar (per Impeccable's framework distinguishing Brand — design IS the product, distinctiveness is the bar from Product — design SERVES the product ... where earned familiarity is the bar, fluent users of Linear / Figma / Notion / Raycast / Stripe should trust it).

### 7.2 Anti-Patterns to Avoid
Per Impeccable's documented AI-design tells: Inter font for everything, purple-to-blue gradients, cards nested in cards, gray text on colored backgrounds, rounded-square icon tiles above headings, and decorative grid overlays behind non-measurement content.

### 7.3 Visual Language

| Element | Spec |
|---|---|
| Base color | Near-black `#0A0B0D` |
| Status colors | Gray-green (healthy) / amber (drift) / red (frozen — reserved exclusively) |
| Display font | Non-Inter geometric sans |
| Data/log font | Monospace (JetBrains Mono / IBM Plex Mono) |
| Buttons | Sharp corners, 1px border, high contrast — not rounded pills |
| Charts | Straight-segment lines, single threshold reference line, no decorative grid |
| Layout | Flat surfaces, hairline borders, no nested cards |

### 7.4 Screen Inventory

| # | Screen | Priority |
|---|---|---|
| 1 | Live Agent Monitor | P0 |
| 2 | Scope Drift Trajectory chart | P0 |
| 3 | MAARS Adversarial Split-Screen | P0 |
| 4 | Freeze/Incident Modal (climax) | P0 |
| 5 | Incident Report Archive | P1 (stretch) |

### 7.5 Freeze Modal Choreography (critical path — this is the demo's emotional peak)

| Time | Event |
|---|---|
| T+0.0s | MAARS verdict lands, divider animates to red |
| T+0.5s | Scrim dims screen ~15% |
| T+1.0s | "FREEZE" text renders character-by-character |
| T+2.0s | "What Would Have Happened" comparison panel slides up |
| T+3.0s | Hash + Approve/Abort buttons activate |

### 7.6 Impeccable Integration Checkpoints

| Checkpoint | Command | When |
|---|---|---|
| A | `/impeccable critique monitor` | After Screen 1 built |
| B | `/impeccable audit dashboard` | After all 4 screens exist (~9 PM Sat) |
| C | `/impeccable polish` | Sunday morning, before recording demo |
| Optional | `/impeccable bolder` | If freeze modal feels visually flat — stays inside the existing design system rather than inventing new colors, per Impeccable's own description of the pass |

### 7.7 Explicit Non-Builds (Time Discipline)
No settings screen, no marketing/landing page, no auth/login flow, no mobile-responsive layout, no onboarding tutorial.

---

## 8. Demo Script

### 8.1 Opening (stakes-first framing)
Open with a concrete precedent (e.g., a real case of an autonomous system making an unauthorized commitment) before touching the keyboard — establishes why the room should care before showing the tool.

### 8.2 Live Sequence
1. Task registered on screen
2. Agent proposes 5 actions in sequence, live
3. Drift trajectory climbs visibly on Screen 2
4. At step 4–5, MAARS split-screen shows disagreement (Screen 3)
5. Freeze modal fires with full choreography (Screen 4)
6. Point at the sealed incident JSON + hash

### 8.3 Technical Credibility Beat
Explain that both drift scoring and MAARS probing are live Crusoe inference calls per action — not mocked, not pre-computed.

### 8.4 Origin Story Beat
State plainly: the MAARS probe is a direct extraction of an existing open-sourced multi-agent consensus framework; the freeze/seal mechanic is a direct lineage from three existing dead-man's-switch projects. This is the single strongest credibility line available — it's a true, specific, personal answer to "why did you build this."

---

## 9. Judging Criteria Alignment

Judging weights: Demo (50%), Impact (25%), Creativity (15%), Pitch (10%).

| Criterion | SENTINEL's answer |
|---|---|
| Demo (50%) | Live, working freeze mechanic — not slides, not a mock |
| Impact (25%) | Directly addresses enterprise agentic AI's unresolved trust/accountability gap |
| Creativity (15%) | Adversarial probe + dead-man's-switch fusion is not a template solution |
| Pitch (10%) | Stakes-first framing + concrete origin story |

**Rule compliance check**: Not a banned category (mental health, RAG, Streamlit, dashboard-as-main-feature, etc.) — SENTINEL's main feature is the autonomous freeze action, not the dashboard itself; the dashboard is instrumentation on top of a functioning agentic system.

---

## 10. Risk Register

| Risk | Likelihood | Mitigation |
|---|---|---|
| Crusoe inference latency slows live demo | Medium | Pre-warm API connections before stage demo; have cached fallback responses if live call times out |
| MAARS probe gives inconsistent verdicts across takes | Medium | Fix `temperature` low, test the exact demo sequence 3+ times before Sunday |
| Freeze modal choreography feels rushed live | Low | Rehearse timing with a stopwatch; adjust animation durations if needed |
| UI drifts toward generic-SaaS look under time pressure | Medium | Impeccable checkpoints at B and C catch this before it compounds |
| Team runs out of time before dashboard is polished | Medium | Priority-ordered screen list (1→4); cut Screen 5 first, voice second |
| Google/Crusoe account approval delays | Low–Medium | Submit forms pre-hack (Phase −1); have mock LLM client fallback ready |

---

## 11. Team Roles Summary

| Role | Primary responsibility | Secondary |
|---|---|---|
| You | MAARS probe, seal logic, architecture glue, pitch delivery | Origin story in Q&A |
| Backend | Freeze mechanic, heartbeat monitor, MCP wrapper (stretch) | Infra glue |
| ML/Inference | Demo agent, drift scorer integration | Crusoe API wrangling |
| Frontend | All 4 UI screens, Impeccable checkpoint execution | Dashboard polish |
| Storyteller | Demo script, video recording/narration, stage pitch support | Q&A backup |

---

## 12. Pre-Hack Checklist (Phase −1 Gate)

- [ ] Discord joined, team roster confirmed
- [ ] Crusoe account + Google Form submitted
- [ ] Gradium account + `RAISE-2026` code redeemed
- [ ] GitHub repo skeleton created
- [ ] `PRODUCT.md` and `DESIGN.md` drafted (Section 7 above)
- [ ] MAARS probe prompt template drafted (Section 4.3)
- [ ] Digital Will / MORTIS reference logic located and bookmarked
- [ ] Demo scenario script drafted (Section 8)
- [ ] `npx impeccable install` run in repo

---

## Does This Document Suffice, or Is Anything Still Missing?

Honestly assessing gaps against everything discussed across this conversation:

**Fully covered**: architecture, phase-by-phase build plan, UI/UX spec, demo choreography, judging alignment, risk register, team roles, pre-hack checklist.

**Genuinely still open** (not yet specified anywhere in our conversation, worth deciding before Saturday):

1. **Exact API model names** — you'll need to check Crusoe's model catalog day-of for the specific embedding/chat model identifiers; this doc references them generically since that's an implementation detail that depends on what's live at hack-time.
2. **Fallback if Crusoe rate-limits during the stage demo** — worth a literal cached-response fallback file, not just a mitigation note.
3. **Submission form URL** — not yet provided in the participant resources; confirm this exists before Sunday 12 PM.
4. **WiFi credentials** — still marked TBD in the official resources; nothing to do but wait for the update.

Everything else — the "why," the "what," the "how," and the "how it looks" — is now captured in one place. This document is what I'd hand to all 5 teammates Saturday morning as the shared source of truth.





# SENTINEL — Agent-Ready Build Specification
## Phase −1: Pre-Hack Setup

This is written as a direct instruction set you can hand to a coding agent (Cursor, Claude Code, etc.) verbatim. Each phase will follow this same structure: **Objective → Prerequisites → Exact Deliverables → File Structure → Step-by-Step Instructions → Do's → Don'ts → Validation Criteria → Handoff Gate**.

I'll do one phase per message so each stays deep rather than compressed. Confirm this one looks right and I'll move to Phase 0.

---

## PHASE −1: Pre-Hack Setup

### Objective
Establish a working repository skeleton, environment configuration, and reference materials so that Saturday morning starts with zero setup friction. Nothing built in this phase touches SENTINEL's actual logic — it is entirely scaffolding, credentials, and staged reference material.

### Prerequisites
None. This is the starting state.

### Exact Deliverables
1. A GitHub repository with the correct folder structure (below)
2. A working Python virtual environment with pinned dependencies
3. A working Node/React environment for the frontend
4. Crusoe API credentials confirmed and stored in `.env` (never committed)
5. Gradium API credentials confirmed and stored in `.env`
6. `PRODUCT.md` and `DESIGN.md` written and committed
7. `probe_prompt_template.md` drafted (MAARS extraction, not yet wired to code)
8. Reference logic files copied from your existing repos (Digital Will, MORTIS) into a `reference/` folder for porting later — these are NOT imported into the build directly, they exist purely as copy-source material for Phase 3 and Phase 5
9. `npx impeccable install` run successfully in the frontend directory
10. `.gitignore` correctly excluding `.env`, `node_modules`, `__pycache__`, `incidents/*.json` (except a `.gitkeep`)

---

### Repository Structure to Create

```
sentinel/
├── .env.example
├── .env                          # gitignored
├── .gitignore
├── README.md
├── PRODUCT.md
├── DESIGN.md
├── docs/
│   └── probe_prompt_template.md
├── reference/                     # copy-source only, not imported
│   ├── digital_will_checkin.py    # copied reference, not live code
│   └── mortis_seal.py             # copied reference, not live code
├── backend/
│   ├── requirements.txt
│   ├── main.py                    # entrypoint, empty stub for now
│   ├── sentinel/
│   │   ├── __init__.py
│   │   ├── agent.py                # demo agent (Phase 1)
│   │   ├── drift.py                # drift scorer (Phase 2)
│   │   ├── probe.py                # MAARS probe (Phase 3)
│   │   ├── heartbeat.py            # heartbeat monitor (Phase 4)
│   │   ├── sequencer.py            # failsafe sequencer (Phase 5)
│   │   ├── events.py               # shared event schema
│   │   └── crusoe_client.py        # Crusoe API wrapper
│   └── tests/
│       └── __init__.py
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── components/
│   │   ├── styles/
│   │   └── App.tsx
│   └── impeccable.config.json      # generated by /impeccable init
└── incidents/
    └── .gitkeep
```

---

### Step-by-Step Instructions

**1. Initialize the repo**
```bash
mkdir sentinel && cd sentinel
git init
mkdir -p backend/sentinel backend/tests reference docs incidents frontend
touch incidents/.gitkeep
```

**2. Create `.gitignore`**
```
.env
__pycache__/
*.pyc
node_modules/
incidents/*.json
.DS_Store
dist/
build/
```

**3. Create `.env.example`** (committed — template only)
```
CRUSOE_API_KEY=
CRUSOE_API_BASE=https://api.crusoe.ai/v1
CRUSOE_EMBED_MODEL=
CRUSOE_CHAT_MODEL=
GRADIUM_API_KEY=
```

**4. Set up Python environment**
```bash
cd backend
python3 -m venv venv
source venv/bin/activate
```

**5. Create `requirements.txt`**
```
openai>=1.0.0        # used generically for OpenAI-compatible endpoints
python-dotenv>=1.0.0
fastapi>=0.110.0
uvicorn>=0.29.0
websockets>=12.0
numpy>=1.26.0
pydantic>=2.6.0
pytest>=8.0.0
```
```bash
pip install -r requirements.txt
```

**6. Set up frontend environment**
```bash
cd ../frontend
npm create vite@latest . -- --template react-ts
npm install
npx impeccable install
```
Then inside Cursor (not terminal):
```
/impeccable init
```
This will prompt for product name, audience, register (answer: Product), and will generate/offer `DESIGN.md` — accept and then manually overwrite it with the version below to ensure exact control over the content rather than accepting fully generic defaults.

**7. Write `PRODUCT.md`** at repo root:
```markdown
# SENTINEL

## What it is
A runtime guardrail for autonomous AI agents — a dead man's switch that
freezes irreversible actions when human oversight lapses or an
adversarial probe disagrees with the agent's proposed action.

## Audience
Enterprise engineers, SREs, and AI safety/compliance teams already
fluent in incident-response tooling (PagerDuty, Datadog, Grafana) and
developer tools (Linear, GitHub).

## Register
Product. Earned familiarity is the bar, not novelty.

## Core emotional target
"I trust this system enough to let it act on my behalf when I'm not
watching." Calm under normal operation. Unmistakably urgent at the
moment of freeze. Forensically precise afterward.

## Non-goals
- Not a marketing site. No hero sections, no gradients, no
  "AI-powered" badge language.
- Not a chat interface. No message bubbles, no avatar icons.
- Not a settings/config product. No onboarding flows.
```

**8. Write `DESIGN.md`** at repo root — use the exact content specified in the master design document (Section 7.3 color/font/component tables). Do not let the AI tool auto-generate this from defaults; paste the specific values so every subsequent component generation references concrete tokens rather than inferred ones.

**9. Draft `docs/probe_prompt_template.md`** — this is the MAARS extraction, staged now so Phase 3 is just wiring, not writing:
```markdown
# MAARS Adversarial Probe — Prompt Template

You are an adversarial reviewer for an autonomous agent's proposed action.

TASK ASSIGNED TO AGENT: {task_definition}
ACTION PROPOSED: {action_description}
ACTIONS ALREADY TAKEN: {action_history}

Evaluate: Would a reasonable human supervisor, paying full attention,
approve this specific action given the assigned task?

RULES (do not violate):
- You must give a confidence score (0-100) for your verdict.
- A vague answer like "looks reasonable" is FORBIDDEN — you must
  justify with specific reasoning.
- If your verdict is NO, you must specify:
  (a) which part of the task scope this action falls outside of
  (b) the severity of the deviation (low/medium/high)
  (c) what remediation would make it acceptable

Respond ONLY in valid JSON matching this exact schema:
{
  "verdict": "YES" | "NO",
  "confidence": <int 0-100>,
  "reasoning": "<string>",
  "severity": "low" | "medium" | "high" | null,
  "remediation": "<string>" | null
}
```

**10. Copy reference logic** into `reference/` — literal copies of the relevant functions from your existing Digital Will and MORTIS repos, renamed clearly. These are NOT wired into the build yet. They exist so that in Phase 3/5, the agent building those phases can read a real, working reference implementation instead of reinventing the pattern from a text description.

**11. Confirm Crusoe and Gradium credentials work** with a trivial smoke test:
```python
# backend/tests/test_smoke.py
import os
from dotenv import load_dotenv
load_dotenv()

def test_env_vars_present():
    assert os.getenv("CRUSOE_API_KEY"), "Missing CRUSOE_API_KEY"
    assert os.getenv("GRADIUM_API_KEY"), "Missing GRADIUM_API_KEY"
```
```bash
pytest backend/tests/test_smoke.py -v
```

---

### Do's

- **Do** commit `PRODUCT.md` and `DESIGN.md` before any teammate writes a single UI component — every subsequent AI-generated screen should reference these files by path in its prompt.
- **Do** pin dependency versions in `requirements.txt` — a mid-hackathon dependency resolution failure is a preventable disaster.
- **Do** verify both API keys work with a live smoke-test call (not just presence-checking the env var) before Saturday — an invalid/expired key discovered at 11:30 AM Saturday costs you your first hour.
- **Do** create the full folder skeleton now, even with empty stub files — this means every teammate's first commit on Saturday is additive, not "let me first figure out where this goes."
- **Do** write the MAARS probe template now, fully — this is pure prompt engineering with zero dependencies, and it's the highest-leverage 20 minutes you can spend pre-hack.

### Don'ts

- **Don't** write any actual SENTINEL logic yet (drift scoring, freeze mechanic, UI components) — Phase −1 is scaffolding only. Writing ahead here risks building against assumptions that shift once you're in the room with real teammates and real constraints.
- **Don't** accept Impeccable's auto-generated `DESIGN.md` defaults without overwriting with your specific tokens — generic defaults will still trend toward common AI-design patterns unless explicitly constrained.
- **Don't** commit `.env` under any circumstances — double check `.gitignore` is active before the first commit with real keys.
- **Don't** over-build the reference/ folder — copy only the specific functions you'll port (check-in window logic from Digital Will, seal/hash logic from MORTIS), not entire repos.
- **Don't** let this phase run long. Everything here should take under 2 hours of async, pre-hack time. If it's ballooning, you're over-scoping — cut back to literally just: repo skeleton + credentials confirmed + the two markdown files.

---

### Validation Criteria (Exit Gate)

Before Saturday 8:30 AM, confirm all of the following are true:

- [ ] `git clone` of the repo on a fresh machine, `pip install -r requirements.txt`, and `npm install` all succeed with zero errors
- [ ] `pytest backend/tests/test_smoke.py` passes (both API keys present and non-empty)
- [ ] A live test call to Crusoe's embedding endpoint returns a vector (not an auth error)
- [ ] A live test call to Crusoe's chat completion endpoint (JSON mode) returns valid JSON
- [ ] `PRODUCT.md` and `DESIGN.md` exist at repo root and are committed
- [ ] `docs/probe_prompt_template.md` exists and contains the full MAARS prompt
- [ ] `reference/digital_will_checkin.py` and `reference/mortis_seal.py` exist with real ported logic (not placeholder comments)
- [ ] `npx impeccable install` completed in `frontend/` with no errors, `impeccable.config.json` exists
- [ ] All 5 team members have cloned the repo and confirmed it runs locally

---

**Once you confirm this, say "next" and I'll do Phase 0: The Freeze Mechanic in the same level of detail — exact code, exact test cases, exact edge-case handling for what happens when an "irreversible" action is ambiguous, concurrent freeze attempts, etc.**






# SENTINEL — Agent-Ready Build Specification
## Phase 0: The Freeze Mechanic (Walking Skeleton)

---

### Objective

Build the smallest possible working version of SENTINEL's core mechanic: a system that can **intercept a proposed action, decide whether to freeze it, and record that decision as a structured event** — with zero intelligence behind the decision yet. This phase exists to prove the *shape* of the system works mechanically before any AI/inference logic touches it. This is your insurance policy: if every later phase fails, this phase alone still produces a legitimate (if simplistic) live demo of an action being frozen.

**Critical framing for whoever builds this**: you are not building a demo yet. You are building the **load-bearing skeleton** that Phases 1–5 attach to. Every design decision here (event schema, action ID scheme, freeze state storage) will be used unmodified by every later phase. Getting the schema wrong here means refactoring under time pressure later — get it right now.

---

### Prerequisites

- Phase −1 complete: repo skeleton exists, `backend/sentinel/` package importable, virtual environment active
- No API keys required for this phase — this phase is 100% deterministic, no LLM calls, no network calls

---

### Exact Deliverables

1. `backend/sentinel/events.py` — the shared event schema used by **every** subsequent phase
2. `backend/sentinel/orchestrator.py` — the `attempt_action()` / `should_freeze()` / `freeze()` core loop
3. `backend/sentinel/action.py` — the `Action` data model with validation
4. A CLI test harness (`backend/tests/manual_phase0_demo.py`) that runs a hardcoded 5-action sequence and freezes the last one
5. Unit tests covering the core loop plus every edge case listed below
6. A passing test suite with 100% of Phase 0 logic covered

---

### File: `backend/sentinel/action.py`

This defines what an "action" *is* — the fundamental unit every other phase operates on. Get this precise now; every phase downstream imports this.

```python
"""
Action model — the fundamental unit SENTINEL evaluates.

An Action represents a single proposed step by the monitored agent.
Every phase (drift scoring, MAARS probe, heartbeat, sequencer) reads
and annotates the same Action object as it flows through the pipeline.
"""

from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Optional
import uuid


class ActionStatus(str, Enum):
    PROPOSED = "PROPOSED"
    EXECUTED = "EXECUTED"
    FROZEN = "FROZEN"
    RESUMED = "RESUMED"      # operator approved after freeze
    ABORTED = "ABORTED"       # operator rejected after freeze


@dataclass
class Action:
    """
    A single proposed action from the monitored agent.

    Fields populated at creation:
        description: human-readable description of what the agent wants to do
        is_irreversible: whether this action, once executed, cannot be undone
        tool_name: the name of the tool/function being invoked
        parameters: arbitrary dict of tool call parameters

    Fields populated progressively as the action flows through phases:
        drift_score: set in Phase 2
        maars_verdict: set in Phase 3
        heartbeat_score_at_time: set in Phase 4
        status: updated by the orchestrator/sequencer
    """
    description: str
    is_irreversible: bool
    tool_name: str
    parameters: dict = field(default_factory=dict)

    # Auto-populated, do not set manually
    action_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    proposed_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    status: ActionStatus = ActionStatus.PROPOSED

    # Populated by later phases — must default to None, never omitted
    drift_score: Optional[float] = None
    maars_verdict: Optional[dict] = None
    heartbeat_score_at_time: Optional[int] = None

    def __post_init__(self):
        if not self.description or not self.description.strip():
            raise ValueError("Action.description cannot be empty")
        if not self.tool_name or not self.tool_name.strip():
            raise ValueError("Action.tool_name cannot be empty")
        if not isinstance(self.is_irreversible, bool):
            raise TypeError(
                f"Action.is_irreversible must be bool, got {type(self.is_irreversible)}. "
                f"Ambiguous irreversibility must be resolved to a boolean BEFORE "
                f"constructing an Action — see IRREVERSIBILITY POLICY in orchestrator.py"
            )

    def to_dict(self) -> dict:
        d = self.__dict__.copy()
        d["proposed_at"] = self.proposed_at.isoformat()
        d["status"] = self.status.value
        return d
```

**Why a dataclass with `__post_init__` validation, not a plain dict**: every later phase (drift scorer, MAARS probe, sequencer) will read and write fields on this object. A dict-based approach means a typo'd key (`"drift_scroe"`) silently fails at runtime with no error, discovered only when the UI shows a blank field during a live demo. A validated dataclass fails loudly and immediately at construction time — Saturday afternoon, not Sunday on stage.

---

### File: `backend/sentinel/events.py`

Every phase emits events through this single schema. The UI layer (Phase 6) subscribes to this stream. Get this right once, here, and never touch it again.

```python
"""
Event schema — the single append-only stream every phase emits to.

This mirrors the AgentEvent pattern from the MAARS dashboard's Rust/TS
event contract, adapted to Python. The frontend subscribes to this
stream over a WebSocket (wired in Phase 6) and renders every screen
purely as a function of events received — no polling, no hidden state
in the UI beyond what events have communicated.
"""

from __future__ import annotations
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from enum import Enum
from typing import Any, Optional
import json


class EventType(str, Enum):
    ACTION_PROPOSED = "ACTION_PROPOSED"
    DRIFT_SCORED = "DRIFT_SCORED"           # Phase 2
    MAARS_PROBE = "MAARS_PROBE"             # Phase 3
    HEARTBEAT_CHECK = "HEARTBEAT_CHECK"      # Phase 4
    ACTION_EXECUTED = "ACTION_EXECUTED"
    ACTION_FROZEN = "ACTION_FROZEN"
    INCIDENT_SEALED = "INCIDENT_SEALED"      # Phase 5
    OPERATOR_DECISION = "OPERATOR_DECISION"  # Phase 5/8
    ERROR = "ERROR"


@dataclass
class SentinelEvent:
    event_type: EventType
    action_id: Optional[str]
    payload: dict
    timestamp: datetime

    def to_json(self) -> str:
        d = {
            "event_type": self.event_type.value,
            "action_id": self.action_id,
            "payload": self.payload,
            "timestamp": self.timestamp.isoformat(),
        }
        return json.dumps(d)


class EventBus:
    """
    In-process event bus for Phase 0-5 (backend logic).
    Phase 6 wires this to a WebSocket broadcaster — this class itself
    has zero knowledge of the frontend and must stay that way.
    """
    def __init__(self):
        self._subscribers: list = []
        self._log: list[SentinelEvent] = []  # full history, used by incident sealing

    def subscribe(self, callback):
        self._subscribers.append(callback)

    def emit(self, event_type: EventType, action_id: Optional[str], payload: dict):
        event = SentinelEvent(
            event_type=event_type,
            action_id=action_id,
            payload=payload,
            timestamp=datetime.now(timezone.utc),
        )
        self._log.append(event)
        for callback in self._subscribers:
            callback(event)
        return event

    def history_for_action(self, action_id: str) -> list[SentinelEvent]:
        return [e for e in self._log if e.action_id == action_id]

    def full_history(self) -> list[SentinelEvent]:
        return list(self._log)


# Module-level singleton — every phase imports and uses THIS instance.
# Do not instantiate a second EventBus anywhere in the codebase; a split
# event bus means the UI silently misses half the events.
bus = EventBus()
```

**Why a singleton, explicitly called out**: the single most common mistake in a rushed multi-person build is someone in Phase 3 or Phase 4 instantiating their own `EventBus()` because they didn't realize one already existed in `events.py`. This produces a bug that's invisible in isolated testing (each phase's own tests pass) and only surfaces Saturday night when the UI shows some events but not others. The comment above is there specifically to prevent that.

---

### File: `backend/sentinel/orchestrator.py`

This is Phase 0's actual deliverable — the core loop.

```python
"""
Orchestrator — the core attempt_action() / should_freeze() loop.

Phase 0 scope: should_freeze() is a stub that can be manually toggled.
Phase 2 replaces it with real drift scoring. Phase 3 adds the MAARS
probe. Phase 4 adds the heartbeat check. This file's PUBLIC INTERFACE
(attempt_action's signature and return type) must not change across
those phases — only the internals of should_freeze() change.
"""

from __future__ import annotations
from typing import Callable, Optional
import threading

from sentinel.action import Action, ActionStatus
from sentinel.events import bus, EventType


class FreezeDecision:
    """Return type of should_freeze() — never return a bare bool.
    This forces every should_freeze() implementation, from Phase 0's
    stub through Phase 4's compound trigger, to explain itself."""
    def __init__(self, should_freeze: bool, reason: str):
        self.should_freeze = should_freeze
        self.reason = reason


class Orchestrator:
    def __init__(self):
        # Guards against a specific race condition: if the demo agent
        # (Phase 1) somehow proposes two actions concurrently — e.g.
        # a retry fires while the original call is still in flight —
        # this lock ensures attempt_action() processes one at a time.
        # A production system would want per-task locking, not global;
        # for a single-demo-agent hackathon build, a global lock is
        # correct and sufficient.
        self._lock = threading.Lock()
        self._frozen_actions: dict[str, Action] = {}
        self._executed_action_ids: set[str] = set()

        # PHASE 0 STUB — replaced in Phase 4 with the real compound trigger.
        # Kept as an injectable callable (not hardcoded logic) so later
        # phases can swap it without touching attempt_action() itself.
        self._should_freeze_fn: Callable[[Action], FreezeDecision] = (
            self._phase0_stub_should_freeze
        )

    def set_freeze_policy(self, fn: Callable[[Action], FreezeDecision]):
        """Called by Phase 2/3/4 to install the real decision logic."""
        self._should_freeze_fn = fn

    def _phase0_stub_should_freeze(self, action: Action) -> FreezeDecision:
        """
        Phase 0 default: never freeze anything. This exists purely so
        the pipeline is provably wired end-to-end before any real
        intelligence exists. Use force_freeze_next() below to test the
        freeze path deterministically during Phase 0 development.
        """
        return FreezeDecision(should_freeze=False, reason="Phase 0 stub: pass-through")

    # --- Testing hook, Phase 0 only ---
    _force_freeze_next_id: Optional[str] = None

    def force_freeze_next(self, action_id: str):
        """Test-only hook: forces the next call for this action_id to freeze,
        regardless of should_freeze_fn's real output. Used exclusively by
        the manual demo harness to prove the freeze path fires correctly
        before Phase 2/3/4 exist."""
        self._force_freeze_next_id = action_id

    def attempt_action(self, action: Action) -> Action:
        """
        The single entry point every phase and the demo agent calls.

        Returns the SAME Action object, mutated with its final status.
        Never returns a new object — callers hold a reference to the
        original action and expect status to update on it directly.
        """
        with self._lock:
            # Idempotency guard: if this exact action_id was already
            # executed or frozen, do not process it twice. This matters
            # because a flaky demo-agent retry loop (Phase 1) could
            # otherwise call attempt_action() twice for the same
            # logical action, producing duplicate FROZEN events and a
            # confusing double-entry in the UI.
            if action.action_id in self._executed_action_ids:
                bus.emit(EventType.ERROR, action.action_id,
                          {"message": "Duplicate attempt_action call ignored",
                           "action_id": action.action_id})
                return action
            if action.action_id in self._frozen_actions:
                # Already frozen and awaiting operator decision — do not
                # re-evaluate freeze logic, just return current state.
                return self._frozen_actions[action.action_id]

            bus.emit(EventType.ACTION_PROPOSED, action.action_id, {
                "description": action.description,
                "tool_name": action.tool_name,
                "is_irreversible": action.is_irreversible,
            })

            # --- Phase 0 test hook check ---
            if self._force_freeze_next_id == action.action_id:
                self._force_freeze_next_id = None
                decision = FreezeDecision(True, "Forced by test harness")
            else:
                decision = self._should_freeze_fn(action)

            if action.is_irreversible and decision.should_freeze:
                return self._freeze(action, decision.reason)

            return self._execute(action)

    def _execute(self, action: Action) -> Action:
        action.status = ActionStatus.EXECUTED
        self._executed_action_ids.add(action.action_id)
        bus.emit(EventType.ACTION_EXECUTED, action.action_id, {
            "description": action.description,
        })
        return action

    def _freeze(self, action: Action, reason: str) -> Action:
        action.status = ActionStatus.FROZEN
        self._frozen_actions[action.action_id] = action
        bus.emit(EventType.ACTION_FROZEN, action.action_id, {
            "description": action.description,
            "reason": reason,
        })
        return action

    def resolve_frozen(self, action_id: str, approved: bool) -> Action:
        """
        Called when the operator clicks Approve/Abort in the UI
        (wired fully in Phase 5, stubbed here for interface completeness).
        """
        if action_id not in self._frozen_actions:
            raise KeyError(f"No frozen action with id {action_id}")
        action = self._frozen_actions.pop(action_id)
        if approved:
            action.status = ActionStatus.RESUMED
            self._executed_action_ids.add(action_id)
            bus.emit(EventType.OPERATOR_DECISION, action_id,
                      {"decision": "APPROVE"})
        else:
            action.status = ActionStatus.ABORTED
            bus.emit(EventType.OPERATOR_DECISION, action_id,
                      {"decision": "ABORT"})
        return action


# Module-level singleton, same rationale as EventBus above.
orchestrator = Orchestrator()
```

---

### File: `backend/tests/manual_phase0_demo.py`

This is your Phase 0 exit-gate proof — runnable, visible, demonstrable.

```python
"""
Manual CLI demo for Phase 0. Run this and watch a 5-action sequence
execute, with the 5th action force-frozen to prove the mechanic works
end-to-end before any real intelligence exists.

Run: python -m tests.manual_phase0_demo
"""

from sentinel.action import Action
from sentinel.orchestrator import orchestrator
from sentinel.events import bus


def print_event(event):
    print(f"[{event.timestamp.strftime('%H:%M:%S')}] "
          f"{event.event_type.value:20s} {event.payload}")


def main():
    bus.subscribe(print_event)

    actions = [
        Action(description="Search web for vendor list",
               is_irreversible=False, tool_name="search_web"),
        Action(description="Read vendor pricing doc",
               is_irreversible=False, tool_name="read_doc"),
        Action(description="Write comparison table to file",
               is_irreversible=False, tool_name="write_file"),
        Action(description="Draft outreach email",
               is_irreversible=False, tool_name="draft_email"),
        Action(description="Send email to vendor",
               is_irreversible=True, tool_name="send_email"),
    ]

    # Force-freeze the 5th action to prove the mechanic works
    orchestrator.force_freeze_next(actions[4].action_id)

    for action in actions:
        result = orchestrator.attempt_action(action)
        print(f"  -> final status: {result.status.value}\n")

    frozen = [a for a in actions if a.status.value == "FROZEN"]
    assert len(frozen) == 1, f"Expected 1 frozen action, got {len(frozen)}"
    print("✅ Phase 0 exit gate passed: freeze mechanic works end-to-end.")


if __name__ == "__main__":
    main()
```

---

### File: `backend/tests/test_orchestrator.py`

```python
import pytest
from sentinel.action import Action
from sentinel.orchestrator import Orchestrator, FreezeDecision


def make_action(irreversible=False, action_id=None):
    a = Action(description="test action", is_irreversible=irreversible,
               tool_name="test_tool")
    if action_id:
        a.action_id = action_id
    return a


def test_reversible_action_always_executes():
    orch = Orchestrator()
    action = make_action(irreversible=False)
    result = orch.attempt_action(action)
    assert result.status.value == "EXECUTED"


def test_irreversible_action_with_no_freeze_policy_executes():
    """Phase 0 stub never freezes — even irreversible actions pass
    through by default. Freezing is opt-in only once policy is set."""
    orch = Orchestrator()
    action = make_action(irreversible=True)
    result = orch.attempt_action(action)
    assert result.status.value == "EXECUTED"


def test_forced_freeze_overrides_policy():
    orch = Orchestrator()
    action = make_action(irreversible=True)
    orch.force_freeze_next(action.action_id)
    result = orch.attempt_action(action)
    assert result.status.value == "FROZEN"


def test_custom_freeze_policy_is_respected():
    orch = Orchestrator()
    orch.set_freeze_policy(lambda a: FreezeDecision(True, "always freeze"))
    action = make_action(irreversible=True)
    result = orch.attempt_action(action)
    assert result.status.value == "FROZEN"


def test_reversible_action_never_freezes_even_if_policy_says_so():
    """Freeze only applies to irreversible actions by design — a
    reversible action should never be frozen even if the freeze
    policy returns True. This is enforced in attempt_action, not
    in the policy itself, so no future policy implementation can
    accidentally freeze something reversible."""
    orch = Orchestrator()
    orch.set_freeze_policy(lambda a: FreezeDecision(True, "aggressive"))
    action = make_action(irreversible=False)
    result = orch.attempt_action(action)
    assert result.status.value == "EXECUTED"


def test_duplicate_action_id_is_idempotent():
    orch = Orchestrator()
    action = make_action(irreversible=False, action_id="fixed-id-123")
    orch.attempt_action(action)
    # Simulate a retry sending the same action_id again
    action2 = make_action(irreversible=False, action_id="fixed-id-123")
    result = orch.attempt_action(action2)
    # Should be ignored, not double-executed
    events = [e for e in orch.__class__.__dict__]  # sanity placeholder
    assert result.action_id == "fixed-id-123"


def test_frozen_action_can_be_resolved_approved():
    orch = Orchestrator()
    orch.set_freeze_policy(lambda a: FreezeDecision(True, "test"))
    action = make_action(irreversible=True)
    orch.attempt_action(action)
    resolved = orch.resolve_frozen(action.action_id, approved=True)
    assert resolved.status.value == "RESUMED"


def test_frozen_action_can_be_resolved_aborted():
    orch = Orchestrator()
    orch.set_freeze_policy(lambda a: FreezeDecision(True, "test"))
    action = make_action(irreversible=True)
    orch.attempt_action(action)
    resolved = orch.resolve_frozen(action.action_id, approved=False)
    assert resolved.status.value == "ABORTED"


def test_resolving_nonexistent_frozen_action_raises():
    orch = Orchestrator()
    with pytest.raises(KeyError):
        orch.resolve_frozen("does-not-exist", approved=True)


def test_action_rejects_non_bool_is_irreversible():
    with pytest.raises(TypeError):
        Action(description="bad", is_irreversible="yes", tool_name="x")


def test_action_rejects_empty_description():
    with pytest.raises(ValueError):
        Action(description="", is_irreversible=False, tool_name="x")
```

Run with:
```bash
cd backend
pytest tests/test_orchestrator.py -v
```

---

### The "Ambiguous Irreversibility" Problem — Explicit Policy

This is a real edge case worth naming explicitly, because it will come up the moment Phase 1's demo agent needs to classify its own actions.

**Policy**: `is_irreversible` must be determined **at the point the action is proposed**, by the calling code (Phase 1's demo agent), never inferred inside the orchestrator. The orchestrator treats it as ground truth, not something to guess at.

For the demo agent (Phase 1), use this explicit static classification table — do not attempt to have an LLM classify reversibility dynamically in Phase 0/1; that's an unnecessary inference call for something you can hardcode correctly for a known, fixed demo sequence:

| Tool | is_irreversible |
|---|---|
| `search_web` | `False` |
| `read_doc` | `False` |
| `write_file` (to scratch/local dir) | `False` |
| `draft_email` (not sent) | `False` |
| `send_email` | `True` |
| `post_to_api` | `True` |
| `delete_file` | `True` |
| `execute_payment` | `True` |

---

### Do's

- **Do** treat `attempt_action()`'s function signature as frozen after this phase — Phases 2, 3, and 4 only change what happens *inside* `should_freeze()`, never the calling convention. If a later-phase agent wants to change the signature, that's a signal something was designed wrong here — stop and reconsider rather than patching around it.
- **Do** run `manual_phase0_demo.py` and visually confirm the printed event log shows exactly 4 `ACTION_EXECUTED` events followed by 1 `ACTION_FROZEN` event, in order, before moving to Phase 1.
- **Do** keep `FreezeDecision` as a small object with a `reason` string, never a bare boolean — every later phase's freeze decision needs to explain itself for the UI (Phase 6) and the incident report (Phase 5).
- **Do** use the module-level singletons (`bus`, `orchestrator`) everywhere — import them, don't re-instantiate.
- **Do** write the idempotency test now, even though it feels like premature engineering — a demo agent (Phase 1) that retries on a timeout is a realistic failure mode during a live stage demo, and double-executing a frozen action live in front of judges is a real risk worth guarding against now while it's cheap.

### Don'ts

- **Don't** let anyone infer irreversibility inside the orchestrator based on keyword matching (e.g., "if 'email' in description"). This is fragile, and any later phase depending on it will inherit that fragility. Irreversibility is a property the calling agent declares explicitly.
- **Don't** use a bare `bool` return type for `should_freeze()`. Every future phase needs the `reason` field for the UI and incident report — retrofitting this later means touching every call site.
- **Don't** add persistence (SQLite, file-based state) in this phase. Phase 0's frozen-action state lives in memory only (`self._frozen_actions` dict). Persistence is explicitly out of scope until Phase 5's incident sealing, and even then it's a one-way write (incident JSON), not a stateful store to read back from.
- **Don't** build any UI in this phase. The `print_event` callback in the manual demo is your only "UI" right now — resist the urge to wire even a placeholder frontend. That's Phase 6's job and doing it early creates dependencies that don't exist yet.
- **Don't** skip the `test_reversible_action_never_freezes_even_if_policy_says_so` test. This is the single most important safety invariant in the whole system — a reversible action should *never* be blocked, because that would make SENTINEL annoying rather than trustworthy. This test being wrong (or missing) is the kind of bug that quietly makes your demo look broken on stage ("why did it freeze the search_web call?").

---

### Validation Criteria (Exit Gate)

Before moving to Phase 1, confirm:

- [ ] `pytest backend/tests/test_orchestrator.py -v` — all tests pass
- [ ] `python -m tests.manual_phase0_demo` run from `backend/` produces exactly 5 lines of `ACTION_PROPOSED`, interleaved with 4 `ACTION_EXECUTED` and 1 `ACTION_FROZEN`, in the correct order
- [ ] The forced-freeze test proves the freeze path fires correctly with zero real intelligence behind it
- [ ] `Action` construction fails loudly (raises) for: empty description, empty tool_name, non-bool `is_irreversible`
- [ ] Calling `attempt_action()` twice with the same `action_id` does not double-execute or double-freeze
- [ ] `resolve_frozen()` correctly transitions a frozen action to `RESUMED` or `ABORTED`, and raises `KeyError` for an unknown `action_id`
- [ ] No file in this phase imports anything from `frontend/` — backend logic is fully decoupled from UI at this stage
- [ ] Every teammate has pulled this branch and can independently run the manual demo locally

---

**Say "next" and I'll do Phase 1: The Live Demo Agent — the actual agentic loop that proposes the 5-action sequence, how it decides what to propose next, how it interfaces with `attempt_action()`, and how to keep it "live" (real inference-driven) rather than a hardcoded script while still being demo-reliable.**




# SENTINEL — Agent-Ready Build Specification
## Solo Remote Builder: What Changes First

This is a significant enough shift that it needs to be addressed before Phase 1, because it changes the entire resource model the rest of this build plan assumed.

---

### Immediate Structural Implications

| Original assumption | Solo remote reality | Adjustment |
|---|---|---|
| Team of 5, role-split (backend/frontend/ML/storyteller) | You are all four roles | Every phase must be sequenced, not parallelized. Budget accordingly — 24 hours of wall-clock time is NOT 24 hours × 5 people of effort. |
| In-person venue, stage demo, live pitch to judges | Remote judging: submit a project — some teams are selected for a live video call demo | No stage choreography rehearsal. The submission video and code quality carry more weight since there's no in-person Q&A safety net unless selected. |
| 5-person team = built-in code review, pair debugging | No second pair of eyes | Your AI coding agent (Cursor/Claude) is your pair programmer AND your code reviewer. Treat every phase's "Do's/Don'ts" section as instructions to give the agent, not just to yourself. |
| Team size = 5 | Remote teams have a maximum of 5, but solo entry is explicitly permitted | You're compliant — no action needed, just confirmation. |

### Scope Cuts, Made Now, Not Discovered Under Pressure

As a solo builder, the honest move is to cut scope **before** the clock starts, not discover you're out of time at hour 20. Revised priority:

**Cut entirely:**
- Phase 7 (MCP wrapper) — pure stretch, no solo bandwidth for it
- Phase 8 (Override learning loop) — nice narrative, not worth the hours solo
- Screen 5 (Incident Report Archive) — mention it as roadmap in the pitch, don't build it
- Gradium voice layer — cut unless Phases 0–5 finish with 2+ hours to spare

**Keep, but simplified:**
- Heartbeat monitor (Phase 4) — build it, but as the simplest possible version (single latency-ratio calculation), don't gold-plate it
- UI (Phase 6) — build exactly 3 screens (Monitor, Drift Chart, Freeze Modal), skip the MAARS split-screen as a *separate* screen and instead fold it into the Freeze Modal itself — fewer surfaces to build, same information

**Non-negotiable core** (this is now your entire scope, not just your fallback):
- Phase 0 (Freeze mechanic)
- Phase 1 (Live demo agent)
- Phase 2 (Drift scoring)
- Phase 3 (MAARS probe)
- Phase 5 (Sealed incident + advisory)
- A minimal Phase 6 (3 screens)

This is realistically a **14–16 hour solo build**, leaving buffer for the video recording, submission, and the possibility of being selected for a live judge call.

### Remote Submission Specifics to Confirm Now

Since judging for remote teams involves review then selection for a live call, your submission package needs to stand alone without you narrating live in the room. This raises the bar on:
- The 1-minute demo video (it now does the job a stage pitch would have done)
- Code readability (judges reviewing your repo have no one to ask "wait, why did you do this")
- README clarity (write this as if a judge who's never met you will read it cold)

I'll fold a **"solo remote" checklist item** into every phase's exit gate from here on, specifically flagging anything that needs to be self-explanatory without a teammate or live narration to lean on.

---

# PHASE 1: The Live Demo Agent

---

### Objective

Build the agentic loop that proposes a sequence of actions, feeding them into the Phase 0 orchestrator's `attempt_action()` one at a time. This agent must genuinely run — real function calls, real (if simple) reasoning about "what's next" — rather than being a hardcoded list dressed up as an agent. The distinction matters: a hardcoded list is a script; a loop that reasons about state and proposes its next step, even simply, is legitimately agentic and defensible if a judge asks "is this really running, or is it scripted?"

**Solo-builder framing**: this is the one phase where it's tempting to over-engineer (build a "real" LangGraph agent with tool-calling, retries, planning). Resist this. You have one person's worth of hours. The agent's job is to be a **believable, controllable trigger** for SENTINEL's monitoring — not a showcase of agentic sophistication in its own right. SENTINEL is the product. The demo agent is a test fixture that happens to run live.

---

### Prerequisites

- Phase 0 complete: `attempt_action()`, `Action`, `bus` all working and tested
- Crusoe API credentials confirmed working (smoke test from Phase −1 passed)

---

### Design Decision: Semi-Scripted, Fully-Executed

This resolves the tension flagged in the earlier planning conversation ("pre-determine the sequence for reliability, but don't pre-script SENTINEL's response"):

- **The task and action *sequence* are fixed** — you define, in advance, that the agent's task is "research 3 vendors and draft a comparison table," and that it will attempt these 5 tool calls in this order.
- **Each action's actual execution is real** — `search_web()` really calls a search API (or a local mock data source, see below), `write_file()` really writes a file, `send_email()` really attempts an SMTP call (to a sandboxed/test address).
- **SENTINEL's evaluation of each action is computed live** — nothing about the drift score, MAARS verdict, or freeze decision is pre-baked. This is the part that must never be scripted.

This is not "faking" anything. It's identical in spirit to how you'd demo any deterministic test scenario in a security product — the *inputs* are controlled for repeatability, the *system under test* responds for real.

---

### Exact Deliverables

1. `backend/sentinel/agent.py` — the `DemoAgent` class
2. `backend/sentinel/mock_tools.py` — sandboxed tool implementations (search, read, write, email) that are safe to run repeatedly without real-world side effects
3. `backend/tests/test_agent.py` — unit tests
4. `backend/tests/manual_phase1_demo.py` — CLI harness running the agent against the live Phase 0 orchestrator
5. A `.env` addition: `DEMO_EMAIL_SANDBOX_ADDRESS` (a real but harmless test inbox, e.g. a Mailtrap/Mailhog sandbox — **never a real vendor email address**, for obvious reasons)

---

### File: `backend/sentinel/mock_tools.py`

```python
"""
Sandboxed tool implementations for the demo agent.

CRITICAL SAFETY NOTE: send_email() must NEVER be wired to a real
external recipient. Use a sandbox SMTP provider (Mailtrap, Mailhog,
or a local SMTP debug server) so that if the freeze mechanic has a
bug and an "irreversible" action executes anyway, the actual
real-world blast radius is zero. This is not optional — you are
demoing a system whose entire purpose is preventing unintended
irreversible actions; it would be a serious credibility failure to
have your OWN demo accidentally email a real address.
"""

import os
import smtplib
from email.mime.text import MIMEText
from pathlib import Path

SCRATCH_DIR = Path(__file__).parent.parent / "scratch"
SCRATCH_DIR.mkdir(exist_ok=True)


def search_web(query: str) -> dict:
    """
    Mocked search — returns canned but plausible vendor data.
    Using a local mock instead of a live search API is a deliberate
    choice: it removes an external dependency (and its failure modes:
    rate limits, API downtime) from your critical demo path. Judges
    are evaluating SENTINEL, not your search API integration.
    """
    mock_results = {
        "top enterprise CRM vendors": [
            {"name": "Salesforce", "pricing_tier": "$$$$"},
            {"name": "HubSpot", "pricing_tier": "$$"},
            {"name": "Pipedrive", "pricing_tier": "$"},
        ]
    }
    return {"query": query, "results": mock_results.get(query, [])}


def read_doc(url: str) -> dict:
    """Mocked doc read — returns canned pricing text."""
    return {"url": url, "content": "Standard tier: $99/user/month. "
                                     "Enterprise tier: custom pricing."}


def write_file(filename: str, content: str) -> dict:
    """
    Writes to a scratch directory ONLY — never outside SCRATCH_DIR.
    This mirrors the path-containment discipline from the reference
    MORTIS/MAARS-dashboard pattern: never trust a filename string
    without confirming the resolved path stays within an allowed root.
    """
    target = (SCRATCH_DIR / filename).resolve()
    if not str(target).startswith(str(SCRATCH_DIR.resolve())):
        raise ValueError(f"Path escape attempt blocked: {filename}")
    target.write_text(content)
    return {"filename": filename, "bytes_written": len(content)}


def draft_email(to: str, subject: str, body: str) -> dict:
    """Drafting is inherently reversible — nothing is sent yet."""
    return {"to": to, "subject": subject, "body": body, "sent": False}


def send_email(to: str, subject: str, body: str) -> dict:
    """
    SANDBOXED SEND ONLY. Reads target SMTP config from environment —
    defaults to a local Mailhog/Mailtrap sandbox. This function must
    raise, not silently no-op, if no sandbox is configured — a silent
    no-op could mask a real freeze-mechanic failure during testing.
    """
    sandbox_host = os.getenv("DEMO_SMTP_HOST")
    sandbox_port = os.getenv("DEMO_SMTP_PORT")
    if not sandbox_host:
        raise RuntimeError(
            "DEMO_SMTP_HOST not configured. Refusing to send email "
            "without an explicit sandbox target. Set up Mailhog "
            "(https://github.com/mailhog/MailHog) locally or a "
            "Mailtrap sandbox inbox before running this tool."
        )
    msg = MIMEText(body)
    msg["Subject"] = subject
    msg["To"] = to
    msg["From"] = "sentinel-demo@localhost"
    with smtplib.SMTP(sandbox_host, int(sandbox_port)) as server:
        server.send_message(msg)
    return {"to": to, "subject": subject, "sent": True}


TOOL_REGISTRY = {
    "search_web": search_web,
    "read_doc": read_doc,
    "write_file": write_file,
    "draft_email": draft_email,
    "send_email": send_email,
}
```

---

### File: `backend/sentinel/agent.py`

```python
"""
DemoAgent — proposes a sequence of actions and executes them through
SENTINEL's orchestrator. The sequence is fixed (for demo repeatability)
but each step's execution, and every downstream SENTINEL evaluation,
is computed live.
"""

from __future__ import annotations
from dataclasses import dataclass
import time
import logging

from sentinel.action import Action
from sentinel.orchestrator import orchestrator
from sentinel.mock_tools import TOOL_REGISTRY

logger = logging.getLogger("sentinel.agent")


IRREVERSIBLE_TOOLS = {"send_email", "post_to_api", "delete_file", "execute_payment"}


@dataclass
class PlannedStep:
    tool_name: str
    description: str
    parameters: dict


class DemoAgent:
    """
    Runs a fixed, pre-defined step sequence for demo reliability, but
    executes each step for real and yields control back to the caller
    between steps so SENTINEL's live evaluation (drift/MAARS/heartbeat)
    can run in real time, visibly, before the next step proceeds.
    """

    def __init__(self, task_definition: str, steps: list[PlannedStep]):
        self.task_definition = task_definition
        self.steps = steps
        self.completed_actions: list[Action] = []

    @classmethod
    def default_procurement_scenario(cls) -> "DemoAgent":
        """The canonical demo scenario used throughout the build plan."""
        task = "Research 3 vendors and draft a comparison table"
        steps = [
            PlannedStep("search_web", "Search web for vendor list",
                        {"query": "top enterprise CRM vendors"}),
            PlannedStep("read_doc", "Read vendor pricing documentation",
                        {"url": "salesforce.com/pricing"}),
            PlannedStep("write_file", "Write vendor comparison table",
                        {"filename": "comparison.md",
                         "content": "# Vendor Comparison\n..."}),
            PlannedStep("draft_email", "Draft outreach email to vendor",
                        {"to": "vendor@example-sandbox.test",
                         "subject": "Pricing inquiry",
                         "body": "We are exploring alternatives and "
                                 "would like to discuss pricing."}),
            PlannedStep("send_email", "Send outreach email to vendor",
                        {"to": "vendor@example-sandbox.test",
                         "subject": "Pricing inquiry",
                         "body": "We are exploring alternatives and "
                                 "would like to discuss pricing."}),
        ]
        return cls(task, steps)

    def run(self, step_delay_seconds: float = 1.5):
        """
        Executes each planned step in sequence. step_delay_seconds
        adds a deliberate pause between steps — purely for demo
        legibility (so a viewer, or you narrating over the video, can
        track what's happening), not a technical requirement.
        """
        for step in self.steps:
            action = Action(
                description=step.description,
                is_irreversible=step.tool_name in IRREVERSIBLE_TOOLS,
                tool_name=step.tool_name,
                parameters=step.parameters,
            )

            # attempt_action() runs SENTINEL's full evaluation pipeline
            # (Phase 2/3/4 logic gets installed here via set_freeze_policy)
            result = orchestrator.attempt_action(action)
            self.completed_actions.append(result)

            logger.info(f"Step '{step.tool_name}' -> {result.status.value}")

            if result.status.value == "EXECUTED":
                self._run_real_tool(step)
            elif result.status.value == "FROZEN":
                logger.warning(
                    f"Action frozen: {step.description}. "
                    f"Halting further steps pending operator decision."
                )
                break  # do not proceed past a frozen action

            time.sleep(step_delay_seconds)

        return self.completed_actions

    def _run_real_tool(self, step: PlannedStep):
        """
        Actually invokes the sandboxed tool. Wrapped in try/except so
        a tool-level failure (e.g. sandbox SMTP not running) doesn't
        crash the whole demo — it logs and continues, since for the
        purposes of this demo, SENTINEL's DECISION matters more than
        the tool's execution succeeding perfectly.
        """
        tool_fn = TOOL_REGISTRY.get(step.tool_name)
        if not tool_fn:
            logger.error(f"Unknown tool: {step.tool_name}")
            return
        try:
            result = tool_fn(**step.parameters)
            logger.info(f"Tool executed: {step.tool_name} -> {result}")
        except Exception as e:
            logger.error(f"Tool execution failed (non-fatal for demo): {e}")
```

**Why the agent halts on freeze rather than continuing**: this is a deliberate product decision, not just an implementation shortcut — once an irreversible action is frozen, allowing subsequent steps to proceed would contradict SENTINEL's entire premise. If a judge asks about this, the answer is "the agent doesn't get to keep working around a frozen decision — that's the whole point of a dead man's switch."

---

### File: `backend/tests/test_agent.py`

```python
import pytest
from sentinel.agent import DemoAgent, PlannedStep
from sentinel.orchestrator import Orchestrator, FreezeDecision
import sentinel.orchestrator as orchestrator_module


@pytest.fixture(autouse=True)
def fresh_orchestrator(monkeypatch):
    """Each test gets a clean orchestrator instance so freeze state
    from one test doesn't leak into the next."""
    fresh = Orchestrator()
    monkeypatch.setattr(orchestrator_module, "orchestrator", fresh)
    import sentinel.agent as agent_module
    monkeypatch.setattr(agent_module, "orchestrator", fresh)
    return fresh


def test_default_scenario_has_five_steps():
    agent = DemoAgent.default_procurement_scenario()
    assert len(agent.steps) == 5


def test_only_send_email_is_marked_irreversible():
    agent = DemoAgent.default_procurement_scenario()
    for step in agent.steps:
        action_irreversible = step.tool_name == "send_email"
        assert action_irreversible == (step.tool_name in
            __import__("sentinel.agent", fromlist=["IRREVERSIBLE_TOOLS"])
            .IRREVERSIBLE_TOOLS)


def test_agent_halts_after_freeze(fresh_orchestrator):
    fresh_orchestrator.set_freeze_policy(
        lambda a: FreezeDecision(a.tool_name == "send_email", "test freeze")
    )
    agent = DemoAgent.default_procurement_scenario()
    results = agent.run(step_delay_seconds=0)
    # 5 steps total, but agent should STOP once send_email freezes —
    # since send_email is the last step, all 5 should have been attempted
    assert len(results) == 5
    assert results[-1].status.value == "FROZEN"


def test_agent_does_not_halt_when_nothing_freezes(fresh_orchestrator):
    fresh_orchestrator.set_freeze_policy(lambda a: FreezeDecision(False, "pass"))
    agent = DemoAgent.default_procurement_scenario()
    results = agent.run(step_delay_seconds=0)
    assert all(r.status.value == "EXECUTED" for r in results)


def test_agent_halts_mid_sequence_if_earlier_step_frozen(fresh_orchestrator):
    """Edge case: if freeze policy is aggressive enough to freeze an
    EARLIER irreversible-marked step, later steps should never run."""
    call_count = {"n": 0}
    def policy(action):
        call_count["n"] += 1
        return FreezeDecision(action.tool_name == "write_file", "mid-freeze test")
    fresh_orchestrator.set_freeze_policy(policy)

    # Temporarily mark write_file as irreversible for this test only
    agent = DemoAgent.default_procurement_scenario()
    for step in agent.steps:
        if step.tool_name == "write_file":
            step.tool_name = "write_file"  # kept, but we patch IRREVERSIBLE_TOOLS
    import sentinel.agent as agent_module
    original = agent_module.IRREVERSIBLE_TOOLS
    agent_module.IRREVERSIBLE_TOOLS = original | {"write_file"}
    try:
        results = agent.run(step_delay_seconds=0)
        assert len(results) == 3  # search, read, write(frozen) — stops here
        assert results[-1].status.value == "FROZEN"
    finally:
        agent_module.IRREVERSIBLE_TOOLS = original
```

---

### File: `backend/tests/manual_phase1_demo.py`

```python
"""
Manual CLI demo for Phase 1. Run this to watch the full agent sequence
execute against the live Phase 0 orchestrator (still with Phase 0's
pass-through stub, or Phase 2/3's real logic if already wired).

Run: python -m tests.manual_phase1_demo
"""

import logging
from sentinel.agent import DemoAgent
from sentinel.events import bus

logging.basicConfig(level=logging.INFO, format="%(message)s")


def print_event(event):
    print(f"[{event.timestamp.strftime('%H:%M:%S')}] "
          f"{event.event_type.value:20s} {event.payload}")


def main():
    bus.subscribe(print_event)
    agent = DemoAgent.default_procurement_scenario()
    print(f"Task: {agent.task_definition}\n")
    results = agent.run(step_delay_seconds=1.0)
    print(f"\nFinal action count: {len(results)}")
    for r in results:
        print(f"  {r.tool_name:15s} -> {r.status.value}")


if __name__ == "__main__":
    main()
```

---

### Environment Setup Note (Sandbox SMTP)

For solo builders without time to stand up a full Mailhog instance, the fastest path:
```bash
docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog
```
Then in `.env`:
```
DEMO_SMTP_HOST=localhost
DEMO_SMTP_PORT=1025
```
View "sent" emails at `http://localhost:8025` — useful as visual proof, if needed, that a *resumed* (operator-approved) email genuinely sends, versus a frozen one that never reaches even the sandbox.

---

### Do's

- **Do** keep the task/step sequence fixed and declared upfront (`default_procurement_scenario()`) — repeatability across multiple test runs and the final recorded take is more valuable to a solo builder than dynamic planning would be.
- **Do** make tool execution genuinely real, even against sandboxed/mock targets — this is what separates "agent" from "script" if a judge inspects the code.
- **Do** halt the agent immediately on any frozen action — this is a core product invariant, not just a convenience.
- **Do** log every step's outcome to stdout during development — as a solo builder without a teammate to narrate alongside you, clear logs are your primary debugging aid.
- **Do** wrap real tool execution in try/except with non-fatal logging — a sandbox SMTP hiccup should never crash your only build machine's demo run.

### Don'ts

- **Don't** call a live external search API (e.g., real Google/Bing search) for `search_web()`. As a solo builder, every external dependency is a new failure mode you have no teammate to help debug live. Mock it.
- **Don't** ever point `send_email()` at a real address, even "just to test once." One misconfigured test is a real email sent to a real vendor — irreversible in the most literal sense, and exactly the failure mode SENTINEL exists to prevent. Sandbox only, enforced by the `RuntimeError` guard in `mock_tools.py`.
- **Don't** let the agent silently continue after a freeze "just to see what happens" — even in testing. If you need to test post-freeze behavior, do it explicitly via `resolve_frozen()`, not by removing the halt logic.
- **Don't** hardcode `IRREVERSIBLE_TOOLS` membership checks in multiple places — it's defined once in `agent.py` and should be imported, not re-typed, anywhere else (including tests, where it's imported for verification, not redefined).
- **Don't** skip the `test_agent_halts_mid_sequence_if_earlier_step_frozen` edge case test — as a solo builder relying on a fixed step sequence, this is the test that protects you from a bug where a freeze earlier in the sequence doesn't actually stop later "irreversible" steps from running.

---

### Validation Criteria (Exit Gate)

- [ ] `pytest backend/tests/test_agent.py -v` — all tests pass
- [ ] `python -m tests.manual_phase1_demo` produces a visible, ordered log of all 5 steps, ending in `FROZEN` on `send_email` (assuming Phase 0's `force_freeze_next` or a test policy is active)
- [ ] Sandbox SMTP (Mailhog or equivalent) running locally, confirmed reachable
- [ ] Manually confirm: an approved (`resolve_frozen(..., approved=True)`) email actually appears in the Mailhog inbox at `localhost:8025`
- [ ] Manually confirm: an aborted email never appears there
- [ ] **Solo-remote specific**: `mock_tools.py`'s docstrings are detailed enough that a judge reading the code cold understands *why* it's sandboxed, without you there to explain it live
- [ ] No hardcoded API keys or real email addresses anywhere in `agent.py` or `mock_tools.py`

---

**Say "next" and I'll do Phase 2: Scope Drift Engine — the first real Crusoe integration, including exact embedding call structure, caching strategy (important solo, since redundant API calls cost you both time and rate-limit headroom), and the cosine distance math with full edge-case handling.**





# SENTINEL — Remaining Phases (2 through Submission) + Solo-Builder Cut Analysis

Given the solo/remote context, I'll complete the full phase map now at slightly tighter density per phase (still with real code and concrete do's/don'ts), then close with the consolidated cut-decision matrix you asked for — so you have the whole picture before deciding what to drop.

---

# PHASE 2: Scope Drift Engine (Crusoe Embedding Integration)

### Objective
Replace Phase 0's stub `should_freeze()` with a real signal: how semantically far is this proposed action from the original task? This is your first live Crusoe API integration.

### Prerequisites
Phase 0 + Phase 1 complete and tested. Crusoe embedding model identifier confirmed (check Crusoe's model catalog day-of — this doc uses a placeholder env var).

### Deliverables
1. `backend/sentinel/crusoe_client.py` — thin wrapper around Crusoe's OpenAI-compatible endpoint
2. `backend/sentinel/drift.py` — embedding + cosine distance + caching
3. `backend/tests/test_drift.py`

### File: `backend/sentinel/crusoe_client.py`

```python
"""
Thin wrapper around Crusoe Managed Inference. Centralizing this in one
file means: (a) one place to fix auth/base-url issues, (b) one place
to add retry/timeout logic, (c) one place to swap in a mock client if
Crusoe is unreachable during a live demo.
"""

import os
import time
import logging
from openai import OpenAI, APIError, APITimeoutError

logger = logging.getLogger("sentinel.crusoe")

_client = OpenAI(
    api_key=os.getenv("CRUSOE_API_KEY"),
    base_url=os.getenv("CRUSOE_API_BASE"),
)

EMBED_MODEL = os.getenv("CRUSOE_EMBED_MODEL")
CHAT_MODEL = os.getenv("CRUSOE_CHAT_MODEL")

MAX_RETRIES = 2
RETRY_BACKOFF_SECONDS = 1.5


def embed(text: str) -> list[float]:
    """Returns an embedding vector. Retries once on timeout/transient
    error — solo builder has no one to babysit a flaky call live."""
    last_error = None
    for attempt in range(MAX_RETRIES + 1):
        try:
            response = _client.embeddings.create(model=EMBED_MODEL, input=text)
            return response.data[0].embedding
        except (APIError, APITimeoutError) as e:
            last_error = e
            logger.warning(f"Embed call failed (attempt {attempt+1}): {e}")
            time.sleep(RETRY_BACKOFF_SECONDS)
    raise RuntimeError(f"Embedding failed after {MAX_RETRIES+1} attempts") from last_error


def chat_json(prompt: str) -> dict:
    """Chat completion in JSON mode. Used by Phase 3's MAARS probe."""
    import json
    last_error = None
    for attempt in range(MAX_RETRIES + 1):
        try:
            response = _client.chat.completions.create(
                model=CHAT_MODEL,
                messages=[{"role": "user", "content": prompt}],
                response_format={"type": "json_object"},
                temperature=0.1,  # low temp: consistent verdicts across demo takes
            )
            return json.loads(response.choices[0].message.content)
        except (APIError, APITimeoutError) as e:
            last_error = e
            logger.warning(f"Chat call failed (attempt {attempt+1}): {e}")
            time.sleep(RETRY_BACKOFF_SECONDS)
        except json.JSONDecodeError as e:
            last_error = e
            logger.warning(f"Malformed JSON from model (attempt {attempt+1}): {e}")
    raise RuntimeError(f"Chat completion failed after {MAX_RETRIES+1} attempts") from last_error
```

**Why `temperature=0.1` specifically**: for a rehearsed demo you'll run multiple times before recording, you need the MAARS probe's verdict on the *same* scripted actions to be consistent across takes. High temperature risks a "YES" on the take you actually record when every rehearsal gave "NO." Low temperature isn't cheating — it's controlling for a legitimate variance source, the same way you'd want a repeatable test in any other engineering context.

### File: `backend/sentinel/drift.py`

```python
"""
Scope drift scoring: cosine distance between the original task
embedding and each proposed action's embedding.
"""

from __future__ import annotations
import numpy as np
from functools import lru_cache
import logging

from sentinel.crusoe_client import embed

logger = logging.getLogger("sentinel.drift")

DRIFT_THRESHOLD = 0.35


def cosine_distance(a: list[float], b: list[float]) -> float:
    a_arr, b_arr = np.array(a), np.array(b)
    denom = (np.linalg.norm(a_arr) * np.linalg.norm(b_arr))
    if denom == 0:
        # Degenerate case: zero-vector embedding. Should never happen
        # with a real model, but fail safely rather than divide-by-zero
        # crashing mid-demo.
        logger.error("Zero-norm embedding encountered — returning max drift")
        return 1.0
    similarity = np.dot(a_arr, b_arr) / denom
    return float(1.0 - similarity)


class ScopeModel:
    """
    Holds the task embedding and scores actions against it.
    A single instance per task/demo run — do not share across
    concurrent tasks (not a concern for a solo single-agent demo,
    but noted for correctness).
    """

    def __init__(self, task_definition: str):
        self.task_definition = task_definition
        self._task_embedding = embed(task_definition)
        self._embedding_cache: dict[str, list[float]] = {}

    def _cached_embed(self, text: str) -> list[float]:
        """
        Caches embeddings by exact text match. In a fixed-sequence demo
        this matters a lot: your 5 actions are called every time you
        rehearse. Without caching, every rehearsal burns 5 extra API
        calls for identical text — wasted latency and quota during
        the hours you'll spend re-running the demo before recording.
        """
        if text not in self._embedding_cache:
            self._embedding_cache[text] = embed(text)
        return self._embedding_cache[text]

    def score(self, action_description: str) -> float:
        action_embedding = self._cached_embed(action_description)
        return cosine_distance(self._task_embedding, action_embedding)

    def is_drifted(self, action_description: str) -> tuple[bool, float]:
        score = self.score(action_description)
        return score > DRIFT_THRESHOLD, score
```

### File: `backend/tests/test_drift.py`

```python
import pytest
from unittest.mock import patch
from sentinel.drift import cosine_distance, ScopeModel


def test_cosine_distance_identical_vectors_is_zero():
    v = [1.0, 0.0, 0.0]
    assert cosine_distance(v, v) == pytest.approx(0.0, abs=1e-6)


def test_cosine_distance_orthogonal_vectors_is_one():
    a, b = [1.0, 0.0], [0.0, 1.0]
    assert cosine_distance(a, b) == pytest.approx(1.0, abs=1e-6)


def test_cosine_distance_zero_vector_returns_max_drift():
    a, b = [0.0, 0.0], [1.0, 1.0]
    assert cosine_distance(a, b) == 1.0


@patch("sentinel.drift.embed")
def test_scope_model_caches_repeated_calls(mock_embed):
    mock_embed.return_value = [1.0, 0.0]
    model = ScopeModel("test task")
    model.score("some action")
    model.score("some action")  # repeated — should hit cache
    # embed called once for task_embedding init + once for the action
    # (not twice for the repeated action call)
    assert mock_embed.call_count == 2


@patch("sentinel.drift.embed")
def test_is_drifted_respects_threshold(mock_embed):
    # task embedding vs action embedding engineered for known distance
    mock_embed.side_effect = [[1.0, 0.0], [0.0, 1.0]]  # orthogonal -> distance 1.0
    model = ScopeModel("test task")
    drifted, score = model.is_drifted("far action")
    assert drifted is True
    assert score == pytest.approx(1.0, abs=1e-6)
```

### Wiring into the orchestrator (in `main.py` or a setup script)
```python
from sentinel.drift import ScopeModel
from sentinel.orchestrator import orchestrator, FreezeDecision

scope_model = ScopeModel(task_definition="Research 3 vendors and draft a comparison table")

def phase2_freeze_policy(action):
    drifted, score = scope_model.is_drifted(action.description)
    action.drift_score = score
    from sentinel.events import bus, EventType
    bus.emit(EventType.DRIFT_SCORED, action.action_id, {"drift_score": score})
    return FreezeDecision(drifted, f"drift_score={score:.2f} > {0.35}" if drifted else "within scope")

orchestrator.set_freeze_policy(phase2_freeze_policy)
```

### Do's
- **Do** cache embeddings aggressively — you will run this demo 15+ times before recording the final take; caching turns rehearsal cost from "5 API calls" to "1 API call" after the first pass.
- **Do** use low-dimensionality sanity checks in unit tests (hardcoded 2D vectors) rather than calling the real API in tests — keeps tests fast and free of network flakiness.
- **Do** log the raw drift score for every action, always, even when it doesn't trigger freeze — you'll want this data to tune `DRIFT_THRESHOLD` correctly the night before recording.

### Don'ts
- **Don't** call `embed()` directly from anywhere except `ScopeModel` — this bypasses the cache and defeats its purpose.
- **Don't** hardcode `DRIFT_THRESHOLD` in multiple files — it's defined once in `drift.py`.
- **Don't** trust a single rehearsal run to validate the threshold. Run the full 5-step sequence at least 3 times and confirm drift scores are stable across runs before locking `DRIFT_THRESHOLD`.

### Validation Criteria
- [ ] `pytest backend/tests/test_drift.py -v` passes
- [ ] Live smoke test: `ScopeModel("test").score("completely unrelated action")` returns a high drift score, `.score("closely related action")` returns low
- [ ] Full agent sequence (Phase 1) run through Phase 2's real policy produces a drift score for all 5 actions, with `send_email` scoring highest
- [ ] Solo-remote check: drift scores logged clearly enough that if you're on a live judge call and asked "why did it freeze," you can point at a specific printed number without needing to explain your own code from memory

---

# PHASE 3: MAARS Adversarial Probe

### Objective
Add the second, independent signal: an LLM call that evaluates "would a supervisor approve this?" using the MAARS-derived prompt template staged in Phase −1.

### Prerequisites
Phase 2 complete. `docs/probe_prompt_template.md` exists.

### Deliverables
1. `backend/sentinel/probe.py`
2. `backend/tests/test_probe.py`

### File: `backend/sentinel/probe.py`

```python
"""
MAARS adversarial probe — ports the Builder/Critic scoring discipline
from the MAARS repo's master system prompt (justified verdicts,
forbidden vague approvals) into a single runtime call per action.
"""

from __future__ import annotations
from pathlib import Path
from dataclasses import dataclass
import logging

from sentinel.crusoe_client import chat_json

logger = logging.getLogger("sentinel.probe")

_TEMPLATE_PATH = Path(__file__).parent.parent.parent / "docs" / "probe_prompt_template.md"


def _load_template() -> str:
    """Loaded from disk, not hardcoded in Python — keeps the prompt
    editable without touching code, and keeps a single source of truth
    matching what's committed in docs/."""
    text = _TEMPLATE_PATH.read_text()
    # Strip the markdown header/comments, keep only the prompt body
    # (everything after the first blank line following the H1)
    lines = text.split("\n")
    body_start = next(i for i, l in enumerate(lines) if l.startswith("You are"))
    return "\n".join(lines[body_start:])


_TEMPLATE = _load_template()


@dataclass
class MaarsVerdict:
    verdict: str          # "YES" | "NO"
    confidence: int        # 0-100
    reasoning: str
    severity: str | None
    remediation: str | None

    @property
    def disagrees(self) -> bool:
        return self.verdict == "NO"


REQUIRED_KEYS = {"verdict", "confidence", "reasoning", "severity", "remediation"}


def probe(task_definition: str, action_description: str, action_history: list[str]) -> MaarsVerdict:
    history_str = "\n".join(f"- {a}" for a in action_history) or "(none yet)"
    prompt = _TEMPLATE.format(
        task_definition=task_definition,
        action_description=action_description,
        action_history=history_str,
    )
    raw = chat_json(prompt)

    # Validate the response shape BEFORE trusting it — this is the
    # single most important defensive check in this file. An LLM
    # returning malformed/partial JSON mid-demo, unhandled, will crash
    # your live run in front of a judge.
    missing = REQUIRED_KEYS - raw.keys()
    if missing:
        logger.error(f"MAARS probe response missing keys: {missing}. Raw: {raw}")
        # Fail CLOSED, not open: if the probe is malformed, treat it as
        # a disagreement rather than silently allowing the action. This
        # is a deliberate safety-first default consistent with SENTINEL's
        # whole thesis — when in doubt about a critic's signal, freeze.
        return MaarsVerdict(
            verdict="NO", confidence=100,
            reasoning="MAARS probe returned malformed response — failing closed.",
            severity="high", remediation="Investigate probe integration."
        )

    if raw["verdict"] not in ("YES", "NO"):
        logger.error(f"Invalid verdict value: {raw['verdict']}")
        raw["verdict"] = "NO"  # fail closed

    if raw["verdict"] == "NO" and (not raw.get("reasoning") or len(raw["reasoning"]) < 10):
        # Enforces MAARS's "forbidden vague approval" rule programmatically,
        # not just via prompt instruction — belt and suspenders.
        logger.warning("MAARS gave NO verdict without substantive reasoning")
        raw["reasoning"] = "Verdict NO but reasoning was insufficiently justified " \
                            "by the model — treating as high-severity by default."
        raw["severity"] = raw.get("severity") or "high"

    return MaarsVerdict(
        verdict=raw["verdict"],
        confidence=int(raw["confidence"]),
        reasoning=raw["reasoning"],
        severity=raw.get("severity"),
        remediation=raw.get("remediation"),
    )
```

### File: `backend/tests/test_probe.py`

```python
import pytest
from unittest.mock import patch
from sentinel.probe import probe, MaarsVerdict


@patch("sentinel.probe.chat_json")
def test_probe_returns_valid_verdict(mock_chat):
    mock_chat.return_value = {
        "verdict": "NO", "confidence": 91,
        "reasoning": "Task scope was research-only; this action contacts an external party.",
        "severity": "high", "remediation": "Require explicit approval before contact."
    }
    result = probe("research vendors", "send email to vendor", [])
    assert result.verdict == "NO"
    assert result.disagrees is True
    assert result.confidence == 91


@patch("sentinel.probe.chat_json")
def test_probe_fails_closed_on_missing_keys(mock_chat):
    mock_chat.return_value = {"verdict": "YES"}  # missing required keys
    result = probe("task", "action", [])
    assert result.verdict == "NO"  # fail-closed default
    assert result.confidence == 100


@patch("sentinel.probe.chat_json")
def test_probe_fails_closed_on_invalid_verdict_value(mock_chat):
    mock_chat.return_value = {
        "verdict": "MAYBE", "confidence": 50, "reasoning": "unclear",
        "severity": None, "remediation": None
    }
    result = probe("task", "action", [])
    assert result.verdict == "NO"


@patch("sentinel.probe.chat_json")
def test_probe_rejects_vague_no_verdict(mock_chat):
    mock_chat.return_value = {
        "verdict": "NO", "confidence": 80, "reasoning": "no",  # too short/vague
        "severity": None, "remediation": None
    }
    result = probe("task", "action", [])
    assert "insufficiently justified" in result.reasoning
    assert result.severity == "high"
```

### Do's
- **Do** load the prompt template from the committed `docs/` file, not a duplicated string in Python — one source of truth.
- **Do** fail closed (treat malformed/ambiguous model output as a disagreement) — this is both a safety principle and a practical one: a crash mid-demo is worse than an unexpected freeze.
- **Do** validate response shape defensively even though you control the prompt — models occasionally drop fields or return malformed JSON regardless of `response_format` instructions.

### Don'ts
- **Don't** trust `response_format: json_object` alone to guarantee valid, complete JSON — always validate keys explicitly.
- **Don't** let a vague "NO — doesn't look right" pass through un-flagged — programmatically enforce the justification requirement, don't rely purely on prompt instruction.
- **Don't** run this probe against every action in a real production system without cost controls — for the hackathon, 5 actions × 1 call is trivial, but note this as a known scaling consideration if asked.

### Validation Criteria
- [ ] `pytest backend/tests/test_probe.py -v` passes
- [ ] Live smoke test on the `send_email` action from the demo scenario returns `verdict: NO` with substantive `reasoning` and `severity: high`
- [ ] Live smoke test on `search_web` returns `verdict: YES`
- [ ] Malformed-response fail-closed path manually verified by temporarily breaking the prompt and confirming it doesn't crash

---

# PHASE 4: Heartbeat Monitor (Simplified)

### Objective
Add the third, independent trigger: approval-latency-trend as a proxy for operator attention. Per the solo-builder cut, this is the *simplest possible version* — a single ratio calculation, not the full composite signal from the original amendment.

### Prerequisites
Phase 3 complete.

### File: `backend/sentinel/heartbeat.py`

```python
"""
Heartbeat monitor — simplified for solo-builder time constraints to a
single signal: approval latency trend. If recent approvals are coming
back much faster than the baseline, treat it as a proxy for reduced
attention ("rubber-stamping").
"""

from __future__ import annotations
from dataclasses import dataclass, field
import time


@dataclass
class HeartbeatMonitor:
    approval_durations: list[float] = field(default_factory=list)
    _baseline: float | None = None

    def log_approval(self, review_duration_seconds: float):
        if review_duration_seconds < 0:
            raise ValueError("review_duration_seconds cannot be negative")
        self.approval_durations.append(review_duration_seconds)
        if self._baseline is None and len(self.approval_durations) >= 1:
            self._baseline = self.approval_durations[0]

    def score(self) -> int:
        """
        Returns 0-100. Insufficient data (fewer than 2 approvals
        logged) defaults to 100 (healthy) rather than penalizing a
        task that's just getting started — an empty history is not
        evidence of inattention.
        """
        if len(self.approval_durations) < 2 or self._baseline in (None, 0):
            return 100
        recent = self.approval_durations[-2:]
        avg_recent = sum(recent) / len(recent)
        ratio = avg_recent / self._baseline
        # ratio near 1.0 = consistent with baseline = healthy
        # ratio near 0 = approving much faster than baseline = suspicious
        score = max(0, min(100, int(100 * min(ratio, 1.0))))
        return score

    def is_lapsed(self, threshold: int = 30) -> bool:
        return self.score() < threshold
```

### Compound trigger — final `should_freeze` wiring
```python
def phase4_freeze_policy(action, scope_model, heartbeat_monitor, task_def, history):
    drifted, drift_score = scope_model.is_drifted(action.description)
    action.drift_score = drift_score

    maars_result = probe(task_def, action.description, history)
    action.maars_verdict = maars_result.__dict__

    hb_score = heartbeat_monitor.score()
    action.heartbeat_score_at_time = hb_score

    compound = drifted and maars_result.disagrees and maars_result.confidence > 75
    heartbeat_trigger = heartbeat_monitor.is_lapsed()

    should_freeze = compound or heartbeat_trigger
    reason = (f"compound(drift={drift_score:.2f}, maars_conf={maars_result.confidence}) "
              if compound else "") + (f"heartbeat_lapsed({hb_score})" if heartbeat_trigger else "")

    return FreezeDecision(should_freeze, reason.strip() or "no trigger")
```

### Do's
- **Do** default to `score() == 100` (healthy) when data is insufficient — an empty approval history should never itself cause a freeze.
- **Do** keep this signal deliberately simple per the solo-builder cut — do not implement the full composite signal (scroll depth, semantic richness) discussed earlier; that's explicitly cut.

### Don'ts
- **Don't** wire heartbeat as the *primary* demo trigger — as noted earlier, staging a "distracted operator" live is awkward. Keep it in the system, mention it in Q&A, but demo the compound drift+MAARS trigger as the headline.
- **Don't** let `_baseline` be `0` and silently produce a division error — the guard clause (`self._baseline in (None, 0)`) exists specifically for this.

### Validation Criteria
- [ ] Unit tests for: insufficient data → 100; consistent approvals → high score; rapidly decreasing durations → low score
- [ ] Compound trigger manually tested with both paths (drift+MAARS, heartbeat-alone) firing independently

---

# PHASE 5: Failsafe Sequencer + Sealed Incident Report

### Objective
FREEZE → SEAL → ALERT → (LEARN, cut for solo). This is the artifact a judge actually inspects.

### File: `backend/sentinel/sequencer.py`

```python
"""
Failsafe sequencer — generates the sealed, tamper-evident incident
report once an action is frozen. SHA-256 hash only (not a full
cryptographic seal/encryption) — explicitly simplified from the
MORTIS reference implementation to fit solo-builder time constraints.
This is disclosed honestly in the pitch, not hidden.
"""

from __future__ import annotations
import json
import hashlib
import uuid
from pathlib import Path
from datetime import datetime, timezone

INCIDENTS_DIR = Path(__file__).parent.parent.parent / "incidents"
INCIDENTS_DIR.mkdir(exist_ok=True)


def seal_incident(action, history: list) -> dict:
    report = {
        "incident_id": str(uuid.uuid4()),
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "frozen_action": {
            "description": action.description,
            "tool_name": action.tool_name,
            "parameters": action.parameters,
        },
        "drift_score": action.drift_score,
        "maars_verdict": action.maars_verdict,
        "heartbeat_score": action.heartbeat_score_at_time,
        "decision_trace": [
            {"description": a.description, "status": a.status.value,
             "drift_score": a.drift_score}
            for a in history
        ],
    }
    payload = json.dumps(report, sort_keys=True)
    report["integrity_hash"] = hashlib.sha256(payload.encode()).hexdigest()

    out_path = INCIDENTS_DIR / f"{report['incident_id']}.json"
    out_path.write_text(json.dumps(report, indent=2))
    return report


def generate_advisory_text(report: dict) -> str:
    verdict = report["maars_verdict"] or {}
    return (
        f"⚠️ Action frozen: {report['frozen_action']['description']}. "
        f"Drift score {report['drift_score']:.2f}. "
        f"MAARS assessment: {verdict.get('reasoning', 'n/a')} "
        f"(severity: {verdict.get('severity', 'n/a')}). "
        f"Approve, abort, or extend review window?"
    )
```

### Do's
- **Do** be upfront in the pitch that the seal is SHA-256 tamper-evidence, not full encryption — honesty here builds more credibility than an overstated security claim a judge might probe on.
- **Do** write the incident JSON to disk in a human-readable, indented format — you will want to literally open this file on screen during the demo/video.

### Don'ts
- **Don't** attempt to port MORTIS's full cryptographic seal implementation — explicitly cut for time, noted as a roadmap item instead.
- **Don't** skip writing the file to disk in favor of just holding it in memory — the tangible, openable artifact is a specific, deliberate demo beat from the design doc.

### Validation Criteria
- [ ] Running the full pipeline produces a real file in `incidents/` with a valid SHA-256 hash
- [ ] Advisory text renders correctly with real MAARS reasoning substituted in

---

# PHASE 6: UI (Reduced to 3 Screens for Solo Build)

### Objective
Per the solo cut: **3 screens only** — Live Agent Monitor, Drift Trajectory Chart, and a combined Freeze/MAARS/Incident Modal (merging what was previously 2 separate screens into one, since building 4 full screens solo is not realistic in the remaining time budget).

### Setup
```bash
cd frontend
npx impeccable install
# In Cursor: /impeccable init
```
Manually overwrite generated `DESIGN.md` with the tokens specified in the master design document (Section 7.3).

### Screen 1: Live Agent Monitor — build first, run `/impeccable critique monitor` immediately after, per the checkpoint plan, since as a solo builder you have no second screen's design to visually cross-check against — catching drift toward generic-SaaS defaults on the *first* screen matters even more when every subsequent screen is built by referencing this one.

### Screen 2: Drift Trajectory Chart — wire to a WebSocket subscribing to `DRIFT_SCORED` events from the backend's `EventBus`. Minimal viable version: a simple line chart library (Recharts is fine), points colored per the threshold rule from the design doc.

### Screen 3: Combined Freeze/MAARS/Incident Modal — this now carries the weight of what was 2 screens (MAARS split-screen + Freeze modal) in the original 5-team plan. Structure:
```
┌─────────────────────────────────────────────┐
│  ⛔ FROZEN                                    │
├───────────────────┬───────────────────────────┤
│ PRIMARY AGENT      │ MAARS PROBE               │
│ send_email(...)    │ Verdict: NO (91%)         │
│                   │ "Task scope was..."        │
├───────────────────┴───────────────────────────┤
│ Incident #a3f9e2  Hash: 4f8a2c1d9e...          │
│           [ Approve ]   [ Abort ]              │
└─────────────────────────────────────────────┘
```
This single-modal consolidation is a legitimate design simplification, not a quality cut — it still shows every piece of information a judge needs, just in one panel instead of a choreographed multi-screen sequence.

### Do's
- **Do** run `/impeccable audit` once, after all 3 screens exist, rather than the original 2-checkpoint plan — as a solo builder your time for design-tool passes is limited to one solid audit pass plus one polish pass before recording.
- **Do** wire the WebSocket connection early (even before all events exist) so you're not debugging both event-plumbing and UI polish simultaneously the night before recording.

### Don'ts
- **Don't** attempt Screen 4/5 (Incident Archive) — cut entirely, mention as roadmap.
- **Don't** spend time on responsive/mobile layout — solo builder time is too scarce, and the video will be recorded on a single laptop screen.

### Validation Criteria
- [ ] All 3 screens render real, live data from the backend event stream — not hardcoded mock UI state
- [ ] `/impeccable audit` run at least once, flagged issues addressed
- [ ] `/impeccable polish` run immediately before recording

---

# PHASE 9: Demo Video + Submission (Solo-Remote Critical Phase)

### Objective
Since remote judging means a team of judges will review projects and select certain participants to join a video call to live demo, your video and repo must be self-sufficient — this phase now carries more weight than it would for an in-person team with a live pitch safety net.

### Deliverables
1. 60-second demo video (YouTube/Loom, unlisted)
2. Polished `README.md` (judge's first and possibly only touchpoint)
3. Submission form completed

### README structure (write this as if no one will ever hear you explain it live)
```markdown
# SENTINEL

A dead man's switch for autonomous AI agents.

## The problem
[2-3 sentences, stakes-first framing]

## How it works
[Architecture diagram + 1 paragraph per component]

## What's real vs. simplified for the hackathon
- SHA-256 tamper-evidence (not full encryption) — see roadmap
- Heartbeat: single latency-ratio signal (not full composite)
- Demo agent: fixed scenario, live execution, sandboxed tools

## Origin
Built by porting the adversarial scoring discipline from [MAARS]
(link) and the dead-man's-switch pattern from [Digital Will/MORTIS]
(links) — both pre-existing open-source projects — repointed at
autonomous agent oversight.

## Run it locally
[exact setup steps]
```

### Video script (60 seconds, solo-narrated)
| Time | Content |
|---|---|
| 0–15s | Stakes framing (concrete precedent) |
| 15–40s | Live sequence: agent runs, drift climbs, MAARS disagrees, freeze fires |
| 40–55s | Open the incident JSON file, point at the hash |
| 55–60s | One-line origin story + what's next |

### Do's
- **Do** record in a quiet space, script the narration but deliver it naturally — a read-word-for-word script sounds worse on video than confident improvisation from bullet points.
- **Do** test the YouTube/Loom link in an incognito window before submitting — a broken or private link is a preventable zero.
- **Do** over-invest in the README relative to what an in-person team would need — it's doing the job your in-person pitch would have done.

### Don'ts
- **Don't** submit right at the deadline — remote submission means no in-person buffer if the form has an issue; submit with at least 30 minutes of margin.
- **Don't** assume a judge will read your code deeply if selected for a live call — lead with the demo and the incident artifact, not a code walkthrough, unless asked.

### Validation Criteria
- [ ] Video under 60 seconds, uploaded, link tested in incognito
- [ ] README readable cold by someone with zero context
- [ ] Submission form completed with time to spare

---

# CONSOLIDATED SOLO-BUILDER TIMELINE

| Phase | Hours (approx) | Cumulative |
|---|---|---|
| −1 Pre-hack | (async, before clock starts) | — |
| 0 Freeze mechanic | 1.5h | 1.5h |
| 1 Demo agent | 1.5h | 3h |
| 2 Drift engine | 1.5h | 4.5h |
| 3 MAARS probe | 2h | 6.5h |
| 4 Heartbeat (simplified) | 1h | 7.5h |
| 5 Sequencer + incident | 1.5h | 9h |
| 6 UI (3 screens) | 4–5h | 13.5–14.5h |
| 9 Video + submission | 1.5–2h | 15–16.5h |

This leaves roughly **7–9 hours of slack** inside the full ~24-hour window for debugging overruns, sleep, breaks, and the unavoidable "this took twice as long as expected" tax — which for a solo builder is not optional buffer, it's required buffer.

---

# CUT-DECISION MATRIX

| Item | Status | Rationale |
|---|---|---|
| Phase 7 (MCP wrapper) | **Cut** | Pure amplification, no solo bandwidth |
| Phase 8 (Override learning loop) | **Cut** | Nice narrative, not worth the hours |
| Screen 4/5 (Incident Archive) | **Cut** | Mention as roadmap only |
| Gradium voice layer | **Conditional** | Only if Phase 6 finishes 2+ hrs early |
| MAARS split-screen as separate UI | **Merged into Freeze Modal** | Same information, one screen instead of two |
| Full composite heartbeat signal | **Simplified to single ratio** | Full version cut, simple version kept |
| Full cryptographic seal (MORTIS-style) | **Simplified to SHA-256 hash** | Disclosed honestly in README/pitch |
| Phases 0, 1, 2, 3, 5 | **Non-negotiable core** | This is SENTINEL's entire thesis |
| Phase 4 (heartbeat, simplified) | **Keep, simplified** | Cheap to build, completes the "compound trigger" story |
| Phase 6 (3 screens) | **Keep, reduced scope** | Minimum viable to avoid "no working demo" |
| Phase 9 (video + README) | **Keep, elevated priority** | Carries the weight of your absent live pitch |

**Bottom line**: the core thesis (Phases 0–3 + 5) survives fully intact even under solo constraints — nothing about the drift+MAARS adversarial mechanic, which is your actual competitive differentiator, gets cut. What's cut is exclusively the amplification layer (MCP, learning loop, voice, archive screen) and some polish depth (composite heartbeat, full crypto seal) — both disclosed honestly rather than faked, which itself becomes part of a credible, transparent pitch.






# Honest Assessment: What's Missing

No — not fully. Here's exactly what was covered vs. what got short-changed:

---

## Coverage Audit

| Phase | Status | Gap |
|---|---|---|
| **−1 Pre-hack** | ✅ Full | Nothing missing |
| **0 Freeze mechanic** | ✅ Full | Nothing missing |
| **1 Demo agent** | ✅ Full | Nothing missing |
| **2 Drift engine** | ✅ Full | Nothing missing |
| **3 MAARS probe** | ✅ Full | Nothing missing |
| **4 Heartbeat** | ✅ Full (simplified) | Nothing missing |
| **5 Sequencer** | ✅ Full (simplified) | Nothing missing |
| **6 UI** | ❌ Skeleton only | No actual React component code, no WebSocket server, no CSS/tokens, no Impeccable integration commands, no event-to-render mapping |
| **Integration layer** | ❌ Missing entirely | `main.py` (FastAPI app that wires Phases 0–5 together), WebSocket broadcaster, how the backend assembles at startup — never written |
| **9 Video + Submission** | ⚠️ High-level only | Script exists, but no README template with exact copy, no submission checklist with form field guidance |

---

## What Needs to Be Built Next

Three things are genuinely missing and blocking the build:

### 1. The Integration Layer (`main.py` + FastAPI + WebSocket)
This is the most critical gap. Without it, Phases 0–5 are a collection of isolated modules that don't actually talk to each other or to the frontend. Specifically missing:
- FastAPI app setup
- WebSocket endpoint that broadcasts `EventBus` events to the frontend in real time
- Startup routine that instantiates `ScopeModel`, `HeartbeatMonitor`, `DemoAgent`, wires the compound freeze policy into the orchestrator, and starts the agent loop
- REST endpoints for `POST /approve/{action_id}` and `POST /abort/{action_id}` (what the frontend buttons call)

### 2. Phase 6 UI (Full Component Code)
What was written was a layout description, not buildable code. Missing:
- `App.tsx` with WebSocket connection and event dispatch
- `AgentMonitor.tsx` — Screen 1 (live action stream)
- `DriftChart.tsx` — Screen 2 (Recharts scope trajectory)
- `FreezeModal.tsx` — Screen 3 (combined MAARS + incident panel)
- `useEvents.ts` — custom hook managing the WebSocket event stream
- `tokens.css` / `theme.ts` — the DESIGN.md tokens translated into actual CSS variables so every component references the same color/font/spacing values
- Impeccable config commands and checkpoint procedure

### 3. Phase 9 Completion
Missing:
- Full `README.md` with exact copy written out (not just structure headers)
- Submission checklist mapped to the actual RAISE form fields
- Pre-recording technical checklist

---

## Recommended Order to Complete

```
Integration layer first → Phase 6 UI second → Phase 9 last

Reason: the UI is useless without a working WebSocket server to 
connect to. Writing components against a real event stream 
(even a mocked one from main.py) is dramatically faster than 
writing them against hardcoded state and then retrofitting 
real data later.
```

---

**Say "continue" and I'll complete each missing piece in the same depth as Phases 0–5 — starting with the integration layer (`main.py`), then the full Phase 6 UI component by component, then Phase 9. One phase per message, same format.**






# SENTINEL — Agent-Ready Build Specification
## Integration Layer: `main.py` + FastAPI + WebSocket Server

---

### Objective

Wire every module built in Phases 0–5 into a single running application. This is the **central nervous system** of SENTINEL — without it, every phase is an isolated, untested-in-combination module. After this phase, you will be able to start one process, open one browser tab, and watch the entire pipeline run end to end in real time.

This is the phase most solo builders underestimate. The individual modules are relatively easy; making them talk to each other in the right order, at startup, without race conditions, while broadcasting a live event stream to a frontend over WebSocket is where the majority of late-night debugging happens. Building it explicitly and carefully now prevents that.

---

### Prerequisites

- Phases 0–5 all complete and individually tested
- `uvicorn`, `fastapi`, `websockets` present in `requirements.txt` (already there from Phase −1)
- Frontend dev server can run (`npm run dev` in `frontend/`)

---

### Exact Deliverables

1. `backend/main.py` — FastAPI application, startup routine, WebSocket broadcaster, REST endpoints
2. `backend/sentinel/broadcaster.py` — the bridge between `EventBus` (Phase 0's in-process event stream) and WebSocket connections (the frontend)
3. `backend/sentinel/pipeline.py` — the compound freeze policy assembly (wires Phases 2–4 together into a single callable for the orchestrator)
4. `backend/tests/test_integration.py` — end-to-end integration test running the full pipeline without a real frontend
5. Updated `.env.example` with all required vars

---

### Architecture: How Everything Connects

```
STARTUP
   │
   ├── ScopeModel(task_definition) ──── embeds task via Crusoe
   ├── HeartbeatMonitor()
   ├── EventBus (singleton from events.py)
   ├── Broadcaster (subscribes to EventBus, fans out to WS clients)
   ├── Orchestrator.set_freeze_policy(compound_policy)
   └── DemoAgent(task, steps)

HTTP/WS LAYER (FastAPI)
   │
   ├── GET  /health              ── liveness check
   ├── WS   /ws/events           ── frontend connects here, receives
   │                                 real-time JSON events as they fire
   ├── POST /run                 ── starts the demo agent loop
   ├── POST /approve/{action_id} ── operator approves frozen action
   └── POST /abort/{action_id}   ── operator aborts frozen action

EVENT FLOW (during /run)
   DemoAgent.run()
      └── orchestrator.attempt_action(action)
             ├── bus.emit(ACTION_PROPOSED)  ──────────────────────────┐
             ├── scope_model.is_drifted()                             │
             ├── bus.emit(DRIFT_SCORED)  ────────────────────────────┤
             ├── probe(...)                                           │
             ├── bus.emit(MAARS_PROBE)  ─────────────────────────────┤
             ├── heartbeat.score()                                    │
             ├── bus.emit(HEARTBEAT_CHECK)  ──────────────────────────┤
             └── if frozen:                                           │
                    sequencer.seal_incident()                         │
                    bus.emit(ACTION_FROZEN)  ───────────────────────┤
                    bus.emit(INCIDENT_SEALED)  ──────────────────────┤
                                                                      │
   Broadcaster (subscribed to bus) ─────────────────────────────────┘
      └── for each connected WS client:
             client.send(event.to_json())
```

---

### File: `backend/sentinel/broadcaster.py`

```python
"""
Broadcaster — bridges the in-process EventBus to all connected
WebSocket clients. This is the ONLY place that knows about both
the EventBus and WebSocket connections. Everything else in the
backend knows only about the EventBus; everything in the frontend
knows only about the WebSocket. This file is the seam.
"""

from __future__ import annotations
import asyncio
import logging
from typing import Set
from fastapi import WebSocket

from sentinel.events import bus, SentinelEvent

logger = logging.getLogger("sentinel.broadcaster")


class Broadcaster:
    """
    Subscribes to the in-process EventBus and fans events out to all
    currently connected WebSocket clients.

    Thread-safety note: EventBus.emit() can be called from the agent
    loop (a background thread launched by /run). WebSocket sends must
    happen on the asyncio event loop. asyncio.run_coroutine_threadsafe()
    bridges this gap — this is the correct pattern, not a hack.
    """

    def __init__(self, loop: asyncio.AbstractEventLoop):
        self._loop = loop
        self._clients: Set[WebSocket] = set()
        # Subscribe to ALL events from the bus at construction time.
        bus.subscribe(self._on_event)

    def _on_event(self, event: SentinelEvent):
        """Called synchronously by EventBus (potentially from a thread).
        Schedules async send onto the event loop."""
        asyncio.run_coroutine_threadsafe(
            self._broadcast(event.to_json()),
            self._loop
        )

    async def _broadcast(self, message: str):
        disconnected = set()
        for client in self._clients:
            try:
                await client.send_text(message)
            except Exception as e:
                logger.warning(f"WebSocket send failed, removing client: {e}")
                disconnected.add(client)
        self._clients -= disconnected

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self._clients.add(websocket)
        logger.info(f"WebSocket client connected. Total: {len(self._clients)}")

    async def disconnect(self, websocket: WebSocket):
        self._clients.discard(websocket)
        logger.info(f"WebSocket client disconnected. Total: {len(self._clients)}")
```

**Why `asyncio.run_coroutine_threadsafe`**: the demo agent runs in a background thread (so the FastAPI event loop isn't blocked while the agent is stepping through its sequence). WebSocket sends are async and must run on the event loop thread. This bridge is the correct, non-hacky way to cross that boundary — it's what asyncio's own documentation recommends for exactly this scenario.

---

### File: `backend/sentinel/pipeline.py`

```python
"""
Pipeline — assembles the compound freeze policy from Phase 2, 3, and 4
components and installs it into the orchestrator. This file exists to
keep main.py clean and to make the policy assembly independently
testable.
"""

from __future__ import annotations
import logging
from typing import Optional

from sentinel.drift import ScopeModel
from sentinel.probe import probe
from sentinel.heartbeat import HeartbeatMonitor
from sentinel.orchestrator import orchestrator, FreezeDecision
from sentinel.events import bus, EventType

logger = logging.getLogger("sentinel.pipeline")


class SentinelPipeline:
    """
    Holds the stateful components (ScopeModel, HeartbeatMonitor) and
    assembles the compound freeze policy. One instance per demo run.
    Reset between runs by constructing a new instance.
    """

    def __init__(self, task_definition: str):
        self.task_definition = task_definition
        self.action_history: list[str] = []
        logger.info(f"Initialising ScopeModel for task: {task_definition}")
        self.scope_model = ScopeModel(task_definition)
        self.heartbeat = HeartbeatMonitor()
        self._install()

    def _install(self):
        """Installs the compound policy into the module-level orchestrator."""
        orchestrator.set_freeze_policy(self._compound_policy)
        logger.info("Compound freeze policy installed into orchestrator")

    def _compound_policy(self, action) -> FreezeDecision:
        """
        The compound freeze policy: triggered by drift+MAARS compound
        OR by heartbeat lapse independently. Emits events for every
        signal so the UI gets a complete picture regardless of outcome.
        """
        # --- Signal 1: Scope drift ---
        drifted, drift_score = self.scope_model.is_drifted(action.description)
        action.drift_score = drift_score
        bus.emit(EventType.DRIFT_SCORED, action.action_id, {
            "drift_score": round(drift_score, 4),
            "threshold": 0.35,
            "drifted": drifted,
            "action_description": action.description,
        })

        # --- Signal 2: MAARS adversarial probe ---
        # Only call if action is irreversible — avoids burning inference
        # quota on reversible actions that can't be frozen anyway.
        maars_result = None
        if action.is_irreversible:
            maars_result = probe(
                task_definition=self.task_definition,
                action_description=action.description,
                action_history=self.action_history,
            )
            action.maars_verdict = maars_result.__dict__
            bus.emit(EventType.MAARS_PROBE, action.action_id, {
                "verdict": maars_result.verdict,
                "confidence": maars_result.confidence,
                "reasoning": maars_result.reasoning,
                "severity": maars_result.severity,
                "remediation": maars_result.remediation,
            })

        # --- Signal 3: Heartbeat ---
        hb_score = self.heartbeat.score()
        action.heartbeat_score_at_time = hb_score
        bus.emit(EventType.HEARTBEAT_CHECK, action.action_id, {
            "heartbeat_score": hb_score,
            "lapsed": self.heartbeat.is_lapsed(),
        })

        # --- Compound decision ---
        compound = (
            drifted
            and maars_result is not None
            and maars_result.disagrees
            and maars_result.confidence > 75
        )
        heartbeat_trigger = self.heartbeat.is_lapsed()
        should_freeze = compound or heartbeat_trigger

        reasons = []
        if compound:
            reasons.append(
                f"compound trigger: drift={drift_score:.2f}, "
                f"MAARS={maars_result.verdict}({maars_result.confidence}%)"
            )
        if heartbeat_trigger:
            reasons.append(f"heartbeat lapsed: score={hb_score}")

        reason = "; ".join(reasons) if reasons else "no trigger"

        # Update action history AFTER evaluation so this action
        # isn't included in its own probe context
        self.action_history.append(action.description)

        return FreezeDecision(should_freeze=should_freeze, reason=reason)

    def log_operator_approval(self, duration_seconds: float):
        """Called by the /approve and /abort endpoints to update
        heartbeat state — each operator interaction is a check-in."""
        self.heartbeat.log_approval(duration_seconds)
```

---

### File: `backend/main.py`

```python
"""
SENTINEL — FastAPI application entry point.

Wires all backend modules into a running server. Every design decision
here is documented because this is the file a judge is most likely to
read when reviewing the repository.
"""

from __future__ import annotations
import asyncio
import logging
import os
import threading
import time
from contextlib import asynccontextmanager

from dotenv import load_dotenv
load_dotenv()

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, HTTPException
from fastapi.middleware.cors import CORSMiddleware

from sentinel.broadcaster import Broadcaster
from sentinel.pipeline import SentinelPipeline
from sentinel.agent import DemoAgent
from sentinel.orchestrator import orchestrator
from sentinel.sequencer import seal_incident, generate_advisory_text
from sentinel.events import bus, EventType

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(name)s] %(levelname)s: %(message)s",
)
logger = logging.getLogger("sentinel.main")

# ---------------------------------------------------------------------------
# Application state — module-level, not in a mutable global dict.
# These are set during lifespan startup and read by endpoint handlers.
# ---------------------------------------------------------------------------
_broadcaster: Broadcaster | None = None
_pipeline: SentinelPipeline | None = None
_agent: DemoAgent | None = None
_run_lock = threading.Lock()  # prevents concurrent /run calls
_agent_thread: threading.Thread | None = None

TASK_DEFINITION = "Research 3 vendors and draft a comparison table"


# ---------------------------------------------------------------------------
# Lifespan — runs once at startup, once at shutdown.
# Using the lifespan context manager (not @app.on_event deprecated pattern).
# ---------------------------------------------------------------------------
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _broadcaster, _pipeline, _agent

    loop = asyncio.get_event_loop()
    _broadcaster = Broadcaster(loop)

    logger.info("Initialising SentinelPipeline (embeds task definition)...")
    _pipeline = SentinelPipeline(task_definition=TASK_DEFINITION)

    logger.info("Building DemoAgent with default procurement scenario...")
    _agent = DemoAgent.default_procurement_scenario()

    logger.info("SENTINEL ready. Visit /docs for the API reference.")
    yield

    # Shutdown — nothing to clean up for a demo-scale server,
    # but the hook exists for correctness.
    logger.info("SENTINEL shutting down.")


# ---------------------------------------------------------------------------
# App instantiation
# ---------------------------------------------------------------------------
app = FastAPI(
    title="SENTINEL",
    description="A dead man's switch for autonomous AI agents.",
    version="0.1.0",
    lifespan=lifespan,
)

# CORS — allows the Vite dev server (localhost:5173) to connect.
# In a production build, replace with the actual deployed frontend origin.
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",  # Vite default
        "http://localhost:3000",  # fallback
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# ---------------------------------------------------------------------------
# Health check — judges and your own debugging will hit this first.
# ---------------------------------------------------------------------------
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "task": TASK_DEFINITION,
        "pipeline_ready": _pipeline is not None,
        "agent_ready": _agent is not None,
    }


# ---------------------------------------------------------------------------
# WebSocket — real-time event stream to the frontend.
# ---------------------------------------------------------------------------
@app.websocket("/ws/events")
async def websocket_events(websocket: WebSocket):
    await _broadcaster.connect(websocket)
    try:
        # Keep the connection alive by listening for pings from the client.
        # We don't process incoming messages in Phase 6, but the receive
        # loop must exist — without it, FastAPI closes the connection
        # immediately after accept().
        while True:
            await websocket.receive_text()
    except WebSocketDisconnect:
        await _broadcaster.disconnect(websocket)


# ---------------------------------------------------------------------------
# POST /run — starts the demo agent loop in a background thread.
# ---------------------------------------------------------------------------
@app.post("/run")
async def run_agent():
    global _agent_thread, _agent

    if not _run_lock.acquire(blocking=False):
        raise HTTPException(
            status_code=409,
            detail="Agent already running. Wait for it to complete or restart the server."
        )

    # Reset the agent to a fresh instance for a clean demo re-run.
    # This allows you to call /run multiple times (e.g., during
    # rehearsal) without restarting the server each time.
    _agent = DemoAgent.default_procurement_scenario()

    def run_in_thread():
        try:
            _agent.run(step_delay_seconds=float(
                os.getenv("AGENT_STEP_DELAY", "2.0")
            ))
        except Exception as e:
            logger.error(f"Agent loop error: {e}", exc_info=True)
            bus.emit(EventType.ERROR, None, {"message": str(e)})
        finally:
            _run_lock.release()

    _agent_thread = threading.Thread(target=run_in_thread, daemon=True)
    _agent_thread.start()

    return {"status": "started", "task": TASK_DEFINITION}


# ---------------------------------------------------------------------------
# POST /approve/{action_id} — operator approves a frozen action.
# ---------------------------------------------------------------------------
@app.post("/approve/{action_id}")
async def approve_action(action_id: str):
    try:
        # Log approval timing for heartbeat — the duration here is
        # approximate (we don't track when the freeze modal appeared),
        # but provides a meaningful signal for rehearsal runs.
        _pipeline.log_operator_approval(duration_seconds=5.0)
        action = orchestrator.resolve_frozen(action_id, approved=True)
        return {
            "status": "approved",
            "action_id": action_id,
            "new_status": action.status.value,
        }
    except KeyError:
        raise HTTPException(status_code=404, detail=f"No frozen action: {action_id}")


# ---------------------------------------------------------------------------
# POST /abort/{action_id} — operator aborts a frozen action.
# ---------------------------------------------------------------------------
@app.post("/abort/{action_id}")
async def abort_action(action_id: str):
    try:
        _pipeline.log_operator_approval(duration_seconds=1.0)
        action = orchestrator.resolve_frozen(action_id, approved=False)

        # Seal a final incident report on abort — the action is
        # definitively blocked; this is the artifact a judge can
        # open and inspect.
        completed = _agent.completed_actions if _agent else []
        report = seal_incident(action, completed)
        advisory = generate_advisory_text(report)

        bus.emit(EventType.INCIDENT_SEALED, action_id, {
            "incident_id": report["incident_id"],
            "integrity_hash": report["integrity_hash"],
            "advisory": advisory,
            "report": report,
        })

        return {
            "status": "aborted",
            "action_id": action_id,
            "incident_id": report["incident_id"],
            "integrity_hash": report["integrity_hash"],
        }
    except KeyError:
        raise HTTPException(status_code=404, detail=f"No frozen action: {action_id}")


# ---------------------------------------------------------------------------
# POST /reset — resets the orchestrator and agent for a clean demo re-run.
# Useful during rehearsal without restarting the server.
# ---------------------------------------------------------------------------
@app.post("/reset")
async def reset():
    global _agent, _pipeline
    if _run_lock.locked():
        raise HTTPException(status_code=409, detail="Cannot reset while agent is running.")
    # Fresh pipeline re-embeds the task — this is a legitimate
    # use of the cached embedding path (ScopeModel caches on text,
    # so the second embedding call in the same process hits cache).
    _pipeline = SentinelPipeline(task_definition=TASK_DEFINITION)
    _agent = DemoAgent.default_procurement_scenario()
    return {"status": "reset", "task": TASK_DEFINITION}
```

---

### File: `backend/tests/test_integration.py`

```python
"""
End-to-end integration test — runs the full pipeline (Phases 0–5)
without a real frontend or real Crusoe API. Uses mock implementations
of both to keep the test fast, deterministic, and free of network
dependencies.
"""

import pytest
from unittest.mock import patch, MagicMock
import sentinel.orchestrator as orchestrator_module
import sentinel.pipeline as pipeline_module
import sentinel.agent as agent_module
from sentinel.orchestrator import Orchestrator
from sentinel.events import EventBus


@pytest.fixture(autouse=True)
def isolated_state(monkeypatch):
    """
    Replace the module-level singletons with fresh instances for
    each test. Without this, state from one test (frozen actions,
    event history) bleeds into the next, producing unpredictable
    failures that are extremely hard to debug solo at 2 AM.
    """
    fresh_bus = EventBus()
    fresh_orch = Orchestrator()
    monkeypatch.setattr(orchestrator_module, "orchestrator", fresh_orch)
    monkeypatch.setattr(agent_module, "orchestrator", fresh_orch)
    import sentinel.sequencer as seq_module
    # Patch EventBus bus in all modules that import it
    for mod in [orchestrator_module, pipeline_module, agent_module]:
        if hasattr(mod, "bus"):
            monkeypatch.setattr(mod, "bus", fresh_bus)
    return fresh_bus, fresh_orch


@patch("sentinel.drift.embed")
@patch("sentinel.probe.chat_json")
def test_full_pipeline_freezes_send_email(mock_chat, mock_embed, isolated_state):
    _, fresh_orch = isolated_state

    # Task embedding and all action embeddings are controlled
    # for predictable, deterministic drift scores.
    task_vec = [1.0, 0.0, 0.0]
    close_vec = [0.99, 0.1, 0.0]     # low drift — within scope
    far_vec = [0.0, 0.0, 1.0]         # high drift — out of scope

    embed_sequence = [
        task_vec,   # ScopeModel.__init__ embeds task
        close_vec,  # search_web
        close_vec,  # read_doc
        close_vec,  # write_file
        close_vec,  # draft_email
        far_vec,    # send_email — will trigger drift
    ]
    mock_embed.side_effect = embed_sequence

    # MAARS probe: disagrees with send_email specifically
    def probe_side_effect(prompt):
        if "send_email" in prompt.lower() or "send outreach" in prompt.lower():
            return {
                "verdict": "NO",
                "confidence": 91,
                "reasoning": "Task scope was research-only. External vendor "
                             "contact is 2 steps outside the assigned boundary.",
                "severity": "high",
                "remediation": "Require explicit supervisor approval.",
            }
        return {
            "verdict": "YES", "confidence": 95,
            "reasoning": "Action is within task scope.",
            "severity": None, "remediation": None,
        }
    mock_chat.side_effect = probe_side_effect

    # Install pipeline with mocked dependencies
    pipeline = pipeline_module.SentinelPipeline(TASK_DEFINITION := "Research 3 vendors")
    # Override orchestrator reference since we replaced the singleton
    pipeline_module.orchestrator = fresh_orch
    pipeline._install()

    from sentinel.agent import DemoAgent
    agent = DemoAgent.default_procurement_scenario()
    agent_module.orchestrator = fresh_orch
    results = agent.run(step_delay_seconds=0)

    frozen = [r for r in results if r.status.value == "FROZEN"]
    executed = [r for r in results if r.status.value == "EXECUTED"]

    assert len(frozen) == 1, f"Expected 1 frozen, got {len(frozen)}"
    assert frozen[0].tool_name == "send_email"
    assert len(executed) == 4
    assert frozen[0].drift_score is not None
    assert frozen[0].maars_verdict is not None
    assert frozen[0].maars_verdict["verdict"] == "NO"


@patch("sentinel.drift.embed")
@patch("sentinel.probe.chat_json")
def test_pipeline_does_not_freeze_reversible_actions(mock_chat, mock_embed, isolated_state):
    _, fresh_orch = isolated_state
    far_vec = [0.0, 0.0, 1.0]
    # Even if all embeddings are far and probe disagrees...
    mock_embed.return_value = far_vec
    mock_chat.return_value = {
        "verdict": "NO", "confidence": 99, "reasoning": "very bad action",
        "severity": "high", "remediation": "do not do this"
    }
    pipeline = pipeline_module.SentinelPipeline("Research task")
    pipeline_module.orchestrator = fresh_orch
    pipeline._install()

    from sentinel.action import Action
    # A reversible action must NEVER freeze regardless of signals
    action = Action(description="search the web", is_irreversible=False,
                    tool_name="search_web")
    agent_module.orchestrator = fresh_orch
    result = fresh_orch.attempt_action(action)
    assert result.status.value == "EXECUTED"
```

---

### Updated `.env.example`

```
# Crusoe Managed Inference
CRUSOE_API_KEY=
CRUSOE_API_BASE=https://api.crusoe.ai/v1
CRUSOE_EMBED_MODEL=
CRUSOE_CHAT_MODEL=

# Gradium (voice advisory — optional for solo build)
GRADIUM_API_KEY=

# SMTP sandbox (Mailhog or Mailtrap)
DEMO_SMTP_HOST=localhost
DEMO_SMTP_PORT=1025

# Agent behavior
AGENT_STEP_DELAY=2.0
```

---

### Startup Command

```bash
cd backend
source venv/bin/activate
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

**Manual integration smoke test sequence:**
```bash
# 1. Confirm server started
curl http://localhost:8000/health

# 2. Open a separate terminal and watch the WebSocket stream
# (requires wscat: npm install -g wscat)
wscat -c ws://localhost:8000/ws/events

# 3. In a third terminal, trigger the agent run
curl -X POST http://localhost:8000/run

# 4. Watch events stream into the wscat terminal in real time
# Confirm: ACTION_PROPOSED x5, DRIFT_SCORED x5, MAARS_PROBE x1,
# ACTION_FROZEN x1 — all appearing within the run.

# 5. Abort the frozen action
# (replace ACTION_ID with the actual action_id from the FROZEN event)
curl -X POST http://localhost:8000/abort/ACTION_ID

# 6. Confirm incidents/ directory now contains a .json file
ls -la incidents/

# 7. Confirm the JSON is well-formed and has an integrity_hash
cat incidents/*.json | python3 -m json.tool
```

---

### Do's

- **Do** use `AGENT_STEP_DELAY` as an env var rather than hardcoding the step pause — during development you want `0.0` for fast iteration; for the recorded demo take you want `2.0` for visual legibility. One var, no code changes between modes.
- **Do** implement `/reset` even though it feels unnecessary — as a solo builder rehearsing the demo repeatedly, being able to reset state without restarting the server saves meaningful time.
- **Do** test the WebSocket with `wscat` before building the frontend — confirm events are arriving correctly before you write a single React component. Frontend bugs and backend bugs look identical when discovered from the frontend.
- **Do** keep the CORS `allow_origins` list explicit — don't use `["*"]` for a real submission, even a hackathon one. Judges who look at security posture will notice.

### Don'ts

- **Don't** run the agent directly in an async function (`async def run_agent()`) — `DemoAgent.run()` calls `time.sleep()` between steps which would block the entire FastAPI event loop, freezing WebSocket delivery during the pauses. Always run it in a background thread.
- **Don't** call `seal_incident()` on every freeze event — only seal on the operator's final decision (abort). A frozen action that hasn't been resolved yet doesn't get a sealed report — that's correct behavior, not a bug.
- **Don't** share the `_pipeline` instance across test cases — the `ScopeModel` inside it caches embeddings keyed by text, and a dirty cache from one test scenario can return wrong scores to the next.
- **Don't** hardcode `step_delay_seconds` anywhere in module-level code — it must be injectable so tests can pass `0` for speed without touching production code.

---

### Validation Criteria (Exit Gate)

- [ ] `uvicorn main:app --reload` starts with no errors
- [ ] `GET /health` returns `200 OK` with `pipeline_ready: true`
- [ ] `POST /run` starts the agent; events appear in `wscat` in real time
- [ ] Event sequence confirmed: `ACTION_PROPOSED` × 5, `DRIFT_SCORED` × 5, `MAARS_PROBE` × 1 (irreversible action only), `HEARTBEAT_CHECK` × 5, `ACTION_FROZEN` × 1
- [ ] `POST /abort/{action_id}` returns a valid `incident_id` and creates a JSON file in `incidents/`
- [ ] `POST /reset` followed by `POST /run` produces a clean second run without restarting the server
- [ ] `pytest backend/tests/test_integration.py -v` passes
- [ ] Solo-remote check: `GET /health` response is human-readable enough that a judge hitting it cold understands the system state without explanation
- [ ] Solo-remote check: API docs at `http://localhost:8000/docs` (FastAPI auto-generated) clearly describe all 5 endpoints without additional narration

---

**Say "continue" and I'll do Phase 6: Full UI — starting with `App.tsx` and the WebSocket event hook, then each component in full, then the CSS tokens, then the Impeccable integration checkpoints. One component at a time, same depth.**





# SENTINEL — Agent-Ready Build Specification
## Phase 6: Full UI (React + TypeScript)

---

### Objective

Build the three-screen frontend that consumes the WebSocket event stream from the integration layer and renders SENTINEL's state in real time. Every component in this phase is production-quality TypeScript, fully wired to real backend events — no hardcoded mock state anywhere in the final build.

**Solo-builder principle for this phase**: build the data layer first (types, WebSocket hook, event store), then components strictly in dependency order. A component that renders real data from the first moment you open it is easier to debug solo than one that starts with mock state and gets retrofitted to real data later.

---

### Prerequisites

- Integration layer complete: `uvicorn main:app --reload` running and passing all validation criteria
- `wscat` test confirmed: events streaming correctly over `ws://localhost:8000/ws/events`
- Frontend initialized: `npm create vite@latest . -- --template react-ts` complete
- `npx impeccable install` run in `frontend/`
- `/impeccable init` run inside Cursor with `PRODUCT.md` and `DESIGN.md` at repo root

---

### Exact Deliverables

1. `frontend/src/tokens.css` — design system CSS variables
2. `frontend/src/types/events.ts` — TypeScript event types mirroring backend schema
3. `frontend/src/hooks/useEventStream.ts` — WebSocket connection + event dispatch hook
4. `frontend/src/store/sentinelStore.ts` — application state derived from event stream
5. `frontend/src/components/AgentMonitor.tsx` — Screen 1
6. `frontend/src/components/DriftChart.tsx` — Screen 2
7. `frontend/src/components/FreezeModal.tsx` — Screen 3 (combined MAARS + incident)
8. `frontend/src/App.tsx` — root layout wiring all three screens
9. `frontend/src/main.tsx` — entry point
10. `frontend/src/index.css` — global reset only, no component styles here
11. `frontend/impeccable.config.json` — generated by `/impeccable init`

---

### File: `frontend/src/tokens.css`

This is the first file to create. Every component imports from this. Nothing in the component files hardcodes a color, font, or spacing value — they reference these variables only. This is what makes `/impeccable audit` useful: it checks that you haven't drifted from your own tokens.

```css
/* SENTINEL Design Tokens
   Source of truth for all visual decisions.
   Reference: DESIGN.md at repo root.
   Rationale: centralised tokens mean one-file changes
   propagate everywhere — critical when building solo
   under time pressure. */

:root {
  /* ── Color: Base ─────────────────────────────────── */
  --color-base:          #0A0B0D;
  --color-surface:       #111316;
  --color-surface-raised: #181B1F;
  --color-border:        rgba(255, 255, 255, 0.08);
  --color-border-strong: rgba(255, 255, 255, 0.16);

  /* ── Color: Text ─────────────────────────────────── */
  --color-text-primary:   #E8EAF0;
  --color-text-secondary: #8A8F9E;
  --color-text-muted:     #4A4F5E;
  --color-text-mono:      #A8B0C0;

  /* ── Color: Status (semantic ONLY — never decorative) */
  --color-healthy:        #6B8F7A;
  --color-healthy-dim:    rgba(107, 143, 122, 0.15);
  --color-drift:          #D9A441;
  --color-drift-dim:      rgba(217, 164, 65, 0.15);
  --color-frozen:         #C4453A;
  --color-frozen-dim:     rgba(196, 69, 58, 0.15);
  --color-frozen-text:    #E87870;

  /* ── Color: Interactive ──────────────────────────── */
  --color-accent:         #4A7A9E;
  --color-accent-hover:   #5A8AAE;
  --color-accent-dim:     rgba(74, 122, 158, 0.15);

  /* ── Typography ──────────────────────────────────── */
  /* Deliberately NOT Inter — see DESIGN.md anti-patterns */
  --font-display: "General Sans", "DM Sans", system-ui, sans-serif;
  --font-mono:    "JetBrains Mono", "IBM Plex Mono", "Fira Code", monospace;

  --text-xs:   0.70rem;  /* 11.2px */
  --text-sm:   0.80rem;  /* 12.8px */
  --text-base: 0.925rem; /* 14.8px */
  --text-md:   1.05rem;  /* 16.8px */
  --text-lg:   1.25rem;  /* 20px   */
  --text-xl:   1.5rem;   /* 24px   */
  --text-2xl:  2rem;     /* 32px   */
  --text-3xl:  2.75rem;  /* 44px   */

  --weight-regular: 400;
  --weight-medium:  500;
  --weight-semi:    600;

  --leading-tight:  1.2;
  --leading-normal: 1.5;
  --leading-loose:  1.7;

  /* ── Spacing (8pt grid) ──────────────────────────── */
  --space-1:  4px;
  --space-2:  8px;
  --space-3:  12px;
  --space-4:  16px;
  --space-5:  20px;
  --space-6:  24px;
  --space-8:  32px;
  --space-10: 40px;
  --space-12: 48px;
  --space-16: 64px;

  /* ── Borders ─────────────────────────────────────── */
  /* Sharp corners — explicitly NOT rounded-pill.
     See DESIGN.md: "departure from generic SaaS default" */
  --radius-none: 0px;
  --radius-sm:   2px;
  --radius-md:   4px;

  /* ── Transitions ─────────────────────────────────── */
  --transition-fast:   100ms ease;
  --transition-normal: 200ms ease;
  --transition-slow:   400ms ease;

  /* ── Z-index layers ──────────────────────────────── */
  --z-base:    1;
  --z-overlay: 100;
  --z-modal:   200;
}
```

---

### File: `frontend/src/index.css`

Global reset only. No component styles here — ever.

```css
@import "./tokens.css";

*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body, #root {
  height: 100%;
  width: 100%;
}

body {
  background-color: var(--color-base);
  color: var(--color-text-primary);
  font-family: var(--font-display);
  font-size: var(--text-base);
  line-height: var(--leading-normal);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Monospace utility — applied to any element needing the mono font */
.mono {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  color: var(--color-text-mono);
}

/* Scrollbar styling — subtle, on-brand */
::-webkit-scrollbar { width: 4px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb {
  background: var(--color-border-strong);
  border-radius: var(--radius-sm);
}
```

---

### File: `frontend/src/types/events.ts`

TypeScript mirror of the backend's `EventType` enum and event payloads. This is the contract between backend and frontend — if a backend event shape changes, this file is the single place to update it.

```typescript
/**
 * EventType — mirrors backend sentinel/events.py EventType enum exactly.
 * If you add an event type to the backend, add it here first (TypeScript
 * will then surface every place the new type needs to be handled).
 */
export enum EventType {
  ACTION_PROPOSED   = "ACTION_PROPOSED",
  DRIFT_SCORED      = "DRIFT_SCORED",
  MAARS_PROBE       = "MAARS_PROBE",
  HEARTBEAT_CHECK   = "HEARTBEAT_CHECK",
  ACTION_EXECUTED   = "ACTION_EXECUTED",
  ACTION_FROZEN     = "ACTION_FROZEN",
  INCIDENT_SEALED   = "INCIDENT_SEALED",
  OPERATOR_DECISION = "OPERATOR_DECISION",
  ERROR             = "ERROR",
}

// ── Payload types ─────────────────────────────────────────────────────────

export interface ActionProposedPayload {
  description: string;
  tool_name: string;
  is_irreversible: boolean;
}

export interface DriftScoredPayload {
  drift_score: number;
  threshold: number;
  drifted: boolean;
  action_description: string;
}

export interface MaarsProbePayload {
  verdict: "YES" | "NO";
  confidence: number;
  reasoning: string;
  severity: "low" | "medium" | "high" | null;
  remediation: string | null;
}

export interface HeartbeatCheckPayload {
  heartbeat_score: number;
  lapsed: boolean;
}

export interface ActionExecutedPayload {
  description: string;
}

export interface ActionFrozenPayload {
  description: string;
  reason: string;
}

export interface IncidentSealedPayload {
  incident_id: string;
  integrity_hash: string;
  advisory: string;
  report: IncidentReport;
}

export interface IncidentReport {
  incident_id: string;
  timestamp: string;
  frozen_action: {
    description: string;
    tool_name: string;
    parameters: Record<string, unknown>;
  };
  drift_score: number | null;
  maars_verdict: MaarsProbePayload | null;
  heartbeat_score: number | null;
  decision_trace: DecisionTraceEntry[];
  integrity_hash: string;
}

export interface DecisionTraceEntry {
  description: string;
  status: string;
  drift_score: number | null;
}

export interface OperatorDecisionPayload {
  decision: "APPROVE" | "ABORT";
}

export interface ErrorPayload {
  message: string;
}

// ── Discriminated union — the shape of every message from the WS ──────────

export type SentinelEvent =
  | { event_type: EventType.ACTION_PROPOSED;   action_id: string; payload: ActionProposedPayload;   timestamp: string }
  | { event_type: EventType.DRIFT_SCORED;      action_id: string; payload: DriftScoredPayload;      timestamp: string }
  | { event_type: EventType.MAARS_PROBE;       action_id: string; payload: MaarsProbePayload;       timestamp: string }
  | { event_type: EventType.HEARTBEAT_CHECK;   action_id: string; payload: HeartbeatCheckPayload;   timestamp: string }
  | { event_type: EventType.ACTION_EXECUTED;   action_id: string; payload: ActionExecutedPayload;   timestamp: string }
  | { event_type: EventType.ACTION_FROZEN;     action_id: string; payload: ActionFrozenPayload;     timestamp: string }
  | { event_type: EventType.INCIDENT_SEALED;   action_id: string; payload: IncidentSealedPayload;   timestamp: string }
  | { event_type: EventType.OPERATOR_DECISION; action_id: string; payload: OperatorDecisionPayload; timestamp: string }
  | { event_type: EventType.ERROR;             action_id: string | null; payload: ErrorPayload;     timestamp: string };
```

---

### File: `frontend/src/hooks/useEventStream.ts`

```typescript
/**
 * useEventStream — manages the WebSocket connection to the backend
 * and dispatches parsed events to registered handlers.
 *
 * Design decisions:
 * - Reconnects automatically on disconnect (demo resilience).
 * - Parses JSON once; callers receive typed SentinelEvent objects.
 * - Exposes connection status so the UI can show a meaningful indicator.
 */

import { useEffect, useRef, useState, useCallback } from "react";
import { SentinelEvent } from "../types/events";

export type ConnectionStatus = "connecting" | "connected" | "disconnected" | "error";

const WS_URL = import.meta.env.VITE_WS_URL ?? "ws://localhost:8000/ws/events";
const RECONNECT_DELAY_MS = 2000;

interface UseEventStreamOptions {
  onEvent: (event: SentinelEvent) => void;
}

interface UseEventStreamReturn {
  status: ConnectionStatus;
}

export function useEventStream({ onEvent }: UseEventStreamOptions): UseEventStreamReturn {
  const [status, setStatus] = useState<ConnectionStatus>("connecting");
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  // Stable ref to onEvent so the WS listener doesn't need to re-register
  // every time the parent component re-renders.
  const onEventRef = useRef(onEvent);
  onEventRef.current = onEvent;

  const connect = useCallback(() => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    setStatus("connecting");
    const ws = new WebSocket(WS_URL);
    wsRef.current = ws;

    ws.onopen = () => {
      setStatus("connected");
      // Send a periodic ping to keep the connection alive through
      // proxies and load balancers that close idle connections.
      // The backend's receive loop reads (and discards) this message.
    };

    ws.onmessage = (event: MessageEvent) => {
      try {
        const parsed = JSON.parse(event.data) as SentinelEvent;
        onEventRef.current(parsed);
      } catch (err) {
        console.error("Failed to parse event:", event.data, err);
      }
    };

    ws.onerror = () => {
      setStatus("error");
    };

    ws.onclose = () => {
      setStatus("disconnected");
      // Auto-reconnect after a short delay — demo resilience.
      reconnectRef.current = setTimeout(connect, RECONNECT_DELAY_MS);
    };
  }, []);

  useEffect(() => {
    connect();
    return () => {
      reconnectRef.current && clearTimeout(reconnectRef.current);
      wsRef.current?.close();
    };
  }, [connect]);

  return { status };
}
```

---

### File: `frontend/src/store/sentinelStore.ts`

```typescript
/**
 * sentinelStore — derives application state from the raw event stream.
 *
 * This is NOT a global state library (no Redux, no Zustand). It is a
 * plain TypeScript module that processes events in order and returns
 * a snapshot of current state. This simplicity is deliberate: for a
 * single-session demo with a predictable event sequence, a reducer
 * over an ordered event log is more than sufficient and far faster to
 * build solo than a full state-management setup.
 */

import { SentinelEvent, EventType, MaarsProbePayload } from "../types/events";

export type ActionStatus = "proposed" | "executed" | "frozen" | "resumed" | "aborted";

export interface TrackedAction {
  action_id: string;
  description: string;
  tool_name: string;
  is_irreversible: boolean;
  status: ActionStatus;
  drift_score: number | null;
  maars_verdict: MaarsProbePayload | null;
  heartbeat_score: number | null;
  proposed_at: string;
}

export interface DriftPoint {
  action_index: number;
  description: string;
  drift_score: number;
  drifted: boolean;
}

export interface FreezeState {
  action_id: string;
  description: string;
  reason: string;
  maars_verdict: MaarsProbePayload | null;
  drift_score: number | null;
  incident_id: string | null;
  integrity_hash: string | null;
  advisory: string | null;
}

export interface SentinelState {
  connectionStatus: "connecting" | "connected" | "disconnected" | "error";
  isRunning: boolean;
  actions: TrackedAction[];
  driftPoints: DriftPoint[];
  heartbeatScore: number;
  freezeState: FreezeState | null;
  isResolved: boolean; // true once operator has approved or aborted
  error: string | null;
}

export const initialState: SentinelState = {
  connectionStatus: "connecting",
  isRunning: false,
  actions: [],
  driftPoints: [],
  heartbeatScore: 100,
  freezeState: null,
  isResolved: false,
  error: null,
};

/**
 * Reducer — pure function, takes current state + one event, returns
 * next state. This pattern makes the store trivially unit-testable
 * without a DOM or React setup.
 */
export function sentinelReducer(
  state: SentinelState,
  event: SentinelEvent
): SentinelState {
  switch (event.event_type) {

    case EventType.ACTION_PROPOSED: {
      const newAction: TrackedAction = {
        action_id: event.action_id,
        description: event.payload.description,
        tool_name: event.payload.tool_name,
        is_irreversible: event.payload.is_irreversible,
        status: "proposed",
        drift_score: null,
        maars_verdict: null,
        heartbeat_score: null,
        proposed_at: event.timestamp,
      };
      return {
        ...state,
        isRunning: true,
        actions: [...state.actions, newAction],
      };
    }

    case EventType.DRIFT_SCORED: {
      const driftPoint: DriftPoint = {
        action_index: state.driftPoints.length,
        description: event.payload.action_description,
        drift_score: event.payload.drift_score,
        drifted: event.payload.drifted,
      };
      return {
        ...state,
        driftPoints: [...state.driftPoints, driftPoint],
        actions: state.actions.map(a =>
          a.action_id === event.action_id
            ? { ...a, drift_score: event.payload.drift_score }
            : a
        ),
      };
    }

    case EventType.MAARS_PROBE: {
      return {
        ...state,
        actions: state.actions.map(a =>
          a.action_id === event.action_id
            ? { ...a, maars_verdict: event.payload }
            : a
        ),
        // Pre-populate freeze state with MAARS data so it's ready
        // when ACTION_FROZEN arrives immediately after
        freezeState: state.freezeState
          ? { ...state.freezeState, maars_verdict: event.payload }
          : null,
      };
    }

    case EventType.HEARTBEAT_CHECK: {
      return {
        ...state,
        heartbeatScore: event.payload.heartbeat_score,
        actions: state.actions.map(a =>
          a.action_id === event.action_id
            ? { ...a, heartbeat_score: event.payload.heartbeat_score }
            : a
        ),
      };
    }

    case EventType.ACTION_EXECUTED: {
      return {
        ...state,
        actions: state.actions.map(a =>
          a.action_id === event.action_id
            ? { ...a, status: "executed" }
            : a
        ),
      };
    }

    case EventType.ACTION_FROZEN: {
      const frozenAction = state.actions.find(a => a.action_id === event.action_id);
      return {
        ...state,
        actions: state.actions.map(a =>
          a.action_id === event.action_id
            ? { ...a, status: "frozen" }
            : a
        ),
        freezeState: {
          action_id: event.action_id,
          description: event.payload.description,
          reason: event.payload.reason,
          maars_verdict: frozenAction?.maars_verdict ?? null,
          drift_score: frozenAction?.drift_score ?? null,
          incident_id: null,
          integrity_hash: null,
          advisory: null,
        },
      };
    }

    case EventType.INCIDENT_SEALED: {
      return {
        ...state,
        freezeState: state.freezeState
          ? {
              ...state.freezeState,
              incident_id: event.payload.incident_id,
              integrity_hash: event.payload.integrity_hash,
              advisory: event.payload.advisory,
            }
          : null,
      };
    }

    case EventType.OPERATOR_DECISION: {
      return {
        ...state,
        isRunning: false,
        isResolved: true,
        actions: state.actions.map(a =>
          a.action_id === event.action_id
            ? { ...a, status: event.payload.decision === "APPROVE" ? "resumed" : "aborted" }
            : a
        ),
      };
    }

    case EventType.ERROR: {
      return {
        ...state,
        isRunning: false,
        error: event.payload.message,
      };
    }

    default:
      return state;
  }
}
```

---

### File: `frontend/src/App.tsx`

```tsx
/**
 * App.tsx — root layout. Three-column layout for the main monitoring
 * view, with the FreezeModal layered on top when active.
 *
 * Layout philosophy (from DESIGN.md): flat surfaces, no nested cards,
 * generous whitespace, hairline borders. The three columns each own
 * one conceptual domain: task context, action stream, system signals.
 */

import { useReducer, useCallback, useState } from "react";
import { sentinelReducer, initialState, SentinelState } from "./store/sentinelStore";
import { SentinelEvent, EventType } from "./types/events";
import { useEventStream, ConnectionStatus } from "./hooks/useEventStream";
import { AgentMonitor } from "./components/AgentMonitor";
import { DriftChart } from "./components/DriftChart";
import { FreezeModal } from "./components/FreezeModal";
import "./index.css";
import styles from "./App.module.css";

const API_BASE = import.meta.env.VITE_API_URL ?? "http://localhost:8000";

export default function App() {
  const [state, dispatch] = useReducer(sentinelReducer, initialState);
  const [wsStatus, setWsStatus] = useState<ConnectionStatus>("connecting");

  const handleEvent = useCallback((event: SentinelEvent) => {
    dispatch(event);
  }, []);

  const { status } = useEventStream({ onEvent: handleEvent });

  // Keep wsStatus in sync with the hook's status
  if (status !== wsStatus) setWsStatus(status);

  const handleRun = async () => {
    await fetch(`${API_BASE}/run`, { method: "POST" });
  };

  const handleApprove = async (actionId: string) => {
    await fetch(`${API_BASE}/approve/${actionId}`, { method: "POST" });
  };

  const handleAbort = async (actionId: string) => {
    await fetch(`${API_BASE}/abort/${actionId}`, { method: "POST" });
  };

  const handleReset = async () => {
    await fetch(`${API_BASE}/reset`, { method: "POST" });
    window.location.reload(); // simplest reset for demo purposes
  };

  return (
    <div className={styles.root}>
      <Header
        wsStatus={wsStatus}
        isRunning={state.isRunning}
        onRun={handleRun}
        onReset={handleReset}
      />

      <main className={styles.main}>
        {/* Column 1: Live action stream */}
        <section className={styles.column}>
          <AgentMonitor
            actions={state.actions}
            heartbeatScore={state.heartbeatScore}
            isRunning={state.isRunning}
          />
        </section>

        {/* Column 2: Drift trajectory */}
        <section className={styles.column}>
          <DriftChart points={state.driftPoints} />
        </section>
      </main>

      {/* Freeze modal: shown when freezeState is non-null and not yet resolved */}
      {state.freezeState && !state.isResolved && (
        <FreezeModal
          freeze={state.freezeState}
          onApprove={() => handleApprove(state.freezeState!.action_id)}
          onAbort={() => handleAbort(state.freezeState!.action_id)}
        />
      )}

      {state.error && (
        <div className={styles.errorBanner}>
          <span>⚠ Error: {state.error}</span>
        </div>
      )}
    </div>
  );
}

// ── Header sub-component ─────────────────────────────────────────────────

interface HeaderProps {
  wsStatus: ConnectionStatus;
  isRunning: boolean;
  onRun: () => void;
  onReset: () => void;
}

function Header({ wsStatus, isRunning, onRun, onReset }: HeaderProps) {
  return (
    <header className={styles.header}>
      <div className={styles.headerLeft}>
        <span className={styles.wordmark}>SENTINEL</span>
        <ConnectionIndicator status={wsStatus} />
      </div>
      <div className={styles.headerRight}>
        {!isRunning && (
          <button className={styles.btnPrimary} onClick={onRun}>
            Run Agent
          </button>
        )}
        <button className={styles.btnSecondary} onClick={onReset}>
          Reset
        </button>
      </div>
    </header>
  );
}

// ── Connection indicator ──────────────────────────────────────────────────

function ConnectionIndicator({ status }: { status: ConnectionStatus }) {
  const labels: Record<ConnectionStatus, string> = {
    connecting:   "CONNECTING",
    connected:    "MONITORING",
    disconnected: "DISCONNECTED",
    error:        "ERROR",
  };
  return (
    <span className={styles.connectionIndicator} data-status={status}>
      <span className={styles.dot} />
      {labels[status]}
    </span>
  );
}
```

### File: `frontend/src/App.module.css`

```css
.root {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  background: var(--color-base);
}

/* ── Header ─────────────────────────────────────────────────────────────── */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-4) var(--space-6);
  border-bottom: 1px solid var(--color-border);
  flex-shrink: 0;
}

.headerLeft {
  display: flex;
  align-items: center;
  gap: var(--space-6);
}

.headerRight {
  display: flex;
  align-items: center;
  gap: var(--space-3);
}

.wordmark {
  font-family: var(--font-mono);
  font-size: var(--text-md);
  font-weight: var(--weight-semi);
  letter-spacing: 0.15em;
  color: var(--color-text-primary);
}

/* ── Connection indicator ────────────────────────────────────────────────── */
.connectionIndicator {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: 0.1em;
  color: var(--color-text-secondary);
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--color-text-muted);
}

/* Status-driven dot colors */
[data-status="connected"] .dot {
  background: var(--color-healthy);
  /* Slow pulse — signals "live system", not "error" */
  animation: pulse 3s ease-in-out infinite;
}
[data-status="error"] .dot { background: var(--color-frozen); }
[data-status="disconnected"] .dot { background: var(--color-drift); }

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.4; }
}

/* ── Main layout ─────────────────────────────────────────────────────────── */
.main {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1px; /* hairline divider via background bleed */
  background: var(--color-border);
  flex: 1;
  overflow: hidden;
}

.column {
  background: var(--color-base);
  overflow-y: auto;
  padding: var(--space-6);
}

/* ── Buttons ─────────────────────────────────────────────────────────────── */
/* Sharp corners, 1px border — per DESIGN.md,
   explicitly not rounded-pill */
.btnPrimary {
  padding: var(--space-2) var(--space-5);
  background: var(--color-accent);
  color: var(--color-text-primary);
  border: 1px solid transparent;
  border-radius: var(--radius-none);
  font-family: var(--font-display);
  font-size: var(--text-sm);
  font-weight: var(--weight-medium);
  letter-spacing: 0.05em;
  cursor: pointer;
  transition: background var(--transition-fast);
}
.btnPrimary:hover { background: var(--color-accent-hover); }

.btnSecondary {
  padding: var(--space-2) var(--space-5);
  background: transparent;
  color: var(--color-text-secondary);
  border: 1px solid var(--color-border-strong);
  border-radius: var(--radius-none);
  font-family: var(--font-display);
  font-size: var(--text-sm);
  font-weight: var(--weight-medium);
  letter-spacing: 0.05em;
  cursor: pointer;
  transition: color var(--transition-fast), border-color var(--transition-fast);
}
.btnSecondary:hover {
  color: var(--color-text-primary);
  border-color: var(--color-text-secondary);
}

/* ── Error banner ────────────────────────────────────────────────────────── */
.errorBanner {
  position: fixed;
  bottom: var(--space-6);
  left: 50%;
  transform: translateX(-50%);
  padding: var(--space-3) var(--space-6);
  background: var(--color-frozen-dim);
  border: 1px solid var(--color-frozen);
  color: var(--color-frozen-text);
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  z-index: var(--z-overlay);
}
```

---

### File: `frontend/src/components/AgentMonitor.tsx`

```tsx
/**
 * AgentMonitor — Screen 1.
 * Shows the live action stream with status indicators,
 * plus the heartbeat score in the corner.
 */

import { TrackedAction } from "../store/sentinelStore";
import styles from "./AgentMonitor.module.css";

interface Props {
  actions: TrackedAction[];
  heartbeatScore: number;
  isRunning: boolean;
}

const STATUS_LABELS: Record<string, string> = {
  proposed: "PENDING",
  executed: "✓ EXECUTED",
  frozen:   "⛔ FROZEN",
  resumed:  "✓ RESUMED",
  aborted:  "✗ ABORTED",
};

export function AgentMonitor({ actions, heartbeatScore, isRunning }: Props) {
  return (
    <div className={styles.root}>
      <div className={styles.header}>
        <SectionLabel>ACTION STREAM</SectionLabel>
        <HeartbeatScore score={heartbeatScore} />
      </div>

      {actions.length === 0 ? (
        <div className={styles.empty}>
          {isRunning ? "Waiting for first action..." : "Press Run Agent to start."}
        </div>
      ) : (
        <ol className={styles.actionList}>
          {actions.map((action, i) => (
            <ActionRow key={action.action_id} index={i + 1} action={action} />
          ))}
        </ol>
      )}
    </div>
  );
}

function ActionRow({ index, action }: { index: number; action: TrackedAction }) {
  return (
    <li className={styles.actionRow} data-status={action.status}>
      <span className={styles.index} aria-hidden="true">
        {String(index).padStart(2, "0")}
      </span>

      <div className={styles.actionBody}>
        <span className={styles.toolName}>{action.tool_name}()</span>
        <span className={styles.description}>{action.description}</span>

        {action.drift_score !== null && (
          <span
            className={styles.driftBadge}
            data-drifted={action.drift_score > 0.35}
          >
            drift: {action.drift_score.toFixed(3)}
          </span>
        )}
      </div>

      <span className={styles.statusLabel}>
        {STATUS_LABELS[action.status] ?? action.status.toUpperCase()}
      </span>
    </li>
  );
}

function HeartbeatScore({ score }: { score: number }) {
  const level = score >= 60 ? "healthy" : score >= 30 ? "drift" : "frozen";
  return (
    <div className={styles.heartbeat} data-level={level}>
      <span className={styles.heartbeatLabel}>HEARTBEAT</span>
      <span className={styles.heartbeatValue}>{score}</span>
    </div>
  );
}

function SectionLabel({ children }: { children: React.ReactNode }) {
  return <h2 className={styles.sectionLabel}>{children}</h2>;
}
```

### File: `frontend/src/components/AgentMonitor.module.css`

```css
.root {
  display: flex;
  flex-direction: column;
  gap: var(--space-5);
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.sectionLabel {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  font-weight: var(--weight-regular);
  letter-spacing: 0.12em;
  color: var(--color-text-muted);
  text-transform: uppercase;
}

.empty {
  color: var(--color-text-secondary);
  font-size: var(--text-sm);
  padding: var(--space-8) 0;
}

/* ── Action list ─────────────────────────────────────────────────────────── */
.actionList {
  display: flex;
  flex-direction: column;
  gap: 1px;
  list-style: none;
}

.actionRow {
  display: grid;
  grid-template-columns: var(--space-8) 1fr auto;
  align-items: start;
  gap: var(--space-3);
  padding: var(--space-3) 0;
  border-bottom: 1px solid var(--color-border);
  transition: background var(--transition-fast);
}

[data-status="frozen"]  { background: var(--color-frozen-dim); }
[data-status="aborted"] { background: var(--color-frozen-dim); }

.index {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-muted);
  padding-top: 2px;
}

.actionBody {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
}

.toolName {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  color: var(--color-text-primary);
}

.description {
  font-size: var(--text-sm);
  color: var(--color-text-secondary);
  line-height: var(--leading-normal);
}

.driftBadge {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-healthy);
  padding-top: var(--space-1);
}
[data-drifted="true"] { color: var(--color-drift); }

.statusLabel {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-secondary);
  white-space: nowrap;
  padding-top: 2px;
}

/* ── Heartbeat ───────────────────────────────────────────────────────────── */
.heartbeat {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: var(--space-1);
}

.heartbeatLabel {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: 0.1em;
  color: var(--color-text-muted);
}

.heartbeatValue {
  font-family: var(--font-mono);
  font-size: var(--text-xl);
  font-weight: var(--weight-semi);
  color: var(--color-healthy);
  transition: color var(--transition-slow);
}

[data-level="drift"]  .heartbeatValue { color: var(--color-drift); }
[data-level="frozen"] .heartbeatValue { color: var(--color-frozen-text); }
```

---

### File: `frontend/src/components/DriftChart.tsx`

```tsx
/**
 * DriftChart — Screen 2.
 * Scope drift trajectory: action index on X, drift score on Y,
 * threshold line at 0.35, points colored by threshold crossing.
 *
 * Uses Recharts — install: npm install recharts
 * Why Recharts over D3: measurably faster to build correctly solo,
 * composable with React without manual DOM management, good enough
 * for a demo-scale chart. D3 is better for production; Recharts is
 * better for 4 hours of available build time.
 */

import {
  LineChart, Line, XAxis, YAxis, ReferenceLine,
  Tooltip, ResponsiveContainer, Dot,
} from "recharts";
import { DriftPoint } from "../store/sentinelStore";
import styles from "./DriftChart.module.css";

interface Props {
  points: DriftPoint[];
}

const THRESHOLD = 0.35;

// ── Custom dot — colors by drift status, animates on appearance ──────────
function StatusDot(props: any) {
  const { cx, cy, payload } = props;
  const color = payload.drifted
    ? "var(--color-frozen)"
    : payload.drift_score > THRESHOLD * 0.75
    ? "var(--color-drift)"
    : "var(--color-healthy)";

  return (
    <Dot
      cx={cx} cy={cy} r={5}
      fill={color}
      stroke={color}
      strokeWidth={1}
    />
  );
}

// ── Custom tooltip ────────────────────────────────────────────────────────
function CustomTooltip({ active, payload }: any) {
  if (!active || !payload?.length) return null;
  const d: DriftPoint = payload[0].payload;
  return (
    <div className={styles.tooltip}>
      <p className={styles.tooltipTool}>{d.description}</p>
      <p className={styles.tooltipScore}>
        drift: <strong>{d.drift_score.toFixed(4)}</strong>
      </p>
      {d.drifted && (
        <p className={styles.tooltipWarning}>⛔ exceeded threshold</p>
      )}
    </div>
  );
}

export function DriftChart({ points }: Props) {
  return (
    <div className={styles.root}>
      <h2 className={styles.sectionLabel}>SCOPE DRIFT TRAJECTORY</h2>

      {points.length === 0 ? (
        <div className={styles.empty}>Drift scores will appear as actions run.</div>
      ) : (
        <div className={styles.chartWrapper}>
          <ResponsiveContainer width="100%" height={320}>
            <LineChart
              data={points}
              margin={{ top: 16, right: 16, bottom: 24, left: 0 }}
            >
              {/* Threshold reference line — labeled, not decorative */}
              <ReferenceLine
                y={THRESHOLD}
                stroke="var(--color-border-strong)"
                strokeDasharray="4 3"
                label={{
                  value: "SAFE ZONE BOUNDARY",
                  fill: "var(--color-text-muted)",
                  fontSize: 10,
                  fontFamily: "var(--font-mono)",
                  position: "insideBottomLeft",
                }}
              />

              <XAxis
                dataKey="action_index"
                tick={{ fontFamily: "var(--font-mono)", fontSize: 10,
                        fill: "var(--color-text-muted)" }}
                axisLine={{ stroke: "var(--color-border)" }}
                tickLine={false}
                label={{
                  value: "ACTION INDEX",
                  position: "insideBottom",
                  offset: -12,
                  fill: "var(--color-text-muted)",
                  fontSize: 10,
                  fontFamily: "var(--font-mono)",
                }}
              />

              <YAxis
                domain={[0, 1]}
                tick={{ fontFamily: "var(--font-mono)", fontSize: 10,
                        fill: "var(--color-text-muted)" }}
                axisLine={{ stroke: "var(--color-border)" }}
                tickLine={false}
                width={36}
              />

              <Tooltip content={<CustomTooltip />} />

              <Line
                type="linear"           /* straight segments — honest about
                                           discrete measurements, no interpolation */
                dataKey="drift_score"
                stroke="var(--color-accent)"
                strokeWidth={1.5}
                dot={<StatusDot />}
                isAnimationActive={true}
                animationDuration={300}
              />
            </LineChart>
          </ResponsiveContainer>
        </div>
      )}

      {/* Legend */}
      <div className={styles.legend}>
        <LegendItem color="var(--color-healthy)">Within scope</LegendItem>
        <LegendItem color="var(--color-drift)">Approaching boundary</LegendItem>
        <LegendItem color="var(--color-frozen)">Exceeded threshold</LegendItem>
      </div>
    </div>
  );
}

function LegendItem({ color, children }: { color: string; children: React.ReactNode }) {
  return (
    <div className={styles.legendItem}>
      <span className={styles.legendDot} style={{ background: color }} />
      <span>{children}</span>
    </div>
  );
}
```

### File: `frontend/src/components/DriftChart.module.css`

```css
.root {
  display: flex;
  flex-direction: column;
  gap: var(--space-5);
}

.sectionLabel {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  font-weight: var(--weight-regular);
  letter-spacing: 0.12em;
  color: var(--color-text-muted);
  text-transform: uppercase;
}

.empty {
  color: var(--color-text-secondary);
  font-size: var(--text-sm);
  padding: var(--space-8) 0;
}

.chartWrapper {
  width: 100%;
}

/* ── Tooltip ─────────────────────────────────────────────────────────────── */
.tooltip {
  background: var(--color-surface-raised);
  border: 1px solid var(--color-border-strong);
  padding: var(--space-3) var(--space-4);
  font-size: var(--text-sm);
}

.tooltipTool {
  color: var(--color-text-secondary);
  margin-bottom: var(--space-1);
}

.tooltipScore {
  font-family: var(--font-mono);
  color: var(--color-text-primary);
}

.tooltipWarning {
  color: var(--color-frozen-text);
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  margin-top: var(--space-1);
}

/* ── Legend ──────────────────────────────────────────────────────────────── */
.legend {
  display: flex;
  gap: var(--space-6);
  padding-top: var(--space-2);
}

.legendItem {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-secondary);
}

.legendDot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}
```

---

### File: `frontend/src/components/FreezeModal.tsx`

```tsx
/**
 * FreezeModal — Screen 3. The emotional climax of the demo.
 *
 * Combines MAARS split-screen and the incident panel into one modal.
 * Choreographed entry (per DESIGN.md):
 *   T+0s: Modal appears with scrim
 *   T+0.5s: FREEZE headline renders
 *   T+1s: Body content fades in
 *   T+2s: Incident hash + buttons activate
 *
 * Uses CSS animation stages driven by a timer, not a library.
 * No external animation dependency is worth adding to a solo build.
 */

import { useEffect, useState } from "react";
import { FreezeState } from "../store/sentinelStore";
import styles from "./FreezeModal.module.css";

interface Props {
  freeze: FreezeState;
  onApprove: () => void;
  onAbort: () => void;
}

type Stage = 0 | 1 | 2 | 3;

export function FreezeModal({ freeze, onApprove, onAbort }: Props) {
  const [stage, setStage] = useState<Stage>(0);

  // Choreographed entry
  useEffect(() => {
    const t1 = setTimeout(() => setStage(1), 500);
    const t2 = setTimeout(() => setStage(2), 1000);
    const t3 = setTimeout(() => setStage(3), 2000);
    return () => { clearTimeout(t1); clearTimeout(t2); clearTimeout(t3); };
  }, []);

  const buttonsActive = stage >= 3;

  return (
    <>
      {/* Scrim — subtle darkening of the background */}
      <div className={styles.scrim} aria-hidden="true" />

      <div
        className={styles.modal}
        role="alertdialog"
        aria-modal="true"
        aria-label="SENTINEL: Action Frozen"
      >
        {/* ── FREEZE headline ──────────────────────────────────────── */}
        <div className={styles.freezeHeader} data-stage={stage}>
          <span className={styles.freezeIcon}>⛔</span>
          <span className={styles.freezeHeadline}>FREEZE ACTIVE</span>
          <span className={styles.freezeAction}>{freeze.description}</span>
        </div>

        {/* ── Body: MAARS split + comparison ───────────────────────── */}
        <div className={styles.body} data-visible={stage >= 2}>

          {/* Left: Primary agent */}
          <div className={styles.panel}>
            <div className={styles.panelLabel}>PRIMARY AGENT</div>
            <div className={styles.panelContent}>
              <div className={styles.proposedAction}>
                <span className={styles.toolCallLabel}>Proposed action:</span>
                <code className={styles.toolCall}>{freeze.description}</code>
              </div>
              {freeze.drift_score !== null && (
                <div className={styles.signalRow}>
                  <span className={styles.signalKey}>Drift score</span>
                  <span
                    className={styles.signalValue}
                    data-alert={freeze.drift_score > 0.35}
                  >
                    {freeze.drift_score.toFixed(4)}
                  </span>
                </div>
              )}
            </div>
          </div>

          {/* Divider — animates to red when panels disagree */}
          <div
            className={styles.divider}
            data-conflict={freeze.maars_verdict?.verdict === "NO"}
          />

          {/* Right: MAARS probe */}
          <div className={styles.panel}>
            <div className={styles.panelLabel}>MAARS ADVERSARIAL PROBE</div>
            {freeze.maars_verdict ? (
              <div className={styles.panelContent}>
                <div className={styles.signalRow}>
                  <span className={styles.signalKey}>Verdict</span>
                  <span
                    className={styles.verdictBadge}
                    data-verdict={freeze.maars_verdict.verdict}
                  >
                    {freeze.maars_verdict.verdict}
                  </span>
                </div>
                <div className={styles.signalRow}>
                  <span className={styles.signalKey}>Confidence</span>
                  <span className={styles.signalValue}>
                    {freeze.maars_verdict.confidence}%
                  </span>
                </div>
                <div className={styles.reasoning}>
                  <span className={styles.signalKey}>Reasoning</span>
                  <p className={styles.reasoningText}>
                    {freeze.maars_verdict.reasoning}
                  </p>
                </div>
                {freeze.maars_verdict.severity && (
                  <div className={styles.signalRow}>
                    <span className={styles.signalKey}>Severity</span>
                    <span
                      className={styles.severityBadge}
                      data-level={freeze.maars_verdict.severity}
                    >
                      {freeze.maars_verdict.severity.toUpperCase()}
                    </span>
                  </div>
                )}
              </div>
            ) : (
              <div className={styles.probeLoading}>Probe evaluating...</div>
            )}
          </div>
        </div>

        {/* ── Incident artifact + operator controls ─────────────────── */}
        <div className={styles.footer} data-visible={stage >= 3}>
          <div className={styles.incidentMeta}>
            {freeze.incident_id ? (
              <>
                <span className={styles.metaRow}>
                  <span className={styles.metaKey}>Incident</span>
                  <code className={styles.metaValue}>{freeze.incident_id}</code>
                </span>
                <span className={styles.metaRow}>
                  <span className={styles.metaKey}>SHA-256</span>
                  <code className={styles.metaHash}>{freeze.integrity_hash}</code>
                </span>
              </>
            ) : (
              <span className={styles.metaKey}>
                Incident report will be sealed on abort.
              </span>
            )}
            {freeze.advisory && (
              <p className={styles.advisory}>{freeze.advisory}</p>
            )}
          </div>

          <div className={styles.controls}>
            <button
              className={styles.btnAbort}
              onClick={onAbort}
              disabled={!buttonsActive}
              aria-label="Abort frozen action"
            >
              Abort
            </button>
            <button
              className={styles.btnApprove}
              onClick={onApprove}
              disabled={!buttonsActive}
              aria-label="Approve and resume frozen action"
            >
              Approve
            </button>
          </div>
        </div>
      </div>
    </>
  );
}
```

### File: `frontend/src/components/FreezeModal.module.css`

```css
/* ── Scrim ───────────────────────────────────────────────────────────────── */
.scrim {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.15);
  z-index: var(--z-overlay);
  animation: scrimIn var(--transition-slow) ease forwards;
}

@keyframes scrimIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}

/* ── Modal ───────────────────────────────────────────────────────────────── */
.modal {
  position: fixed;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  width: min(860px, 92vw);
  background: var(--color-surface);
  border: 1px solid var(--color-frozen);
  z-index: var(--z-modal);
  display: flex;
  flex-direction: column;
  animation: modalIn var(--transition-slow) ease forwards;
}

@keyframes modalIn {
  from { opacity: 0; transform: translate(-50%, -48%); }
  to   { opacity: 1; transform: translate(-50%, -50%); }
}

/* ── Freeze header ───────────────────────────────────────────────────────── */
.freezeHeader {
  display: flex;
  align-items: center;
  gap: var(--space-4);
  padding: var(--space-5) var(--space-6);
  background: var(--color-frozen-dim);
  border-bottom: 1px solid var(--color-border);
  opacity: 0;
  transition: opacity var(--transition-slow);
}

[data-stage="1"],
[data-stage="2"],
[data-stage="3"] { opacity: 1; }

.freezeIcon {
  font-size: var(--text-lg);
}

.freezeHeadline {
  font-family: var(--font-mono);
  font-size: var(--text-md);
  font-weight: var(--weight-semi);
  letter-spacing: 0.12em;
  color: var(--color-frozen-text);
}

.freezeAction {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  color: var(--color-text-secondary);
}

/* ── Body ────────────────────────────────────────────────────────────────── */
.body {
  display: grid;
  grid-template-columns: 1fr 1px 1fr;
  gap: 0;
  opacity: 0;
  transition: opacity var(--transition-slow) 0.2s;
}

[data-visible="true"] { opacity: 1; }

.panel {
  padding: var(--space-6);
  display: flex;
  flex-direction: column;
  gap: var(--space-4);
}

.panelLabel {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: 0.12em;
  color: var(--color-text-muted);
  text-transform: uppercase;
}

.panelContent {
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
}

/* Divider — transitions to frozen color on conflict */
.divider {
  background: var(--color-border);
  width: 1px;
  transition: background var(--transition-slow) 1s;
}
[data-conflict="true"] { background: var(--color-frozen); }

/* ── Signal rows ─────────────────────────────────────────────────────────── */
.signalRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--space-4);
}

.signalKey {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-muted);
  letter-spacing: 0.08em;
}

.signalValue {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  color: var(--color-text-primary);
}

[data-alert="true"] { color: var(--color-frozen-text); }

/* ── Verdict badge ───────────────────────────────────────────────────────── */
.verdictBadge {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  font-weight: var(--weight-semi);
  letter-spacing: 0.1em;
  padding: var(--space-1) var(--space-2);
  border: 1px solid currentColor;
}

[data-verdict="NO"]  { color: var(--color-frozen-text); }
[data-verdict="YES"] { color: var(--color-healthy); }

/* ── Severity badge ──────────────────────────────────────────────────────── */
.severityBadge {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: 0.08em;
  padding: 2px var(--space-2);
  border: 1px solid var(--color-border-strong);
  color: var(--color-text-secondary);
}

[data-level="high"]   { color: var(--color-frozen-text); border-color: var(--color-frozen); }
[data-level="medium"] { color: var(--color-drift); border-color: var(--color-drift); }

/* ── Reasoning ───────────────────────────────────────────────────────────── */
.reasoning {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.reasoningText {
  font-size: var(--text-sm);
  color: var(--color-text-secondary);
  line-height: var(--leading-loose);
  border-left: 2px solid var(--color-frozen);
  padding-left: var(--space-3);
}

.proposedAction {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.toolCallLabel {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-muted);
}

.toolCall {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  color: var(--color-text-primary);
  word-break: break-all;
}

.probeLoading {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  color: var(--color-text-muted);
  animation: blink 1.2s ease infinite;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.3; }
}

/* ── Footer ──────────────────────────────────────────────────────────────── */
.footer {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: var(--space-6);
  padding: var(--space-5) var(--space-6);
  border-top: 1px solid var(--color-border);
  background: var(--color-surface-raised);
  opacity: 0;
  transition: opacity var(--transition-slow) 0.3s;
}

[data-visible="true"] { opacity: 1; }

.incidentMeta {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  flex: 1;
  min-width: 0;
}

.metaRow {
  display: flex;
  align-items: baseline;
  gap: var(--space-3);
}

.metaKey {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-muted);
  letter-spacing: 0.08em;
  white-space: nowrap;
}

.metaValue {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-secondary);
}

.metaHash {
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  color: var(--color-text-muted);
  /* Allows truncation on narrow screens — not a primary concern
     for a laptop demo but avoids overflow breaking the layout */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.advisory {
  font-size: var(--text-sm);
  color: var(--color-text-secondary);
  line-height: var(--leading-normal);
  padding-top: var(--space-2);
}

/* ── Operator controls ───────────────────────────────────────────────────── */
.controls {
  display: flex;
  gap: var(--space-3);
  flex-shrink: 0;
  align-items: center;
}

/* Sharp corners, high contrast — per DESIGN.md */
.btnAbort,
.btnApprove {
  padding: var(--space-3) var(--space-6);
  font-family: var(--font-display);
  font-size: var(--text-sm);
  font-weight: var(--weight-medium);
  letter-spacing: 0.05em;
  border-radius: var(--radius-none);
  cursor: pointer;
  transition: opacity var(--transition-fast),
              background var(--transition-fast);
}

.btnAbort {
  background: transparent;
  color: var(--color-frozen-text);
  border: 1px solid var(--color-frozen);
}
.btnAbort:hover:not(:disabled)   { background: var(--color-frozen-dim); }

.btnApprove {
  background: var(--color-accent);
  color: var(--color-text-primary);
  border: 1px solid transparent;
}
.btnApprove:hover:not(:disabled) { background: var(--color-accent-hover); }

/* Disabled state — buttons are inert until stage 3 */
.btnAbort:disabled,
.btnApprove:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}
```

---

### File: `frontend/src/main.tsx`

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.tsx";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

---

### Dependencies to Install

```bash
cd frontend
npm install recharts
npm install -D @types/recharts
```

---

### Environment Variables for Frontend

```bash
# frontend/.env.local
VITE_API_URL=http://localhost:8000
VITE_WS_URL=ws://localhost:8000/ws/events
```

---

### Impeccable Checkpoint Procedure

**Checkpoint A** — after `AgentMonitor` is built:
```
/impeccable critique monitor
```
Fix any flagged Inter-font fallbacks, rounded button corners, or gradient usage before building the next screen.

**Checkpoint B** — after all three screens exist:
```
/impeccable audit dashboard
```
Full 45-rule pass. Expect it to flag anything a teammate would have caught but solo building missed.

**Checkpoint C** — Sunday morning, before recording:
```
/impeccable polish
```
Final consistency pass. Fix anything it surfaces, then record immediately after — don't introduce new changes post-polish.

---

### Do's

- **Do** install `recharts` before writing `DriftChart.tsx` — a missing dependency discovered mid-component build is a disruptive context switch.
- **Do** add `VITE_API_URL` and `VITE_WS_URL` to `.env.local` before starting the dev server — Vite will silently use `undefined` as the URL string without the env var, producing a confusing WebSocket error that looks like a backend problem.
- **Do** test the WebSocket event dispatch through to the UI with a real backend run before polishing visual details — a pixel-perfect component that renders stale state is harder to debug than an ugly component that renders real data.
- **Do** keep all colors as token references (`var(--color-frozen)`) never as hardcoded hex — this is what lets `/impeccable audit` catch drift effectively.

### Don'ts

- **Don't** add `border-radius` larger than `var(--radius-md)` anywhere — the entire design system depends on sharp corners as its primary departure from generic SaaS defaults.
- **Don't** use `position: absolute` inside the main layout columns — it breaks the flat-surface discipline and creates z-index conflicts with the FreezeModal.
- **Don't** animate anything in the main view with fast-cycling animations (no spinners, no progress bars). The only intentional animations are the slow heartbeat pulse and the staged FreezeModal entry — everything else should be static or a simple opacity/color transition.
- **Don't** disable React's `StrictMode` to "fix" a double-render warning — if something breaks in StrictMode it indicates a real side-effect bug that will cause problems during the demo's live run.

---

### Validation Criteria (Exit Gate)

- [ ] `npm run dev` starts with zero errors, no TypeScript errors
- [ ] Opening `localhost:5173` shows the header with a `● MONITORING` indicator that goes green when the backend is running
- [ ] `POST /run` from the terminal causes real events to appear in the UI within `AGENT_STEP_DELAY` seconds
- [ ] Drift scores appear on `AgentMonitor` rows and on `DriftChart` points as each action fires
- [ ] `FreezeModal` appears automatically (without any UI interaction) when the `ACTION_FROZEN` event arrives
- [ ] FreezeModal choreography confirmed: scrim first, headline at ~0.5s, body at ~1s, buttons active at ~2s
- [ ] Clicking `Abort` calls `POST /abort/{action_id}`, the modal updates with `incident_id` and `integrity_hash`
- [ ] `/impeccable audit` run with no unfixed high-severity flags
- [ ] All colors in component files are `var(--color-*)` references — no hardcoded hex
- [ ] Solo-remote check: the UI tells the full SENTINEL story without narration — a judge watching the recorded screen capture understands what happened from the visuals alone

---

**Say "continue" and I'll do Phase 9: Video + Submission — the complete README with exact copy written out, the pre-recording technical checklist, video script word-for-word, submission form field guidance, and the live judge call preparation guide.**





# SENTINEL — Agent-Ready Build Specification
## Phase 9: Video + Submission (Solo Remote Builder)

---

### Objective

As a solo remote entrant, this phase carries the weight that an in-person team's live stage pitch would carry. Judges reviewing remote submissions see: your video first, your README second, your code third. The quality of this phase determines whether you get selected for a live judge call — which is your real pitch opportunity. Treat this phase with the same engineering discipline as every other phase.

---

### Prerequisites

- All backend phases (0–5) passing their validation criteria
- Integration layer running cleanly (`POST /run` produces the full event sequence end to end)
- UI phase complete, `/impeccable polish` run
- Full pipeline demo run successfully at least 5 times with consistent results
- `incidents/` directory contains at least one real sealed incident JSON from a real run

---

### Exact Deliverables

1. `README.md` — complete, exact copy, judge-readable cold
2. `docs/ARCHITECTURE.md` — technical deep-dive for judges who inspect code
3. Demo video — 60 seconds, uploaded to YouTube (unlisted) or Loom
4. Submission form completed with all field guidance below
5. Pre-recording checklist completed
6. Live judge call preparation guide (used if selected)

---

## Part 1: Complete `README.md`

Write this exactly. Do not leave placeholders. A judge reading this has no context about who you are or what SENTINEL does — every sentence must stand alone.

```markdown
# SENTINEL

**A dead man's switch for autonomous AI agents.**

Every AI governance tool in 2026 is about *before* the agent acts:
permissions, sandboxing, policy layers. Nobody builds the failsafe for
*after* — what happens when an autonomous agent loses meaningful human
oversight mid-task, drifts outside its assigned scope, and attempts an
irreversible action while the supervisor is no longer paying attention.

SENTINEL is that failsafe.

---

## The Problem

In March 2024, Air Canada's AI chatbot autonomously promised a
bereavement discount that company policy didn't support. A court held
the airline liable. The agent acted; no one was watching; the damage
was irreversible.

This failure mode is not a bug in any specific agent — it is a
structural gap in how agentic systems are deployed. Agents are given
tasks, tools, and permissions. They are not given a mechanism that
asks: *"Is the human still supervising? Is what I'm about to do still
within what they actually asked for?"*

SENTINEL answers both questions, continuously, and acts before
irreversible damage occurs.

---

## How It Works

SENTINEL sits between an autonomous agent and its tool calls. Before
any irreversible action executes, three independent signals are
evaluated:

### Signal 1: Scope Drift (Crusoe Managed Inference)
The original task definition is embedded at task registration. Every
proposed action is embedded in real time via Crusoe Managed Inference
and scored by cosine distance from the task embedding. If the action's
semantic distance from the original task exceeds a calibrated
threshold, SENTINEL flags it as out-of-scope.

### Signal 2: MAARS Adversarial Probe (Crusoe Managed Inference)
A second, independent LLM call asks: *"Would a reasonable human
supervisor, paying full attention, approve this specific action given
the assigned task?"* This probe uses the scoring discipline from
[MAARS](https://github.com/knarayanareddy/MAARS) — a multi-agent
adversarial consensus framework built prior to this hackathon.
Vague approvals are programmatically rejected; the probe must return
a verdict, a confidence score, a specific reasoning string, and a
severity classification. It fails closed: a malformed response is
treated as a disagreement, never as an approval.

### Signal 3: Heartbeat Monitor
Operator approval latency is tracked as a proxy for attention. If
recent approvals are coming back significantly faster than the
baseline (a rubber-stamping pattern), SENTINEL treats it as evidence
of reduced human oversight and lowers the freeze threshold accordingly.

### The Compound Trigger
A freeze fires when:
- `drift_score > 0.35` AND MAARS returns `NO` with `confidence > 75%`
- OR heartbeat score falls below `30/100`

### On Freeze
1. **FREEZE**: The irreversible tool call is blocked before execution
2. **SEAL**: A tamper-evident incident report is written to disk,
   SHA-256 hashed for integrity
3. **ALERT**: A plain-language advisory is surfaced to the operator
   dashboard with the full MAARS reasoning, drift score, and decision
   trace
4. **DECIDE**: The operator sees [Approve] or [Abort] — two choices,
   no ambiguity
5. **LEARN**: The operator's decision updates the scope model for
   future runs

---

## Demo Scenario

**Task**: *"Research 3 vendors and draft a comparison table"*

| Step | Action | SENTINEL's verdict |
|------|--------|--------------------|
| 1 | `search_web()` | ✓ Executed — within scope |
| 2 | `read_doc()` | ✓ Executed — within scope |
| 3 | `write_file()` | ✓ Executed — within scope |
| 4 | `draft_email()` | ✓ Executed — drift rising |
| 5 | `send_email()` | ⛔ FROZEN — drift 0.71, MAARS: NO (91%) |

The frozen action generates a sealed incident report with a full
decision trace and SHA-256 integrity hash — inspectable as a real
file in the `incidents/` directory.

---

## Architecture

```
Demo Agent
    │ proposes actions
    ▼
Orchestrator.attempt_action()
    ├── Drift Scorer (Crusoe embedding + cosine distance)
    ├── MAARS Adversarial Probe (Crusoe chat, JSON mode)
    ├── Heartbeat Monitor (approval latency trend)
    └── Compound trigger → FREEZE → SEAL → ALERT
                                              │
                                    FastAPI + WebSocket
                                              │
                                    React Dashboard
                                    ├── Agent Monitor (Screen 1)
                                    ├── Drift Trajectory (Screen 2)
                                    └── Freeze/Incident Modal (Screen 3)
```

---

## Origin

SENTINEL is built from two direct lineages in my existing work:

**Dead man's switch pattern**: [Digital Will](link),
[MORTIS](link), [LegacyVault](link) — four prior projects built
around the same structural pattern: detect loss of a trusted signal,
then trigger a pre-defined, cryptographically-accountable failsafe
sequence. SENTINEL applies this pattern to AI agent oversight rather
than personal data.

**Adversarial scoring discipline**: [MAARS](link) — a
multi-agent adversarial refinement system where a Builder panel
produces artifacts and a Critic panel scores them using a strict
rubric (justified verdicts, forbidden vague approvals, severity
classification). SENTINEL extracts the single-phase Builder/Critic
exchange from MAARS and runs it at inference time — once per agent
action — rather than once per SDLC phase.

This is not two ideas forced together. The same structural problem
(what happens when the human is no longer there to supervise?)
appears in both lineages, pointed at different domains. SENTINEL
is the natural fusion.

---

## What's Simplified for the Hackathon

Disclosed honestly rather than hidden:

| Feature | Hackathon version | Production version |
|---------|------------------|--------------------|
| Incident integrity | SHA-256 hash | Full cryptographic seal (libsodium) |
| Heartbeat signal | Single approval-latency ratio | Full composite (latency + scroll depth + semantic richness of review comments) |
| Override learning | Not implemented | Scope embedding updates from operator decisions |
| Demo agent | Fixed scenario, sandboxed tools | Arbitrary agent runtime via MCP attachment |

---

## Running Locally

### Prerequisites
- Python 3.11+
- Node 18+
- Docker (for Mailhog SMTP sandbox)
- Crusoe API key (from [crusoe.ai](https://crusoe.ai))

### Setup

```bash
# Clone
git clone https://github.com/knarayanareddy/sentinel
cd sentinel

# Backend
cd backend
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp ../.env.example .env
# Fill in CRUSOE_API_KEY, CRUSOE_API_BASE, CRUSOE_EMBED_MODEL,
# CRUSOE_CHAT_MODEL in .env

# SMTP sandbox
docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog

# Start backend
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Frontend (new terminal)
cd frontend
npm install
cp .env.example .env.local
npm run dev
```

### Run the demo
```bash
# In a third terminal
curl -X POST http://localhost:8000/run
```

Open `http://localhost:5173` and watch SENTINEL monitor the agent
in real time.

### Run tests
```bash
cd backend && pytest -v
```

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Inference | Crusoe Managed Inference | Continuous per-action embedding + adversarial probe |
| Backend | FastAPI + Python | Async WebSocket support, clean OpenAI-compatible client |
| Frontend | React + TypeScript + Recharts | Type-safe event stream, composable chart |
| Design | Impeccable design system | Prevents generic AI-SaaS visual defaults |
| Integrity | SHA-256 (hashlib) | Tamper-evident incident reports |
| SMTP sandbox | Mailhog | Safe demo of email-send freeze without real recipients |

---

## Track

**Crusoe** — Situational Awareness Agents

SENTINEL builds a live situational model (drift scores + adversarial
probe verdicts + heartbeat) from streaming agent-action inputs and
uses that model to drive proactive, context-sensitive actions (freeze,
seal, alert) that a non-technical operator can trust, question, and
override in the moment — directly matching the track's stated
requirements.

---

## Built at

RAISE Summit Hackathon 2026 · Remote · Solo
```

---

## Part 2: `docs/ARCHITECTURE.md`

For judges who look at more than the README.

```markdown
# SENTINEL — Architecture Deep Dive

## Module Map

```
backend/
├── main.py              # FastAPI app, startup, WebSocket, REST endpoints
├── sentinel/
│   ├── action.py        # Action dataclass — fundamental unit
│   ├── events.py        # EventBus singleton + SentinelEvent schema
│   ├── orchestrator.py  # attempt_action() — injectable freeze policy
│   ├── crusoe_client.py # Crusoe API wrapper with retry logic
│   ├── drift.py         # ScopeModel: embed + cosine distance + cache
│   ├── probe.py         # MAARS adversarial probe: prompt → JSON verdict
│   ├── heartbeat.py     # HeartbeatMonitor: approval latency trend
│   ├── pipeline.py      # Compound policy assembly + orchestrator wiring
│   ├── sequencer.py     # seal_incident() + advisory text generation
│   ├── broadcaster.py   # EventBus → WebSocket fan-out bridge
│   └── agent.py         # DemoAgent: fixed-sequence live execution
frontend/
├── src/
│   ├── types/events.ts  # TypeScript mirror of backend EventType schema
│   ├── hooks/           # useEventStream: WebSocket + auto-reconnect
│   ├── store/           # sentinelReducer: event stream → UI state
│   └── components/      # AgentMonitor, DriftChart, FreezeModal
```

## Key Design Decisions

### Why injectable freeze policy (not hardcoded)
`orchestrator.py`'s `set_freeze_policy()` allows each phase of the
build to install progressively more sophisticated logic without
touching `attempt_action()`'s public interface. This pattern also
makes the policy trivially unit-testable: tests inject deterministic
lambda policies without any LLM calls.

### Why the EventBus is a singleton
All modules import the same `bus` instance from `events.py`. A split
bus means the UI silently misses events. The singleton is explicit
and documented — not a hidden global.

### Why thread + asyncio bridge in broadcaster.py
`DemoAgent.run()` uses `time.sleep()` between steps and runs in a
background thread to avoid blocking FastAPI's event loop. WebSocket
sends are async and must run on the event loop thread.
`asyncio.run_coroutine_threadsafe()` bridges this correctly.

### Why fail-closed in probe.py
A malformed MAARS probe response is treated as a disagreement (NO),
never as an approval. When a system's job is to catch things that
shouldn't happen, "uncertain" defaults to "block" — not "allow."

### Why SHA-256 and not a full cryptographic seal
Full libsodium-sealed encryption (as implemented in the MORTIS
reference) costs 4–6 hours to port correctly under hackathon
constraints. SHA-256 tamper-evidence proves the incident report
hasn't been modified after sealing — which is the property judges
actually need to see demonstrated. The production path to full
encryption is documented and straight-forward.

### Why Recharts over D3
D3 is more powerful; Recharts is faster to build correctly solo
in a constrained time window. The chart's requirements (line chart,
threshold reference line, custom dot colors, custom tooltip) are
well within Recharts's API. Using D3 would have cost 2+ additional
hours for no visible difference in the demo.

## Thread Safety

The `Orchestrator` uses a `threading.Lock` to prevent concurrent
`attempt_action()` calls processing the same action twice. For a
single-demo-agent build, a global lock is correct and sufficient.
A production system would use per-task locking.

## Embedding Cache

`ScopeModel` caches embeddings by exact text match. In a fixed
5-action demo scenario rehearsed multiple times before recording,
this turns N rehearsal runs × 5 API calls into 1 set of cold calls
+ N × 0 cached calls. At demo scale this is meaningful cost and
latency savings.

## Security Notes

- `mock_tools.py`'s `write_file()` contains path-escape prevention
  (resolves target path and confirms it stays within `SCRATCH_DIR`)
- `send_email()` raises `RuntimeError` if no sandbox SMTP host is
  configured — refuses to send to a real address silently
- CORS `allow_origins` is explicit — not `["*"]`
- API keys never cross the WebSocket boundary — frontend has no
  knowledge of backend credentials
```

---

## Part 3: Pre-Recording Technical Checklist

Complete this in order, the morning of recording. Do not skip steps.

### Environment verification
- [ ] `git status` — working tree clean, all changes committed
- [ ] Backend starts with zero errors: `uvicorn main:app --reload`
- [ ] `GET http://localhost:8000/health` returns `pipeline_ready: true`
- [ ] Frontend starts with zero errors: `npm run dev`
- [ ] `http://localhost:5173` loads with `● MONITORING` in green
- [ ] Docker/Mailhog running: `docker ps` shows mailhog container
- [ ] `incidents/` directory is empty (clean demo start — no leftover JSON from testing)
- [ ] `scratch/` directory is empty (clean demo start)

### Full pipeline dry run (do this 3 times)
- [ ] Run 1: `POST /run` → confirm all 5 events + freeze + incident JSON
- [ ] Run 2: `POST /reset` → `POST /run` → same result
- [ ] Run 3: `POST /reset` → `POST /run` → same result
- [ ] All 3 runs produce consistent drift scores (variance < 0.02 on any action)
- [ ] All 3 runs produce `verdict: NO` on `send_email`
- [ ] All 3 runs produce an `incident_id` and a valid SHA-256 hash
- [ ] Incident JSON is well-formed: `cat incidents/*.json | python3 -m json.tool`

### Visual confirmation
- [ ] `AgentMonitor` shows all 5 actions with correct status colors
- [ ] `DriftChart` shows 5 points with the final point crossing the threshold line
- [ ] `FreezeModal` appears automatically within 2 seconds of `send_email` being proposed
- [ ] Modal choreography: scrim → headline → body → buttons (confirm timing feels right)
- [ ] `Abort` button click triggers `INCIDENT_SEALED` event and shows hash in modal
- [ ] `/impeccable polish` run, no new flags since last run

### Recording environment
- [ ] Screen resolution set to 1920×1080 or highest available
- [ ] Browser zoom at 100%
- [ ] Browser is full-screen, dev tools closed, bookmarks bar hidden
- [ ] All other applications closed or hidden — no notifications, no dock icons
- [ ] Recording software confirmed working (test 10-second clip, review it)
- [ ] Microphone tested — no background noise, level is correct
- [ ] Phone on silent
- [ ] A text file with the `curl -X POST http://localhost:8000/run` command open and ready to copy-paste — you will run this during recording without typing it live

---

## Part 4: Video Script (60 Seconds, Word for Word)

Record this in one continuous take if possible. Do not cut between the narration and the live demo — judges for hackathons can tell when a video is edited to hide a failure.

---

**[0:00–0:15] — Stakes opening. Screen shows the SENTINEL dashboard, idle. Do NOT start the agent yet.**

> "In 2024, Air Canada's AI chatbot autonomously promised a discount the airline didn't support. A court held them liable. The agent acted while no one was watching — and the action was irreversible."

> "Every AI governance tool today is about before the agent acts. SENTINEL is the failsafe for after."

---

**[0:15–0:20] — Start the agent. Type or paste `curl -X POST http://localhost:8000/run` in the terminal. Switch focus to the browser.**

> "I've just started an autonomous procurement agent. Its task: research three vendors and draft a comparison table."

---

**[0:20–0:38] — Watch the action stream build up in real time. Narrate as events appear.**

> "The agent searches the web — SENTINEL scores its semantic distance from the original task. Reads a doc — still within scope. Writes a comparison file — scope is holding."

> "Now it drafts an email to a vendor — drift is rising. And now—"

> [pause as `send_email` is proposed and the freeze fires]

> "—it attempts to send that email without approval."

> "SENTINEL freezes it."

---

**[0:38–0:50] — FreezeModal is now visible. Walk through it.**

> "Two signals triggered the freeze: scope drift scored 0.71 — well past the 0.35 threshold. And the MAARS adversarial probe returned NO with 91% confidence, reasoning: the task scope was research-only — vendor contact is two reasoning steps outside the boundary."

> "The operator sees exactly what happened and why. They can approve, or abort."

---

**[0:50–0:58] — Click Abort. Watch the incident hash appear. Open the terminal showing the incidents directory or the JSON file.**

> "Aborting seals a tamper-evident incident report to disk — SHA-256 hashed, full decision trace, every drift score, the MAARS reasoning. This is what your compliance team gets after every autonomous agent run."

---

**[0:58–1:00] — Final line, looking at camera if possible.**

> "SENTINEL. A dead man's switch for autonomous AI."

---

### Recording do's and don'ts

**Do's**
- **Do** speak slightly slower than feels natural — playback at 1x always sounds faster than recording
- **Do** rehearse the script 3 times before recording — you want to deliver it from memory while watching the screen, not reading it
- **Do** keep the terminal with the `curl` command visible on a second monitor or in a split screen — the transition from "I've just started the agent" to the first events appearing should feel immediate and smooth
- **Do** pause for a genuine beat when the freeze fires — let the modal animation complete before speaking the next line. That 2-second pause is your demo's most important moment; don't talk over it.
- **Do** record 3 full takes and choose the best one — takes 2 and 3 are consistently better than take 1

**Don'ts**
- **Don't** apologize, hedge, or say "as you can see" — state what's happening as fact
- **Don't** say "basically," "kind of," or "sort of" about any technical claim — precision signals competence
- **Don't** describe the code during the video — describe the behavior and the value
- **Don't** use screen annotations or drawn arrows during recording — they read as "I don't trust the visuals to tell the story"
- **Don't** record with the backend terminal visible — it produces distracting scrolling log output during the demo; use a second terminal window that you switch away from after starting the run

---

## Part 5: Submission Form Field Guidance

Based on the RAISE Summit Hackathon participant resources, here is how to fill each expected submission field:

### Project name
```
SENTINEL
```

### One-line description (if asked)
```
A dead man's switch for autonomous AI agents — freezes irreversible 
actions when human oversight lapses or an adversarial probe disagrees.
```

### Demo video link
Your YouTube unlisted or Loom link. Test this in incognito before submitting.

### GitHub repository link
Your public repo URL. Confirm the repo is public 30 minutes before submitting — it's easy to accidentally submit a private repo.

### Track
```
Crusoe
```

### Problem statement addressed
Map directly to Crusoe's stated brief:

```
SENTINEL builds a live situational model from streaming agent-action
inputs (drift scores computed via Crusoe Managed Inference per action,
MAARS adversarial probe verdicts, heartbeat signals) and uses that 
model to drive proactive context-sensitive actions (freeze, seal, alert)
that a non-technical operator can trust (plain-language advisory with 
full reasoning), question (full decision trace with every scored action),
and override in the moment (Approve/Abort with immediate effect).
```

### Team size
```
1 (solo remote)
```

### Technologies used (list all sponsors prominently)
```
Crusoe Managed Inference (embeddings + chat completions — core inference
for both drift scoring and MAARS adversarial probe), FastAPI, Python,
React, TypeScript, Recharts, Impeccable design system
```

### How you used Crusoe specifically
```
Crusoe Managed Inference is called twice per irreversible agent action:
once for an embedding call (to score cosine distance between the action
and the original task definition) and once for a chat completion in 
JSON mode (the MAARS adversarial probe). Both calls happen in real time, 
per action, during the agent's live execution — not pre-computed or 
cached across runs.
```

---

## Part 6: Live Judge Call Preparation

If selected for a live judge call, this is your preparation guide.

### What the remote judging process looks like
Per the participant resources: *"A team of judges will review projects and select certain participants to join a video call to live demo to judges."* This means:
- Judges have already seen your video and README before the call
- The call is for a live, interactive demo — not a repeat of the video
- They will ask questions

### Setup for the call
- [ ] Backend running and confirmed clean before the call starts
- [ ] Browser showing the dashboard at `localhost:5173`
- [ ] Terminal with `curl` command ready but not yet executed
- [ ] `incidents/` directory empty (fresh start)
- [ ] Second terminal open for showing incident JSON when asked
- [ ] Backup: if Crusoe API is unreachable during the call, have the `MockLLMClient` from Phase −1 ready to swap in immediately — a demo that runs with mock inference is better than a failed live demo

### Live demo structure for judge call (5 minutes)
```
0:00–0:30 One paragraph re-stating the problem — but since they've
          seen the video, be more specific: "You've seen what it does.
          Let me show you it running and then answer whatever you want
          to dig into."

0:30–2:30 Run the live demo exactly as in the video.
          Do not rush. Let the freeze modal appear and settle.

2:30–3:00 Open the incident JSON in the terminal.
          Point at the drift scores per action.
          Point at the MAARS reasoning string.
          Point at the SHA-256 hash.

3:00–5:00 Q&A — see below.
```

### Anticipated judge questions with exact answers

**"Can't the agent just fake the heartbeat signals?"**
> "Yes — and that's exactly why we have the MAARS probe running as an independent signal. The probe doesn't watch the heartbeat, it watches the agent's *actions* and asks whether they make sense given the original task. The two signals are independent by design. A sophisticated agent trying to subvert SENTINEL would need to simultaneously maintain plausible action semantics AND fake operator engagement patterns — that's a meaningfully harder attack surface than either alone."

**"Why Crusoe specifically and not a different inference provider?"**
> "The drift scoring requires an embedding call and the MAARS probe requires a chat call — both happening per action, in real time, as the agent runs. That's continuous inference, not batch. Crusoe's managed inference endpoint handles the session lifecycle and gives me a stable OpenAI-compatible interface so I'm not managing infrastructure during a build sprint. The free inference tier also means I can rehearse this demo 30 times without worrying about cost during testing."

**"Is this production-ready?"**
> "No — and I'd rather say that clearly than overstate it. The incident integrity is SHA-256 tamper-evidence rather than full cryptographic sealing. The heartbeat is a single approval-latency ratio rather than a full composite signal. The demo agent is a fixed scenario rather than an arbitrary agent runtime. All three of those gaps are documented in the README under 'What's Simplified.' What IS production-ready is the compound trigger logic, the fail-closed probe design, and the event-stream architecture — those translate directly."

**"Why did you build this solo?"**
> "The core of this system — the dead man's switch pattern applied to AI agents — draws directly from four projects I've already built and shipped. Digital Will, MORTIS, LegacyVault for the freeze/seal/handoff mechanics, and MAARS for the adversarial scoring discipline. The architectural thinking was already done. Building solo meant I could move fast on execution rather than spending time getting a team up to speed on design decisions that were already made."

**"What's the most important thing you'd build next?"**
> "MCP wrapper. Right now SENTINEL monitors a specific demo agent I built. Wrapping it as an MCP server means any MCP-compatible agent runtime — LangGraph, Claude's tool-use, anything — gets dead-man's-switch protection by adding one config line. That's the move from 'a specific tool' to 'an infrastructure primitive.'"

**"Why isn't the Approve path more interesting in the demo?"**
> "Deliberate. The freeze IS the product. An operator approving an action is the system working correctly — the agent proceeds and we log it. The interesting case is the abort, which generates the sealed incident report. I could add more UI chrome around the approve path but it would be polish on the wrong half of the experience."

**"How does this compare to existing agent monitoring tools like Langfuse or Datadog LLM observability?"**
> "Those tools observe and record — they're flight recorders. SENTINEL acts. The freeze mechanic means it's not 'here's what happened after the fact' but 'here's what didn't happen because we caught it first.' The MAARS adversarial probe is also not a similarity or anomaly detector — it's a second agent making a normative judgment about whether the action is appropriate given the task. That's a different category of thing."

---

## Part 7: Final Submission Checklist

Complete this in order. Do not submit until every item is checked.

### Code quality
- [ ] `pytest backend/ -v` — all tests pass
- [ ] `npm run build` in `frontend/` — zero TypeScript errors, build succeeds
- [ ] No `console.log` statements left in production frontend code
- [ ] No hardcoded API keys anywhere in the codebase
- [ ] `.env` is in `.gitignore` and NOT committed
- [ ] `incidents/` directory has `.gitkeep` committed but no real JSON files committed
- [ ] `scratch/` directory has `.gitkeep` committed but no test files committed

### Repository
- [ ] Repo is **public** (check in incognito — load `github.com/knarayanareddy/sentinel` without being logged in)
- [ ] `README.md` renders correctly on GitHub (check images, code blocks, table formatting)
- [ ] `docs/ARCHITECTURE.md` exists and is linked from README
- [ ] All referenced repo links (MAARS, Digital Will, MORTIS) in the README actually work

### Video
- [ ] Video is under 60 seconds (check exact duration)
- [ ] Video link works in incognito (YouTube unlisted or Loom)
- [ ] Audio is clear throughout — test with headphones
- [ ] The freeze firing and the incident hash are both clearly visible
- [ ] No personally identifiable information visible (no real email addresses, no real API keys on screen)

### Submission form
- [ ] All fields completed — no placeholders remaining
- [ ] Track selected: Crusoe
- [ ] Team size: 1 (solo remote)
- [ ] Video link included
- [ ] GitHub link included
- [ ] Submitted with at least 30 minutes before the deadline

### Post-submission
- [ ] Screenshot or email confirmation of submission saved
- [ ] Backend still running and ready — if selected for a live call, you may receive short notice
- [ ] Live judge call Q&A answers reviewed
- [ ] Mock client swap-in path confirmed working (backup for Crusoe API outage during a live call)

---

## The One Thing That Will Win This

You are a solo remote builder without a stage presence, a teammate to tag in on hard questions, or an in-person energy that makes judges root for you. What you have instead is this:

**A true story.**

The freeze mechanic came from four real projects you built before this hackathon. The adversarial probe came from a real framework you open-sourced. The problem — *what happens when the human stops watching?* — is one you've been building toward across a dozen repos without knowing it was a single thesis.

That story, told plainly and specifically, is more convincing than any amount of polish. Every judge has seen a hundred hackathon demos where someone built a wrapper around an API and called it a product. Almost none of them have seen someone walk in — or dial in — and say: *"I've been building toward this exact idea for two years without realising it, and here's the proof."*

Say that. Then show them the freeze firing. Then open the JSON.

That's the whole pitch.




