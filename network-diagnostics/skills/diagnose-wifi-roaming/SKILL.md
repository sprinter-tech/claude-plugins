---
name: diagnose-wifi-roaming
description: >
  Diagnose a Wi-Fi client's link quality and sticky-client / roaming problems,
  read-only, from Sprinter's stored WiFi evidence (with a narrow live
  controller-API fallback). The link-quality verdict works on any supported WiFi
  platform; the deeper roaming/mesh/min-RSSI analysis is UniFi-controller-specific
  today and degrades gracefully on other platforms. Use when a device has a
  weak/jittery/retry-heavy wireless link, is "stuck" on a far access point, when
  someone asks why a client won't roam to a closer AP, or when a controller
  reports a healthy "Experience"/satisfaction rating that contradicts observed
  retries/latency. Read-only: it never changes controller or device config —
  it explains what is happening and what the fix would be. The narrow
  roaming/mesh follow-up to diagnose-wifi-basic; often reached from
  triage-network-complaint once a complaint is localized to a Wi-Fi client.
argument-hint: "[device-address-or-id]"
allowed-tools: >
  Bash, Read, Write, Edit, Grep, Glob, Skill, WebSearch,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__network_tech_stack,
  mcp__sprinter__show_device,
  mcp__sprinter__find_device,
  mcp__sprinter__device_presence_history,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__network_issues,
  mcp__sprinter__network_http,
  mcp__sprinter__network_ping
---

# Diagnose Wi-Fi Roaming / Sticky-Client Problems (read-only)

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Sprinter
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

This skill investigates why a Wi-Fi client has a poor link or won't roam to a
nearer access point. The platform-neutral link-quality verdict runs on **any**
supported WiFi platform; the deeper roaming/mesh/min-RSSI analysis is **UniFi
controller**-specific today (UDM / UDM-Pro / Cloud Key) — see Scope below for how
the skill branches by platform. Three data sources, each for a different job:

- **The client's live link quality (signal, retry rate, trend) comes from
  VictoriaMetrics** via `timeseries_instant` / `timeseries_range` — ~1-min fresh,
  queried by `device_id`. This is the **primary** source for "is the link bad
  *now*, and is it getting worse". The per-platform metric set + the PromQL +
  the health bands are in the generated reference — fetch it with the
  `get_reference_doc` MCP tool (`name: wifi-metrics-reference`).
- **The topology — serving AP, the AP fleet, per-AP radios, mesh uplinks —
  comes from evidence** (`show_device` `wifiSnapshots`). Sprinter's wifi service
  persists it on the discovery cadence; it is structural, so its staleness is
  fine. **Do not read the client's health *values* from evidence** — those
  scalars are frozen at the discovery cadence (a day old on real networks); read
  health from VM.
- **The live controller API** is touched only for residual facts: per-SSID
  min-RSSI config, and — when a mesh AP is a roam candidate — the true backhaul
  signal/rates from `stat/device`.

**It is strictly read-only.** It never sets min-RSSI, never reconnects, blocks,
or removes a client, never changes any WLAN or AP config. It diagnoses and
recommends; the human acts. (A future version may add an opt-in "reconnect"
action — out of scope here.)

**Scope — full analysis is UniFi-only; a live-link verdict still runs on any
platform.** The roaming/mesh/min-RSSI analysis below (the AP-fleet scan, the
mesh-backhaul checks, the residual `rest/wlanconf` min-RSSI read) is
**UniFi-specific** and needs a UniFi controller (UDM / UDM-Pro / Cloud Key).
But the **Step 1b live link-quality verdict** (signal + retry/error rate,
read from VM, scored against the catalog bands) is platform-neutral and is
worth running on **any** platform — so do NOT STOP outright on a non-UniFi
client.

Before branching, call `network_tech_stack` and read this network's WiFi
platform note. Roaming/mesh analysis only makes sense on a platform that runs a
**multi-AP fleet and reports per-client association/roaming data** — the note
tells you upfront whether that is the case, plus how many controllers/APs exist.
On a platform that structurally withholds a client roster or per-client health
(e.g. Google Wifi), the roaming-specific analysis is impossible by construction:
say so and skip it, rather than discovering the gap probe by probe.

Branch on the platform key (from Step 1a's `wifiSnapshots[].source`, `wifi_`
stripped):

- **`ubiquiti` (UniFi controller present):** run the whole skill.
- **Any other platform with a section in `wifi-metrics-reference`** (e.g.
  `att-bgw` — a BGW320 gateway): there is no AP fleet to roam across (it is a
  single gateway), so the topology / mesh / min-RSSI analysis does not apply.
  Run **Step 1b only** — the live link-quality verdict and the accept-link /
  coverage portion of the candidate-fix ledger — then say the roaming-specific
  analysis was skipped because this platform has a single gateway, not a
  multi-AP fleet. Do **not** invent a UniFi controller or run the residual
  `rest/wlanconf` / `stat/device` calls.
- **A platform key with NO section in `wifi-metrics-reference`** (unknown /
  unsupported source): report that this platform is not yet supported and stop —
  do not proceed with UniFi assumptions.

The stored `wifiSnapshots` evidence shape is vendor-neutral; only the residual
layer is UniFi-bound. The vendor-neutral *analysis* (the
Experience-vs-link-quality trap and the two min-RSSI failure modes) lives in
the reference `interpreting-wifi-telemetry` — fetch it with the
`get_reference_doc` MCP tool (`name: interpreting-wifi-telemetry`).
The platform keys themselves are enumerated in `wifi-metrics-reference` (step 1);
the planned path to a richer multi-vendor roaming analysis is
`diagnose-wifi-roaming-generalization-todo` — fetch it with the
`get_reference_doc` MCP tool before adding a second controller vendor.

The background reference for *interpreting* the numbers this skill collects is
`interpreting-wifi-telemetry`. Fetch it with the `get_reference_doc` MCP tool
(`name: interpreting-wifi-telemetry`) — the "Experience != link quality" trap and
the "when min-RSSI is the WRONG tool" failure modes are the heart of the
analysis, and this skill is the automated front end to that doc.

No working-directory check is needed to run this skill. Unlike rule-authoring
skills, it neither creates nor edits rule files; the diagnosis is pure
evidence reads plus at most two live controller-API calls. The reference docs
above are reference reading, not a dependency.

## What you need before starting

1. **The client device** the user is asking about — an IP, MAC, hostname, or
   device_id. Resolve it with `find_device` / `show_device`. Note its
   **device_id** and its **network_id**.
2. **The UniFi controller device** on that network. The client's own evidence
   names it: read `evidence.wifiSnapshots[].controllerDeviceId` (empty for
   cloud-managed sources and for snapshots written before the field shipped).
   If it is empty/missing, fall back to
   `find_device(device_class="wifi_controller")` (or `gateway`) on the
   client's network. You need its **device_id** (for `show_device` topology
   evidence) and its **IP address** (the residual API call must target the
   IP — that is what triggers the automatic device auth). For the residual
   min-RSSI check it must also have stored credentials — a `ct_api_token`
   channel (visible as
   `configurationSnapshot.availableDeviceChannels` of type `ct_api_token` in
   `show_device`). The UniFi controller is required only for the roaming/mesh/
   min-RSSI analysis. If there is no `wifi_controller`-class device, branch per
   the scope rule above — for a single-gateway platform (`att-bgw`) run the
   Step 1b live-link verdict only; for an unknown platform stop. If the
   controller exists but has no token channel, run the evidence-only diagnosis
   and **skip the min-RSSI check — and say so in the verdict**.

## Where the data lives — health from VM, topology from evidence

The client's **link quality** (signal, retry rate, and their trend) is read
**live from VictoriaMetrics** by `device_id` — ~1-min fresh. The **topology**
(serving AP, the AP fleet, mesh uplinks) is read from **evidence**
`wifiSnapshots`, which is structural and fine at the discovery cadence.

`show_device` returns `evidence.wifiSnapshots` in its default overview (on
older deployed servers that predate this, `sections="discovery"` also carries
them — use that as a fallback).

| Source                                  | Field                               | Gives you                                                                                                                                                                                         |
|-----------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **VM** (`timeseries_*`, by `device_id`) | `sprinter_wifi_client_*`            | the client's LIVE signal + retry/deauth/disassoc **rate** + trend — the link-quality verdict. Metric set / PromQL / bands per platform: via `get_reference_doc` (`name: wifi-metrics-reference`). |
| **Evidence**                            | `wifiSnapshots[].association`       | STRUCTURE: serving AP (`anchorDeviceId`), `ssid`, `band`, `wired`, `connectedSince`. **Not** the live health values.                                                                              |
| **Evidence**                            | `wifiSnapshots[].controllerSummary` | The whole AP fleet, per-AP radios, client counts, and wired-vs-wireless uplink topology — in ONE call.                                                                                            |

### Structural fields on the client's `association` (evidence)

Read these for **structure** only:

- `anchorDeviceId` — the serving AP, already name-resolved (MAC fallback)
- `ssid`, `band` (`2_4ghz` / `5ghz` / `6ghz`), `wired` (true ⇒ Ethernet
  behind an AP), `connectedSince`, `observedAt`

The evidence also carries frozen `signalDbm` / `txRetriesRatio` scalars — **do
not use them for the verdict**; they are discovery-cadence stale (and may even be
a different-source reading, e.g. `mdns_probe`, not the live link metric). Read the
live signal and retry **rate** from VM (Step 1). The frozen scalar is **not** a
fallback: if the recent VM window is empty (device offline), query VM's **last
available points** instead — that gives you the last measured health *and* the
time it went quiet, both better than the snapshot. Only when VM has **no series at
all** (MAC never resolved to a `device_id`) are you reduced to structure + a
labelled-stale snapshot value — say so.

### Key fields on the controller's `controllerSummary`

- `accessPointsTotal`, `accessPointsOnline`, `clientsTotal`, `clientsWireless`
- `accessPoints[]`: `vendorDeviceId`, `name`, `model`, `mac`, `ip`, `online`,
  `clientCount`, `firmwareVersion`
- `accessPoints[].radios[]`: `band`, `channel`, `channelWidthMhz`,
  `txPowerDbm`, `clientCount`, `channelUtilRatio` (0–1, `-1` = unknown),
  `enabled`
- `accessPoints[].uplink`: `type` (`"WIRE"` vs **`"WIRELESS"`** = mesh),
  `parentMac` / `parentName` / `parentDeviceId`, `backhaulRssiDbm`,
  `backhaulChannel`, `backhaulBand`

**Mesh detection: `uplink.type == "WIRELESS"` is the authoritative
mesh/extender signal.** Never infer mesh from interface names (`apclii0`
etc.) when uplink is present — `backhaulBand` tells you the live backhaul
band directly.

**`backhaulRssiDbm` units trap (UniFi).** Despite the name, on UniFi this
field currently carries the controller's `uplink.rssi`, which is a small
**positive SNR-like number** (e.g. `3`), not negative dBm. Do NOT dismiss a
positive value as a data glitch and move on — a single-digit value means the
backhaul sits only that many dB above the noise floor, i.e. a **very poor
backhaul** (live-confirmed example: evidence `backhaulRssiDbm: 4` ↔ live
`uplink.signal: -92` dBm at 26 Mbps). When a mesh AP's backhaul quality
matters to the verdict (it does whenever that AP is a roam candidate — see
Step 2a), resolve the true figure live from `stat/device` rather than
guessing either way.

### What evidence does NOT carry per client (the residual)

Channel, noise / SNR, tx/rx rate, the UniFi "satisfaction"/Experience score,
and second-by-second liveness. Per-SSID policy (min-RSSI in `rest/wlanconf`)
is also not in evidence. Those need a live controller API call.

## Live controller API (the residual fallback only)

Use `network_http` — auth is **implicit**: a GET/HEAD request whose URL host
is the IP address of a network device with stored credentials is
authenticated automatically with that device's credentials. There is no auth
parameter, and credentials never reach you. Address the controller by its IP
(as reported by `find_device`/`show_device`) — a DNS hostname is not
recognized as a device and the request runs anonymous. Only GET/HEAD are ever
authenticated; the read-only stance is enforced server-side. Example for the
min-RSSI check:

- `url`: `https://<controller-addr>/proxy/network/api/s/default/rest/wlanconf`
- `method`: GET, `verify_cert=false`, `return_stats_only=false`

If the controller's stored credentials fail, the call returns an error (class
+ detail) instead of running anonymous — report that as "stored credentials
are broken"; it is user-actionable. On an older deployed server the request
just runs anonymous (you'll see the login page / a 401) — fall back to
evidence-only and **say what couldn't be read** — for this skill that means
the min-RSSI safety check is skipped.

> The canonical, shared reference for the UniFi endpoint + field maps is
> `unifi-controller-api` — fetch it with the `get_reference_doc` MCP tool
> (`name: unifi-controller-api`). Consult it if anything is unclear or drifts.

## Procedure

### Step 1 — Structure (evidence) + live link quality (VM)

**1a — Structure.** `show_device` on the client; read
`evidence.wifiSnapshots[].association` for the structural facts: `band`, `ssid`,
`anchorDeviceId`, `wired`, `connectedSince`, `observedAt`. Capture the device's
**`device_id`** and the snapshot's **`source`** (`"wifi_<platform>"` → strip
`wifi_` for the platform key). If `wired` is true, the device is Ethernet-attached
behind an AP — say so and stop. If there is no association, it is offline (or
wired); **offline is not a stop** — its VM series still holds the last points, so
proceed to 1b and read them (they date when it dropped off the air). The client's
**channel** is not in evidence; if it matters, it is part of the residual.

**1b — Live link quality (VM, primary).** Call the `get_reference_doc` MCP tool
with `name: wifi-metrics-reference`, find the platform's section, and query VM
by `device_id` for the client's **live** signal and retry rate.

**Anchor the range window to the real clock, NOT to the evidence snapshot.** The
`wifiSnapshots[].observedAt` (Step 1a) is the discovery-cadence time and can be
hours stale — never use it as "now". Run the `timeseries_instant` first: its
returned timestamp **is** the server's current time. Set your range `end` from
that (or omit `end` to mean "now") and `start` to end − a few hours. Anchoring to
a stale `observedAt` silently truncates the window into the past and you miss the
most recent — and most relevant — data.

- **Signal:** `timeseries_instant` on `sprinter_wifi_client_signal_dbm{device_id="<id>"}`
  for the current RSSI (and to fix "now"); `timeseries_range` with
  `avg_over_time(...[1h])` for the trend. (Platforms that report only bars have no
  dBm series — fall back to the evidence `signalBars` and say so.)
- **Retries:** the live retry signal is a **rate**, not the frozen
  `txRetriesRatio`. Where the platform reports `wifi_client_tx_retries_total`
  (UniFi), query `rate(...[15m])`. Where it reports per-client error counters
  instead (BGW320: `deauth_total` / `disassoc_total` / `trans_errors_total`),
  query their rates — a climbing deauth/disassoc rate is the flapping-link signal.

Score against the catalog health bands in the reference. This live signal + retry
rate is what feeds the roaming decision below — **not** the stale evidence scalar.

WiFi series now carry `device_id` and `network_id` stamped at emit (the producer
resolves the client MAC → `device_id` via the graph), and `network_id` is
**load-bearing** for the read: the server-side tenant-isolation rewrite matches
**no** series without it, so always pass the client's `network_id`. Read an empty
result carefully:

- **Empty *recent* window but the series exists** (device went offline): do NOT
  fall back to the snapshot — widen the query (`timeseries_instant`, or a 24 h
  `timeseries_range`) and take VM's **last available points**. The last value is
  the last measured health and its timestamp dates when the link went quiet —
  both more useful than the frozen scalar.
- **Truly no series at all** for a client you confirmed is *associated* (Step 1a):
  the MAC → `device_id` resolution failed (the client MAC is not yet a Sprinter
  device). Only here are you reduced to the evidence `signalDbm` / `txRetriesRatio`
  — label it a stale snapshot value, not current health.

Either way, never report "no data" as if the link were idle when older VM points
exist.

**1c — Roam/disassoc event timeline (the direct evidence of *movement*).** The VM
signal/retry trend tells you the link is *bad*; it does not tell you whether the
client is actually **roaming, stuck, or flapping**. For that, call
`device_presence_history` with the client's `device_id` over the same window you
used in 1b. It returns the device's state-transition timeline interleaved with the
Wi-Fi event stream — each `roamed AP <from_ap> -> <to_ap>` and `disassociated from
AP <ap>` line is a real, timestamped roam/disassoc, not an inference. Read it
alongside the link verdict:

- **Many `roamed` events clustered in time** = the client is bouncing between APs
  (the opposite of sticky) — look for a min-RSSI set too aggressively, or two APs
  at similar RSSI fighting for it (Step 5's "min-RSSI collision" / ping-pong case).
- **Zero `roamed` events while signal is poor and an obviously-closer AP exists** =
  the **sticky-client** signature — the client is glued to a far AP and never
  hands off. This is the canonical roaming complaint; corroborate with the Step 2
  fleet map (is there actually a nearer AP to roam to?).
- **Repeated `disassociated` without a following `roamed`** = the client keeps
  dropping the air entirely (deauth storm / flapping link), not roaming — pair
  with the deauth/disassoc **rate** from 1b.
- **`online -> sleep` presence transitions** are NOT roam faults — `sleep` is a
  healthy power-save state (associated but not answering ping); do not read it as
  a drop.

Omit `start`/`stop` for the last 24h; pass an explicit window to match a specific
complaint ("it was unusable during the 7-9pm call"). On a single-gateway platform
(`att-bgw`) there is no AP fleet, so `roamed` events will not appear — that is
expected, not a gap.

**Check the point count against the window before trusting a trend.** A range of
`W` at step `S` should return about `W/S` points (6 h at 15 m ≈ 24). If you get
**far fewer**, do NOT guess "sparse / just discovered" — **read the returned
timestamps** and say which case it is: (a) the series *starts* partway into the
window (first timestamp ≫ your `start`, then contiguous) ⇒ the metric only began
flowing then (newly-resolved client / just-deployed producer) — report "data
begins at T"; or (b) *holes in the middle* (a gap between contiguous runs, e.g.
01:30 → 03:45) ⇒ collection was intermittent — name the gap. Then **never present
a retry/error trend that spans a coverage gap as if it were continuous**: if a
deauth/error rate "climbs over the evening" across a 2 h+ hole, the endpoints are
real but the slope between them is unobserved — state the gap rather than
laundering missing data into a clean worsening story.

### Step 2 — Map the fleet from the controller's evidence

`show_device` on the controller; read
`evidence.wifiSnapshots[].controllerSummary`. Confirm the client's
`anchorDeviceId` against the `accessPoints[]` list. Then scan the fleet for
**wireless-uplinked APs** (`uplink.type == "WIRELESS"`): note each mesh AP's
parent (`parentName` / `parentMac`), `backhaulRssiDbm`, and `backhaulBand`.
**This is the critical step** — a mesh AP whose backhaul shares the serving
AP (and SSID band) with the troubled client is what makes min-RSSI unsafe.

### Step 2a — Grade the roam candidate's own uplink (mandatory for mesh APs)

If the better-AP candidate — the AP the client "should" be on, whether you
inferred it or the user named it — has `uplink.type == "WIRELESS"`, its own
backhaul becomes part of the client's would-be path, so you MUST grade it
before recommending (or even discussing) a move there. The client's effective
path through a mesh AP is bounded by its **weakest hop**: a strong
client→mesh-AP link buys nothing if the mesh AP's backhaul is as weak as (or
weaker than) the client's current direct link.

Fetch `stat/device` via `network_http` against the controller's IP (same
implicit auth as Step 3), find the mesh AP's row (`type == "uap"`, match by
MAC or name), and read its `uplink` object:

- `signal` — the **true backhaul RSSI in negative dBm** (the evidence
  `backhaulRssiDbm` is the SNR-like `rssi` — see the units trap above)
- `tx_rate` / `rx_rate` — backhaul PHY rates in Kbps; pinned-low rates
  (tens of Mbps on a 5 GHz link) corroborate a starved backhaul
- `ap_mac` / `uplink_device_name` — confirms the backhaul parent

Grade the backhaul with the same `signal_dbm` good/fair/poor bounds the
reference defines for clients (roughly: good ≳ −67, poor ≲ −80 dBm — use the
exact bounds from `wifi-metrics-reference`, not these illustrative figures).
Carry the grade into Step 5's mesh-bottleneck check and into the candidate-fix
ledger. If the live read is unavailable, grade from the
evidence `backhaulRssiDbm` interpreted as SNR (single digits = poor) and say
the true dBm figure was not readable.

**Topology trap — the controller's wired-uplink attribution can lie.** On
sites whose wired fabric is third-party (no UniFi switches), the controller
cannot see the real upstream switch, and a wired AP's uplink-discovery
heuristic can latch onto the only UniFi MAC it hears — often **its own mesh
child** — producing an impossible parent loop (AP A's wireless parent is B,
while B's "wired" parent is A). Resolution rules:

- **Trust the wireless-uplink record over the wired one.** A
  `type: "wireless"` uplink carries device-reported RF facts (`signal`,
  `essid`, channel, PHY rates) and is self-consistent; a `type: "wire"`
  uplink's *neighbor identity* is controller inference and is the part that
  fails.
- **Misattribution fingerprints:** `uplink_remote_port: -1` (remote port
  unknown); the named "neighbor" is a portless mesh-only AP (e.g.
  UAP-BeaconHD, model code `UDMB`, has no Ethernet jack — a wired uplink
  terminating there is physically impossible); honestly-wired APs on the same
  site report bare third-party switch MACs with **no** `uplink_device_name`.
- The wire itself (`up`, `speed`, `full_duplex`) is real even when the
  neighbor name is wrong — and check `speed` against the port: a gigabit
  port negotiated at 100 Mbps (`speed: 100`, `media: "GE"`) means a damaged
  or 2-pair cable that caps the whole branch, a finding in its own right.
- If you detect such a loop, say so in the report and state the inferred true
  topology (wired AP is the anchor; the portless mesh AP hangs off it) —
  do not present the controller's loop as fact.

**Resolve the wired truth independently when SNMP switches exist.** When the
roam candidate's uplink is `type: "wire"` (or the controller's wired
attribution is in doubt per the loop trap above), and the network has
SNMP-capable switches (`find_device(device_class="network_switch")` returns
any with an `snmpv2` service), hand off to `Skill(locate-device-uplink)` with
the AP's wired MAC. It reads the switches' MAC forwarding tables to pinpoint
the AP's real attachment port and grade that link's speed and error counters —
which is the AP's true wired backhaul quality, independent of anything the
controller inferred. A wired uplink capped at 100 Mbps or accumulating errors
caps every client on that AP, so carry the grade into the mesh-bottleneck check
and the candidate-fix ledger exactly as you would a weak wireless backhaul.

### Step 3 — Read the SSID config (live API read)

Fetch `rest/wlanconf` via `network_http` against the controller's IP address
(auth is implicit — see above).
For the client's `ssid`, report `wlan_band` (`both`/`2g`/`5g`) and any
min-RSSI fields. The min-RSSI key names **drift across controller versions**,
so check this candidate set on each WLAN object and report whichever is
present (non-null):

```python
MINRSSI_KEYS = [
    "minrssi", "minrssi_enabled",        # classic legacy network app
    "min_rssi", "min_rssi_enabled",      # newer UniFi OS network app
    "minrssi_mode",                      # some versions: mode enum
    "ap_isolation",                      # unrelated, do NOT confuse — skip
]
# report: {k: w.get(k) for k in MINRSSI_KEYS if k != "ap_isolation" and w.get(k) is not None}
```

If none are present/non-null, say "no per-SSID min-RSSI floor is currently set
(or this controller version stores it under a key not in our candidate set)" —
do not assert "no floor" as fact when it might be a key-name miss. A `both`-band
SSID (`wlan_band == "both"`) means a per-SSID floor applies to BOTH 2.4 and
5 GHz — the trap that severs a 5 GHz mesh backhaul when you only meant to nudge
a 2.4 GHz client.

If this step is unavailable (no `ct_api_token` channel, broken stored
credentials, or an older server that left the request anonymous), **skip it
and carry the gap into the verdict**: the min-RSSI safety check could not be
performed.

### Step 4 (optional) — Confirm jitter with a live probe

If the user mentioned latency/jitter, `network_ping` the client a few times to
show RTT variance corroborating the retry rate. This is one live-probe touch;
the rest is the Step 1b VM health, the evidence topology, and the Step 3 residual.

### Step 4.5 — "When did this start?" and what happened around then

Step 1c already gave you the client's roam/disassoc timeline. Before concluding,
**pin the onset and check whether something on the client's Wi-Fi infrastructure
changed at the same moment** — a roaming problem that began at a specific time is
often a *config push* or an *AP/backhaul event*, not a gradual RF drift, and the
fix is different. Full recipe: fetch `when-did-this-start` with the
`get_reference_doc` MCP tool (`name: when-did-this-start`). Wi-Fi-specific
short form:

1. **Pin the onset `T`** from Step 1c's roam/disassoc clustering (or the VM
   signal/retry inflection from 1b) — when did the bouncing / sticky pattern
   begin?
2. **Sweep the client's Wi-Fi dependencies around `T`** (`T ± 15 min`), NOT every
   device on the network:
   - **The serving AP** (`anchorDeviceId`) and any roam-candidate AP — run
     `device_presence_history` on each: did one go `offline -> online` (an AP
     reboot) at ≈ `T`? An AP bounce re-shuffles every client and looks like a
     roaming storm.
   - **The controller** — a controller-pushed change (SSID edit, band steering,
     a min-RSSI floor set, a channel change) re-shapes roaming for the whole
     fleet at one instant. If `network_issues` over the window flags anything on
     the controller/AP devices, or multiple clients began roaming at the same
     `T`, suspect a config push over an individual-client RF problem.
   - **`network_issues`** over `[T − w, T + w]` for the network: a `pt_dhcp`
     change or `pt_traceroute`/`pt_ping` infra event coinciding with `T` means
     the "roaming" is collateral from a wider change, not a sticky-client fault.
3. **Carry the onset into the verdict.** "All clients on this AP began roaming at
   21:12, when the AP rebooted" → not a min-RSSI problem. "Only this client, no
   infra event in the window, signal has been weak for days" → genuine
   sticky-client / coverage. A clean infra sweep *strengthens* the RF diagnosis;
   a co-occurring AP/controller event *redirects* it.

### Step 5 — Analyse and conclude

Apply `interpreting-wifi-telemetry` (fetched via the `get_reference_doc` MCP
tool, `name: interpreting-wifi-telemetry`):

1. **The contradiction.** The UniFi "satisfaction"/Experience score must not be
   trusted as link quality (it is an airtime/throughput score). The diagnosis
   rests on the **retries + RSSI (+ jitter)** triplet, read **live from VM**
   (Step 1b): a high retry/deauth **rate** (in the catalog fair/poor band) with
   weak `signal_dbm` (in the reference's poor band, roughly ≲ −80 dBm) is a bad
   link, full stop. If the user quotes a
   healthy Experience rating that contradicts this, explain the trap (see the
   doc). If the platform reports no retry series (and only the frozen
   `txRetriesRatio` exists, possibly `-1`), say so and lean on the live RSSI +
   jitter.
2. **Sticky-client check.** Weak RSSI on a same-SSID multi-AP network = the
   client chose to stay on a far AP. A Reconnect (UI) or device-side radio
   bounce only forces re-association and may re-pick the same AP.
3. **Is min-RSSI safe here?** Run BOTH failure-mode checks from the doc:
   - **Mesh-backhaul collision:** does any `WIRELESS`-uplinked AP use the
     *same serving AP* as the client, on the *same SSID band* a per-SSID floor
     would cover? Compare the client's `signalDbm` to the backhaul's
     `backhaulRssiDbm`. If they are close (few dB) or the backhaul is weaker,
     **no floor value is safe** — it evicts the backhaul too (symptom: the
     mesh AP goes isolated/re-adopting). Remember a `both`-band SSID floor
     hits the mesh band even when the client and backhaul are on different
     bands.
   - **Nowhere-to-roam:** is the serving AP plausibly the *closest* AP to the
     client (you can only infer this from RSSI — say so)? If no better AP
     exists, eviction just re-picks the same AP. There is no roaming fix.
4. **Mesh-bottleneck check — would the near AP actually help?** Whenever the
   candidate better AP is a wireless mesh node, decide explicitly whether
   moving the client there improves anything, using the Step 2a backhaul
   grade. Compare the client's current `signalDbm` to the candidate's
   backhaul `signal` (true dBm) and rates. If the backhaul is comparable to
   or worse than the client's current direct link, **roaming there does not
   fix the problem** — the traffic still crosses the same weak RF gap, now as
   two airtime hops (client→mesh AP, then the backhaul, often to the very AP
   the client was "stuck" on). This must be stated in the verdict, because
   "get the device onto the nearby AP" is the fix the user is almost
   certainly expecting.
5. **Verdict.** If either failure mode holds, **min-RSSI is the wrong tool** —
   recommend RF/coverage fixes instead: wire the mesh AP's uplink (removes its
   backhaul from the RSSI game, then a floor becomes safe for real clients),
   add/relocate an AP, or accept a poor-but-working link for a low-stakes IoT
   client. If neither holds AND the client and backhaul are cleanly separable
   (different SSID, or no mesh on that AP), a per-SSID min-RSSI on the client's
   SSID is the durable fix — but present it as the human's action, with the
   value (e.g. start −75 dBm) and the caveat that it evicts *any* client below
   the floor on that AP. If Step 3 was skipped, the recommendation must also
   say the current floor config was not readable.

## Honesty requirements

- **Never claim physical placement as fact.** Neither evidence nor the API has
  location ground truth. "Garage is the closest AP" is an *inference* from
  RSSI and/or the user — label it as such.
- **Trust the stored uplink record over a UI screenshot.** The active mesh
  backhaul band/parent comes from `controllerSummary.accessPoints[].uplink`
  (`backhaulBand`, `parentName`); a dual 2.4/5 readout in the UI lists
  candidates, not the live path. Never fall back to interface-name inference.
  For backhaul *quality*, remember the units trap: `backhaulRssiDbm` is
  SNR-like — the true dBm is the live `stat/device` `uplink.signal`.
- **State data gaps.** The diagnosis is based on **live VM** signal + retry rate
  (+ jitter). If the device was offline, say so and report VM's **last** points
  with the time they stop (when it went quiet) — do not substitute the frozen
  snapshot scalar. Only if VM had **no series at all** (MAC unresolved) did you
  fall back to a snapshot value; label it stale. SNR, tx/rx rate, channel, and
  the satisfaction score are not in evidence — if they were not read live either,
  say so rather than guessing. Same for min-RSSI if Step 3 was skipped.

## Output

Give the user: (1) the link-quality verdict with the actual numbers
(signal / retry% — and satisfaction only if it was read live), (2) the
topology finding (serving AP name, any mesh backhaul that collides — with the
backhaul's true signal/rates when a mesh AP was a roam candidate), (3) the
candidate-fix ledger below, and (4) the recommended action framed as theirs
to take. Offer to record a novel finding into the `interpreting-wifi-telemetry`
reference (fetched via the `get_reference_doc` MCP tool) if the case exposes a
pattern the doc doesn't already cover.

### The candidate-fix ledger (required in the final verdict)

Enumerate the obvious/typical remedies for the diagnosed problem class and
dispose of each one explicitly — **"works here"**, **"won't work here because
<evidence>"**, or **"couldn't assess because <data gap>"**. For a
sticky-client / weak-link diagnosis the standard candidates are:

1. **Reconnect / kick the client** (re-association is a coin-flip; may
   re-pick the same AP)
2. **Per-SSID min-RSSI floor** — run both failure-mode checks (mesh-backhaul
   collision, nowhere-to-roam) and, when the trap applies, spell it out
   concretely: which SSID/band coupling, what gets evicted, what the user
   would observe (mesh AP isolated / re-adopting)
3. **Get the client onto the nearer AP** — decided by the mesh-bottleneck
   check when that AP is a mesh node (cite the backhaul's signal/rates)
4. **Wire the mesh AP's uplink, or add/relocate a wired AP**
5. **Accept the link as-is** (often right for a low-stakes IoT client)

Two rules govern the ledger:

- **Investigate before you dispose.** If a remedy is plausible and typical
  for the problem at hand and the data to support or refute it is available
  (in evidence or one residual call), collect that data and judge it — do not
  wave it off on an assumption or an "anomalous-looking" field.
- **Refuted remedies stay in the report.** The fix the user is most likely to
  try on their own (or read on any forum) must NOT be silently omitted just
  because you determined it won't work — the won't-work-and-why, stated with
  the numbers, is often the most valuable part of the verdict. A refutation
  made mid-investigation that never reaches the final write-up is a miss.

## Standing assumption: no physical-layout ground truth

**Always operate as if you do NOT know the physical placement of APs or
clients.** There is no evidence or controller-API source for it, and in
practice users rarely have a map to share. So the "nowhere to roam" check is
ALWAYS an inference from RSSI (and whatever the user volunteers) — state it as
an inference, never as fact, and never block a diagnosis waiting for layout
data.

If a user *does* volunteer a floor plan / AP-location map, it sharpens the
nearest-AP reasoning considerably — but treat that as a lucky bonus, not a
prerequisite. We will wire placement into this skill only if and when such
data becomes routinely available; until then, RSSI inference is the method.

(Note: the live reads are intentionally narrow — per-SSID min-RSSI
(`rest/wlanconf`) because it is config the wifi service does not snapshot,
and the mesh backhaul's true signal/rates (`stat/device`) because the
evidence `backhaulRssiDbm` carries the SNR-like `uplink.rssi`, not dBm. The
durable speed-up is the `MINRSSI_KEYS` candidate list above, which saves
rediscovering the version-specific key names.)
