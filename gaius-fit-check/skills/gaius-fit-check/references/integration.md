# What Adopting GAIuS Looks Like

For judging fit: this is the shape and size of the integration work, not a setup guide.

## The developer surface

**Python SDK** (`ia-sdk`, package `ia.gaius`) is the primary interface:

```python
from ia.gaius.agent_client import AgentClient

agent = AgentClient(agent_info)          # host/key config
agent.connect()
agent.set_ingress_nodes(["P1"])          # where observations go
agent.set_query_nodes(["P1"])            # where predictions come from

# The canonical supervised pattern
agent.clear_wm()                                          # clean slate
agent.observe({"strings": [], "vectors": [features], "emotives": {}})
agent.observe({"strings": [f"label|{label}"], "vectors": [], "emotives": {}})
agent.learn()                                             # commit to KB

# Predict: observe features only, read the label from the future field
agent.clear_wm()
agent.observe({"strings": [], "vectors": [features], "emotives": {}})
predictions = agent.get_predictions()
predicted = predictions[0]["future"]     # e.g. [["label|setosa"]]
```

Everything the SDK does is also plain **REST** (JSON over HTTP), so non-Python stacks
integrate directly. The API covers the core loop, runtime parameter tuning, knowledge-base
inspection/editing/export, and node state control.

**Deployment**: a Docker Compose stack (engine nodes plus API server) on CPU-only
hardware, cloud or fully offline. Commercially licensed; a license conversation is
part of getting started.

## Where the real work is: representation

Integration effort is dominated by one task — expressing your data in GDF so that similar
situations look similar to the engine. Two proven recipes, by data type:

- **Numeric features → vectors, curated.** Select the features that carry signal rather
  than sending everything — a feature-selection pass pays for itself — and send them as
  vectors. Windowed/temporal data benefits from stacking a few consecutive windows into
  one vector.
- **Events and categories → symbolic tokens.** `key|value` strings (`STAGE|RECON`,
  `SPEED|HIGH`) that repeat across experiences. Put identifiers (IDs, IPs, timestamps)
  in `metadata`, where they're stored but never pattern-matched, so unique values don't
  dilute learning.
- **Mixing the two:** legitimate and production-proven, with one rule — mix only when
  each channel adds *different* information (e.g. symbolic event tokens plus a small
  vector of numeric context). Don't restate the same signal in both channels: coarse
  tokens duplicating a precise vector cost accuracy. Assess both paths for any dataset;
  pick the primary channel by where the signal lives.

Two disciplines that prevent the classic silent failures:

- **Order and hygiene**: observe features, then the label last; call `learn()` *before*
  clearing working memory; clear between sequences. Getting this wrong doesn't error —
  it silently learns nothing. And working memory is unbounded by default: a long-running
  process that observes continuously without learning or clearing accumulates
  indefinitely, so build the learn-or-clear cadence into the integration.
- **Format stability**: the representation at prediction time must match training —
  same token vocabulary, same vector dimensionality. Version your preprocessing.

Budget rough effort accordingly: naturally symbolic event data (log events, workflow
states, categorical features) is a thin mapping layer — hours to days. Continuous or
high-dimensional sources need the feature-curation pass and validation loop — days to a
couple of weeks, iterating against prediction quality.

## Evaluating whether it's working

- The SDK ships **PVT (Performance Validation Testing)**, a harness for systematic
  evaluation across classification and utility-prediction test types.
- Prediction internals are the debugging tool: a large `missing` list means a prediction
  fired on a partial match. That is normal for vector data (in the shipped digit demo,
  correct calls often share only 1 of 4 vector features with the matched pattern), but if
  partial matches are noise for your data, raise `recall_threshold` (higher = stricter,
  fewer and more complete matches); predictions collapsing to one label means the
  data shaping is producing nearly-unique fingerprints (fix the data, not the knobs).
- Nineteen step-by-step tutorial notebooks ship with the product: eleven on the core
  engine, from hello-world through sequences, topologies, emotives, classification,
  utility prediction, recommendations, and vision, then eight on ThinkFlux and planning.
  They double as executable documentation of every pattern in these files.

## Operational realities

- **Knowledge persistence**: the KB supports export/import to file; snapshot before
  destructive operations. Knowledge bases are compact and fast to move: a real corpus
  of ~19,000 learned patterns exports to ~25 MB of JSON in about 2 seconds and restores
  in about 4, end-to-end over HTTP on laptop-class hardware (single-run,
  order-of-magnitude figures). Clearing all memory on a node affects every client of
  that node — treat it like dropping a database table.
- **Recovery**: a cold stack goes from container start to a warm agent with its full
  KB restored in about a minute — dominated by container startup, not KB volume.
- **Isolation**: one node = one shared memory space. Separate tenants/experiments by
  node or by instance, and design that in from the start.
- **Latency**: end-to-end over HTTP, retrieving predictions is cheap — single-digit to
  tens of milliseconds. `observe` is the dominant cost, and it grows with KB size
  (sublinearly): single-digit milliseconds on small KBs, ~0.6 s per event on a real
  ~19,000-pattern node — roughly 1–2 observations/second per node single-threaded, which
  is the number to design throughput against. Cost is flat in event width and in
  working-memory depth, so long sequences are not a latency problem. The engine is
  CPU-bound and degrades sharply when the host is saturated — don't co-locate heavy
  batch work with latency-sensitive inference. (Single-run, order-of-magnitude
  figures.)
- **Scale**: the engine has sustained multi-million-call production workloads. Very
  large knowledge bases make full KB export slow, and per-observe cost grows with KB
  size — high-entropy token design (the fingerprint failure) is what actually blows up
  KB size, so data shaping is also the capacity plan.
