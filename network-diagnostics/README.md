# Network Diagnostics Plugin

A Claude Code plugin for network device identification, diagnostics, and
troubleshooting using [Sprinter](https://github.com/sprinter-tech) MCP tools.

It bundles an MCP server connection to the Sprinter platform plus a set of
skills that turn natural-language requests ("is this Wi-Fi slow?", "what is
device 10.0.14.1?", "what went wrong on this network last night?") into guided,
read-only investigations.

## Installation

This plugin is distributed as a Claude plugin **marketplace** repo, so the same
source installs into both Claude Code and Claude Desktop and auto-updates when a
new version is published.

### Claude Code

```
/plugin marketplace add sprinter-tech/claude-plugins
/plugin install network-diagnostics@sprinter
```

The first command registers the Sprinter plugin marketplace; the second installs
this plugin from it. Installed plugins refresh from the marketplace at startup,
so you get new skills without reinstalling.

### Claude Desktop

1. Open **Settings → Extensions** (the plugins/extensions panel).
2. Choose **Add marketplace** (or "Add from a repository") and enter the repo:
   ```
   sprinter-tech/claude-plugins
   ```
3. From that marketplace, **install** the `network-diagnostics` plugin.
4. Restart Claude Desktop if prompted. The plugin's skills and its Sprinter MCP
   connection (`.mcp.json`) load automatically.

> Reference content (Wi-Fi metric bands, controller-API details, etc.) is **not**
> shipped as files — Desktop has no filesystem the model can read. The skills
> fetch it at runtime through the Sprinter MCP tool `get_reference_doc`, so the
> MCP connection (below) must be reachable for the Wi-Fi/diagnostic skills to pull
> their reference material.

### Local uploads fallback (a `.zip`)

If you cannot add the marketplace repo, install a one-off archive built by the
monorepo (`bazel build //plugins/network-diagnostics:zip` →
`network-diagnostics.zip`) via the **Local uploads** option in the same
Extensions panel. This path does **not** auto-update — re-upload to upgrade.

## Permissions

Approve all Sprinter MCP tools to avoid permission prompts during
investigations:

```bash
claude settings add-permission "mcp__sprinter__*" --scope user
```

## MCP server

The plugin connects to the Sprinter MCP server over HTTP — see `.mcp.json`. The
default endpoint is the production cluster (`prod-us-west-1.sprinter-tech.net`).

## Skills

All skills are **read-only** — they observe and report, they never change device
or controller configuration.

| Skill                      | What it does                                                                                                                                                                                                                                          |
|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `triage-network-complaint` | Localize a vague complaint ("internet is slow", "calls keep dropping") to a layer — ISP/WAN, gateway, DNS, Wi-Fi, or a single device — then dispatch to a specialist skill. **Start here** when the cause isn't already named.                        |
| `troubleshoot-device`      | Troubleshoot/identify a device by IP, MAC, or hostname. "What is this device?" / "diagnose connectivity for 10.0.14.1".                                                                                                                               |
| `locate-device-uplink`     | Find what a device attaches to (switch + port, or AP) and grade that link (speed, errors/discards), from Sprinter's persisted topology graph. Live SNMP only when the graph is unsure. "Which switch port is 10.0.14.50 on?"                          |
| `interface-metrics`        | Read per-interface traffic, speed, errors, discards, link state (and device CPU/memory/uptime) for an SNMP device from stored time-series metrics — real rates over a window, not a live counter read. "Is this uplink clean / saturated / flapping?" |
| `diagnose-wifi-basic`      | Basic Wi-Fi link health check for one client on any supported WiFi platform (signal, retries, noise, rate, jitter). Hands off to roaming when the cause is a far-AP problem.                                                                              |
| `diagnose-wifi-roaming`    | Sticky-client / roaming / mesh analysis. The link-quality verdict runs on any platform; the deeper roaming/min-RSSI analysis is UniFi-controller-specific today.                                                                                          |
| `network-assessment`       | Assess connection quality — speed, latency, jitter, and suitability for VoIP / gaming / streaming / video calls.                                                                                                                                      |
| `network-issues-report`    | Report significant network issues over a time range (loss, latency spikes, mean/variance shifts, DHCP changes, rogue DHCP servers).                                                                                                                   |
| `troubleshoot-printer`     | Check a network printer's status — toner/ink, jams, errors, supply levels, page counts.                                                                                                                                                               |
| `printer-report`           | Generate a full HTML report on a printer (identity, capabilities, supplies, counters, status).                                                                                                                                                        |
| `select-network`           | Select or switch the active network for MCP tool calls when a `network_id` is needed.                                                                                                                                                                 |

### Examples

Just ask naturally:

> "Why is video on the TV in the den so choppy?"
> "What is device 10.0.14.18?"
> "How is this network performing — good enough for video calls?"
> "What went wrong on this network between 2pm and 4pm?"
> "Is the printer at 10.0.14.50 out of toner?"

Or invoke a skill directly:

```
/troubleshoot-device 10.0.14.1
/triage-network-complaint my-home-net
```

## Notes on Wi-Fi skills

Sprinter's WiFi monitoring is **multi-platform** — UniFi, AT&T BGW320, OpenWrt,
Luxul, Arris, and more, with new platforms added over time. The Wi-Fi skills are
**vendor-neutral**: they read link facts from `show_device`
`evidence.wifiSnapshots` (a vendor-neutral shape) and grade live metrics from
VictoriaMetrics, adapting to whichever platform a device reports. The
authoritative, always-current list of **supported platforms** and the **metric
each emits** is the `wifi-metrics-reference` doc (its "Supported platforms"
roster), fetched at runtime via the `get_reference_doc` MCP tool.

One part is genuinely platform-specific: `diagnose-wifi-roaming`'s deeper
**roaming / mesh / min-RSSI** analysis needs a **UniFi controller** (UDM /
UDM-Pro / Cloud Key) and degrades gracefully on other platforms — the
platform-neutral link-quality verdict still runs everywhere. Vendor-neutral
*interpretation* guidance (the "Experience != link quality" trap, the min-RSSI
failure modes) is likewise fetched on demand via `get_reference_doc`, so the MCP
connection must be reachable for the skills to pull their reference material.

## Building & publishing

This plugin's source lives in the Sprinter monorepo under
`plugins/network-diagnostics/`. Maintainers build and publish from there:

```bash
bazel build //plugins/network-diagnostics:zip   # build network-diagnostics.zip (Local-uploads)
make publish-plugin                              # push the payload to the public marketplace repo
```

`make publish-plugin` stamps the monorepo version into `plugin.json` /
`marketplace.json` and pushes the skills + manifests to
`sprinter-tech/network-diagnostics-plugin-claude`. Reference docs are served by
the Sprinter MCP server (not shipped in the payload).
