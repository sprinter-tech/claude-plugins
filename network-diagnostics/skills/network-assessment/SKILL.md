---
name: network-assessment
description: >
  Assess internet connection quality for a network. Use when the user
  asks about network performance, connection quality, speed test results,
  latency, jitter, suitability for VoIP/gaming/streaming, or asks
  "how is this network performing?" or "is this connection good enough
  for video calls?"
argument-hint: "[network-id]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill, WebSearch,
  mcp__sprinter__show_network,
  mcp__sprinter__show_agent,
  mcp__sprinter__list_networks,
  mcp__sprinter__show_probes,
  mcp__sprinter__network_issues,
  mcp__sprinter__network_benchmarks,
  mcp__sprinter__analytics_report,
  mcp__sprinter__network_ping,
  mcp__sprinter__network_traceroute,
  mcp__sprinter__network_dns_lookup,
  mcp__sprinter__network_jitter_test,
  mcp__sprinter__network_speed_test,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__ask_user
---

Assess network $ARGUMENTS

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

If the user provided a `network_id`, use it directly. Otherwise:

1. **`list_networks`** — show available networks.
2. **`ask_user`** — if multiple networks exist, ask the user which one.

Once you have a `network_id`:

3. **`show_network`** — get network details, ISP info, and the list of
   agents (names and IDs). This tells you what agents are online and gives
   you the network's timezone.

## Step 2: Gather Performance Data

Use these tools to build a picture of network quality. Start with the
analytics report for a comprehensive view, then drill into specifics.

**Comprehensive assessment:**
- **`analytics_report`** — returns suitability scores (VoIP, gaming,
  streaming), baseline assessments per probe type (connectivity, jitter,
  DNS, HTTP, DHCP), and RRUL/speed test benchmark data. This is the
  primary tool for network quality assessment. Request specific sections
  to reduce response size:
  - `sections: ["scores"]` — suitability scores plus underlying
    benchmarks and baselines
  - `sections: ["benchmarks"]` — RRUL and speed test results only
  - `sections: ["baselines"]` — probe baseline assessments only

**Benchmark details:**
- **`network_benchmarks`** — latest RRUL and speed test results from
  VictoriaMetrics. Use when you only need throughput, latency, jitter,
  and MOS numbers without the full report. Defaults to last 7 days.

**Active issues:**
- **`network_issues`** — on-demand issue detection. Returns outlier
  clusters (traffic spikes/drops), mean shifts (sustained metric
  changes), and variance shifts (stability changes). Each issue has a
  `kind` field you can filter on and `severity`/`confidence` scores.

**Probe inventory:**
- **`show_probes`** — list all probes running on the network. Useful
  to understand what is being monitored (which DNS servers, ping
  targets, IRTT servers, etc.) and to correlate probe IDs from
  baselines and issues back to human-readable names and targets.

## Step 3: Live Diagnostics (Optional)

If the collected data suggests a problem, or if the user asks about a
specific aspect, run live diagnostic probes:

- **`network_ping`** — verify current reachability and latency to a
  specific target
- **`network_traceroute`** — trace the path to identify where latency
  or loss occurs
- **`network_dns_lookup`** — check DNS resolution time and correctness
- **`network_jitter_test`** — measure current jitter and latency (IRTT)
- **`network_speed_test`** — run a fresh speed test (takes ~30 seconds)

Use these sparingly — the baseline data from `analytics_report` already
covers the historical picture. Live probes are for confirming current
state or investigating specific targets.

## Step 3a: Wi-Fi Is Out of Scope Here — Hand Off

This skill assesses the **internet/WAN connection** (latency, jitter,
throughput, VoIP/gaming/streaming suitability). It does **not** read per-client
Wi-Fi link health. If the user's question is about a **wireless device's**
experience — "is the Wi-Fi good enough for video calls on my laptop?", a slow/
choppy wireless client, signal/SNR/retries/channel-congestion — that is a
different metric system (the `sprinter_wifi_*` series, platform-shaped and
graded against catalog health bands), and this skill has neither the
`timeseries_*` tools nor the platform logic to read it.

Hand off instead:
- **`Skill(diagnose-wifi-basic)`** — the general per-client Wi-Fi link health
  check (signal, retries, congestion, rate); the right tool for "is this
  wireless device's link OK?".
- **`Skill(diagnose-wifi-roaming)`** — when the wireless complaint is
  "stuck on a far AP" / won't roam / mesh.

Those skills read live link health from VictoriaMetrics (~1-min fresh), adapt per platform
(UniFi rich vs BGW320 sparse), and grade against the catalog bands described in the reference
doc fetched via the `get_reference_doc` MCP tool (`name: wifi-metrics-reference`). A WAN
assessment here can still be a useful *companion* (a clean WAN with a bad per-client Wi-Fi link
localizes the problem to the air), but the Wi-Fi verdict itself belongs to those skills.

## Step 4: Present Assessment

Structure your assessment around what the user cares about:

**For general "how is my network?" questions:**
- Overall connection quality (suitability scores)
- Download/upload speeds (from benchmarks)
- Latency and jitter characteristics (from RRUL or baselines)
- Any active issues detected
- Comparison to typical requirements (VoIP needs <150ms RTT, <30ms
  jitter; gaming needs <30ms RTT; 4K streaming needs >25 Mbps)

**For specific use-case questions ("can I run video calls?"):**
- Lead with the relevant suitability score
- Back it up with the specific metrics that drive the score
- Flag any issues that could cause intermittent problems

**For "what's wrong?" questions:**
- Start with `network_issues` to find detected anomalies
- Correlate issues with baseline data to show deviation from normal
- Run live probes to confirm current state
- Suggest remediation if the cause is identifiable

Always ground your assessment in the actual data — cite specific
numbers (latency, jitter, throughput, MOS scores) rather than making
qualitative statements without evidence.
