---
name: derivation-python
description: Write Estuary derivations in Python — async-generator transforms, Pydantic-typed source/output, optional persistent state, and dependency installation via uv. Use for ML/embeddings, async API calls, or any transform where SQL and TypeScript are awkward. Derivations add complexity and cost (and Python derivations require a private/BYOC data plane) — confirm the user wants a Python derivation before reaching for this. Use when user says "python derivation", "write a derivation in python", "embed with a python derivation", "ML derivation", "call an API from a derivation", or anything mentioning model2vec / huggingface / pydantic in a derivation context.
---

# derivation-python

Estuary derivation implemented as a Python class with async-generator transforms. Runtime is `uv`, types come from Pydantic models generated at `flowctl generate` time, and the whole thing is validated in pyright strict mode at publish.

**Prereq:** read `derivation-basics` first for concepts, project layout, stateless-vs-stateful, and the universal workflow.

**Hard constraint:** Python derivations run on **private or BYOC data planes only** — never on shared public planes. Reach out to your Estuary account manager if you're interested in a private or BYOC data plane.

**Docs:** [concept page](https://docs.estuary.dev/concepts/derivations/#python) · [walkthrough](https://docs.estuary.dev/guides/transform_data_using_python/) · [upstream examples](https://github.com/estuary/flow/tree/master/examples/derive-patterns) (`stateful.flow.py`, `pipeline.flow.py`).

## When to reach for Python (vs SQL / TypeScript)

- **ML / embeddings / tokenisers** — `model2vec`, `fastembed`, `transformers`, `tiktoken`
- **Async I/O** — per-document `await httpx.get(...)`, external APIs with bounded concurrency
- **Heavy Python ecosystem deps** — `pandas`, `pyarrow`, custom parsers
- **Rich type safety** — Pydantic validators; pyright-strict at publish

Reach for other skills when plain filter/transform fits SQL (`derivation-filter-transform`), the logic is reducible (`derivation-aggregate-metrics`), or you want stateful logic without a private DP (`derivation-stateful-logic`, SQLite).

## How it works

You declare `derive.using.python.module: <file>.flow.py`. `flowctl generate` introspects your collection schemas and emits `IDerivation`, `Document`, and `Request` symbols you import. Your `Derivation(IDerivation)` class has one **async generator method per transform** — named as snake_case of the YAML `transforms[].name`. Each transform method consumes `Request.Read<TransformName>` (typed Pydantic `read.doc`) and `yield`s `Document(...)` values. Dependencies in `derive.using.python.dependencies` are resolved by uv at publish time and at runtime. Optional state is persisted across restarts via `__init__` / `start_commit` / `reset` lifecycle methods.

## Canonical example — stateless

Rule-based topic classifier over a text stream. Real code from an audience-submission demo; minimal shape, 1:1 input→output.

### Project layout

```
enrich/
├── flow.yaml
└── enrich.flow.py
```

### `flow.yaml`

```yaml
collections:
  acmeCo/classified/submissions:
    key: [/id]
    writeSchema:
      type: object
      required: [id, created_at, text, category]
      properties:
        id:         { type: string, format: uuid }
        created_at: { type: string, format: date-time }
        text:       { type: string }
        category:
          type: string
          enum: [ai-agents, fraud-risk, revenue, observability, other]
        keywords:
          type: array
          items: { type: string }
          default: []
    readSchema:
      allOf:
        - $ref: flow://write-schema
        - $ref: flow://inferred-schema

    derive:
      using:
        python:
          module: enrich.flow.py
      transforms:
        - name: enrich
          source: acmeCo/source/submissions
          shuffle: any
```

### `enrich.flow.py`

```python
"""Rule-based category bucketing + keyword extraction (stateless, 1:1)."""

import re
from collections.abc import AsyncIterator
from typing import Literal

from acmeCo.classified.submissions import IDerivation, Document, Request

Category = Literal["ai-agents", "fraud-risk", "revenue", "observability", "other"]

CATEGORIES: list[tuple[Category, set[str]]] = [
    ("ai-agents",    {"agent", "llm", "copilot", "rag", "prompt"}),
    ("fraud-risk",   {"fraud", "risk", "anomaly", "abuse"}),
    ("revenue",      {"revenue", "churn", "conversion", "upsell"}),
    ("observability", {"alert", "slo", "incident", "paging"}),
]
STOPWORDS = {"the", "and", "for", "that", "with", "you", "our"}
TOKEN_RE = re.compile(r"[a-zA-Z][a-zA-Z\-]{2,}")


def classify(tokens: set[str]) -> Category:
    for name, triggers in CATEGORIES:
        if tokens & triggers:
            return name
    return "other"


class Derivation(IDerivation):
    async def enrich(self, read: Request.ReadEnrich) -> AsyncIterator[Document]:
        src = read.doc
        tokens = TOKEN_RE.findall(src.text.lower())
        yield Document(
            id=src.id,
            created_at=src.created_at,
            text=src.text,
            category=classify(set(tokens)),
            keywords=[t for t in tokens if t not in STOPWORDS][:5],
        )
```

The import line's package path mirrors the collection name: `acmeCo/classified/submissions` → `from acmeCo.classified.submissions import ...`. Slashes become dots; dashes in path segments become underscores.

## Canonical example — stateful

Per-key running counts, persisted across restarts. Adapted from `examples/derive-patterns/stateful.flow.py`.

### `flow.yaml` (excerpt)

```yaml
derive:
  using:
    python:
      module: running_totals.flow.py
  transforms:
    - name: fromInts
      source: acmeCo/source/ints
      shuffle: { key: [/Key] }   # keyed shuffle for stateful work
```

Keyed shuffle is non-negotiable: if two docs for the same `/Key` land on different shards, each shard sees empty state and counts never accumulate. This is the same rule as `derivation-stateful-logic`.

### `running_totals.flow.py`

```python
from collections.abc import AsyncIterator
from pydantic import BaseModel, Field
from acmeCo.processed.running_totals import IDerivation, Document, Request, Response


class State(BaseModel):
    class KeyState(BaseModel):
        count: int
        sum: int
    keys: dict[str, KeyState] = Field(default_factory=dict)


class Derivation(IDerivation):
    def __init__(self, open: Request.Open):
        super().__init__(open)
        self.state = State(**open.state)         # rehydrate from prior transaction
        self.touched = State()                   # keys changed this transaction

    async def from_ints(self, read: Request.ReadFromInts) -> AsyncIterator[Document]:
        ks = self.state.keys.setdefault(read.doc.Key, State.KeyState(count=0, sum=0))
        ks.count += 1
        ks.sum += read.doc.Int
        self.touched.keys[read.doc.Key] = ks
        yield Document(Key=read.doc.Key, Count=ks.count, Sum=ks.sum)

    def start_commit(self, _: Request.StartCommit) -> Response.StartedCommit:
        updated = self.touched.model_dump()
        self.touched = State()
        return Response.StartedCommit(
            state=Response.StartedCommit.State(updated=updated, merge_patch=True),
        )

    async def reset(self):
        self.state = State()
        self.touched = State()
```

Lifecycle:
- **`__init__(self, open)`** runs once per task shard. `open.state` is the previously-persisted JSON (empty dict on first run).
- **`start_commit(self, _)`** runs at the end of each transaction. `merge_patch=True` persists only the keys that changed; `False` replaces the full state. Merge patch is strictly better for large state.
- **`reset(self)`** runs between catalog tests so they don't carry state between cases.

Full source: https://github.com/estuary/flow/blob/master/examples/derive-patterns/stateful.flow.py

## Workflow

```bash
# 1. Pull the source collection locally
flowctl --profile <profile> catalog pull-specs --name acmeCo/source/submissions

# 2. Add the derive stanza to flow.yaml (as above)

# 3. Generate Pydantic stubs, pyproject.toml, pyrightconfig.json
flowctl --profile <profile> generate --source flow.yaml

# 4. Implement the class in <name>.flow.py

# 5. Preview locally against real source data
flowctl --profile <profile> preview --source flow.yaml \
  --name acmeCo/classified/submissions --timeout 30s

# 6. Publish to a PRIVATE data plane (required the first time for a new prefix)
flowctl --profile <profile> catalog publish --source flow.yaml \
  --init-data-plane ops/dp/private/estuary/aws-us-east-1-c1 \
  --auto-approve
```

`flowctl generate` creates `flow_generated/python/<tenant>/.../__init__.py` with typed `IDerivation`, `Document`, `Request`, `Response`, plus a skeleton `.flow.py` if absent. Re-running generate **does not** overwrite your implementation — only the `flow_generated/` tree.

## Dependencies

Declared under `derive.using.python.dependencies` as a map of package → version specifier (PEP 440). Installed by uv on container start — which means **at publish, preview, and `flowctl catalog test`**. A slow or failing install breaks all three.

```yaml
derive:
  using:
    python:
      module: embed.flow.py
      dependencies:
        model2vec: ">=0.3.0"
        httpx: ">=0.27.0"
        numpy: ">=2.0"
```

`pydantic >=2.0` and `pyright >=1.1` are auto-included — don't list them.

### Container environment

- **Python version is pinned to 3.14** at the container level (`ghcr.io/astral-sh/uv:python3.14-trixie-slim`). You don't choose it — whatever's in the container runs. In your `pyproject.toml`, use `requires-python = ">=3.12"` (or `>=3.14`); narrower upper bounds like `<3.13` cause uv to fail resolution.
- **No build toolchain.** The slim container has no `rustc`, `gcc`, or `make`. uv falls back to source builds when wheels aren't available for `cp314-linux_x86_64` — that's where dependency selection fails.

### What works

Pure-Python packages, numpy-family, and anything with pre-built `cp314` wheels for Linux x86_64. Before adding a dep, check on PyPI that a wheel exists for `cp314` (or that the package is pure-Python).

### What fails

Packages that require native build tools or hefty downloads. Concrete examples from production use:

- `sentence-transformers` — pulls ~1.5 GB CUDA torch as a transitive dep; install times out under the container's network budget.
- `fastembed` — depends on `py-rust-stemmers`, which has no `cp314` wheel and needs `rustc` to build from source.
- Anything declaring `setuptools >=78.0.0` dependencies with dash-separated keys in `setup.cfg` — pin `setuptools<78` alongside if you hit `Invalid dash-separated key 'description-file'`.
- Packages with wheels only up to `cp312` or `cp313` — uv will attempt a source build and fail.

For embeddings specifically, `model2vec` (pure-numpy lookup via distilled static vectors) is the path of least resistance — ships pre-built wheels, ~30 MB model download, no native deps.

### Debugging install failures

If publish/preview/test fails before your code runs, the install step is the most likely culprit. Reproduce locally with `uv pip install --python 3.14 <pkg>` against a fresh venv to check wheel availability before committing to a dependency.

## Async patterns

Every transform is an async generator. You can `await` I/O freely, but per-doc serial awaits limit throughput — for API calls, pipeline with bounded concurrency (full example: https://github.com/estuary/flow/blob/master/examples/derive-patterns/pipeline.flow.py):

```python
import asyncio
import httpx
from typing import AsyncIterator
# IDerivation, Document, Request imported from your generated module

class Derivation(IDerivation):
    MAX_CONCURRENT = 10

    def __init__(self, open: Request.Open):
        super().__init__(open)
        self.client = httpx.AsyncClient()   # shared across calls; reused within shard lifetime
        self.pending: set[asyncio.Task[Document]] = set()

    async def _enrich(self, src) -> Document:
        # Per-doc work — e.g. await an external API for each input doc.
        resp = await self.client.get(f"https://api.example.com/lookup/{src.id}")
        return Document(id=src.id, **resp.json())

    async def enrich(self, read: Request.ReadEnrich) -> AsyncIterator[Document]:
        if len(self.pending) >= self.MAX_CONCURRENT:
            done, self.pending = await asyncio.wait(self.pending, return_when=asyncio.FIRST_COMPLETED)
            for t in done:
                yield await t
        self.pending.add(asyncio.create_task(self._enrich(read.doc)))

    async def flush(self) -> AsyncIterator[Document]:
        # Drain remaining tasks at transaction close.
        for doc in await asyncio.gather(*self.pending):
            yield doc
        self.pending.clear()
```

`httpx.AsyncClient` is hoisted onto `self` so a single connection pool is reused for the shard's lifetime — constructing a new client per call leaks connections and defeats keep-alive.

`flush()` is a lifecycle hook the runtime calls at the end of each transaction, separate from your transform methods. Use it when you've buffered pending work.

## Runtime container constraints

- **`HOME=/nonexistent`** — `huggingface-hub` crashes with `PermissionError` trying to write `~/.cache/huggingface/`. Set `HF_HOME` + `XDG_CACHE_HOME` **before** importing the library:

  ```python
  import os
  os.environ.setdefault("HF_HOME", "/tmp/hf_cache")
  os.environ.setdefault("HF_HUB_CACHE", "/tmp/hf_cache/hub")
  os.environ.setdefault("XDG_CACHE_HOME", "/tmp/xdg_cache")
  from model2vec import StaticModel   # safe to import now
  ```

- **`/tmp` is ephemeral** — model re-downloads on every shard cold start. Fine for 30 MB; plan otherwise for 500 MB+.
- **Persistent state lives in RocksDB** on the reactor's `/mnt/local` and is shared with other tasks. No enforced cap — runaway per-task state can cause reactor-wide disk pressure, so self-police state growth.

## Testing

### Local — `flowctl preview --fixture`

```bash
flowctl --profile <profile> preview --source flow.yaml \
  --name acmeCo/classified/submissions \
  --fixture fixture.jsonl --timeout 30s
```

`fixture.jsonl`:

```json
["acmeCo/source/submissions", {"id":"1","created_at":"2026-04-24T10:00:00Z","text":"Fraud detection that sees transactions in seconds"}]
["acmeCo/source/submissions", {"id":"2","created_at":"2026-04-24T10:01:00Z","text":"LLM agent that schedules meetings"}]
{"commit": true}
```

Preview runs the derivation locally (no private DP needed for preview itself), but **does require a Docker daemon** — Python derivations execute inside `ghcr.io/estuary/derive-python:dev`. Same for `flowctl catalog test`. Set `DOCKER_CLI=podman` if you prefer podman. Stateful derivations build state within the session — use `--sessions` to exercise restart behavior.

Gotchas specific to local preview:
- **Preview can fail with `No space left on device` at `/tmp/.tmpXXX`** — local disk, not reactor. Clean `/tmp` and retry.
- **Preview runs everything in-process** — shard-splitting semantics don't match production. Good enough for correctness checks, not for load testing.

### Spec-defined tests — `flowctl catalog test`

Same `tests:` block syntax as other derivation types — see `derivation-basics`. `reset(self)` is called between test cases.

> **Caveat for Python derivations:** `flowctl catalog test` validates the spec but times out waiting for the Python container to become ready (shard-readiness window is too short for Python bootstrap). For runtime validation, publish to your private data plane and inspect the derived collection.

<!-- TODO: add a concrete `tests:` block for the `enrich` canonical example — ingest a few `acmeCo/source/submissions` docs covering each category branch + the "other" fallback, then `verify` the resulting `acmeCo/classified/submissions` documents. Include a stateful test for the `running_totals` example exercising `reset()` between cases. -->

## Gotchas specific to Python

- **Private/BYOC only.** Publishing a Python derivation to a public data plane fails at container start with `"Python derivations may only run in private data-planes"`. Use `--init-data-plane ops/dp/private/...` (or set the storage mapping) to land on a private plane.
- **Transform method name = snake_case of the YAML name.** `name: fromOrders` → `async def from_orders`. Mismatch raises `TypeError: Can't instantiate abstract class Derivation with abstract method <name>` at startup — the generated `IDerivation` base declares each transform as `@abstractmethod`, so a missing override blocks instantiation.
- **`async def` + `yield` is mandatory.** Regular `def` or missing `yield` isn't registered as an async generator — no documents flow. For state-only transforms keep the stub's `if False: yield` line.
- **Import from your own generated module for pyright coverage** — `from acmeCo.my.collection import IDerivation, Document, Request`. Path-based imports work but lose type safety.
- **`flow_generated/` is regenerated** every `flowctl generate` — never edit. Edit only `.flow.py` and the collection YAML.
- **`merge_patch=True` is almost always right** for `start_commit`. `False` rewrites full state JSON every commit.
- **Model re-downloads on every shard cold start.** Pre-warm with a dummy ingest before anything live.
- **Cross-plane source reads** (e.g., derivation on private reading a collection on public) work between non-legacy data planes as of 2026. The documented exception is firewalled BYOC planes, where cross-plane reads still don't work. Confirm specifics with your account manager for customer deployments.

## Related

- `derivation-basics` — prerequisite reading
- `derivation-stateful-logic` — SQLite alternative for stateful work on public planes
- `derivation-aggregate-metrics` / `derivation-windowing` — reductions that don't need procedural state
- [Python concept page](https://docs.estuary.dev/concepts/derivations/#python)
- [Python tutorial](https://docs.estuary.dev/guides/transform_data_using_python/)
- [Upstream examples](https://github.com/estuary/flow/tree/master/examples/derive-patterns)
- [Private deployments overview](https://docs.estuary.dev/private-byoc/private-deployments/)
