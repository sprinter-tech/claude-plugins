---
name: locate-device-uplink
description: >
  Locate where a device attaches to the network fabric — which switch and
  physical port, or which access point — and grade that attachment. Reads
  Sprinter's persisted topology graph, which already resolved the attachment
  from LLDP / switch MAC-forwarding-table / Wi-Fi-association evidence and
  recorded how it knows and how confident it is. Use when the user asks "which
  switch port is this device on?", "where is this device plugged in?", "what is
  this device connected to?", "is this device's wired link healthy?", "find the
  uplink for this AP/camera/server", or when a topology answer from a Wi-Fi
  controller or other source looks wrong and needs independent confirmation.
  Works for ANY device, wired or wireless. Other skills (troubleshoot-device,
  diagnose-wifi-roaming) hand off here for attachment location and uplink
  grading.
argument-hint: "[device-address-or-mac]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill,
  mcp__sprinter__topology_neighbors,
  mcp__sprinter__topology_path,
  mcp__sprinter__show_network,
  mcp__sprinter__list_networks,
  mcp__sprinter__show_device,
  mcp__sprinter__find_device,
  mcp__sprinter__snmp_get,
  mcp__sprinter__snmp_walk,
  mcp__sprinter__network_ping,
  mcp__sprinter__network_arp_table,
  mcp__sprinter__show_device_classes,
  mcp__sprinter__ask_user,
  mcp__sprinter__get_reference_doc
---

Locate the uplink for $ARGUMENTS

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Sprinter
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

## What this skill does

Given any device (by IP, MAC, hostname, or device_id), find **what it attaches
to** — the switch and physical port for a wired device, the access point for a
wireless one — and **how healthy that attachment is** (link speed, operational
status, error and discard rates).

**The attachment comes from the topology graph, not from a live SNMP walk.**
Sprinter's topology resolver continuously reads switch LLDP neighbor tables and
MAC-forwarding tables (FDB), plus Wi-Fi controller association data, and persists
the resolved attachment as a graph edge — together with **how** it knows
(`method`) and **how confident** it is (`confidence`). One `topology_neighbors`
call returns that answer, along with the `if_index` you need to grade the link
from stored metrics.

This matters: walking the FDB yourself costs several SNMP round-trips per switch
and produces a **weaker** answer than the graph already has. Where a switch
speaks LLDP, the graph carries an authoritative neighbor record; an FDB walk can
only *infer* an edge port from MAC counts.

This is **read-only**. State that contract if the user asks what you will do.

Background reference (fetch via the `get_reference_doc` MCP tool, `name:
topology-graph-reference`): how to read the graph output, the full `method` /
`status` / `reason` vocabularies, and when a live probe *is* still warranted.

## MCP Server Availability — Check First

Before starting, verify the Sprinter MCP tools (`mcp__sprinter__*`) are
available. If any call fails with a connection, authentication, or
"server disconnected" error, **stop immediately** and tell the user the
Sprinter MCP server is unavailable and must be reconnected. Do not work around a
failed MCP connection by guessing or using placeholder data.

## Step 0 — Resolve the network and the target

If you do not already have a `network_id`, hand off to `Skill(select-network)`
or resolve it from context. Then `find_device` the target to get its
**`device_id`** — that is the key `topology_neighbors` is anchored on. (You no
longer need the MAC: the graph is keyed by `device_id`, and the resolver already
did the MAC→port matching, including for Wi-Fi clients with randomized MACs.)

## Step 1 — Read the attachment from the graph (one call)

```
topology_neighbors(network_id=<net>, device_id=<target device_id>, depth=1)
```

That is the whole location step. The result is an edge list; the edge that
matters is the one **out of the target device**:

```
Tesla (7e57d79c) --ATTACHED_TO:wifi:WIFI_ASSOC:0.90--> ap-garage (a1b2c3d4)  [1 hop, upstream]
    ssid=Crocodile band=5ghz
```

```
hub (f7596f98) --ATTACHED_TO:wired:LLDP:0.95--> was-sw1201 (3c0ca753)  [1 hop, upstream]
    port=1/0/32 if_index=32
```

Read off:

- **The infrastructure peer** — the switch or AP the device attaches to.
- **`port=` / `if_index=`** — the physical port (wired). `if_index` is the join
  key into the interface metrics (Step 2).
- **`ssid=` / `band=`** — the wireless attachment (Wi-Fi).
- **`method` and `confidence`** — see Step 3; this is what tells you whether to
  trust the answer or reach for a live probe.

**Want the upstream chain too?** `depth=2` or `depth=3` walks device → AP/switch
→ gateway → ISP in the same call. Use it when the question is "did something
upstream of this device break", not just "which port is it on".

**No attachment?** The tool reports `found: false` with a placement `status`
(`PLACED` / `PLACED_BY_HINT` / `UNPLACED`) and a `reason`. That is a **real
answer**, not an error — report it. `ARP_NO_MATCH` (by far the most common),
`NOT_IN_FDB`, `NO_FDB_SOURCE`, and the rest each say something specific; the
`topology-graph-reference` doc lists them all. Do **not** invent a port.

## Step 2 — Grade the link (hand off to `interface-metrics`)

With the switch and the `if_index` from Step 1, hand off to
`Skill(interface-metrics)` with the switch's `device_name` and that `if_index`.
It reads the stored `sprinter_interface_*` series and returns throughput vs
speed, error and discard **rates** over a window, and link state.

**Port identity comes from the graph; link health comes from the time series.**
Never sample a live SNMP counter twice with a sleep to fake a rate — the stored
series already gives you rates, and `interface-metrics` handles the no-series
fallback (a single live `snmp_get`, reported honestly as a cumulative snapshot).

Carry its verdict into Step 4: a port erroring/discarding, flapping, or
negotiated below its capable speed caps every device on the branch.

## Step 3 — Judge the evidence: `method` + `confidence`

The graph tells you not just *what* it concluded but *how*. This replaces the
old MAC-count edge-vs-uplink heuristic — the resolver already made that
determination, and it made it with more evidence than a single FDB walk has.

| `method`     | Trust   | What it means                                  |
|--------------|---------|------------------------------------------------|
| `LLDP`       | High    | Switch and device told us. Authoritative.      |
| `SNMP_FDB`   | High    | Read from the switch's MAC-forwarding table.   |
| `WIFI_ASSOC` | High    | Controller reports this client on this AP.     |
| `WIFI_MESH`  | Med     | Mesh association.                              |
| `ARP`        | **Low** | Inferred from ARP. Usually carries **no port** |
| `MANUAL`     | —       | A human pinned it.                             |

Note the exact strings: **`SNMP_FDB`**, not `FDB`; **`WIFI_ASSOC`**, not `WIFI`.

`confidence` is 0..1. An `LLDP:0.95` edge is authoritative. An `ARP:0.3` edge
with no `port=` is the graph telling you honestly that it does not really know —
**that, and only that, is when a live probe earns its cost** (Step 5).

> **`MESH_ANCHOR` edges carry NO confidence — it renders as `-`, not `0.00`.**
> Absent is not zero. Do not read a missing confidence on a mesh anchor as
> low confidence and go probing.

**Behind an unmanaged switch.** When the graph places a device on a port that
also carries many other devices, or leaves it `UNPLACED` with `MULTI_MAC_PORT`,
the device sits behind an unmanaged (or SNMP-silent) switch or an AP. Say so
honestly rather than claiming a port.

## Step 4 — Present findings

Report, for the target device:

- **Attachment point** — switch name + physical port (or AP + SSID/band), with
  the `method` and `confidence` that back it. "Attached to `was-sw1201` port
  `1/0/32` (LLDP, confidence 0.95)" is a much stronger statement than a bare
  port number, and the qualifier is the point.
- **Link grade** — speed, operational status, and the error/discard **rates**
  from `interface-metrics` (Step 2). Flag a gigabit-capable port negotiated at
  100 Mbps, or a sustained nonzero error/discard rate, as a finding in its own
  right — that caps every device on the branch. If `interface-metrics` had to
  fall back to a single live counter snapshot, say the number is
  cumulative-since-boot, not a rate.
- **Upstream chain** — if you ran `depth>1`, the chain of links up to the
  gateway and ISP.
- **Topology corrections** — when the graph contradicts a controller's reported
  topology, state the corrected topology and note that the controller's was
  inference (see the wired-attribution trap, via `get_reference_doc`, `name:
  unifi-controller-api`). The graph's LLDP/FDB view is independent of any
  controller.

**Honesty about limits.** State what you could NOT determine, and say which
`method`/`confidence` the answer rests on. The graph cannot tell whether a
100 Mbps port is a damaged/2-pair cable or a 100M-only far-end device negotiating
down — that is device *configuration*, and it needs `dot3StatsDuplexStatus`
(EtherLike-MIB `.1.3.6.1.2.1.10.7.2.1.19.<ifIndex>`). Do not assert the cause;
name the ambiguity and the next probe that would resolve it.

**Truncation is never silent.** If the output ends with
`# TRUNCATED: showing N of M edges`, say so or re-run with a higher `max_edges`.

## Step 5 — The live-SNMP escape hatch (only when the graph says it is unsure)

Live SNMP is for **configuration** and for **facts we do not collect** — not for
re-deriving topology the graph already resolved. Reach for it **only** when:

1. **The graph says it is unsure** — low `confidence` (e.g. `ARP:0.3` with no
   `port=`), or the device is `UNPLACED`. The `method`/`confidence` fields exist
   precisely so you can see this.
2. **You need a fact the graph does not store** — e.g. `dot3StatsDuplexStatus`
   to separate a damaged cable from a 100M-only far end. That is configuration,
   not topology.
3. **You suspect staleness** — the device has been silent long enough to age out
   of the switch's FDB. In practice this surfaces as case 1 (`UNPLACED`).

**Refreshing a stale FDB entry is cheap and often enough**: `network_ping` the
device to make the switch re-learn its MAC, then re-run `topology_neighbors`.
Try that before a full FDB walk.

When you do fall through, walk the FDB yourself:

- Enumerate switches: `find_device(network_id=<net>, device_class="network_switch")`.
  **The class string is `network_switch`** (NETWORK category) — `switch` is a
  smart *light* switch (POWER category). The inline `services=[...]` shows an
  `snmpv2` entry when SNMP is reachable, so this one call pre-filters.
- **Q-BRIDGE first, BRIDGE as fallback.** A VLAN-aware switch advertises both
  MIBs but parks its FDB under Q-BRIDGE and leaves the classic table **empty** —
  walking the wrong one and concluding "device not on this switch" is the trap.
  - Q-BRIDGE: `snmp_walk(oid=".1.3.6.1.2.1.17.7.1.2.2.1.2")` — suffix is
    `<VLAN>.<6 MAC octets in decimal>`, value is the **bridge port**. Match on
    the **last six** octets.
  - Classic BRIDGE: `snmp_walk(oid=".1.3.6.1.2.1.17.4.3.1.2")` — suffix is the
    6 MAC octets.
- **Bridge port ≠ ifIndex.** Always walk `dot1dBasePortIfIndex`
  (`.1.3.6.1.2.1.17.1.4.1.2`) to map bridge port → ifIndex. The identity map is
  vendor-dependent; never assume it.
- Convert the MAC to its decimal suffix with `Bash` (`python3`), and tally MACs
  per port there too — do not eyeball a 50-row table. **1–2 MACs on a port** =
  edge/access port (the device is attached here); **many MACs** = uplink/trunk
  (the device is downstream, attached elsewhere).
- Then read `ifName` (`.1.3.6.1.2.1.31.1.1.1.1`, base64 octet strings — decode
  with `Bash`) and `ifAlias` (`.1.3.6.1.2.1.31.1.1.1.18`) to name the port, and
  hand `if_index` to `interface-metrics` as in Step 2.

**Say in the report that you fell through, and why** — "the graph placed this
device only by ARP (confidence 0.3, no port), so I walked the switch FDB
directly". A live walk that merely re-confirms a high-confidence LLDP edge is
wasted cost; do not run one.

The full methodology, worked example, and OID reference live in the
`snmp-topology-discovery-skill-design` doc (fetch via `get_reference_doc`).

## Guards and failure modes

- **Trust the graph by default.** Its LLDP evidence is *better* than what an FDB
  walk can infer. Fall through to live SNMP only on the Step 5 triggers.
- **`found: false` is an answer.** Report the `status` and `reason`; never invent
  a port. `ARP_NO_MATCH` is the dominant reason and means exactly what it says.
- **Absent ≠ zero.** `MESH_ANCHOR` has no `confidence`; it renders `-`.
- **Names are not unique.** Real networks contain several distinct devices
  sharing one name — the output prints `name (device_id-prefix)` for exactly this
  reason. Disambiguate by id, not name.
- **Stale attachment**: a device silent long enough ages out of the FDB. It
  surfaces as `UNPLACED`; `network_ping` then re-query before escalating.
- **Rates come from metrics, not a sleep** (Step 2): grade the link via
  `interface-metrics` (stored `sprinter_interface_*` series), never by sampling
  a live counter twice with a delay.

## Time travel — "it used to work"

`topology_neighbors` takes an `as_of` (RFC3339). Run it now, run it again as of
when things worked, and diff. A device that silently moved to a different switch
port, or an extender whose backhaul flipped from wired to wireless, shows up
immediately — and that diff is often the whole diagnosis. There is no live-SNMP
equivalent of this: the switch only knows *now*.

## When this skill is reached from another skill

- **From `troubleshoot-device`** (its "find which physical switch port" step):
  return the attachment point (with `method`/`confidence`) and the link grade;
  that skill folds it into its device summary.
- **From `diagnose-wifi-roaming`** (grading a mesh AP's real wired uplink, or
  confirming/refuting the controller's wired-uplink attribution): return the
  switch port the AP attaches to and its link grade, or say honestly that the AP
  is behind an unmanaged segment. The graph's view is derived from the switches'
  own LLDP/FDB, independent of anything the controller inferred — which is
  exactly why it can refute the controller. **This verdict covers ONLY the wired
  backhaul** — make that explicit so it is not read as a complete uplink verdict.
  The **wireless** side of the AP (per-client signal/SNR/retries, radio health)
  is graded separately by the caller via `diagnose-wifi-basic` /
  `diagnose-wifi-roaming` against the WiFi catalog bands (via
  `get_reference_doc`, `name: wifi-metrics-reference`); this skill emits no Wi-Fi
  metric.

## Seeing the whole picture in the UI

The same graph this skill reads is rendered network-wide in the Sprinter web UI
under a network's **Topology** tab: an interactive view of devices, their
switch/AP attachments, and the gateway→ISP uplink, with each device colored by
presence. When a user wants to *see* where a device sits in the fabric (rather
than get a single port verdict), point them there; this skill remains the tool
for a precise, evidence-backed single-device port location and link grade.
