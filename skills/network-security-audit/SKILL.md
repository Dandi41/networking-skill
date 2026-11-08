---
name: network-security-audit
description: "Audit network device security: local hardening and perimeter."
version: 1.0.0
author: Dandi Suryadi, Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [network, security, audit, cve, hardening, perimeter, compliance]
    related_skills: [network-guidelines, network-inventory, network-device-cli, network-change-management, network-troubleshooting]
---

# Network Security Audit

Security audit of a network device in two directions at once: **local** (is the
device itself hardened) and **perimeter** (what can reach it from outside).
Read-only for the entire run; remediation is a separate skill with its own
approval gate.

This is a guided procedure, not a bundled parser. Vendors differ in syntax,
output shape, and even in which controls exist — so the skill supplies the
*order of thinking* and a control catalog, and you supply the platform-specific
commands, verified against official documentation.

## Core rules

1. **Audit only when the user asks.** Never run one proactively — not "while
   we're at it", not "to be thorough". Triggers: an explicit audit request, a
   security check, or an incident that needs a security assessment.
2. **Read-only, start to finish.** Every audit command is `show` / `print` /
   `display` / `get`. No `set`, `configure`, `no`, `undo`, `write`, `save`,
   `commit`. If your evidence gathering would change state, it is the wrong
   command.
3. **Report, never authorize or veto.** The audit produces evidence, severity,
   and remediation advice. It never blocks, permits, delays, or decides a
   change the user asked for — including one that removes DHCP clients,
   firewall rules, services, routes, or accounts. The user decides; the change
   workflow handles consent, rollback, and verification.
4. **Evidence, not opinion.** Every finding carries verbatim command output
   plus, for CVEs, an official link. No output → `not checked`. No link →
   `not yet verified`.
5. **Both directions.** Local hardening and perimeter exposure. The perimeter
   is usually the more fatal side, and the combination findings ("telnet on,
   and reachable from the WAN") are the ones that get exploited.
6. **CVEs from research, not memory.** Version comes from the device, CVEs come
   from the vendor PSIRT then NVD, and relevance comes from the features you
   actually found enabled.
7. **Unknown is not safe.** An unreachable device or an unverifiable control is
   `unknown` / `not checked`. Never `pass`. Never "secure".

## When to Use

- "Audit the security of the lab RouterOS device", "check our network
  security", "is this device secure?"
- Before a device enters production (initial audit)
- After an incident: audit plus `network-troubleshooting` for forensics
- Fleet review, when the user names a set of devices

Not for: changing config (→ `network-change-management`), diagnosing outages
(→ `network-troubleshooting`), adding devices (→ `network-inventory`).

## Procedure

The full step-by-step is **`references/audit-playbook.md`** — seven phases,
each with its exit condition and per-platform command blocks. Read it before
starting; the summary here is the shape, not the detail.

### Phase 0 — Scope

One short question to the user: which devices, local or perimeter or both, and
whether an active outside-in test is permitted. Record it. Don't assume.

### Phase 1 — Identity and reachability

Confirm the device answers, then pin down exact vendor, model, OS train,
version, and build. No device-specific phase proceeds on a guess about the
platform. Version ≠ patch level.

### Phase 2 — Management plane

Services, protocols, accounts, session policy. This phase comes early because
exposed management is the most common real compromise path.

### Phase 3 — Perimeter and forwarding

Zones first, then filters in **rule order**. Read reachability of what you
found enabled in Phase 2. Counters tell you what the policy is doing; the
config tells you what it claims.

### Phase 4 — Control plane

SSH/SNMP versions, logging, NTP, routing authentication, L2 protections where
the role needs them, IPv6 handled deliberately.

### Phase 5 — Change trail and drift

Read the change log (`network-troubleshooting` → `references/change-log.md`):
unsettled applies first, then failures, then the device timeline. Out-of-band
changes and unsettled applies are findings in their own right; verify live
config before trusting stored snapshots.

### Phase 6 — Version, lifecycle, CVE

Follow `references/threat-research.md`: lifecycle status → vendor PSIRT → NVD
cross-check → feature applicability → record with URL and research date.

### Phase 7 — Report

`templates/audit-report-template.md`. Fill `Not checked` and `Limitations`
honestly; they are what make the report credible. Then stop — no remediation,
no suggested audit cadence.

## Multi-device runs

Repeat Phases 1–6 per device, one evidence file each, aggregate at the end.

- Never aggregate across an `unknown`: a fleet is not clean if one member was
  unreachable or one check went unverified.
- Same finding on many devices → one entry with the device list, plus whether
  the cause is shared (same image, same template, same operator habit).

## Division of labor across networking skills

| Skill | Role |
|---|---|
| `network-inventory` | device records, baselines, change history |
| `network-device-cli` | how to reach a device CLI directly over SSH |
| `network-troubleshooting` | outage diagnosis, OSI layers, hypothesis tests |
| `network-change-management` | change approval, rollback discipline |
| `network-security-audit` (this) | security findings, control catalog, CVE research, report |

Complementary flow: the audit finds → the user approves remediation →
`network-change-management` plans, gets approval, applies →
`network-troubleshooting` verifies nothing broke.

## Pitfalls

- **CVE without applicability evidence.** Match product, model, build,
  affected-version range, *and* the enabled feature before assigning severity.
  Otherwise it is INFO with a stated reason.
- **Version ≠ patch level.** Check the release's fixed-version advisories and
  changelog; do not infer patch status from the number's shape.
- **The perimeter needs an outside vantage point.** Config shows intended
  policy. Measured reachability needs an active test, explicit permission, and
  a note in the report that it was active.
- **Rule order, not rule count.** A default-permit above the deny makes a
  present-looking policy completely inert.
- **Unreachable ≠ hardened.** Mark `unknown`.
- **Don't run an audit unrequested.** A user rule, not a preference.
- **Audit ≠ outage diagnosis.** Slowness and flapping are
  `network-troubleshooting`.
- **A rejected command is not a finding.** Verify syntax against official docs
  before concluding the device lacks the control.

## Verification

There is no bundled parser, so verification is the evidence itself: real
output, pasted verbatim, adapted to the actual platform, with sources for every
external claim. The audit report is the record — audit runs are not written to
the device inventory.

The playbook's command blocks are compiled from vendor documentation and a
previously working read-only battery. Treat them as "confirmed syntax, pending
a full live pass" per platform: when a device of a given platform is in front
of you, run Phases 1–4 once and promote the result into a dated note here.