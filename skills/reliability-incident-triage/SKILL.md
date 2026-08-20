---
description: Runbook for triaging a reliability incident in a Trillo Observability deployment — an agent or agent fleet is failing. Walks from "something is failing" to a root-cause class (code vs deployment vs dependency), impacted systems, and a drafted investigation report. Use when the user reports errors, a degraded agent, a failure spike, or asks "why did fleet X degrade".
---

# Runbook: reliability incident triage

Goal: turn "an agent is failing" into a **defensible root cause + blast radius +
recommended action**, then file a report. Reason from **structure, not payload**
(see `sre:sre-overview`). Prefer parallel tool calls where steps are independent.

## Step 1 — Find the incident

1. `get_top_findings` filtered to reliability / `status = open` (and a time window
   if the user gave one) → the ranked failing findings.
2. Pick the finding that matches the user's report (agent, app, time). Note its
   `findingId`.

## Step 2 — Blast radius & root-cause class (the key step)

3. `get_failure_cluster_stats(findingId)` → counts by **status / agent / app** and
   the **time span**. Read the **spread**:
   - failures **spread across many instances / locations** of **one** logical
     agent (often one version) ⇒ **likely CODE** (a regression in the agent's
     definition);
   - failures **concentrated in one instance / location / cluster** ⇒ **likely
     DEPLOYMENT** (bad pod, region-local outage, config drift);
   - the failing node is a **shared tool/system across multiple agents** ⇒
     **likely DEPENDENCY**.
4. `get_impacted_agents(findingId)` → for a DEPENDENCY case, the other agents
   sharing the failing dependency (the multi-agent blast radius).

## Step 3 — Confirm the failing span & dependency

5. From the finding/cluster, take a representative execution and call
   `get_execution(executionId)` → the **span skeleton**; identify the **first
   failing span** (category + status).
6. `get_correlated_logs(traceId|executionId)` → correlated logs/events for that
   failure (error signatures, retries, timeouts).
7. `get_dependency_topology` (or `get_agent_dependency_tree(agentId)`) → confirm
   the failing tool → external system edge and what else depends on it.

## Step 4 — Is it new? (regression check)

8. `get_agent_baseline(agentId, metric=error_rate)` → is this a step change vs the
   baseline? A step aligned to a version boundary supports the **CODE/regression**
   hypothesis from Step 2.

## Step 5 — Conclude & file

9. Synthesize: **likely root cause + class** (code / deployment / dependency),
   **supporting evidence** (spread numbers, first failing span, dependency edge,
   baseline delta), **impacted systems/agents**, **recommended next action**, and
   a **confidence** level. Separate facts (from tools) from inference.
10. Draft it and call `write_investigation_report` (target `AiAnalysis`,
    `analysisType = external_sre_copilot`) so it appears in the Trillo Observability UI. Reference
    the `findingId` in the evidence. Confirm with the user before filing.

## Notes

- If `get_failure_cluster_stats` shows wide cross-instance spread, **lead with the
  code-regression hypothesis** and check the version boundary — that is the
  fleet-scale signal single-trace tools miss.
- Stay in **structure**: you do not need raw prompts/outputs to place a reliability
  root cause. If a step seems to require them, say so rather than assuming content.
