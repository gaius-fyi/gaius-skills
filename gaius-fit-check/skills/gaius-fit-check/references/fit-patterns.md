# Where GAIuS Fits

The single question that predicts fit better than any other:

> **Does your system produce events, situations, or feature snapshots that recur in a
> recognizable way?** If yes, GAIuS can learn them and tell you exactly why it predicted
> what it predicted.

## The headline result: benchmarked data efficiency

On the UCI Human Activity Recognition benchmark (6 activities), a five-node GAIuS agent (a
specialist configuration) — built with standard SDK calls, CPU-only, no GPU — was
benchmarked head-to-head against RandomForest, KNN, and XGBoost on **the same 150
features, identical train/test splits, 3 seeds**:

| Training data     | GAIuS     | XGBoost | RandomForest | KNN   |
|-------------------|-----------|---------|--------------|-------|
| 10% (726 samples) | **91.7%** | 88.4%   | 88.2%        | 87.8% |
| 20% (1,453)       | **92.2%** | 90.5%   | 89.0%        | 87.9% |
| 40% (2,907)       | **92.4%** | 91.3%   | 88.6%        | 88.3% |
| 100% (7,268)      | **92.4%** | 91.0%   | 88.7%        | 88.6% |

Every method was given the same 150 features, chosen by a random-forest ranking. Routing
between the GAIuS nodes ran in the test harness, and a single node scores 87–88%.

Two things to notice:

- **On the same features and splits, GAIuS beat XGBoost, RandomForest and KNN at every
  training size** — 92.4% with all the training data (mean of three seeds).
- **GAIuS with 10% of the data (91.7%) beats every baseline trained on 100% of it.**
  It reaches within a point of its own ceiling from 726 examples.

(Footnote: heavyweight deep-learning models — and classical models given all 2,805
features — can score higher on this benchmark. With all 2,805 features, XGBoost scores
95.05% and RandomForest 93.3%. The comparison above holds at the same 150 features.)

The architecture behind the number is itself a selling point: it was built entirely with
application-level code on the public SDK — standard calls, no engine modifications, no
GPU. The same observe-then-label pattern drives the digit demo below.

---

## Strong-fit patterns

### 1. Learning from scarce data — the signature strength

No minimum dataset size, no training run, no convergence. A pattern observed once is
usable on the next prediction. The table above is the proof: near-ceiling accuracy from
hundreds of examples, not thousands. If your reality is "we have dozens or hundreds of
labeled examples, not millions" — new product, rare events, expensive labels, cold-start —
this is the fit.

### 2. Next-event prediction over event streams

Workflows, protocol progressions, user journeys, alarm cascades, attack-stage sequences —
anywhere events arrive in meaningful order and "what typically comes next?" changes what
your system does. This is the engine's home turf: every prediction itemizes which signals
matched and what is expected next. Nodes can be stacked into layered topologies.

### 3. Backward reconstruction — inferring what led to a state

The same capability as next-event prediction, aimed the other way. A forward-only
predictor (an autoregressive language model, a next-step forecaster) would need a second
model trained on reversed data to do this. In GAIuS, "what comes next" and "what led here"
come from the same learned pattern: the `past` field comes with every prediction. This is
the shape of a whole class of
problems: **incident and root-cause forensics** (you're paged at the failure, not the
trigger — reconstruct the sequence that produced it), **audit and compliance** (given the
outcome, which steps *should* have happened? a skipped step surfaces directly in the
`missing` field), and **gap-filling traces or telemetry** (dropped spans, sampled metrics —
infer the steps between the ones you kept, in either direction). If the developer routinely
gets handed the end of a story and has to work out the beginning, this is the fit.

### 4. Classification — numeric or symbolic features

Feature-event → label-event, learn, predict. The HAR result above is this
pattern on numeric sensor vectors; a shipped image-digit demo does it on pixel vectors
(96.7%, 348 of 360 held-out handwritten digits, learned from 1,437 examples, each stored
as its own pattern and matched individually rather than averaged into class shapes);
a deployed security-monitoring integration does it on symbolic event tokens. Works from
the first labeled example and incorporates each new example without a retraining run, as
long as the representation stays the same (see integration.md, format stability).

### 5. Explainability and auditability as requirements

Predictions decompose by construction: which stored patterns fired, which observed
signals matched, what was expected but missing. If a regulator, analyst, or customer has
to trust the output, the explanation is the mechanism itself, not a post-hoc
approximation bolted onto a black box. And because knowledge lives in discrete, editable
patterns, you can inspect, correct, or **delete any specific learned pattern on demand** —
data governance and error correction without retraining anything.

### 6. Continuous learning in production

Observe → learn → live. A newly learned pattern is usable on the next prediction. Systems
that must adapt — analyst feedback loops, evolving behavior, weekly-new patterns — get
adaptation without a retrain-and-redeploy pipeline. You can also pre-seed: curate a
starter knowledge base offline, ship it with the product, keep learning on top.

### 7. Edge, air-gapped, and CPU-only environments

The engine runs in Docker on commodity CPUs with no GPU. It is designed to run fully
offline, with no cloud dependency. The HAR benchmark above ran CPU-only. If your
deployment cannot ship data out or power a GPU, that narrows the field of alternatives
considerably.

### 8. Predictions that carry utility, risk and cost

Emotives are named scalar signals (utility, risk, cost) attached to an event, learned with
the pattern and reported as an average on each prediction. They do not affect the
engine's ranking; your application can re-rank or filter predictions by them — for
example, to rank candidate futures by expected value, not just by how well they match.

A separate tier, ThinkFlux, adds planning and concept discovery. [Contact us to learn more](https://www.gaius.fyi/about#contact).

---

## Where to point them elsewhere

A few clear no-gos, stated plainly.

- **Generative and language tasks.** GAIuS doesn't write text, summarize, converse, or
  do semantic similarity. Pair it with an LLM if you need language; it won't replace one.
- **Regressing continuous magnitudes.** Predicting exact prices, volumes, or sensor
  values is not the tool's shape. Reframing to discrete states (LOW/MED/HIGH regime,
  RISING/FALLING trend) is the way to use GAIuS here, when the decision only needs the
  state — but if only the exact number will do, use a regression model.
- **Raw perception at scale.** Real-world imagery, audio, video need a perception model
  in front; GAIuS then reasons over its symbolic outputs.
- **Zero-engineering expectations.** GAIuS rewards representation work (see
  integration.md) — it's a power tool for a team that engages with its data, not a
  drop-in AutoML endpoint. Similarly, multi-tenant isolation is an architecture you
  design (node- or instance-per-tenant), not a checkbox.

---

## Calibration notes for the assessor

- Lead with the opportunities. Find where the developer's system produces recurring
  events or scarce-labeled decisions, and map those to patterns 1–8 with the concrete
  gain spelled out.
- Ground every claim in this file or its siblings; quote the benchmark numbers as
  stated — caveats included.
- Call out genuine "don't use it for this" cases whenever they apply — the assessment
  is only useful if the no-gos are as real as the yeses.
- Mixed verdicts are normal: "strong fit here, wrong tool there" is often the accurate
  answer.
