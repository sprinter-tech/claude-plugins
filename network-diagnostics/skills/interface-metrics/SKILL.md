---
name: interface-metrics
description: >
  Read per-interface traffic, speed, errors, and discards (and device-level
  CPU/memory/uptime) for an SNMP-monitored device from stored time-series
  metrics, not live SNMP. Use when troubleshooting any switch/router/AP
  interface — "is this link saturated / erroring / flapping", "how much
  traffic on port X", "is this uplink clean", "show interface counters",
  "is the device's CPU/memory healthy" — or as a universal step when grading
  a wired link for any device. Other skills (locate-device-uplink,
  troubleshoot-device, diagnose-wifi-roaming) hand off here to read interface
  health.
argument-hint: "[device-name-or-ip] [if-index]"
allowed-tools: >
  Bash, Read, Skill,
  mcp__sprinter__show_network,
  mcp__sprinter__find_device,
  mcp__sprinter__show_device,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__snmp_get,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__ask_user
---

Read interface metrics for $ARGUMENTS

## What this skill does

Read per-interface and device-level health for an SNMP-monitored device
(switch, router, AP, gateway) from Sprinter's **stored time-series metrics** in
VictoriaMetrics — traffic, speed, errors, discards, link state, and device
CPU/memory/uptime. This is the universal "is this interface/device healthy?"
step.

**Read metrics, do not run live SNMP for this.** Sprinter already polls these
counters (~60s cadence) and stores them as Prometheus-style series. Querying the
stored series gives a real **rate over a window**; reading a raw SNMP counter
gives a cumulative-since-boot number you cannot interpret without a second
sample. **Never** read a counter twice with a sleep to estimate a rate — that is
both unreliable (the window is too short) and redundant.

This skill is **read-only**.

**Scope: the WIRED interface/uplink port only.** It grades the SNMP IF-MIB
interface (and device CPU/memory/uptime) — the same `rate()`-over-a-window
discipline below applies to any of those counters. It is **not** a Wi-Fi radio /
client metric reader. For per-client wireless link health (signal, SNR, retries,
channel utilization — the `sprinter_wifi_ap_radio_*` / `sprinter_wifi_client_*`
families) the metrics are platform-shaped (UniFi rich vs BGW320 sparse) and live
in a different catalog: route those questions to `Skill(diagnose-wifi-basic)`
and fetch the wifi-metrics reference via the `get_reference_doc` MCP tool
(`name: wifi-metrics-reference`). When you are reached to grade an **AP**, you
grade its wired uplink port; the wireless side is the WiFi skill's.

## MCP Server Availability — Check First

If any `mcp__sprinter__*` call fails with a connection / auth / "server
disconnected" error, stop and tell the user the Sprinter MCP server is
unavailable and must be reconnected. Do not work around it with guesses.

## Step 1 — Identify the device and interface

Resolve the `network_id` (hand off to `Skill(select-network)` if not known).
`find_device` the target to get its `device_name` (the metric label) and
`device_id`. Interfaces are keyed by **`if_index`** (and labelled `if_name`,
e.g. `1/0/25`). If you don't know the `if_index`, query without it (Step 2
returns one series per interface) and pick by `if_name`, or read
`sprinter_interface_info` to list interfaces with their `if_alias`/`if_role`.

## Step 2 — Query the metrics (`timeseries_range`)

Queries are **automatically scoped to your network** — do NOT add a
`network_id` label filter to the PromQL; the `network_id` argument handles it.
Filter the series by `device_name` (or `device_id`) and `if_index`.

| Want                  | Metric / query                                                                                              |
|-----------------------|-------------------------------------------------------------------------------------------------------------|
| Throughput (bits/s)   | `rate(sprinter_interface_in_octets_total{device_name="X",if_index="N"}[5m]) * 8` (and `*_out_octets_total`) |
| Inbound error rate    | `rate(sprinter_interface_in_errors_total{device_name="X",if_index="N"}[15m])`                               |
| Outbound error rate   | `rate(sprinter_interface_out_errors_total{device_name="X",if_index="N"}[15m])`                              |
| Inbound discard rate  | `rate(sprinter_interface_in_discards_total{device_name="X",if_index="N"}[15m])`                             |
| Outbound discard rate | `rate(sprinter_interface_out_discards_total{device_name="X",if_index="N"}[15m])`                            |
| Link speed (bits/s)   | `sprinter_interface_speed_bps{device_name="X",if_index="N"}` (gauge)                                        |
| Link up?              | `sprinter_interface_operational_up` / `sprinter_interface_admin_up` (1/0 gauges)                            |
| Port identity         | `sprinter_interface_info` — labels `if_name`, `if_alias` (operator desc), `if_role`                         |

Example:

```
timeseries_range(
  network_id=<net>,
  query='rate(sprinter_interface_in_errors_total{device_name="was-sw1201",if_index="25"}[15m])',
  start=<now-1h>, end=<now>, step="5m")
```

Use `timeseries_instant` for a single current value (e.g. "what's the link
speed right now"); use `timeseries_range` for trends and rates over a window.
Pick `start`/`end`/`step` to fit the question — last hour at 5m for a spot
check, last 24h at 30m–1h for "has this been erroring all day".

### Device-level health (same tools, no `if_index`)

| Want       | Metric                                                                             |
|------------|------------------------------------------------------------------------------------|
| Uptime     | `sprinter_system_uptime_seconds`                                                   |
| CPU        | `sprinter_system_cpu_utilization_ratio` (0..1)                                     |
| Memory     | `sprinter_system_memory_utilization_ratio` (0..1), `*_used_bytes`, `*_total_bytes` |
| Filesystem | `sprinter_system_filesystem_utilization_ratio` (per `mount_point`)                 |

A recent jump in `sprinter_system_uptime_seconds` back to near-zero means the
device rebooted — worth flagging when interfaces look like they "flapped".

## Step 3 — Interpret

- **Errors/discards**: a **sustained nonzero rate** is an active problem
  (bad cable/optics, duplex mismatch, congestion). A **flat-zero rate** over the
  window means whatever absolute counter value exists is old and benign — say
  so, don't alarm on a large cumulative count.
- **Traffic vs speed**: compare `rate(octets)*8` against
  `sprinter_interface_speed_bps`. Sustained throughput near the link speed =
  saturation; that plus rising `*_discards` = the link is the bottleneck.
- **Speed sanity**: a gigabit-capable port reading `100000000` (100 Mbps) is a
  negotiated-down/damaged-cable finding in its own right — it caps every device
  on the branch.
- **Link state**: `interface_operational_up` toggling 1↔0 across the window =
  flapping.

## Step 4 — Fallback when metrics are absent

If `timeseries_range` returns no series for the interface (newly discovered
device, or a gap in the metrics pipeline), fall back to a **single** live
`snmp_get` of the IF-MIB columns and **say explicitly** that you could read only
a cumulative-since-boot snapshot, not a rate:

| Fact                | OID (append `.<ifIndex>`)  |
|---------------------|----------------------------|
| ifInErrors          | `.1.3.6.1.2.1.2.2.1.14`    |
| ifOutErrors         | `.1.3.6.1.2.1.2.2.1.20`    |
| ifInDiscards        | `.1.3.6.1.2.1.2.2.1.13`    |
| ifOutDiscards       | `.1.3.6.1.2.1.2.2.1.19`    |
| ifHighSpeed (Mbps)  | `.1.3.6.1.2.1.31.1.1.1.15` |
| ifOperStatus (1=up) | `.1.3.6.1.2.1.2.2.1.8`     |
| sysUpTime           | `.1.3.6.1.2.1.1.3.0`       |

Report the counts in `sysUpTime` context. Do **not** re-sample with a sleep.

## Step 5 — Present findings

State the interface (device + `if_name`/`if_index`), the throughput, the
error/discard verdict (active vs benign, with the window you used), link speed
and state, and any device-level health flag (reboot, CPU/memory pressure). When
a finding is "no problem", say it plainly — a clean link over the queried window
is itself a useful result that rules out the physical layer.

## When this skill is reached from another skill

Return the interface verdict (throughput, error/discard rate over the window,
speed, link state) so the caller can fold it into its own report. Callers:
`locate-device-uplink` (grade the attachment port once located),
`troubleshoot-device` (device/interface health), `diagnose-wifi-roaming`
(grade a mesh AP's wired uplink port).
