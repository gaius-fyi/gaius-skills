# Where GAIuS Fits

The single question that predicts fit better than any other:

> **Does your system produce events, situations, or feature snapshots that recur in a
> recognizable way?** If yes, GAIuS can learn them — from remarkably few examples — and
> tell you exactly why it predicted what it predicted.

## The headline result: benchmarked data efficiency

On the UCI Human Activity Recognition benchmark (561 smartphone-sensor features, 6
activities), a GAIuS classification architecture — built entirely with standard SDK calls,
CPU-only, no GPU — was benchmarked head-to-head against RandomForest, KNN, and XGBoost
on **identical features, identical train/test splits, 3 seeds**:

| Training data     | GAIuS     | XGBoost | RandomForest | KNN   |
|-------------------|-----------|---------|--------------|-------|
| 10% (726 samples) | **91.7%** | 88.4%   | 88.2%        | 87.8% |
| 20% (1,453)       | **92.2%** | 90.5%   | 89.0%        | 87.9% |
| 40% (2,907)       | **92.4%** | 91.3%   | 88.6%        | 88.3% |
| 100% (7,268)      | **92.4%** | 91.0%   | 88.7%        | 88.6% |

Two things to notice:

- **GAIuS wins at every training size** — best overall accuracy 92.4%.
- **GAIuS with 10% of the data (91.7%) beats every baseline trained on 100% of it.**
  It reaches within a point of its own ceiling from 726 examples, because it learns each
  pattern from a single exposure rather than fitting a statistical surface.

(Honest footnote: heavyweight deep-learning models can score higher on this benchmark —
with GPUs, training runs, and none of the explainability. Against the classical ML a
production team would actually reach for, GAIuS won outright.)

The architecture behind the number is itself a selling point: it was built entirely with
application-level code on the public SDK — standard calls, no engine modifications, no
GPU — and the approach transfers to other classification tasks.

---

## Strong-fit patterns

### 1. Learning from scarce data — the signature strength

No minimum dataset size, no training run, no convergence. A pattern observed once is
usable on the next prediction. The table above is the proof: near-ceiling accuracy from
hundreds of examples, not thousands. If your reality is "we have dozens or hundreds of
labeled examples, not millions" — new product, rare events, expensive labels, cold-start —
this inverts the usual ML calculus, and it's where conventional models are weakest.

### 2. Next-event prediction over event streams

Workflows, protocol progressions, user journeys, alarm cascades, attack-stage sequences —
anywhere events arrive in meaningful order and "what typically comes next?" changes what
your system does. This is the engine's home turf and its strongest shipped pattern: in a
production integration, a layered topology learned multi-step stage progressions and
predicted the next stage of in-progress sequences with maximum confidence after seeing
each pattern only a handful of times — and every prediction itemized which signals
matched and what was expected next.

### 3. Backward reconstruction — inferring what led to a state

The same capability as next-event prediction, aimed the other way, and it's one no
autoregressive model (LLM, RNN, forecaster) can match: those run forward only, so to go
backward you'd train a second model on reversed data. In GAIuS, "what comes next" and
"what led here" come from the same learned pattern — the `past` field falls out for
free, no second model. This is the shape of a whole class of
problems: **incident and root-cause forensics** (you're paged at the failure, not the
trigger — reconstruct the sequence that produced it), **audit and compliance** (given the
outcome, which steps *should* have happened? a skipped step surfaces directly in the
`missing` field), and **gap-filling traces or telemetry** (dropped spans, sampled metrics —
infer the steps between the ones you kept, in either direction). If the developer routinely
gets handed the end of a story and has to work out the beginning, this is the fit.

### 4. Classification — numeric or symbolic features

Feature-event → label-event, learn, predict. The HAR result above is this
pattern on numeric sensor vectors; a shipped image-digit demo does it on pixel vectors;
production integrations do it on symbolic event tokens. Works from the first labeled
example, improves with each one, never needs a retrain to incorporate the next.

### 5. Explainability and auditability as requirements

Predictions decompose by construction: which stored patterns fired, which observed
signals matched, what was expected but missing. If a regulator, analyst, or customer has
to trust the output, the explanation is the mechanism itself, not a post-hoc
approximation bolted onto a black box. And because knowledge lives in discrete, editable
patterns, you can inspect, correct, or **delete any specific learned pattern on demand** —
data governance and error correction without retraining anything.

### 6. Continuous learning in production

Observe → learn → live. New patterns learned this minute predict this minute. Systems
that must adapt — analyst feedback loops, evolving behavior, weekly-new patterns — get
adaptation without a retrain-and-redeploy pipeline. You can also pre-seed: curate a
starter knowledge base offline, ship it with the product, keep learning on top.

### 7. Edge, air-gapped, and CPU-only environments

Everything above runs in Docker on commodity CPUs, fully offline. No GPU, no cloud API,
no data leaving the premises. The HAR benchmark above ran on exactly this footprint. If
your deployment can't ship data out or power a GPU, this constraint alone eliminates
most alternatives.

### 8. Utility-weighted prediction and recommendations

Emotives attach scalar signals (utility, risk, cost) to events, and predictions carry
them — so you can rank candidate futures by expected value, not just likelihood. The
prediction's `missing` field (expected-but-not-present) drives shipped
recommendation-style use cases: "people in this pattern usually also have X."

### 9. Deciding, not just predicting — planning and concept discovery

The ThinkFlux tier (see what-gaius-is.md) extends predictions into action: bootstrap the
agent's learned patterns into named concepts, categorize new instances against them, and
run goal-directed planning — assigning assets/actions to targets/goals with
utility-weighted trade-offs — in a live observe → plan → act loop. It also does
**unsupervised concept discovery**: given unlabeled observations, it clusters
co-occurring symbols into concepts and generates hypotheses describing them. If the
developer says "we don't even know what categories exist in our data" or "we need the
system to choose an action, not just forecast," this is the fit — it's the deepest part
of the shipped tutorial curriculum (8 of 19 tutorials, ending in an applied planning
scenario).

---

## Where to point them elsewhere

A few clear no-gos, stated plainly.

- **Generative and language tasks.** GAIuS doesn't write text, summarize, converse, or
  do semantic similarity. Pair it with an LLM if you need language; it won't replace one.
- **Regressing continuous magnitudes.** Predicting exact prices, volumes, or sensor
  values is not the tool's shape. Reframing to discrete states (LOW/MED/HIGH regime,
  RISING/FALLING trend) works well in practice — but if only the exact number will do,
  use a regression model.
- **Raw perception at scale.** Real-world imagery, audio, video need a perception model
  in front; GAIuS then reasons over its symbolic outputs.
- **Zero-engineering expectations.** GAIuS rewards representation work (see
  integration.md) — it's a power tool for a team that engages with its data, not a
  drop-in AutoML endpoint. Similarly, multi-tenant isolation is an architecture you
  design (node- or instance-per-tenant), not a checkbox.

---

## Calibration notes for the assessor

- Lead with the opportunities. Find where the developer's system produces recurring
  events or scarce-labeled decisions, and map those to patterns 1–9 with the concrete
  gain spelled out.
- Ground every claim in this file or its siblings; quote the benchmark numbers as
  stated — caveats included.
- Call out genuine "don't use it for this" cases whenever they apply — the assessment
  is only useful if the no-gos are as real as the yeses.
- Mixed verdicts are normal: "strong fit here, wrong tool there" is often the accurate
  answer.
