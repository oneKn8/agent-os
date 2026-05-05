# Agent OS

> Local-first broker for the user's scattered AI agents. The neutral control plane no vendor will build.

## One-line goal

The user's agents are fragmenting across vendors — Claude Code locally, Codex CLI in another terminal, Devin rented at $500/month, custom GPTs in ChatGPT, Cursor in their editor, a local Llama for private work, specialist agents per task. Each vendor builds a walled garden. **Agent OS is the local-first daemon that brokers across all of them so they behave like one coherent team.** JARVIS-grade for the user's daily work, regardless of where individual agents physically live.

## The structural opportunity

```
Anthropic     │ won't ship a broker — wants you in Claude
OpenAI        │ won't ship a broker — wants you in ChatGPT/Operator
Cognition     │ won't ship a broker — wants you in Devin
Cursor        │ won't ship a broker — wants you in Cursor
Every vendor  │ has the same incentive: lock-in
              │
              ▼
              The broker has to come from outside the vendors.
              Local-first is the natural home — anywhere else
              is just another vendor garden.
```

## Why this is a separate product from machine-memory

`machine-memory` (sibling repo at `~/projects/ai/machine-memory`) is the **shared filesystem of knowledge** an agent OS needs. It is necessary but very far from sufficient. The hard parts of multi-agent coordination — identity, communication, task ownership, arbitration, safety, observability — are not solved by good memory. They need their own product.

Stack:

```
┌──────────────────────────┐
│   AGENT OS  (this)       │  coordination, comms, supervision
├──────────────────────────┤
│   machine-memory (mmd)   │  shared state, file index, wiki, activity
└──────────────────────────┘
```

Agent OS depends on machine-memory but is a separate codebase, separate daemon, separate ship cycle.

## Documents

- [`docs/00-vision.md`](./docs/00-vision.md) — full architecture, the boost mechanism, what "hardcore engineering" actually means
- [`docs/01-scope.md`](./docs/01-scope.md) — explicit in-scope / out-of-scope / non-goals
- [`docs/02-roadmap.md`](./docs/02-roadmap.md) — sequencing (24-36 month build to JARVIS-grade)

## Status

**Pre-spec.** Vision captured, scope locked, roadmap sketched. No code yet. Build does not begin until machine-memory is past Phase 3 (wiki compiler), per the sequencing in `02-roadmap.md`.

## Honest framing

"Near AGI" feel is the *outcome* of a working agent OS for narrow domains, not a guaranteed result. A great agent OS will make this machine feel JARVIS-like for software engineering, research, content production — anywhere the agents have strong tool access. It will NOT make the underlying models smarter. The intelligence ceiling is still the best individual model in the swarm. Read [`docs/00-vision.md`](./docs/00-vision.md) §"One thing to be honest about" before getting excited.
