# 01 — Scope and Goal

## Goal (one sentence)

Build a local-first daemon that turns the user's existing collection of AI agents (Claude Code, Codex CLI, local LLMs, specialist agents) into a coordinated team that handles open-ended work end-to-end with full audit + permission control.

## Success criteria (how we know it works)

The OS is shipping when all five hold:

1. **Multi-agent throughput.** A user goal that would take one agent 30 minutes serial completes in under 10 minutes when decomposed across the swarm. Measured on at least three real workflow templates (research-then-write, audit-then-fix, monitor-then-triage).

2. **Cross-tool agents work as one team.** Claude Code, Codex CLI, and at least one local LLM can be assigned subtasks of a single goal and their outputs merge into one coherent result. No babysitting required.

3. **Audit + replay.** For any completed goal, the user can see who did what, in what order, with what cost, and replay any agent's contribution for debugging. This is not optional — without it the system is unauditable and untrustworthy.

4. **Cost stays sane.** Multi-agent execution does not exceed 2x the cost of running the best single agent for the same goal. Measured per-goal across the same three workflow templates above. (Naive multi-agent often costs 5-10x.)

5. **Kill switch works.** User can halt any agent or the entire swarm in under 1 second. No agent can take an irreversible action (file delete, git push, message send, payment) without explicit permission gate clearance.

## In scope

- **Local-first orchestration** on a single machine, single user. Daemon runs on user's machine; agents (local or remote) plug in.
- **Cross-vendor agent brokering** — the headline feature. Adapter layer normalizes auth, streaming, tool format, billing units, rate limits, latency profiles, and capability declarations across:
  - **Local agents:** Claude Code (MCP), Codex CLI (subprocess), local LLMs via llama.cpp/ollama (HTTP), user's own specialist agents (SDK).
  - **Cloud-rented agents:** Devin (webhook + polling), per-task agent rentals as the market emerges.
  - **Vendor-hosted agents:** ChatGPT/Operator, Cursor agents, custom GPTs, Claude API direct.
- **Coordination layer**: agent identity + capability registry, typed comm bus, task queue with claim/release semantics, supervisor/scheduler, conflict arbiter.
- **Permission + safety model**: per-agent permission gates, irreversible-action blocks, kill switch, full audit log.
- **Cost ledger**: unified $-denominated tracking across heterogeneous billing units (tokens, requests, compute-seconds, per-task fees). Pre-execution cost estimate, post-execution actual, drift bounded.
- **Observability**: per-agent cost tracking, per-task latency, full message replay, eval harness for output quality.
- **Integration with machine-memory** as the shared-state substrate. Files, activity, wiki all flow through `mmd`.
- **Voice/ambient interface** as the eventual primary surface (long-term, post-v1).
- **Native daemon in Go** (Rust as alt if Rust expert on team). Not Node, not Python — see `00-vision.md` §"Why this requires Go".

## Out of scope (explicitly)

- **Building new LLMs.** The OS uses existing frontier + local models as black boxes. No model training.
- **The daemon itself running in the cloud.** Single-user local-first only. The OS *talks to* cloud-hosted agents via adapters (Claude API, OpenAI API, Devin, etc. — that's the whole product). But the OS *daemon* never runs on someone else's machine; that would just make it another vendor garden, defeating the point.
- **Replacing existing agent tools.** Claude Code stays Claude Code. The OS coordinates them, does not replace them. Adapters are thin wrappers, not reimplementations.
- **General-purpose chat UI.** The interface layer routes intents to agents; it is not a competing chat product. Conversation already happens inside the constituent agents.
- **General AGI.** See `00-vision.md` §"What this is NOT". Outcome is JARVIS-feel for narrow domains, not a smarter underlying model.
- **Becoming a vendor of agents.** The OS is a broker, not a marketplace. The user picks which agents to connect; the OS coordinates them.

## Non-goals (deliberately not pursued)

- **Fully autonomous agents without human oversight.** Kill switch + audit + permission gates are required at every layer. The user is always-in-the-loop for irreversible actions.
- **Cross-machine agent coordination** (initial versions). Single machine only. Multi-machine is a possible Phase 4+ extension once the single-machine version is solid.
- **Agent marketplace / third-party agent ecosystem.** The user's own agents only. Building a "store" of community agents is a separate product.
- **Optimization of the swarm via reinforcement learning.** Heuristics + cost-aware routing only. No ML on top of the orchestrator. (Could add later but adds debug complexity that is not worth it for v1.)

## What machine-memory contributes (and why it is not enough on its own)

`machine-memory` (sibling repo) provides:

- File index with FTS5 (`mm_find`)
- Per-file metadata + activity timeline (Phase 2 planned)
- Auto-compiled wiki of projects/concepts/people (Phase 3 planned)
- Live event stream via `mm_subscribe` (Phase 4 planned)

Agent OS uses ALL of these as substrate. But machine-memory does NOT include:

- Agent identity, capability registry, online/offline state
- Inter-agent message routing
- Task ownership + claim semantics
- Supervisor/scheduler logic
- Permission gates + audit log
- Cost tracking + budget enforcement
- Eval harness

Those are this product's job. Agent OS = coordination on top, machine-memory = state below.

## What success looks like in plain language

The user types or says "research the local rental market in Dallas, build me a comparison table, draft an email to my agent, and queue it for review." Within 5 minutes:

- A research agent (specialist scraper + web search) has pulled 30 listings.
- A data agent (Codex CLI or Claude Code) has built the comparison.
- A writing agent (Claude) has drafted the email.
- The OS surfaces the draft + table to the user for review with full audit ("research-agent-1 ran 23 web fetches over 2m17s costing $0.14, data-agent ran 4 sql queries on the listings table over 18s costing $0.03, writing-agent generated 412 tokens over 9s costing $0.02, total $0.19").
- User approves; OS sends the email after one explicit confirmation gate.

That is the JARVIS-feel target. Not magic — a well-engineered coordination layer over agents that already exist on the machine.
