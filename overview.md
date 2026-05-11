<!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.

-->
# Agentic OS — Overview

> **Bounded autonomy, cryptographically proven.** Grant an agent the
> authority to act by declaring its scope. The substrate proves on
> every dispatch that the agent stayed inside that scope; the moment
> it doesn't, the gate stops it.

For the deep methodology, per-run measurements, and limitations, see
[Agentic OS — Demo Methodology and Results](methodology.md).
This page is the operator's view: what it's for, what's happening
when it runs, and what you get.

## What you'd use this for

Three deployments this substrate enables today that without it
require constant human-in-the-loop review:

1. **Autonomous code review agents writing structured findings to a
   shared knowledge graph.** An agent reads a PR diff, validates
   claims against the graph, and writes verdicts (HOLDS / DISPUTES /
   INCONCLUSIVE) with confidence and cited source vertices as durable
   `finding:` rows. A human reviewer audits the conclusions by
   querying the graph, not by re-reading the agent's chat transcript.

2. **Research agents traversing and writing to a production graph.**
   The agent that ran this demo. It answers multi-hop questions
   ("enumerate every implementer of `X` across the codebase") by
   iteratively reading vertices, running sandboxed Python compute,
   and writing structured conclusions. Every dispatch is
   cryptographically anchored; if the agent's trajectory ever exceeds
   its declared scope, the substrate circuit-breaks it before the
   next write lands.

3. **Operations agents triaging incidents with read+write access to
   live state.** Agent reads metrics, runs sandboxed compute against
   them, writes recommendations or applies bounded fixes. The ops
   engineer reviews via the audit trail — not by trusting the agent
   to be honest about what it did, but by querying the graph for the
   actual dispatch chain.

In all three, the operator pattern is the same: **declare scope, let
the agent run, trust the substrate not the agent**. The agent is the
worker; the substrate is the contract.

## What's actually happening in 75–110 seconds

A representative agentic-OS run on this demo (Row B, run 1, 109.8s
wall time) issued **20 tool calls**, all but 1 against the graph:

| Operation | Count | What it does |
|---|---|---|
| `query_vertex` | 9 | Single-row graph read via the read-fleet (`shoal`), ~300 ms each |
| `record_finding` | 5 | Graph write + PCA chain narrowing + replay-ledger emit + classification injection (one durable structured finding per call) |
| `run_rag_query` | 3 | Full production RAG: vector search + graph expansion to depth 2 + cross-encoder reranking + LLM synthesis (~2–3 s end-to-end; each one internally walks hundreds of graph edges) |
| `query_neighborhood` | 1 | Multi-hop graph traversal capped at depth 3 / 100 vertices |
| `recall_fold` | 1 | Cross-session memory lookup |

That's **19 graph operations + 1 memory recall**, plus the LLM time
to reason between each turn and write ~5 000 output tokens of
structured findings. The shortest run (74.9 s) still did 12 graph
operations plus 1 RAG query plus 1 finding write. The agentic loop is
not an LLM thinking in a vacuum — it is substrate-heavy work.

## What you get when it works

Headline numbers from the 2026-05-11 N=3 run on the `cl-kgun2u`
tenant, against a 8 600-fragment ingested codebase:

| Claim | Measurement |
|---|---|
| Agent enumerates all 4 `LogCloser` implementations across two directories | ✅ 4 of 4 on every agentic run (one-shot graph RAG returns 3 of 4) |
| Writes structured findings every run | ✅ 5 `finding:` vertices per run with verdict + confidence + cited sources |
| PCA chain cryptographically linked end-to-end | ✅ 5/5 hops linked via HMAC-SHA256 parent_sig |
| Replay ledger bound to dispatches | ✅ 5/5 replay rows carry `_command_hash` matching the SAG canonical action text |
| Post-session continuity verifier passes | ✅ 6/6 dimensions (origin / hops / time / scope / sigs / chain↔ledger / classification round-trip) |

## What you get when it goes wrong (the load-bearing claim)

Mid-run, the demo deliberately injects a scope violation — a PCA hop
that re-expands its allowed ops to include `delete`, an op the parent
hop did not grant. The substrate, with **no agent cooperation
required**:

1. Writes a `continuity_violation:` row with the cryptographic
   evidence (parent_sig prefix, command_hash, executor identity, the
   exact dimension that broke).
2. Transitions the agent's `agent_state` to `circuit_broken` via a
   `TRIGGERED_BY` edge anchored to the violation.
3. Rejects every subsequent dispatch from that agent at the gate
   (`HTTP 403 agent_circuit_broken`).

The operator's view of the failure is reconstructed entirely from
graph rows. There is no log file to grep. There is no agent claim to
verify. The trajectory of what the agent tried to do is durable in
the substrate, cryptographically anchored, and machine-checkable by
anyone with read access.

When the operator resolves the violation (`_resolution_state: open →
repaired`), a `resumed` `agent_state` row is written and dispatches
flow again from that agent. The audit trail is preserved end-to-end —
nothing is rewritten or hidden by the repair.

## What this substrate is _not_

- **Not a faster `/rag`.** For single-hop lookups (e.g. "what does
  class X do?"), one-shot graph RAG returns in ~22 s and is the right
  surface. The agentic path's value compounds with question
  complexity, multi-hop traversal, and audit requirements. Use the
  right tool for the shape of the question.
- **Not a trust-me LLM wrapper.** The substrate does not trust the
  agent's claims about what it did. Every dispatch is anchored in
  cryptographic state separate from the agent's text output. An
  agent that hallucinates "I called record_finding 5 times" leaves no
  PCA chain entries; the absence is detectable.
- **Not a sandbox.** Sandboxing for `run_python` is a separate
  component (gVisor-isolated, 60 s wall, 512 MB, no outbound network).
  The continuity substrate operates on the agent's _graph dispatch
  trajectory_, not its compute environment.

## Where this is heading: GPU-resident working sets for very large graphs

The 75–110 s wall time on this demo is bound by **LLM output
generation, not retrieval** — the agent writes ~5 000 output tokens
per run at ~85 tokens/sec on Sonnet 4.6, which alone accounts for
roughly 60 % of total latency. Prompt caching already closed the
prefill cost (uncached input tokens dropped 224 K → ~10 in this run).
The substrate-side overhead is < 0.1 %. Output generation is the
floor on the current architecture, not the substrate.

That picture changes as graphs grow. On this demo's 8 600-fragment
tenant, a `run_rag_query` is ~2–3 s end-to-end and the vector-search
phase inside it is ~100–500 ms. On a **million-fragment graph**,
vector search alone climbs to 2–5 s per query; with 3 RAG queries
per run, that's 9–15 s of vector search on the critical path.
Retrieval becomes the bottleneck at scale.

The architecturally clean next move is **GPU-accelerated vector search
(cuVS / CAGRA) over session-scoped subgraph working sets resident in
GPU memory**:

```
   persistent graph (RFiles, GCS)
          │
    [shoal read fleet — today]
          │
   ┌──────┴──────────────────────────────┐
   │  Session working set (GPU-resident) │
   │   • subgraph adjacency (CSR)        │
   │   • embeddings buffer               │
   │   • cuVS index for vector search    │
   │   • TTL = session lifetime          │
   └──────┬──────────────────────────────┘
          │
   agentic loop: many tool calls, all served from the working set
```

Sanity-check on the memory footprint: 100 K vertices × 768-dim ×
4 bytes ≈ 300 MB of embeddings, fits on any single GPU. A
million-vertex working set is ~3 GB — also comfortable. cuVS
benchmarks against the same IVF-PQ index this platform already
ships show 10–50× speedups over CPU search at the scales where
retrieval becomes the bottleneck.

The architectural question that decides whether this composes
cleanly with the PIC substrate: do reads against the working set
still emit dispatch records? Two viable answers, each with explicit
tradeoffs:

- **(a)** every read still goes through `gate_write` for full
  auditability, with a small synchronous overhead per lookup;
- **(b)** reads are working-set-local; the substrate sees batched
  read profiles emitted async every N ops, not individual lookups.

Writes go through `gate_write` in both cases — the substrate's
write-side guarantees stay intact regardless. The choice is whether
the read-side trajectory is auditable at per-lookup granularity or
at session-aggregate granularity.

**The smallest spike that produces a real number** is half a day's
work: take the existing IVF-PQ index, load codebook + posting lists
into a GPU buffer via cuVS, run the same `run_rag_query` calls from
this demo against both paths, compare wall time. Doesn't touch the
substrate. Produces a measured baseline for how much GPU
acceleration would shift the picture on the workloads where it
matters — graphs an order of magnitude larger than the one this
demo runs against.

## Where to read more

- **[Agentic OS — Demo Methodology and Results](methodology.md)**
  — full N=3 measurement table, per-run breakdowns, latency
  decomposition, substrate bugs surfaced during build, limitations,
  reproducibility instructions, and the full Acts 2–4 forensic trace.
