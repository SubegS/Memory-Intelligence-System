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
