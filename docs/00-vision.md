# 00 — Vision

> Captured 2026-05-05 from a conversation that started with "my vision is to assemble all the claude codex session, local agents in a agent os where they work as a real team, co operate."

## The product

A local-first agent operating system. **Brokers across the user's scattered AI agents — wherever they physically live — and makes them behave like one coordinated team.** Local Claude Code, Codex CLI in another terminal, a rented Devin instance, custom GPTs in ChatGPT, Cursor in the editor, a local Llama for private work, specialist agents per domain — all of them appear as one roster, take orders from one supervisor, share state through one substrate.

User-facing experience: ambient, conversational, voice-capable. User states an intent ("research X and write a doc", "audit my repo and fix the issues you find", "monitor my email and triage"). The OS decomposes the goal into tasks, picks the right agent for each based on capability + cost + availability (regardless of vendor), manages handoffs, gates on quality, surfaces results.

## The structural opportunity (why this is defensible)

The agent population is fragmenting. Users will own (or rent) agents from many vendors — Anthropic, OpenAI, Cognition (Devin), Cursor, custom GPTs, local LLMs, specialist providers. Each vendor builds a walled garden because **lock-in is every vendor's business model**. Anthropic will never make Claude play nicely with OpenAI Operator. OpenAI will never make ChatGPT coordinate with Devin. The walled gardens are deliberate.

This leaves an open, structurally-defensible position:

- **No vendor can ship the broker** — it disadvantages them. So the broker has to come from outside the vendors.
- **A cloud broker would just be another vendor garden** — whoever runs the broker server gets the lock-in. So the broker has to be local-first.
- **An open standard (MCP, etc.) is necessary but not sufficient** — the protocol layer doesn't solve scheduling, cost arbitrage, audit, or capability matching. So the broker has to be a real product, not just a spec.

Agent OS sits exactly in that hole: **local-first, vendor-neutral, real product (not just protocol).** Once the user has 3+ agent subscriptions, the broker becomes the most valuable piece of software they own — because every other agent runs through it.

## Architecture

```
                          ┌──────────────────┐
                          │     YOU          │
                          └────────┬─────────┘
                  voice / chat / cli / ambient
                                   │
              ┌────────────────────▼────────────────────┐
              │     INTERFACE LAYER                     │
              │  intent classifier, dispatcher,         │
              │  notification router (push, voice)      │
              └────────────────────┬────────────────────┘
                                   │
              ┌────────────────────▼────────────────────┐
              │     SUPERVISOR / SCHEDULER              │
              │  decomposes goals → tasks               │
              │  matches tasks to agent capabilities    │
              │  enforces budget + cost ceiling         │
              │  decides synchronous vs parallel        │
              └─┬──────────────────┬──────────────────┬─┘
                │                  │                  │
       ┌────────▼──────┐  ┌────────▼──────┐  ┌────────▼──────┐
       │   COMM BUS    │  │  TASK QUEUE   │  │   ARBITER     │
       │ typed msgs    │  │ claim/release │  │ conflict      │
       │ pub/sub       │  │ handoffs      │  │ resolution    │
       │ direct addr   │  │ retries       │  │ voting / GT   │
       └────────┬──────┘  └────────┬──────┘  └────────┬──────┘
                │                  │                  │
       ┌────────▼──────────────────▼──────────────────▼──────┐
       │            ADAPTER LAYER  (the moat)                │
       │ normalizes auth + streaming + tool format +         │
       │ billing units + capability declarations across      │
       │ every agent vendor's idiosyncratic API              │
       └────────┬────────────────────────────────────────────┘
                │
       ┌────────▼────────────────────────────────────────────┐
       │            AGENT ROSTER  (your team)                │
       │                                                     │
       │  LOCAL                CLOUD-RENTED       VENDOR     │
       │  ┌──────────┐         ┌──────────┐      ┌─────────┐ │
       │  │ Claude   │         │ Devin    │      │ ChatGPT │ │
       │  │ Code     │         │ ($500/mo)│      │ Operator│ │
       │  └──────────┘         └──────────┘      └─────────┘ │
       │  ┌──────────┐         ┌──────────┐      ┌─────────┐ │
       │  │ Codex CLI│         │ Custom   │      │ Cursor  │ │
       │  │          │         │ GPTs     │      │ agents  │ │
       │  └──────────┘         └──────────┘      └─────────┘ │
       │  ┌──────────┐         ┌──────────┐                  │
       │  │ Local    │         │ Per-task │                  │
       │  │ Llama    │         │ rentals  │                  │
       │  └──────────┘         └──────────┘                  │
       │  ┌──────────┐                                       │
       │  │ Specialist│ (your own — voice, vision, scrapers, │
       │  │ agents    │  domain-specific)                    │
       │  └──────────┘                                       │
       └────────┬────────────────────────────────────────────┘
                │
       ┌────────▼────────────────────────────────────────────┐
       │      SHARED STATE  (machine-memory + extensions)    │
       │  files+wiki+activity (mmd) │ agents │ tasks │       │
       │  messages │ permissions │ reputation │ audit log    │
       └────────┬────────────────────────────────────────────┘
                │
       ┌────────▼────────────────────────────────────────────┐
       │       SAFETY + OBSERVABILITY                        │
       │  permission gates │ audit │ eval harness │ kill sw  │
       └─────────────────────────────────────────────────────┘
```

## Where the boost comes from (six engineerable mechanisms)

Not magic. Each mechanism is concrete engineering. Stack 4-6 of these and the system genuinely outperforms any single agent on real tasks.

| Mechanism | What it gives | Hard part |
|---|---|---|
| **Specialization** | Small fast model triages; big slow model synthesizes. Cheaper + faster + often better than one model doing everything. | Capability registry + routing logic. |
| **Parallelism** | 5 agents research different angles concurrently → 5x throughput on decomposable work. | Task decomposition that doesn't create coordination overhead. |
| **Self-review** | Agent B reviews A's output, catches errors A would miss. (Demonstrated in the machine-memory PR review cycles — hostile reviewer found real blockers we shipped.) | Reviewer must be independent enough to disagree. |
| **Persistence** | Knowledge accumulates across sessions. No re-derivation of the same context every time. | The shared-state layer (machine-memory). |
| **Tool composition** | A → B → C pipelines (research → plan → implement → test → commit), each stage focused. | Typed contracts between stages so handoffs don't lose information. |
| **Cost arbitrage** | Cheap model for grunt work, expensive model for hard reasoning, dynamically. | Supervisor that can route by difficulty. |

## What "hardcore engineering" actually means

The naive version of multi-agent is *worse* than single-agent. Five agents talking past each other, hallucinating confidently because they trust each other's output, burning 10x tokens for marginal benefit, impossible to debug when something goes wrong. That is what most "agent framework" projects ship and why they don't deliver.

The hardcore engineering is solving these seven coordination problems:

```
1. COORDINATION OVERHEAD       agents must spend < 30% of tokens
                                coordinating; otherwise serial agent wins
2. ERROR COMPOUNDING            wrong A → wrong B → wrong C; need
                                gates between stages that catch drift
3. HALLUCINATION AMPLIFICATION  agents trust other agents' confidence;
                                need ground-truth checkpoints
4. COST CONTROL                 multi-agent can be 5-10x more expensive;
                                need budgets + cost-aware routing
5. DEBUGGABILITY                when output is wrong, who failed? Need
                                full audit trail + replay
6. PERMISSION + SAFETY          agents on YOUR machine doing YOUR work;
                                need permission gates + kill switch
7. EVALUATION                   "did the team produce good output?"
                                need automated eval, not just user trust
```

Plus seven *adapter* problems that come with brokering across vendors (this is the real moat — every vendor has their own idiosyncrasies):

```
A. AUTH NORMALIZATION          Anthropic API key + OpenAI bearer token +
                                Devin webhook secret + local llama no-auth
                                + Cursor session cookie — all behind one
                                interface
B. STREAMING SEMANTICS         SSE vs WebSocket vs chunked HTTP vs polling;
                                tool-call interleaving differs per vendor
C. TOOL FORMAT TRANSLATION     OpenAI function calling vs Anthropic tool
                                use vs MCP vs custom — same task, different
                                wire format on each side
D. BILLING UNIT NORMALIZATION  tokens (input/output, cached/fresh) vs
                                per-request vs compute-seconds vs per-task
                                fees ($0.50 per Devin task, $0.02 per
                                Claude turn, $0/local). All into one cost
                                ledger.
E. RATE LIMIT MANAGEMENT       each upstream has different limits, different
                                backoff signals, different reset windows
F. LATENCY VARIANCE            local Llama: 10ms first token; Claude API:
                                500ms; Devin: 30+ seconds for a task.
                                Supervisor must budget time AND know the
                                latency profile per agent type
G. CAPABILITY DECLARATION      vendors describe what their agent can do
                                differently (or not at all). Need a
                                normalized capability schema so the
                                supervisor can match tasks
```

These adapter problems are why you need real systems engineering — not a Python script with 14 if-statements per vendor. The adapter layer must be:

- **A pluggable abstraction** (new vendor support in < 1 day, not a fork)
- **Cost-aware end-to-end** (dollar amount for any goal pre-execution, exact actual cost post-execution, drift < 5%)
- **Failure-isolating** (one vendor's outage doesn't bring down the OS — supervisor reroutes)
- **Audit-complete** (every external call logged with full request/response for replay)

Solve all 14 problems (7 coordination + 7 adapter) and you have an agent OS. Skip them and you have a chat that fans out to multiple LLMs.

## Why this requires Go (or Rust) — not Node, not Python

The previous machine-memory daemon is Node + TypeScript. That works because mmd is mostly I/O — chokidar events, SQLite writes, MCP responses. Concurrency is light, latency is loose.

Agent OS is different. Adapters need real concurrency (one daemon, dozens of in-flight upstream calls, each with its own streaming semantics + retry logic + cost tracking). The hot path needs predictable memory (no GC stalls during a critical handoff). Single-binary deployment matters because the daemon will run on every user's machine. eBPF tooling (`cilium/ebpf` for Go, `aya` for Rust) needs to be reachable without an FFI maze.

```
Language pick rationale:

Go     ✓ production-grade goroutines map cleanly to per-agent workers
       ✓ single binary, easy systemd + portability
       ✓ cilium/ebpf is the most mature eBPF tooling outside C
       ✓ ships fast — half the time-to-v1 of Rust
       ✗ GC pauses possible (mitigated by careful allocation, off-heap
          for hot paths)

Rust   ✓ no GC; tighter perf ceiling; tokio is excellent
       ✓ aya for eBPF is also production-ready
       ✗ build time + cognitive load slow v0.1 by months
       ✗ async Rust is still a sharp tool; needs a senior dev who's
          done it before to ship without regret

Node   ✗ single-threaded event loop wrong shape for the workload
       ✗ adapters with native bindings (cost ledger crypto, eBPF, etc.)
          would be a worker_threads / native-addon nightmare
       ✗ used in mmd by historical accident, not because it's the
          right tool

Python ✗ no real concurrency story for this workload
       ✗ would force every adapter to wrap async/await around blocking
          libs; never ends well at production load
```

**Recommendation: Go for v0.1.** Rust if a Rust expert is on the team day one. Decision is logged in `02-roadmap.md`.

## What this is NOT

**"Near AGI" is the outcome of a working agent OS for narrow domains, not a guaranteed result.** A great agent OS that orchestrates Claude Code + Codex + local LLMs will *feel* AGI-like for software engineering, research, content production — anywhere the agents have strong tool access. It will NOT make the underlying models smarter. The intelligence ceiling is still the best individual model in the swarm.

That is not a limitation that should kill the vision — feeling AGI-like for 90% of daily work is enormous value. But the framing matters: **agent OS = orchestration leverage on top of frontier models**, not a path to building a smarter model. If the goal is actual AGI, that is not on this product's path — that is a model-training program, completely different beast.

## Today's evidence the approach works

Within a single Claude Code session, the Agent tool already spawns subagents that report back to the orchestrator. That is a team-of-agents pattern, just scoped to one conversation. It works — the machine-memory PR review cycles in this repo's sister project demonstrate concrete value (3 review passes, zero false positives, real blockers caught before merge).

What's missing to turn that primitive into an OS:

1. **Persistence** — subagents die when the conversation ends. Cross-session continuity needs the shared-state layer (machine-memory).
2. **Cross-tool addressing** — Claude Code can spawn its own subagents but cannot address a Codex CLI session running in another terminal, or a local Llama daemon. There is no shared "agent bus."
3. **Long-running** — subagents are tied to a conversation turn. An agent OS needs background daemons that outlive any single user interaction.
4. **Identity** — every subagent is ephemeral. No "Claude-research agent has been working on this for 3 days."

The OS layer adds these. The pattern (orchestrator + specialized workers) is already validated.
