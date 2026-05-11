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
# Veculo Agentic OS

> **Bounded autonomy, cryptographically proven.** Grant an agent the
> authority to act by declaring its scope. The substrate proves on
> every dispatch that the agent stayed inside that scope; the moment
> it doesn't, the gate stops it.

Veculo's Agentic OS is the substrate for running autonomous agents
against production graph state. Each dispatch is captured as a
cryptographically-anchored row in the graph; a continuity verifier
checks the agent's trajectory against its declared scope on every
dispatch and continuously via a cron sweep; the moment the agent
strays, the gate rejects further work. Operators audit what actually
happened by querying the graph — not by reading the agent's chat
transcript.

This directory contains two documents. Pick the one that fits what
you came for.

## [overview.md](overview.md) — operator's view

Read this first. About 1 200 words. Covers:

- **What you'd use this for** — three concrete deployment shapes
  (code review, research, ops triage)
- **What's actually happening in a 75–110 second run** — tool-call
  breakdown showing 19 graph operations per run, not LLM-in-a-vacuum
- **What you get when it works** — N=3 measurement headline numbers
- **What you get when it goes wrong** — the load-bearing claim:
  scope-violation injection → circuit-broken at the gate → forensic
  replay from graph rows alone
- **What this substrate is _not_** — explicit non-claims (not a faster
  `/rag`, not a trust-me wrapper, not a sandbox)

## [methodology.md](methodology.md) — engineering record

Read this when you want to verify the overview's claims. About
5 000 words. Covers:

- The full apples-to-apples three-row comparison (one-shot RAG vs
  agentic-no-memory vs agentic-with-memory-and-verifier)
- N=3 measurement table with mean ± stdev, per-run breakdown,
  cache-hit rates, output-token volumes
- Latency decomposition (~61% output generation, ~10% prefill,
  <0.1% PIC overhead)
- Acts 2–4: failure injection, forensic replay, operator repair
- Substrate bugs surfaced during the build (7 of them, each
  documented with cause + fix)
- Limitations — what the measurement does and does not defend
- Reproducibility — exact env-var + script invocation to run it
  yourself

## Quick navigation by question

| If you want to know… | Read |
|---|---|
| Why this matters / what to use it for | overview.md |
| Does it actually work / what did you measure | methodology.md |
| How would I deploy this | overview.md → §"What you'd use this for", then methodology.md → §"Reproducibility" |
| What happens when an agent misbehaves | overview.md → §"What you get when it goes wrong" + methodology.md → §"Acts 2–4" |
| What's the catch / where are the gaps | methodology.md → §"Limitations" |
| Where does the substrate code live | methodology.md → §"Reproducibility" → file table at the bottom |
