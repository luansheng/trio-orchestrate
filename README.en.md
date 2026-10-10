# trio-orchestrate

A skill that launches and orchestrates a parameterized team of agents with one command (default: kimi orchestrates / codex reviews / claude executes; **the orchestrator backend can be named from the verified list kimi/claude via the roster**).

## 1. Overview

trio-orchestrate is a multi-agent collaboration skill: it splits a task among several **slots**, each keeping to its own lane — in the default team, kimi orchestrates and accepts, codex reviews independently, and claude executes.

From r6 on, members are no longer fixed by three hard-coded names but determined by the **roster declaration** given at invocation: slot names (protocol identity) and backend CLIs (the programs actually doing the work) are fully decoupled, a team may have 2~N members, and optional members include genox. From r7 on, **the orchestrator is no longer hard-coded to kimi either** — point `orch` at any backend on the **verified list** (kimi/claude) and that backend orchestrates. Entering the list means running the capability matrix in "Orchestrator profile" of `SKILL.md` cell by cell (it can receive ACKs, dispatch work, and stay alive on its own); "it has a hook" or "it starts up" is not enough. When kimi's quota runs out, claude can take orch directly. From r8 the branching is simplified one layer further: every CLI has **exactly one** launch parameter set and there is one way of clearing permissions — no per-environment tiers and no enabling conditions (see §5).

| Slot | Capability | Default backend | Responsibility |
|---|---|---|---|
| `orch` | orchestrate + accept | kimi (or another listed backend, e.g. claude) | dispatch work, re-check the key numbers, keep the taskboard; exactly one per roster, irreplaceable; the backend must come from the **verified list** (kimi/claude, see "Orchestrator profile" in `SKILL.md`), and a backend whose machine default model is the weakest tier (claude here) requires an explicit model declaration |
| `rev` | review | codex | independent review; verdicts go to disk with a pointer |
| `exec` | execute | claude | claims only [机核] items whose method is already frozen and whose acceptance line is machine-checkable |
| `rev2` / `revtest` / `exec2`…`exec9` | extensions | your choice | a second review line / a review+test composite slot / more execution lines |

The actual agent name = **`<slot>-<task tag>`** (e.g. `orch-g07`, `rev-g07`, `exec-g07`).

The skill depends on herdr (terminal multiplexer) to create panes, start agents, relay messages and watch status: the user only needs to load this skill inside a session of **any verified orchestrator backend** (`/skill:trio-orchestrate` on a kimi host, `/trio-orchestrate` on a claude host), and the remaining panes are created automatically, with the orch on the left and the other members running down the right column (the default team being `orch` left | `rev` top right | `exec` bottom right).

Three principles shape the design. First, **acceptance rests on persisted evidence** (exact-equality ACK comparison, result files, re-running and re-computing), not on status observations or written reports. Second, **there is exactly one way of clearing permissions** — each CLI's launch parameters are fixed once at launch, not split into two sets depending on whether a human happens to be around; the permission mode only saves confirmations and a worktree only separates files and the index, so neither is a **security boundary**, and the real hard boundaries (remote protection / credential isolation / a process-level sandbox) are the user's call for their own environment. Third, **the reviewer must not be same-origin as the doer** — within a roster, rev-capability members and exec-capability members must not share a CLI and must not share a model family.

## 2. Installation

Copy the **whole `trio-orchestrate` directory** (including `claude-exec-deny.json`) into the skill directory of **the backend you intend to use as orchestrator**:

```bash
cp -r trio-orchestrate ~/.kimi-code/skills/     # kimi host
cp -r trio-orchestrate ~/.claude/skills/        # claude host (or a project's .claude/skills/)
```

Then check the hash, so old copies cannot get mixed in (the taskboard header records this same hash):

```bash
sha256sum ~/.kimi-code/skills/trio-orchestrate/SKILL.md   # compare with the "skill version" sha256 line in the board header
```

`claude-exec-deny.json` is the execution line's permanent deny list and must be copied together with `SKILL.md`; without it, the execution line loses that layer of permission constraint. **Publishing/reinstalling = copying the whole directory into the target backend's skill directory and then checking the hash matches**; when several backends each hold a copy, "which skill text launched this task" is settled by the board header's sha256.

## 3. Prerequisites

- **Any verified orchestrator backend on this machine** (required): kimi or claude — the skill runs inside whichever backend holds orch (the default team's orchestrator is kimi); the criterion is the capability matrix (it can receive ACKs, dispatch work, and stay alive on its own), see "Orchestrator profile" in `SKILL.md`; this machine's default claude model is the weakest tier, so **using claude as orch requires an explicit model declaration in the roster**;
- **herdr** (required): handles tabs/panes, agent registration and status detection; only when `herdr integration status` shows the status hook installed is idle/working detection reliable — otherwise only manual `pane read` watching is left;
- **codex / claude** (the default team's review and execution backends, swappable): if either is missing, fall back to "Team assembly and substitutes" — enumerate all combinations from the capability's candidate pool, under the hard constraint that **review and execution must not share a CLI and must not share a model family** (same-origin blind spots mean reviewing equals not reviewing), and that every exec member has a non-same-origin signing member; the `orch` slot cannot be substituted (a missing orch always stops and waits for the user; nothing is auto-restarted);
- **genox** (optional): it may only take a rev/exec-class slot, **never orch** (it is not on the verified list), and only that one interactive set is usable (**bare `genox`**; herdr does not recognize genox, so there is no status hook); see "Backend profiles" in `SKILL.md`;
- **Each CLI's own login state**: each CLI must already be logged in on this machine. Before first use, run it through manually once: each CLI's login, the repository trust confirmation and the first-time confirmation of bypass mode must not be left to a period nobody is watching.

## 4. Quick Start

```
/skill:trio-orchestrate <tag> <roster>       # kimi host
/trio-orchestrate <tag> <roster>             # claude host
```

(a claude host obtains the parameters through the same-named `arguments:` declaration and the body's text substitution; on a host with no skill support — codex/copilot etc. — the user writes the **literal values** of tag/roster into the message and the orchestrator writes those literals into the assignment lines before executing; the three paths and the iron rule that "parameters that cannot be obtained must never fall back silently" are in "Orchestrator profile" of `SKILL.md`.)

Both parameters are optional: `<tag>` is the task tag (only `[a-z0-9_-]`, at most 24 characters), used both as the tab name and as the task directory `/tmp/<tag>/`; `<roster>` is the team declaration, and omitting it uses the default three-member team.

| Team you want | How to write it |
|---|---|
| The default three-piece set (kimi orchestrates / codex reviews / claude executes) | `/trio-orchestrate g07` (roster left empty) or `/trio-orchestrate g07 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude` |
| 2 members: orchestration + review/execution combined | `/trio-orchestrate hotfix orch=kimi+revtest=codex:gpt-6.1-sol:high` |
| 4 members: two execution lines, the second on genox | `/trio-orchestrate bench4 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude+exec2=genox` |
| Swapping the backend holding orch (e.g. kimi's quota is exhausted) | `/trio-orchestrate tune2 orch=claude:claude-opus-5-5+revtest=codex:gpt-6.1-sol:high` (orch=claude **must declare model explicitly**) |

The roster is a **single string**, members joined by `+`, each member written as `<slot>=<cli>[:<model>[:<effort>[:<family>]]]` — for example `rev=codex:gpt-6.1-sol:high` means the review slot runs codex's `gpt-6.1-sol` at reasoning effort `high`; `exec=claude` gives only the backend and leaves the rest at that CLI's defaults.

The launch flow in outline:

1. **Parameter and team validation (mandatory)**: tag and roster first pass the model gate and the whole-string regexes; the roster is then checked slot by slot against the vocabulary, member count ≥2, all three capabilities present, **exactly one orch whose backend is on the verified list** (orch=claude additionally requires an explicit model), **review and execution not same-origin**, and **every exec member having a non-same-origin signing member** (a conflict reports the frozen wording); any failure stops the launch; only an empty roster falls back to the default team, and before that fallback the default team's orch backend is checked to be usable on this machine;
2. **Create the panes**: open a separate tab, split it into N panes with the layout loop and write `panes.txt` to disk; in a git repository, first create one worktree per non-orch slot, then start the agents with those directories;
3. **Start the agents**: take the **single** launch parameter set per slot from "Backend profiles" and rename them all; after startup, verify the real permission mode and the model/reasoning effort;
4. **bootstrap loop**: the orchestrator sends a bootstrap file carrying this run's batch id and waits for `orch` to answer; only when the ACK is **exactly equal** to the expected text (`cmp -s`) is "launch complete" declared, and the exclusive lock is released right away;
5. **Dispatch**: the taskboard `/tmp/<tag>/taskboard.md` has four columns (Backlog / In progress / Pending review / Accepted); every task carries a task ID and a base commit, and receipts are the files on disk.

## 5. Slots and Launch Parameters

The default team's three members (the actual agent name carries the task tag, e.g. `orch-g07`):

| Slot | Capability | Default backend | Launch parameters (exactly one set per CLI) |
|---|---|---|---|
| `orch` | orchestrate + accept | kimi (or another listed backend, claude; see "Orchestrator profile") | `--auto`; with orch=claude, claude's **same** parameter set + `--model <the value declared in the roster>` (no separate relaxed list) |
| `rev` | review | codex (model slug and reasoning effort set separately) | `--approve-for-me` |
| `exec` | execute | claude (each person chooses) | `--dangerously-skip-permissions` + `--settings` mounting `claude-exec-deny.json` + `--add-dir "$TD"` |

What the table above gives is each CLI's **single** launch parameter set, not split by environment. The permission mode only saves confirmations and a worktree only isolates files and the index, so neither is a **security boundary**; the real hard boundaries (remote branch protection / credential isolation / a process-level sandbox) are the user's call for their own environment — the skill does not judge that for them, nor does it change the launch parameters on that account.

The parameters are fixed at launch and checked against what is actually displayed afterwards; **the permission mode is a property of the process** — changing a shell variable or writing a board note cannot touch an already running process, so switching means stopping it and relaunching with the same set.

Each slot's launch parameters are not hard-coded here; always look them up per that slot's CLI in the "**Backend profiles**" section of `SKILL.md`, one lookup per roster member.

## 6. Known Limitations

The boundaries, stated honestly (see the "Known limitations" section of `SKILL.md` for detail):

- **The permanent push ban is not a hard block**: the entries in `claude-exec-deny.json` are **textual prefix rules**, a soft defense line that equivalent formulations can bypass (e.g. `git push origin HEAD:main`, `+refspec` force-push, the `git -C` wrapper, argument-less `git push`); a true permanent ban needs remote branch protection or permission-restricted credentials;
- **Reading secrets only covers direct reads within cwd**: `~/.ssh/`, CLI credential directories, and indirect reads such as a subprocess opening files itself are all out of reach; dangerous forms enter the list by **enumeration**, and enumeration is inherently incomplete — so an environment nobody is watching should not mount non-essential credentials;
- **A worktree is not a security boundary**: it isolates only working files and the index, while git metadata (config/refs) is shared with the main tree; "only the orchestrator can touch the main tree" is a disciplinary convention that does not hold mechanically;
- **Substitute backends have no equivalent control**: when claude is absent the `--settings` mount becomes void and a substitute CLI has no equivalent mechanism, so that slot's automatic clearance has no equivalent defence line, and a hard boundary there needs a process-level sandbox of its own;
- **`/tmp` may be cleaned by the system**: the taskboard, ACKs and verdicts all live in `/tmp/<tag>/`; the key manifest, the final verdict and recovery pointers must be stored separately in a persistent location inside the project;
- **The heartbeat mechanism belongs to the current session**: each host calls it something different but the principle is the same (on a claude host it is an in-session scheduled task); a new session does not inherit it and must recreate it — it may not be treated as a shared facility available across sessions;
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
