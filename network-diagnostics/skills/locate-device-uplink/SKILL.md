---
name: locate-device-uplink
description: >
  Locate where a device physically attaches to the wired fabric and grade
  that attachment, using switch MAC forwarding tables over SNMP. Use when the
  user asks "which switch port is this device on?", "where is this device
  plugged in?", "is this device's wired link healthy?", "find the uplink for
  this AP/camera/server", or when a topology answer from a Wi-Fi controller or
  other source looks wrong and needs independent confirmation. Works for ANY
  device (not just Wi-Fi) as long as the network has SNMP-capable managed
  switches. Other skills (troubleshoot-device, diagnose-wifi-roaming) hand off
  here for switch-port location and wired-uplink grading.
argument-hint: "[device-address-or-mac]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill,
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

Locate the wired uplink for $ARGUMENTS

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Sprinter
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

## What this skill does

Given any device (by IP, MAC, hostname, or device_id), find **which switch and
physical port it attaches to** and **how healthy that attachment is** (link
speed, operational status, error and discard counters). It does this by reading
the switches' bridge MAC forwarding tables (FDB) over SNMP — the same data a
network engineer reads to trace a cable.

This is **read-only**. Every SNMP operation is a GET or WALK. Nothing changes
switch or device state. State this contract if the user asks what you will do.

This skill is **generic** — it is not Wi-Fi specific. An AP, a camera, a server,
a printer, or a laptop are all located the same way. It is also
**vendor-independent**: it reads standard MIBs and works on any SNMP-capable
managed switch, which makes it the antidote to a Wi-Fi controller (or any other
system) that is blind to third-party switches and mis-reports topology.

Background reference (ships with this plugin): call the `get_reference_doc` MCP tool with
`name: snmp-topology-discovery-skill-design` — the full methodology, worked example, and OID
reference.

## MCP Server Availability — Check First

Before starting, verify the Sprinter MCP tools (`mcp__sprinter__*`) are
available. If any call fails with a connection, authentication, or
"server disconnected" error, **stop immediately** and tell the user the
Sprinter MCP server is unavailable and must be reconnected. Do not work around a
failed MCP connection by guessing or using placeholder data.

## Step 0 — Resolve the network and the target

If you do not already have a `network_id`, hand off to `Skill(select-network)`
or resolve it from context. Then `find_device` the target to get its **MAC
address** (the key the FDB is indexed by) and its current IP. If you were handed
a MAC directly, use it as-is. Match the FDB on the address Sprinter recorded for
the device — Wi-Fi clients with private/randomized MACs appear under that
randomized MAC, so do not substitute a vendor-OUI guess.

## Step 1 — Enumerate SNMP-capable switches (one call, read evidence — do not probe)

```
find_device(network_id=<net>, device_class="network_switch")
```

**The class string is `network_switch`** (NETWORK category). Do NOT use
`switch` — that is a smart *light* switch (POWER category) and will return the
wrong devices. `find_device` lists each device's `services=[...]` inline; an
`snmpv2` entry there means SNMP is reachable, so this one call already
pre-filters to the switches worth walking.

Sprinter records SNMP capability in evidence, so you never probe a device just
to learn whether it speaks SNMP. If you need the authoritative MIB list for a
switch, `show_device(query=<switch>, sections="discovery")` returns:

- `evidence.sharedServices[]` — an entry with `type: "snmpv2"` and
  `attributes.supported_mibs` (e.g. `"BRIDGE-MIB,ENTITY-MIB,IF-MIB,Q-BRIDGE-MIB"`).
- `evidence.supportedMibs` — a per-MIB map of one representative OID each.

The two MIBs that decide FDB strategy:

| MIB present    | FDB table to walk                             |
|----------------|-----------------------------------------------|
| `Q-BRIDGE-MIB` | `dot1qTpFdbPort` (per-VLAN) — **prefer this** |
| `BRIDGE-MIB`   | `dot1dTpFdbPort` (classic, VLAN-unaware)      |

**Critical:** a VLAN-aware switch advertises BOTH MIBs but parks its forwarding
database under the Q-BRIDGE table and leaves the classic `dot1dTpFdbPort`
**empty**. When `Q-BRIDGE-MIB` is present, walk `dot1qTpFdbPort` first; fall
back to `dot1dTpFdbPort` only if Q-BRIDGE is absent or its walk returns nothing.
Walking the wrong table and concluding "device not on this switch" is the trap.

## Step 2 — Walk each switch's FDB

**Q-BRIDGE path (preferred):**

```
snmp_walk(address=<switch IP>, oid=".1.3.6.1.2.1.17.7.1.2.2.1.2", total_time="40s")
```

Each returned OID's suffix is `<VLAN>.<6 MAC octets in decimal>`; the value is
the **bridge port number**. The trailing six octets are the MAC regardless of
VLAN-prefix length, so match a target on the **last six** octets.

**Classic BRIDGE path (fallback):**

```
snmp_walk(address=<switch IP>, oid=".1.3.6.1.2.1.17.4.3.1.2", total_time="40s")
```

Suffix is just the 6 MAC octets; value is the bridge port.

**Converting a MAC to the decimal suffix:** `74:83:c2:23:e5:75` →
`116.131.194.35.229.117` (each hex octet as decimal). Use `Bash` (`python3`) for
the conversion and for tallying — do not eyeball a 50-row table.

## Step 3 — Map bridge port → ifIndex (never assume identity)

Bridge port numbers are **not** ifIndex values. Walk:

```
snmp_walk(address=<switch IP>, oid=".1.3.6.1.2.1.17.1.4.1.2", total_time="20s")
```

`dot1dBasePortIfIndex`: suffix = bridge port, value = ifIndex. On some switches
this is an identity map (port N → ifIndex N) but **that is vendor-dependent —
always read this table**; many switches offset or re-order, and LAG/CPU ports
use large synthetic indices.

## Step 4 — Read the interface facts

Read the interface's identity (`ifName`/`ifAlias`) live to confirm the port you
located (4a), then hand off to `interface-metrics` for the actual link grade —
throughput, speed, error/discard rates, link state (4b).

### 4a — Identity, speed, status (live `snmp_get`, append `.<ifIndex>`)

| Fact                   | OID base                   |
|------------------------|----------------------------|
| `ifName`               | `.1.3.6.1.2.1.31.1.1.1.1`  |
| `ifAlias` (port label) | `.1.3.6.1.2.1.31.1.1.1.18` |
| `ifHighSpeed` (Mbps)   | `.1.3.6.1.2.1.31.1.1.1.15` |
| `ifOperStatus` (1=up)  | `.1.3.6.1.2.1.2.2.1.8`     |

`ifName` values come back base64-encoded octet strings (e.g. `MS8wLzI1` =
`1/0/25`) — decode with `Bash`. These are fine to read live point-in-time.

### 4b — Traffic / error / discard rates (hand off to `interface-metrics`)

Once you know the attachment port's `if_index`, hand off to
`Skill(interface-metrics)` with the switch `device_name` and that `if_index` to
grade the link: throughput vs speed, error and discard **rates** over a window,
and link state. That skill reads stored metrics (`sprinter_interface_*`) rather
than live SNMP counters — **never** sample a counter twice with a sleep to fake
a rate. It also handles the no-series fallback (a single live `snmp_get`,
reported as a cumulative snapshot).

Carry its verdict into Step 6: a port erroring/discarding, flapping, or
negotiated below its capable speed caps every device on the branch.

## Step 5 — Edge port vs uplink/trunk (the determination that makes this correct)

A MAC learned on a port means the device is **reachable through** that port — not
necessarily plugged into it. A switch's **uplink** learns every MAC on the far
side of the fabric. Disambiguate by **how many distinct MACs that port learned**
(tally from the same FDB walk):

- **1–2 MACs on the port** → edge/access port; the device is (almost certainly)
  directly attached here. This port's interface health IS the device's real
  wired link.
- **Many MACs on the port** → uplink/trunk toward the rest of the network; the
  device is *downstream*, reached through this switch but attached elsewhere.
  Find the switch where the MAC sits on a **low-count** port — that is the true
  attachment point.

**Multi-switch correlation.** With more than one SNMP switch, walk all of them
and cross-reference:

- The switch that sees the **other switches on distinct ports** (not all funneled
  onto one uplink) is the **core/aggregation** switch.
- Each leaf switch sees everything-but-itself on its single uplink port, pointing
  back toward the core.
- A target that appears on a low-count edge port on exactly one switch is
  attached there. A target that appears only on **uplink/downlink ports
  everywhere** is behind an unmanaged or SNMP-silent switch (or an AP) — say so
  honestly rather than claiming a port.

## Step 6 — Present findings

Report, for the target device:

- **Attachment point** — switch name + physical port (`ifName`/`ifAlias`), or
  "behind an unmanaged switch off <core> port <p>" when that is the truth.
- **Link grade** — `ifHighSpeed` + `ifOperStatus` (live, from 4a) and the
  error/discard **rates** from metrics (4b). Flag a gigabit-capable port
  negotiated at 100 Mbps, or a sustained nonzero error/discard rate, as a
  finding in its own right — that caps every device on the branch. If you had to
  fall back to a single live counter snapshot (no metrics series), say the
  number is cumulative-since-boot, not a rate.
- **Path to the gateway** — the chain of uplink ports across switches, if
  multiple switches.
- **Topology corrections** — when this independent FDB view contradicts a
  controller's reported topology, state the corrected topology and that the
  controller's was inference (see the wired-attribution trap, via
  `get_reference_doc`, `name: unifi-controller-api`).

**Honesty about limits.** State what you could NOT determine. FDB alone cannot
tell whether a 100 Mbps port is a damaged/2-pair cable or a 100M-only far-end
device negotiating down — that needs the far device or `dot3StatsDuplexStatus`
(EtherLike-MIB `.1.3.6.1.2.1.10.7.2.1.19.<ifIndex>`). Do not assert the cause;
name the ambiguity and the next probe that would resolve it.

## Guards and failure modes

- **dot1d-vs-dot1q fork** (Step 1): Q-BRIDGE first when present; an empty classic
  table on a VLAN-aware switch is expected, not "device absent."
- **Bridge-port ≠ ifIndex** (Step 3): always read `dot1dBasePortIfIndex`.
- **Stale FDB**: entries age out (~5 min). A device silent for a while may be
  missing from the table though still attached. If an expected MAC is absent,
  `network_ping` it to refresh the switch's learning, then re-walk. Note this in
  the report.
- **L2 scope**: FDB only resolves devices in the same L2 domain as the switch. A
  target on a different VLAN/subnet won't appear; use `network_arp_table` (it
  exposes a `router_snmp` source) to locate the L3 gateway, then the switch
  behind it.
- **SNMP budget**: per switch this is one FDB walk + one port→ifIndex walk + a
  small batch of per-ifIndex GETs. With many switches, walk all FDBs first,
  narrow to the switch owning the MAC on a low-count port, and only THEN pull
  full interface stats — do not pull every port's counters on every switch.
- **Rates come from metrics, not a sleep** (Step 4b): grade the link via the
  `interface-metrics` skill (stored `sprinter_interface_*` series), never by
  sampling a live counter twice with a delay.

## When this skill is reached from another skill

- **From `troubleshoot-device`** (its "find which physical switch port" step):
  return the attachment point and link grade; that skill folds it into its device
  summary.
- **From `diagnose-wifi-roaming`** (grading a mesh AP's real wired uplink, or
  confirming/refuting the controller's wired-uplink attribution): grade the
  switch port the AP's wired MAC attaches to, and report whether the AP is on a
  managed edge port or behind an unmanaged segment — that is the AP's true wired
  backhaul quality, independent of anything the controller inferred. **This
  verdict covers ONLY the wired backhaul** — make that explicit so it is not read
  as a complete uplink verdict. The **wireless** side of the AP (per-client
  signal/SNR/retries, the radio health) is graded separately by the caller via
  `diagnose-wifi-basic` / `diagnose-wifi-roaming` against the WiFi catalog bands
  (via `get_reference_doc`, `name: wifi-metrics-reference`); this skill emits no Wi-Fi metric.
