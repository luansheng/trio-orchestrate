# trio-orchestrate

A skill that launches and orchestrates a parameterized team of agents with one command (default: kimi orchestrates / codex reviews / claude executes).

## 1. Overview

trio-orchestrate is a multi-agent collaboration skill: it splits a task among several **slots**, each keeping to its own lane — in the default team, kimi orchestrates and accepts, codex reviews independently, and claude executes.

From r6 on, members are no longer fixed by three hard-coded names but determined by the **roster declaration** given at invocation: slot names (protocol identity) and backend CLIs (the programs actually doing the work) are fully decoupled, a team may have 2~N members, and optional members include genox.

| Slot | Capability | Default backend | Responsibility |
|---|---|---|---|
| `orch` | orchestrate + accept | kimi | dispatch work, re-check the key numbers, keep the taskboard; exactly one per roster, irreplaceable |
| `rev` | review | codex | independent review; verdicts go to disk with a pointer |
| `exec` | execute | claude | claims only [机核] items whose method is already frozen and whose acceptance line is machine-checkable |
| `rev2` / `revtest` / `exec2`…`exec9` | extensions | your choice | a second review line / a review+test composite slot / more execution lines |

The actual agent name = **`<slot>-<task tag>`** (e.g. `orch-g07`, `rev-g07`, `exec-g07`).

The skill depends on herdr (terminal multiplexer) to create panes, start agents, relay messages and watch status: the user only needs to start kimi and load this skill, and the remaining panes are created automatically, with the orch on the left and the other members running down the right column (the default team being `orch` left | `rev` top right | `exec` bottom right).

Three principles shape the design. First, **acceptance rests on persisted evidence** (exact-equality ACK comparison, result files, re-running and re-computing), not on status observations or written reports. Second, **granting permissiveness is conditional** — whether to use the unattended parameter set is decided by three gates before work starts, and it is not the default. Third, **the reviewer must not be same-origin as the doer** — within a roster, rev-capability members and exec-capability members must not share a CLI and must not share a model family.

## 2. Installation

Copy the **whole `trio-orchestrate` directory** (including `claude-exec-deny.json`) into kimi's skills directory:

```bash
cp -r trio-orchestrate ~/.kimi-code/skills/
```

`claude-exec-deny.json` is the execution line's permanent deny list and must be copied together with `SKILL.md`; without it, the execution line loses that layer of permission constraint.

## 3. Prerequisites

- **kimi CLI** (required): the skill itself runs inside kimi, and it is the orchestrator of the default team;
- **herdr** (required): handles tabs/panes, agent registration and status detection; only when `herdr integration status` shows the status hook installed is idle/working detection reliable — otherwise only manual `pane read` watching is left;
- **codex / claude** (the default team's review and execution backends, swappable): if either is missing, fall back to "Team assembly and substitutes" — enumerate all combinations from the capability's candidate pool, under the hard constraint that **review and execution must not share a CLI and must not share a model family** (same-origin blind spots mean reviewing equals not reviewing); the `orch` slot cannot be substituted;
- **genox** (optional): it may only take a rev/exec-class slot, **never orch**, and only its "attended" set is usable (herdr does not recognize genox, so there is no status hook); see "Backend profiles" in `SKILL.md`;
- **Each CLI's own login state**: each CLI must already be logged in on this machine. Before first use, run it through manually once: each CLI's login, the repository trust confirmation and the first-time confirmation of bypass mode must not be left to an unattended period.

## 4. Quick Start

```
/skill:trio-orchestrate <tag> <roster>
```

Both parameters are optional: `<tag>` is the task tag (only `[a-z0-9_-]`, at most 24 characters), used both as the tab name and as the task directory `/tmp/<tag>/`; `<roster>` is the team declaration, and omitting it uses the default three-member team.

| Team you want | How to write it |
|---|---|
| The default three-piece set (kimi orchestrates / codex reviews / claude executes) | `/skill:trio-orchestrate g07` (roster left empty) or `/skill:trio-orchestrate g07 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude` |
| 2 members: orchestration + review/execution combined | `/skill:trio-orchestrate hotfix orch=kimi+revtest=codex:gpt-6.1-sol:high` |
| 4 members: two execution lines, the second on genox | `/skill:trio-orchestrate bench4 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude+exec2=genox` |

The roster is a **single string**, members joined by `+`, each member written as `<slot>=<cli>[:<model>[:<effort>[:<family>]]]` — for example `rev=codex:gpt-6.1-sol:high` means the review slot runs codex's `gpt-6.1-sol` at reasoning effort `high`; `exec=claude` gives only the backend and leaves the rest at that CLI's defaults.

The launch flow in outline:

1. **Parameter and team validation (mandatory)**: tag and roster first pass the model gate and the whole-string regexes; the roster is then checked slot by slot against the vocabulary, member count ≥2, all three capabilities present, **exactly one orch**, and **review and execution not same-origin** (a conflict reports the frozen wording); any failure stops the launch;
2. **Mode selection (mandatory)**: verify the three gates item by item and write the evidence into `/tmp/<tag>/mode-selection.md`, then choose the unattended or the attended parameter set accordingly; starting without doing mode selection = a process violation;
3. **Create the panes**: open a separate tab, split it into N panes with the layout loop and write `panes.txt` to disk; in a git repository, first create one worktree per non-orch slot, then start the agents with those directories;
4. **Start the agents**: take the parameter set per slot from "Backend profiles" and rename them all; after startup, verify the real permission mode and the model/reasoning effort;
5. **bootstrap loop**: the orchestrator sends a bootstrap file carrying this run's batch id and waits for `orch` to answer; only when the ACK is **exactly equal** to the expected text (`cmp -s`) is "launch complete" declared, and the exclusive lock is released right away;
6. **Dispatch**: the taskboard `/tmp/<tag>/taskboard.md` has four columns (Backlog / In progress / Pending review / Accepted); every task carries a task ID and a base commit, and receipts are the files on disk.

## 5. Slots and Modes

The default team's three members (the actual agent name carries the task tag, e.g. `orch-g07`):

| Slot | Capability | Default backend | Default mode (unattended parameter set) |
|---|---|---|---|
| `orch` | orchestrate + accept | kimi | `--auto` |
| `rev` | review | codex (model slug and reasoning effort set separately) | `--approve-for-me` |
| `exec` | execute | claude (each person chooses) | `--dangerously-skip-permissions` + `--settings` mounting `claude-exec-deny.json` |

The "default mode" column is the value of the **unattended parameter set**. Whether to use it is decided by the three gates before work starts, and if any gate fails, switch to the **attended parameter set** (explicit manual approval, with the real mode verified after startup; inheritance of the machine's own defaults is not accepted):

| Gate | What is verified |
|---|---|
| 1 Push boundary | the execution identity itself has no bypass permission; the target main/master is actually covered by branch protection; other branches allowed to be pushed also cannot be force-pushed |
| 2 Credential surface | the runtime environment mounts no non-essential credentials (or isolation via a separate Unix user / container), with a mount list kept |
| 3 Directory and permission scope | it is a git repository and a worktree can be created; the process's write boundary has a restricted scope (container / separate user), or the user has explicitly chosen the authorization scope |

Each slot's launch parameters are not hard-coded here; always look them up per that slot's CLI in the "**Backend profiles**" section of `SKILL.md`, one lookup per roster member.

## 6. Known Limitations

The boundaries, stated honestly (see the "Known limitations" section of `SKILL.md` for detail):

- **The permanent push ban is not a hard block**: the entries in `claude-exec-deny.json` are **textual prefix rules**, a soft defense line that equivalent formulations can bypass (e.g. `git push origin HEAD:main`, `+refspec` force-push, the `git -C` wrapper, argument-less `git push`); a true permanent ban needs remote branch protection or permission-restricted credentials;
- **Reading secrets only covers direct reads within cwd**: `~/.ssh/`, CLI credential directories, and indirect reads such as a subprocess opening files itself are all out of reach; dangerous forms enter the list by **enumeration**, and enumeration is inherently incomplete — so an unattended environment should not mount non-essential credentials;
- **A worktree is not a security boundary**: it isolates only working files and the index, while git metadata (config/refs) is shared with the main tree; "only the orchestrator can touch the main tree" is a disciplinary convention that does not hold mechanically;
- **Substitute backends have no equivalent control**: when claude is absent the `--settings` mount becomes void and a substitute CLI has no equivalent mechanism, so that slot's unattended capability is disabled along with it;
- **`/tmp` may be cleaned by the system**: the taskboard, ACKs and verdicts all live in `/tmp/<tag>/`; the key manifest, the final verdict and recovery pointers must be stored separately in a persistent location inside the project;
- **The heartbeat cron belongs to the current session**: a new session does not inherit it and must recreate it;
- **Members without hooks have no status semantics**: herdr does not recognize genox, so `herdr agent read` reports `agent_not_found` outright and idle/working can only be inferred from the screen with `herdr pane read`, which is weaker than a hook; the protocol's closed loop does not depend on it (it anchors on the on-disk ACK), but dispatching timeliness is degraded.

## 7. Files

| File | Description |
|---|---|
| `SKILL.md` | the skill itself (Chinese): the full launch protocol + runtime protocol |
| `SKILL.en.md` | full English translation of the skill, aligned section by section with `SKILL.md` |
| `claude-exec-deny.json` | the execution line's permanent deny list (sudo / disk destruction / force-push / push to main / reading secrets), mounted via `--settings`; every claude-family instance in a roster mounts one copy |
| `README.md` | this file's Chinese counterpart |
| `README.en.md` | this file: the English description, aligned with `README.md` |
| `LICENSE` | MIT licence |

## 8. License

MIT License, Copyright (c) 2026 sheng luan. The full text is in `LICENSE`.
