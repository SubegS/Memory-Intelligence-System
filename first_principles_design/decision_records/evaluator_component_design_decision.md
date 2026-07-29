# System Design Document
## Memory Intelligence System — Evaluator Component: Design Decision Record

**Document type:** Design Decision Record (component-level)
**Component:** Evaluator (Component 1 of the Memory Intelligence System)
**Status:** Approved
**Author:** [Your name]
**Date:** [Fill in]

---

## 1. Purpose and Scope

This document records the design decision for how the **Evaluator component** of the
Memory Intelligence System will be implemented. It captures the problem being solved,
the options considered, the reasoning behind the chosen approach, and what was
explicitly rejected and why. This is intended to serve as both a design justification
for review and a reference for implementation.

This document covers **only the implementation strategy of the Evaluator** — not its
full internal pipeline design (pre-filter, decomposition, classification, conflict
check), which is covered in the earlier architecture discussion and can be attached as
a companion document.

---

## 2. Background

The Memory Intelligence System is composed of two primary flows: an **ingestion path**
(evaluate → classify → store) and a **retrieval path** (query → search → rank →
return). The Evaluator sits at the front of the ingestion path and is responsible for:

1. Filtering out conversational input that is not worth storing.
2. Decomposing input into atomic candidate memory units.
3. Classifying each candidate into a memory type (episodic, semantic, procedural).
4. Scoring each candidate for confidence, salience, and sensitivity.
5. Checking each candidate against existing memory for redundancy or conflict.
6. Producing a structured decision object that downstream components act on.

Because this component determines *what enters memory at all*, its correctness has a
compounding effect on every later stage — a bad decision here either pollutes storage
with noise or silently drops something that mattered. This made the implementation
strategy for this component worth deciding deliberately, rather than defaulting to
"just call a library" or "build everything by hand."

---

## 3. Problem Statement

**How should the Evaluator component be implemented?**

Three implementation strategies were considered:

- **Option A** — Use an existing memory library's evaluator as-is (e.g., Mem0, Zep,
  Letta), treating it as a black box.
- **Option B** — Build the entire Evaluator from scratch, including the underlying
  infrastructure (embeddings, vector indexing, storage primitives).
- **Option C** — Build the Evaluator's decision logic independently, while using
  existing, well-established infrastructure primitives (embedding models, vector
  stores) as supporting plumbing rather than reimplementing them.

---

## 4. Options Considered

### Option A — Adopt an existing library's evaluator directly

Call a pre-built pipeline (e.g. `mem0.add(message)`) and let its internal
extraction/classification logic make all decisions.

| Pros | Cons |
|---|---|
| Fastest path to a working system | Decision logic is a black box — no insight into *why* it classified something a certain way |
| Production-tested, benchmarked on real datasets (e.g. LoCoMo) | Fixed taxonomy and schema — cannot easily add a 4th memory type or change scoring logic |
| Frees time for other components (retrieval, ranking, UI) | Not defensible as a system design contribution — "I called a library" demonstrates integration, not design |
| | Creates schema/API lock-in to a third party |

### Option B — Build everything from scratch, including infrastructure

Write the evaluator logic *and* the embedding generation *and* the vector storage
layer independently.

| Pros | Cons |
|---|---|
| Full ownership and understanding of every layer | Re-derives infrastructure work (embeddings, ANN search) that took other teams months of tuning to get right |
| No hidden assumptions inherited from someone else's schema | Disproportionate time spent on plumbing that isn't the actual learning objective |
| Maximum flexibility for experimentation | Higher risk of subtle, hard-to-diagnose bugs with no reference implementation to check against |
| | Scope risk — a system design project can stall entirely on infrastructure before the interesting component (the evaluator's judgment logic) is even reached |

### Option C — Own the decision logic, reuse infrastructure primitives

Write the Evaluator's pre-filter, LLM-based classification prompt, scoring logic, and
conflict-check logic independently. Use an existing embedding model and vector store
as underlying infrastructure, accessed through a clean interface.

| Pros | Cons |
|---|---|
| Full ownership of the component that actually constitutes the design contribution | Requires understanding both the custom logic and enough of the underlying libraries to integrate correctly |
| Avoids reimplementing solved problems (embedding generation, ANN search) that every serious system — including Mem0 and Zep — also treats as swappable dependencies | Some setup overhead beyond Option A |
| Mirrors real-world engineering practice: logic is owned, infrastructure is borrowed | No ground truth to calibrate confidence/salience thresholds against — expect iteration |
| Enables direct comparison later ("my evaluator vs. Mem0's extraction pipeline") using shared infrastructure as a controlled variable | |

---

## 5. Decision

**Option C is adopted.**

### 5.1 Reasoning

1. **The Evaluator's decision logic is the actual subject of this system design
   project.** The value being demonstrated is the ability to reason about
   memory-worthiness, classification, scoring, and conflict resolution — not the
   ability to stand up a vector index, which is a solved, well-documented problem with
   no meaningful design decisions left to make at this scope.

2. **Option A was rejected** because adopting a library wholesale removes the design
   decision entirely — there would be nothing left to explain, defend, or iterate on
   in this component. It also locks the project to a third party's taxonomy and
   scoring assumptions, which conflicts with the goal of understanding *why* each
   decision (worthiness, type, salience) is made.

3. **Option B was rejected** because building infrastructure from scratch introduces
   effort and risk that is disproportionate to the learning objective. Every
   production framework surveyed (Mem0, Zep, Letta) treats embeddings and vector
   storage as a dependency, not as something they built in-house from first
   principles. Re-deriving that work would consume project time without adding
   design insight, and would introduce failure modes (bad embeddings, unstable
   search) that are hard to distinguish from actual evaluator logic bugs.

4. **Option C isolates the variable that matters.** By keeping infrastructure
   constant (a standard embedding model, a standard vector store) and varying only
   the evaluator's own logic, the resulting system is easier to reason about, easier
   to debug, and easier to benchmark against existing approaches like Mem0's
   extract-then-reconcile pipeline as a point of comparison.

### 5.2 What "own vs. reuse" means concretely

| Layer | Ownership |
|---|---|
| Pre-filter (cheap heuristic gate) | Built independently |
| Atomic decomposition logic | Built independently |
| Content evaluation prompt (worthiness, type, salience, sensitivity) | Built independently, using an off-the-shelf LLM API as the inference backend |
| Redundancy / conflict-check logic | Built independently |
| Decision schema / output contract | Built independently (defined in the companion data-format document) |
| Embedding generation | Reused (existing embedding model/API) |
| Vector similarity search | Reused (existing vector store — e.g. pgvector, Chroma, FAISS) |
| LLM inference | Reused (existing model API, called with a custom prompt) |

The boundary is: **anything that expresses a judgment or a decision rule is owned;
anything that is a generic computational primitive is reused.**

---

## 6. Implementation Approach

The Evaluator will be implemented as an independently testable module with a single
clear interface:

```
evaluate(message: ConversationTurn) -> list[Decision]
```

Internally, it follows the five-stage pipeline already defined for this component
(pre-filter → decompose → content evaluation → redundancy/conflict check → decision
output), with:

- The **pre-filter** implemented as local rule-based logic (no external calls).
- The **content evaluation stage** implemented as a structured-output LLM call against
  a custom prompt, validated against the JSON decision schema.
- The **redundancy/conflict check** implemented using vector similarity search against
  the existing memory store (via the reused vector store infrastructure), with the
  comparison and resolution logic (duplicate vs. conflict vs. novel) written
  independently.

This keeps the module boundary clean: the vector store and embedding model can be
swapped later without touching the Evaluator's decision logic, and the decision logic
can be tested with mocked infrastructure.

---

## 7. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| No ground-truth data to calibrate confidence/salience thresholds | Start with conservative thresholds, log all decisions, manually review a sample to tune iteratively |
| LLM-based classification may be inconsistent across similar inputs | Use structured output / schema-validated responses rather than free-text parsing; consider few-shot examples in the prompt |
| Integration bugs between custom logic and reused infrastructure | Keep the interface between the Evaluator and the vector store narrow and well-typed; test each side independently before integration |
| Time overrun from underestimating LLM prompt iteration | Timebox prompt design; fall back to Mem0's published extraction prompt structure as a reference starting point if iteration stalls |

---

## 8. Next Steps

1. Finalize the JSON schema for the Evaluator's decision output (already drafted in
   the companion data-format document).
2. Draft and iterate on the content-evaluation LLM prompt.
3. Implement the pre-filter as a standalone, unit-testable function.
4. Select and integrate the embedding model and vector store.
5. Implement and test the redundancy/conflict-check logic against a small seeded
   memory set.
6. Benchmark the completed Evaluator against Mem0's published extraction approach as
   a reference comparison point.

---

## 9. References

- CoALA: Cognitive Architectures for Language Agents — Princeton (arXiv:2309.02427)
- Generative Agents: Interactive Simulacra of Human Behavior — Park et al., Stanford (arXiv:2304.03442)
- MemGPT: Towards LLMs as Operating Systems — Packer et al. (arXiv:2310.08560)
- Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory — Chhikara et al. (arXiv:2504.19413)
- Zep: A Temporal Knowledge Graph Architecture for Agent Memory — Rasmussen et al. (arXiv:2501.13956)
