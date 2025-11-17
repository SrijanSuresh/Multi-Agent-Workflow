# Multi-Agent Workflow

<div align="center">
<strong>OOP-first scaffold for an LLM↔LLM ring: <code>LLM1 → LLM2 → LLM3 → LLM1 → …</code> until critic + agreement rules pass.</strong>
</div>
> Status: **v0 prototype** – only minimal ring (mock agents + consensus) implemented. Remaining modules are planned (see Roadmap).

---
<img width="1170" height="799" alt="image" src="https://github.com/user-attachments/assets/38d41818-6567-46a6-8d6c-4b3c645fdcd1" />

`In this workflow, the API accepts a user prompt and hands off to a worker that runs the ring scheduler; API and worker use a cache for idempotency, short‑lived turn state, and rate limits, while a db persists runs, messages, decisions, and metrics. Inside the red loop, Coordinator → Planner → Retriever → Critic exchange Message objects; Tools (search, RAG, OCR, vision) sit beside them and are fronted by a cache to memoize retrieval and tool results, with a db beneath Tools for retrieval stores or artifact pointers. The Critic updates common ground, and the scheduler checks agreement; when accepted, results are finalized and written to the db.`


## Table of Contents
1. [Concept Overview](#concept-overview)
2. [Directory Layout](#directory-layout)
3. [Component Interactions](#component-interactions)
4. [Core Contracts](#core-contracts)
5. [Development Workflow](#development-workflow)
6. [Quickstart](#quickstart)
8. [Roadmap & Status](#roadmap--status)
9. [Extending the System](#extending-the-system)
10. [Testing Strategy](#testing-strategy)
11. [FAQ](#faq)

---

## Concept Overview
This project simulates a collaborative job environment among specialized LLM agents. A user prompt seeds shared state; agents exchange strictly structured `Message` objects. A planned `critic` agent will validate claims against retrieved evidence and a `CommonGround` store. The loop halts on **agreement** (all outputs converge) or guardrail triggers (max loops, timeout, veto). Retrieval-Augmented Generation (RAG) becomes iterative: retrieval + validation inside the multi-turn agent conversation.

---

## Directory Layout
| Path | Purpose | Status |
|------|---------|--------|
| `apps/api/main.py` | FastAPI app (POST `/orchestrate`)
| `apps/worker/run_worker.py` | Current ring loop + consensus 
| `core/schemas.py` | Pydantic models (`Message`, request/response, trace)
| `core/config.py` | Settings via `BaseSettings` 
| `core/scheduler.py` | Advanced ring scheduler (timeouts/backoff)
| `core/telemetry.py` | Metrics/tracing hooks
| `agents/retriever.py` | Retrieval stub 
| `agents/base.py` | Abstract `Agent` contract 
| `agents/coordinator.py` | Normalizes user goal; seeds state
| `agents/planner.py` | Goal decomposition (plan/DAG) 
| `agents/critic.py` | Claim & citation validator
| `tools/base.py` | `LLMTool` + deterministic `MockLLM` 
| `tools/*` | Future tool adapters (search/RAG/OCR/vision) 
| `tests/test.py` | Basic consensus test 
| `eval/run_eval.py` | Evaluation harness entrypoint 
| `metrics.md` | Measurement definitions 
| `docker-compose.yml` | Local container orchestration 
| `Dockerfile` | Image for API worker 
| `Makefile` | Convenience tasks 

---

## Component Interactions
1. User sends prompt (API/CLI).
2. Scheduler (currently `run_multi_agent_loop`) initializes message list.
3. Each agent (MockLLM now, future `Agent.handle`) appends output as a `Message`.
4. (Planned) Retrieval/search tools invoked; artifacts with citations are added.
5. (Planned) Critic reviews outputs + evidence; may veto → extra retrieval.
6. Agreement check chooses finalize vs another loop.

---

## Core Contracts
| Contract | Intent |
|----------|--------|
| `Message` | Sole inter-agent communication unit. |
| `AgentResponse` | Structured agent output (content + optional reasoning). |
| `LLMTool.generate(messages)` | Provider-agnostic text generation interface. |
| `Agent.handle(message, state)` | (Planned) Pure-ish agent turn enabling deterministic tests. |
| `CommonGround` | (Planned) Shared evolving beliefs/evidence graph. |
| `Tool.run(**kwargs)` | Backend-neutral tool invocation returning artifacts + citations. |

---

## Development Workflow
```powershell
make install      # pip install -r requirements.txt
make dev          # run FastAPI with reload
make test         # run unit tests
make docker-up    # build & start containers
make docker-down  # stop containers
```
Future targets: `make lint`, `make format`, `make eval`, `make seed-data`.

---

## Quickstart
1. Install deps:
	```powershell
	pip install -r requirements.txt
	```
2. Run API:
	```powershell
	uvicorn apps.api.main:app --reload --host 0.0.0.0 --port 8000
	```
3. Call endpoint:
	```http
	POST http://localhost:8000/orchestrate
	Content-Type: application/json

	{
	  "prompt": "What is 2 + 2?",
	  "max_loops": 5
	}
	```
4. Run tests:
	```powershell
	python -m unittest -v
	```
Response JSON: `final_output`, `agreed`, `steps`, `trace`.

---

## Observability & Safety (Planned)
Metrics per turn: latency, tokens, cost, tool calls, critic verdicts, agreement score.
Guardrails: `max_loops`, global timeout, retry + jitter, citation enforcement, context size limits.

---

## Roadmap & Status
| Version | Focus | Key Additions | Status |
|---------|-------|---------------|--------|
| v0 | Text-only ring | Mock agents, consensus, minimal API | ✅ Current |
| v1 | Basic RAG | Retriever wiring, citation packing | ⏳ Pending |
| v2 | Critic & rubric | Claim validation, disagreement fetch | ⏳ Pending |
| v3 | Multimodal | Vision/OCR tools + tasks | ⏳ Pending |
| v4 | Eval harness | Benchmarks, scoring, release prep | ⏳ Pending |

---

## Extending the System
1. Add provider: subclass `LLMTool` (e.g. `OpenAILLM`) and inject in `apps/api/main.py`.
2. Create `agents/base.py` & refactor loop to use `Agent.handle`.
3. Implement advanced scheduler (`core/scheduler.py`) with timeouts/backoff.
4. Add `CommonGround` structure; integrate with critic + planner.
5. Wire retrieval (enhance `agents/retriever.py` to return docs; prepend context messages).
6. Telemetry hooks (`core/telemetry.py`) called each round.

---

## Testing Strategy
Layers:
1. Unit – normalization, agreement logic, mock generation.
2. Integration – ring + retrieval + critic (future).
3. Evaluation – task suites measuring convergence speed, factuality, citation coverage.
4. Property tests – agreement monotonicity under normalization (future).

Current test: `tests/test.py` asserts consensus and prefixed answer.

---

## FAQ
**Why a ring (vs hub)?** Predictable ordering and simpler early reasoning; hub can be layered later.

**Why start with a MockLLM?** Deterministic, cheap, facilitates rapid iteration and CI before adding provider variability.

**How will citations work?** Retrieved docs assigned IDs; agents embed citations (e.g. `[doc:3]`); critic enforces coverage + relevance.

**How is agreement computed now?** Unanimous normalized text equality. Future: semantic similarity threshold + critic acceptance.

---


### Prototype Flow (v0)
```text
User Prompt → [LLM1] → [LLM2] → [LLM3] → agreement? → (repeat or finalize)
```
Final output shape: `{ final_output, agreed, steps, trace }`.

---

Happy building – contributions adding planned modules welcomed once scaffolding stabilizes.
