# Evaluator Component — Technical Implementation Spec

## 1. Input: exact data structure

The Evaluator consumes a **`ConversationTurn`** object — not raw text. In production you
almost never pass a bare string; you pass a structured object so you retain metadata
needed for provenance, deduplication, and downstream storage.

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Literal, Optional

class ConversationTurn(BaseModel):
    turn_id: str                     # UUID, unique per message
    conversation_id: str             # UUID, groups turns into a session
    user_id: str                     # who this memory belongs to
    role: Literal["user", "assistant"]
    content: str                     # the raw message text
    timestamp: datetime
    # Optional but useful:
    prior_turns: list[str] = []      # short rolling context window (e.g. last 3-5 turns)
                                      # needed because a single turn is often ambiguous
                                      # ("yeah, her too" means nothing without context)
```

**Why a rolling context window matters technically:** an LLM classifying "her too" in
isolation cannot decompose it into anything meaningful. Most real memory systems (Mem0,
Zep) pass a small buffer of recent turns alongside the target turn specifically to
resolve pronouns and ellipsis. You don't need the whole conversation — 3–5 turns is the
typical window used in published implementations.

`Pydantic` is the standard choice here (not just a raw `dict`) because:
- It gives you runtime validation for free (reject malformed turns before they hit the LLM).
- It gives you `.model_json_schema()` — auto-generated JSON schema you can hand directly
  to an LLM's structured-output / tool-use feature (see §3).

---

## 2. Output: exact data structure

The Evaluator returns a **`list[Decision]`** — one `Decision` per atomic fact extracted
from the turn (a single turn can produce 0, 1, or several decisions).

```python
from enum import Enum

class MemoryType(str, Enum):
    EPISODIC = "episodic"        # something that happened, time-bound
    SEMANTIC = "semantic"        # a stable fact about the world/user
    PROCEDURAL = "procedural"    # a preference or how-to pattern

class Action(str, Enum):
    STORE_NEW = "store_new"
    UPDATE_EXISTING = "update_existing"
    SKIP_DUPLICATE = "skip_duplicate"
    SKIP_LOW_VALUE = "skip_low_value"
    FLAG_CONFLICT = "flag_conflict"

class Decision(BaseModel):
    candidate_id: str                  # UUID for this extracted candidate
    source_turn_id: str                # traceability back to ConversationTurn
    content: str                       # the atomic fact, normalized text
                                        # e.g. "User's daughter is named Priya"
    memory_type: MemoryType
    confidence: float                  # 0.0-1.0, how sure the classifier is
    salience: float                    # 0.0-1.0, how important/worth retrieving later
    sensitivity: Literal["none", "low", "high"]  # PII/health/financial flags
    action: Action
    related_memory_id: Optional[str]   # set if action is UPDATE_EXISTING or FLAG_CONFLICT
    embedding_ref: Optional[str]       # pointer to the vector store entry, not the raw vector
    reasoning: Optional[str]           # short justification, useful for debugging/audit
```

This is the **contract** between the Evaluator and everything downstream (the Classifier
and Storage components). Nothing downstream should ever touch raw LLM text — only this
validated schema.

---

## 3. Technology stack

| Layer | What it does | Typical choice |
|---|---|---|
| Schema / validation | Enforce input/output shape | **Pydantic v2** |
| LLM inference | Decomposition + classification + scoring | **Anthropic API** (Claude) or OpenAI API, called with structured output / tool-use so the response is guaranteed valid JSON matching your `Decision` schema — not free-text you regex out |
| Embeddings | Turn text into vectors for similarity search | OpenAI `text-embedding-3-small`, or open-source (`bge-large`, `nomic-embed`) via `sentence-transformers` if you want to avoid a second vendor dependency |
| Vector store | Store + search embeddings | **pgvector** (if you're already on Postgres — simplest for a project), or **Chroma** (embedded, zero-infra, great for a solo project), or **FAISS** (if you want raw speed and don't need persistence/metadata filtering) |
| Orchestration | Wiring the pipeline together | Plain Python functions/async calls are enough at this scale — you don't need LangChain/LlamaIndex for a 5-stage pipeline you're designing yourself. Pulling in a heavy orchestration framework here would also blur the "what did I build" line from your design doc. |
| Testing | Unit test each stage independently | `pytest`, with the LLM and vector store mocked for the pre-filter/decompose/conflict-logic tests |
| API layer (optional) | Expose `evaluate()` as a service | FastAPI, if you want this callable over HTTP rather than as a library function |

**Why structured output / tool-use matters technically:** if you just prompt "return
JSON" and parse the raw text, you will periodically get broken JSON, markdown code
fences, or extra commentary that breaks `json.loads()`. Anthropic and OpenAI both
support passing your Pydantic schema as a tool definition so the model's output is
constrained to match it. This removes an entire category of bugs and is what makes
Option C (own the logic, reuse the LLM as infrastructure) actually reliable in
practice.

---

## 4. End-to-end data flow

```
ConversationTurn (Pydantic object)
        │
        ▼
┌───────────────────┐
│ 1. Pre-filter      │  Pure Python, no network call.
│ (rule-based)       │  Regex/heuristics: message length, filler-word ratio,
│                    │  known low-value patterns ("ok", "lol", "thanks").
└───────────────────┘
        │ passes filter?
        ▼ yes
┌───────────────────┐
│ 2. Decompose        │  LLM call #1 (structured output).
│                    │  Input: turn.content + prior_turns.
│                    │  Output: list[str] — atomic candidate statements.
└───────────────────┘
        │
        ▼ (for each candidate, can be batched into one call)
┌───────────────────┐
│ 3. Classify + score│  LLM call #2 (structured output, schema = partial Decision).
│                    │  Input: one atomic candidate.
│                    │  Output: memory_type, confidence, salience, sensitivity.
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ 4. Embed            │  Embedding API call.
│                    │  candidate text → vector (e.g. 1536-dim).
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ 5. Similarity search│  Vector store query (cosine similarity, top-k=5).
│  + conflict logic   │  If max similarity > high threshold → likely duplicate.
│  (owned logic)      │  If similarity mid-range + semantically contradictory
│                    │  (checked via a small LLM comparison call) → conflict.
│                    │  Else → novel.
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ 6. Decision assembly│  Pure Python. Combines steps 3-5 into a validated
│  (owned logic)      │  Decision object. Sets `action` field.
└───────────────────┘
        │
        ▼
   list[Decision]  →  handed to the Classifier/Storage component
```

**Network calls per turn:** roughly 2–4 LLM calls (decompose, classify, optional
conflict-comparison) + 1 embedding call + 1 vector store query — per atomic candidate
for classify, but decompose is once per turn. Batch candidates into a single classify
call where possible to cut latency and cost.

---

## 5. Module boundaries (what you write vs. what you call)

```python
# evaluator/pipeline.py  — YOU write this file, it's the actual deliverable

def evaluate(turn: ConversationTurn) -> list[Decision]:
    if not passes_prefilter(turn):          # owned: pure Python
        return []

    candidates = decompose(turn)             # owned logic, calls llm_client (reused infra)
    decisions = []
    for c in candidates:
        scored = classify_and_score(c)       # owned logic, calls llm_client
        vector = embed_client.embed(c.text)  # reused infra
        matches = vector_store.query(vector) # reused infra
        action = resolve_conflict(scored, matches)  # owned logic
        decisions.append(build_decision(scored, action, matches))  # owned logic
    return decisions
```

Everything with `# owned logic` is what you'd defend in a design review. Everything
calling `llm_client`, `embed_client`, `vector_store` is a thin wrapper around someone
else's well-tested infrastructure — swappable without touching your logic, and testable
by mocking those three objects.

---

## 6. Practical implementation notes

- **Idempotency:** use `turn_id` to make `evaluate()` safe to re-run (e.g. on retry)
  without double-storing the same memory.
- **Batching classify calls:** send all decomposed candidates from one turn in a single
  structured-output call returning `list[Decision]` rather than one call per candidate —
  cuts latency/cost significantly.
- **Similarity threshold tuning:** you have no ground truth initially (noted as a risk in
  the design doc) — log raw similarity scores for every decision during early testing,
  then pick thresholds from the observed distribution rather than guessing a number.
- **Sensitivity flagging:** treat this as a separate lightweight classification (can be a
  rule-based keyword/entity check layered on top of the LLM output, not solely trusted
  to the LLM) since under/over-flagging PII has real consequences downstream.
