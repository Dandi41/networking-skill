---
name: network-guidelines
description: "Behavioral guidelines for network device work: change, review, diagnose."
version: 1.0.0
author: Dandi Suryadi, Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [network, guidelines, discipline, blast-radius, change-management, verification]
    related_skills: [network-inventory, network-change-management, network-troubleshooting, network-security-audit, network-device-cli]
---

# Network Guidelines

Behavioral guidelines to reduce common AI-agent mistakes when managing network
devices — derived from Andrej Karpathy's observations on LLM coding pitfalls
(https://x.com/karpathy/status/2015883857489522876). The failure modes
transfer, but the physics differ: code has an undo button and a test suite; a
network change lands on shared live state, often without either. So each
principle is re-derived for network work, not word-swapped.

Tradeoff: these guidelines bias toward caution over speed. For a description
line on an isolated lab port, use judgment.

## 1. Think Before Touching

Don't assume. Don't hide confusion. Surface the blast radius.

Before any change or diagnosis:

- State your assumptions about the topology explicitly: which interface faces
  where, who is behind it, which path your own session rides. If uncertain,
  ask — or read the device. Never infer one device's layout from another
  because they "look similar".
- If the request has multiple interpretations ("block port 80" — inbound or
  outbound? which zone? which direction?), present them. Don't pick silently.
- If a simpler approach reaches the same outcome — a static route instead of a
  routing protocol for two sites, one filter rule instead of a policy
  framework — say so. Push back when warranted.
- If something is unclear, stop. Name what is confusing. Ask.

The network-specific trap: inventing an IP, interface name, VLAN ID, AS
number, or next hop. Read it from the device or the inventory, or ask. A
hallucinated interface name in code is a bug; on a device it is an outage.

## 2. Minimum Config

Minimum configuration that solves the stated problem. Nothing speculative.

- No config beyond what was asked. No "while I'm in here" VLANs, ACLs, or
  hardening bundled into an unrelated change.
- No abstractions for single-use: don't build prefix-lists, object groups, or
  route-maps where one explicit line does the job.
- No "flexibility" that wasn't requested: no wildcard permits "for future
  hosts", no backup routes "just in case" on links that cannot fail over.
- No handling for scenarios that cannot occur on this device in this role.
- If the change script is 40 lines and could be 6, rewrite it.

Config that exists must be maintained, and nobody remembers why line 47 is
there. Ask: "would a senior network engineer looking at this diff say 'why is
this here?'" If yes, simplify.

## 3. Surgical Change

Touch only what you must. Clean up only your own mess.

- Don't reformat adjacent config, reorder ACL lines, or "tidy" descriptions.
- Don't refactor what isn't broken — don't migrate OSPF to BGP because you
  prefer it, don't rename existing objects to your taste.
- Match the device's existing conventions: interface description style, object
  naming, comment language.
- If you notice an unrelated problem — a stale static route, an unused ACL, a
  flapping port — report it. Don't fix it inside this change.
- When your change creates orphans, remove them: the old address you moved, the
  ACL entry your new rule supersedes, the description that no longer matches.
  Don't remove pre-existing cruft unless asked.

The test: every changed line traces directly to the request.

## 4. Verify Before, Not After

Define success as an observable device state before applying. This is the
sharpest inversion of the coding rule: in code you loop until tests pass,
cheaply, as many times as you like. On a live network you often get one shot,
rollback costs a window, and "it applied cleanly" proves nothing.

- Transform vague goals into verifiable ones:
  - "fix the routing" → "these prefixes must appear in the table via the
    expected next-hop, and a ping across must pass"
  - "make it faster" → "this counter must stop climbing across two readings"
  - "harden the device" → "these services must be gone from the listener list"
- Write the verification command and its expected output **before** writing
  the change. Never invent the check after applying.
- `STATUS: APPLIED` (or the device echoing your lines back) means only "the
  CLI did not reject the input". It is not functional success. Read the
  resulting state back and quote the device's own output line.
- Counters need two readings separated by a real pause; only a climbing number
  is a fault.
- For multi-step changes, state the plan with a verification per step:
  1. [step] → verify: [check]
  2. [step] → verify: [check]
- If verification shows something other than the expected state, report the
  real output and stop. Do not re-apply, do not patch with an unplanned change.

Weak criteria ("make it work") force constant clarification. Strong criteria
let you work — and stop — independently.

## 5. Blast Radius First

Before applying anything, answer: what does this take down with it?

- The management path is sacred. Identify the interface, VTY/line, access-class,
  and default route your own session depends on — and never change them in the
  same change that does anything else. Locking yourself out is total failure:
  on a remote box there is no console cable in the chat.
- Who else rides this path? A "harmless" interface change can carry a trunk, a
  routing adjacency, a DHCP relay, or someone else's default gateway.
- What happens mid-apply? A partial apply on a network is a state, not an
  exception: half an ACL is a different policy, half a routing config is a
  black hole. Prefer changes that are atomic per device, and know which lines
  are not.
- Is there a rollback, and has it been **read**, not just generated? A
  generated rollback can be wrong in dangerous ways — the classic is an
  injected `shutdown` on an interface that was up. Read it line by line; fill
  every placeholder from the pre-change snapshot before it is ever needed.
- State the blast radius out loud in the plan. "This touches only X" is the
  sentence the approver is actually buying.

## How the principles compose

1 → 4 is the loop: understand, minimize, isolate, prove. 5 is the gate that
runs before any of it touches a device. When principles conflict — the minimal
change happens to sit on the management path — 5 wins: split the change, use a
different path, or get explicit approval for the risk.

## Where the details live

| Task | Skill |
|---|---|
| Device inventory, reachability, config baseline | `network-inventory` |
| The approval gate, rollback discipline, plan format | `network-change-management` |
| Diagnosis when something breaks | `network-troubleshooting` |
| Security assessment | `network-security-audit` |
| Reaching a device CLI directly over SSH | `network-device-cli` |