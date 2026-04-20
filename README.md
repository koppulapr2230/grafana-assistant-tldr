# Radar

Claude Code skills for triaging Grafana Cloud.

## grafana-tldr

Point it at a dashboard — get back a TLDR, a top-3 focus list, and ranked root-cause hypotheses grounded in Sift.

### Setup

1. **Install `uv`** (one-time, provides `uvx`):
   ```
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. **Create a Grafana Cloud service account token**
   Grafana → Administration → Users & Access → Service accounts → Add.
   Role: Viewer is enough (Editor if you want the skill to run datasource queries, which it will).
   Copy the generated token — you won't see it again.

3. **Set env vars.** Copy `.env.example` to `.env`, fill in, then before launching Claude:
   ```
   set -a && source .env && set +a && claude
   ```
   Or export them permanently in your shell rc.

4. **Verify.** Inside Claude: `/mcp` — you should see `grafana` listed with tools like `search_dashboards`, `get_dashboard_summary`, `query_prometheus`, `find_slow_requests`.

### Use

```
/grafana-tldr <dashboard-uid>
/grafana-tldr https://<stack>.grafana.net/d/<uid>/<slug>
/grafana-tldr "API latency overview"
```

### What's here

- `.mcp.json` — registers the Grafana MCP server (`uvx mcp-grafana`). Token is sourced from env, never committed.
- `.claude/skills/grafana-tldr/SKILL.md` — the skill definition.
- `.env.example` — template for the two env vars.

### Notes

- The skill is read-only. Write tools (`update_dashboard`, `create_incident`, etc.) are exposed by the MCP server but the skill avoids them.
- Some MCP tools (ClickHouse, CloudWatch, Elasticsearch, `run_panel_query`, `search_logs`, admin) are disabled by default in mcp-grafana. Enable them via `--enabled-tools` in `.mcp.json` if you need them.
- Sift (`find_slow_requests`, `find_error_pattern_logs`) is the grounded-root-cause layer — available on Grafana Cloud.
