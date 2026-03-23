---
name: cf-account-status
description: "Cloudflare account dashboard — all resources, health, and cost estimates. Workers, Durable Objects (SQL rows read, OOM crashes, instance sprawl), D1, R2, KV, Queues. Use when asked to 'check cloudflare', 'cf status', 'account overview', 'DO costs', 'billing check'."
argument-hint: "[setup] [health:1h] [cost:7d] [--json] [--silent]"
---

# Cloudflare Account Status

Full account dashboard: resource inventory, health alerts, Durable Object (DO) usage metrics, and estimated spend. Cost figures are derived from GraphQL analytics usage data x published pricing — they are **estimates**, not invoice-grade billing. Actual invoiced amounts may differ due to credits, discounts, or invoice cycle alignment.

## Installation

1. Copy this skill to your Claude Code skills directory:
   - **Per-user:** `~/.claude/skills/cf-account-status/SKILL.md`
   - **Per-project:** `.claude/skills/cf-account-status/SKILL.md`
2. Connect the **Cloudflare API MCP server** (see below)
3. Run `/cf-account-status`

## Prerequisite: Cloudflare API MCP Server

Before running queries, check if `mcp__cloudflare-api__execute` is available. If not, tell the user:

> Add to your `.mcp.json` (project or global):
> ```json
> { "mcpServers": { "cloudflare-api": { "type": "url", "url": "https://mcp.cloudflare.com/mcp" } } }
> ```
> Restart Claude Code — it will prompt you to authenticate via browser (OAuth). No API tokens or credentials needed.
>
> Docs: https://developers.cloudflare.com/agents/model-context-protocol/mcp-servers-for-cloudflare/

---

## How to run

1. Parse args. Two independent windows:
   - **`health:Xh`** (default `1h`): worker errors, CPU, DO request errors
   - **`cost:Xd`** (default `7d`): DO usage metrics and spend estimate. Cost queries use date-granularity (YYYY-MM-DD), so hours are rounded up to the nearest day (e.g., `cost:12h` → 1 day).
   - Both always use their prefix. Omitted = use default (or config value).
   - Examples: `/cf-account-status` → health:1h cost:7d. `/cf-account-status health:4h cost:30d`. `/cf-account-status health:12h cost:14d`.
   - Health accepts `Nh` or `Nd`. Cost accepts `Nd` (or `Nh` rounded up). Max retention: ~90 days.
2. **Get current time first.** Run `date -u +%Y-%m-%dT%H:%M:%SZ` or `new Date().toISOString()` via MCP. Do NOT guess. Then compute:
   - `HEALTH_END` = now, `HEALTH_START` = now minus health window
   - `COST_END_DATE` = today (`YYYY-MM-DD`), `COST_START_DATE` = today minus cost window
3. Run queries via `mcp__cloudflare-api__execute` — **max 3 parallel** to avoid rate limiting. GraphQL first, then REST.
4. For **query 2 (DO usage)**: use `START_DATE` / `END_DATE` as the filter — same window as everything else. For short windows (< 24h), the DO daily query may return just 1-2 days of data, which is fine.
5. **After all queries complete, tell the user:** "Data collected. Analyzing [N] queries across [resources found]. Building report..." — this prevents the user from thinking the skill is stuck during the analysis phase.
6. Combine results and present as interpreted summary (see Output section).

**Note on query code blocks:** The JavaScript functions below are templates for `mcp__cloudflare-api__execute`. The `accountId` variable and `cloudflare.request()` method are provided by the MCP server runtime — they are not defined in this skill.

---

## Queries

### 1. Workers — requests, errors, CPU (GraphQL, health window)

```javascript
async () => {
  const query = `query { viewer { accounts(filter: {accountTag: "${accountId}"}) {
    workersInvocationsAdaptive(limit: 100, filter: {
      datetime_geq: "${START_TIME}", datetime_leq: "${END_TIME}"
    }) {
      sum { requests errors subrequests }
      quantiles { cpuTimeP50 cpuTimeP99 }
      dimensions { scriptName }
    }
  }}}`;
  const r = await cloudflare.request({ method: "POST", path: "/graphql", body: { query } });
  const rows = r.result?.viewer?.accounts?.[0]?.workersInvocationsAdaptive || [];
  const m = {};
  for (const d of rows) {
    const n = d.dimensions.scriptName;
    if (!m[n]) m[n] = { name: n, requests: 0, errors: 0, cpu_p99: 0 };
    m[n].requests += d.sum.requests;
    m[n].errors += d.sum.errors;
    m[n].cpu_p99 = Math.max(m[n].cpu_p99, d.quantiles.cpuTimeP99);
  }
  return Object.values(m).sort((a, b) => b.requests - a.requests);
}
```

### 2. DO usage — SQL rows, duration, OOM crashes (GraphQL, cost window with daily breakdown)

The most important query. `durableObjectsPeriodicGroups` has 14 fields for usage metrics. CF docs only show `cpuTime` as an example — the full field list is available via GraphQL schema introspection.

```javascript
async () => {
  const query = `query { viewer { accounts(filter: {accountTag: "${accountId}"}) {
    durableObjectsPeriodicGroups(
      filter: {date_geq: "${LAST_30D_START}", date_leq: "${TODAY}"}, limit: 10000
    ) {
      sum {
        rowsRead rowsWritten duration activeTime cpuTime
        exceededCpuErrors exceededMemoryErrors fatalInternalErrors
        storageReadUnits storageWriteUnits storageDeletes
        subrequests inboundWebsocketMsgCount outboundWebsocketMsgCount
      }
      dimensions { namespaceId date }
    }
  }}}`;
  const r = await cloudflare.request({ method: "POST", path: "/graphql", body: { query } });
  return r.result?.viewer?.accounts?.[0]?.durableObjectsPeriodicGroups || [];
}
```

Group results by `namespaceId`, then compute subtotals for `date >= LAST_24H_START`, `date >= LAST_7D_START`, and all rows (30d).

**Response size warning:** The CF MCP proxy caps responses at ~6K tokens. 30 days x multiple namespaces can exceed this, causing truncation. If the response appears incomplete (missing days or namespaces), split into per-namespace queries or two 15-day ranges.

### 3. DO requests — errors (GraphQL, health window)

```javascript
async () => {
  const query = `query { viewer { accounts(filter: {accountTag: "${accountId}"}) {
    durableObjectsInvocationsAdaptiveGroups(
      filter: {datetime_geq: "${START_TIME}", datetime_leq: "${END_TIME}"}, limit: 100
    ) { sum { requests errors } dimensions { scriptName } }
  }}}`;
  const r = await cloudflare.request({ method: "POST", path: "/graphql", body: { query } });
  return r.result?.viewer?.accounts?.[0]?.durableObjectsInvocationsAdaptiveGroups || [];
}
```

### 4. DO storage — account-wide total (GraphQL)

`durableObjectsStorageGroups` only supports `date`/`datetimeHour` dimensions — no per-namespace breakdown. Requires a date filter (empty `{}` is rejected). Note: this metric tracks legacy KV-style DO storage (`storage.put()`/`storage.get()`). DOs using SQLite storage (the `sql` API, including Agents SDK DOs) may report 0 here even when they have data. The instance `hasStoredData` flag from query 5 is more reliable for detecting SQLite-based DO storage.

```javascript
async () => {
  const query = `query { viewer { accounts(filter: {accountTag: "${accountId}"}) {
    durableObjectsStorageGroups(filter: {date_geq: "${LAST_7D_START}", date_leq: "${TODAY}"}, limit: 1) {
      max { storedBytes }
    }
  }}}`;
  const r = await cloudflare.request({ method: "POST", path: "/graphql", body: { query } });
  const groups = r.result?.viewer?.accounts?.[0]?.durableObjectsStorageGroups || [];
  return { total_bytes: groups[0]?.max?.storedBytes || 0 };
}
```

### 5. DO namespaces + instance counts (REST)

Instance list is capped at 1000 per namespace. If the count equals 1000, show "1000+" in the output.

```javascript
async () => {
  const ns = await cloudflare.request({
    method: "GET", path: `/accounts/${accountId}/workers/durable_objects/namespaces`,
    query: { per_page: 100 }
  });
  const results = [];
  for (const n of (ns.result || [])) {
    const inst = await cloudflare.request({
      method: "GET",
      path: `/accounts/${accountId}/workers/durable_objects/namespaces/${n.id}/objects`,
      query: { limit: 1000 }
    });
    const objects = inst.result || [];
    results.push({
      id: n.id, name: n.name, class: n.class, script: n.script,
      instances: objects.length,
      with_storage: objects.filter(o => o.hasStoredData).length
    });
  }
  return results;
}
```

### 6-10. Resource lists (REST)

```javascript
// 6. D1
async () => {
  const r = await cloudflare.request({ method: "GET", path: `/accounts/${accountId}/d1/database`, query: { per_page: 50 } });
  return (r.result || []).map(d => ({ name: d.name, size_bytes: d.file_size || 0, tables: d.num_tables || 0 }));
}

// 7. R2
async () => {
  const r = await cloudflare.request({ method: "GET", path: `/accounts/${accountId}/r2/buckets`, query: { per_page: 50 } });
  return (r.result?.buckets || r.result || []).map(b => ({ name: b.name }));
}

// 8. KV
async () => {
  const r = await cloudflare.request({ method: "GET", path: `/accounts/${accountId}/storage/kv/namespaces`, query: { per_page: 100 } });
  return (r.result || []).map(n => ({ title: n.title, id: n.id }));
}

// 9. Queues
async () => {
  const r = await cloudflare.request({ method: "GET", path: `/accounts/${accountId}/queues`, query: { per_page: 50 } });
  return (r.result || []).map(q => ({ name: q.queue_name || q.name }));
}

// 10. Pages — SKIPPED. The CF MCP OAuth token does not include Pages scope.
// Use `wrangler pages project list` or CF Dashboard instead.
```

---

## Output Modes

### (default) — full report

`/cf-account-status` or `/cf-account-status health:4h cost:14d`

**Present results as a formatted markdown summary to the user.** See "Report: What to Include" below.

### `setup` — configure thresholds

`/cf-account-status setup`

Run a full report, then offer to create `.claude/cf-account-status.local.md` with recommended thresholds based on actual usage. Can be combined with windows: `/cf-account-status setup cost:30d`.

### `--json` — structured output for other skills

`/cf-account-status --json`

**Return structured JSON only** (no prose). For piping to other skills/workflows. Schema:

```json
{
  "timestamp": "ISO8601",
  "health_window": "1h",
  "cost_window": "7d",
  "cost_estimate_usd": N,
  "alerts": [{ "level": "warn|critical", "type": "...", "message": "..." }],
  "workers": { "count": N, "requests": N, "errors": N },
  "durable_objects": { "namespaces": [...], "totals": {...} },
  "d1": [...], "r2": [...], "kv": [...], "queues": [...], "pages": [...]
}
```

### `--silent` — watchdog mode for `/loop`

`/loop 24h /cf-account-status --silent`

**Output NOTHING if all metrics are below their configured warn thresholds** (both cost and health). Only speak up when any threshold is exceeded. If no config file exists, uses defaults. Run `/cf-account-status setup` first to calibrate.

---

## Configuration (optional)

To customize thresholds, create `.claude/cf-account-status.local.md`:

```yaml
---
# Default windows (overridden by CLI args)
health_window: 1h               # worker errors, CPU, DO request errors
cost_window: 7d                 # DO usage metrics, spend estimate

# Cost thresholds (USD)
cost_24h_warn: 1.00
cost_24h_critical: 10.00
cost_7d_warn: 7.00
cost_7d_critical: 50.00
cost_30d_warn: 25.00
cost_30d_critical: 100.00

# DO SQL rows read
do_rows_24h_warn: 1000000        # 1M/day
do_rows_24h_critical: 100000000  # 100M/day
do_rows_7d_warn: 7000000         # ~1M/day avg
do_rows_7d_critical: 500000000   # 500M/week
do_rows_30d_warn: 25000000       # 25M/30d
do_rows_30d_critical: 1000000000 # 1B/30d

# DO instances per namespace
do_instances_warn: 500
do_instances_critical: 5000

# DO memory exceeded errors (OOM)
do_oom_24h_warn: 1               # any OOM is a warn
do_oom_7d_critical: 10           # sustained crashing

# Worker health (applies to health window)
worker_error_rate_warn: 10       # percent
worker_error_rate_critical: 50
worker_cpu_p99_warn: 5000        # ms
worker_cpu_p99_critical: 25000

# --silent mode uses the warn thresholds above.
# If all metrics are below *_warn values, --silent produces no output.
---
```

### Where to put it

| Install type | Config location | Scope |
|---|---|---|
| **Per-project** | `.claude/cf-account-status.local.md` | This project only |
| **Per-user** | `~/.claude/cf-account-status.local.md` | All projects |

Project-level takes precedence. Add to `.gitignore` (account-specific preferences).

### First-time setup

If no config exists on the first run, **after showing the report**, offer to create one:

> "Want to set up alert thresholds? I can create a config based on the usage I just measured — warning at 2x current, critical at 10x. Say 'yes' to create the config file."

Generate from actual data:
- `cost_30d_warn` = current 30d cost x 2 (min $1)
- `cost_30d_critical` = current 30d cost x 10
- `cost_24h_warn` = cost_30d_warn / 30
- `do_rows_30d_warn` = current 30d rows x 2
- `do_instances_warn` = current max instances x 2 (min 50)

### Reading config

Check project-level first, then user-level. Read YAML frontmatter. Missing fields fall back to defaults above.

---

## Cost Calculation

Compute estimated spend for the **cost window** from daily data. Shows raw usage cost (usage x pricing). The "included" allowances apply at CF's monthly invoice level — this skill shows usage cost for burn rate visibility.

**Durable Objects pricing (Workers Paid plan, verified 2026-03-23 — [check current](https://developers.cloudflare.com/durable-objects/platform/pricing/)):**

| Metric | Included | Overage |
|---|---|---|
| Requests | 1M/month | $0.15/M |
| Duration | 400K GB-sec/month | $12.50/M GB-sec |
| Rows read | 25B/month | $0.001/M |
| Rows written | 50M/month | $1.00/M |
| Storage | 1 GB | $0.20/GB-month |

**Workers pricing (verified 2026-03-23 — [check current](https://developers.cloudflare.com/workers/platform/pricing/)):**

| Metric | Included | Overage |
|---|---|---|
| Requests | 10M/month | $0.30/M |
| CPU time | 30M ms/month | $0.02/M ms |

**Estimated spend (for the requested timeframe):**

Show a single cost figure: `DO estimated spend (last Xh/Xd): $Y.YY`

Workers request costs are not computed (the GraphQL query returns request counts and CPU percentiles, not billable CPU millisecond totals). Workers on the Paid plan have generous included usage (10M requests, 30M CPU-ms/month). If Workers cost matters, check the CF dashboard directly.

---

## Report: What to Include

1. **Estimated spend** (for the cost window) — first thing users see
2. **Alerts** — anything exceeding configured thresholds (warn/critical)
3. **Resource inventory** — Workers, DOs, D1, R2, KV, Queues, Pages counts
4. **DO usage table** — per-namespace: rows read/written, duration, OOM, instances (last 30d)
5. **Worker health** — only flag workers with errors or high CPU (health window)
6. **Caveats** (always mention):
   - "Usage data excludes deleted DO namespaces — usage from deleted DOs is still billed but not returned by the API."
   - "CF does not currently support billing alerts for DO SQL rows read."
7. If any resource type has **zero entries**, show `0` in the inventory line but skip its detail section.

### Example

```
## CF Account Status (last 7d)

DO estimated spend: **$2.94**

### Alerts
- DO instances: RateLimiterDO — 1000+ instances (threshold: 500)

### Resources
Workers: 4 | DOs: 2 | D1: 2 | R2: 1 | KV: 3 | Queues: 1 | Pages: 0

### Durable Objects (last 7d)
| Namespace      | Rows Read | Written | Duration (GB-s) | OOMs | Instances |
|----------------|-----------|---------|-----------------|------|-----------|
| ChatRoomDO     | 1.9M      | 48K     | 2,800           | 0    | 15        |
| RateLimiterDO  | 95K       | 19K     | 130,000         | 0    | 1000+     |

Note: Estimates based on GraphQL analytics, not invoice data. Excludes deleted namespaces.

### Workers (last 7d)
4 workers, 8,680 requests, 0 errors.
```

---

## Error Handling

Queries may fail. **Degrade gracefully** — report what succeeded, flag what failed.

| Error | Cause | Behavior |
|---|---|---|
| `Authentication error 10000` | CF MCP OAuth token lacks scope for that endpoint. | Report "&lt;service&gt;: auth error" — don't retry |
| `429 Too Many Requests` | Too many parallel calls through one OAuth session | Retry once after 2s. If still 429, report "rate limited" |
| `unknown field "X"` | GraphQL field doesn't exist on that dataset | Skip that query, note in output |
| MCP tool not found | `mcp__cloudflare-api__execute` unavailable | Show setup instructions (see Prerequisite) |

**Rate limit prevention:** Max 3 parallel queries. GraphQL first, then REST.

## Limitations

- **Deleted DO namespaces invisible** — usage is billed but GraphQL returns nothing for deleted namespaces
- **DO usage fields not in CF docs** — `durableObjectsPeriodicGroups` has 14 fields; CF docs only show `cpuTime` as an example. Full list available via GraphQL schema introspection.
- **DO storage: no per-namespace breakdown** — `durableObjectsStorageGroups` only supports `date`/`datetimeHour` dimensions
- **DO instance list capped at 1000** per namespace via REST API
- **No DO billing alerts on CF platform** — D1 has billing notifications; DOs do not
- **D1 row-level analytics not queried** — D1 has its own GraphQL dataset (`d1AnalyticsAdaptiveGroups`) with rows read/written; not included in this skill yet
- **GraphQL analytics retention: ~90 days** — aggregate metrics only. Per-request logs (Workers Observability) have 7-day retention on paid plans.
- **MCP response token limit (~6K tokens)** — large GraphQL responses (30d x many namespaces) may be truncated. Split into per-namespace or shorter time ranges if data is incomplete.
- **DO `storedBytes` tracks legacy KV storage only** — `durableObjectsStorageGroups` does not reflect SQLite-based DO storage (the `sql` API). The `hasStoredData` flag from the instance REST API is more reliable for SQLite DOs.
- **Pages not supported** — CF MCP OAuth token does not include Pages API scope. Use `wrangler pages project list` or the CF Dashboard instead.
- **`billing/usage/paygo` API (Beta, select accounts)** — would provide actual per-service costs and invoice cycle dates. May return 404 if not enabled for your account. When available, the skill should use it for cost verification. See [CF API reference](https://developers.cloudflare.com/api/resources/billing/subresources/usage/methods/paygo/).
