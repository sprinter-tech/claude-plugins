---
name: troubleshoot-device
description: >
  Troubleshoot, diagnose, or investigate a network device. Use when
  the user asks to troubleshoot a device, diagnose connectivity issues,
  investigate what a device is, or asks "what is this device?" — whether
  they mention "device" explicitly or just provide an IP address, MAC
  address, or hostname.
argument-hint: "[device-address]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill, WebSearch,
  mcp__sprinter__show_network,
  mcp__sprinter__network_tech_stack,
  mcp__sprinter__show_agent,
  mcp__sprinter__list_networks,
  mcp__sprinter__find_network,
  mcp__sprinter__list_devices,
  mcp__sprinter__show_device,
  mcp__sprinter__show_hints,
  mcp__sprinter__find_device,
  mcp__sprinter__device_presence_history,
  mcp__sprinter__network_port_scan,
  mcp__sprinter__network_tcp_banner,
  mcp__sprinter__network_mdns_probe,
  mcp__sprinter__network_ssdp_probe,
  mcp__sprinter__network_windows_info,
  mcp__sprinter__network_http,
  mcp__sprinter__network_ping,
  mcp__sprinter__network_traceroute,
  mcp__sprinter__network_dns_lookup,
  mcp__sprinter__network_jitter_test,
  mcp__sprinter__snmp_get,
  mcp__sprinter__snmp_walk,
  mcp__sprinter__ipp_printer,
  mcp__sprinter__show_device_classes,
  mcp__sprinter__network_arp_table,
  mcp__sprinter__show_probes,
  mcp__sprinter__network_issues,
  mcp__sprinter__issue_chart,
  mcp__sprinter__network_info,
  mcp__sprinter__isp_info,
  mcp__sprinter__ioda,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__ask_user
---

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Sprinter
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

Troubleshoot device $ARGUMENTS

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

## Multi-Step Investigations

When troubleshooting requires several dependent probes — "read the error
counter, correlate with the interface state, check whether the firmware
matches" — issue the individual MCP probe tools (`snmp_get`, `snmp_walk`,
`network_http`, `network_ping`, …) in sequence and correlate their results
in chat.

**Do not attempt to run a Python (or any) code snippet to bundle these
probes.** There is no snippet/code-execution tool available to this plugin,
and the backend does not permit it. Stick to the dedicated read-only MCP
tools.

## Context Management

**Do not shell out to process tool results.** Parse JSON responses inline —
never save MCP tool output to files or pipe it through Python/jq scripts.

**Use `show_device` progressively.** The tool supports three detail levels
via the `sections` parameter:

1. **Default** (omit `sections`): device overview — interfaces, tags, system
   info. HTTP response bodies are replaced with metadata (url, port, code,
   headers, body_length).
2. **`sections=discovery`**: minimal device identity + full discovery data
   (banners, SNMP MIBs, identification log). HTTP bodies shown as 500-char
   previews.
3. **`sections=http_body`**: page through a specific HTTP response body.
   Pass `body_url` (required), `body_offset` (default 0), and `body_limit`
   (default 4000).

## Step 1: Resolve the Network

If the user mentions a network by **name** (e.g. "on network 'codeminders'"),
you must resolve it to a `network_id` before proceeding. **Network names and
network IDs are different things.** A network ID is a fixed-length opaque
string (e.g. `cma5pqbzu000001nmvtfv1m64`); a network name is a human-readable
label (e.g. `codeminders`). Never use a network name where a `network_id` is
required.

**Multi-org: prefer inference, disambiguate by org.** The user may belong to more
than one organization (org); every network belongs to one org, and MCP tools span
all of them. Resolve in this order:

1. If the prompt names the **device** (a name, an IP/MAC, or a description) — call
   **`find_device`** for it, omitting `network_id` to search across every org. A
   single match resolves the `network_id` (and its `tenant` org) directly.
2. Else if the prompt names a **network by name** — call **`find_network`** (name
   prefix, across all orgs). A single match resolves the `network_id`.
3. Else — call **`list_networks`** (spans all orgs; each row carries `tenant_id` +
   `tenant_name`) and match the name against `name`.

If more than one candidate matches — including the same network name in **two
different orgs** — present the candidates **with the org** and ask via `ask_user`
(`org — network — device`; drop the device column when only networks are
ambiguous). When the user belongs to only one org, org labels are unnecessary.

If the user did not mention a network, you can get the `network_id` from the
`show_device` result in the next step (its result also carries the org).

## Step 2: Gather Context

Always start with these two calls:

1. **`show_device`** with the device address (no `sections` parameter) —
   returns device overview with interfaces, tags, and HTTP response metadata.
   This gives you the identification status and `network_id`.
2. **`show_hints`** with the same address — returns investigation hints from
   previous sessions. If hints are returned, follow them before falling back
   to generic investigation.

Use the `network_id` from Step 1 if already resolved, otherwise use the
`network_id` from the `show_device` result. Do not guess or use placeholder
values for `network_id`. If `show_device` does not return the device, use
`find_device` to look it up.

**Check current identification status.** Tell the user what's already known
about the device (vendor, product_name, device_class, description). If any
fields are missing, offer to investigate further.

**Understand what sits around the device (optional).** When the problem plausibly
involves the gateway, an upstream switch/router, or a WiFi platform's coverage,
call `network_tech_stack`. It returns the network's gateway and infrastructure
(each with `device_id` and IP, ready for `show_device` / `network_ping` /
`snmp_get`) plus authored per-platform WiFi notes. Use those notes to avoid
misdiagnosing a **structural platform limit** (e.g. Google Wifi reports no
per-client data) as a device fault.

## Step 3: Investigate

Use the appropriate tools based on what you're troubleshooting:

**Device identification** — figure out what the device is:
- `network_port_scan` → discover open services
- `network_tcp_banner` → identify services on open ports
- `network_http` → probe web interfaces for vendor/model info
- `network_mdns_probe`, `network_ssdp_probe` → discover advertised services
- `network_windows_info` → Windows device identity via SMB/NetBIOS
- `snmp_get`, `snmp_walk` → query SNMP-enabled devices (see below)
- `show_device_classes` → see valid class names when classifying a device

**Device status and operational queries** — answer questions about the
device’s current state (storage, temperatures, firmware version, etc.):
- `snmp_get`, `snmp_walk` → many devices expose operational status via
  SNMP (UPS battery, sensor readings, interface counters)
- `network_http` → **use this to query the device’s web interface
  directly.** Most managed devices (NAS, routers, APs, cameras) have
  embedded web servers with status pages that show more detail than SNMP.
  Fetch pages like `/`, `/status`, `/api/status` and parse the HTML or
  JSON response for the information the user asked about. **Do not tell
  the user to open a browser** — you can fetch and read these pages
  yourself with `network_http`.
- For **printers**: if port 631 is in the device's `open_ports` (from
  `show_device`), try `ipp_printer` with `path="/ipp/print"` first (if
  that fails, try `/ipp/port1`, `/ipp/printer`, `/ipp`). For deeper
  diagnostics (paper trays, vendor OIDs, alerts), use the `troubleshoot-printer`
  skill which has specialized SNMP OIDs and web interface paths.

**Wireless client / AP / controller** — if the device is on Wi-Fi, do NOT
diagnose the link here. Check `evidence.wifiSnapshots` in the `show_device`
result (an `association` block, or a `controllerSummary` for an AP/controller):
if present, the device is wireless. Hand off to the WiFi specialists, which read
live link health from VictoriaMetrics (~1-min fresh) and grade it against the
catalog health bands — capabilities this skill does not have:
- **`Skill(diagnose-wifi-basic)`** — the general "is this wireless link OK, and
  if not why" check (weak signal, high retries, congestion, low rate). Use this
  for a slow/choppy/laggy wireless client.
- **`Skill(diagnose-wifi-roaming)`** — when the complaint is "stuck on a far
  AP" / won't roam / mesh-involved.

If the question broadens to **many/all clients** ("do other devices have the same
weak signal?", "rank every wireless client"), still hand off to
`Skill(diagnose-wifi-basic)` — it answers fleet questions with a **single**
network-scoped VM query (`sort(sprinter_wifi_client_<metric>{network_id="<net>"})`,
any WiFi metric, server-sorted) that even joins in device names via
`* on(device_id) group_left(device_name) sprinter_deviceInfo` — NOT a per-device
`show_device` loop over `wifiSnapshots`. The snapshot signal lags (it can be a
stale `mdns_probe` value); looping `show_device` to scrape it is the slow, wrong
path.

Pass the device and the **platform key** — strip the `wifi_` prefix off
`evidence.wifiSnapshots[].source` (`wifi_ubiquiti` → `ubiquiti`,
`wifi_att-bgw` → `att-bgw`) so the WiFi skill adapts per platform. The two
platforms emit **different** per-client metric sets (UniFi is rich:
per-client SNR + retries; the BGW320 is sparse: per-client deauth/disassoc +
per-radio error counters), so a field absent on one platform is expected, not a
fault — the WiFi skill handles that. WiFi counters (error/discard/byte/retry/
deauth/disassoc) are graded as **per-second rates** via `rate(...[window])`,
the same rate-over-window discipline used for `sprinter_interface_*` below; the
WiFi skill applies it.

**Connectivity diagnostics** — figure out why something isn't working:
- `network_ping` → check reachability and latency
- `network_traceroute` → trace the network path
- `network_dns_lookup` → verify DNS resolution
- `network_jitter_test` → measure latency and jitter

**Presence/roam timeline — "keeps going offline", "drops at night", "was it
up at 3pm?"** When the complaint is about *intermittency over time* rather than
the device's state *right now* — it disconnects and reconnects, goes offline on
a schedule, "worked this morning then stopped", or (Wi-Fi) keeps bouncing
between access points — call **`device_presence_history`** with the resolved
`device_id`. `network_ping` only tells you up-or-down *now*; this tool returns
the device's **state-transition timeline** so you can see the pattern. It reads
from the device-state reducer's history stream (presence transitions) plus the
Wi-Fi event stream (roam / disassoc), so a single call answers "when did it go
offline, how often, and was it a Wi-Fi roam or a true drop?":

- **Input:** `device_id` (required — resolve a name/IP/MAC first via
  `find_device` / `show_device`). Optional `start` / `stop` RFC3339 timestamps
  bound the window; omit both for the **last 24h** (omitting `stop` alone means
  "now"). For a "what happened last night" question, pass an explicit window.
- **Output (plain text):** a header line with the device's **state at the end
  of the interval** (`online` / `sleep` / `offline` / `unknown`), then one line
  per event oldest-first:
  - `<ts>  <from> -> <to>  (<reason>)` — a presence transition. States are
    `online` (answering ping), `sleep` (Wi-Fi-associated but not answering ping
    — a normal low-power state, NOT a fault), and `offline`. Reasons:
    `ping_reply`, `ping_miss`, `wifi_assoc`, `presence_ttl_expiry`. A first-seen
    transition shows `(none) -> ...`.
  - `<ts>  roamed AP <from_ap> -> <to_ap>` — the client moved between access
    points (Wi-Fi).
  - `<ts>  disassociated from AP <ap>` — the client left an AP (Wi-Fi).
- **How to read it.** A device cycling `online -> sleep -> online` is usually a
  **healthy power-saving client** (phone, tablet, IoT), not a problem — `sleep`
  means "still associated, just dozing". Frequent `online <-> offline` flapping,
  or `offline` stretches during the complaint window, is the real signal. Many
  `roamed`/`disassociated` events clustered in time point at a Wi-Fi
  roaming/sticky-client problem → hand off to `Skill(diagnose-wifi-roaming)`.
  A device that simply does not answer ping but shows steady `sleep` (never
  `offline`) is present and dozing, not down.

**Network context:**
- `network_arp_table` → see devices on the local segment
- `show_network` → network configuration, ISP info, and agents (with IDs)
- `list_devices` → all known devices on the network
- `show_probes` → list active probes and their targets for a network/agent
- `network_issues` → check for recent performance issues (outlier clusters,
  mean/variance shifts) that may correlate with the device problem

**HTTP probing tips.** Many devices have embedded web servers with status
pages, configuration panels, and API endpoints. Use `network_http`
to fetch these pages — you can read and parse the HTML/JSON response
directly. **Never suggest the user open a browser when you can fetch the
page yourself.** Start with `/`, then try vendor-specific paths. Limit
endpoint-guessing to 10 total attempts.

**SNMP probing.** If the device has a `working_channel` field in the
`show_device` output, SNMP is reachable. Use `snmp_get` and `snmp_walk`
with the `working_channel` value as the `channel` parameter.

*Identification OIDs* — start here to determine what the device is:
- `1.3.6.1.2.1.1` — system group (sysDescr, sysObjectID, sysName,
  sysLocation). Walk this first for vendor/model/OS.
- `1.3.6.1.2.1.47.1.1.1` — entPhysicalTable (hardware inventory,
  serial numbers, firmware versions on managed devices).

*Interface health* — for traffic, speed, errors, discards, and link state on
an SNMP-monitored device, **hand off to `Skill(interface-metrics)`** with the
device and (if known) the `if_index`. It reads the stored `sprinter_interface_*`
time series, so you get a real **rate over a window** instead of a
cumulative-since-boot SNMP counter you can't interpret from one read — never
sample a counter twice with a sleep to fake a rate. The OIDs below are for
reference / the no-metrics fallback (which `interface-metrics` handles):
- `1.3.6.1.2.1.2.2` — ifTable: interface status (ifOperStatus,
  ifAdminStatus), speed (ifSpeed), MTU, and counters (ifInOctets,
  ifOutOctets, ifInErrors, ifOutErrors, ifInDiscards, ifOutDiscards).
  Non-zero error or discard counters indicate physical layer problems
  (bad cables, failing optics) or congestion.
- `1.3.6.1.2.1.31.1.1` — ifXTable: 64-bit counters (ifHCInOctets,
  ifHCOutOctets) for high-speed interfaces, plus ifAlias (port
  description configured by the admin).
- `1.3.6.1.2.1.10.7.2` — dot3StatsTable (EtherLike-MIB): duplex
  status (dot3StatsDuplexStatus), late collisions, FCS errors. Duplex
  mismatch is a common cause of poor throughput — look for half-duplex
  on links that should be full-duplex, or high late collision counts.

*Neighbor discovery OIDs* — see what's connected to each port:
- `1.0.8802.1.1.2.1.4` — LLDP remote table (lldpRemTable): neighbor
  chassis ID, port ID, system name, and capabilities.
- `1.3.6.1.4.1.9.9.23.1.2` — CDP cache table (Cisco devices):
  neighbor device ID, platform, and IP address.

*Device health OIDs* — check resource utilization:
- `1.3.6.1.4.1.2021.11` — UCD-SNMP CPU stats (ssCpuUser, ssCpuSystem,
  ssCpuIdle) on Linux-based devices.
- `1.3.6.1.4.1.2021.4` — UCD-SNMP memory stats (memTotalReal,
  memAvailReal, memTotalSwap, memAvailSwap).
- `1.3.6.1.2.1.4.21` — ipRouteTable: routing table entries, useful
  for verifying next-hop reachability.
- `1.3.6.1.2.1.4.22` — ipNetToMediaTable: ARP table via SNMP.

*PoE OIDs* — for switches powering APs, cameras, or phones:
- `1.3.6.1.2.1.105.1.1` — pethMainPseTable: total PoE power budget
  and consumption per PSE group.
- `1.3.6.1.2.1.105.1.2` — not available on all devices; for per-port
  PoE status use `1.3.6.1.2.1.105.1.1` and vendor-specific MIBs.

## Step 4: Gather Additional Context

If the network has SNMP-capable switches, locate which physical switch port the
device attaches to and grade that wired link — hand off to
`Skill(locate-device-uplink)` with the device's MAC/IP. That skill reads the
switches' MAC forwarding tables over SNMP to pinpoint the attachment point
(switch + port), report link speed and error/discard counters, and distinguish a
true edge port from an uplink the device merely transits. Fold its result into
your findings; a gigabit port negotiated at 100 Mbps or climbing interface
errors on the device's branch is a finding worth surfacing.

## Step 5: When did this start, and what happened around then?

Once you have a diagnosis (or a strong symptom — the device is dropping, the
link went bad, latency spiked), do not stop at *what* is wrong. Ask **when it
started** and **what else happened at that moment**, because devices affect each
other and the subject is often the victim of a change on a device it depends on.

This is a read-only correlation step on tools you already have — no new data
source. The full recipe is in the `when-did-this-start` reference (fetch via the
`get_reference_doc` MCP tool, `name: when-did-this-start`); the short form:

1. **Pin the onset `T`.** `device_presence_history` on the subject `device_id`
   over a generous window — the first `online -> offline` flap or the first
   clustered `roamed`/`disassociated` is the onset (a steady `online -> sleep ->
   online` cycle is healthy power-save, NOT an onset). A flagged
   `network_issues` `startTime` can also be the onset.
2. **Build the dependency set** — the infra this device depends on, NOT every
   device on the network: the **gateway** (`network_info`), the **DHCP server**
   (the gateway, or a DHCP event's `dhcp_server_ip`), the **serving AP**
   (`evidence.wifiSnapshots[].association.anchorDeviceId`) for a Wi-Fi client,
   the **wired switch** (`find_device(device_class="network_switch")`), and the
   upstream **ISP** (`isp_info` / `ioda`).
3. **Sweep a tight window around `T`** (typically `T ± 15 min`). Call
   `network_issues` over `[T − w, T + w]` and read events by `probeType`:
   `pt_dhcp` (config change / rogue server — a DHCP change on the gateway just
   before the device dropped is a prime cause), `pt_traceroute` (path change
   upstream), `pt_ping`/`pt_http`/`pt_dns` on the gateway or a shared target
   (infra-wide, not device-local). Also run `device_presence_history` on the
   **serving AP / gateway** themselves — did one transition `offline` at `T`?
   (DHCP and traceroute have **no** dedicated history tool — their events surface
   through `network_issues`; do not look for `dhcp_history`/`traceroute_history`.)
4. **Report a timeline and read it honestly.** A co-occurring event on a
   dependency is a *candidate* cause, not proof — state it as a correlation. A
   **clean** sweep is also a finding: "nothing changed on the gateway, the
   serving AP, or the path in ±15 min around onset → this is device-local."
   Don't mistake a collection gap at `T` for an onset.

## Step 6: Present Findings

Summarize what you found:
- Device identity (vendor, model, device class) if determined
- Connectivity status and any issues found
- Network path and latency characteristics if relevant
- **Onset + correlation timeline** — when the problem started and what happened
  on the device's infra dependencies around that time (or that the window was
  clean, ruling out infra)
- Recommendations for next steps or remediation
