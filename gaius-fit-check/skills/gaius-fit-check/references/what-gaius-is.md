# What GAIuS Is

GAIuS is a cognitive engine: it learns sequences of events (symbolic tokens, numeric
vectors, or both) from observation, predicts what comes next, and can explain every
prediction it makes. It is not an LLM and not a neural network. There is no gradient
training and no GPU. There is no separate training phase: learning and prediction work on
the same live knowledge base, and the engine learns only when you call `learn` (or switch
automatic learning on). It runs on CPU, in Docker, and is designed to run fully offline,
with no cloud dependency.

## The core loop

Three operations, called through a Python SDK or REST API:

1. **Observe** — feed the engine one event at a time. Events accumulate in **working
   memory (WM)** in the order observed.
2. **Learn** — commit the sequence currently in working memory to the **knowledge base
   (KB)** as a stored pattern. One call; the pattern is usable on the very next
   prediction. Learning stores a new pattern without changing any other pattern's
   sequence; learning an identical sequence again adds to that pattern's count. It does
   update KB-wide symbol statistics, which feed the ranking score, so the order of other
   predictions can shift.
3. **Predict** — given the events currently in working memory, the engine matches them
   against stored patterns and returns ranked predictions. Every learned pattern reads in
   both directions: **forward** to what comes next, and **backward** to what led here —
   no separate reverse model, nothing trained backward. Reveal part of a process and the
   engine can reconstruct the rest in either direction.

## The data format (GDF)

Every observed event is a small JSON structure with four channels:

- `strings` — symbolic tokens that occurred together in this time step (words, labels,
  discrete states — e.g. `"SPEED|HIGH"`, `"ACTIVITY|WALKING"`). The primary channel for
  event and categorical data; the engine generalizes by finding tokens that repeat
  across experiences.
- `vectors` — numeric feature lists, matched by proximity: by default each vector is
  represented by the stored vectors nearest to it, so nearby vectors share those and
  partially match. The engine does not build per-class averages or prototypes (an
  optional hyperclustering mode, off by default, can replace stored vectors with
  clusters). The primary channel for numeric data — the engine's benchmarked
  classification results run on vectors (see fit-patterns.md).
- `emotives` — named scalar signals (utility, risk, cost) attached to an event, learned
  with the pattern and reported as an average on each prediction. They do not affect the
  engine's ranking; your application can re-rank or filter predictions by them.
- `metadata` — arbitrary context stored with the event but **never pattern-matched**.
  This is where identifiers (IDs, IPs, timestamps, names) belong, so they don't pollute
  learning.

**Combining channels:** both are first-class, and one event can carry both — with one
rule to know: choose the primary channel by where the signal lives, and mix only when
each channel adds *different* information. Symbolic tasks gain from adding a small
supporting vector of numeric context (used in a deployed security-monitoring
integration — it lets near-miss numeric context partially match). Vector tasks should keep `strings` empty rather than restating
the numbers as coarse tokens — duplicating a precise vector as blurry symbols costs
accuracy.

A sequence is just an ordered list of events. The canonical supervised pattern is:
observe the feature events, observe one final label event, then learn — at prediction
time the label shows up in the prediction's `future` field.

## What a prediction looks like

Predictions are structured objects, not opaque scores:

- `future` — what the engine expects to happen next (the actual predicted events).
- `past` — what the engine infers led up to the observed events. Given only a middle or
  end fragment, this reconstructs the earlier steps you never saw — same learned pattern,
  opposite direction, no reverse training involved.
- `matches` — which observed tokens matched the stored pattern.
- `missing` — tokens the pattern expected that were not observed.
- `extras` — observed tokens the pattern did not expect.
- `confidence` — 0 to 1: the share of the symbols in the pattern's aligned events that
  were observed. A match-completeness score, not a probability of being right.
- `potential` — the engine's ranking score. Predictions come back ordered by it, highest
  first. It is not a probability of being right.
- `frequency` — how many times this exact pattern has been learned.
- Several other metrics ride along with each prediction.

Describe these fields only as defined here. Never call `confidence` or `potential`
certainty, calibration, or the likelihood of being correct: whether confidence tracks
accuracy has not been characterized yet.

Every prediction traces back to specific stored patterns. You can fetch any learned
pattern and read exactly what sequence produced it — the explanation is the mechanism, not a
post-hoc approximation.

## The knowledge base is editable

Because knowledge is stored as discrete patterns rather than distributed weights, the KB
supports operations that a weight-based model cannot do directly:

- Inspect any pattern; search patterns by the symbols they contain.
- Delete one bad pattern without modifying any other pattern and without retraining.
- Correct or inject patterns programmatically.
- Blacklist noisy symbols so they stop entering working memory.
- Export the entire KB to a file and load it into another processor. Loading replaces the
  target processor's entire KB; export it first.

This matters for data governance (remove a specific learned fact on request), for
debugging (find exactly which learned pattern caused a wrong prediction and remove it),
and for bootstrapping (ship a curated KB with your product).

## Runtime tuning

Behavior is controlled by runtime parameters ("genes") you can change live, per node,
without restarting or retraining. The two you'll touch most:

- `recall_threshold` — how similar working memory must be to a stored pattern before it
  fires a prediction. Low = many lenient predictions; high = few strict ones.
- `max_predictions` — how many ranked candidates to return.

## Topologies: patterns of patterns

An agent can run multiple processing nodes wired together. Nodes can be stacked into
layered topologies.

Honest caveat: tokens at higher layers are machine-generated identifiers. Decoding what a
higher-layer pattern *means* requires a back-tracing step through the SDK, not just a
glance. Explainability is built-in but at higher layers it takes work.

## ThinkFlux

A separate tier, ThinkFlux, adds planning and concept discovery. [Contact us to learn more](https://www.gaius.fyi/about#contact).

## Boundaries to respect

- **Symbolic matching is exact.** Tokens match or they don't — no semantic similarity
  between `"SPEED|HIGH"` and `"velocity_fast"`. Keep preprocessing consistent between
  learning and prediction (vectors, by contrast, match by proximity).
- **The engine models order, not elapsed time.** "A then B, 50 ms apart" and "A then B,
  six hours apart" are the same sequence. When elapsed time carries signal, encode it as
  symbols — bucket the gap and observe it as an event token (`"GAP|SHORT"`,
  `"GAP|OVERNIGHT"`). Time-dependence that isn't encoded this way is invisible to the
  engine.
- **Emotives operate at the layer where they're observed** — they don't currently
  propagate up through abstraction layers.
- **Isolation is architectural.** Clients of one node share its memory; multi-tenant
  designs use separate nodes or instances per tenant.
- **Topology is set at startup.** Nodes and wiring come from the genome file; plan the
  topology, then run it.

## Deployment shape

Docker Compose stack (engine nodes plus API server), driven by the
GAIuS Python SDK or plain REST. CPU-only. Designed to run on a laptop, a server, or
an air-gapped network. Commercially licensed — deployment and licensing details come from a
conversation, not this document.
