---
name: select-network
description: >
  Select or switch the active network for MCP tool calls. Use when the user
  wants to work with a specific network, switch networks, or when a tool
  call requires a network_id that hasn't been determined yet. Also use
  proactively before any MCP tool call that needs network_id if no network
  has been selected in this conversation.
argument-hint: "[network-name]"
allowed-tools: >
  mcp__sprinter__list_networks,
  mcp__sprinter__ask_user
---

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play. Call tools without describing the act
> of calling them. Surface only what matters to the user: the findings, the
> supporting evidence, and the verdict/next step. Keep any interim text minimal.

Select network $ARGUMENTS

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

## Procedure

### Step 1: Get the network list

Call **`list_networks`** to retrieve all available networks. The response
includes `network_id`, `name`, `subnets`, `timezone`, `device_count`, and
`online_count` for each network.

### Step 2: Match or ask

- **If the user provided a network name** (via $ARGUMENTS or earlier in the
  conversation), match it against the `name` field (case-insensitive). If
  exactly one network matches, use it. If multiple match, present the
  matches and ask the user to pick one using `ask_user`.

- **If only one network exists**, use it automatically — no need to ask.

- **If multiple networks exist and no name was given**, present the list to
  the user via `ask_user` and ask them to choose. Format the choices
  clearly, for example:

  > Which network would you like to work with?
  > 1. **home** — 10.0.14.0/24 (46 devices)
  > 2. **codeminders** — 192.168.12.0/24 (7 devices)
  > 3. **vlad** — 192.168.10.0/24, 192.168.11.0/24, 192.168.12.0/24 (23 devices)
  > 4. **apt319** — 10.0.0.0/24 (3 devices)

  Omit networks with zero devices unless the user explicitly asks for all
  networks.

### Step 3: Confirm and remember

Once a network is selected, report the choice back to the user:

> Selected network **home** (10.0.14.0/24, 46 devices).

Use the selected `network_id` for all subsequent MCP tool calls in this
conversation. If the user later says "switch network", "use a different
network", or invokes this skill again, repeat the procedure from Step 1.
