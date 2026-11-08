---
name: network-troubleshooting
description: "OSI-first network device diagnosis with vendor commands."
version: 1.0.0
author: Dandi Suryadi, Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [network, troubleshooting, osi, diagnosis, cisco, mikrotik, huawei, aruba, arista, juniper, ruijie, fortinet]
    related_skills: [network-guidelines, network-inventory, network-device-cli, network-change-management, network-security-audit]
---

# Network Troubleshooting

Diagnose a network device fault with the OSI ladder as the reasoning model and
live read-only evidence as the only thing that counts as proof.

This is a guided procedure, not an output parser. Vendors differ in syntax,
pagers, and error text — so the skill supplies the *order of thinking* plus
per-platform command hints, and you supply the interpretation from actual
output. Nothing here auto-verdicts from text patterns; that is your job.

## When to Use

- A device in the inventory is unreachable, slow, or behaving oddly.
- "No internet", "can't ping", "link down", "duplex errors", "it flaps".
- You need real state before deciding on a change.
- You must distinguish device down / path down / your own session dying.
- You need to rule out a change as the cause.

Not for: changing configuration (→ `network-change-management`),
adding/removing devices (→ `network-inventory`), security assessment
(→ `network-security-audit`), or plain Linux servers.

## Core principles

1. **OSI layers, don't skip.** L1 → L2 → L3 → L4 → L7. Prove the lower layer is
   healthy before moving up. Exception: a sharply specific symptom (DNS
   timeout) may go straight to its layer.
2. **Read-only first.** Every diagnostic command is read-only. No `set`,
   `configure`, `no`, `undo`, `clear`, `reset`, `shutdown`, `reload`. Changes
   are `network-change-management`.
3. **One symptom = one hypothesis.** Each hypothesis needs a command that can
   falsify it, named before you run it. No "try restarting" without evidence.
4. **Absolute counters mean nothing.** Read counters twice with a real pause;
   only a **climbing** number is a fault. Record both numbers and the interval.
5. **Record findings.** What was checked, key output, conclusion, what was
   ruled out. Without notes the same checks get repeated.
6. **A falsified hypothesis is discarded.** Don't patch an old hypothesis to
   fit new evidence — write a new one.
7. **Unknown stays unknown.** If you cannot determine something, say so. An
   ambiguous reachability result is a result.

## Procedure

The full step-by-step is **`references/diagnosis-playbook.md`** — eight phases,
each with an exit condition. Read it before starting; the summary here is the
shape, not the detail.

### Phase 0 — Scope

Device, symptom in the observer's words, since when, what's affected, and
whether anything changed recently. One question, then work.

### Phase 1 — Is it there?

Probe reachability — from a host, and from a neighbouring device as a second
vantage point. Ping failure is one signal, not a verdict: device down, ICMP
filtered, and broken session all look identical. Get a second vantage point
before concluding. Reachability is a question, not an assumption.

### Phase 2 — Baseline

Pull a fresh running-config snapshot (see `network-inventory`) and pin down
the exact OS and version — they decide which diagnostic commands are valid.
If SSH is dead but ICMP answers, work from an upstream neighbour.

### Phase 3 — Read state

One batch per device, saved to a file, grouped by layer. Session mechanics
(PTY vs exec-mode, pacing, prompt parsing, paging) are the
`network-device-cli` skill's subject. Command hints per platform:
`references/vendor-commands.md` — hints for confirmed syntax only; a rejected
command is not a fault, verify against official docs.

### Phase 4 — Climb or bisect

`references/osi-ladder.md` holds the symptom → layer map and the traps per
layer. Climb when you know nothing; bisect when the symptom is narrow (last
working point, first failing point).

When the symptom is routing-shaped — **adjacency up but routes missing, or
prefixes arriving from one side only** — go straight to
`references/routing-protocol-interop.md`. That file covers the failure modes
that produce no error at all: OSPF network-type mismatch (`FULL/BDR` on one
end, `not-reachable in topology Base` in the database, every route silently
discarded), RouterOS `passive` being a flag rather than a parameter, MTU and
authentication mismatches, and area/stub disagreement.

### Phase 5 — Two readings

Every counter claim needs `reading, pause, reading`. Flat is fine; climbing at
X per N minutes is the finding.

### Phase 6 — Research, in order

Vendor docs for the exact release → vendor advisories and release notes (a
known bug in that version is a valid root cause) → community for leads only,
verified before it enters the report. `references/official-sources.md`.

### Phase 7 — Report and propose

Symptom, scope, checked layers with evidence, confirmed facts, hypothesis with
a confidence level, what you ruled out, remaining unknowns, the next cheapest
test, and the proposed fix. **Never apply anything** — that is
`network-change-management` with its approval gate and rollback.

### When the symptom could be change-related

Read the change log **first** (`references/change-log.md`): attempts with no
verdict, then recorded failures, then the device's timeline. An unsettled
apply means live state is UNKNOWN — verify live config before trusting stored
data. Skip this for clearly physical/link symptoms; the ladder settles those.

## During an actual outage

Priority changes from "find the cause" to "restore service, then find the
cause". Note what you did and when, because a restart destroys evidence. If the
symptom began right after a change and a stored rollback exists, propose the
rollback through `network-change-management` instead of hand-fixing live.
Never change the interface carrying your own session.

## Cross-vendor practice

- Confirm vendor, OS/version, model, and role before choosing commands. Don't
  infer from notes about a different platform.
- The command tables are hints. Similar keywords do not guarantee identical
  behaviour across brands, and a release can change syntax.
- Prefer standard signals that exist everywhere: link state, counters,
  addressing, routes/neighbours, logs, end-to-end tests.
- Unfamiliar platform? Ask for safe read-only output or read the docs. Never
  run an unverified batch to "see what happens".
- Tools may help transport or preserve output, but must not replace reasoning
  or force brand-specific interpretation.

## Pitfalls

- **Never change the management path.** Identify the interface carrying your
  session before anything else; diagnosis does not authorize changes.
- **Verify platform syntax.** RouterOS has no `show` — every command starts
  with `/`. Huawei uses `display`. Assuming Cisco shapes across brands wastes
  time and can be read as a fault that isn't there.
- **Ping is one signal.** ICMP may be filtered by policy.
- **`up/up` is not healthy.** Rising CRC/input errors are a physical fault with
  a green light.
- **Two readings or no claim.** Absolute counters are archaeology.
- **A bug in that exact release is a legitimate root cause.** Check release
  notes before theorising about your own config.
- **Ruled-out items belong in the report.** They stop the next person
  repeating your work.

## Verification

No bundled parser — verification is the evidence: raw output saved per device
and layer, read and interpreted manually, adapted to the confirmed platform.

The change log is the first evidence step when symptoms may be change-related:
entries are written before each attempt and settled afterwards, so an empty
pending filter means nothing was left hanging and a non-empty one means a
process died mid-apply. See `references/change-log.md`.