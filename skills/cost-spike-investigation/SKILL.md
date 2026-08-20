---
description: Runbook for investigating a cost or token spike in a Trillo Agent Observability deployment. Walks from "spend went up" to the application/agent/model driving it and a drafted report. Use when the user reports rising cost, a budget alert, a token spike, or asks "why did our AI spend jump".
---

# Runbook: cost-spike investigation

Goal: attribute a spend increase to the **application / agent / model / location**
driving it, and file a report. Reason from **structure, not payload** (see
`tao:tao-overview`).

## Step 1 — Find the cost finding

1. `get_top_findings` filtered to `findingType = cost` (or `tokenEfficiency`),
   `status = open`, and the time window → the cost/spike findings the platform's
   sweepers already produced.
2. Pick the finding that matches the user's report; note its `findingId` and read
   its evidence (which app/agent/model, magnitude, window).

## Step 2 — Ecosystem context

3. `get_executive_health_summary` → the cost posture (MTD, trend, forecast
   context) so you can frame the spike against the whole ecosystem.

## Step 3 — Attribute the increase

4. From the finding evidence, identify the offending **application → agent →
   model**. If the spike aligns with a **version boundary** of an agent, that
   points at a change in the agent's prompt/model (a regression in efficiency).
5. Check whether it's a **volume** increase (more executions — legitimate growth)
   vs. a **per-execution** increase (more tokens/exec, or a pricier model) — the
   latter is the optimization target. Use the finding's rate/per-exec evidence.

## Step 4 — Conclude & file

6. Synthesize: **what is driving the spend** (app/agent/model, volume vs
   per-exec), **supporting evidence**, **recommended action** (e.g. model
   right-sizing, prompt-caching, cap review), **confidence**. Separate facts from
   inference.
7. Draft and `write_investigation_report` (reference the `findingId`); confirm
   with the user before filing.

## Tool gaps (raise with the app team if deeper drill is needed)

The **deep cost tools are not currently exposed** as `agent_tool` — this runbook
works from the cost *findings* + executive summary. To drill into **top token
consumers**, a **cost forecast**, or **per-model aggregates**, the app team would
need to expose these functions (tag `invocationMode: agent_tool`):
`get_top_token_consumers`, `forecast_costs`, `aggregate_costs_and_tokens`. Until
then, reason from the finding evidence; note the limitation in the report if it
blocks a conclusion.
