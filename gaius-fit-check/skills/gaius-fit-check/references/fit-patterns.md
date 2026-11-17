# Where GAIuS Fits

The single question that predicts fit better than any other:

> **Does your system produce events, situations, or feature snapshots that recur in a
> recognizable way?** If yes, GAIuS can learn them and tell you exactly why it predicted
> what it predicted.

## The headline result: benchmarked accuracy

On the UCI Human Activity Recognition benchmark (6 activities), a five-node GAIuS agent (a
specialist configuration) — built with standard SDK calls, CPU-only, no GPU — scored
**92.4%** (mean of three seeds) on the official test split. Every method was given **the
same 150 features**, chosen by a random-forest ranking, and the same train/test split:

| Method                 | Accuracy  |
|------------------------|-----------|
| GAIuS (5 nodes)        | **92.4%** |
| LogisticRegression     | 91.2%     |
| GradientBoosting       | 91.2%     |
| XGBoost                | 91.1%     |
| SVM (RBF)              | 91.0%     |
| LightGBM               | 90.3%     |
| MLP                    | 89.6%     |
| KNN (k=10)             | 88.9%     |
| RandomForest           | 88.7%     |
| KNN (k=5)              | 88.6%     |

Caveats, to quote along with the number:

- Routing between the GAIuS nodes ran in the test harness, and a single node scores 87–88%.
- The configuration was tuned against the published test split, with no separate
  validation set.
- Given all 2,805 features instead of 150, classical models score higher: XGBoost 95.05%,
  RandomForest 93.3%. The ranking above holds only at the same 150 features.
- **Do not claim data efficiency** (high accuracy from a small fraction of the training
  data). The benchmark does not support it: on equal training data, a single GAIuS node
  scores below XGBoost at every training size tested.

The architecture behind the number is itself a selling point: it was built entirely with
application-level code on the public SDK — standard calls, no engine modifications, no
GPU. The same observe-then-label pattern drives the digit demo below.

---

## Strong-fit patterns

### 1. Learning without a training run

No minimum dataset size, no training run, no convergence. A pattern observed once is
usable on the next prediction. This is about *when* knowledge becomes usable, not about
accuracy from few examples: the benchmark above does not show GAIuS reaching higher
accuracy than standard models on less data. If your reality is "new labeled examples keep
arriving and must count immediately, without a retrain" — cold-start, rare events,
analyst feedback — this is the fit.

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
  events or decisions where new labels must count immediately, and map those to patterns 1–8 with the concrete
  gain spelled out.
- Ground every claim in this file or its siblings; quote the benchmark numbers as
  stated — caveats included.
- Call out genuine "don't use it for this" cases whenever they apply — the assessment
  is only useful if the no-gos are as real as the yeses.
- Mixed verdicts are normal: "strong fit here, wrong tool there" is often the accurate
  answer.
