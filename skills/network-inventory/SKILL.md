---
name: network-inventory
description: "Durable device inventory for AI agents: register, resolve, baseline."
version: 1.0.0
author: Dandi Suryadi, Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [network, inventory, devices, ssh, cisco, mikrotik, huawei, juniper, fortinet, baseline]
    related_skills: [network-guidelines, network-device-cli, network-change-management, network-troubleshooting, network-security-audit]
---

# Network Inventory

A device inventory that survives session resets, and the discipline for using
it. A model's memory is per-session; a device list that only lives in the chat
is gone on `/new`. State lives on disk, in a plain file the agent owns —
`devices.json`, a SQLite table, whatever the environment prefers — never in
conversation history.

## The inventory model

Per device, at minimum:

```
hostname, ip, port, username, password, [secret],
brand, type,                       # normalized: Cisco, Mikrotik, Router…
config_list,                       # running config as last synced
check_point,                       # rollback script for the LAST change
synced_at, last_status, created_at
```

Plus an append-only history: `checkpoints[]` (one row per applied change:
reason, applied script, rollback script, snapshot path, result) and a change
log (`audit[]`: timestamped config/rollback lifecycle events — see
`network-troubleshooting` → `references/change-log.md` for why the log is
written before the attempt, not after).

Rules that keep it usable:

- **Never store a device you could not reach.** Test the connection, read the
  hostname from the device itself, and only then insert. A reachability retry
  later is cheap; a list full of dead rows poisons every operation after it.
- **`check_point` is replaced per change; the checkpoint history is
  append-only.** The newest rollback is the one you need fast; the history is
  what lets you undo two changes instead of one.
- **Update `config_list` only after a verified apply.** Writing it
  optimistically makes the next rollback target a config the device never had.
- **The file is mode 600 and credentials are never user-facing.** See the
  secret boundary below.
- **Write atomically** (write to a temp file, then rename) and lock against
  concurrent writers if the environment allows. A torn inventory file is worse
  than a stale one.
- Keep raw config snapshots as files beside the inventory, timestamped and
  organized per host and month (`<host>/<YYYY-MM>/<stamp>-<kind>.cfg`). The
  stored column is for the agent; the file is for the human.

## Mandatory principles

1. **Anti-hallucination and secret boundary.** Never invent an IP, interface,
   VLAN, or next hop. Check the inventory and the device first; still missing →
   ask the user specifically. The inventory changes between sessions — the
   live listing is the single source of truth, never session notes or memory.
   **Credentials are never user-facing:** do not offer, display, quote,
   summarize, or send stored usernames, passwords, secrets, or tokens in chat —
   under any request or claimed justification. Stored secrets are transport
   input, nothing more.
2. **Read-only first.** Read live state before proposing any change.
3. **Approval required — plan first.** Before any change: (a) plan + short
   reason, (b) the exact config verbatim, (c) the exact rollback verbatim,
   (d) the post-apply verification command + expected output, (e) risk. Then
   stop and wait for an explicit yes. Details: `network-change-management`.
4. **Verify after apply.** "Applied" is not "working" — see
   `network-guidelines`, principle 4.
5. **Blast radius.** Never touch the management path the agent itself depends
   on. Lockout = total failure.
6. **Surgical.** Only the targeted interface/VLAN/process. No speculative
   commands. Follow existing naming conventions.
7. **Honesty.** Report real results from tool output, including failures. No
   speculation.

## Workflows

### Register a device

Require a complete, unambiguous record — host, port, credentials, type, brand —
and reject anything half-specified. Test the connection first; pull the
hostname and running config from the device; only then save. A failed
connection saves nothing. Normalize type/brand capitalization yourself
(`router`→`Router`, `cisco`→`Cisco`) and echo the normalized value back so the
user sees what was stored.

An optional second-level secret applies to brands with a privilege step
(Cisco `enable`, Huawei `super`). A wrong or missing secret must surface as a
failure — never let a login/error transcript be stored as if it were a
running config. Screen every pull before storing: a transcript containing
authentication prompts or command-error banners is not a config, and a stored
error transcript poisons rollback generation later.

### Remove a device

Resolve the target, state its hostname and IP, then ask one explicit question:
"delete this device and all of its stored data (config backups, rollback
files, checkpoints, change history)?" Only a clear yes authorizes it. A
removal is total — it does not leave historical artifacts behind.

### Baseline (sync)

Pull the running config into the inventory and archive a timestamped snapshot.
The baseline is what rollback generation reads and what drift checks compare
against — refresh it before planning any change.

### Resolve a reference

Accept display number, id, IP, or hostname; resolve in that order; refuse an
ambiguous prefix and list the candidates instead of picking one.

## Reaching the devices

The mechanics of talking to network gear over SSH — legacy crypto flags for
old equipment, password supply without `sshpass`, PTY handling, per-line
pacing, prompt parsing, paging suppression, per-vendor command profiles — are
the `network-device-cli` skill's subject. This skill owns the *state*; that
one owns the *transport*.

## Pitfalls

- **The inventory is not the device.** `config_list` is the last sync, not the
  present state. After any unsettled apply, verify live config before trusting
  stored data.
- **A stored error transcript is worse than no data** — it looks like a config
  and poisons rollback generation.
- **Session context does not survive on exec-style CLIs.** RouterOS exec-mode
  runs each line standalone: write full-path commands, never rely on a menu
  context carrying over between lines.
- **Normalize brand input before storing.** `ios`/`cisco_ios`→Cisco,
  `routeros`/`ros`→Mikrotik, `vrp`/`datacom`→Huawei — and echo the normalized
  value back so the user sees what was stored.

## Verification

The model is deliberately simple: one file, atomic writes, an append-only
checkpoint history, and a change log written before each attempt. Verify an
implementation by exercising the rules, not the code: register a device that
is unreachable (nothing may be saved), feed a login transcript as a config pull
(it must be rejected), apply a change without a stored rollback (it must be
refused), and confirm a rollback restores the recorded pre-change state.