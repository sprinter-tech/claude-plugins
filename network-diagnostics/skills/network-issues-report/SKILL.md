---
name: network-issues-report
description: >
  Generate a report of significant network issues (performance degradation,
  packet loss, latency spikes, mean/variance shifts, DHCP configuration
  changes, rogue DHCP servers) for a network over a specified time range.
  Use when the user asks about network problems, instability, outages,
  incidents, or "what went wrong on this network?"
argument-hint: "[network-name] [time-range]"
allowed-tools: >
  Bash, Read, Write, Grep, Glob, WebSearch,
  mcp__sprinter__list_networks,
  mcp__sprinter__show_network,
  mcp__sprinter__network_issues,
  mcp__sprinter__show_probes,
  mcp__sprinter__ask_user,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__issue_chart
---

Generate a network issues report for $ARGUMENTS

## Scope

This report covers **network-level anomalies** detected by Sprinter's analytics
(loss / latency / variance shifts from probes), DHCP config changes, and
**cable-modem / DOCSIS** plant health. It does **not** cover per-client **Wi-Fi**
link health: there is no `sprinter_wifi_*` anomaly stream here, and this skill
has no `timeseries_*` tool to read those series or grade them against the WiFi
catalog bands. If the user is really asking "is my wireless device's link OK?"
(signal, SNR, retries, channel congestion), that is `Skill(diagnose-wifi-basic)`
/ `Skill(diagnose-wifi-roaming)` — point them there rather than implying a WiFi
finding from anomaly data. For the WiFi metric bands and grading reference, fetch
it via the `get_reference_doc` MCP tool (`name: wifi-metrics-reference`). (Note:
the `snr` / `signal` semantics in the DOCSIS section below are cable-plant MER,
**not** 802.11.)

## MCP Server Availability — Check First

Before starting, verify that the Sprinter MCP tools (`mcp__sprinter__*`) are
available. If any MCP tool call fails with a connection error, authentication
error, or "server disconnected" message, **stop immediately** and tell the
user:

> I cannot proceed because the Sprinter MCP server is unavailable
> (connection failed / requires re-authentication). Please reconnect the
> MCP server and try again.

Do not attempt to work around a failed MCP connection by guessing, using
placeholder data, or skipping MCP-dependent steps.

## Context Management

**Do not shell out to process tool results.** Parse JSON responses inline —
never save MCP tool output to files or pipe it through Python/jq scripts.

## Step 1: Identify the Network

If the user provided a network name or ID (via $ARGUMENTS or earlier in the
conversation), use it. Otherwise:

1. **`list_networks`** — show available networks.
2. **`ask_user`** — if multiple networks exist, ask the user which one.
   Format choices clearly:
   > Which network would you like the issues report for?
   > 1. **home** — 10.0.14.0/24 (46 devices)
   > 2. **vlad** — 192.168.10.0/24 (23 devices)

   Omit networks with zero devices unless the user explicitly asks.

Once you have a `network_id`, proceed to Step 2.

## Step 2: Get Network Details

Call **`show_network`** with the resolved `network_id`. Extract:

- `name` — for the report title
- `timezone` — IANA timezone string (e.g. `America/Los_Angeles`). This is
  the timezone where the network physically exists.
- Agent names and IDs — for correlating issues to agents later.

## Step 3: Timezone Preference

The user needs to choose which timezone the report displays times in. This
matters for MSP engineers who may be in a different timezone than the network.

**Detect local timezone first.** Run:

```bash
date +%Z  # e.g. "PDT"
```

and

```bash
# Get IANA timezone name
if [ -n "$TZ" ]; then echo "$TZ"
elif [ -f /etc/timezone ]; then cat /etc/timezone
elif [ -L /etc/localtime ]; then readlink /etc/localtime | sed 's|.*/zoneinfo/||'
else echo "unknown"
fi
```

If local timezone detection fails or returns "unknown", label the choice as
"My local time (could not detect — you'll be asked to specify)".

**Present the choice via `ask_user`:**

> In which timezone should the report show times?
> 1. My local time ({detected_local_tz})
> 2. Network timezone ({network_tz})
> 3. UTC

**Special cases:**

- If the network timezone is empty or unset, omit choice 2.
- If the detected local timezone equals the network timezone, collapse into
  two choices:
  > 1. Local / network timezone ({tz})
  > 2. UTC
- If the user selects "My local time" but detection returned "unknown", ask
  them to type their timezone (e.g. "America/New_York", "Europe/London").

Remember the chosen **display timezone** — all times in the report use it.

## Step 4: Determine Time Range

Parse the user's time range from $ARGUMENTS or the conversation. Supported
formats:

- **Relative:** "last 12 hours", "last 3 days", "past 24h", "since yesterday"
- **Absolute:** "from 2026-04-07 10:00 to 2026-04-07 22:00",
  "Apr 7 10am to Apr 7 10pm"
- **Omitted:** if no time range was provided, ask the user:

  > What time range should the report cover? Examples:
  > - "last 12 hours"
  > - "last 3 days"
  > - "from Apr 7 10:00 to Apr 7 22:00"
  >
  > Times without an explicit timezone will be interpreted as
  > {display_timezone}.

**Time interpretation rules:**

1. For relative ranges ("last N hours"), compute from current time. No
   timezone ambiguity.
2. For absolute ranges without explicit timezone (e.g. "from 10:00 to
   22:00"), interpret in the **display timezone** chosen in Step 3. This is
   the most intuitive behavior — if the user chose to see the report in
   Pacific time and says "from 10:00 to 22:00", they mean Pacific.
3. For absolute ranges with explicit timezone (e.g. "10:00 UTC",
   "10:00 EST"), honor the explicit timezone regardless of display choice.
4. Always confirm the interpreted range before fetching:
   > Fetching issues from Apr 7 10:00 AM to Apr 7 10:00 PM
   > (America/Los_Angeles).

**Convert to Unix milliseconds** for the `network_issues` tool. Use bash
for the conversion:

```bash
# Example: convert "2026-04-07 10:00" in America/Los_Angeles to Unix ms
TZ="America/Los_Angeles" date -d "2026-04-07 10:00" +%s000
```

On macOS (BSD date), use:

```bash
TZ="America/Los_Angeles" date -j -f "%Y-%m-%d %H:%M" "2026-04-07 10:00" +%s000
```

For relative ranges, simply compute from current epoch:

```bash
# 12 hours ago in milliseconds
echo $(( ($(date +%s) - 12*3600) * 1000 ))
```

## Step 5: Fetch Issues

Call **`network_issues`** with the display timezone so the response includes
pre-converted local timestamps:

```
network_issues(
  network_id = "<resolved>",
  start_unix_ms = <computed>,
  end_unix_ms = <computed or omit for "now">,
  display_timezone = "<display timezone from Step 3>"
)
```

The response includes:

- Top-level `start_time_local`, `end_time_local` — report range in the
  requested timezone (RFC3339)
- Top-level `start_unix_ms`, `end_unix_ms` — report range as Unix ms
  (use these for `issue_chart` calls)
- Per-issue `startTimeLocal`, `endTimeLocal` — issue times in the requested
  timezone (RFC3339)
- Per-issue `startUnixMs`, `endUnixMs` — issue times as Unix ms (use these
  for `issue_chart` overlay params)

**Use `*Local` fields for display text and `*UnixMs` fields for tool
calls.** Do not convert timestamps yourself.

If the call fails, report the error and stop.

## Step 6: Filter by Significance

**Keep only issues where `scoreOverall > 0.2`.** This filters out
low-confidence or low-impact detections that would add noise to the report.

**Every issue that passes the threshold MUST appear in the report as its own
section.** Do not merge, skip, or summarize away qualifying issues. If three
issues pass the filter, the report must have three Issue sections. The Summary
section at the end is for synthesis — it does not replace individual issue
sections.

Count how many issues were returned total and how many passed the filter.

**If zero issues pass the filter**, report:

> **No significant issues detected** for network **{name}** from
> {start} to {end} ({display_tz}).
>
> {total} minor issue(s) were detected but filtered out (all had overall
> scores below 0.2).

This tells the user the tool worked and data exists — it was just below the
significance threshold.

**If zero issues were returned at all** (empty response), report:

> **No issues detected** for network **{name}** from {start} to {end}
> ({display_tz}).
>
> The network appears stable during this period.

## Step 7: Enrich with Probe Context (Optional)

If the issues reference probe IDs that are not self-explanatory from their
`probeName` field, call **`show_probes`** to map probe IDs to human-readable
names and targets. This helps the user understand which monitoring probe
detected the issue.

Skip this if all probe names are already clear (e.g. `ping_8_8_8_8` is
obviously "ping to Google DNS 8.8.8.8").

## Step 8: Generate Charts (deferred to Step 9 Phase 2)

**Do NOT generate charts before the inline summary.** Charts are only
needed for the HTML report. Skip this step during the initial response and
come back to it in Step 9 Phase 2 if the user requests an HTML report.

When generating charts, call `issue_chart` for every timeseries issue (any
issue that has a `metric` field).

**Chart time range:** use the **report's overall start/end** from Step 4
(not the issue's start/end). This gives full context — the user sees the
entire requested period with the issue highlighted within it.

For each qualifying issue, use the `*UnixMs` fields from the
`network_issues` response directly — no timestamp conversion needed:

```
issue_chart(
  network_id = "<network>",
  probe_id = "<from issue.probeId>",
  probe_type = "<from issue.probeType, e.g. pt_ping>",
  metric = "<from issue.metric, e.g. max_rtt_ms>",
  start_unix_ms = <top-level start_unix_ms from network_issues response>,
  end_unix_ms = <top-level end_unix_ms from network_issues response>,
  timezone = "<display_timezone from Step 3>",
  issues = [{
    start_unix_ms = <issue.startUnixMs from network_issues response>,
    end_unix_ms = <issue.endUnixMs from network_issues response>,
    label = "<issue.description>",
    kind = "<issue.kind>",
    baseline_value = <parsed from details baseline_mean, if available>
  }]
)
```

The response JSON contains a `chart.src` field with a base64 SVG data URI.
Save each chart's `src` value for embedding in the HTML report.

**Skip `issue_chart` for DHCP config/discovery events** — those have no
time-series metric to plot.

If a chart call fails, log the error and continue — the text narrative still
works without the chart.

## Step 9: Present the Report

**Two phases:** first show the inline summary immediately, then offer the
full HTML report.

### Phase 1: Inline Summary (show immediately)

Present the report as a formatted response in the conversation. This is the
primary output — the user gets results without waiting for chart generation
or file export.

```markdown
## Network Issues Report: {network_name}

**Period:** {start_display} – {end_display} ({display_tz})
**Issues found:** {passed_count} significant ({filtered_count} minor
filtered out)
**Agent(s):** {agent_names}

---

### Issue 1: {description} — {probe_name} → {target}

| Field    | Value                                                       |
|----------|-------------------------------------------------------------|
| Time     | {start_time} – {end_time}                                   |
| Duration | {duration}                                                  |
| Agent    | {agent_name}                                                |
| Probe    | {probe_name} ({target_class})                               |
| Type     | {kind_label}                                                |
| Metric   | {metric}                                                    |
| Score    | {overall} (sev: {severity}, conf: {confidence}, cov: {coverage}) |

**What happened:** {narrative paragraph}

---

### Summary

{2-4 sentence synthesis}
```

After the inline summary, ask the user:

> Would you like me to generate an HTML report with charts? I can also
> upload it to Google Drive.

**If the user declines or doesn't respond**, stop here. The inline summary
is the complete deliverable.

**If the user says yes**, proceed to Phase 2.

### Phase 2: HTML Report with Charts (on request)

1. **Generate charts** by calling `issue_chart` for each timeseries issue
   (Step 8 — defer chart generation to this phase to avoid slowing down the
   inline summary).
2. **Write the HTML file** with embedded charts.
3. **Open it** in the browser:
   ```bash
   open "{filename}"
   ```
4. Tell the user the file path.

### HTML Report File

Write a self-contained HTML file named
`network-issues-{network_name}-{YYYY-MM-DD}.html` to the current working
directory using the Write tool.

The HTML must be a single file with all charts embedded as inline SVG data
URIs (from Step 8's `chart.src` values). No external dependencies — the
file must render correctly when opened via `file://` in a browser.

Use this HTML structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Network Issues Report: {network_name}</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
        sans-serif;
      max-width: 960px;
      margin: 2rem auto;
      padding: 0 1rem;
      color: #1a1a1a;
      line-height: 1.6;
    }
    h1 { border-bottom: 2px solid #2563eb; padding-bottom: 0.5rem; }
    .meta { color: #555; margin-bottom: 2rem; }
    .issue { border: 1px solid #e5e7eb; border-radius: 8px;
             padding: 1.5rem; margin-bottom: 1.5rem; }
    .issue h2 { margin-top: 0; color: #1e40af; }
    table { border-collapse: collapse; margin: 1rem 0; }
    th, td { text-align: left; padding: 0.4rem 1rem;
             border-bottom: 1px solid #e5e7eb; }
    th { background: #f9fafb; font-weight: 600; }
    .chart { margin: 1rem 0; text-align: center; }
    .chart img { max-width: 100%; height: auto; }
    .narrative { margin: 1rem 0; }
    .summary { background: #f0f7ff; border-radius: 8px;
               padding: 1.5rem; margin-top: 2rem; }
    .score { font-weight: 600; }
    .score-high { color: #dc2626; }
    .score-med { color: #d97706; }
    .score-low { color: #059669; }
  </style>
</head>
<body>
  <h1>Network Issues Report: {network_name}</h1>
  <div class="meta">
    <p><strong>Period:</strong> {start_display} – {end_display}
       ({display_tz})</p>
    <p><strong>Issues:</strong> {passed_count} significant
       ({filtered_count} minor filtered out)</p>
    <p><strong>Agent(s):</strong> {agent_names}</p>
  </div>

  <!-- Repeat for each issue -->
  <div class="issue">
    <h2>Issue 1: {description} — {probe_name} → {target}</h2>
    <table>
      <tr><th>Time</th><td>{start_time} – {end_time}</td></tr>
      <tr><th>Duration</th><td>{duration}</td></tr>
      <tr><th>Agent</th><td>{agent_name}</td></tr>
      <tr><th>Probe</th><td>{probe_name} ({target_class})</td></tr>
      <tr><th>Type</th><td>{kind_label}</td></tr>
      <tr><th>Metric</th><td>{metric}</td></tr>
      <tr><th>Score</th>
        <td class="score {score_class}">{overall}
          (sev: {severity}, conf: {confidence}, cov: {coverage})</td>
      </tr>
    </table>
    <div class="narrative">
      <p>{narrative paragraph}</p>
    </div>
    <!-- For timeseries issues: embed chart from Step 8 -->
    <div class="chart">
      <img src="{chart.src}" alt="{metric} chart">
    </div>
  </div>
  <!-- /issue -->

  <div class="summary">
    <h2>Summary</h2>
    <p>{2-4 sentence synthesis}</p>
  </div>
</body>
</html>
```

**Score color classes:** use `score-high` for overall >= 0.6,
`score-med` for >= 0.35, `score-low` below 0.35.

### Formatting Rules

**Timestamps:**

- **Use the pre-converted `*Local` fields** from the `network_issues`
  response. These are already in the display timezone (RFC3339 format like
  `2026-04-07T11:53:00-07:00`). Reformat them for display (e.g.
  "Apr 7 11:53 AM") — this is string reformatting, not timezone math.
- **Do not convert UTC timestamps yourself.** The server has already done
  the timezone conversion.
- Use 12-hour format with AM/PM: "Apr 7 11:53 AM – 12:17 PM"
- Include the date only when the range spans multiple days.
- Show the display timezone name in the report header (not on every
  timestamp).

**Issue titles:**

- Use the `description` field as the base (e.g. "Choppy latency for 24m").
- Append the probe target for context: "Choppy latency — ping to 8.8.8.8".

**Narrative ("What happened"):**

The narrative format depends on the issue kind. Issues fall into two
categories: **timeseries anomalies** (have a `metric` field) and
**configuration/discovery events** (`metric` is empty, `is_instant` is
true).

#### Timeseries Anomalies (`outlier_cluster`, `mean_shift`, `variance_shift`)

Parse the `details[]` array to build a narrative. Common keys:

For `outlier_cluster` kind:
- `baseline_mean`, `baseline_median`, `baseline_p95` — normal behavior
- `cluster_mean`, `cluster_max` — anomalous values during the cluster
- `cluster_outlier_count` — number of anomalous samples
- `cluster_duration_min` — how long the cluster lasted (minutes)
- `cluster_density_per_min` — outlier frequency
- `cluster_severity_ratio` — ratio of cluster_max to baseline_p95
- `cluster_mean_robust_z`, `cluster_max_robust_z` — statistical deviation

For `mean_shift` kind:
- `segment_before_mean` — metric value before the shift
- `segment_after_mean`, `segment_after_p95`, `segment_after_max` — after
- `delta_mean_vs_baseline`, `ratio_mean_vs_baseline` — magnitude of change

Example narrative:

> Latency spiked from a baseline of ~12.6 ms (median 10.2 ms) to a
> cluster mean of 30.6 ms with peaks at 56.5 ms. The cluster lasted 25
> minutes with 12 outlier samples. The worst sample was 31.9σ above the
> robust baseline.

#### DHCP Configuration Changes (`dhcp_config_change`)

These are instant events (`is_instant: true`) with no metric. Parse
`details[]` for these keys:

- `dhcp_fields_changed_count` — number of fields that changed
- `dhcp_fields_changed` — comma-separated field names (e.g.
  "default_gateway,dns_servers,lease_duration_sec")

The changed fields come from DHCP server configuration: `default_gateway`,
`dns_servers`, `domain_name`, `subnet_mask`, `broadcast_address`,
`lease_duration_sec`, `renewal_time`, `rebinding_time`, `ntp_servers`,
`server_ip`, `server_name`, `relay_agent_circuit_id`,
`relay_agent_remote_id`.

Present as a change summary rather than a chart:

> **DHCP configuration changed** on server 192.168.10.1 at 3:42 PM.
> 3 fields changed: default_gateway, dns_servers, lease_duration_sec.

**Do not call `issue_chart` for DHCP config change issues** — there is no
time-series metric to plot. Use a text-only presentation.

#### New DHCP Server (`dhcp_new_server_seen`)

Parse `details[]` for:

- `dhcp_server_ip` — IP of the new server
- `dhcp_relay_agent_ip` — relay agent IP (if present)
- `dhcp_interface` — interface where it appeared
- `dhcp_vlan_id` — VLAN ID (if applicable)

Present as a discovery alert:

> **New DHCP server detected** at 192.168.10.2 on interface eth0 at
> 5:15 PM. This may indicate a rogue DHCP server.

#### Multiple DHCP Servers (`dhcp_multiple_servers_seen`)

Parse `details[]` for:

- `dhcp_server_count` — number of servers responding
- `dhcp_server_ips` — comma-separated list of server IPs

Present as:

> **Multiple DHCP servers detected** at 2:30 PM: 192.168.10.1,
> 192.168.10.2 (2 servers). Multiple DHCP servers can cause IP conflicts
> and connectivity issues.

#### Cable Modem / DOCSIS Metrics

Issues on a `pt_scripted` probe whose `metric` starts with `docsis_` come
from the cable-modem monitoring rule (e.g. `cm2500_docsis_triage`). These
metrics let the report distinguish **LAN-side** from **cable-side**
(ISP / plant) problems. Treat them as first-class issues in the report and
add a triage line that maps the observed metric to a likely cause.

**Metric semantics and healthy ranges** (Netgear CM2500 and similar DOCSIS
3.1 modems):

| Metric                           | Meaning                                 | Healthy band                     | Out-of-band implies                                |
|----------------------------------|-----------------------------------------|----------------------------------|----------------------------------------------------|
| `docsis_connectivity_ok`         | Modem overall connectivity state        | `1`                              | `0` -> modem offline (ISP / plant)                 |
| `docsis_boot_ok`                 | Registered with CMTS                    | `1`                              | `0` -> registration failure                        |
| `docsis_locked_channels_ratio`   | Fraction of DS + US channels locked     | `1.0`                            | `< 1.0` -> flapping bonding, partial service       |
| `docsis_ds_power_dbmv_min`       | Min downstream RX power                 | `-7 … +7 dBmV`                   | `< -7` -> long run / too many splitters            |
| `docsis_ds_power_dbmv_max`       | Max downstream RX power                 | `-7 … +7 dBmV`                   | `> +7` -> overdriven input / amplifier issue       |
| `docsis_ds_snr_db_min`           | Min downstream SNR/MER                  | `? 33 dB` SC-QAM, `? 35 dB` OFDM | below -> noise ingress on plant                    |
| `docsis_us_power_dbmv_max`       | Max upstream TX power                   | `35 … 51 dBmV`                   | `> 51` -> modem "shouting", return path attenuated |
| `docsis_ds_uncorrectables_total` | Counter of uncorrectable codewords (DS) | rate flat                        | rising rate -> user-visible packet loss            |

**Triage rules** — when narrating a DOCSIS issue, always append one of these
conclusions to the narrative:

- `docsis_connectivity_ok == 0` or `docsis_boot_ok == 0` during the cluster
  → **cable side (call ISP)** — modem could not reach or register with the
  CMTS.
- `docsis_locked_channels_ratio` dropped below `1.0` → **cable plant** —
  one or more bonded channels lost lock (drop cable, splitter, ingress).
- `docsis_ds_power_dbmv_min` / `_max` out of `−7…+7 dBmV`, or
  `docsis_ds_snr_db_min` below threshold → **cable plant RF** — signal
  level or noise floor on the coax is wrong.
- `docsis_us_power_dbmv_max > 51 dBmV` → **return-path attenuation** —
  modem is transmitting near the top of its range to be heard.
- `docsis_ds_uncorrectables_total` increase (rising counter) → **cable
  plant** — uncorrectable FEC errors translate directly to packet loss
  that the user will feel as stalls, retransmits, or game lag.
- None of the above, but LAN-side probes (ping, DNS, HTTP to local
  targets) show issues → **LAN side** — DOCSIS layer is healthy; problem
  is inside the home (Wi-Fi, router, switch, client device).

**Narrative example** (for an `outlier_cluster` on `docsis_us_power_dbmv_max`):

> Upstream transmit power climbed from a baseline of 44.2 dBmV (p95
> 46.1 dBmV) to a cluster mean of 50.8 dBmV with peaks at 52.3 dBmV —
> above the healthy 35–51 dBmV band. The modem is straining to be heard
> by the CMTS. **Likely cable-side cause: return-path attenuation** (bad
> connector, corroded splitter, or damaged drop cable between the house
> and the tap).

**Ordering within the report:** DOCSIS issues typically co-occur (e.g. a
plant problem will trip ratio, SNR, and uncorrectables together). Keep the
API's `scoreOverall` ordering, but call out in the Summary section if
multiple DOCSIS metrics move together — that is stronger evidence of a
cable-side problem than any single metric alone.

**Charts:** DOCSIS issues have a `metric` field, so `issue_chart` works
normally. Use the same pattern as other timeseries anomalies.

**Scores:**

- Round all scores to 2 decimal places.
- Show the overall score prominently; show component scores (severity,
  confidence, coverage) in parentheses.

**Kind labels:** Use human-readable labels:

| Raw kind                     | Display label             |
|------------------------------|---------------------------|
| `outlier_cluster`            | Outlier cluster           |
| `mean_shift`                 | Sustained mean shift      |
| `variance_shift`             | Stability change          |
| `dhcp_config_change`         | DHCP configuration change |
| `dhcp_new_server_seen`       | New DHCP server detected  |
| `dhcp_multiple_servers_seen` | Multiple DHCP servers     |

**Issue ordering:**

- Present issues sorted by `scoreOverall` descending (most significant
  first). The API already returns them this way.

**Summary section:**

- Group related issues (same probe/target) together.
- Note temporal patterns (e.g. "two episodes ~1 hour apart").
- State whether issues are resolved (have `endTime`) or still open
  (`open: true`).
- If all issues are on the same probe, note that other probes on the
  network were unaffected (if `show_probes` was called).

### PDF Export for Google Drive

If the user asks to upload to Google Drive, convert the HTML to PDF first
(Drive renders PDF natively but may strip SVG from HTML):

```bash
# Try wkhtmltopdf first, fall back to Chrome headless
if command -v wkhtmltopdf &>/dev/null; then
  wkhtmltopdf "{html_file}" "{pdf_file}"
elif command -v "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" &>/dev/null; then
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless --disable-gpu --print-to-pdf="{pdf_file}" \
    "file://$(pwd)/{html_file}"
else
  echo "Neither wkhtmltopdf nor Chrome found for PDF conversion"
fi
```

If PDF conversion succeeds, upload the PDF. If it fails, upload the HTML
and warn the user that charts may not display in Drive's HTML preview.

If Google Drive MCP tools are not available in the environment, tell the
user:

> Google Drive upload is available in Claude Cowork. I've saved the file
> locally to {path} — you can upload it manually.
