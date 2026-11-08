---
name: network-change-management
description: "Gate network changes: plan, approval, rollback, verify."
version: 1.0.0
author: Dandi Suryadi, Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [network, change-management, rollback, approval, gate, cisco, mikrotik, huawei, juniper, fortinet]
    related_skills: [network-guidelines, network-inventory, network-device-cli, network-troubleshooting, network-security-audit]
---

# Network Change Management

The gate between "change script drafted" and "applied to a device". It never
touches hardware by itself — the user decides whether a change runs; this skill
makes sure the decision is informed and the way back exists before the first
line is sent.

Behavioral grounding: `network-guidelines` — especially *Verify Before, Not
After* and *Blast Radius First*. This skill turns those principles into a
procedure with a hard stop.

## Plan approval gate — HARDLINE

No config change — and no command that writes state — until the user has seen
all five blocks below in chat and replied with an explicit go-ahead. This holds
even when the user asked for the change in the same message, and even when the
change is "just one line".

1. **Plan + brief rationale** — what changes, why, what it fixes or enables.
   Plain language first: one-line analogy for the mechanism, then the blocks.
   The user's approval is on the *content*, so the plan must be readable
   without decoding CLI jargon.
2. **Configuration to be applied** — the exact script, verbatim, in a code
   block. Not a summary, not "and similar lines".
3. **Rollback** — the exact undo script, verbatim, derived per
   `references/rollback-rules.md` and read line by line before it is shown.
4. **Post-apply verification** — the exact read-only command that proves it
   worked, and the expected output token or state. Written before applying,
   never invented after.
5. **Risk** — what the change could break (especially the management path the
   session rides), which lines carry that risk, and why they are acceptable.

Then stop. End the turn and wait. `references/plan-template.md` is the same
five blocks in the shape the user reads.

- Approval covers **exactly** that script. If any line changes after approval —
  even one character — re-gate and re-ask. The old yes is void.
- "Continue" / "ok" / "go ahead" on a *different* message counts as approval
  for the plan shown immediately before it. A generic "do it" that arrives
  **before** any plan has been shown is not approval — show the plan first.
- Never apply a subset to "test the waters", and never chain a second
  unreviewed change after a successful one.
- If verification after apply does not show the expected state, report the real
  output and stop. Do not re-apply, do not fix it with an unplanned change.

## Procedure — before every config change

1. **Fresh baseline.** Pull the running config (see `network-inventory`) and
   archive it. The rollback is only as accurate as the config it was derived
   from; a stale baseline makes every TODO fill a guess.
2. **Write the change script** to a scratch file. Full-path commands for
   exec-style CLIs (RouterOS keeps no context between lines); config-mode
   blocks for PTY CLIs.
3. **Derive the rollback** per `references/rollback-rules.md`: classify every
   line against the baseline (create / modify / negate / idempotent /
   navigation), apply the vendor negation rules, fill every placeholder from
   the snapshot, narrow every broad `find`, and read the result line by line.
4. **Present the five blocks** and end the turn.
5. **Only after the explicit go-ahead**, apply through the execution path:
   pre-change snapshot → apply → save → read-back. Capture the per-line
   replies; a per-line error marker (not the session's overall exit code)
   decides success — RouterOS exec-mode exits 0 even on failure.
6. **Verify** with the exact read-only commands promised in block 4. This needs
   no new approval. Quote the device's own output line and compare it against
   the expected state. The device echoing the config line back is not
   verification; reading the resulting state back is.
7. **Report**: changed lines confirmed, verification output, inventory
   (`config_list`, `check_point`) updated. If anything is missing, say so
   plainly and stop.

## Checkpoints and undo

Every applied change stores its rollback script in the device's `check_point`
(the inventory field), and the full record — applied script, rollback script,
snapshot path, result — goes into the append-only checkpoint history. The
newest `check_point` is what a fast undo needs; the history is what lets you
undo two changes instead of one.

When a change must be undone: list the stored checkpoints first, then replay
the recorded rollback — exactly what the approved apply stored, not an
improvised script that was never gated. If `check_point` is empty there is
nothing to undo; do not improvise.

A successful rollback also publishes its own archive pair (config + rollback)
and clears `check_point` — the inventory then truthfully says "nothing pending
to undo".

## Backup archives

Every successful apply (and rollback) publishes a config + rollback pair under
the archive layout, alongside the pre-change snapshots. Retention is a policy
decision for the environment — a sane default is a handful of archives per
month per device, the last few months in full plus a yearly keeper. The pair
is the durable record; scratch files from the session are not.

## Pitfalls

- **A successful apply is not the end.** Block 4's verification command is a
  promise; skipping it because the CLI reported success is the failure mode
  this gate exists to stop.
- **MikroTik rollbacks are object removals.** There is no `no` form; every
  rollback is a `remove [find …]`, and a broad `find` deletes more than the
  change created. Narrow to the exact object — a unique comment set by the
  change makes this trivial.
- **FortiGate rollbacks need their `config …` context.** Without it, emit an
  explicit TODO rather than guessing the block.
- **Huawei rolls back with `undo`, Junos with `delete`** (the `set` keyword is
  dropped: `set X` → `delete X`).
- **A `no shutdown`-shaped rollback re-shuts a working link.** It carries a
  visible warning for a reason — decide consciously, don't paste blindly.
- **TODO lines are deliberately incomplete commands.** The pre-change value
  exists only in the snapshot; never let a TODO reach the device.
- **Vendor auto-detection from config text** (RouterOS `/export verbose` vs
  Cisco-style) is a convenience, not an oracle — pass the vendor explicitly
  when the config shape is ambiguous.
- **Never change the management path inside a functional change.** If the
  change must touch it, that is its own gated change with its own recovery
  plan (console/out-of-band stated before applying).

## Verification

The gate is behavioral, so it is verified by exercising it: a change proposed
without the five blocks must be refused; a one-line change must still be
gated; a script edited after approval must re-gate; verification output that
misses the expectation must stop the workflow, not trigger a re-apply. The
rollback rules are verified per shape: a created block rolls back to one
deletion line, a modified attribute rolls back to the snapshot value, a
negation of pre-existing config rolls back to a restore, and an idempotent
line is omitted with a written reason.