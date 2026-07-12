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
  mcp__sprinter__find_network,
  mcp__sprinter__find_device,
  mcp__sprinter__ask_user
---

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Sprinter
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

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

## Multi-org context (important)

A user can belong to **more than one organization** (org). Every network belongs
to exactly one org, and MCP tools span **all** the user's orgs. The **network** is
the unit of selection — resolving a `network_id` automatically determines the org,
so you never ask the user to pick an org on its own. Org is only a **disambiguation
dimension**: surface it when two networks share a name across different orgs.

Every tool here returns an org label per row: `list_networks` includes `tenant_id`
+ `tenant_name`, and `find_network` / `find_device` include `tenant` (the org
name). When the user belongs to more than one org, show the org alongside the
network wherever you list or confirm a network.

## Procedure — infer from the prompt first, picker last

Resolve the network in this order. Stop at the first step that yields a single
match.

### Step 1: Did the prompt name a device?

If the user named a **device** (a name, an IP/MAC, or a description like "the
front-office printer") — with or without a network — call **`find_device`** with
that query. Omit `network_id` to search across every org (supply it if a network
was also named/known). A **single** device match resolves everything: the result's
`network_id` (and its `tenant`) is the answer — a device is globally unique, so its
network and org come with it. Done, no picker needed. If more than one device
matches, disambiguate (see "Disambiguate" below).

### Step 2: Else, did the prompt name a network by name?

Call **`find_network`** with the network name (matched as a case-insensitive
prefix across all your orgs). A single match resolves the `network_id` (and its
`tenant`). More than one match → disambiguate by org. (A subnet/CIDR mention
cannot be searched by `find_network` in v1 — fall through to the picker, Step 4.)

### Step 3: Prompt named both a device and a network?

Run both `find_device` (scoped to the named network if you resolved it) and
`find_network`. The **device** match is authoritative for the `network_id` (it is
the more specific signal); the network match sanity-checks it.

### Step 4: Nothing inferable → the org-aware picker

Call **`list_networks`** (spans all your orgs; each row has `network_id`, `name`,
`tenant_id`, `tenant_name`, `subnets`, `device_count`, `online_count`).

- **If a name was provided** and it matches exactly one network across all orgs,
  use it. If it matches networks in **more than one org**, present the matches
  **with the org** via `ask_user`:

  > "SD office network" exists in two organizations:
  > 1. **SD office network** — *Lincoln SD* — 10.0.0.0/24 (12 devices)
  > 2. **SD office network** — *Jefferson SD* — 10.0.1.0/24 (9 devices)
  > Which one?

- **If only one network exists**, use it automatically — no need to ask.

- **If multiple networks exist and no name was given**, present the list via
  `ask_user`. When the user belongs to **more than one org**, group by org:

  > Which network would you like to work with?
  > **Lincoln SD**
  >   1. **SD office network** — 10.0.0.0/24 (12 devices)
  >   2. **Lincoln High** — 10.0.2.0/24 (40 devices)
  > **Jefferson SD**
  >   3. **SD office network** — 10.0.1.0/24 (9 devices)

  When the user belongs to only one org, omit the org grouping (the flat list is
  fine). Omit networks with zero devices unless the user explicitly asks for all.

You may accept an `org/network` phrasing ("Lincoln SD / SD office network") as a
convenience — parse it leniently and, on any ambiguity, fall back to the picker.
Never **require** that form.

## Disambiguate (device or network)

When more than one candidate matches, present the candidates and ask via
`ask_user`, showing **org — network — device** (drop the device column when only
networks are ambiguous):

> Two devices match "front office printer":
> 1. **Lincoln SD** — *SD office network* — *HP-LaserJet-Front* (10.0.0.40)
> 2. **Jefferson SD** — *SD office network* — *Brother-MFC-Lobby* (10.0.1.51)
> Which one?

Because `find_device` / `find_network` are membership-scoped server-side, every
candidate shown is already one the user may access.

## Confirm and remember

Once a network is selected, report the choice back. Include the org when the user
belongs to more than one:

> Selected network **SD office network** in **Lincoln SD** (10.0.0.0/24, 12 devices).

Use the selected `network_id` for all subsequent MCP tool calls in this
conversation. If the user later says "switch network", "use a different network",
or invokes this skill again, repeat the procedure from Step 1.
