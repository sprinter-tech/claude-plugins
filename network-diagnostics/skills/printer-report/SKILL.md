---
name: printer-report
description: >
  Generate a comprehensive HTML report about a network printer or
  multifunction device. Use when the user asks for a printer report,
  printer inventory, printer details, or wants a full overview of a
  printer's identity, capabilities, supplies, counters, and status —
  whether they mention "printer" explicitly or provide an IP address
  of a known printer.
argument-hint: "[printer-address]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill, WebSearch, Write,
  mcp__sprinter__show_network,
  mcp__sprinter__show_agent,
  mcp__sprinter__list_networks,
  mcp__sprinter__list_devices,
  mcp__sprinter__show_device,
  mcp__sprinter__show_hints,
  mcp__sprinter__find_device,
  mcp__sprinter__network_port_scan,
  mcp__sprinter__network_tcp_banner,
  mcp__sprinter__network_http,
  mcp__sprinter__network_ping,
  mcp__sprinter__network_dns_lookup,
  mcp__sprinter__snmp_get,
  mcp__sprinter__snmp_walk,
  mcp__sprinter__ipp_printer,
  mcp__sprinter__show_device_classes,
  mcp__sprinter__network_arp_table,
  mcp__sprinter__show_probes,
  mcp__sprinter__network_issues,
  mcp__sprinter__ask_user
---

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play. Call tools without describing the act
> of calling them. Surface only what matters to the user: the findings, the
> supporting evidence, and the verdict/next step. Keep any interim text minimal.

Generate a printer report for $ARGUMENTS

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

## Goal

Produce a **self-contained HTML file** with a polished, user-friendly report
covering everything knowable about the printer: identity, capabilities,
current status, consumable levels, page counters, paper trays, connectivity,
advertised services, error history, and actionable recommendations.

The report is saved to disk; tell the user the file path when done.

## Step 1: Resolve the Network

If the user mentions a network by **name** (e.g. "on network 'codeminders'"),
resolve it to a `network_id` before proceeding. **Network names and network
IDs are different things.** A network ID is a fixed-length opaque string
(e.g. `cma5pqbzu000001nmvtfv1m64`); a network name is a human-readable
label (e.g. `codeminders`). Never use a network name where a `network_id`
is required.

To resolve: call **`list_networks`**, match the user's name against the
`name` field (case-insensitive), and use the corresponding `network_id`.
If multiple networks match, use `ask_user` to let the user choose. If no
match is found, present the full list via `ask_user`.

If the user did not mention a network, you can get the `network_id` from
the `show_device` result in the next step.

## Step 2: Gather Context

Always start with these two calls:

1. **`show_device`** with the printer address (no `sections` parameter) —
   returns device overview with interfaces, tags, and HTTP response metadata.
   This gives you the identification status, `network_id`, and
   `working_channel` (SNMP community string).
2. **`show_hints`** with the same address — returns investigation hints from
   previous sessions. If hints are returned, follow them before falling back
   to generic investigation.

Use the `network_id` from Step 1 if already resolved, otherwise use the
`network_id` from the `show_device` result. Do not guess or use placeholder
values for `network_id`. If `show_device` does not return the device, use
`find_device` to look it up.

**Check current identification status.** Confirm the device is actually a
printer before proceeding with printer-specific data collection.

### Multiple Printers on the Network

If the user did not specify a printer address and there are multiple printers
on the network, use `ask_user` to ask which one to report on. **Do not
present raw hostnames** like `BRN001BA9DA156B` — these are vendor-assigned
internal names and are meaningless to the user. Instead, identify each
printer first (using `show_device` for each) and present choices using:

1. **Consumer model name** (e.g. "Brother MFC-J825DW", "HP LaserJet Pro M404n")
2. **Short type description** (e.g. "inkjet printer/scanner", "laser printer",
   "color laser multifunction")
3. **IP address** for disambiguation

Example `ask_user` prompt:

> Which printer would you like to report on?
> 1. **Brother MFC-J825DW** — inkjet printer/scanner (10.0.14.52)
> 2. **Brother HL-L2350DW** — laser printer (10.0.14.53)

## Step 3: Data Collection — Prioritized Sources

**CRITICAL RULE: Do NOT use SNMP if IPP succeeds.** IPP provides higher
quality, more granular data than SNMP for the same data points. If the
`ipp_printer` tool returns a successful response with supply levels and
printer status, **skip SNMP entirely** — do not walk any SNMP OIDs for
"confirmation" or "additional detail". The only exception is input tray
paper levels, which IPP does not provide.

Before calling any SNMP tool, ask yourself: "Did IPP already give me this
data?" If yes, do not call SNMP. If you are about to walk multiple SNMP
OID subtrees after a successful IPP call, **stop — you are doing it wrong.**

The data collection order below is strict. Complete each priority level,
then evaluate what data is still missing before proceeding to the next.

### Priority 1: IPP (Internet Printing Protocol)

Check `open_ports` in the `show_device` response. **If port 631 is listed**,
try the `ipp_printer` tool. The `path` parameter is required — try these
paths in order until one succeeds:

1. `/ipp/print` — most modern printers (IPP Everywhere standard)
2. `/ipp/port1` — Brother printers (PCL queue)
3. `/ipp/printer` — some older HP and Lexmark models
4. `/ipp` — some Canon and Kyocera models

**Always request the extended attribute set** — the default set misses
several useful attributes. Pass the `attributes` parameter explicitly:

```
ipp_printer(
  address="10.0.14.52",
  path="/ipp/print",
  network_id="...",
  attributes=[
    "printer-make-and-model", "printer-name", "printer-serial-number",
    "printer-firmware-string-version", "printer-uuid", "printer-dns-sd-name",
    "printer-device-id", "printer-info",
    "printer-state", "printer-state-reasons", "printer-state-message",
    "printer-is-accepting-jobs", "printer-up-time",
    "marker-names", "marker-levels", "marker-high-levels",
    "marker-low-levels", "marker-colors", "marker-types",
    "printer-supply", "printer-supply-description",
    "printer-impressions-completed", "pages-per-minute", "pages-per-minute-color",
    "queued-job-count",
    "color-supported", "sides-supported", "media-ready",
    "document-format-supported", "printer-resolution-supported",
    "printer-input-tray", "printer-output-tray",
    "printer-alert", "printer-alert-description",
    "printer-current-time", "printer-location", "printer-more-info",
    "printer-geo-location"
  ]
)
```

The extended attributes add:
- `printer-supply` / `printer-supply-description` — newer structured
  supply format that bundles all supply info (alternative to `marker-*`)
- `document-format-supported` — accepted PDL formats (PDF, PCL, PS, etc.)
- `printer-resolution-supported` — available print resolutions
- `printer-input-tray` / `printer-output-tray` — tray status (when
  supported; not all printers populate these via IPP)
- `printer-alert` / `printer-alert-description` — active alerts
- `printer-current-time` — printer's clock
- `printer-info` — admin-set description
- `printer-geo-location` — geographic coordinates

If one path returns an error, try the next. Stop as soon as one succeeds.
If all fail, move to Priority 2. **Do not try paths beyond these four
without first searching the web for the specific printer model** — some
paths could cause the printer to print a test page.

**If the printer returns status 0x0503 (version-not-supported)**, retry
with `ipp_version="1.0"` (same extended attribute list):

```
ipp_printer(address="10.0.14.52", path="/ipp/port1", ipp_version="1.0",
            network_id="...", attributes=[...same list...])
```

IPP provides:
- **Identity**: make/model, serial number, firmware version, UUID,
  device ID, DNS-SD name
- **Status**: printer state (idle/processing/stopped), active warnings
  and errors, alerts with descriptions, current time
- **Consumables**: per-supply name, type (toner/ink/drum), color, and
  level as a percentage (0–100). Special values: -1 = unknown, -3 = full.
  Both `marker-*` arrays and newer `printer-supply` strings.
- **Page counters**: total impressions completed, pages per minute
- **Capabilities**: document formats, color support, duplex modes,
  print resolutions, media loaded
- **Trays**: input/output tray status (when supported)

**After a successful IPP response**, you already have identity, status,
supply levels, page counters, and capabilities. **Proceed directly to
Priority 2 (web interface) for counter breakdowns and history.** Do NOT
proceed to SNMP.

### Priority 2: Web Interface (EWS)

The web interface provides data that IPP and SNMP typically lack:
detailed page counter breakdowns (by function, paper size, media type),
scan counters, paper jam history, error logs, and replacement counts.

**Always try the web interface** even if IPP succeeded — it is the best
source for historical and per-function data.

**Discovery strategy:**

1. Start with root page `/` — parse navigation links.
2. Try common vendor-neutral paths: `/status`, `/supplies`, `/info`.
3. After identifying the vendor, try vendor-specific paths:

   **Brother**: `/general/status.html`, `/general/information.html`,
   `/general/consumables.html`, `/etc/mnt_info.csv` (detailed counters)

   **HP**: `/DevMgmt/ProductStatusDyn.xml`, `/DevMgmt/ConsumableConfigDyn.xml`,
   `/DevMgmt/ProductUsageDyn.xml`, `/DevMgmt/MediaHandlingDyn.xml`,
   `/hp/device/InternalPages/Index?id=SuppliesStatus`

   **Epson**: `/PRESENTATION/HTML/TOP/PRTINFO.HTML`,
   `/PRESENTATION/HTML/TOP/INDEX.HTML`

   **Xerox**: `/status/consumables`, `/status/deviceStatus`

   **Canon**: `/English/pages/main.html`, `/portal.html`

   **Ricoh**: `/web/guest/en/websys/status/getSystemStatus.cgi`

   **Lexmark**: `/cgi-bin/dynamic/printer/PrinterStatus.html`

   **Samsung**: `/sws/app/information/supplies/suppliesView.sws`

   **Konica Minolta**: `/wcd/system_device.xml`, `/wcd/supply.xml`

4. Use `show_device` with `sections=http_body`, `body_url`, `body_offset`,
   `body_limit` to page through large HTTP responses.

5. **Limit probing to 15 URL paths total.** Stop as soon as you have good data.

### Decision Gate: Do You Need SNMP?

After completing Priority 1 (IPP) and Priority 2 (web interface), evaluate
what you have. **In most cases you are done collecting data.** Proceed to
Step 4 (Connectivity Check).

**Use SNMP only if ALL of the following are true:**
1. IPP was completely unavailable (port 631 closed or every path failed), AND
2. The web interface gave you no supply levels or status data

If IPP succeeded, **do not use SNMP at all** — not for supplies, not for
status, not for counters, not for "confirmation". The only narrow exception:
if you need input tray paper levels and neither IPP's `printer-input-tray`
attribute nor the web interface provided them, you may walk the single OID
`1.3.6.1.2.1.43.8.2.1` (prtInputTable) for tray data only.

If you do need SNMP as the primary data source (because IPP failed), see
the **SNMP Reference** appendix at the end of this document for OIDs.

## Step 4: Connectivity Check

Collect network connectivity data for the report:

1. **`network_ping`** — reachability, latency, packet loss.
2. **`network_port_scan`** — which printing ports are open:
   - 9100 (RAW/JetDirect), 631 (IPP), 515 (LPR/LPD), 80/443 (HTTP/S),
     161 (SNMP), 23 (Telnet)
3. **`network_dns_lookup`** — if addressed by hostname, verify DNS.

Note open ports, closed printing ports, and any connectivity concerns
for the report.

## Step 5: Generate HTML Reports

Generate **two** self-contained HTML files: a **main report** focused on
current state, and a **detail report** with historical data and deep-dive
information. The main report links to the detail report.

Both reports should be visually polished, use CSS-only styling (no external
dependencies), and be easy to read on both desktop and mobile.

### File Naming

- Main report: `printer-report-{HOSTNAME_OR_IP}.html`
- Detail report: `printer-report-{HOSTNAME_OR_IP}-detail.html`

Use the printer's hostname from `show_device` if available (e.g.
`printer-report-BRN001BA97EF38D.html`), otherwise use the IP address
with dots replaced by dashes (e.g. `printer-report-10-0-14-52.html`).

Save both files in the current working directory.

### Main Report Structure

The main report answers: **"What is this printer and what is its current
state?"** It should feel clean and scannable — a user should be able to
glance at it and know whether the printer needs attention.

Use this exact section order. **Only include sections where you have data.**

#### Header
- Title: "Printer Inspection Report"
- Subtitle line: generation date, network name, data sources used
  (list only the sources that actually provided data, e.g.
  "IPP 2.0, HTTP (EWS)" — do NOT mention SNMP if it was not used)

#### 1. Summary & Recommendations
A yellow highlighted box at the top with:
- One-sentence overall status (online/offline, accepting jobs, any errors)
- Bulleted recommendations, each with severity color coding:
  - **Red**: critical issues (supply exhausted, printer stopped, errors)
  - **Orange**: warnings (supply low, unusual jam rate, closed ports)
  - **Default**: informational (supply nearing end, maintenance notes)
- Include specific, actionable advice (e.g. cartridge model numbers to
  buy, ports to enable, cleaning suggestions)
- Reference specific data from the report (e.g. "Drum at 15% — approximately
  1,800 pages remaining")
- At the end, a link to the detail report: "See [detailed historical
  report](printer-report-XXX-detail.html) for page counters, error
  history, and maintenance data."

#### 2. Device Identity
Table with: model, type (with technology badge like "LED", "Laser",
"Inkjet"), serial number, firmware version, memory, print resolution,
color capability, duplex capability, IP address, MAC address, hostname,
uptime.

#### 3. Current Status (side-by-side grid with Network Connectivity)
- **Status card**: printer state, state reasons, accepting jobs, queued
  jobs, **current active errors only** (not historical counters), alerts
- **Connectivity card**: reachability, latency stats, connection type,
  open ports table (with badges), note any closed printing ports with
  warning icon

#### 4. Supply Levels
For each consumable (toner, ink, drum, waste toner, fuser, etc.):
- Name and color
- Visual progress bar with percentage label
- Color-coded bar: green (>50%), yellow (20-50%), orange (10-20%),
  red (<10%)
- Footnote with additional detail (replacement count, data source notes,
  known vendor quirks like Brother SNMP -3 reporting)

#### 5. Paper Trays (side-by-side grid: input and output)
- Tray name, capacity, current level, media size, media type
- Flag empty or near-empty trays

#### 6. Capabilities & Services
- Supported document formats (PDL), print resolutions
- mDNS/SSDP services with ports
- Capabilities from mDNS TXT records (color, copies, duplex, etc.)
- Supported paper sizes and media types
- Security notes (e.g. "Telnet open — consider disabling")

#### Footer
- Link to detail report
- Generation attribution line with data sources and IP address

### Detail Report Structure

The detail report answers: **"What has this printer been doing over its
lifetime?"** It contains historical counters, error logs, and maintenance
data. This is for IT staff doing capacity planning or investigating
recurring issues.

#### Header
- Title: "Printer Detail Report — {Model Name}"
- Subtitle: same as main report, plus "Companion to [main report](link)"

#### 1. Page Counters
- Total lifetime pages (large, prominent number)
- Breakdown by function: print, copy, fax, scan (with percentages)
- Breakdown by paper size
- Breakdown by media type
- Scan counters (ADF vs flatbed) if available
- Pages per month estimate (total pages / uptime or age)

#### 2. Paper Jam History
- Total jam count and jam rate (jams per N pages)
- Breakdown by location (inside, rear, tray, duplex, ADF)
- Analysis of patterns (e.g. "94% occur inside — suggests paper path
  friction")

#### 3. Error History
- Last 10–20 errors with page numbers at which they occurred
- Error type distribution if available

#### 4. Maintenance & Replacement History
- Toner/cartridge replacement count and average yield per cartridge
- Drum replacement count and current drum page count
- Comparison to rated yield (e.g. "averaging 1,585 pages per cartridge
  vs rated 2,600 — consider higher-yield cartridges")

#### 5. Detailed Supply Information
- Full supply table with all fields: description, type, unit, max
  capacity, current level, low threshold, color hex code
- Both `marker-*` and `printer-supply` data if both are available

#### Footer
- Link back to main report
- Generation attribution line

### CSS Design Requirements

Use these design tokens and patterns:

```css
:root {
  --bg: #f8f9fa; --card: #ffffff; --border: #dee2e6;
  --text: #212529; --muted: #6c757d; --accent: #0d6efd;
  --green: #198754; --yellow: #ffc107; --red: #dc3545; --orange: #fd7e14;
}
```

- Clean card-based layout with `max-width: 960px`
- 2-column grid for side-by-side sections (`grid-template-columns: 1fr 1fr`)
- Responsive: stack to single column below 700px
- Supply level bars: colored `div` inside a gray background `div`
- Status dots: 10px colored circles before status text
- Badges: inline rounded labels for categories and states
- No external fonts, images, or scripts — fully self-contained
- Use system font stack: `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`

### Data Source Attribution

In the report, **only mention the data sources you actually used**. If IPP
provided all supply levels and SNMP was not needed, do not mention SNMP in
the report at all. Be precise: say "IPP 2.0" or "IPP 1.0", not just "IPP".
Say "HTTP (EWS)" for web interface data. Say "SNMPv2c" if SNMP was used.

### Handling Missing or Conflicting Data

- If different sources report conflicting values (e.g. SNMP says toner
  is exhausted but web interface shows 90%), **note the discrepancy** in
  the report and indicate which source is more reliable. Vendor-specific
  SNMP quirks are common (especially Brother reporting -3 for supplies
  that are actually not exhausted).
- If a data point is unavailable, omit it from the report rather than
  showing "N/A" or "Unknown" everywhere. Only show "Unknown" when the
  data point is important and its absence is notable (e.g. tray level).
- Prefer the most specific/granular data source for each data point.

---

## Appendix: SNMP Reference (Only When IPP Is Unavailable)

**Do not read this section unless IPP failed.** This appendix exists only
for printers where port 631 is closed or all IPP paths returned errors.
If IPP succeeded, this entire appendix is irrelevant — go back to Step 4.

If the device has a `working_channel` field in the `show_device` output,
SNMP is reachable. Use `snmp_get` and `snmp_walk` with the
`working_channel` value as the `channel` parameter.

### Core Printer MIB OIDs

**Device identity:**
- `1.3.6.1.2.1.1` — sysDescr, sysObjectID, sysName

**Printer status and alerts:**
- `1.3.6.1.2.1.25.3.5.1` — hrPrinterStatus, hrPrinterDetectedErrorState
- `1.3.6.1.2.1.43.18.1.1` — prtAlertTable

**Supply levels:**
- `1.3.6.1.2.1.43.11.1.1` — prtMarkerSuppliesTable
  - `.6` description, `.7` unit, `.8` maxCapacity, `.9` level
  - Percentage: `(level / maxCapacity) * 100`
  - maxCapacity=-2: treat level as approximate percentage

**Page counters:**
- `1.3.6.1.2.1.43.10.2.1.4` — prtMarkerLifeCount

**Input trays:**
- `1.3.6.1.2.1.43.8.2.1` — prtInputTable
  - `.8` maxCapacity, `.9` currentLevel, `.10` mediaName, `.13` trayName

**Output trays:**
- `1.3.6.1.2.1.43.9.2.1` — prtOutputTable

**Marker info:**
- `1.3.6.1.2.1.43.10.2.1` — prtMarkerTable (markTech, resolution)

### Vendor-Specific SNMP OIDs

- **HP**: `1.3.6.1.4.1.11.2.3.9.4.2.1.1`, `1.3.6.1.4.1.11.2.3.9.4.2.1.4`
- **Brother**: `1.3.6.1.4.1.2435.2.3.9.4.2.1.5.5`
- **Epson**: `1.3.6.1.4.1.1248.1.2.2`
- **Xerox**: `1.3.6.1.4.1.253.8.53.13`
- **Canon**: `1.3.6.1.4.1.1602`
- **Ricoh**: `1.3.6.1.4.1.367.3.2.1`
- **Samsung**: `1.3.6.1.4.1.236.11.5.11.81`
- **Lexmark**: `1.3.6.1.4.1.641.6`
- **Konica Minolta**: `1.3.6.1.4.1.18334`
