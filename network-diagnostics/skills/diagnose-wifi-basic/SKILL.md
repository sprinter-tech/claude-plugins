---
name: diagnose-wifi-basic
description: >
  Basic Wi-Fi link health check for one client, read-only. Reads live link
  health from VictoriaMetrics (~1-min fresh) with structure from device evidence.
  Use when a wireless device is slow, choppy, laggy, has jittery
  ping, drops packets, or "feels bad" but it is NOT specifically a roaming /
  sticky-client / mesh question. Establishes whether the link is healthy and,
  if not, which factor is to blame (weak signal, high retries, congestion/noise,
  low rate, intermittent association). Hands off to diagnose-wifi-roaming when
  the cause turns out to be a far-AP / roaming problem.
argument-hint: "[device-address-or-id]"
allowed-tools: >
  Bash, Read, Skill,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__show_device,
  mcp__sprinter__find_device,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__network_http,
  mcp__sprinter__network_ping
---

# Basic Wi-Fi Link Health Check (read-only)

A general "is this wireless link OK, and if not why" check for a single client.
This is the **broad** Wi-Fi skill; `diagnose-wifi-roaming` is the **narrow**
roaming/mesh follow-up. Often invoked from `triage-network-complaint` once a
complaint is localized to a Wi-Fi client.

**Read-only.** It never changes controller or device config.

## Where the data lives — three tiers

Sprinter's wifi service polls the network's WiFi controller (~every minute). The
same per-client readings land in **two** places on **two clocks**, and the skill
must use the right one for the right job:

| Tier | Source | Freshness | Use it for |
|------|--------|-----------|------------|
| **1 — health** | **VictoriaMetrics** (`timeseries_instant` / `timeseries_range`) | **~1 min** | the link-quality verdict: signal level + trend, retry / deauth / disassoc **rates**, throughput |
| **2 — structure** | **evidence** (`show_device` `wifiSnapshots`) | discovery cadence (10 min–1 day) | what the client is connected to: serving AP, SSID, band, `connectedSince` — and a **liveness lower bound** (see below) |
| **3 — residual** | **controller API** (`network_http`, UniFi only) | live | channel, noise/SNR, tx rate, `satisfaction` not in VM |

> **VM is the health source — ALWAYS, including when the device is offline. The
> snapshot's frozen health values are not a fallback; they are a confusion trap.**
>
> - **Read every health value from VM (Tier 1).** Signal, SNR, rate, error/retry
>   rates — all of it. Read evidence (Tier 2) only for *structural* facts VM does
>   not carry (serving AP, SSID, band, `connectedSince`).
> - **The `wifiSnapshots[].association` scalars (`signalDbm`, `txRetriesRatio`, …)
>   are frozen at the discovery cadence and may be a day old — or even a
>   different-source value (e.g. an `mdns_probe` reading, not the live link
>   metric). Never read one and present it as current, and do NOT use it as the
>   fallback when VM looks empty.** It looks like a live reading and is not; that
>   is exactly what misleads.
> - **If VM has no *recent* points, do not give up and do not fall back to the
>   snapshot value — query VM for the LATEST available points** (a wide
>   `timeseries_range`, or `timeseries_instant` which returns the last sample).
>   For an **offline** device this is the most useful thing you have: the last
>   point's **value** is the last real measured health, and its **timestamp**
>   tells you roughly *when the device dropped off the air*. That beats the
>   snapshot on both counts.
> - **The snapshot's one genuine use for liveness is as a *bracket*, not an
>   answer.** `observedAt` is a time the device *definitely was* associated — a
>   **lower bound** on "last online", not necessarily the last time it was online
>   (it may have stayed up afterward). VM's last point is the better estimate of
>   when it actually went quiet. Use the snapshot timestamp only to bound, never
>   its frozen health numbers to diagnose.

The interpretation reference is `interpreting-wifi-telemetry` — especially that
the UniFi "Experience"/`satisfaction` score is NOT link quality. Fetch it with
the `get_reference_doc` MCP tool (`name: interpreting-wifi-telemetry`).

## Which metrics exist depends on the platform — read the catalog reference

Different WiFi platforms emit different metric sets (UniFi reports per-client
retries + SNR; the BGW320 reports per-client deauth/disassoc + per-radio error
counters instead). **Do not hardcode this per vendor.** The platform-keyed map of
**which metrics each platform emits, their type (counter/gauge), the PromQL to
query each, and the health bands to score against** is the generated reference
`wifi-metrics-reference` — fetch it with the `get_reference_doc` MCP tool
(`name: wifi-metrics-reference`). The procedure below tells you when to fetch it.

## Procedure

### Step 1 — Structure + platform (evidence, Tier 2)

`show_device` on the client and read `evidence.wifiSnapshots[].association` for
the **structural** facts only: `anchorDeviceId` (serving AP, name-resolved),
`ssid`, `band`, `wired`, `connectedSince`. Also capture the device's
**`device_id`** (you query VM by it) and the snapshot's **`source`** (it is
`"wifi_<platform>"` — strip `wifi_` to get the platform key).

If the `device_id` is **empty** (the client MAC has not yet been promoted to a
Sprinter device — common for a brand-new or randomized-MAC client), the
by-`device_id` VM query in Step 3 will return nothing **for that reason**, not
because the link is quiet. This is the **one** case where VM genuinely has no
series to query at all — distinct from an *offline* device, whose series exists
and still holds its last points (query them — see Step 3). With an empty
`device_id` you can only report structure; if you mention the snapshot's frozen
`signalDbm`, label it explicitly as a stale snapshot reading, never as current
health.

If there is no association (or `wired: true`), the client is wired, offline, or
on a platform the wifi service does not cover. Distinguish: `wired: true` ⇒ stop
(it is Ethernet behind an AP). But **offline is not a dead end** — the device's
VM series still holds its last points, so go to Step 3 and query the **latest
available** points anyway (the last value is the last measured health; its
timestamp dates roughly when it dropped off the air). A live `network_ping`
corroborates the current symptom. Only a genuinely unknown/uncovered platform is
a true stop.

### Step 2 — Look up the platform's metric set (generated reference)

Call the `get_reference_doc` MCP tool with `name: wifi-metrics-reference` and
find the section for this device's platform key (from Step 1's `source`). That
section is the **complete,
authoritative** list of metrics this platform reports, each with its type, health
bands, and the exact PromQL to use. A metric absent from that section is **not
emitted** by this platform — its absence is a signal, not missing data. Use only
this platform's metrics; never assume another platform's metric exists.

### Step 3 — Health (VictoriaMetrics, Tier 1 — the verdict rests here)

For each relevant metric in the platform's section, query VM **by `device_id`**
using the PromQL from the reference. **`network_id` is required, not optional:**
the server-side tenant-isolation rewrite matches **no** series without it, so an
omitted `network_id` returns an empty result that looks exactly like a missing
series. Always pass the client's `network_id`.

**Anchor the range window to the real clock, NOT to the evidence snapshot.** The
`wifiSnapshots[].observedAt` timestamp is the *discovery cadence* time and can be
hours stale — never use it as "now" when choosing `start`/`end`. Run the
`timeseries_instant` first: its returned timestamp **is** the server's current
time. Pick your range `end` from that (or omit `end` to mean "now") and set
`start` to end − a few hours. Anchoring a window to a stale `observedAt` silently
truncates it (e.g. ending the window 5h in the past) so you miss the most recent —
and most relevant — data.

- **Gauges** (signal, SNR, quality index [a 0..95 UniFi index, *not* dBm — see
  the reference], PHY rate): `timeseries_instant` for the current value (and to
  fix "now"); `timeseries_range` with `avg_over_time(...[1h])` for a smoothed
  **trend** (is it drifting worse?).
- **Counters** (retries, deauth, disassoc, trans-errors, bytes): **never read the
  raw total** — query the **rate** with `timeseries_range` and
  `rate(<name>{device_id="<id>"}[15m])`. The per-second rate is the signal (e.g.
  "deauths/sec climbing" = a flapping link). The reference's PromQL column already
  wraps counters in `rate()`.

Score each result against that metric's **health bands** from the reference
(good / fair / poor). The bands are catalog-derived, so your verdict, the UI, and
alerting all agree.

**If the device is offline / the recent window is empty, do NOT fall back to the
snapshot — widen the query and read VM's last points.** `timeseries_instant`
returns the most recent sample even if it is old; or run a wide `timeseries_range`
(e.g. last 24 h) and take the final point(s). The last value is the last real
measured health (grade it, noting it is as-of its timestamp), and the timestamp of
the last point tells you roughly **when the device went quiet**. This is strictly
more informative than the frozen snapshot scalar — use it, not the snapshot.

**Check the point count against the window before trusting a trend.** A range of
`W` at step `S` should return about `W/S` points (e.g. 6 h at 15 m ≈ 24). If you
get **far fewer**, do NOT guess "sparse / just discovered" — **read the returned
timestamps** and diagnose which of two distinct cases it is:

- **Series starts partway into the window** (first timestamp ≫ your `start`, then
  contiguous): the metric only began flowing at that time — a newly-resolved
  client, a just-deployed producer, or a device that only recently associated.
  Report it as "data begins at T", not "polling is intermittent".
- **Holes in the middle** (gaps between contiguous runs — e.g. 01:30 → 03:45
  missing): collection was genuinely intermittent there. Name the gap.

Then **never present a trend that spans a coverage gap as if it were continuous.**
If "trans_errors climbing 0.06 → 1.1 /s over the evening" is read across a 2 h+
hole, say so explicitly — the endpoints are real but the slope between them is
unobserved. State the gap; do not launder missing data into a clean monotonic
story.

### Step 4 — Optional live symptom

If the complaint is "choppy"/"laggy", `network_ping` the client several times to
quantify RTT and jitter — this corroborates (or contradicts) the retry rate.

### Step 5 — Verdict and hand-off

Decide health from the **live VM numbers + their trends** (Step 3), scored by the
catalog bands, with the structural frame from Step 1. Common shape:

- Signal: weak signal (poor band) ⇒ coverage/distance problem.
- Retry / deauth / disassoc **rate** in the fair/poor band ⇒ retransmission-heavy
  / flapping link (RF contention, interference, weak association).
- Where the platform reports SNR / tx-rate (UniFi), a low SNR or pinned-low PHY
  rate corroborates.
- Ping jitter spiky ⇒ matches a high retry rate.

The `satisfaction`/Experience score (UniFi, residual Tier 3) is **not** the
verdict — a near-silent IoT client can show 100 on a bad link (see the
interpretation doc). Report it only for completeness if you read it.

Then:

- **Link healthy** (good bands, steady ping): the Wi-Fi link is not the problem —
  bounce back to `triage-network-complaint`'s other layers (WAN/DNS/device).
- **Weak signal + the client is on a far AP when nearer APs exist, or a mesh AP
  is involved:** this is a roaming/sticky-client problem — hand off to
  `Skill(diagnose-wifi-roaming)` for the topology analysis and the min-RSSI
  safety checks. Do NOT reason about min-RSSI here. The serving AP is
  `association.anchorDeviceId` — if the user says the device "is near AP X" but
  the anchor resolves to a different AP, that is exactly this smell. Include the
  user's expectation in the hand-off args (which AP they thought the device should
  use) — the roaming skill must explicitly assess that AP, including its own
  uplink if it is a mesh node.
- **High retries / noise / low rate without a roaming angle:** report it as RF
  quality — congestion, interference, channel width, or distance — with the
  numbers. Recommendations (channel/width/power) are the operator's to apply;
  this skill only diagnoses. In the verdict, dispose of the obvious remedies the
  user is likely to try (move device/AP, change channel/width, accept as-is)
  explicitly — including any that the data shows won't work, with the why —
  rather than silently omitting them.

## Network-wide audit — one labelled query, not a per-device loop

When the question is about **many or all clients at once** — "which devices have
weak signal / high retries / low rate", "rank every client by <metric>", "is this
a network-wide problem or just this one device" — do **NOT** loop `show_device`
over the device list to scrape `wifiSnapshots`. That is wrong on two counts: the
snapshot value **lags** (it is the discovery-cadence reading, and may even be a
stale `mdns_probe` value hours old, not the live link metric), and it is dozens of
calls where **one** suffices.

**The pattern is general, not signal-specific.** Every `sprinter_wifi_*` series
carries the same label set — `network_id`, `device_id`, `client_mac`, `band`
(plus `ap_mac` on radio metrics) — so **any** metric in the platform's reference
section can be swept across the whole network in a single labelled query, and VM
can sort/filter/threshold it server-side. Drop the `device_id` filter, keep
`network_id`, and let VM do the work:

| Network-wide question | One query (`timeseries_instant`) |
|---|---|
| Every client's current signal, weakest first | `sort(sprinter_wifi_client_signal_dbm{network_id="<net>"})` |
| Only clients below a threshold | `sprinter_wifi_client_signal_dbm{network_id="<net>"} < -75` |
| Every client's SNR, worst first (UniFi) | `sort(sprinter_wifi_client_snr_db{network_id="<net>"})` |
| Highest error/retry **rates** right now | `sort_desc(rate(sprinter_wifi_client_trans_errors_total{network_id="<net>"}[15m]))` |
| Per-radio congestion across all APs | `sort_desc(sprinter_wifi_ap_radio_channel_util_ratio{network_id="<net>"})` |

The result is one series **per client/radio**, each row labelled with its
`device_id` and `client_mac` — fresh (~1 min), ordered by VM. Substitute any
metric name from the platform's reference section; the recipe is identical because
the labels are. Then score each row against the **same catalog health bands** from
the reference, and report the outliers. Use `timeseries_range` (same
dropped-`device_id` form) only when you need each client's *trend*, not just the
current ranking.

**Reading the result: expect one `timeSeries` object per device, not a single
blob.** The tool returns the `probeData.timeSeries[]` array with **one entry per
matched series** — each entry has its own `metricName` (the full label set:
`device_id`, `client_mac`, `band`, etc.) paired with its own `values`. Read each
entry as one device and map its value to its labels directly. If you instead see
**one** `timeSeries` entry whose `values[]` holds *all* the values but whose
`metricName` is just one device's labels, you are talking to an **older,
un-redeployed MCP server** (a fixed-server release returns per-series rows); in
that case the values are still correct and sorted, but the value↔device mapping is
lost — fall back to identifying the few outliers by pinning their `device_id`
individually rather than trusting the single collapsed label set.

**Get device names in the SAME query — join, don't loop.** Don't follow the sweep
with `show_device`/`list_devices` calls to turn `device_id`s into names. There is
an info metric, **`sprinter_deviceInfo`** (value `1`, one series per device,
labels include `device_id`, `device_name`, `device_address`, `device_mac`,
`vendor`, `model`, `device_class`, and `network_id`), purpose-built to be joined.
Pull the name (and anything else) into the result with a PromQL `group_left`:

```promql
sort(
  sprinter_wifi_client_signal_dbm{network_id="<net>"}
    * on(device_id) group_left(device_name, device_address)
  sprinter_deviceInfo{network_id="<net>"}
)
```

Every row now carries `device_name` + `device_address` alongside the value —
no follow-up calls. The recipe generalizes: wrap any sweep metric (or a
`rate(...)` of a counter) as the left side, list whatever `deviceInfo` labels you
want in `group_left(...)`. Note the **empty-`device_id` caveat**: a client whose
MAC never resolved to a Sprinter device has `device_id=""`, so it won't match
`deviceInfo` and drops from the join — that is correct (it has no name), but if you
must still see those rows, run the bare sweep (no join) and report them by
`client_mac`.

Frame the finding correctly: a single client deep in the poor band while the rest
sit good/fair is a **localized** problem (that device's placement/radio); a whole
cluster in the fair/poor band is **network-wide** (coverage/congestion). The
one-query sweep is what lets you tell those apart cheaply.

## Controller-API residual (Tier 3, UniFi only — optional)

For fields neither VM nor evidence carries (channel, noise/SNR if not in the
platform's VM set, the `satisfaction` score), resolve the controller from the
client's own evidence (`wifiSnapshots[].controllerDeviceId`; empty for
cloud-managed/older snapshots → `find_device(device_class="wifi_controller")`),
`show_device` it for its IP, and plain-GET it with `network_http`: a GET/HEAD
whose URL host is the IP of a device with stored credentials is authenticated
automatically with that device's credentials — no auth parameter, credentials
never reach you. Address the controller **by IP** (a DNS hostname runs anonymous).
UniFi example: GET `https://<controller addr>/proxy/network/api/s/default/stat/sta`
with `verify_cert=false`, `return_stats_only=false`. If stored credentials fail,
the call returns an error (class + detail) — report that as "stored credentials
are broken" (user-actionable). On an older deployed server the request runs
anonymous (login page / 401) — stay evidence+VM only and say what couldn't be
read.

## Honesty

- The verdict rests on **live VM** health (signal + retry/deauth rate + trend)
  scored by catalog bands; never on the Experience score, and never on a frozen
  evidence scalar presented as current.
- Don't claim physical placement or "nearest AP" as fact — it's an RSSI
  inference (and if it becomes central, that's the roaming skill's job).
- State what you couldn't read, and **why**, distinguishing the empty-VM cases:
  (a) the client's `device_id` was empty (MAC not yet a Sprinter device), so
  there is **no series at all** to query — the only case where you fall back to
  structure + a labelled-stale snapshot value; or (b) the `device_id` exists but
  the recent window was empty (device offline) — here you do **not** fall back to
  the snapshot, you read VM's **last available points** (last measured health +
  the timestamp it went quiet) and report as-of that time. Also flag no
  controller access (SNR/tx-rate/satisfaction unknown). Say which case you hit
  and what the diagnosis rested on — never present an empty recent window as "the
  link is quiet" when older VM points exist.
