# Change log — the record that survives a dead session

Every config/rollback attempt is written to an append-only change log
**before** it touches the device, and settled with a verdict afterwards. The
ordering is the whole point: an attempt that dies mid-flight still leaves its
entry, so the log is the one record that survives a dead session — use it
before blaming the transport, the credentials, or the device.

The log lives beside the inventory (one file or one table — see
`network-inventory`). Only config/rollback lifecycle events belong in it;
reads and reachability probes do not.

## Entry lifecycle

```
<pending>   written immediately before the first line is sent
   ↓
<started>   the same entry, marked as having reached a verdict
   ↓
<ok | failed | aborted>
```

| State | Meaning | What it tells you |
|---|---|---|
| `config-pending` / `rollback-pending` | Written before apply; **never settled** | The process died mid-apply — device state is UNKNOWN |
| `config-started` / `rollback-started` | A pending entry that was later settled | An attempt happened at that timestamp and reached a verdict |
| `config-ok` | The device accepted the apply | Functional success still needs the planned read-only checks |
| `config-failed` | The device rejected commands, or the session failed | The error output or transport failure is in the transcript |
| `config-aborted` | Died before or while applying | A guard fired or the session was cut |
| `rollback` | Rollback succeeded | The stored `check_point` was consumed and cleared |
| `rollback-failed` | The rollback apply was rejected | The undo did not land — investigate before retrying |

Each entry carries: timestamp, device reference, reason, and the script path.
Write enough that a stranger (or you, next week) can reconstruct what was
attempted.

## Retention

Prune on every write: entries older than ~100 days go, the log is capped (a
few hundred entries is plenty), and only config/rollback lifecycle actions are
kept at all. **`*-pending` entries never age out** — an unresolved pending is
exactly the artifact you must see.

## Reading the log for troubleshooting

In order:

1. **Pending first.** Anything unresolved means an apply died mid-flight and
   the device may not match the inventory's `config_list`. Verify the live
   config before trusting anything stored.
2. **Failures next.** `config-failed` / `rollback-failed` mean a rejected
   script — the transcript holds the device's own error.
3. **Then the device timeline.** Did someone change this device shortly before
   the symptom started? Was there an attempt that never reported a verdict?
4. Only then continue with live diagnosis (the OSI ladder).

An unexplained entry often *is* the cause: a change applied by someone else,
an apply that was interrupted, a device that drifted from the inventory.

## Recovering from an unsettled attempt

An unsettled entry means "attempted, outcome unknown". Recover in this order:

1. Read what is actually configured now — a targeted read-only pull against
   the areas the script touched.
2. Compare against the newest pre-change snapshot to see what the attempt may
   have changed.
3. If a `check_point` still exists for that device, replay it to restore the
   pre-change state — through the change gate, not improvised.
4. If the device is unreachable, treat it as an outage: the early layers
   (L1/L2, management path) of `network-troubleshooting` come first.

## What the log does NOT contain

- Command output — that lives in the config snapshots and session transcripts.
- Reads: inventory views, exec batches, and reachability probes are not
  audited — only inventory and config changes are.
- Anything older than the retention window (except pending entries).