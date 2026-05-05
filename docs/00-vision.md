# 00 — Vision

> Captured 2026-05-05 from a conversation that started with "my vision is to assemble all the claude codex session, local agents in a agent os where they work as a real team, co operate."

## The product

A local agent operating system. Connects every AI agent on the machine — Claude Code, Codex CLI, local Llama, web scraper, voice TTS/STT, vision OCR/CV, specialist agents — into a coordinated team that handles open-ended work end-to-end without babysitting.

User-facing experience: ambient, conversational, voice-capable. User states an intent ("research X and write a doc", "audit my repo and fix the issues you find", "monitor my email and triage"). The OS decomposes the goal into tasks, assigns each to the right agent, manages handoffs, gates on quality, surfaces results.

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
       │              AGENT ROSTER  (the team)               │
       │ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
       │ │Claude   │ │Codex    │ │Local    │ │Web      │ ... │
       │ │Code     │ │CLI      │ │Llama    │ │scraper  │     │
       │ └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
       │ ┌─────────┐ ┌─────────┐ ┌─────────┐                 │
       │ │Voice    │ │Vision   │ │Specialist│                │
       │ │TTS/STT  │ │OCR/CV   │ │(yours)   │                │
       │ └─────────┘ └─────────┘ └─────────┘                 │
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

The hardcore engineering is solving these seven problems:

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

Solve these and you have an agent OS. Skip them and you have a chat that fans out to multiple LLMs.

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
