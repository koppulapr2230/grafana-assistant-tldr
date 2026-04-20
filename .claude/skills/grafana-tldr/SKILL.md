---
name: grafana-tldr
description: Triage a Grafana Cloud dashboard and produce a TLDR with focus areas and root-cause hypotheses. Trigger when the user asks to summarize, analyze, triage, or investigate a Grafana dashboard (by UID, URL, or name).
---

# grafana-tldr

Analyze one Grafana Cloud dashboard and return a scannable brief with what's wrong right now and why. Uses the `grafana` MCP server (configured in `.mcp.json`). All tools listed below are read-only and enabled by default in mcp-grafana.

## Resolve the target

The user gives a UID, a Grafana URL, or a dashboard name.

- URL form `https://<stack>.grafana.net/d/<uid>/<slug>` — extract `<uid>`.
- Name — call `search_dashboards` with the name as the query; if >1 match, list the top 5 (title + folder) and ask which.
- Bare UID — use as-is.

## Gather

Call these in parallel when possible:

1. `get_dashboard_summary` — cheap structural overview (panel count, variables, time range).
2. `get_dashboard_panel_queries` — per panel: title, datasource uid, raw query, panel id.
3. `get_dashboard_property` with path `panels[*].fieldConfig.defaults.thresholds` — threshold config per panel.
4. `list_datasources` — to map datasource uids → type (prometheus/loki/other).
5. `alerting_manage_rules` (list action) — alert rules; filter to those referencing this dashboard's uid or sharing a folder.
6. `list_incidents` with status `active` — any open incidents.

## Measure

For each panel whose datasource is prometheus or loki and whose query is non-empty:

- Prometheus: `query_prometheus` with `queryType: range`, `start: now-1h`, `end: now`, `step: 60s`. Also `query_prometheus` with `queryType: instant` for the current value. For p95/p99 `histogram_quantile` queries, prefer `query_prometheus_histogram` — it handles the buckets correctly.
- Loki: `query_loki_logs` over `now-1h`, plus `query_loki_stats` for volume. For error-shape panels, use `query_loki_patterns` to cluster.

Skip `text`, `row`, `news`, `dashlist`, `table` (when no query), and `stat` panels whose query reduces to a constant.

If a panel has >20 queries or a query is obviously expensive (wide label matcher, no time bound), skip and note "not measured — query too broad."

## Tag each panel

- `breaching` — current instant value is past the panel's configured threshold (direction matters: red-above vs red-below).
- `large_delta` — current value differs from the previous-hour average by >20% (or >3σ for noisy series).
- `alert_correlated` — a firing alert rule's labels match this panel's query label selectors (service, job, instance, cluster, namespace).
- `incident_correlated` — an active incident's labels match.

A panel with none of these tags is "quiet."

## Root-cause layer (Sift)

Grafana Cloud's Sift already does grounded root-cause analysis. Use it:

1. Pull labels from breaching/alert-correlated panels (especially `service`, `namespace`, `cluster`).
2. `find_slow_requests` with those labels over `now-1h` — calls out latency outliers.
3. `find_error_pattern_logs` with those labels — surfaces new/rising error patterns.
4. `list_sift_investigations` — if an investigation already exists for this incident, include its link; don't duplicate the work.

Treat Sift output as evidence, not conclusion. Cite it in the hypothesis, don't paraphrase as fact.

## Deeplinks

Call `generate_deeplink` for every panel cited in output and for the dashboard itself. Include links in the evidence table.

## Output shape

```markdown
## TLDR
One paragraph. Plain English. State right now, whether any SLO/threshold is breached, whether anything is actively firing.

## Focus
1. **<Panel title>** — <current> (threshold <x>), <delta vs 1h ago>. [link]
2. ...
3. ...
(Max 3. If nothing breaching or large-delta, say "nothing red" and skip.)

## Root-cause hypotheses
Ranked. Each entry:
- **<Hypothesis>** — supported by: <the correlation>. Evidence: <Sift finding or metric fact>. Confidence: low/medium/high.
Cap at 3. If evidence is thin, say so — do not invent.

## Evidence
| Panel | Current | Threshold | Tags | Alert | Link |
|-------|---------|-----------|------|-------|------|
...

## What I didn't check
Panels skipped and why (one line each). Active incidents not correlated. Anything gated behind a disabled tool.
```

## Guardrails

- Don't invoke write tools (`update_dashboard`, `create_incident`, `alerting_manage_rules` with create/update actions, annotations create/update). This is a read-only skill.
- Don't exceed ~30 datasource queries per run; if the dashboard is huge, sample the breaching/alert-correlated panels first and explicitly say you sampled.
- Never fabricate values. If a query returns empty or errors, mark it explicitly and move on.
- Token budget: prefer `get_dashboard_summary` + `get_dashboard_panel_queries` over raw `get_dashboard_by_uid`. Only fetch the full dashboard JSON if you need something the summary didn't include.
