# Rollback rules — deriving the undo script by hand

A rollback script is a *derived* artifact: the inverse of a change, computed
against the pre-change config. Deriving it naively — prefixing every line with
the vendor's negation word — is wrong in ways that range from useless to
destructive. These are the rules a careful derivation follows. They are written
out because the derivation is judgment work, not mechanical work.

Input: the change script + the pre-change running config (the baseline
snapshot). Output: a rollback script where every line is either a verified
inverse, an explicit TODO filled from the snapshot, or an omission with a
written reason.

## Rule 0 — read the change against the baseline first

For every line of the change script, classify it against the pre-change config:

| Classification | Meaning | Rollback action |
|---|---|---|
| **create** | The object does not exist in the baseline | Delete the whole object |
| **modify** | The object exists; the line changes an attribute | Restore the previous value from the snapshot |
| **negate** | The line removes config (`no …`, `undo …`, `delete …`) | Re-add what was removed, from the snapshot |
| **idempotent** | The line's end state already matches the baseline | Omit — with a written reason |
| **navigation** | Session plumbing, not state | Omit — always |

The classification is what a naive prefixer skips, and skipping it is how a
rollback re-shuts a working link or deletes an interface that existed before
the change.

## Rule 1 — negation prefix per vendor family

| Vendor family | Negation | Example |
|---|---|---|
| Cisco IOS / IOS-XE, Arista, Ruijie, Aruba | `no` | `no vlan 40` |
| Huawei VRP / H3C | `undo` | `undo vlan 40` |
| Juniper Junos | `delete` (the `set` keyword is dropped) | `set interfaces xe-0/0/0 mtu 9000` → `delete interfaces xe-0/0/0 mtu` |
| MikroTik RouterOS | no negation form — remove the object | `/ip address remove [find …]` |
| Fortinet FortiGate | `unset`/`delete` inside the owning `config …` block | context-dependent |

## Rule 2 — blocks are undone as blocks

A block is a header followed by indented children (`interface …`, `vlan …`,
`router ospf …`, `config system …`).

- A block the change **created** → delete the whole block: `no interface
  Loopback77`. One line, not one per child.
- A block that **existed before**, where the change added children → remove
  only the added children. Never emit the block deletion against a
  pre-existing parent: `no interface Ethernet0/2` on an interface that existed
  before the change deletes the interface with all its config — that is a
  destructive rollback, not an undo.
- A block that existed before, where the change modified attributes → restore
  each attribute's previous value from the snapshot, inside the block.

## Rule 3 — restore values from the snapshot, never from memory

A modify line's previous value exists only in the pre-change snapshot. If the
rollback cannot state it, write an explicit placeholder:

```
# TODO restore from snapshot: <what the value was> — see <snapshot file>
```

A TODO line is a deliberately incomplete command. It must never survive to an
apply: fill it from the snapshot first, or stop.

## Rule 4 — MikroTik `find` must be narrowed

RouterOS has no negation form, so the rollback removes objects. A generated
`remove [find]` with a broad or empty condition can match more than the change
created. Narrow every `find` to the exact object the change added — by the
unique comment the change set, or the exact address/name:

```
# change:    /ip address add address=10.99.99.1/24 interface=lo comment="chg-demo-2026"
# rollback:  /ip address remove [find comment="chg-demo-2026"]     ← narrowed
# NOT:       /ip address remove [find]                        ← deletes every address
```

Best practice: give every object the change creates a unique comment, and make
the rollback remove by that comment.

## Rule 5 — FortiGate needs its context

FortiGate config is a tree of `config … end` blocks that apply on input. A
rollback line is only meaningful inside the owning block:

```
config system interface
    edit <ifname>
        unset <attribute>
    next
end
```

When the context cannot be determined confidently, emit an explicit
`# TODO manual rollback: <line>` rather than guessing the block.

## Rule 6 — navigation lines are never rolled back

`configure terminal`, `end`, `exit`, `system-view`, `return`, `commit`,
`quit` — session plumbing. Omit them from the rollback unconditionally.

Comment-only content (IOS `!` separators) is stripped before classification,
or the rollback fills with negated comment lines.

## Rule 7 — dangerous shapes get a visible warning

Some inverses are technically correct and operationally terrible:

- **`no shutdown` in the change** → the rollback shuts the interface. If that
  interface carries traffic, the rollback is an outage. Mark it:
  `# WARNING: this re-shuts the link — decide consciously`.
- **Rolling back a management-path change** (VTY, access-class, default route,
  the interface the session rides) → the rollback can cut the session itself.
  Same warning, louder.
- **A negation of something the baseline already lacked** → idempotent, omit
  with a reason. Rolling it back would *create* config the device never had.

## Rule 8 — read the result line by line

The finished rollback is read as critically as the change: every line either
undoes exactly what the change did, restores an exact snapshot value, or
carries a warning/TODO that has been consciously resolved. A rollback that was
never read is a second, unreviewed change waiting for a bad day.

## Error markers — knowing the apply actually failed

Per-line error detection decides whether the rollback is ever needed. Treat a
reply as failed when it matches, case-insensitively:

| Vendor family | Error markers |
|---|---|
| Cisco / Arista / Ruijie / Aruba | `% Invalid input`, `% Incomplete command`, `% Ambiguous command`, `% Unrecognized command`, `% Error` |
| Huawei / H3C | `Error:`, `Wrong parameter`, `Too many parameters`, `Unrecognized command found` |
| Juniper | `syntax error`, `unknown command`, `statement not found` |
| MikroTik | `syntax error`, `failure:`, `bad command name`, `invalid value`, `no such item`, `expected end of command` |
| FortiGate | `command parse error`, `Unknown action`, `Unknown attribute` |

A line starting with `%` is worth treating as an error even without the word
`Invalid` — a rejected command is more often reported than explained. Note
that RouterOS exec-mode exits 0 even on failure: the output patterns, not the
exit code, are the verdict.

## Worked example

Change (Cisco IOS):

```
interface Loopback77
 description chg-test-multiline
 ip address 10.77.77.1 255.255.255.255
```

Classification against the baseline: `Loopback77` does not exist → **create**.
Rollback:

```
no interface Loopback77
```

One line — the block was created, so it is deleted whole. If instead the
change had been `ip address 10.77.77.1 255.255.255.255` on an interface that
already had `10.77.77.2`, the rollback would be
`ip address 10.77.77.2 255.255.255.255` — the previous value, read from the
snapshot, not guessed.