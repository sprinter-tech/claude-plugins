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
  mcp__sprinter__find_network,
  mcp__sprinter__show_network,
  mcp__sprinter__network_tech_stack,
  mcp__sprinter__find_device,
  mcp__sprinter__show_device,
  mcp__sprinter__network_issues,
  mcp__sprinter__show_probes,
  mcp__sprinter__ask_user,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__issue_chart,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__timeseries_range
---

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Sprinter
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

Generate a network issues report for $ARGUMENTS

## Scope

This report covers **network-level anomalies** detected by Sprinter's analytics
(loss / latency / variance shifts from probes), DHCP config changes, and the
health of the **WAN link itself** — the cable, fiber, cellular, or Starlink
satellite connection the network reaches the internet through (Step 5b).

It does **not** cover per-client **Wi-Fi** link health: there is no
`sprinter_wifi_*` anomaly stream here. If the user is really asking "is my
wireless device's link OK?" (signal, SNR, retries, channel congestion), that is
`Skill(diagnose-wifi-basic)` / `Skill(diagnose-wifi-roaming)` — point them there
rather than implying a WiFi finding from anomaly data.

Note the WAN metrics are a *different layer* from Wi-Fi and the words collide:
`docsis_ds_snr_db` is cable-plant MER on the coax, **not** 802.11 signal
quality. Do not mix them in one verdict.

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

1. If a network name was given, call **`find_network`** (name prefix, across all
   your orgs) to resolve it directly.
2. Otherwise call **`list_networks`** — it spans all your organizations (orgs) and
   each row carries `tenant_id` + `tenant_name`.
3. **`ask_user`** — if multiple networks exist (or a name matches networks in
   more than one org), ask the user which one, showing the **org** when you belong
   to more than one:
   > Which network would you like the issues report for?
   > **Lincoln SD**
   >   1. **SD office network** — 10.0.0.0/24 (12 devices)
   > **Jefferson SD**
   >   2. **SD office network** — 10.0.1.0/24 (9 devices)

   Omit networks with zero devices unless the user explicitly asks. When you
   belong to only one org, the org grouping is unnecessary.

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

## Step 5b: WAN Link Health (always run this)

The anomaly stream tells you *that* the network misbehaved. The WAN metrics tell
you *where* — inside the LAN, or out on the cable plant / fiber / cellular link.
Run this **every time, even when zero anomalies fired**: a modem with a
degrading upstream or climbing FEC errors is worth reporting whether or not the
detector happened to notice, and a clean WAN link is what lets you say
"the problem is inside the building" with confidence.

1. **`network_tech_stack`** — read **Connection technology** (`cable` / `fiber` /
   `cellular` / `dsl` / `unknown`) and the **internet gateway's `device_id`**.
   Note **Starlink reports `unknown`** — its dish is a generic `modem` class, so
   satellite is never named here; step 2's sweep is how you catch it.
2. **If the technology is `unknown`, do NOT conclude "unsupported."** On the very
   common **bridge-mode** topology — a standalone cable modem behind a consumer
   router — `network_tech_stack` names the *router* as the gateway and reports
   `unknown`, while the modem sits on its own IP (typically `192.168.100.1`)
   emitting DOCSIS the whole time. **Starlink is the same shape**: the dish sits at
   `192.168.100.1` streaming satellite telemetry while the Starlink *router* (a
   `gateway` at `192.168.1.1`) is what the tech stack names. Sweep for the real
   WAN device with `find_device(device_class=...)` over `cable_modem`,
   `cable_gateway`, `fiber_ont`, `fiber_gateway`, `cellular_gateway`, and `modem`
   (a `modem`-class hit with `vendor == "Starlink"` is the dish → satellite). A hit
   gives you both the technology and the `device_id` to query. The reference doc
   (next step) spells this out.
3. **`get_reference_doc`** with `name: wan-metrics-reference`. Use **only** the
   section for the technology you resolved. It lists every metric that technology
   emits, its health bands, and the exact PromQL to run.
4. **`timeseries_instant`** each metric in that section, substituting the **WAN
   device's** `device_id` — in bridge mode that is the *modem*, not the device the
   tech stack called the gateway. Use `timeseries_range` when you need to see
   whether a value moved *during* the report window — that is what ties a WAN
   fault to a user-visible incident.
5. **Grade each value against its band** and report the **cause**, not the
   number.

**Do not hardcode metric names or thresholds in this skill.** They live in the
catalog and reach you through the reference doc. (This section previously carried
its own DOCSIS table; the catalog moved to per-channel, vendor-neutral names and
the table silently rotted into describing metrics that no longer existed. Fetch,
don't remember.)

**Two-sided metrics: say which tail.** Some WAN metrics are bad in *both*
directions, and the two tails are *different faults with opposite fixes*. A DOCSIS
downstream at `+10 dBmV` is not "strong signal" — it is an **overdriven input**
(remove an amplifier, check the tap), the opposite problem from a weak one (bad
splitter, long run). The reference doc names what each tail means; use its words.
Never report "the value is out of band" and stop there — that discards the half of
the answer the operator needs.

**Before calling loss "upstream" — correlate the loss across LAN targets.**

A packet-loss episode on the `ping_8_8_8_8` / first-hop probes tells you the
agent lost the internet; it does **not** tell you *where* the fault was. The agent
reaches the internet through the LAN (a wired or wireless hop to the gateway), so
an **internal** fault — a broadcast/multicast storm, a switching loop, a gateway
melting down, a duplex mismatch — shows up as WAN loss from the agent's vantage
**even though the cause is inside the building.** Reporting it as a "one-off WAN
event" then misses the real, recurring problem.

**First, check whether Sprinter already answered this.** The multi-ping fleet probe
(`pt_multi_ping`) pings every device on the network and the insights pipeline
correlates the result: a **`lan_wide_connectivity_loss`** issue in your Step 5 fetch
IS the LAN-vs-WAN verdict, already computed (see its narration section below). If one
is present for the loss window, **use it** — the loss was internal, and you can skip
the manual query below. Only fall back to the manual correlation when no such issue
fired (e.g. the storm was below the fleet-fraction threshold, or the probe is not
running on this network).

Manual fallback: query `sprinter_ping_loss_ratio` for the **LAN infrastructure** —
the gateway/router, any switches (`device_class="network_switch"`), the Wi-Fi
controller, and a couple of always-on internal hosts — and compare against the
egress anchors:

```
# egress path (what you already looked at)
sprinter_ping_loss_ratio{probe_id="<ping_8_8_8_8 probe>"}
# LAN — was the storm internal? one series per internal device_id
max_over_time(sprinter_ping_loss_ratio{device_id=~"<gateway|switch|internal hosts>"}[5m])
```

Read the pattern, and **say which one you found**:

- **Loss on the WAN anchors only, LAN clean** → a genuine upstream/WAN event.
  Now "one-off, upstream of your problem" is a supported conclusion.
- **Loss on LAN infra too (gateway + internal hosts)** → the fault is **inside the
  network**, and the internet loss is a *symptom* seen from the agent's vantage,
  not the cause. Broadcast storm, switch loop, or gateway failure — a storm
  typically hits many internal targets at once and elevates their RTT/jitter as
  well. This is a materially different diagnosis; report it as internal even when
  the triggering complaint is something else.

**Reporting the WAN verdict:**

- Any WAN metric grading `poor` → **the problem is on the WAN side.** Name the
  technology, the metric, the value, and the tail's cause. If LAN-side probes also
  show issues, the WAN fault is very likely their cause — say so.
- All WAN metrics `good`, but probes show loss/latency → **the WAN link is
  healthy; the problem is inside the network.** Do the LAN-wide correlation above
  to localize it — network-wide LAN loss points at a storm/loop, not the WAN.
  This is a real, useful finding — state it positively rather than omitting it.
- WAN metrics missing → see the **When metrics are missing** table in the
  reference doc, and follow it exactly. The distinction it draws matters: asking
  for credentials is right for a cable modem and **wrong** for fiber/cellular/
  Starlink (those rules use unauthenticated endpoints — the Starlink dish speaks
  a credential-free local gRPC API — so missing metrics never mean "no
  credentials").

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

#### LAN-Wide Connectivity Loss (`lan_wide_connectivity_loss`)

This is the **highest-value connectivity verdict the report can carry**, and it is
the automated answer to "was the loss internal or upstream?" — the question Step 5b
otherwise makes you derive by hand. `probeType = pt_multi_ping`, `metric` empty, and
it is an **interval** issue (a real `start`/`end`, not an instant): Sprinter pings
*every* device on the network from the agent, and this issue fires when a large
fraction of the fleet went lossy **in the same window**. Simultaneous loss across
many internal devices is the signature of an **internal** fault — a broadcast/
multicast storm, a switching loop, a gateway meltdown — NOT an upstream/WAN event.

Parse `details[]` for:

- `lan_wide_peak_fraction` — the largest share of the pinged fleet lossy at once
  (0..1). `0.89` means ~89% of devices were losing packets simultaneously.
- `lan_wide_affected_count` / `lan_wide_fleet_size` — devices lossy at the peak vs
  devices pinged (e.g. `34` of `38`).
- `lan_wide_duration_sec` — how long the episode lasted.
- `lan_wide_affected_device_ids` — comma-separated device_ids to pivot into with
  `show_device` if you want to name the affected devices.
- `lan_wide_gateway_affected` / `lan_wide_wan_anchor_affected` — present (`true`)
  only when the gateway, or a WAN anchor like the `8.8.8.8` ping target, was also in
  the lossy set. `wan_anchor_affected=true` is the tell that WAN loss **coincided
  with** LAN-wide loss (so the internet outage was a *symptom* of the internal
  fault), not a standalone WAN event.

Present as a network-scoped verdict, not a single-device chart:

> **LAN-wide connectivity loss** from 1:52 PM to 2:16 PM (24 min): at the peak, 34
> of 38 pinged devices were losing packets simultaneously (89% of the fleet). Loss
> hitting this many internal devices at once points at an **internal** fault — a
> broadcast/multicast storm, a switching loop, or a failing gateway — not an
> upstream/ISP event. The gateway was among the affected devices.

**When this issue is present, it OVERRIDES the manual LAN-vs-WAN correlation in
Step 5b** — the conclusion is already stated. Do NOT then present a coincident
`pt_ping` loss on `ping_8_8_8_8` as a separate "one-off WAN event": it is the same
storm seen from the egress vantage. Lead with the storm; treat the per-anchor ping
issues as corroboration. **Do not call `issue_chart` for this issue** — it is
network-scoped with no single metric series to plot.

**Beyond these three events, the DHCP server has a health/performance axis.** The
events above are config-drift and rogue-appeared detections; they say nothing about
whether the server is *answering* or *how fast*. Sprinter runs an active DORA probe
that emits `sprinter_dhcp_*` metrics (DORA timing, ack/offer/timeout/nack counters,
`num_responses`). If a `dhcp_*` event fired — or a client-connectivity complaint
points at DHCP — confirm the live state with those metrics: `show_probes` for the
`pt_dhcp` `probe_id`, then `get_reference_doc(name: "dhcp-metrics-reference")` and
`timeseries_range`. Climbing `dhcp_timeouts_total` with flat `dhcp_acks_seen_total`
is a server-down finding the event stream alone will not give you. **These series
are labeled `probe_id` / `dhcp_server`, not `device_id`.**

#### WAN Metrics (DOCSIS / optical / cellular / Starlink)

An issue on a `pt_scripted` probe whose `metric` is a WAN metric
(`docsis_*`, `optical_*`, `pon_*`, `cellular_*`, `starlink_*`) comes from the WAN
monitoring rule for the network's internet gateway or dish. Treat these as
first-class issues.

**Do not grade them from memory — fetch the bands.** `get_reference_doc`
(`name: wan-metrics-reference`) carries every WAN metric's meaning, its health
band, and (for the two-sided ones) what each tail means. You will already have
fetched it in Step 5b; reuse it.

Narrate a WAN issue the same way as any other timeseries anomaly (baseline vs
cluster stats from `details[]`), then append the **cause** from the reference
doc's band — the tail's meaning, not just "out of band". Example:

> Upstream transmit power climbed from a baseline of 47.8 dBmV to a cluster mean
> of 54.9 dBmV, peaking at 57.2 dBmV — into the `poor` band. The modem is
> straining to be heard by the CMTS. **Likely cause: return-path attenuation** —
> a bad connector, corroded splitter, or damaged drop cable between the house and
> the tap.

**Ordering:** WAN issues co-occur (a plant problem trips SNR, uncorrectables, and
channel lock together). Keep the API's `scoreOverall` ordering, but call it out in
the Summary when several WAN metrics move together — that is far stronger evidence
of a WAN-side fault than any single metric alone.

**Charts:** WAN issues have a `metric`, so `issue_chart` works normally.

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
