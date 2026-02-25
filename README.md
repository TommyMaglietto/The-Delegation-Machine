# The Delegation Machine (V5.1) — Disruptor Shared-State Engine

A systems engineering architecture for **autonomous multi-agent AI software engineering** — designed to eliminate token bloat, kill centralized orchestration, and scale agent swarms at O(1) cost per task.

## What Is This?

The Delegation Machine is an evolving architectural framework that treats AI agents not as chatbot teammates, but as **high-throughput compute kernels** operating on shared state. It draws from high-performance distributed systems, compiler theory, and game theory to solve the fundamental problems of multi-agent coordination:

- **Token cost explosion** (solved via zero-copy prefix caching)
- **Orchestrator bottlenecks** (solved via lock-free ring buffers)
- **Agent coordination overhead** (solved via stigmergic shared state)
- **Quality control latency** (solved via deterministic fail-fast supervision)

## Architecture Overview

### Core Engine
- **LMAX Disruptor Ring Buffer** — Lock-free task ingestion replacing centralized orchestrators
- **CodeCRDT (Conflict-free Replicated Data Types)** — Shared mutable codebase state; agents observe and mutate, never "chat"
- **Zero-Copy Prefix Caching (RadixAttention)** — Shared GPU KV-cache prefix across all agents; only delta tokens processed per task
- **Static Single Assignment (SSA) Reasoning** — Forward-only reasoning DAGs enabling aggressive token pruning

### Intelligent Scheduling (V5.1)
- **HEFT (Heterogeneous Earliest Finish Time)** — Critical-path DAG prioritization
- **Tomasulo's Algorithm** — Out-of-order agent execution via dependency tag resolution
- **Cilk Work-Stealing** — Idle agents steal sub-DAG roots from busy agents' deques

### Elastic Memory Management (V5.1)
- **kvcached** — Virtual memory for KV-cache; reserve large, allocate on demand
- **Hierarchical Offloading** — `Local VRAM → Peer VRAM → Host DRAM → Network Storage`
- **PyramidKV** — Attention-aware eviction (lower layers get more budget, sinks are pinned)
- **Fisher Market Bidding** — Agents bid for cache slots under VRAM pressure via Proportional Response Dynamics

### Quality & Incentive Alignment (V5.1)
- **Erlang/OTP Fail-Fast Supervision** — Deterministic compilers/tests as supervisors; crashed agents are killed instantly, state rolled back
- **SHARP (Shapley-based Rewards)** — Tripartite scoring: global accuracy + marginal contribution + tool efficiency
- **EigenTrust Reputation** — Peer-to-peer trust propagation with temporal decay

## Repository Contents

| File | Description |
|------|-------------|
| `MASTER_AI_PROCEDURES.md` | The operational rulebook — agent lifecycle, scheduling protocols, supervision trees, incentive alignment, and continuous homeostasis |
| `CONTEXT_AND_MEMORY_ARCHITECTURE.md` | The technical blueprint — memory hierarchy, KV-cache economics, CRDT state management, event sourcing, and SSA reasoning |

## Evolution

| Version | Codename | Key Innovation |
|---------|----------|----------------|
| V4 | Zero-Trust Constitutional Framework | 3-Branch separation of powers, VCM paging, GraphRAG |
| V5 | Disruptor Engine | CodeCRDT, LMAX Ring Buffer, prefix caching, Erlang supervision |
| **V5.1** | **Swarm Operating System** | **HEFT/Tomasulo scheduling, Fisher Market memory, SHARP incentives, work-stealing** |

## Getting Started

1. Read `MASTER_AI_PROCEDURES.md` first — it defines the execution model and agent lifecycle
2. Then read `CONTEXT_AND_MEMORY_ARCHITECTURE.md` — it details the memory hierarchy and state management

## Current Status: Architecture Specification

This is an **architectural specification**, not a running system. The concepts reference real, proven systems (LMAX Disruptor, Erlang OTP, CRDTs, Shapley values), but nobody has wired them together for AI agent swarms before. The gap between "this should work in theory" and "this actually works" is non-trivial in distributed systems.

We believe transparency about this strengthens the architecture — acknowledging specific, addressable limitations is a sign of maturity, not weakness.

## Known Limitations & Design Tradeoffs

### 1. CRDT Granularity: File-Level, Not Line-Level

The CodeCRDT operates at **coarse-grained, file-level coordination** — closer to a distributed filesystem with mutex semantics than a Google Docs-style real-time editor.

**Why this matters:** Text CRDTs exist (Automerge, Yjs), but merging concurrent *code* edits where Agent A refactors a function that Agent B is calling creates semantic conflicts that CRDTs cannot resolve mathematically. A syntactically valid CRDT merge can produce code that doesn't compile.

**Our approach:**
- File-level locking prevents concurrent edits to the same file
- The Supervisor tier (compilers/tests) catches any semantic breakage post-merge
- Future iterations may introduce AST-aware merge strategies for finer granularity

> See `CONTEXT_AND_MEMORY_ARCHITECTURE.md` §1.1 for the full CodeCRDT specification.

### 2. Prefix Caching Requires Self-Hosted Inference

The O(1) token cost claim depends on running **vLLM or SGLang with RadixAttention** on self-hosted or dedicated inference endpoints. If you're using OpenAI's API, Anthropic's API, or any hosted provider, you don't get zero-copy KV-cache sharing across agents.

**Degradation path for API-based setups:**
- Anthropic and OpenAI offer **prompt caching** (automatic for repeated prefixes), which provides partial benefit — typically 50-90% cost reduction on cached prefixes, not the full O(1)
- The architecture still works with API providers; you lose the extreme cost optimization but retain all other properties (scheduling, supervision, incentives)
- The Elastic KV-Cache and Fisher Market bidding sections are only relevant for self-hosted deployments

> See `CONTEXT_AND_MEMORY_ARCHITECTURE.md` §4 for prefix caching details.

### 3. Shapley Values Use Approximation at Scale

Computing exact Shapley values is **O(2^n)** where n is the number of agents. With 5 agents this is trivial. With 50 agents it's astronomically expensive.

**Our approach:**
- Production deployments use **Monte Carlo Shapley sampling** (random coalition sampling) — standard in the cooperative game theory literature
- For swarms of 10-20 agents: exact computation is feasible
- For swarms of 50+: approximate Shapley with configurable sampling budget (1,000-10,000 samples) provides statistically significant attribution at bounded cost
- The SHARP scoring system degrades gracefully — the Global Accuracy and Tool-Process Efficiency components require zero coalition computation

> See `MASTER_AI_PROCEDURES.md` §4.1 for the SHARP scoring specification.

### 4. "Let It Crash" Is Augmented with Partial Recovery

Pure Erlang-style crash-and-restart is expensive for LLM agents — an agent that spent 30 seconds generating 2,000 tokens of near-correct code shouldn't be killed and restarted from scratch for a one-line error.

**Our approach (V5.1 refinement):**
- **First failure:** The Supervisor passes the diff + compiler error back to the *same* agent as delta context. The agent attempts a **targeted fix** without regenerating from scratch. This is the "partial recovery" path.
- **Second failure:** Full restart with the accumulated error context (both stack traces) injected into the task's delta context for a fresh agent.
- **Third failure:** Circuit Breaker activates — task is escalated to a Senior Architecture agent or a human developer.

This three-tier strategy bounds token waste to ~1.3x the optimal case (targeted fix succeeds) rather than the naive 3x (three full regenerations).

> See `MASTER_AI_PROCEDURES.md` §2 for the full supervision tree.

### 5. Adversarial Agent Robustness

EigenTrust handles **lazy** agents (reputation decays). But what about a **poisoned** agent that intentionally introduces vulnerabilities? The architecture currently assumes agents are honest-but-potentially-incompetent, not malicious.

**Mitigation strategy:**
- The deterministic Supervisor tier already catches many classes of malicious output (code that doesn't compile, tests that fail)
- For semantic attacks (backdoors that pass tests), the Supervisor tier should include **static analysis tools** (Semgrep, CodeQL, Bandit) alongside compilers
- EigenTrust's reputation propagation naturally isolates agents whose outputs cause downstream failures, even if those failures only manifest in other agents' work
- For open/adversarial deployments: add a **quarantine period** for new agents where outputs receive mandatory secondary review before CRDT commit

> See `MASTER_AI_PROCEDURES.md` §4.2 for EigenTrust details.

### 6. Infrastructure Requirements

The full V5.1 architecture assumes:

| Component | Requirement | Fallback |
|-----------|------------|----------|
| Inference | Self-hosted vLLM/SGLang | API providers (lose prefix caching optimization) |
| CRDT State | Redis or custom CRDT server | SQLite for single-node deployments |
| Event Ledger | Kafka or compatible event stream | SQLite append-only log for development |
| KV-Cache Management | Multi-GPU with NVLink | Single-GPU with DRAM offloading |
| CI/CD Supervisor | Docker containers with compilers/test suites | Local process execution |

## Roadmap

| Phase | Milestone | Description |
|-------|-----------|-------------|
| **PoC** | Proof-of-Concept | 2 agents sharing a CRDT, 1 supervisor killing failed agents, basic ring buffer. ~200 lines of Python. |
| **V5.1.1** | Single-Node Engine | Full HEFT scheduling, Erlang supervision tree, SQLite event ledger. Single GPU. |
| **V5.1.2** | Multi-Agent Scaling | Work-stealing, Fisher Market bidding, EigenTrust reputation. Multi-GPU. |
| **V5.1.3** | Production Hardening | Adversarial robustness, static analysis in supervisor tier, Monte Carlo Shapley at scale. |

## Practical Applications

- **AI App Factories** — Parallel agent teams building full-stack apps with deterministic build supervisors
- **Autonomous Dev Agencies** — Multi-agent swarms producing MVPs at drastically reduced token cost
- **Research Frameworks** — Testbed for novel multi-agent coordination, incentive design, and elastic resource allocation

## License

MIT
