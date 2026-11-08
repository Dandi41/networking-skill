# networking-skill

Skills that teach an AI agent to manage network devices — routers, switches,
firewalls — over SSH. **Vendor-neutral and engine-neutral**: the skills define
the order of thinking, the gates, and the evidence discipline; the agent
supplies the platform-specific commands and the execution path.

## What it is

Six skills, one discipline:

| Skill | Role |
|---|---|
| `network-guidelines` | behavioral ground rules for all network work — the cornerstone |
| `network-inventory` | durable device inventory, registration, baselines, secret boundary |
| `network-device-cli` | reaching a device CLI over SSH: legacy crypto, askpass, PTY, pacing |
| `network-change-management` | the plan-approval gate, rollback derivation, checkpoint discipline |
| `network-troubleshooting` | OSI-first diagnosis: symptom → layer → evidence → report |
| `network-security-audit` | hardening + perimeter audit: 7 phases, control catalog, CVE research |

The guidelines skill is the one to read first — every other skill assumes it.

## Design principles

- **Procedures, not parsers.** No bundled audit or triage scripts: syntax and
  output vary across vendors, OS versions, roles, and models. The skills teach
  the agent to identify the platform, choose safe read-only commands, and
  interpret the output itself.
- **Evidence over assertion.** Every claim carries verbatim device output.
  "Applied" is not "working"; unknown is not "safe".
- **The gate is hard.** No config change without a five-block plan (plan,
  exact config, exact rollback, verification, risk) and an explicit user
  go-ahead — even for one line, even when the user asked in the same message.
- **Rollbacks are derived, then read.** A rollback is classified against the
  pre-change snapshot line by line; placeholders are filled from the snapshot,
  never guessed.
- **State lives on disk.** Device inventory, checkpoints, and a change log
  written *before* each attempt — nothing important lives only in the
  conversation.

## Install

Copy the skill directories into your agent's skill directory. For
[Hermes Agent](https://hermes-agent.nousresearch.com):

```bash
mkdir -p ~/.hermes/skills/networking
cp -R skills/* ~/.hermes/skills/networking/
```

Then restart the agent so the skills load. No other dependencies — the skills
are plain markdown; the only tools they assume are an SSH client and whatever
file access the agent already has.

## Requirements

- An agent that can run shell commands and read/write files.
- SSH reachability to the devices, with credentials the user provides.
- Nothing else. No Python, no daemons, no database server.

## License

MIT — see [LICENSE](LICENSE).