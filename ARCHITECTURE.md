# What Quilt should build for ML

Two references, read properly. One of them turns out to be empty, and that is the useful
finding.

---

## First: the fork is empty, and that is a real problem

`SuperInstance/llama3-from-scratch-in-quilt` — **public**, 16 MB, last pushed **2024-05-23**.

I compared it to upstream `naklecha/llama3-from-scratch` (15,209 stars) blob by blob:

```
files: fork=43  upstream=43
identical blobs : 43
DIVERGED blobs  : 0
only in fork    : 0
only in upstream: 0
```

**Two commits, both upstream's** — "art should be free", "everything is art". There is no
SuperInstance commit. No divergence. The innovation was never brought in.

And it is *public*, under a name that promises Quilt work. Anyone who finds it will read
"in-quilt", look, and find a fork of a fork with nothing Quilt in it. That is worse than
having no repo, because the name is a promise the contents don't keep.

**Decision for you:** archive it with provenance, or replace it with the real thing. Either
is fine. Leaving it as-is is not — it is a false claim about our own work, and we have spent
all night removing exactly that pattern from other repos.

---

## Second: what `simple-llm` actually is, and why it matters more than its 482 stars

~950 lines. 563 engine + 324 model + 92 tokenizer. Restricted to `gpt-oss-120b` on one H100.

And it **beats vLLM at batch 64** — 4,041 tok/s against 3,846 — while matching it at batch 1.

That is not the interesting part. The interesting part is the stated reason it exists:

> *Researchers trying new ideas. … Research labs that would otherwise fork vLLM. … Students
> learning how inference engines work.*

**Its design target is legibility, and it achieved competitiveness anyway.** 950 lines you
can read end to end, versus vLLM's hundreds of thousands.

**The fleet has 4,869 public repos.** That number is the same disease `simple-llm` was built
against. We are a lab that would otherwise fork vLLM — except we fork *everything*, and the
result is that nothing in the fleet is small enough to hold. The fix isn't more repos. It's
the discipline simple-llm demonstrates: **the unit has to fit in someone else's head.**

---

## The architecture

### The observation underneath it

A transformer forward pass and a Quilt cell graph **are the same shape**: a DAG of typed
nodes, edges carrying tensors, each node a function of its inputs.

Quilt's `BIND / LINK / EFFECT / VIEW / TICK` is already a vocabulary for that. A cell
graph is a legal Quilt program.

So the architecture is not "run a model inside a cell." It is:

> **The model is a quilt. Every activation is a cell boundary. Every cell is witnessable.**

### What that buys, which nothing else in the stack does

**1. Witness at cell granularity, not repo granularity.**

We log receipts per repository. A repo is far too coarse to say *where* a model went
wrong. If the forward pass is a cell graph, the witness log records per-cell: which cell
received which tensor, what it produced, and what it predicted it would produce.

This is the witness-log problem solved by a graph that already exists. We do not need to
invent the instrumentation; we need to stop not having a graph.

**2. The canary moves down to the activation.**

`0x024a555471370b18d` currently guards *source code* — a crate, a port, a substrate. The
same digest over **each cell's output** is a strictly stronger claim: a drifting cell is
caught at inference, in production, not in a test suite that ran on Tuesday.

**3. Forecast → witness → e-process, on a real object.**

The research from earlier tonight lands here. `E_t = E_{t-1} · q_t(x_t)/p_t(x_t)`, valid
for any stopping time, **provided the claimed conditional law is correct**. The model
forecast its own output statistics; the witness log records what happened; the e-process
tests them continuously.

The honest statement the system can then make:

> "No evidence against model version `v` has been detected at level α, under the stated
> conditional model."

Not "the model is true." That distinction is the whole design.

**4. Integrity, at the right granularity.**

Merkle tree over cells within a batch → hash-chain the batch roots → signed, externally
anchored checkpoints → consistency proofs. A linear chain over every cell is O(n) hashes
for a log only ever read at checkpoints, which is the thing we got wrong the first time.

**5. Legibility, enforced structurally.**

Every cell must be small enough to read. `simple-llm`'s constraint is the constraint: *if
you cannot hold the unit, it is not a cell, it is a subsystem.* That is checkable — and it
is the discipline that would have prevented both the 141-crate catalogue with median 104
lines of unspecified behaviour and the 4,869-repo sprawl.

### What to bring in from the *other* repo

Nothing — the fork is empty. But the *pedagogy* is the real contribution and it should be
rebuilt Quilt-native rather than forked:

Karpathy's build-up shows one matrix multiplication at a time, with an explanatory image at
each step, in a notebook you run until each stage comes alive. **That is
"artifact before narrative" made visual**, and it is why the repo has 15k stars. In Quilt
terms, each step is a cell, each image is a witness, and the notebook is the defuse view.

The Quilt version is strictly better than the fork: each step carries the canary, and each
step is independently testable, so the build-up is verified rather than merely narrated.

---

## What I would actually build, in order

**1. The cell-graph inference harness.** Small. One model, a handful of cell kinds, every
boundary witnessed. No attempt at speed — the point is that the graph exists and is
legible. This is the thing everything else depends on and it should be ~1000 lines by the
`simple-llm` standard, which is the discipline being tested.

**2. The canary at the activation.** FNV-1a 64 per cell output, compared against the
running fleet canary. Cheap, and it turns polyformalism from a claim about source code into
a claim about *computation*. If two substrates disagree mid-inference, that is a witness.

**3. The e-process over cell statistics.** One multiplier, log-space, running maximum,
Ville bound. The cheap correct version from the witness design. Per cell and per model, with
the version bump that keeps the error budget intact.

**4. The build-up, Quilt-native.** The `llama3-from-scratch` pedagogy with the canary
attached to every step, replacing the empty fork.

**5. Publish `simple-llm`'s discipline as a fleet rule, not a repo.** "If you cannot hold
the unit, it is not a cell." That is a doctrine worth more than another repository.

---

## The honest risks

- **A transformer block is not a cell.** Cells in this fleet have identity, witness, and
  self-witness; a matmul has none of those. The mapping is a *design proposal* until the
  witness story is real, and the witness story is the hard part.
- **E-processes need a forecast, not a point prediction.** A predicted mean is not a
  density. Until cells emit genuine predictive distributions, the e-process is not
  available and pretending otherwise would repeat exactly the mistake I caught in the
  verifier tonight.
- **This is a third architecture direction** after polyformalism and the typed/cloud
  runtimes. The doctrine says keep ecosystems separate. This proposal has to justify itself
  as a *use* of the cell vocabulary, not a fourth ecosystem wearing its clothes.
- **The two reference repos are not the same kind of thing.** One is pedagogy, one is an
  engine. Merging their lessons into one plan is my synthesis, not their claim, and it
  should be argued with.

## What I did not do

I did not fork, archive, or rename anything. The empty fork is a decision for you. I read
both repos and thought about the architecture, and that is all.
