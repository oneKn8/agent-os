# 02 — Roadmap

> 24-36 month build to JARVIS-grade. Real timeline for a real product, not a vibe.

## Sequencing principle

Substrate first, coordination on top. Build the agent OS only after `machine-memory` is past Phase 3 (wiki compiler). Reasons:

- Agents need shared understanding to coordinate around. Wiki gives them that.
- Activity events (machine-memory Phase 2) are required for any timeline-aware coordination ("what happened in the last hour").
- Building coordination on top of an unstable memory layer means refactoring both.

## NEAR (now → 6 months) — finish the substrate

This is `machine-memory` work, not Agent OS work. Listed for sequencing clarity.

- Slice 4 (installer) — so other agents on the user's machine can register `mmd` themselves.
- machine-memory Phase 2 (activity events) — agents need to know what is happening across the machine.
- machine-memory Phase 3 (wiki compiler) — agents need shared understanding to coordinate around.
- SKIP machine-memory Phase 4's `mm_chat` (humans only); KEEP `mm_subscribe` (agents need it).
- SKIP macOS port (per user direction).

**Exit criterion:** `mmd` runs on user's machine 24/7, indexing live, exposing files + wiki + activity + event stream over MCP. Other agents (Claude Code, Codex CLI) can already query it.

## MID (6-12 months) — Agent OS v0.1

Minimum viable swarm. ONE coordination pattern that works end-to-end.

### Phase 1 — Agent identity + roster + first adapter
- `agentd` daemon (Go binary) — separate process from `mmd`, runs as systemd user unit.
- `agent_records` table in shared state.
- Registration RPC: `agent.register({ name, capabilities[], adapter_kind, adapter_config })`.
- Heartbeat + online/offline tracking.
- `agent` CLI: `agent list`, `agent inspect <name>`, `agent register`, `agent rm`.
- **First adapter: Claude API direct.** Proves the adapter SDK shape end-to-end before fanning out to other vendors. Includes auth, streaming response handling, tool-call format translation, cost ledger entry per call.

### Phase 2 — Task queue
- `task_records` table.
- Claim/release semantics (worker claims a task, sends heartbeats while working, releases on completion or timeout).
- Handoff protocol (one task's output becomes another task's input via typed contract).
- Retries on failure with exponential backoff.

### Phase 3 — Comm bus
- Direct addressing (`agent.message(to, payload)`).
- Pub/sub (`agent.subscribe('topic') / agent.publish('topic', payload)`).
- Persistence: every message logged to shared state, replayable.
- Typed contracts: payload schemas registered per topic so consumers know what to expect.

### Phase 4 — Adapter expansion + first cross-vendor pipeline
Add the second and third adapters; prove the broker thesis end-to-end.

- **Second adapter: local LLM via llama.cpp / ollama.** Proves the abstraction handles a vendor with completely different latency, billing (free), and tool-call shape. If the SDK forces compromises here, fix it before going further.
- **Third adapter: a cloud-rented agent (Devin webhook + polling, OR an OpenAI Operator session).** Proves the abstraction handles long-running, async, expensive-per-task agents.
- Supervisor agent (small fast model on the cheapest adapter — local Llama if possible) decomposes a goal.
- Worker agents claim subtasks. **Each subtask is routed to the cheapest agent that meets the capability + quality bar** — this is the cost-arbitrage payoff.
- Reviewer agent (separate model, different vendor, hostile by default) gates merge.
- Full audit trail visible via `agent show <task_id>`. Includes per-vendor cost breakdown.

**Recommended first pipeline: research-plan-implement-review-merge** (workflow already validated for software work). Supervisor on local Llama, plan + implement on Claude Code, review on a different vendor (e.g., Devin or local Llama with hostile prompt) so reviewer-implementer collusion can't happen.

**Exit criterion:** User states a software-engineering goal; the swarm completes it without babysitting; result is auditable; cost is < 2x single-agent baseline; **at least two different vendors participated** (the cross-vendor part is the headline — without it we haven't proven the broker thesis).

## LATE (12-24 months) — supervisor that decomposes general goals

Beyond the one validated pipeline. The supervisor learns to decompose arbitrary goals.

- Capability registry: each agent declares what it can do (research, code, web-fetch, voice, image-gen, etc.) at registration. Supervisor matches tasks to capabilities.
- Cost-aware routing: supervisor picks the cheapest agent that meets the quality bar for each subtask.
- Workflow templates as code: research-then-write, audit-then-fix, monitor-then-triage, etc. New templates added by the user.
- Eval harness: automated quality scoring on a per-task basis (does the output meet the contract). Reputation tracks per-agent eval scores over time; cost-router uses reputation as a tiebreaker.
- Conflict arbiter: when agents disagree (review failed, two researchers found contradictory data), the arbiter applies a resolution strategy (request human, vote, ground-truth check, escalate to higher-tier model).

**Exit criterion:** User states a goal for which no template exists; supervisor decomposes it on the fly; swarm completes it; user audits + approves the workflow as a new reusable template.

## LATER (24m+) — JARVIS

The polish + ambient layer.

- **Voice ambient interface.** TTS/STT agent integrated; user speaks goals naturally; OS responds with spoken updates + asks for confirmation when permission gates fire.
- **Multi-machine.** User's laptop + workstation + occasional cloud agent (a heavy-compute LLM rented for an hour). Sync via shared state replication (mmd already has this property — its DB is replicable).
- **Self-improvement loops.** Workflow templates auto-tune based on observed outcomes (which agent assignment produced the best eval; which decomposition strategy completed fastest). NOT reinforcement learning — heuristic learning only, simpler to debug.
- **Specialist agent SDK.** User builds their own agents (e.g., a domain-specific scraper for their immigration case work) and registers them with the OS in 30 minutes. Agents are user-owned, not platform-owned.

**Exit criterion:** User goes a full work week without typing a single command; ambient voice + push notifications drive the interaction; the OS handles 80% of routine work without prompting.

## What the timeline depends on

- **Frontier model capabilities staying roughly stable.** If GPT-6 / Claude 5 ship a built-in agent OS that subsumes this product, the build dies. Mitigation: focus on the parts no frontier vendor owns (cross-vendor coordination, local LLM integration, user-owned specialist agents).
- **MCP / agent protocol standardization.** If MCP fragments or a new protocol wins, adapters need to evolve. Mitigation: thin adapter layer that can swap underlying protocols without changing the OS core.
- **machine-memory not slipping more than 3 months.** If the substrate slips, this slips with it. Mitigation: clear gating — do not start Agent OS Phase 1 until machine-memory Phase 3 is done.

## What kills the project (honest pre-mortem)

Risks ranked by how likely they are to kill the build, plus the mitigation for each:

1. **Coordination overhead exceeds the boost** (the naive multi-agent failure mode). Mitigation: every Phase has a measured benchmark vs. single-agent baseline; abandon any pattern that does not beat baseline by 2x.
2. **No single agent OS user beyond the developer.** Mitigation: focus on the user's actual workflows from day one, not abstract "what an agent OS should do."
3. **Frontier vendor ships their own agent OS.** Mitigation: bet on cross-vendor + local-first; vendor agent OSes will be vendor-locked.
4. **Cost runs away.** Mitigation: hard budget per goal, kill switch wired into supervisor, eval harness catches "expensive but bad" before user pays.
5. **Permission model fails (agent does something irreversible without consent).** Mitigation: irreversible-action allowlist by default empty; user must explicitly grant permission per agent per action class. Audit log surfaces every permission grant.

## Decision points before building anything

These are the questions that need explicit answers before code starts. Capture them in a future `03-decisions.md`:

- ~~**Daemon language.**~~ **Resolved: Go for v0.1.** Rust acceptable if a Rust expert is on the team day one (see `00-vision.md` §"Why this requires Go" — the workload is wrong shape for Node/Python). Decision is final unless a Rust expert joins; otherwise revisit only if Go GC pauses prove a real problem in a measured benchmark.
- **Comm bus implementation** (in-process channels for v0.1, gRPC/NATS when first remote agent adapter lands)?
- **Storage** (extend `mmd`'s SQLite for agent_records / task_records / message_records vs separate DB)? Bias toward extension — single shared substrate is the whole point.
- **Process model** (one `agentd` daemon vs per-agent supervisor processes)? Bias toward one daemon for v0.1 simplicity; per-agent processes for v0.2 if isolation matters.
- **Adapter SDK shape** — what does writing a new adapter look like? Should be < 200 LOC for a typical vendor + < 1 day from start to working in the supervisor. This is the moat, so the SDK is a first-class deliverable.
- **External-agent authentication** — how does the OS prove identity to vendor APIs (held tokens) AND prove identity TO local agents (so a malicious script can't impersonate the OS to a worker)? Local UDS + filesystem permissions for v0.1; signed tokens for v0.2.
- **Permission UX** (CLI prompts? GUI tray app? Voice confirm?)? Bias toward CLI prompts for v0.1; tray app once non-developers are users.
- **Cost ledger schema** — needs to handle tokens (input/output, cached/fresh), per-request, per-task, compute-seconds, and "free local" all in one normalized $-denominated schema. Decide before any adapter ships.

None of these need answering today. They are the first set of plan-PR discussions when MID phase begins. The language decision being resolved early is deliberate — it unblocks every other technical conversation.
