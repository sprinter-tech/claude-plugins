---
name: troubleshoot-printer
description: >
  Troubleshoot, diagnose, or check status of a network printer or
  multifunction device. Use when the user asks about printer status,
  toner/ink levels, paper jams, print errors, supply levels, page counts,
  or any printer-related question — whether they mention "printer"
  explicitly or provide an IP address of a known printer.
argument-hint: "[printer-address]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill, WebSearch,
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

Troubleshoot printer $ARGUMENTS

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

## Step 1: Resolve the Network

If the user mentions a network by **name** (e.g. "on network 'codeminders'"),
you must resolve it to a `network_id` before proceeding. **Network names and
network IDs are different things.** A network ID is a fixed-length opaque
string (e.g. `cma5pqbzu000001nmvtfv1m64`); a network name is a human-readable
label (e.g. `codeminders`). Never use a network name where a `network_id` is
required.

To resolve: call **`list_networks`**, match the user's name against the `name`
field (case-insensitive), and use the corresponding `network_id`. If multiple
networks match, use `ask_user` to let the user choose. If no match is found,
present the full list via `ask_user`.

If the user did not mention a network, you can get the `network_id` from the
`show_device` result in the next step.

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

**Check current identification status.** Tell the user the printer's
vendor, model, and device class. Confirm it is actually a printer before
proceeding with printer-specific diagnostics.

### Multiple Printers on the Network

If the user did not specify a printer address and there are multiple printers
on the network, use `ask_user` to ask which one to troubleshoot. **Do not
present raw hostnames** like `BRN001BA9DA156B` — these are vendor-assigned
internal names and are meaningless to the user. Instead, identify each printer
first (using `show_device` for each) and present choices using:

1. **Consumer model name** (e.g. "Brother MFC-J825DW", "HP LaserJet Pro M404n")
2. **Short type description** (e.g. "inkjet printer/scanner", "laser printer",
   "color laser multifunction")
3. **IP address** for disambiguation

Example `ask_user` prompt:

> Which printer would you like to check?
> 1. **Brother MFC-J825DW** — inkjet printer/scanner (10.0.14.52)
> 2. **Brother HL-L2350DW** — laser printer (10.0.14.53)

If the model name is not yet identified for a printer, do a quick
`show_device` or `snmp_get` on `1.3.6.1.2.1.1.1.0` (sysDescr) to get it
before presenting the choice. Fall back to the hostname only if identification
fails entirely.

## Step 3: Quick Status via IPP

Check the `open_ports` array in the `show_device` response from Step 2.
**If port 631 is listed**, try the `ipp_printer` tool. The `path` parameter
is required — try these paths in order until one succeeds:

1. `/ipp/print` — most modern printers (IPP Everywhere standard)
2. `/ipp/port1` — Brother printers (PCL queue)
3. `/ipp/printer` — some older HP and Lexmark models
4. `/ipp` — some Canon and Kyocera models

```
ipp_printer(address="10.0.14.52", path="/ipp/print", network_id="...")
```

If one path returns an error, try the next. Stop as soon as one succeeds.
If all fail, skip to Step 4 (SNMP). **Do not try paths beyond these four
without first searching the web for the specific printer model** — some
paths could cause the printer to print a test page.

**If the printer returns status 0x0503 (version-not-supported)**, it only
supports an older IPP version. Retry with `ipp_version="1.0"`:

```
ipp_printer(address="10.0.14.52", path="/ipp/port1", ipp_version="1.0", network_id="...")
```

This is common with pre-2010 printers (older Brother, HP, Lexmark).
The default IPP version is 2.0; most printers made after 2010 accept it.

The response includes:
- **Identity**: make/model, serial number, firmware version, UUID
- **Status**: printer state (idle/processing/stopped), active warnings
  and errors (toner-low, media-jam, door-open, cover-open)
- **Consumables**: per-supply name, type (toner/ink/drum), color, and
  level as a percentage (0–100). Special values: -1 = unknown,
  -3 = full.
- **Page counters**: total impressions completed, pages per minute

If IPP returns good data, you may already have enough to answer the
user's question without SNMP or web scraping. IPP consumable levels and
printer state reasons are the same data that SNMP provides via the
Printer MIB, but in a more accessible format.

**When IPP is not enough**, continue to Step 4 (SNMP) for:
- Input tray status (paper levels, media sizes) — not available via IPP
- Detailed alert table with severity codes
- Vendor-specific OIDs with part numbers and maintenance counters
- Output tray capacity

**If port 631 is not in `open_ports`**, skip this step — the printer does
not support IPP or the port is blocked. Proceed to Step 4 (SNMP).

## Step 4: Collect Printer Status via SNMP

SNMP is the richest source of printer status data. If the device has a
`working_channel` field in the `show_device` output, SNMP is reachable.
Use `snmp_get` and `snmp_walk` with the `working_channel` value as the
`channel` parameter.

### Printer MIB — Core Status OIDs

Walk these OID subtrees to get a comprehensive picture of the printer:

**Device identity (if not already known):**
- `1.3.6.1.2.1.1` — sysDescr, sysObjectID, sysName. Walk first.
- `1.3.6.1.2.1.25.3.2.1` — hrDeviceDescr (Host Resources device table).

**Printer status and alerts:**
- `1.3.6.1.2.1.25.3.5.1` — hrPrinterStatus and hrPrinterDetectedErrorState.
  - hrPrinterStatus values: 1=other, 2=unknown, 3=idle, 4=printing,
    5=warmup.
  - hrPrinterDetectedErrorState is a bit field: lowPaper, noPaper,
    lowToner, noToner, doorOpen, jammed, offline, serviceRequested,
    inputTrayMissing, outputTrayFull, markerSupplyMissing, outputNearFull,
    inputTrayEmpty, overduePreventMaint.
- `1.3.6.1.2.1.43.18.1.1` — prtAlertTable. Walk this for active alerts
  with severity, group, code, and description.

**Supply levels (toner, ink, drums, waste toner, fuser):**
- `1.3.6.1.2.1.43.11.1.1` — prtMarkerSuppliesTable. Key columns:
  - `.6` — prtMarkerSuppliesDescription (e.g. "Black Toner Cartridge")
  - `.7` — prtMarkerSuppliesSupplyUnit (units: 7=tenThousandthsOfInches,
    12=thousandthsOfOunces, 13=tenthsOfGrams, 15=percent, 19=sheets)
  - `.8` — prtMarkerSuppliesMaxCapacity (-1 means unknown, -2 means
    unknown but non-zero)
  - `.9` — prtMarkerSuppliesLevel (-1 means unknown, -2 means unknown
    but remaining, -3 means exhausted/at-limit)
  - To calculate remaining percentage: `(level / maxCapacity) * 100`.
    If maxCapacity is -2, treat level as an approximate percentage.

**Page counters:**
- `1.3.6.1.2.1.43.10.2.1.4` — prtMarkerLifeCount (total pages printed
  per marker/engine).
- `1.3.6.1.2.1.43.10.2.1.5` — prtMarkerPowerOnCount.
- `1.3.6.1.2.1.25.3.5.1.1` — hrPrinterStatus also useful cross-ref.
- Vendor-specific counters often live under private enterprise OIDs.

**Input trays (paper):**
- `1.3.6.1.2.1.43.8.2.1` — prtInputTable. Key columns:
  - `.9` — prtInputCurrentLevel (-1 unknown, -2 unknown but present,
    -3 means empty, 0 means empty)
  - `.8` — prtInputMaxCapacity
  - `.10` — prtInputMediaName (e.g. "Letter", "A4")
  - `.13` — prtInputName (tray label, e.g. "Tray 1", "Manual Feed")

**Output trays:**
- `1.3.6.1.2.1.43.9.2.1` — prtOutputTable. Key columns:
  - `.9` — prtOutputMaxCapacity
  - `.10` — prtOutputRemainingCapacity
  - `.7` — prtOutputName

**Marker (print engine):**
- `1.3.6.1.2.1.43.10.2.1` — prtMarkerTable. Key columns:
  - `.2` — prtMarkerMarkTech (1=other, 3=electrophotographicLaser,
    4=electrophotographicLED, 10=inkjet, etc.)
  - `.3` — prtMarkerCounterUnit
  - `.7` — prtMarkerAddressabilityXFeedDir (X resolution in pixels/line)
  - `.8` — prtMarkerAddressabilityFeedDir (Y resolution)

### Recommended SNMP Collection Sequence

Run these walks in parallel for efficiency:

1. `snmp_walk` OID `1.3.6.1.2.1.43.11.1.1` — all supply levels
2. `snmp_walk` OID `1.3.6.1.2.1.43.8.2.1` — input tray status
3. `snmp_walk` OID `1.3.6.1.2.1.43.18.1.1` — active alerts
4. `snmp_walk` OID `1.3.6.1.2.1.43.10.2.1` — marker/page counters
5. `snmp_get` OID `1.3.6.1.2.1.25.3.5.1.1.1` — printer status

If SNMP is not available (no `working_channel`), skip to Step 5.

### Vendor-Specific SNMP OIDs

Many vendors expose richer data under their private enterprise OIDs. After
identifying the vendor from sysDescr or sysObjectID, try these:

**HP/HPE printers:**
- `1.3.6.1.4.1.11.2.3.9.4.2.1.1` — HP LaserJet MIB (serial, model,
  firmware, job count).
- `1.3.6.1.4.1.11.2.3.9.4.2.1.4` — HP supply status (detailed
  cartridge part numbers and status).

**Brother printers:**
- `1.3.6.1.4.1.2435.2.3.9.4.2.1.5.5` — Brother supply and maintenance
  counters.

**Epson printers:**
- `1.3.6.1.4.1.1248.1.2.2` — Epson-specific status and ink levels.

**Xerox printers:**
- `1.3.6.1.4.1.253.8.53.13` — Xerox supplies MIB with part numbers
  and replacement info.

**Canon printers:**
- `1.3.6.1.4.1.1602` — Canon enterprise MIB for device status.

**Ricoh printers:**
- `1.3.6.1.4.1.367.3.2.1` — Ricoh MIB for printer status and supplies.

**Samsung/HP (formerly Samsung):**
- `1.3.6.1.4.1.236.11.5.11.81` — Samsung printer MIB.

**Lexmark printers:**
- `1.3.6.1.4.1.641.6` — Lexmark MIB for supplies and diagnostics.

**Konica Minolta / Bizhub:**
- `1.3.6.1.4.1.18334` — Konica Minolta enterprise MIB.

When you detect a vendor, try a walk of their enterprise subtree to discover
what additional data is available. Limit to a single walk of the top-level
vendor OID prefix first, then drill into interesting subtrees.

## Step 5: Collect Printer Status via Web Interface

Most modern printers have embedded web servers (EWS) that expose detailed
status pages. Use `network_http` to fetch and parse these pages.
**Never suggest the user open a browser** — fetch and read pages yourself.

### Discovery Strategy

1. **Start with the root page** — `network_http` with path `/`.
   Parse the HTML for navigation links to status, supplies, and
   configuration pages.

2. **Try common vendor-neutral paths:**
   - `/` — home/status page (often has supply levels in summary)
   - `/status` or `/status.html` — device status
   - `/supplies` or `/supplies.html` — consumable levels
   - `/info` or `/info.html` — device information
   - `/config` or `/configuration` — device configuration
   - `/DevMgmt/ProductStatusDyn.xml` — HP XML status API
   - `/DevMgmt/ConsumableConfigDyn.xml` — HP XML supply levels

3. **Vendor-specific paths (try after identifying vendor):**

   **HP printers (Embedded Web Server):**
   - `/hp/device/InternalPages/Index?id=SuppliesStatus` — supply status
   - `/hp/device/InternalPages/Index?id=UsagePage` — usage/page counts
   - `/hp/device/InternalPages/Index?id=ConfigurationPage` — config
   - `/DevMgmt/ProductUsageDyn.xml` — usage stats in XML
   - `/DevMgmt/MediaHandlingDyn.xml` — paper tray status
   - `/IoConfig/HttpConfig/` — HP embedded web config

   **Brother printers:**
   - `/general/status.html` — printer status overview
   - `/general/information.html` — device information
   - `/general/consumables.html` — toner/drum levels

   **Epson printers:**
   - `/PRESENTATION/HTML/TOP/PRTINFO.HTML` — printer information
   - `/PRESENTATION/HTML/TOP/INDEX.HTML` — main status page
   - `/Epson_IPD_PO_TOP.htm` — Epson status portal
   - `/PRESENTATION/ADVANCED/INFO_PRTINFO/TOP` — advanced info

   **Xerox printers:**
   - `/status/consumables` — supplies status
   - `/status/deviceStatus` — device status summary
   - `/properties/configuration.php` — configuration

   **Canon printers:**
   - `/English/pages/main.html` or `/tlogin.cgi` — status portal
   - `/English/pages/supplies.html` — supplies
   - `/portal.html` — newer Canon models

   **Ricoh printers:**
   - `/web/guest/en/websys/status/getSystemStatus.cgi` — system status
   - `/web/guest/en/websys/webArch/getDevice.cgi` — device info

   **Lexmark printers:**
   - `/cgi-bin/dynamic/printer/PrinterStatus.html` — printer status
   - `/cgi-bin/dynamic/config/gen/reports/GenerateReport.html` —
     generate reports

   **Konica Minolta / Bizhub:**
   - `/wcd/system_device.xml` — device status XML
   - `/wcd/supply.xml` — supply status XML
   - `/ws/status` — REST status endpoint

   **Samsung printers:**
   - `/sws/index.html` — Samsung web service portal
   - `/sws/app/information/supplies/suppliesView.sws` — supplies

4. **Parse the response.** Use `show_device` with
   `sections=http_body`, `body_url`, and `body_offset`/`body_limit` to
   page through large HTTP responses. Look for:
   - Percentage bars or text like "90%", "Low", "OK", "Replace"
   - Color names (Black, Cyan, Magenta, Yellow, Photo Black)
   - Tray names and paper sizes
   - Error messages and warning banners
   - Serial numbers and firmware versions

5. **Limit probing.** Do not try more than 15 URL paths total. If the
   first few paths give you good data, stop. Prioritize SNMP data over
   web scraping when both are available, as SNMP is more structured.

   When several Printer-MIB tables need correlating (supply levels by
   consumable, page counts by cover/tray, alert table + status reason),
   issue the individual `snmp_walk` / `snmp_get` MCP calls and join the
   rows by index in chat. **Do not attempt to run a Python snippet to
   bundle these probes** — there is no such tool available to this plugin.

## Step 6: Connectivity Check

If the user reports printing failures or the device seems unreachable:

1. **`network_ping`** — verify the printer is reachable and measure
   latency.
2. **`network_port_scan`** — check that printing ports are open:
   - Port 9100 (RAW/JetDirect) — most common network printing port
   - Port 631 (IPP/CUPS) — Internet Printing Protocol
   - Port 515 (LPR/LPD) — Line Printer Daemon
   - Port 80/443 (HTTP/HTTPS) — web interface
   - Port 161 (SNMP) — management
3. **`network_dns_lookup`** — if the printer is addressed by hostname,
   verify DNS resolves correctly.

## Step 7: Present Findings

Structure your report around what the user asked about. Always include:

**Printer identity:**
- Vendor, model, serial number, firmware version
- IP address, hostname, MAC address

**Current status:**
- Operational state (idle, printing, error, offline)
- Active errors or warnings (paper jam, door open, service needed)

**Supply levels** (present as a clear table):
- Toner/ink cartridges with remaining percentage and color
- Drum units with remaining life
- Waste toner container level
- Fuser/maintenance kit status if available
- Flag any supply below 10% as critical, below 20% as low

**Paper trays:**
- Each tray's paper level, media type, and size
- Flag empty or near-empty trays

**Page counters:**
- Total pages printed (lifetime)
- Color vs monochrome breakdown if available

**Connectivity** (if relevant):
- Network reachability and latency
- Open printing ports (9100, 631, 515)
- Any connectivity issues found

**Recommendations:**
- Supplies that need replacement soon
- Errors that need attention
- Configuration issues if detected
