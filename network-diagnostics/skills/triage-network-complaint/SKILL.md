---
name: triage-network-complaint
description: >
  Localize a vague network complaint ("internet is slow", "video calls keep
  freezing", "it works then stops", "pages won't load", "this device keeps
  going offline") to a layer — ISP/WAN, gateway, DNS, Wi-Fi, or a single device
  — using cheap broad checks, then dispatch to the right specialist skill. Use
  this FIRST when a complaint does not already name a specific cause. Read-only:
  it observes and routes, it does not change anything.
argument-hint: "[network-id-or-name] [optional: affected device]"
allowed-tools: >
  Bash, Read, Skill,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__show_network,
  mcp__sprinter__list_networks,
  mcp__sprinter__find_device,
  mcp__sprinter__show_device,
  mcp__sprinter__device_presence_history,
  mcp__sprinter__network_issues,
  mcp__sprinter__issue_chart,
  mcp__sprinter__isp_info,
  mcp__sprinter__ioda,
  mcp__sprinter__network_info,
  mcp__sprinter__network_ping,
  mcp__sprinter__network_dns_lookup,
  mcp__sprinter__network_speed_test,
  mcp__sprinter__network_jitter_test,
  mcp__sprinter__network_traceroute
---

# Triage a Network Complaint

Most real complaints are **underspecified**: "is the internet slow today?" can
mean a download-speed drop, DNS failures, intermittent dead spells, one bad
device, or an ISP outage. This skill's job is **localization, not deep
diagnosis** — narrow the vague report to a layer, then hand off. It is
**read-only**.

No working-directory check needed (no repo writes). It dispatches to other
skills, which run their own checks.

## Step 1 — Minimal intake (don't over-ask)

Pin down only what changes the search:
- **Scope:** one device, or everything? (One device → likely that device's
  link, not the network.)
- **When / since when, constant or intermittent?** Intermittent + time-of-day
  pattern points at congestion/Wi-Fi; constant points at config/ISP.
- **What "slow/broken" means to them:** can't load pages (DNS/WAN), buffering
  video (throughput/jitter), calls drop (loss/jitter/roaming), specific app.

Resolve the network (`show_network` / `list_networks`) and any named device
(`find_device`). Get `network_id` and `tenant_id`.

## Step 2 — Broad instruments first (cheap, in parallel)

Run the wide checks before any narrow probe. These tell you which layer to
suspect:

- **What has Sprinter already flagged?** `network_issues` over the complaint
  window (loss/RTT/variance shifts, DNS/DHCP/HTTP probe issues). This is the
  highest-signal first look. Use `issue_chart` to see a flagged metric over
  time.
- **"This device keeps going offline / drops intermittently" (a named device):**
  call `device_presence_history` with the device's `device_id` over the complaint
  window. It returns the device's **state-transition timeline** (online ⇄ sleep ⇄
  offline, plus Wi-Fi roam/disassoc events), which localizes an intermittency
  complaint in one cheap read — a single `network_ping` only sees up-or-down
  *now*. Read it as: `online <-> offline` flapping = a real drop pattern;
  `online -> sleep -> online` = a healthy power-saving client (`sleep` =
  associated-but-dozing, not a fault); clustered `roamed`/`disassociated` events
  = a Wi-Fi roaming problem (→ `diagnose-wifi-roaming`). Omit `start`/`stop` for
  the last 24h, or pass an explicit window for "what happened last night".
- **Upstream / ISP:** `isp_info` for the ASN, then `ioda` over the window for a
  known ISP/regional outage. Rules out "it's not us."
- **Gateway / network basics:** `network_info` (gateway, addressing),
  `network_ping` the gateway and a public anchor (e.g. 1.1.1.1) for loss/RTT.
- **DNS:** `network_dns_lookup` a couple of names — resolution failure or slow
  resolver is a classic "internet is broken" that is neither WAN nor Wi-Fi.
- **Throughput / jitter** (only if "slow"/"buffering"): `network_speed_test`,
  `network_jitter_test`.
- **Wi-Fi link (if a device is named or the complaint smells wireless):**
  `show_device` on the client and read `evidence.wifiSnapshots[].association`
  — `signalDbm` (RSSI; some sources only give `signalBars` 0–5), `band`,
  `txRetriesRatio` (0..1; -1 = not reported), and the serving AP
  (`anchorDeviceId`). Also read `wifiSnapshots[].source` — it is
  `wifi_<platform>` and names the platform: **`wifi_ubiquiti`** (UniFi) is
  **rich** per client (signal, SNR, retries), while **`wifi_att-bgw`**
  (BGW320 gateway) is **sparse** (no per-client SNR/retries — only signal +
  deauth/disassoc). Missing SNR/retries on a BGW320 is **expected, not a
  fault** — do not read a sparse snapshot as a problem. `wired: true` means
  Ethernet behind an AP — rule Wi-Fi out. This is Sprinter's own evidence,
  polled from the controller about every minute (`observedAt` 1–2 min stale is
  fine); this snapshot read is a **routing signal only** — the authoritative
  live-VM read and catalog-band grading happen in `diagnose-wifi-basic`, so no
  controller API call or band-grading is needed here. (Metric names and health
  bands: fetch via `get_reference_doc`, `name: wifi-metrics-reference`.) If the
  overview lacks `wifiSnapshots` (older server), retry with
  `sections="discovery"`.

## Step 3 — Localize to a layer

From the above, decide where the problem lives:

| Signal                                               | Layer            |
|------------------------------------------------------|------------------|
| `ioda`/`isp_info` shows upstream outage              | ISP / WAN        |
| Gateway ping bad, WAN side bad, LAN side fine        | Gateway / WAN    |
| DNS lookups fail/slow, ping-by-IP fine               | DNS              |
| Speed test low but loss/latency fine                 | Throughput / WAN |
| Only one device bad; others on same AP fine          | That device      |
| Wi-Fi client(s): weak signal / high retries / jitter | Wi-Fi            |

## Step 4 — Dispatch

- **Wi-Fi, single client or "slow/choppy on wireless":** invoke
  `Skill(diagnose-wifi-basic)` with the device — the general Wi-Fi link
  health check.
- **Wi-Fi, "stuck on far AP" / won't roam / mesh involved:** invoke
  `Skill(diagnose-wifi-roaming)`. The association evidence helps pick this
  route: weak `signalDbm` while `anchorDeviceId` names a distant AP is the
  roaming-shaped signature.
- **Wi-Fi, "which/how many devices have weak signal / high retries" (the whole
  network, not one client):** invoke `Skill(diagnose-wifi-basic)` — it answers
  fleet questions with a **single** network-scoped VM query that even joins in
  device names (`... * on(device_id) group_left(device_name) sprinter_deviceInfo`),
  not a per-device `show_device` loop. Do not scrape `wifiSnapshots` signal
  device-by-device here; the snapshot lags and that is the slow, wrong path.
- **Unknown single device acting up:** `Skill(troubleshoot-device)` first to
  learn what it is and check its identity/connectivity (it asks "what is this
  device?" and hands off to the Wi-Fi specialists if the device turns out to be
  wireless).
- **ISP/WAN, gateway, DNS, throughput:** no specialist skill exists yet —
  report the localized finding and the evidence directly. These are good
  candidates for future triage→specialist funnels; note that to the user.

## Step 5 — "When did this start?" before you hand off

Localization answers *where* the problem lives; the specialist will answer *why*.
But one cheap correlation pays off before (or alongside) the hand-off: **pin when
the trouble started and sweep what else happened at that moment.** A complaint
that began at 21:14 is far more diagnosable once you see the gateway's DHCP
config changed at 21:13, or the serving AP went offline at 21:12 — and that
co-occurrence often *is* the localization.

Read-only, on tools already listed above. Full recipe: fetch via
`get_reference_doc`, `name: when-did-this-start`. Short form:

1. **Pin the onset `T`.** If a device is named, `device_presence_history` over
   the complaint window — the first `online -> offline` flap or clustered
   `roamed`/`disassociated` is the onset (an `online -> sleep -> online` cycle is
   healthy power-save, not an onset). For a network-wide complaint, the earliest
   significant `network_issues` `startTime` is the onset.
2. **Sweep a tight window around `T`** (`T ± 15 min`) with `network_issues`,
   reading events by `probeType`: **`pt_dhcp`** (a config change / rogue server
   on the gateway at ≈ `T` explains "lost connection" / "weird IP" for the whole
   LAN), **`pt_traceroute`** (path change upstream), `pt_ping`/`pt_dns` on the
   gateway (infra-wide). DHCP and traceroute events have **no** dedicated history
   tool — they come through `network_issues`. Check `isp_info`/`ioda` for an ISP
   outage that began at `T`.
3. **Let the correlation sharpen the dispatch.** A DHCP/path/ISP event at ≈ `T`
   re-points triage away from "one device" toward the infra layer — say so and
   route accordingly. A clean sweep around a single device's onset reinforces
   "this device" as the layer.

This is localization-grade (cheap, broad), not the specialist's deep timeline —
note onset + any co-occurring infra event in the hand-off so the specialist
starts from it.

## Honesty

- Report what you checked and what you did NOT (e.g. "did not run a speed test
  because the complaint was DNS-shaped"). Don't imply full coverage.
- "It's the ISP" requires positive evidence (`ioda`/`isp_info`), not just
  "everything looks fine locally."
- If the broad checks come back clean, say so — "no current network-level issue
  detected in the window; if it was intermittent, give me a tighter time range"
  — rather than inventing a cause.
