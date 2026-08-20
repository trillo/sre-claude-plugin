# TAO SRE Copilot — User Guide

Investigate a **Trillo Agent Observability (TAO)** deployment from inside Claude
Code. This guide is for the **SRE / on-call engineer** who wants to troubleshoot an
agent-fleet incident by conversation instead of clicking through dashboards.

---

## 1. What it is

TAO watches a fleet of AI agents through their OpenTelemetry data — reliability,
latency, cost, tokens, dependencies, governance. This plugin turns your Claude
Code into an **investigation copilot** over that data: you describe the problem
("why did the checkout agent start failing across the East region at 2am?"), and
Claude runs the right investigation steps against TAO and comes back with a
root-cause writeup you can file.

**Reach for it when** you're triaging an incident, chasing a cost or latency
spike, sanity-checking a canary rollout, or confirming an agent is degrading over
time — and you'd rather stay in your terminal than context-switch to the console.

## 2. What it can and can't do

- **Read-only against the fleet.** It reads findings, failure clusters, traces,
  dependencies, baselines, and health — it never changes telemetry, config, or the
  app.
- **One write:** it can file an **investigation report** back into TAO, which
  shows up in the product UI next to the platform's own analyses. That's the only
  thing it writes, and only when you confirm.
- **Structure, not payload.** The tools return the *shape* of an incident —
  which agent / tool / dependency / version / location / time failed, with counts
  and rates — **not** raw prompts or model outputs. Troubleshooting rarely needs
  the prompt text, and this keeps sensitive content inside your boundary. Your
  RBAC role also scopes what you can see and masks sensitive fields.

## 3. Prerequisites

- A **Trillo AOS account** with access to the TAO deployment, and an RBAC role
  (`admin` / `auditor` / `user` / `owner`) — your role decides what you can read
  and whether you can file reports.
- **Claude Code** (a recent version with plugin support).

## 4. Install & connect

```bash
claude plugin marketplace add trillo/tao-claude-plugin
claude plugin install tao
```

Then authenticate (one time per machine):

1. Run `/mcp`, select **tao**, choose **authenticate**.
2. A browser opens Trillo AOS login. Sign in (and pick the right
   workspace/tenant if prompted).
3. Claude Code stores the tokens; the **tao** connection flips to authenticated.

There's no token file to manage. If a tool ever returns "unauthorized", re-run
`/mcp` → **tao** → authenticate.

> **Endpoint override:** set `TAO_MCP_URL` if your TAO runs at a non-default URL
> (self-hosted / regional). Otherwise the default in `.mcp.json` is used.

## 5. Your first investigation

You don't have to name a tool or a runbook — just describe the problem. Claude
picks the matching runbook and drives it. A typical reliability session:

> **You:** The `order-router` agent has been throwing errors since about 2am, looks
> like it's spreading. Can you find out why?

Claude will (roughly):

1. Look up the open reliability findings and pick the one that matches
   `order-router` around 2am.
2. Read the **blast radius** — how many instances / locations / agents are hit —
   and classify it: **code** (a regression across many instances of one version),
   **deployment** (one bad instance/region), or **dependency** (a shared tool or
   service everyone leans on).
3. Pull a representative execution's **span skeleton** and correlated logs to find
   the first failing step, and walk the **dependency topology** if a shared
   service is implicated.
4. Check the agent's **baseline** to see if this is a new step-change (and whether
   it lines up with a version boundary).
5. Draft an **investigation report** — likely root cause, the evidence it rests
   on, impacted systems/agents, a recommended next action, and a confidence level.

You review the draft; when you say go, Claude files it into TAO with
`write_investigation_report`, and it appears in the product UI.

**Tip:** give it the specifics you already know — agent name, time window, region,
symptom. It narrows the search and improves the report.

## 6. The runbook catalog

Each runbook is a guided investigation. You can invoke one explicitly (e.g.
`/tao:cost-spike-investigation`) or just describe the problem and let Claude choose.

| Runbook | Use it when | Status |
| :-- | :-- | :-- |
| **reliability-incident-triage** | an agent or fleet is failing | full |
| **cost-spike-investigation** | spend or token usage jumped | works (deeper cost drill needs an admin to expose extra tools — §8) |
| **latency-regression-drilldown** | it got slow / a P95–P99 spike | works (deeper latency breakdown needs an admin to expose extra tools — §8) |
| **drift-confirmation** | an agent seems to be "getting worse over time" | partial until the Drift feature ships |
| **canary-go-no-go** | should version B roll out wider? | partial until the A/B Comparison feature ships |

"Partial" runbooks still help today (they compare against baselines); they'll
become one-step calls when the corresponding platform feature lands. Claude will
tell you in-session when it's running a partial flow and why.

Start any session with the **tao-overview** guide if you want Claude to orient
itself first.

## 7. Reading the report

A good investigation report separates **facts** (what the tools returned — counts,
spread, the failing span, the baseline delta) from **inference** (the likely root
cause). It carries a **confidence** level — treat a low-confidence report,
especially one built on a thin data sample (e.g. a fresh canary), as a lead, not a
verdict. The report lands in TAO tagged as copilot-authored, with your name as the
author, so the team can see where it came from.

## 8. Troubleshooting

- **"unauthorized"** → `/mcp` → **tao** → authenticate again.
- **"that tool isn't available"** for a cost/latency deep-dive, drift, or A/B
  comparison → the underlying platform function isn't exposed to the copilot yet.
  That's an **admin/app-team** step (see the appendix); the runbook will fall back
  to what it can do and say so.
- **Empty or thin results** → widen the time window, or double-check the agent /
  app name. If a canary has little traffic, results will be low-confidence by
  nature.
- **Connection not listed** → confirm the plugin is installed
  (`claude plugin install tao`) and, if self-hosted, that `TAO_MCP_URL` is set.

---

## Appendix — for admins enabling the plugin

Two things gate what SREs can do; both live on the TAO side, not in the plugin:

1. **Expose the app.** The TAO app's `appName` must be on the catalog allow-list,
   and only functions marked as agent tools are offered to the copilot. RBAC
   (`allowedAppRoles`) is enforced on every call.
2. **Optional deeper tools.** To power the fuller cost/latency drill-downs (and,
   later, drift and A/B comparison), the app team exposes a few additional
   functions and implements the investigation-report write path. The internal
   specs for this — the tool manifest and the write-path handoff — live in the
   **trillo-observability** repo, not here.

Nothing an admin does requires a plugin change: as TAO exposes more tools, the
copilot picks them up automatically.
