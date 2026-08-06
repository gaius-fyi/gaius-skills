# What GAIuS Is

GAIuS is a cognitive engine: it learns symbolic sequences from observation, predicts what
comes next, and can explain every prediction it makes. It is not an LLM and not a neural
network. There is no gradient training, no GPU, and no training/inference split — learning
and predicting interleave continuously on CPU, in Docker, fully offline if needed.

## The core loop

Three operations, called through a Python SDK or REST API:

1. **Observe** — feed the engine one event at a time. Events accumulate in **working
   memory (WM)** in the order observed.
2. **Learn** — commit the sequence currently in working memory to the **knowledge base
   (KB)** as a stored model. One call, immediate; the pattern is usable on the very next
   prediction. There is no retraining cycle — learning one new pattern does not touch the
   others.
3. **Predict** — given the events currently in working memory, the engine matches them
   against stored models and returns ranked predictions. Every learned pattern reads in
   both directions: **forward** to what comes next, and **backward** to what led here —
   no separate reverse model, nothing trained backward. Reveal any part of a process and
   the engine reconstructs the rest either way.

## The data format (GDF)

Every observed event is a small JSON structure with four channels:

- `strings` — symbolic tokens that occurred together in this time step (words, labels,
  discrete states — e.g. `"SPEED|HIGH"`, `"ACTIVITY|WALKING"`). The primary channel for
  event and categorical data; the engine generalizes by finding tokens that repeat
  across experiences.
- `vectors` — numeric feature lists, matched geometrically (nearby vectors partially
  match). The primary channel for numeric data — the engine's benchmark-winning
  classification results run on vectors (see fit-patterns.md).
- `emotives` — named scalar signals (utility, risk, cost) attached to the event. They
  mix into predictions, which enables utility-weighted prediction and recommendation-style
  ranking.
- `metadata` — arbitrary context stored with the event but **never pattern-matched**.
  This is where identifiers (IDs, IPs, timestamps, names) belong, so they don't pollute
  learning.

**Combining channels:** both are first-class, and one event can carry both — with one
rule to know: choose the primary channel by where the signal lives, and mix only when
each channel adds *different* information. Symbolic tasks gain from adding a small
supporting vector of numeric context (proven in production — it lets near-miss numeric
context partially match). Vector tasks should keep `strings` empty rather than restating
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
- `matches` — which observed tokens matched the stored model.
- `missing` — tokens the model expected that were not observed.
- `extras` — observed tokens the model did not expect.
- Scoring metadata — `confidence`, `frequency` (how often this pattern has been seen),
  and several other ranking metrics.

Every prediction traces back to specific stored models. You can fetch any learned model
and read exactly what sequence produced it — the explanation is the mechanism, not a
post-hoc approximation.

## The knowledge base is editable

Because knowledge is stored as discrete models rather than distributed weights, the KB
supports surgical operations no gradient-trained model can offer:

- Inspect any model; search models by the symbols they contain.
- Delete one bad model without touching anything else and without retraining.
- Correct or inject models programmatically.
- Blacklist noisy symbols so they never enter working memory again.
- Export the entire KB to a file and restore it elsewhere.

This matters for data governance (remove a specific learned fact on request), for
debugging (find exactly which learned pattern caused a wrong prediction and remove it),
and for bootstrapping (ship a curated KB with your product).

## Runtime tuning

Behavior is controlled by runtime parameters ("genes") you can change live, per node,
without restarting or retraining. The two you'll touch most:

- `recall_threshold` — how similar working memory must be to a stored model before it
  fires a prediction. Low = many lenient predictions; high = few strict ones.
- `max_predictions` — how many ranked candidates to return.

## Topologies: patterns of patterns

An agent can run multiple processing nodes wired together. In a layered topology, a
higher node learns patterns over a lower node's *prediction outputs* — e.g. a lower node
classifies individual events into stages, and a higher node learns the sequence of stages
and predicts which stage comes next. This is the engine's strongest shipped pattern —
production deployments use it for multi-step progression prediction.

Honest caveat: tokens at higher layers are machine-generated identifiers. Decoding what a
higher-layer pattern *means* requires a back-tracing step through the SDK, not just a
glance. Explainability is built-in but at higher layers it takes work.

## The reasoning tier above: ThinkFlux

ThinkFlux is an orchestration layer that sits on top of GAIuS agents and turns learned
patterns into higher-level reasoning:

- **Concept bootstrapping** — automatically distill the patterns an agent has learned
  into named concepts usable for categorizing new instances and matching rules.
- **Unsupervised concept discovery** — find the categories in *unlabeled* data and
  generate hypotheses (discrete values and numeric ranges) describing each discovered
  concept. Use it when you don't yet know what kinds of things exist in your data.
- **Goal-directed planning** — assign assets to targets / actions to goals over those
  concepts, with mood-weighted emotive signals (utility, risk) steering the decisions,
  and a live loop of observe → plan → act.

This tier is covered by eight of the nineteen shipped tutorials, culminating in an
applied planning scenario. If the developer's need is "decide what to do," not just
"predict what's next," this is the relevant surface.

## Boundaries to respect

- **Symbolic matching is exact.** Tokens match or they don't — no semantic similarity
  between `"SPEED|HIGH"` and `"velocity_fast"`. Keep preprocessing consistent between
  learning and prediction (vectors, by contrast, match by proximity).
- **The engine models order, not elapsed time.** "A then B, 50 ms apart" and "A then B,
  six hours apart" are the same sequence. When elapsed time carries signal, encode it as
  symbols — bucket the gap and observe it as an event token (`"GAP|SHORT"`,
  `"GAP|OVERNIGHT"`). Time-dependence that isn't encoded this way is invisible to the
  model.
- **Emotives operate at the layer where they're observed** — they don't currently
  propagate up through abstraction layers.
- **Isolation is architectural.** Clients of one node share its memory; multi-tenant
  designs use separate nodes or instances per tenant.
- **Topology is set at startup.** Nodes and wiring come from the genome file; plan the
  topology, then run it.

## Deployment shape

Docker Compose stack (engine nodes plus API server), driven by a
Python SDK (`ia-sdk`) or plain REST. CPU-only. Runs on a laptop, a server, or an
air-gapped network. Commercially licensed — deployment and licensing details come from a
conversation, not this document.
