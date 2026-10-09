---
name: trio-orchestrate
description: "One-command launch and orchestration of three collaborating agents: kimi (orchestrate + accept) / codex (review) / claude (execute). Use when the user wants to start a new multi-agent task, or says 'set up three-way collaboration / start work / launch review and execution / set up the taskboard'. Takes an argument: /skill:trio-orchestrate <task tag>; the argument is used directly as the task tag. Depends on herdr (terminal multiplexer) to create panes, start agents, relay messages and watch status. The user only needs to start kimi and load this skill; the remaining panes are created automatically."
arguments:
  - tag
---

# Three-agent collaboration: one-command launch + orchestration (kimi orchestrates · codex-review reviews · claude-exec executes)

> Author: sheng luan
> Dependencies: kimi CLI + herdr; if codex/claude is missing, fall back to "substitute team assembly". How to share: copy the whole `trio-orchestrate/` directory (including `claude-exec-deny.json`) into the other party's `~/.kimi-code/skills/`.
> This skill is both a **launch protocol** and a **runtime protocol**. After sharing it, if you worry that ordinary conversation might accidentally trigger tab creation / mode switching, you may optionally add `disableModelInvocation: true` to the frontmatter so it is invoked only by explicit command.

## Roles and fixed names (no longer varying per task)

| Role | Fixed name (agent name + pane label) | Backend | Mode |
|---|---|---|---|
| Orchestrate + accept | `kimi-orch` | kimi | `--auto` |
| Review | `codex-review` | codex, **model and reasoning effort set separately** (e.g. model slug `gpt-6.1-sol` + reasoning effort `high`; the exact model is whatever appears in each person's codex `/model` list) | `--approve-for-me` |
| Execute (layer C) | `claude-exec` | claude (each person chooses the backend; record the CLI kind and provider/model on the taskboard) | `--dangerously-skip-permissions` + at startup `--settings` mounts the skill directory's `claude-exec-deny.json` permanent deny list (sudo / disk destruction / force-push / push to main / reading secrets; bound only to claude-exec, does not affect the user running claude separately; everything else is allowed). **This list is only a textual prefix soft defense line, not a security boundary — see "Known limitations" for the forms that bypass it** |

> The "Mode" column above gives the values of the **unattended parameter set**; whether to use them is decided by the three-gate result of "**Mode selection**" in Step 0 — if any gate fails, switch to the **attended parameter set**; they are not defaults.

**Naming rule (key to multiple tabs coexisting)**: the actual agent name = `<role>-<task tag>`, e.g. `codex-review-g07tune`. Reason: herdr's **agent names are a global registry, unique across tabs/workspaces** (a collision was measured to report `agent_name_taken`; a pane label is only a display label, not an identity); legal characters are only `[a-z0-9_-]`, ≤32. The three-piece set in one tab shares one tag; the pane label stays consistent with the agent name. **The reuse check must use JSON exact matching of the three full agent names**, not substring matching such as `grep -- "-<tag>"`: `herdr agent list` returns compact single-line JSON, and grep would treat `g070` as `g07` and count another task's whole line in (measured 2026-10-09). See "Step 0" for an example. Another tab's three-piece set does not count, and is never reused across tasks.

## Step 0: Launch the panes (three brand-new panes, a separate tab)

> The code block below is a **conceptual flow**, not a script you can paste as a whole: execute it step by step and **check the return code of every step**; if any prerequisite fails, stop immediately (fail-fast). Always parse P0/P1/P2 from the JSON returned by the command, never copy them by hand; see the example below. Before first use, run it through manually once: each CLI's login, the repository trust confirmation, and the first-time confirmation of bypass mode must not be left to an unattended period. **All code blocks are written in Bash syntax** (parameter sets use arrays, `"${ARR[@]}"` turns each element into its own argv; Zsh does not split unquoted scalars and behaves differently, so do not copy this directly into Zsh).

```bash
command -v herdr >/dev/null || { echo "herdr not installed, stop and tell the user"; exit 1; }
[ "${HERDR_ENV:-}" = 1 ] || { echo "not running inside herdr, stop and tell the user"; exit 1; }
herdr integration status           # whether each agent's status hook is installed: binary present but hook missing → status detection unreliable
for a in kimi codex claude; do command -v $a >/dev/null && echo "$a ✓" || echo "$a missing → use substitute team assembly"; done

# ── Model gate (the first gate, before any command containing TAG, executed by the orchestrator = model) ──────────────
# $tag has already been text-expanded into this document when the skill loads — the assignment line below is itself shell source code:
# Writing the validation after the assignment cannot block text that enters the source code with $tag "before the assignment". The real first gate is
# reading-based judgement: after reading the expanded value of $tag, first check that the whole string matches ^[a-z0-9_-]{1,19}$ (empty / spaces /
# uppercase / over-long / newline are all illegal); if illegal, stop and ask the user to re-enter — do not type any line below, and do not
# write the illegal value into any command. This is a model gate, not a shell gate; "assign first, validate afterwards" is not allowed.
TAG=$tag                           # call argument: /skill:trio-orchestrate <tag> (the skill: prefix is required); if empty, wait for the user to name it when assigning the task. Once set it stays fixed for the whole run, and doubles as the tab label and the taskboard directory /tmp/<TAG>/
# Second line of defense (shell level; it blocks continued use after the assignment, and cannot replace the model gate above): whole-string validation
# (measured 2026-10-09: $'g07\n../outside' slips through the old grep -qE line anchoring; bash [[ =~ ]] on the whole string rejects it)
[[ "$TAG" =~ ^[a-z0-9_-]{1,19}$ ]] || { echo "TAG invalid (empty / contains characters outside [a-z0-9_-] / over 19 characters / contains a newline), reject and ask the user to re-enter"; exit 1; }
# 19 = 32 (agent name ceiling) − 13 (length of the codex-review- prefix); anything over-long is always rejected, never silently truncated (truncation makes two tags collide, measured 2026-10-09)

# ── Task directory: classify before taking the lock, fix the path plan first; never overwrite unconditionally ──────────────────────
# Order (ruling N3, 2026-10-09): create the directory → directory/file safety check → fix the path plan → classify (new / reentry / resume)
#   → only if the classification passes, take the exclusive lock → create/verify the manifest.
# Do not take the lock before classifying: the freshly taken lock itself makes the directory non-empty, so a legitimate new launch would be killed as "non-empty and no manifest" (self-blocking).
# Exclusivity: one TAG allows only one launch flow at a time (concurrently sharing the ACK file is itself not allowed); the lock is released after the launch completes (ACK passes),
# if it stops on a mid-way failure, do not delete the lock yet — finish the manual handling first, then delete the lock and re-enter, to prevent accidental reentry
# ══ First-run flow block (start): extract this whole block, set TAG and PWD (the project root cwd), and it runs standalone ══
TD="/tmp/$TAG"                                          # task directory (state directory); this block always uses $TD
mkdir -p "$TD"                                          # idempotent

# Directory/file safety: check every path read or written — not a symlink (`[ -L ]`), owner is yourself; a non-existent path counts as creatable
guard() {
  [ -e "$1" ] || [ -L "$1" ] || return 0
  [ -L "$1" ] && { echo "reject: $1 is a symlink (it may point outside the task directory)"; return 1; }
  [ "$(stat -c %u -- "$1" 2>/dev/null)" = "$(id -u)" ] || { echo "reject: $1 is not owned by the current user"; return 1; }
  return 0
}
for p in "$TD" "$TD/manifest.txt" "$TD/taskboard.md" "$TD/bootstrap.md" "$TD/bootstrap-ack.md" "$TD/ack-expected" "$TD/mode-selection.md"; do
  guard "$p" || { echo "task directory/file safety check failed: stop for manual confirmation"; exit 1; }
done

ROOT=$(realpath -m "$PWD")                              # canonical absolute project root: the anchor of task identity
# The path plan is fixed before the manifest is generated (this is not "compute the sibling worktrees first, then reject the non-git fallback the body allows"):
#   git repo → the three role cwds = main tree + two sibling worktrees; non-git → all three cwds record the main tree (allowed only in attended mode)
if git -C "$PWD" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  PATH_PLAN=worktree
  EXPECT_ORCH=$(realpath -m "$PWD")
  EXPECT_REVIEW=$(realpath -m "$PWD/../wt-$TAG-review")
  EXPECT_EXEC=$(realpath -m "$PWD/../wt-$TAG-exec")
else
  PATH_PLAN=main-only
  EXPECT_ORCH=$(realpath -m "$PWD"); EXPECT_REVIEW="$EXPECT_ORCH"; EXPECT_EXEC="$EXPECT_ORCH"
fi

# ① Classify: read-only judgement, write nothing yet
if [ -s "$TD/manifest.txt" ]; then
  CLASSIFY=reentry
elif [ -n "$(ls -A "$TD" 2>/dev/null)" ]; then
  echo "$TD is non-empty and has no manifest (leftovers, a lock from a concurrent launch, or another task): stop for manual confirmation, do not auto-clean"; exit 1
else
  CLASSIFY=new                                          # empty directory → legitimate new launch
fi
# ② Only after the classification passes, take the exclusive lock
mkdir "$TD/launch.lock" 2>/dev/null || { echo "a launch flow is already running (launch.lock): no concurrent launch, stop"; exit 1; }

# ③ Per the classification, create the manifest / verify task identity
if [ "$CLASSIFY" = reentry ]; then
  # Reentry: **unique parse** of the manifest — required fields (tag / project-root / the three cwd lines / created / bootstrap-id)
  # exactly one line each with a non-empty value; a field name appearing twice is rejected (contradictory values do not count as "field-by-field agreement"); one cwd line per role, compared by realpath
  _mout=$(python3 - "$TAG" "$ROOT" "$EXPECT_ORCH" "$EXPECT_REVIEW" "$EXPECT_EXEC" "$TD/manifest.txt" <<'PY'
import os, sys
tag, root, eo, er, ee, path = sys.argv[1:7]
want_cwd = {f"kimi-orch-{tag}": eo, f"codex-review-{tag}": er, f"claude-exec-{tag}": ee}
seen = {}
for line in open(path, encoding="utf-8"):
    line = line.rstrip("\n")
    if not line:
        continue
    if ":" not in line:
        sys.exit(f"unparseable manifest line: {line!r}")
    k, v = line.split(":", 1)
    if not v.strip():
        sys.exit(f"manifest field {k} has an empty value")
    seen.setdefault(k, []).append(v.strip())
def one(k):                                             # a required field must be exactly one line; a duplicate is a conflict
    vs = seen.get(k, [])
    if len(vs) != 1:
        sys.exit(f"manifest field {k} should be exactly one line (got {len(vs)})")
    return vs[0]
if one("tag") != tag: sys.exit("tag mismatch")
if one("project-root") != root: sys.exit("project-root mismatch")
one("created"); one("bootstrap-id")                     # present, exactly one line, non-empty value
cwds = seen.get("cwd", [])
if len(cwds) != 3: sys.exit(f"there should be exactly three cwd lines (got {len(cwds)})")
got = {}
for v in cwds:
    name, sep, p = v.partition(" ")
    if sep != " " or not p.strip(): sys.exit(f"cwd line is missing its separator or the path is empty: {v!r}")
    if name in got: sys.exit(f"cwd role {name} is duplicated")
    got[name] = p.strip()
if set(got) != set(want_cwd): sys.exit("the set of cwd roles does not match this run")
for name, exp in want_cwd.items():
    if os.path.realpath(got[name]) != os.path.realpath(exp):
        sys.exit(f"{name} cwd={got[name]} ≠ expected {exp} (still different after realpath normalization)")
print("manifest unique parse passed: no duplicate fields, no empty values, the three role cwds match this run", file=sys.stderr)
for _n in (f"kimi-orch-{tag}", f"codex-review-{tag}", f"claude-exec-{tag}"):
    print(got[_n])                                  # stdout emits only the three verified paths (fixed order) for the shell to read back line by line
PY
  ) || { echo "manifest does not match this run (duplicate field / empty value / project root / role path / missing path): not recognized as this task, stop for manual confirmation"; exit 1; }
  # The expected cwds are emitted directly by the same parser above (reentry does not reassemble paths or parse a second time — an earlier awk re-parse had field semantics that drifted from the manifest line format and blanked all three EXPECTs, 2026-10-09 N4); read them back line by line and guard against empty values
  { IFS= read -r EXPECT_ORCH; IFS= read -r EXPECT_REVIEW; IFS= read -r EXPECT_EXEC; } <<< "$_mout"
  [ -n "$EXPECT_ORCH" ] && [ -n "$EXPECT_REVIEW" ] && [ -n "$EXPECT_EXEC" ] || { echo "an expected cwd read back on reentry is empty: stop for manual confirmation"; exit 1; }
  echo "reentry: manifest unique parse agrees, reuse the existing taskboard, fill gaps only, never overwrite"
else
  CREATE_ID=$(date +%s)-$RANDOM                         # directory-creation batch id: part of task identity, taken only on a new launch
  cat > "$TD/manifest.txt" <<EOF
tag: $TAG
project-root: $ROOT
cwd: kimi-orch-$TAG $EXPECT_ORCH
cwd: codex-review-$TAG $EXPECT_REVIEW
cwd: claude-exec-$TAG $EXPECT_EXEC
created: $(date -Is)
bootstrap-id: $CREATE_ID
EOF
  echo "new: task directory initialized (path plan $PATH_PLAN; lock taken, released after the ACK passes)"
fi
# ══ First-run flow block (end) ══

# Reuse check: exact JSON matching of the three full agent names; then verify each role's cwd == that role's **expected cwd** (git plan = that line's worktree;
# non-git fallback = the main tree) — the expected values always come from the manifest above
# (it is not a requirement that the three cwds be equal — the three sharing a tab but wrongly running in another directory still cannot be reused); the expected paths come from the manifest above,
# and both sides are compared after realpath normalization — a path built with .. and the canonical cwd pointing at the same directory must compare equal, otherwise a normal reentry would be misjudged;
# if either tab_id or workspace_id is missing, exit non-zero (fail-closed); "it managed to print a conclusion" does not count as passing
herdr agent list | python3 -c '
import json,os,sys
agents=json.load(sys.stdin)["result"]["agents"]
tag,main,wt_review,wt_exec=sys.argv[1:5]
expect={f"kimi-orch-{tag}":main, f"codex-review-{tag}":wt_review, f"claude-exec-{tag}":wt_exec}
def norm(p):
    return os.path.realpath(p) if p else p
got={a.get("name"):a for a in agents}
missing=[w for w in expect if w not in got]
if missing:
    print("missing: "+", ".join(missing)); sys.exit(2)
def bad(msg):
    print(msg); sys.exit(3)
for w,exp in expect.items():
    a=got[w]
    if not a.get("tab_id") or not a.get("workspace_id"):
        bad(w+" is missing tab_id/workspace_id, fail-closed: do not reuse")
    cwd=a.get("cwd")
    if norm(cwd)!=norm(exp):
        bad(w+" cwd="+str(cwd)+" ≠ expected worktree="+str(exp)+" (still different after realpath normalization): do not reuse")
tabs={got[w].get("tab_id") for w in expect}
if len(tabs)!=1:
    bad("the three-piece set is not in the same tab: "+", ".join(w+"@"+str(got[w].get("tab_id")) for w in expect))
print("three-piece set complete: same tab, and each cwd matches its expected worktree (after realpath normalization)")' "$TAG" "$EXPECT_ORCH" "$EXPECT_REVIEW" "$EXPECT_EXEC"
# Exit codes: 0 = reusable; 2 = a role is missing (fill in only the missing ones); 3 = topology/directory mismatch or a missing field (fail-closed, do not auto-reuse; rebuild after manual confirmation)
# The expected cwds always come from the manifest: on a new launch they are written from the path plan fixed in the "first-run flow block" (git → main tree + two sibling worktrees;
# non-git → all three cwds record the main tree, allowed only in attended mode); on reentry they are extracted by the unique parse — here the three expected paths are passed to the reuse check
```

### Mode selection (after detection, before launch — mandatory)

**The startup parameters are the result of mode selection, not defaults; starting without doing mode selection = a process violation.** The task directory, the manifest (project root / each role's expected cwd / creation time / bootstrap ID) and the exclusive lock have already been handled in the "detection" step above; this step verifies the three enablement gates of "unattended mode" one by one, and each one leaves evidence in `/tmp/$TAG/mode-selection.md` (evidence that cannot be written clearly = that gate fails). Once the taskboard is created, paste a pointer to that file into it:

| Gate | What is verified (not just "a protection flag exists") | What the evidence records |
|---|---|---|
| 1 Push boundary | ① **the execution identity itself has no bypass permission** (both direct pushes by remote admins and the bypass list of the protection rules must be checked); ② the target ref (main/master) is actually covered by protection rules; ③ other branches that are allowed to be pushed also cannot be force-pushed | the actual branch protection settings (e.g. `gh api repos/<owner>/<repo>/branches/main/protection` to see `enforce_admins` and the rules), or read-only evidence from the credential side; if it cannot be verified → fail |
| 2 Credential surface | record **which execution instance** (the claude-exec-$TAG process), **which credentials are required** (keep only each CLI's own login state), the **list of sensitive paths** (mount points, readable directories), and the isolation method (separate Unix user / container / not mounting) | a minimal verifiable isolation configuration + mount list; a container existing ≠ no sensitive data mounted, a separate user name ≠ files unreadable — write it out item by item |
| 3 Directory and permission scope | **worktree availability and the process's permission scope are verified separately**: a worktree isolates only working files and the index, and cross-directory writes by the same Unix user are not constrained by it (see "Known limitations" item 3) | ① it is a git repository and a worktree can be created; ② the process's write boundary: there is a restricted scope such as a container / separate user, **or the user explicitly chose the "allow same-user cross-directory access" authorization scope** (if chosen, restate unattended item 3 as that chosen scope, and stop claiming that crossing directories always requires confirmation); if no restricted scope can be verified and no explicit choice was made → fail |

All three gates pass → **unattended parameter set**; any gate fails → **attended parameter set**, and state on the taskboard which gate failed and what evidence failed:

| Parameter set | kimi-orch | codex-review | claude-exec |
|---|---|---|---|
| Unattended parameter set (all three gates pass) | `--auto` | `--approve-for-me` | `--dangerously-skip-permissions --settings ${KIMI_SKILL_DIR}/claude-exec-deny.json` |
| Attended parameter set (any gate fails) | no `--auto` (its semantics are Never Ask, see below) | **explicit** `-a on-request -c approvals_reviewer=user` (inheritance is not accepted: omitting the flag ≠ manual approval — this machine's user config was measured to contain `approval_policy = "never"`; the approver is decided separately by `approvals_reviewer`) | **explicit** `--permission-mode default` (from ≥2.1.283, not passing bypass may still inherit auto or the user's `permissions.defaultMode` setting, **inheritance is not accepted**) + `--settings` (deny only tightens, never loosens); after startup, verify the real mode |

```bash
# The parameter sets use Bash arrays (all code blocks in this skill are Bash-only; eval is forbidden — quote characters must not be stuffed into a string expecting a second round of parsing.
# Building argv by string concatenation is wrong: Bash splits on spaces, so a path with a space breaks the --settings path into two items;
# Zsh is worse — scalars are not split automatically at all, so the whole set becomes one argument. Measured 2026-10-09, two modes × two shells × eight samples)
# (a) unattended parameter set — all three gates pass
KIMI_ARGS=(--auto)
CODEX_ARGS=(--approve-for-me)
CLAUDE_ARGS=(--dangerously-skip-permissions --settings "${KIMI_SKILL_DIR}/claude-exec-deny.json")
# (b) attended parameter set — any gate fails; replace the three lines above with (codex explicit policy + approver: see the acceptance item under "Mode selection"):
# KIMI_ARGS=(); CODEX_ARGS=(-a on-request -c approvals_reviewer=user); CLAUDE_ARGS=(--permission-mode default --settings "${KIMI_SKILL_DIR}/claude-exec-deny.json")
# An empty array "${KIMI_ARGS[@]}" expands to nothing (safe; it does not become an empty-string argument)
```

The failure branches are not "keep the permissive parameters as before and just write one line on the taskboard":

- Gate 1 fails → **banning push on the execution line must be enforced by capability**: the execution instance does not hold push credentials, or the remote grants only restricted write permission on the line's branch; writing "push banned" on the taskboard alone does not count;
- Gate 2 fails → **do not issue the bypass/auto startup commands**; switch to the attended parameter set (claude **explicitly** `--permission-mode default`, codex **explicitly** `-a on-request -c approvals_reviewer=user`);
- Gate 3 fails (non-git / no restricted permission scope) → same as above, fall back to attended mode with a human as the backstop.

**The attended set is "explicit manual mode + acceptance", not "omitting the flags":**

- From claude ≥2.1.283, not passing bypass may still inherit auto or the user's configured default mode — so manual approval must be chosen explicitly; after startup, **verify the real mode**: `herdr pane read "$P2"` to look at the status line, or have claude run `/status` to confirm it is currently manual approval (not bypass and not auto), with the evidence recorded in `mode-selection.md`; **if manual mode cannot be verified = the gate failed, stop dispatching**, troubleshoot first or stop and start over — never dispatch work under an unknown mode;
- codex **must be given the approval policy and approver explicitly**: `-a on-request -c approvals_reviewer=user` — omitting the flag inherits this machine's user config, which is not the same as manual approval (on this machine, the user-level `.codex/config.toml` was measured to contain `approval_policy = "never"`: it never asks the user for approval; the approver is decided separately by `approvals_reviewer`, and only `user` asks a person); **on-request does not mean every command asks a person** — it means "it asks when the model actively requests it"; to have every batch of actions released by a person first, you need a separate dispatch gate (the orchestrator waits for the user's nod per task before sending); after startup, use `herdr pane read` to check the approval policy and approver shown in the status line / session header, record the evidence in `/tmp/$TAG/mode-selection.md`, and if it cannot be verified, stop and do not dispatch;
- **Existing sessions are not subject to startup parameters**: the permission mode is a process-level property — changing a shell variable or a taskboard entry does not change an already running process. If a gate fails and the target agent is already running (especially already inside bypass/auto), **it must be stopped first (exit/kill) and relaunched in manual mode**, and dispatching resumes only after the mode is re-verified; "I changed a variable, so consider it downgraded" is not allowed;
- Closing off the kimi-orch exception clause (the exception in the last item of Step 0, "the current session acts as orchestrator"): `--auto` means **Never Ask** — it does not interrupt the user, executes automatically and decides automatically; the user sitting at the terminal = attended, but **being present ≠ user approval**, and rename only registers an identity — it neither changes the CLI's permission mode nor makes the CLI wait for a person's nod before acting. If the current session is already running in `--auto` and cannot pass the three gates, only two choices are allowed: **(a) documented narrowing (not equivalent to manual approval)**: the user writes the authorization scope explicitly into `/tmp/$TAG/mode-selection.md` (which operations may be executed automatically, and how supervision and loss-cutting work); that branch executes as "**authorized automatic execution + human supervision**" and is labelled truthfully, no longer claiming to be equivalent to manual approval; **(b) without that record → stop work**. If you want actions to truly wait for a person's approval beforehand, you need a separate external dispatch/execution approval gate (this skill has no such built-in layer).

Create a separate tab and launch three **brand-new clean** agents in the fixed layout (do not resume from old sessions). **Create each line's worktree in the main tree first, then create the panes and start the agents with the corresponding cwd** (a worktree isolates only working files and the index, not git metadata, and is not a security boundary — see "Known limitations"; when the project directory is **non-git**, fall back to launching with all three in the main tree, the isolation guarantee drops, **and unattended mode automatically falls back to attended mode** — see enablement gate 3 of "Unattended mode", and note it on the taskboard):

```bash
# 0) The two worktrees: one for the review line and one for the execution line (skip if they already exist); branch names carry the tag and never overlap.
#    Execute strictly per the path plan fixed in the "first-run flow block": the non-git fallback (main-only) skips this, and all three cwds use the main tree
if [ "$PATH_PLAN" = worktree ]; then
  git worktree add "$PWD/../wt-$TAG-review" -b "trio-$TAG-review"
  git worktree add "$PWD/../wt-$TAG-exec"   -b "trio-$TAG-exec"
fi

# 1) New tab: the kimi slot stays in the main tree; the root pane P0 is parsed from the returned JSON
P0=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$EXPECT_ORCH" --label <task name> --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["root_pane"]["pane_id"])')
# 2) Split P0 to the right → P1 (review slot); split P1 downward → P2 (execution slot) — cwd uses the expected values from the manifest (= the main tree in the non-git fallback)
P1=$(herdr pane split "$P0" --direction right --cwd "$EXPECT_REVIEW" --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
P2=$(herdr pane split "$P1" --direction down --cwd "$EXPECT_EXEC" --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')

# 3) Start the agents: the parameters come from the arrays of "Mode selection"; hard-coding bypass/auto is a process violation
herdr agent start "kimi-orch-$TAG"    --kind kimi   --pane "$P0" -- "${KIMI_ARGS[@]}"
herdr agent start "codex-review-$TAG" --kind codex  --pane "$P1" -- -m gpt-6.1-sol -c model_reasoning_effort=high "${CODEX_ARGS[@]}"
herdr agent start "claude-exec-$TAG"  --kind claude --pane "$P2" -- "${CLAUDE_ARGS[@]}"
# Each array element becomes its own argv, so paths containing spaces are not split and an empty array produces no arguments; do not fall back to string concatenation (see the measurements under "Mode selection")
# The model slug and the reasoning effort must be passed separately: the slug carries no effort, and the effort uses its own config key; `--settings` builds its path from the skill-directory variable and never hard-codes a user name (expanded on this machine = ~/.kimi-code/skills/trio-orchestrate), so it follows the install location after sharing

# 4) Rename all three panes (mandatory; skipping it leaves the agent name and the display label inconsistent); a pane label and an agent name are two objects
herdr pane rename "$P0" kimi-orch-$TAG
herdr pane rename "$P1" codex-review-$TAG
herdr pane rename "$P2" claude-exec-$TAG

# 5) bootstrap: the task directory / manifest / lock were already handled in the "detection" step → create the taskboard skeleton (only if missing; reentry does not overwrite) →
#    take this launch's batch id (bootstrap ID) → wait for the ACK to be exactly equal to the expected text (cmp) — that is the criterion for "launch complete"
TD="/tmp/$TAG"                             # the same task directory as in the "first-run flow block"; reset here so this block can be extracted and self-tested
mkdir -p "$TD"                             # idempotent
SKILL_ABS=<absolute skill path known to the sender>   # e.g. /home/<user>/.kimi-code/skills/trio-orchestrate; always write absolute paths inside the package, and do not assume the other side will expand ${KIMI_SKILL_DIR}
# ══ bootstrap ACK closed-loop block (start): extract it whole, set TD/TAG/SKILL_ABS, and it runs standalone (the other side's herdr may be replaced by a stub) ══
# Seconds + a random tail only lowers the chance of collision; it does not promise concurrent uniqueness; concurrency exclusion relies on launch.lock — one TAG allows only one launch flow at a time
BOOT_ID=$(date +%s)-$RANDOM                # this launch's batch id: old ACKs and leftovers from failures are told apart by it
[ -f "$TD/taskboard.md" ] || cat > "$TD/taskboard.md" <<'EOF'
# taskboard

## Backlog (with acceptance line + conflict surface)
## In progress (claimant + time)
## Pending review
## Accepted (with evidence path)
EOF
rm -f "$TD/bootstrap-ack.md"               # clear the old ACK: only the receipt written by this run counts
cat > "$TD/bootstrap.md" <<EOF
bootstrap-id: $BOOT_ID
skill file: $SKILL_ABS/SKILL.md (absolute path; activate the skill first and read this file, do not rely on the placeholder being expanded automatically)
tag: $TAG
taskboard: $TD/taskboard.md (the skeleton is already created)
role names: kimi-orch-$TAG (orchestrate + accept) / codex-review-$TAG (review) / claude-exec-$TAG (execute, [机核] only)
instruction: after reading the skill and the taskboard, write the ACK to $TD/bootstrap-ack.md. The ACK must be **byte-for-byte identical** to the five lines below (copy them verbatim, one item per line, no extra characters, no trailing spaces, keep exactly one final newline):
bootstrap-id: $BOOT_ID
tag: $TAG
kimi-orch-$TAG
codex-review-$TAG
claude-exec-$TAG
Do not run the launch flow again (do not create a new tab, do not start new agents).
EOF
# The full expected ACK text (byte-for-byte identical to the five lines given to the other side above), accepted by an exact-equality comparison:
# An existence grep cannot stop "two contradictory values for the same field / duplicate lines / extra lines" (ruling 2026-10-09); the whole text must be compared
cat > "$TD/ack-expected" <<EOF
bootstrap-id: $BOOT_ID
tag: $TAG
kimi-orch-$TAG
codex-review-$TAG
claude-exec-$TAG
EOF
herdr agent prompt kimi-orch-$TAG "read $TD/bootstrap.md and carry out its instructions" --wait --timeout 120000
ack="$TD/bootstrap-ack.md"
cmp -s "$TD/ack-expected" "$ack" || { echo "the ACK does not match the expected text (contradiction / duplicate / extra line / missing line): launch not complete"; exit 1; }
rmdir "$TD/launch.lock"                    # ACK passes = launch complete, release the exclusive lock; if it stopped on a mid-way failure, finish the manual handling before deleting the lock and re-entering
# ══ bootstrap ACK closed-loop block (end) ══
```

The layout is always: kimi-orch on the left | codex-review top right | claude-exec bottom right (right first, then down; independent of width or height).

- The model name is whatever appears in codex's `/model` list; **the model slug and the reasoning effort must be passed separately**: `-m <slug> -c model_reasoning_effort=high`. Treating "GPT-6.1-Sol high" as a single model name gets rejected by the API with a 400 (under a ChatGPT account, high is a reasoning effort and not part of the model name — hit in a measurement on 2026-10-09: `-m` reported no error at the time, and it only blew up on the first request); the slug is whatever appears in the local `/model` list, and that model's default reasoning effort may not be high, so it must be set explicitly; if `-m` is rejected, start the default model and then pick the model + reasoning effort manually with `/model` in the pane, and note it on the taskboard — a manual workaround is acceptable, but it does not count as the normal "one-command / unattended" path;
- **Status semantics are unified throughout the whole document** (the same set in Step 0, Step 3 and the restart checkpoint): **able to receive a new task = `idle` or `done`** (herdr's done is equally able to accept input); `working` is never used as a criterion, only as a supporting observation; `blocked` is a troubleshooting branch;
- `agent start` only counts if it returns ready; if it will not start, first `herdr pane read` to see where it is stuck (codex often sticks on an upgrade prompt — press Esc to dismiss it); **when startup fails, first distinguish the two kinds**: a parameter validation failure where the process was never created → you may reuse that shell pane to fix the parameters and resend; the process already exists but is blocked/not-ready → troubleshoot first or wait for it to exit, do not blindly resend;
- If the pane split succeeds but `agent start` fails (e.g. invalid_agent_name, in which case no process was created): the pane does not need to be recreated; fix the parameters and resend `agent start` on the same pane;
- Acceptance: `herdr pane read "$P1" | tail` must show both the `<model> <reasoning effort>` segments in the status line, **and then send an insignificant test message to confirm no 400 pops up** before it truly counts as usable (looking at the displayed model name alone does not count); P2 must show bypass permissions on (unattended parameter set only) — **this only proves the permission mode, not that the deny list is mounted**. List acceptance has two levels: the file existing and the JSON being valid is a **syntax check** (verified in one line with `python3 -c "import json;json.load(open('...'))"`); the **effect check** is the orchestrator running `herdr pane read "$P2"` or having claude run `/status` (`/permissions`) and looking at the **settings sources and the list of effective rules**, pasting the effective rule entries into the taskboard as evidence — a syntax check ≠ an effect check, they are two different things. If no effective rules can be verified / the list is not in effect → **pause dispatching**: stop the already running claude first (exit/kill), relaunch it with the attended set (explicit `--permission-mode default`), and re-verify the mode and the rules after the relaunch; resume dispatching only once verified — merely changing a shell variable or writing one line on the taskboard cannot touch the old process and does not count as a downgrade;
- The completion criterion of step 5 is **the ACK being exactly equal to the full text of `/tmp/$TAG/ack-expected` (`cmp -s`; content = the five lines: bootstrap ID, tag, and the three full role names)** — contradictory fields, duplicate fields and extra lines are all rejected; it is not some status observation, nor an existence grep: `agent get` serves only as a supporting observation — a short bootstrap is usually already idle/done by the time `--wait` returns, so working cannot be waited for (an older version treated working as a hard condition, judged wrong on 2026-10-09); once the ACK is complete, tell the user to switch to the new tab and hand the task to kimi-orch (it is the subsequent orchestrator; this launching session is only responsible for setting things up);
- Exception (**precondition: mode selection has passed; or documented authorization has been made per "Mode selection" — the user has written the scope of authorized automatic execution into `/tmp/$TAG/mode-selection.md`**; `--auto` = Never Ask, presence ≠ user approval): if the user loaded this skill inside some kimi --auto session and wants that very session to act as the orchestrator, then do not create a new tab — **first register the agent identity with `herdr agent rename "$HERDR_PANE_ID" "kimi-orch-$TAG"`** (renaming a pane/tab is only a display label, a different object from the agent name; changing only the label does not register this session under the role name), then take the current pane as P0, split P1/P2 by the worktree/cwd rules above, start the two agents, rename all three panes as usual, **and also `herdr tab rename` the tab to the task name** (when that tab still mixes in other unrelated panes, instead append `<TAG>` after the original name). This exception does not exempt mode selection: if the three gates fail and there is no documented authorization as above, **disable the exception and stop work** (see "Mode selection"); rename only registers the agent identity, it does not change the CLI permission mode of this session and does not constitute user approval.

## Substitute team assembly (when the three-piece set is incomplete)

- **kimi-orch cannot be substituted**: the skill runs inside kimi, but the binary check must still be performed (being able to run in the current session ≠ a kimi that can start a new instance is on PATH); only a missing review/execution agent leads to this section;
- **Availability has two levels of checking, both required**: ① `command -v <cli>` — the binary exists (a CLI on PATH that can start a new instance; being runnable in the current session does not count); ② `herdr integration status` shows `current` (the status hook is installed). If ② is missing you can install it with `herdr integration install <agent>`; if it cannot be installed, downgrade to a spare — it can start and run, but idle/working detection is unreliable, the `agent get` observation of communication protocol step 3 becomes ineffective, and only manual `pane read` watching is left;
- **Candidates must be recorded**: for each candidate, write down the CLI kind + provider/model + the role's capabilities and permission boundary; "different backend" is judged by those two items, not by the binary name alone; substitute order (general priority; on each machine the two-level check decides, the annotations below are examples only):

| Role | First choice | Substitute order |
|---|---|---|
| Review | codex | copilot → opencode (install the hook first) → a second kimi |
| Execute | claude | codex → copilot → opencode (install the hook first) |

- **Hard constraint: review and execution must not share a backend** — same-origin blind spots mean reviewing equals not reviewing. "Different backend" here = **different CLI and different model family** (the same CLI with another model, or different CLIs connected to the same provider/model, both count as same-origin);
- **Cast roles by enumeration, not by greediness**: list the review candidates × execution candidates as pairs and take the **first legal pair satisfying "different CLI and different model family"** (the two roles also cannot be the same live agent). Counter-example: when only codex and kimi are available, the legal solution is **kimi reviews + codex executes**; a greedy rule of "review keeps its first choice, only the execution rank slides down" would wrongly report no solution;
- Candidate health checks are at the same level as for the main backend (binary + integration `current` + account/model availability); if enumeration still yields no legal pair, **report clearly that no team can be assembled and stop — do not force one**;
- Role names do not change: `codex-review-<tag>` / `claude-exec-<tag>` are **slot names, not backend names** (the protocol, the taskboard and communication all stay the same), and the actual backend and kind are written into the taskboard header; `--kind` takes the actual backend (the list herdr supports is whatever `herdr agent start --help` shows);
- Substitute CLIs differ in their no-confirmation parameters (codex `--approve-for-me`, claude `--dangerously-skip-permissions`, others per that CLI's actual `--help`); after launching, use `herdr pane read` to check the permission mode and accept the model/reasoning effort actually in effect, recording them on the taskboard;
- When claude is absent, the `--settings` mount of `claude-exec-deny.json` automatically becomes void — a substitute backend has no equivalent mechanism, so **that line's unattended capability is disabled along with it** (switch to attended, or set up a process-level sandbox separately); you cannot just note the disarmament and keep following unattended discipline.

## Step 1: Set up the taskboard

`/tmp/<TAG>/taskboard.md` (TAG being the task tag fixed in Step 0), four columns: Backlog (with acceptance line + conflict surface) / In progress (claimant + time) / Pending review / Accepted (with evidence path). Write the rules into the taskboard header:

1. **Single writer: taskboard.md is written only by the orchestrator (kimi-orch)**; working agents do not edit the taskboard directly, they only write their results to their own result file under `/tmp/<TAG>/` (e.g. `<task-id>-result.md`), and the orchestrator moves columns by pointer — several parties writing the same Markdown overwrite each other;
2. Claiming writes your name, moving a column carries evidence (test counts / hashes / file paths); words alone are not proof;
3. **Evidence is bound to a code version**: every task carries a task ID and a **base commit**; the receipt must carry the **tested commit** + artefact path + checksum; review and merge check the same revision — once the code changes again (dirty worktree, moved HEAD), the old acceptance is void and must be re-run and re-signed;
4. claude-exec only claims entries marked **[机核]** (acceptance line machine-checkable: test counts / hashes / gate scripts); its written report is not evidence;
5. codex-review pulls the "Pending review" column and writes the verdict to disk under /tmp and sends a pointer;
6. **Commit discipline (unified across all roles)**: each line has its own index; `git add` only by task path, never `git add -A`; check `git diff --cached` before committing to confirm it contains only this task's changes, and never make a bare commit without looking at the staged content; line prefixes match the actual role: `[kimi-orch]`, `[codex-review]`, `[claude-exec]`; renames/moves must be completed atomically together with the commit;
7. The full suite takes the slot serially; multi-line work uses `git worktree` double buffering;
8. Pre-registered numbers (write the expectation down before running) take precedence over after-the-fact explanation.

## Unattended mode (pre-authorization for fully automatic collaboration)

When the user is not present and nobody clicks confirm, modifications/deletions/git operations are **pre-authorized and released**. First distinguish three layers: the **permission mode** (bypass) only saves confirmations, and **isolation** (a worktree per line) only separates working files and the index; neither is **a security boundary** (see "Known limitations"); the real hard boundary is a process-level sandbox/container + restricted credentials + remote branch protection.

**Enablement gates (conditionally on, not on by default) — if any of the three is not satisfied, fall back to attended mode and do not start work with default permissiveness:**

| # | Must be satisfied before enabling | Behaviour when not satisfied |
|---|---|---|
| 1 | Push hard boundary: the remote main/master has **branch protection** set (or pushes use only permission-restricted credentials) | The execution line is **banned from pushing**; changes stay on a local branch and the orchestrator pushes them manually |
| 2 | Credential surface: the runtime environment does not mount non-essential credentials (or isolation via a separate Unix user / container) | Unattended mode is not enabled |
| 3 | The directory is a git repository and a worktree can be created (a non-git directory has no usable isolation line) | **A non-git directory automatically falls back to attended mode** — this is a downgrade path, not an equivalent substitute |

All three are **checks to perform before starting work**, not explanations afterwards; if not verified, treat them as unsatisfied. How to write the item-by-item evidence for the three, and how "all three gates pass → unattended parameter set / any fails → attended parameter set" lands, is in the "**Mode selection**" part of Step 0 — the startup parameters are chosen there, and default permissiveness is not granted here. The attended set you fall back to **must give claude an explicit `--permission-mode default` and verify the real mode**: not passing bypass ≠ manual approval (≥2.1.283 may inherit auto or the user's default mode), and changing variables in an existing session has no effect — it must be stopped and relaunched.

1. **One git worktree per line** (a precondition only once enablement gate 3 is satisfied; the orchestrator creates them in Step 0 before starting the agents): each line has its own branch and its own index, so `commit -a` / `stash` / `reset` are limited to that line's working files — but see "Known limitations" item 3, git metadata is not isolated;
2. Only kimi-orch merges/writes to the main tree; claude-exec/codex-review do not touch the main tree — this is a **disciplinary convention**, and mechanically it cannot be stopped (the execution line was measured to be able to change main-tree refs), so it relies on pre-commit checks and acceptance as a backstop;
3. Freedom to delete and modify is limited to your own worktree and the `/tmp/<TAG>/` task directory; deleting or modifying anything beyond that still requires the user's confirmation — this is the **default authorization scope**; if, per gate 3 of "Mode selection", the user explicitly chose "same-user cross-directory access", then record it here as "the authorization scope the user has chosen" and stop simultaneously claiming that crossing directories always requires confirmation;
4. Pushing is allowed only to each line's branch and is reported along with the taskboard; pushing to main/master and force-pushing are permanently banned — but **the deny list is only a textual prefix soft defense line and can be bypassed by equivalent formulations** (measured forms in "Known limitations" item 1); for a true permanent ban, set branch protection on the remote / use permission-restricted credentials;
5. The deny list keeps only permanent bans (sudo / disk destruction / force-push / push to main / reading secrets) — unattended, a long list only deadlocks batch runs; the real backstop is machine-checkable acceptance + worktree isolation + commit discipline (their boundaries are in "Known limitations").

## Known limitations (promises downgraded, stated honestly)

1. **The permanent push ban is not a hard block**: the `Bash(git push ...)` entries in `claude-exec-deny.json` are only **textual prefix rules**, a soft defense line that equivalent formulations can bypass; the bypass forms are distinguished by method (not lumped together as "all measured") — `git push origin HEAD:main` (the main of a local bare remote was actually rewritten) and `git push origin --force` are **real Git executions**, evidence `/tmp/skill-review/evidence.json`; `git push upstream main`, `+refspec` force-push, the `git -C <dir> push ...` wrapper, and argument-less `git push` (which reads the upstream/refspec configuration) are only **prefix-rule text modelling** (the command string compared against the rule list, with no real Git execution record); none of the above forms **were measured inside the Claude permission engine**. `--force`, `-f` and `--force-with-lease` are three independent prefixes, so the list is incomplete. A true permanent ban can only come from **remote branch protection / restricted credentials / a controlled push entry point**, acting on the parsed target ref and force semantics;
2. **Reading secrets only covers direct reads within cwd**: the `Read(./**/...)` rules do not reach `~/.ssh/` or CLI credential directories, nor do they stop **indirect reads**: a subprocess (Python/Node) opening files itself, or a search that does not explicitly list file names. Dangerous forms enter the deny list by **enumeration** (`.env.local` / `.env.production` / `.env.development` / `.env.staging`, `credentials.json`, `*.key`, `id_rsa`, etc.), so template names (`.env.example` / `.env.sample` / `.env.template`) are naturally never matched — permissions are evaluated deny → ask → allow, **a broad deny overrides a narrow allow, so do not count on a whitelist exemption**; enumeration is inherently incomplete, and an unattended environment **should not mount non-essential credentials**; the read boundary must be enforced by process-level file permissions/a sandbox, and no configuration may claim to have enumerated "all secrets";
3. **A worktree is not a security boundary**: it isolates only working files and the index, while **git metadata (config/refs under the common dir) is shared with the main tree** — it was measured that the execution line's `git config` is read by the main tree, that `git update-ref refs/heads/main` can change the main tree's main ref, and that it can push to a local remote's main. So "only the orchestrator can touch the main tree" is a **disciplinary convention that does not hold mechanically**; for a hard boundary you need a process-level sandbox/container + a separate Unix user + restricted remote credentials, enabled after the user has explicitly chosen the authorization scope;
4. **Substitute backends have no equivalent control**: when claude is absent the `--settings` mount becomes void, and a substitute CLI has no equivalent deny mechanism; that line's unattended capability should then be disabled or switched to attended (see the last item of "Substitute team assembly");
5. **`/tmp` may be cleaned by the system**: the taskboard, ACKs and verdicts all live in `/tmp/<TAG>/`, and a reboot or a cleanup service will delete them; the key manifest, the final verdict and recovery pointers must be stored separately in a persistent location inside the project (see item 1 of "Context/token discipline");
6. **The heartbeat cron belongs to the session**: a scheduled task belongs to the current session and is not inherited by a new session, so it must be recreated (see the last item of Step 3).

Until the hard boundaries corresponding to items 1–3 above (remote protection, credential isolation, worktree availability) have been accepted, handle them per the **enablement gates** of "Unattended mode": fall back to attended mode, and do not grant permissiveness by default.

## Step 2: Dispatch work by layer (by judgement density, not by pane)

| Layer | Content | Dispatched to |
|---|---|---|
| A Judgement-dense (debugging / design / tradeoffs) | kimi subagents (Agent tool, may run several in parallel) |
| B Implementation whose spec is already frozen | kimi subagents, or claude-exec + a strict spec |
| C Mechanical execution (run the suite / regenerate / read large files / reconcile tables) | claude-exec |
| Review | codex-review |

**The criterion for dispatch grading (fixed 2026-10-09)**: if the task book still contains "how should this be designed" decisions (which implementation to choose, how to split it, what threshold to set) = **layer A**, left to kimi subagents — **it must not be marked [机核] and dispatched to claude-exec**; [机核] marks only layer-C work whose method is already frozen and whose acceptance line is machine-checkable (commands, paths and expected values all present, requiring no judgement from the executor). Handing design work down as [机核] equals making the executor take your design decisions for you, and no machine-checkable receipt can make up for that.

Quota discipline: reading large files and running long batches go to layer C. When a subagent reports 403, first **sort by type** — quota wall / authentication failure / network failure: on a quota wall, save progress first and back off waiting, then continue with `Agent(resume=...)` once it recovers, without restarting; **no unlimited retries**; authentication/network are handled per their own cause; if the model/proxy/parameters change, record it as a new execution condition and do not automatically inherit the old acceptance.

## Context/token discipline (context is a cache, the taskboard is the truth)

Automatic compaction only triggers near the ceiling (staying put at 70% is normal), and compaction itself burns quota — **accumulating less + externalizing state + restarting on schedule is more effective than compacting**:

1. **Externalize all state**: task books, verdicts and batch results all go to `/tmp/<TAG>/`; any agent can be killed and restarted, and after a restart it recovers from the **persisted checkpoints** (taskboard + pointer files + ACK/result files). Note that `/tmp` may be cleaned by the system: the key manifest, the final verdict and recovery pointers must be stored separately in a persistent location inside the project;
2. **Persist in both directions**: not only is the dispatch persisted, the agent's **output is persisted too** — a report may only return "conclusion + evidence path + key numbers"; pasting large files/logs back into the conversation is forbidden (paste once and every later round carries it and burns it);
3. **Slim the prompt**: the pane receives only a short pointer + the explicit output format (conclusion first, a few lines at most); no matter how long the task book is, it is sent as a path;
4. **Layer C is short-lived**: claude-exec is a **stateless executor** — once a batch of machine-checkable work is done, restart it (after `/exit`, relaunch with `agent start`); do not cling to context; its memory is on the taskboard, not in the session;
5. **Restart the review layer as needed**: codex-review's verdict history must live in a verdict file under `/tmp/<TAG>/`, and a restart recovers from the persisted checkpoints; restart when crossing tasks or after a long backlog;
6. **Threshold checks + restart state machine**: thresholds follow each CLI's current configuration (do not write them as universal facts for all backends). The orchestrator's heartbeat `pane read` watches the status line; restarts happen only at safe checkpoints: confirm the agent is in an **able-to-receive state (idle or done)** and that artefacts/uncommitted state are on disk (do not send `/compact` while working, do not interrupt a task) → send `/compact` (supported by both claude and codex), and restart only if that has no effect. Restart sequence: `/exit` → `pane read` to confirm **it is back at the shell prompt** (this only shows the screen returned to the shell, **it does not mean the registered state has been released**) → `herdr agent list` to check the old name has disappeared from the registry → then `agent start`; if the process exists but is blocked/not-ready, troubleshoot first or wait for it to exit, and do not blindly resend.

## Quota exhaustion contingency (quota is a schedulable resource, not an incident)

Running out of quota (the 5h window / daily quota) is, like a network disconnect, a **normal external event**, and the principle is to absorb it rather than tough it out:

1. **Detection goes into the heartbeat**: the orchestrator uses `pane read` to catch quota signals (codex status line `5h limit: N% left · resets at HH:MM` — **this is an example from the author's environment, not a detection contract**: status-line items are configurable and versions/authentication methods differ, so the truth is whatever `/status` measures; if it is not shown or cannot be obtained, mark it **unknown** and do not treat it as sufficient); a kimi 403 is only a candidate signal, so sort it per the "Quota discipline" item into "quota / authentication / network" before entering the handling chain. The same account may be subject to several limits at once (e.g. a 5h window + a weekly limit), so observe them separately. **Below 30%, record it on the taskboard and tighten dispatching** (dispatch only critical tasks; pause critical acceptance — this is our own scheduling policy, not a vendor-guaranteed safety threshold);
2. **Quota-pool independence must be verified, and cannot be inferred from the CLI name**: different CLIs may sit on the same billing account (when Claude uses an API key and Copilot uses BYOK, the two CLIs burn the same pool). Before substituting, check **provider + authentication method + billing account/organization + quota bucket and window** item by item, and only call it quota failover once the candidate is confirmed not to be constrained by the exhausted bucket; changing the client alone does not count. The verification record contains no keys;
3. **Swapping brains to recover presupposes externalized state**: with the taskboard + manifest + task ID on disk, a substitute recovers from the **last confirmed checkpoint** — note that the interrupted scene may be "executed but not yet reported", so before taking over, check the artefacts/processes/commits and confirm the old executor has stopped writing; register any necessary re-run on the taskboard. The recovery promise is limited to "not repeating work already verified as done", and does not promise that any interruption is lossless;
4. **The contingency chain is per role**: review drops → ① wait for the reset window ② backup review (borrowed, after notifying its orchestrator first) ③ substitute review (labelled as downgraded, **does not serve as the final release signature**); execution drops → swap the backend per substitute team assembly (the constraint that review/execution use different backends still applies);
5. **The quality tier is not lowered to save quota**: final review and critical gates keep their established reasoning effort; do not run critical acceptance on the last train of the quota — better to wait for the reset. (The author's single observation on 2026-10-09: at medium effort and with low remaining quota, a deterministic bug was still caught — the evidence for the bug itself is /tmp/skill-review/verdict-r5.md; the effort tier and the remaining-quota number are observations of the orchestrator's status line via `pane read` at the time, **not archived and not independently verified, labelled as such**; this is a single observation, not proof that medium is generally sufficient; low was never tested, and is not used for final review);
6. **Quota goes on the taskboard**: before dispatching a large task, record the current remaining amount, the quota bucket/window and the reset time (this is pre-registration); the consumption trend afterwards is review data, and the two are recorded separately.

## Step 3: Communication protocol (criteria for delivery and re-delivery)

- **Every task carries a task ID** (T001/T002…, consistent with the taskboard entry); before sending, confirm the other side is in an **able-to-receive state: `idle` or `done`** (done can equally accept input); do not send while `working` (that is the previous task running), and troubleshoot `blocked` first;
- Prefer sending with `herdr agent prompt <name> "..." --wait --timeout <ms>` (with a finite timeout); check the returned blocked/stalled/timeout;
- **The criterion for acceptance/completion is the ACK/result file on disk, and it must be bound to the task ID + the current batch** (batch id/bootstrap ID); do not rely on a single status observation: `working` is only a **supporting observation** — a short task may finish before `agent get` ever sees it (working is simply never observed); and if the other side was already working, seeing working does not prove the new task has started;
- Before re-delivering, first check the ACK and the artefacts (result file, process, commit); **the task must be idempotent** (safe to re-run), so that a re-delivery after a timeout does not become a duplicate execution;
- Long bodies/task books **go to disk under /tmp/<TAG>/ and only the path is sent**; the pane receives only a short pointer (the pane channel was measured to lose long text);
- The codex backend occasionally reports connection failed: first troubleshoot from your own network egress (see the appendix for the author's original environment); once it is confirmed to be a network problem and the previous task truly was not executed, re-deliver per the re-delivery flow above; if it is still unusable → **backup review** (borrow the review backend from the other line, notifying its orchestrator first, and return it as soon as it is recalled);
- Long tasks get a heartbeat cron (avoiding :00/:30), deleted once the loop closes; **a cron belongs to the current session, and a new session must recreate it**.

## Step 4: Acceptance culture

- A fix-type change must have **two pieces of evidence: red before, green after**;
- The orchestrator personally re-checks the key numbers (re-run/re-compute) rather than relaying them;
- "Should be identical" ≠ "measured identical"; static declarations and measurements are written separately;
- **An acceptance gate must include behavioural probes**: for executable things such as validation predicates, ACK comparisons and startup parameter sets, the gate must **construct adversarial samples and actually run them** — contradictory fields, empty values, duplicate lines, wrong paths, lock contention, leftover directories, each class needs a sample that can be judged red/green; grepping a string does not count as acceptance (whether a string is present and whether the predicate judges correctly are two different things — the first four rounds of gates on 2026-10-09 were all green, and they went red the moment behavioural samples ran). The adversarial sample sets are in `/tmp/skill-review/` and `/tmp/skillfix/r*-evidence/`, and can be copied;
- **The gate script itself must also be verified**: after writing a gate, first prove it can go red and green; when the pattern string starts with `-` you must add a `--` separator such as `grep -qF -- "$3"`, since misplaced arguments make the gate silently mis-check (the orchestrator made this mistake twice on 2026-10-09); after changing a gate, replay at least one sample on which "the old implementation must go red";
- Statements of your own that were overturned get an honest correction record.

## Step 5: Freezing and parallelism

- **Freeze the repository** during a large computation batch (each fold reading the live tree with a new process); freezing/unfreezing is announced by the orchestrator;
- Realistic data scale exposes defects a demo cannot show (soft scoping, outer joins, type chunking) — the acceptance evidence needs at least one copy at real data scale.

## Launch message template

> Task: <one sentence>. I (kimi-orch) orchestrate + accept; codex-review (high reasoning effort) and claude-exec (limited to [机核]) have been created.
> Taskboard: /tmp/<TAG>/taskboard.md. Discipline: commit with line prefix, add by path, full suite serially, persist to disk and send pointers.
> Dispatches carry a task ID and base commit; receipts are the ACK/result files on disk, and `working` counts only as a supporting observation.

## Pre-release smoke checklist

Go through it item by item before sharing/releasing. Evidence has four tiers; the vague [measured] is retired: **[real run]** = the command/script was actually executed, with input and result paths kept; **[prefix modelling]** = only a textual comparison of the command string against the rule list/predicate, with no real execution; **[static reasoning]** = manual reasoning or a one-off enumeration, not frozen into a re-runnable record; **[not measured]** = not yet run, only required by the rules. Protocol version: **2026-10-09-r5**.

| # | Check | Status and evidence |
|---|---|---|
| 1 | Parameter validation: empty / over-long / illegal characters / **multi-line injection** are all rejected; and **the model gate precedes any command containing TAG** | **[real run]** 2026-10-09: the bash `[[ =~ ]]` whole-string check was run case by case over 8 examples, 1 allowed and 7 rejected (evidence `/tmp/skillfix/r3-evidence/tag-validation.txt`); the item "the expanded value is first judged by reading it in the model" itself is [not measured] (it requires actually invoking the Kimi template engine with an illegal $tag) |
| 2 | Reentry with the same tag: the three names match exactly + same tab + **each role's cwd == the expected worktree (compared after `os.path.realpath()` normalization on both sides, so a path built with `..` and the canonical path pointing at the same directory must compare equal)**; a missing `tab_id`/`workspace_id` is fail-closed | **[real run]** 2026-10-09: 6 samples — **new positive case** where the expectation carried `..` while the actual cwd was the canonical path → 0; same tab with all cwds correct → 0; all cwds wrong → 3; missing topology field → 3; cross-tab → 3; missing role → 2 (evidence `/tmp/skillfix/r4-evidence/reuse-realpath.txt`; the r3 topology 5 samples in `/tmp/skillfix/r3-evidence/reuse-topology.txt`); **backfill and rollback on failure** [not measured] |
| 3 | Partial team-assembly failure: if no process was created, resend on the same pane; if the process exists but is blocked/not-ready, troubleshoot first | [not measured] |
| 4 | No substitute solution: report no solution only after enumerating all (review, execution) candidate pairs | **[real run]** 2026-10-09 script: with review candidates = [codex,kimi] and execution candidates = [codex], the greedy rule wrongly reported no solution while enumeration found kimi reviewing + codex executing (script and output: `/tmp/skillfix/r3-evidence/role-pair-enum.txt`); a live drill [not measured] |
| 5 | deny negative case (B1): `git push origin HEAD:main`, `git push origin --force` | **[real run]** a temporary Git command was really run: the local bare remote's main was actually rewritten (evidence `/tmp/skill-review/evidence.json`) |
| 5b | deny negative case (B1), remaining forms: `+refspec`, the `git -C` wrapper, `upstream main`, argument-less `push` | **[prefix modelling]** a textual comparison of the command string against the rule list, with no real Git execution record; the end-to-end negative case inside the permission engine is [not measured] |
| 6 | deny negative case (N1): the template name `.env.example` is not matched by any deny rule (enumerate dangerous forms, do not rely on a whitelist) | **[prefix modelling]** rule-intersection comparison (fnmatch, 2026-10-09); the final decision inside the engine is [not measured] |
| 7 | Restart recovery: `/exit` → prompt → `herdr agent list` to check the old name has been released → relaunch | [not measured] |
| 8 | Model parameters: `-m <slug> -c model_reasoning_effort=high` takes effect (both segments in the status line + a test message that does not pop a 400) | Effective after the fix, to be verified [not measured]; the pitfall (treating the whole string as a model name → 400) **[real run]** 2026-10-09 |
| 9 | bootstrap: only when the ACK is **exactly equal to the full text of `/tmp/<TAG>/ack-expected` (`cmp -s`)** does "launch complete" count — contradictory fields, duplicate fields, extra lines, missing lines and trailing spaces are all rejected; task identity comes from the manifest (absolute project root + each role's expected cwd + creation time + bootstrap ID), and reentry does a **unique parse**: each required field exactly one line with a non-empty value, a field name appearing twice rejected; before reading or writing the task directory/files, check that they are not symlinks (`[ -L ]`) and that the owner is yourself; one TAG allows only one launch flow at a time (launch.lock) | **[real run]** the ACK 7 samples are unchanged from r4 (evidence `/tmp/skillfix/r4-evidence/cmp-ack.txt`); **r5 re-tested with the new predicates**: manifest unique parse 8 samples + directory safety 5 samples (evidence `/tmp/skillfix/r5-evidence/manifest-unique-parse.txt`, `/tmp/skillfix/r5-evidence/dir-guard.txt`; the old r4 manifest 4 samples are superseded by the new evidence); a live launch ACK [not measured] |
| 10 | Dispatch status semantics: the `done` state can receive tasks; the ACK is bound to the task ID + the current batch; `working` is not a criterion | [not measured] |
| 11 | Mode selection: the three gates' evidence goes to `/tmp/<TAG>/mode-selection.md` → all pass uses the unattended parameter set, any failure uses the attended parameter set (no bypass/auto is sent to the execution line; **when attended, claude gets an explicit `--permission-mode default` and codex an explicit `-a on-request -c approvals_reviewer=user`, neither accepting inheritance of this machine's unaudited defaults**) | **[static reasoning]** the two parameter sets being chosen by gate result is manual reasoning; launching once in each mode on a live machine is [not measured] |
| 12 | claude's effective rules, the **effect check**: `/status` (`/permissions`) to see the settings sources and the effective rules, with the entries pasted onto the taskboard; if it cannot be verified → fall back to attended and **stop first, then relaunch in explicit manual mode and re-verify**; after startup verify the real mode (`pane read` / `/status`), and if manual mode cannot be verified = the gate failed, stop dispatching | [not measured] (the rule-intersection comparison in row 6 is [prefix modelling], which is not the same as being in effect inside the engine) |
| 13 | The parameter sets are bash arrays and `"${ARR[@]}"` makes each element its own argv: a path containing a space is not split, and the Zsh scalar non-splitting problem does not reappear; an empty parameter set `=()` expands to nothing | **[real run]** 2026-10-09: bash 5.2.21 and zsh 5.9 × two modes × two paths (including `/tmp/skill space`), 8 samples in all — the array form is item-by-item correct on both sides; the old scalar string was split into two items at the space path in bash and became one single argument in zsh (evidence `/tmp/skillfix/r4-evidence/argv-arrays.txt`) |
| 14 | Attended mode lands: dispatching resumes only after an explicit `--permission-mode default` **and** the real manual mode verified after startup; codex gets an explicit approval policy/approver with `pane read` confirming what is displayed ( `on-request` only guarantees "it asks when requested", not that every command asks a person); existing sessions are not subject to startup parameters and must be stopped before relaunching; the kimi-orch rename exception is narrowed to "mode acceptance passed, or the user has persisted the authorization scope to `mode-selection.md`" (`--auto` = Never Ask, presence ≠ user approval) | [not measured] (documentation basis: since claude `v2.1.283`, not passing bypass may still inherit auto or the user's `defaultMode`; this machine's codex user config `approval_policy = "never"` is a read-only narrow read) |
| 15 | **Classify before taking the lock** (N3): the complete chain empty directory → classified "new" → take the lock → create the manifest → ACK exactly equal → release; a leftover directory (non-empty, no manifest) is rejected before the lock is taken; a concurrent lock grab for the same TAG is rejected; deleting the lock to turn things green is never allowed | **[real run]** 2026-10-09: the whole "first-run flow block" was extracted from the SKILL and really run (real filesystem; the other side's ACK was a stub, no agents started), 4 branches + an ordering probe for the gates (classification line number < lock-taking line number), evidence `/tmp/skillfix/r5-evidence/lock-flow.txt`; launching real agents [not measured] |

## Appendix: the original author's experience (not universal facts; verify in your own environment)

- Quota: kimi/codex quota is fairly tight, so reading large files and running long batches should go to layer C as far as possible;
- Backend choice: each person decides what backend claude-exec mounts, and backends with unstable output get only machine-checkable work;
- 2026-10-08 shared-index incident: `git add -A` swept away changes another line had staged;
- Measured pitfalls on 2026-10-09: an over-long tag reported `invalid_agent_name`; an earlier silent truncation made two tags collide (since removed); `-m "GPT-6.1-Sol high"` treated the whole string as a model name, and `-m` reported no error, blowing up with a 400 only on the first request;
- The pane channel loses long text: in the author's environment the same message was lost four times in one day, so everything is persisted to disk and sent as a pointer;
- Network: in the author's environment codex goes through a VPN while kimi/deepseek do not — so when only codex drops, check the VPN first; each person troubleshoots by analogy with their own egress situation.
