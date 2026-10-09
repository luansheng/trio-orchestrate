# trio-orchestrate

A skill that launches and orchestrates a three-agent collaboration of kimi / codex / claude with one command.

## 1. Overview

trio-orchestrate is a multi-agent collaboration skill: it splits a task among three roles, each keeping to its own lane — kimi orchestrates and accepts, codex reviews independently, claude executes. The three are identified by fixed names that do not change from task to task.

| Role | Fixed name | Responsibility |
|---|---|---|
| Orchestrate + accept | `kimi-orch` | dispatch work, re-check the key numbers, keep the taskboard |
| Review | `codex-review` | independent review; verdicts go to disk with a pointer |
| Execute | `claude-exec` | claims only [机核] items whose method is already frozen and whose acceptance line is machine-checkable |

The skill depends on herdr (terminal multiplexer) to create panes, start agents, relay messages and watch status: the user only needs to start kimi and load this skill, and the remaining panes are created automatically, in the fixed layout `kimi-orch` on the left | `codex-review` top right | `claude-exec` bottom right.

Two principles shape the design. First, **acceptance rests on persisted evidence** (exact-equality ACK comparison, result files, re-running and re-computing), not on status observations or written reports. Second, **granting permissiveness is conditional** — whether to use the unattended parameter set is decided by three gates before work starts, and it is not the default.

## 2. Installation

Copy the **whole `trio-orchestrate` directory** (including `claude-exec-deny.json`) into kimi's skills directory:

```bash
cp -r trio-orchestrate ~/.kimi-code/skills/
```

`claude-exec-deny.json` is the execution line's permanent deny list and must be copied together with `SKILL.md`; without it, the execution line loses that layer of permission constraint.

## 3. Prerequisites

- **kimi CLI** (required): the skill itself runs inside kimi, and kimi is the orchestrator;
- **herdr** (required): handles tabs/panes, agent registration and status detection; only when `herdr integration status` shows the status hook installed is idle/working detection reliable — otherwise only manual `pane read` watching is left;
- **codex / claude** (optional): the default backends of the review and execution lines. If either is missing, fall back to "substitute team assembly" — enumerate candidate pairs from the available CLIs, under the hard constraint that **review and execution must not share a CLI and must not share a model family** (same-origin blind spots mean reviewing equals not reviewing); `kimi-orch` cannot be substituted;
- **Each CLI's own login state**: each CLI must already be logged in on this machine. Before first use, run it through manually once: each CLI's login, the repository trust confirmation and the first-time confirmation of bypass mode must not be left to an unattended period.

## 4. Quick Start

```
/skill:trio-orchestrate <tag>
```

`<tag>` is the task tag (only `[a-z0-9_-]`, at most 19 characters), used both as the tab name and as the task directory `/tmp/<tag>/`. The launch flow in outline:

1. **Mode selection (mandatory)**: verify the three gates item by item and write the evidence into `/tmp/<tag>/mode-selection.md`, then choose the unattended or the attended parameter set accordingly; starting without doing mode selection = a process violation;
2. **Create the panes**: open a separate tab and split it into three panes in the fixed layout; in a git repository, first create one worktree each for the review and execution lines, then start the agents with those directories;
3. **Start the agents**: the three-piece set starts with the chosen parameter set and is renamed in full; after startup, verify the real permission mode and the model/reasoning effort;
4. **bootstrap loop**: the orchestrator sends a bootstrap file carrying this run's batch id and waits for `kimi-orch` to answer; only when the ACK is **exactly equal** to the expected text (`cmp -s`) is "launch complete" declared, and the exclusive lock is released right away;
5. **Dispatch**: the taskboard `/tmp/<tag>/taskboard.md` has four columns (Backlog / In progress / Pending review / Accepted); every task carries a task ID and a base commit, and receipts are the files on disk.

## 5. Roles and Modes

The three fixed roles (the actual agent name carries the task tag, e.g. `kimi-orch-g07`):

| Role | Fixed name | Default backend | Default mode |
|---|---|---|---|
| Orchestrate + accept | `kimi-orch` | kimi | `--auto` |
| Review | `codex-review` | codex (model slug and reasoning effort set separately) | `--approve-for-me` |
| Execute | `claude-exec` | claude (each person chooses) | `--dangerously-skip-permissions` + `--settings` mounting `claude-exec-deny.json` |

The "default mode" column is the value of the **unattended parameter set**. Whether to use it is decided by the three gates before work starts, and if any gate fails, switch to the **attended parameter set** (explicit manual approval, with the real mode verified after startup; inheritance of the machine's own defaults is not accepted):

| Gate | What is verified |
|---|---|
| 1 Push boundary | the execution identity itself has no bypass permission; the target main/master is actually covered by branch protection; other branches allowed to be pushed also cannot be force-pushed |
| 2 Credential surface | the runtime environment mounts no non-essential credentials (or isolation via a separate Unix user / container), with a mount list kept |
| 3 Directory and permission scope | it is a git repository and a worktree can be created; the process's write boundary has a restricted scope (container / separate user), or the user has explicitly chosen the authorization scope |

## 6. Known Limitations

The boundaries, stated honestly (see the "Known limitations" section of `SKILL.md` for detail):

- **The permanent push ban is not a hard block**: the entries in `claude-exec-deny.json` are **textual prefix rules**, a soft defense line that equivalent formulations can bypass (e.g. `git push origin HEAD:main`, `+refspec` force-push, the `git -C` wrapper, argument-less `git push`); a true permanent ban needs remote branch protection or permission-restricted credentials;
- **Reading secrets only covers direct reads within cwd**: `~/.ssh/`, CLI credential directories, and indirect reads such as a subprocess opening files itself are all out of reach; dangerous forms enter the list by **enumeration**, and enumeration is inherently incomplete — so an unattended environment should not mount non-essential credentials;
- **A worktree is not a security boundary**: it isolates only working files and the index, while git metadata (config/refs) is shared with the main tree; "only the orchestrator can touch the main tree" is a disciplinary convention that does not hold mechanically;
- **Substitute backends have no equivalent control**: when claude is absent the `--settings` mount becomes void and a substitute CLI has no equivalent mechanism, so that line's unattended capability is disabled along with it;
- **`/tmp` may be cleaned by the system**: the taskboard, ACKs and verdicts all live in `/tmp/<tag>/`; the key manifest, the final verdict and recovery pointers must be stored separately in a persistent location inside the project;
- **The heartbeat cron belongs to the current session**: a new session does not inherit it and must recreate it.

## 7. Files

| File | Description |
|---|---|
| `SKILL.md` | the skill itself (Chinese): the full launch protocol + runtime protocol |
| `SKILL.en.md` | full English translation of the skill, line-for-line aligned with `SKILL.md` |
| `claude-exec-deny.json` | the execution line's permanent deny list (sudo / disk destruction / force-push / push to main / reading secrets), mounted via `--settings` |
| `README.md` | this file's Chinese counterpart |
| `README.en.md` | this file: the English description, aligned with `README.md` |
| `LICENSE` | MIT licence |

## 8. License

MIT License, Copyright (c) 2026 sheng luan. The full text is in `LICENSE`.
