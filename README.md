![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | AI Solution Architecture

## Overview

You will not write code today. You will design a system. Pick one of three business scenarios, sketch the architecture, justify the pattern you chose, and capture one consequential decision as an ADR.

This is a 90-minute exercise. The deliverable is three small artifacts in a repo — a diagram, a one-page write-up, and an ADR. The goal is to prove you can move from a vague business prompt to a defensible system design.

## Learning Goals

By the end of this lab you should be able to:

- Translate a business problem into a serving pattern (batch / online / streaming) with explicit justification
- Produce a clean architecture diagram that names the moving parts and the contracts between them
- Capture an architectural trade-off as an ADR that another engineer could read in 60 seconds

## Setup

Fork this repo, clone it, and work on a branch. No language toolchains required. You will need:

- A Markdown editor
- A diagramming tool — **any** of these is fine:
  - [Mermaid](https://mermaid.js.org/) inside Markdown (recommended — version-controlled, renders on GitHub)
  - [draw.io](https://app.diagrams.net/) (export PNG + the `.drawio` source)
  - [Excalidraw](https://excalidraw.com/) (export PNG + the `.excalidraw` source)

## Pick a Scenario

Choose **one**. Don't switch halfway through.

### Scenario A — Real-time fraud scoring (payments app)

A consumer payments app needs to score every transaction for fraud risk before the user sees confirmation. ~300 transactions/second at peak, p95 latency budget 80 ms end-to-end. Account history and device fingerprint must be available to the model. Block / allow / step-up-auth are the three downstream actions.

### Scenario B — Weekly churn predictions (B2B SaaS)

A SaaS sales team wants a ranked list of accounts at risk of churning, delivered every Monday morning. ~120,000 accounts. The list feeds a CRM dashboard and triggers outreach playbooks. The team will revisit the list throughout the week but the underlying score only needs to refresh weekly.

### Scenario C — Image moderation (UGC platform)

A user-generated content platform needs to moderate uploaded images for prohibited content. ~2 uploads/second average, 20/second peak, p95 latency budget 600 ms. Some content goes straight through; ambiguous cases must escalate to a human reviewer queue. Mistakes have legal weight.

## Tasks

### Task 1 — Architecture diagram

Produce a diagram (`architecture.md` with Mermaid, or `architecture.png` + source file) that shows:

1. **All components** your system needs — ingestion, storage, feature lookup, training, registry, serving, monitoring, downstream consumers. Name them. Don't draw a single "model" box.
2. **The arrows between them**, labeled with what flows (request, event, batch write, schema, signal).
3. **Where the serving boundary is** — what's online, what's batch, what's offline.
4. **The feedback loop** — how does monitoring output feed retraining?

A clean diagram has fewer than ~15 boxes. If you have 30, you're drawing every microservice; zoom out.

### Task 2 — Justification write-up

Write `JUSTIFICATION.md` (one page, ~500 words) answering:

- **Which serving pattern did you pick** (batch / online / streaming) and **why**? Tie the choice to specific lines in the scenario.
- **Where does inference run** (cloud / edge / hybrid)? Why?
- **What are your latency, throughput, and cost targets** — pick two to optimize, name one as the budget.
- **What's the fallback** when the model is unavailable or wrong?

Be specific. "It needs to be fast" is not an answer. "p95 must stay under 80 ms because the user is staring at a payment screen" is.

### Task 3 — One ADR

Pick the **single most consequential trade-off** in your design and capture it as `adr/0001-<short-slug>.md`. Use this five-section format:

```markdown
# ADR 0001: <Title — what you decided>

## Context
Two sentences on what's happening in the system that forces a decision.

## Decision
One paragraph on what you chose.

## Alternatives rejected
2–3 bullets on what you considered and why you said no.

## Consequences
2–3 bullets on what this commits you to (good and bad).

## Revisit if
One bullet — what change in assumptions would make you reopen this.
```

Examples of decisions worth ADR-ing for these scenarios:
- *Run an edge pre-filter for image moderation* — rejected: cloud-only (latency); rejected: edge-only (model size).
- *Use streaming over online API for fraud* — rejected: online (need joins with live enrichment).
- *Refresh churn scores weekly, not daily* — rejected: daily (no business consumer; doubles compute).

## Submission

Open a Pull Request to the lab repository with these files:

```
architecture.md             # or architecture.png + source
JUSTIFICATION.md
adr/0001-<short-slug>.md
```

Paste the PR link as your deliverable.

## Quality bar

You will be reviewed on:

- **Does the diagram name the components and contracts**, or is it three boxes labeled "data," "model," "users"?
- **Is the serving pattern justified by the scenario**, or is it "I picked online because online is good"?
- **Does the ADR capture a real trade-off** that another engineer could push back on, or is it generic?

Three small artifacts. Make them sharp.
