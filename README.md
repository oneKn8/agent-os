# Agent OS

> Local agent operating system. Connects every AI agent on the machine into a cooperating team.

## One-line goal

Make the swarm of AI agents already running on this machine — Claude Code, Codex CLI, local LLMs, specialist scrapers/voice/vision agents — behave like one coherent assistant that handles open-ended work end-to-end. JARVIS-grade for the user's daily work.

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
