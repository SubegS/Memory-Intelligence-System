# Future Design Opportunity: Consolidation-Based Memory Abstraction

Right now, the Evaluator takes the straightforward route — it classifies everything
into episodic, semantic, or procedural memory in real time, mostly because that's the
simpler thing to build and reason about while getting the system off the ground. But a
few papers, PlugMem (arXiv:2603.03296) in particular, point toward a more thoughtful
way of structuring this: semantic and procedural memory aren't really independent
categories sitting alongside episodic memory — they're built out of it. A fact or a
habit doesn't appear out of nowhere; it emerges once enough individual episodic
moments start showing the same pattern. Taking that seriously as a future direction,
the Evaluator could stop trying to sort everything into three buckets the moment it
arrives, and instead let episodic memory be the one true entry point, with semantic
and procedural memory forming later through a separate, periodic consolidation step.
Beyond being closer to how memory abstraction actually seems to work, this would also
sidestep a real problem with the current approach — forcing a snap judgment on every
single input tends to produce misclassifications that a slower, pattern-based process
wouldn't.

In practice, this wouldn't be a huge structural overhaul. The real-time Evaluator
would simply write everything to episodic memory at ingestion time, tagged with the
entities, topics, and timestamps it already extracts today. Then, on its own
schedule, a separate consolidation job would comb back through episodic memory
looking for clusters of related entries — grouped by shared entity, topic, or
embeddings that are similar enough to count as "the same idea coming up again" — and
hand each cluster to an LLM with a prompt built for summarizing and abstracting. If
the pattern looks like a recurring fact or preference, it becomes a new semantic
memory record; if it looks like a repeated behavior, it becomes a procedural one —
either way, carrying a `derived_from` field pointing back to the episodic entries it
came from. Semantic and procedural memory stop being things the system writes to
directly, and become more like views that get computed over episodic memory. And
because the consolidation job runs on its own time — nightly, or after some number of
new episodic entries pile up — none of this slows down the actual real-time ingestion
path.

# Future Work: Memory Decay & Self-Organization

**Status:** Not planned for v1. Revisit after Evaluator, Storage, and Retrieval are working end-to-end.

## Concept

Two ideas worth adding later, both about making stored memories smarter over time instead of static:

1. **Forgetting curve decay (MemoryBank)** — instead of a flat recency score, decay each memory's relevance based on time elapsed *and* its salience. Important memories decay slower, trivial ones fade faster. Practically: add a decay function on top of the existing `salience` + `last_accessed_at` fields at retrieval-ranking time — no schema change needed, just a scoring function.

2. **Self-linking memory notes (A-Mem / Zettelkasten)** — instead of storing memories as flat, independent rows, link each new memory to related past memories (found via embedding similarity), and let related old memories get their descriptions updated when relevant new context arrives. Reported to meaningfully improve multi-hop reasoning and cut retrieval token usage.

## Why later, not now

Both require a working Storage + Retrieval pipeline first — there's nothing to decay or link until memories are actually being stored and read back. Also adds real complexity (bidirectional links, rewrite logic) that isn't worth it before the core pipeline is proven.

## When to revisit

Once basic retrieval works and you notice: (a) old irrelevant memories crowding out useful recent ones → build decay, or (b) related facts scattered across memories with no connection between them → build linking.

## References

- MemoryBank (forgetting curve): https://arxiv.org/pdf/2603.18718
- A-Mem (Zettelkasten-style memory): https://www.alphaxiv.org/overview/2502.12110 , https://arxiv.org/pdf/2601.07582
- Reported results: https://www.emergentmind.com/topics/agentic-memory-framework
