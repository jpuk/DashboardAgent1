# Improvement pass: Agent Lab dashboards

The initial build is complete and verified. Your job now is ONE small,
verified improvement per session. Nobody will answer questions.

## Tools
- Your Grafana tools are named `mcp__MCP_DOCKER__*`. They connect a few seconds after the session starts. If you don't see them, call `WaitForMcpServers` once, then use them.
- Never look for Grafana by other routes (curl, hostnames, hosts files, config files). The MCP tools are the only way in.

## Each session
1. Read PROGRESS.md and IMPROVEMENTS.md (create IMPROVEMENTS.md if missing,
   with a "## Backlog" and a "## Done" section).
2. If the backlog is empty, review the dashboards and reports and add
   candidate improvements to it, each with one line saying why it helps.
3. Pick the single most valuable backlog item. Do only that.
4. Verify every query you add or change returns data via the MCP tools.
5. Upload with `update_dashboard` using the dashboard's existing uid and
   overwrite: true, and save the JSON to `dashboards/`.
6. Move the item to "## Done" with what changed and the query evidence.
   Commit to git, then stop.

## Good improvements
- Check to see if new metrics have been added and are availbile for use on existing dashboards, add new findings to INVENTORY.md
- Use GTT, not VRAM, in any dashboard. On Strix Halo, “VRAM” is just a small fixed slice of memory reserved in the BIOS, here 1 GB, which is always nearly full. The GPU’s real working memory is GTT, shared system RAM 
- Re-check claims in FINDINGS.md against fresh queries and correct any
  that are wrong (e.g. which oper-down ports actually have an ifAlias)
- Panels for data that exists but isn't shown yet
- Clearer legends (use ifAlias), correct units, sensible thresholds
- Hiding internal interfaces from variables and panels

## Rules
- All hard rules from TASK.md still apply.
- Never delete a dashboard or remove a working panel without recording why.
- Panels for rare events (errors, warnings, security events) can be legitimately empty. Never broaden a filter just to make a panel return data. An empty panel because nothing went wrong is correct; fix only syntax errors and wrong label or metric names.
- When nothing in the backlog is worth doing, write
  "NO FURTHER IMPROVEMENTS" in IMPROVEMENTS.md and stop.