---
name: trio-orchestrate
description: "One-command launch and orchestration of a parameterized team of agents (default three-role setup: kimi orchestrates + accepts / codex reviews / claude executes; the team is declared by a roster and may have 2~N members, with genox among the optional members). Use when the user wants to start a new multi-agent task or says 'set up three-way collaboration / let's start / launch review and execution / put up the taskboard'. Takes parameters: /skill:trio-orchestrate <task tag> <roster>; the first parameter is the task tag, the second is the team declaration. Depends on herdr (terminal multiplexer) to create panes, start agents, relay messages and watch status. The user only starts kimi and loads this skill; the remaining panes are built automatically."
arguments:
  - tag
  - roster
---

# Parameterized team orchestration: one-command launch + orchestration (default roster = kimi orchestrates · codex reviews · claude executes)

> Author: sheng luan
> Protocol version: **2026-10-10-r6**
> Depends on: kimi CLI + herdr; if another CLI declared in the roster is missing, go to "Team assembly and substitutes". How to share: copy the whole `trio-orchestrate/` directory (including `claude-exec-deny.json`) into the other person's `~/.kimi-code/skills/`.
> This skill is both a **launch protocol** and a **running protocol**. After sharing it, if you worry that ordinary conversation might accidentally trigger tab creation or mode switching, add `disableModelInvocation: true` to the frontmatter as needed, so it can only be invoked by an explicit command.

## Slots and roster declaration (roles are determined by the roster parameter)

### Slot vocabulary (closed, enumerable, checkable)

| Slot | Capability set | Notes |
|---|---|---|
| `orch` | {orch} | Orchestration + acceptance, **exactly one per roster, hard constraint** |
| `rev` | {rev} | Review |
| `rev2` | {rev} | Second review line (extension) |
| `revtest` | {rev, exec} | Review + test composite slot (extension) |
| `exec` | {exec} | Execution (layer C) |
| `exec2`…`exec9` | {exec} | Additional execution lines |

**Rules**:

1. **Coverage constraint**: every roster must have orch ≥1, rev ≥1, exec ≥1; member count **N ≥ 2** (one person is not a team). A missing capability is a rejection, and **the error names which capability is missing**.
2. **Exactly one orchestrator** (hard constraint). Every reason is a direct corollary of existing discipline, and relaxing any of them tears the protocol: the single-writer taskboard rule requires exactly one writer, so two orch = two writers; ownership of launch.lock, creation/unique-parsing of the manifest and the classification verdict each need one unique authority, and two orch members would produce two classification verdicts; mode selection is the three-gate verdict of a "unique decision-maker", so two orch members would produce two parameter-set verdicts; the layer-A verdict and dispatch right, and the ACK collection and acceptance right, all anchor on a single orchestrator. **There is no "multiple orchestrators exception"**: the existing exception "the current session acts as orchestrator" is only an exception about where the orch **instance comes from** (do not create a new tab), not an exception about the number of orch members, and r6 keeps that reading.
3. **Composite slot revtest and the invalidity of self-review**: "review + test" may be carried by one member, but that member's **review of work it produced itself does not count** — for such tickets the review signature must come from a rev-capability member in a different slot, or be escalated to the orch for a final verdict (annotated "self-review invalid, orch final verdict"). The orch already holds final-verdict authority (orchestration + acceptance); this clause only spells out "self-review is invalid".

### The roster DSL

Entry point: `/skill:trio-orchestrate <tag> <roster>`. The roster is a **single-string DSL**, with members joined by `+` (not spaces, avoiding any ambiguity in parameter tokenization), each member being:

```
<slot>=<cli>[:<model>[:<effort>[:<family>]]]
```

- `slot`: slot ID, taken from the closed vocabulary above;
- `cli`: backend CLI name (kimi/codex/claude/copilot/opencode/genox…), i.e. the herdr `--kind` or the backend-profile name;
- `model`: model slug, optional (may be omitted for kimi/claude to use the default);
- `effort`: reasoning level (high/medium/low), optional; the existing convention of **passing it separately from model** is kept;
- `family`: explicit model-family declaration, optional, **default = the `model` string itself** (used for the same-origin check below).

**Default declaration**: an empty roster → the default three-member team

```
orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude
```

This is the capability-equivalent of the existing three-piece setup and **guarantees that old invocation styles behave unchanged**.

**Complete examples**:

2 members (the smallest team with orchestration + review/execution combined; suited to light tasks):
```
/skill:trio-orchestrate hotfix orch=kimi+revtest=codex:gpt-6.1-sol:high
```
Expanded: orch=kimi (main tree) | revtest=codex gpt-6.1-sol high (its own worktree; capabilities={rev,exec}, its own output cannot be self-reviewed, the final verdict goes to the orch)

3 members (the roster spelling of the current default):
```
/skill:trio-orchestrate genox orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude
```

4 members (two execution lines):
```
/skill:trio-orchestrate bench4 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude+exec2=genox
```
Expanded: orch=kimi | rev=codex gpt-6.1-sol high | exec=claude | exec2=genox (the genox slot's parameter set and limits are in "Backend profiles")

### The pairwise constraint (declarative)

- **Same-origin definition** (generalizing the existing "different backends"): members i and j are same-origin ⇔ `cli_i == cli_j` **or** `family_i == family_j` (family defaults to the `model` string). That is the equivalent form of the existing "different backends = different CLI and different model family", with unchanged semantics.
- **Constraint**: ∀(i ∈ rev-capability members, j ∈ exec-capability members, i ≠ j): i and j **must not be same-origin**. The slot dimension is taken **by capability** — a composite slot like revtest participates in the rev set with its rev capability and in the exec set with its exec capability; i=j (self-pairing) does not enter here and is covered by the "self-review invalid" clause above.
- **Conflict error wording (frozen, so that layers B/C/D speak with one voice)**:

  > 组队冲突:槽位 <slotA>(<cliA>/<modelA>) 与槽位 <slotB>(<cliB>/<modelB>) 同源(共享 <CLI 名|模型族 X>):审了等于没审。请更换其中一端的 CLI 或模型族,或在声明中给两端标注不同 family。
  >
  > (Team conflict: slot <slotA> (<cliA>/<modelA>) and slot <slotB> (<cliB>/<modelB>) are same-origin (sharing <CLI name | model family X>): a review by one of the other is no review at all. Change the CLI or model family on one side, or annotate the two sides with different `family` values in the declaration.)

### Naming rules (the key to many coexisting tabs)

The actual agent name is **`<slot>-<task tag>`**, e.g. `orch-g07tune`. Why: herdr's **agent names are a global registry, unique across tabs and workspaces** (a clash is observed as `agent_name_taken`; a pane label is only a display label and carries no identity); the legal character set is `[a-z0-9_-]` only, ≤32. Members of the same tab share the tag; **the pane label stays identical to the agent name**.

**Length**: check `len(slot)+1+len(tag) ≤ 32` **per slot** against the actual roster, and that the full name matches `^[a-z0-9_-]{1,32}$`; additionally give the static ceiling for the default vocabulary: the longest slot name is `revtest` (7 chars) → **tag ≤ 24**
(tag ceiling = 32 (agent-name ceiling) − 1 (hyphen) − 7 (longest slot name `revtest`); **over-length is always rejected, never silently truncated** — truncation makes two tags collide, measured 2026-10-09).

**The reuse check must exactly match the N full agent names against the JSON**, never a substring match like `grep -- "-<tag>"`: `herdr agent list` returns compact single-line JSON, and grep would take `g070` for `g07` and count some other task's whole line (measured 2026-10-09). See "Step 0" for the example. Members in other tabs do not count, and reuse across tasks never happens.

## Step 0: bring up the panes (a fresh pane group, in its own tab)

> The code block below is a **conceptual flow**, not a script to paste wholesale: execute it step by step, **check the return code at every step**, and stop immediately on any prerequisite failure (fail-fast). Always parse pane IDs out of the returned JSON, never copy them by hand; see the examples below. Before first use, do one manual end-to-end run: each CLI's login, repo trust confirmation and the first bypass-mode confirmation must not be left to an unattended window. **Write code-block syntax as Bash throughout** (parameter sets as arrays, `"${ARR[@]}"` expanding to one argv each; Zsh does not word-split an unquoted scalar and behaves differently, so do not copy them there directly).

```bash
command -v herdr >/dev/null || { echo "herdr not installed, stop and tell the user"; exit 1; }
[ "${HERDR_ENV:-}" = 1 ] || { echo "not running inside herdr, stop and tell the user"; exit 1; }
herdr integration status           # is each agent's status hook installed: binary present but hook missing → status detection unreliable

# ── Model gate (the first gate, ahead of any command containing $tag/$roster; executed by the orchestrator = the model) ──
# $tag and $roster were already textually expanded into this document when the skill loaded — those two assignment lines below
# are themselves shell source: a check written after the assignment cannot stop text that entered the source *before* it.
# The real first gate is the reading judgement: after reading the expanded values, first verify that the whole TAG string matches
# ^[a-z0-9_-]{1,24}$ and the whole ROSTER string matches the DSL regex below (an empty member / empty segment / :: / uppercase /
# space / newline is all illegal); if illegal, stop and ask the user for a new one — do not type any line below, and do not write
# the illegal value into any command. This is a model gate, not a shell gate; "assign first, validate later" is not allowed.
# If the roster did not arrive (the engine may return only the first argument; behaviour [not tested]) → degrade per R2: ask the
# user face to face for the roster declaration, and put the string through the same model gate + shell validation before
# continuing; both sources are treated alike, and the protocol does not fork.
TAG=$tag                           # invocation parameter: /skill:trio-orchestrate <tag> <roster> (the skill: prefix is required); if empty, wait for the user to name it when assigning the task. Fixed for the whole run, and used both as the tab label and as the taskboard directory /tmp/<TAG>/
ROSTER=$roster                     # second parameter; if empty → use the default declaration (see "Slots and roster declaration")
[ -n "$ROSTER" ] || ROSTER='orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude'
# Second line of defence (shell layer, stopping continued use after the assignment; it cannot replace the model gate above):
# whole-string validation (measured 2026-10-09: $'g07\n../outside' passes the old grep -qE line anchoring; bash [[ =~ ]] on the
# whole string rejects it)
[[ "$TAG" =~ ^[a-z0-9_-]{1,24}$ ]] || { echo "TAG illegal (empty / chars outside [a-z0-9_-] / longer than 24 / contains a newline), reject and ask the user for a new one"; exit 1; }
[[ "$ROSTER" =~ ^[a-z0-9]+=[a-z0-9_.-]+(:[a-z0-9_.-]+){0,3}(\+[a-z0-9]+=[a-z0-9_.-]+(:[a-z0-9_.-]+){0,3})*$ ]] \
  || { echo "ROSTER illegal (empty member / empty segment / :: / uppercase / space / newline), reject and ask the user for a new one"; exit 1; }
# The regex rejects empty members (two `+` in a row), empty segments (nothing after `=`), `::`, uppercase, spaces and newlines; at most 4 segments per member (slot=cli:model:effort:family)

# ── roster structural validation (after parsing, one python step, stop on failure) ──
# In order: ① slot name ∈ vocabulary, no duplicates within one roster, ≤5 segments and no empty segment; ② N ≥ 2; ③ capability coverage orch≥1/rev≥1/exec≥1;
# ④ exactly one orchestrator (orch exactly 1) **and its CLI must be kimi** (the skill itself runs inside kimi; a hookless backend such as genox must never hold orch);
# ⑤ pairwise constraint enumeration (rev capability × exec capability, not same-origin); ⑥ every member's full name matches ^[a-z0-9_-]{1,32}$ and is ≤32;
# ⑦ cli ∈ this machine's backend-profile table (see "Backend profiles"; an unknown CLI → "no profile, add a backend profile before launching");
# ⑧ a declared model/effort must have a matching launch flag in that CLI's profile (a field with no flag is refused **by name**, never silently dropped — see "Backend profiles", the flag-composition table).
# On success it prints the normalized roster string (slot order fixed: orch always first, the rest in declaration order), which is
# written into the manifest's roster: line and into the bootstrap, so the expected set can be rebuilt on re-entry.
# This is the **single** copy of the roster validation/normalizing source (the DSL below): besides this run's argument, the
# roster: line read back from the manifest on re-entry is re-validated by the same source and compared by full value
# (backend/model/effort/family), so the two sides' rules cannot drift apart.
ROSTER_DSL=$(cat <<'PY'
import re, sys
CAPS = {"orch": {"orch"}, "rev": {"rev"}, "rev2": {"rev"}, "revtest": {"rev", "exec"}, "exec": {"exec"}}
for _i in range(2, 10):
    CAPS[f"exec{_i}"] = {"exec"}                       # exec2…exec9 十个扩展执行线
KNOWN_CLI = {"kimi", "codex", "claude", "copilot", "opencode", "genox"}   # 本机后端档案表;扩充档案时同步这里
CLI_FIELDS = {                                          # 各 CLI 档案里**有**对应启动 flag 的字段(实测,见"后端档案"参数合成表):
    "kimi": {"model"},                                  # 没列出的字段=该后端不支持,声明即拒,不静默丢弃
    "codex": {"model", "effort"},                       # codex: -m <model> -c model_reasoning_effort=<effort>
    "claude": {"model", "effort"},                      # claude: --model <model> --effort <level>
}
roster, tag = sys.argv[1], sys.argv[2]
full = re.compile(r"^[a-z0-9_-]{1,32}$")
members = []
for part in roster.split("+"):
    seg = part.replace("=", ":", 1).split(":")      # 先拆一次 `=`(seg[0] 里装的是 `slot=cli` 整串,不是 slot),再按 `:` 分段
    if len(seg) > 5:                                # 段数上限 slot=cli:model:effort:family;多出的段非法,不静默丢弃
        sys.exit(f"成员段数越界:{part!r}(最多 slot=cli:model:effort:family 五段)")
    slot, cli = seg[0], seg[1]
    model = seg[2] if len(seg) > 2 else ""
    effort = seg[3] if len(seg) > 3 else ""
    family = seg[4] if len(seg) > 4 else model      # family 缺省 = model 原串
    for _x in seg[2:]:                              # 空段(如 `::`、尾随 `:`)是非法声明,不当"未声明该字段"放过
        if not _x:
            sys.exit(f"成员段为空:{part!r}(空段非法;不声明就整段不写)")
    if slot not in CAPS:                            # 未知槽位:点名拒绝(中文报错,不让 KeyError 栈回溯冒出来)
        sys.exit(f"未知槽位名 {slot!r}:槽位取自封闭词表 orch/rev/rev2/revtest/exec/exec2…exec9")
    members.append(dict(slot=slot, cli=cli, model=model, effort=effort, family=family, caps=CAPS[slot]))
# ① no duplicate slots
slots = [m["slot"] for m in members]
if len(set(slots)) != len(slots):
    sys.exit(f"roster 槽位重复:{slots}")
# ② N ≥ 2
if len(members) < 2:
    sys.exit(f"编队至少 2 名成员(实得 {len(members)})")
# ③ 能力覆盖
for cap in ("orch", "rev", "exec"):
    if not any(cap in m["caps"] for m in members):
        sys.exit(f"编队缺少 {cap} 能力:每个 roster 必须有 orch ≥1、rev ≥1、exec ≥1")
# ④ exactly one orchestrator + the orch backend restriction
_oslots = [m for m in members if "orch" in m["caps"]]
if len(_oslots) != 1:
    sys.exit("必须单编排:orch 槽位恰好一个")
if _oslots[0]["cli"] != "kimi":
    sys.exit(f"orch 槽位的 CLI 必须是 kimi(实得 {_oslots[0]['cli']}):技能就跑在 kimi 里,无钩子后端不得承担 orch(见\"组队与替补\")")
# ⑤ 成对约束:rev 能力 × exec 能力 不得同源(复合槽位自配对跳过,走"自审无效"条款)
revs = [m for m in members if "rev" in m["caps"]]
execs = [m for m in members if "exec" in m["caps"]]
for i in revs:
    for j in execs:
        if i["slot"] == j["slot"]:
            continue
        if i["cli"] == j["cli"] or (i["family"] and i["family"] == j["family"]):
            shared = f"CLI 名 {i['cli']}" if i["cli"] == j["cli"] else f"模型族 {i['family']}"
            sys.exit(
                f"组队冲突:槽位 {i['slot']}({i['cli']}/{i['model']}) 与槽位 "
                f"{j['slot']}({j['cli']}/{j['model']}) 同源(共享 {shared}):审了等于没审。"
                f"请更换其中一端的 CLI 或模型族,或在声明中给两端标注不同 family。")
# ⑥ 全名长度与字符集
for m in members:
    name = f"{m['slot']}-{tag}"
    if len(name) > 32 or not full.match(name):
        sys.exit(f"agent 全名不合法或超 32:{name}")
# ⑦ CLI 有档案
for m in members:
    if m["cli"] not in KNOWN_CLI:
        sys.exit(f"CLI {m['cli']} 无档案,先补后端档案再拉起")
# ⑧ 逐槽位字段能力:后端档案没有对应 flag 的字段,声明即拒(不静默忽略)
for m in members:
    _allow = CLI_FIELDS.get(m["cli"], set())
    for _f in ("model", "effort"):
        if m[_f] and _f not in _allow:
            sys.exit(f"槽位 {m['slot']}({m['cli']})不支持 {_f} 声明 {m[_f]!r}:该后端档案里没有对应启动参数,"
                     f"声明即拒、不静默丢弃(见\"后端档案\"参数合成表);删掉该字段或换后端")
order = {"orch": 0}
members.sort(key=lambda m: (order.get(m["slot"], 1), slots.index(m["slot"])))
out = []
for m in members:
    seg = [f"{m['slot']}={m['cli']}"]
    if m["model"]:
        seg.append(m["model"])
        if m["effort"]:
            seg.append(m["effort"])
            if m["family"] and m["family"] != m["model"]:
                seg.append(m["family"])
    out.append(":".join(seg))
print("+".join(out))                                # stdout 只出一行规范化 roster 串
PY
)
ROSTER_NORM=$(python3 -c "$ROSTER_DSL" "$ROSTER" "$TAG"
) || { echo "roster structural validation failed: stop, fix per the error above, then launch"; exit 1; }
export ROSTER_DSL ROSTER_NORM                       # the validator source and the canonical roster travel through the environment to the later parsers (the manifest's roster: line is re-validated by the very same source)
echo "normalized roster: $ROSTER_NORM"

# ── Task directory: classify before taking the lock, fix the path plan first; never overwrite unconditionally ──
# Order (verdict N3, 2026-10-09): create directory → directory/file safety check → fix the path plan → classify
#   (new / re-entry / resume) → only if classification passes, take the exclusive lock → create/verify the manifest.
# Do not take the lock before classifying: the freshly taken lock itself makes the directory non-empty, and a legitimate new
# launch gets killed as "non-empty and no manifest" (self-blocking).
# Exclusivity: one TAG allows only one launch flow at a time (concurrent sharing of the ACK file is itself disallowed); the
# lock is released once the launch completes (ACK passed). If it fails midway, do not delete the lock yet — handle it by hand,
# then delete the lock and re-enter, to prevent an accidental re-entry.
# ══ First-run flow block (begin; protocol version 2026-10-10-r6): lift this whole block out, set TAG/ROSTER/ROSTER_NORM and PWD (the project-root cwd), and it runs standalone ══
TD="/tmp/$TAG"                                          # task directory (state directory); this block always uses $TD
mkdir -p "$TD"                                          # idempotent

# Directory/file safety: check every path read or written — not a symlink (`[ -L ]`), owned by yourself; a non-existent path counts as creatable;
# after normalization it must also stay inside the task directory (a path crossing the boundary is refused up front, before anything is written)
guard() {
  [ -e "$1" ] || [ -L "$1" ] || return 0
  [ -L "$1" ] && { echo "reject: $1 is a symlink (it may point outside the task directory)"; return 1; }
  [ "$(stat -c %u -- "$1" 2>/dev/null)" = "$(id -u)" ] || { echo "reject: $1 is not owned by the current user"; return 1; }
  return 0
}
TD_REAL=$(realpath -m "$TD")
guard_td() {                                            # guard + task-directory boundary: after normalization it must equal $TD or sit under it
  guard "$1" || return 1
  case "$(realpath -m "$1")" in
    "$TD_REAL"|"$TD_REAL"/*) return 0 ;;
    *) echo "reject: $1 normalizes outside the task directory $TD_REAL"; return 1 ;;
  esac
}
# The static list carries this round's new on-disk JSON snapshots (agent-list / pane-list) and the missing list (missing-slots.txt): they too must
# refuse symlinks, have their owner checked and respect the boundary — "it is a herdr output redirect" or "the reuse check writes it itself" is no excuse
# (2026-10-10 re-review: a symlink pass-through overwrites files outside the task directory)
for p in "$TD" "$TD/manifest.txt" "$TD/taskboard.md" "$TD/bootstrap.md" "$TD/bootstrap-ack.md" "$TD/ack-expected" "$TD/mode-selection.md" "$TD/panes.txt" "$TD/agent-list.json" "$TD/pane-list.json" "$TD/missing-slots.txt"; do
  guard_td "$p" || { echo "task directory/file safety check failed: stop for manual confirmation"; exit 1; }
done

ROOT=$(realpath -m "$PWD")                              # canonical absolute project root: the anchor of task identity
# Slot order is fixed: orch always first, the rest in roster declaration order — the manifest's cwd lines, the ACK lines and
# panes.txt all follow it
SLOTS=$(python3 -c 'import sys;print(" ".join(p.split("=")[0].split(":")[0] for p in sys.argv[1].split("+")))' "$ROSTER_NORM")
N=$(echo $SLOTS | wc -w)
SLOT_ARR=($SLOTS)                                       # slots in fixed order, indexable (slot names are [a-z0-9_] only, so word splitting is safe)
declare -A CLI_OF                                       # slot → backend CLI: the start loop branches on the CLI (hookless backends start via the pane)
declare -A MODEL_OF EFFORT_OF                           # slot → declared model / effort: the per-slot argv is composed from these two tables (no longer one shared set per CLI)
while read -r _s _c _m _e; do CLI_OF[$_s]=$_c; MODEL_OF[$_s]=$_m; EFFORT_OF[$_s]=$_e; done < <(python3 - "$ROSTER_NORM" <<'PY'
import sys
for part in sys.argv[1].split("+"):
    seg = part.replace("=", ":", 1).split(":")
    model = seg[2] if len(seg) > 2 else ""
    effort = seg[3] if len(seg) > 3 else ""
    print(seg[0], seg[1], model, effort)
PY
)
declare -A EXPECT_OF                                    # slot → expected cwd: both the new and the re-entry path must fill it; the reuse check and the split loop always read it here
NO_HOOK_CLI="genox"                                     # backends whose profile declares "no status hook": not in the herdr agent registry (see "Backend profiles")
HOOKLESS=""
for s in $SLOTS; do
  case " $NO_HOOK_CLI " in *" ${CLI_OF[$s]} "*) HOOKLESS="$HOOKLESS $s=${CLI_OF[$s]}";; esac
done
HOOKLESS=${HOOKLESS# }                                  # hookless slots (fixed slot order), each recorded as <slot>=<cli>: the reuse check needs that slot's CLI name to match the pane's foreground process name, so the slot name alone is not enough
# A hookless slot's pane-evidence JSON (pane-info-<slot>.json) is named per slot: add each to the safety check (again refused before anything is written)
for _hs in $HOOKLESS; do
  guard_td "$TD/pane-info-${_hs%%=*}.json" || { echo "task directory/file safety check failed (hookless slot pane evidence): stop for manual confirmation"; exit 1; }
done

# Only two candidate formulas are prepared up front for the path plan (both are pure functions of $PWD/$TAG/slot, neither consults git):
# under the worktree plan orch = the main tree and the rest = ../wt-$TAG-<slot>; under main-only all = the main tree.
# **Which plan is used is fixed only inside the "new" branch, from the git state at that moment**;
# a re-entry always restores it from the frozen manifest and never looks at the current git state (classify first, compute paths second)
EXPECT_MAIN=$(realpath -m "$PWD")
expect_wt_of() {                                        # worktree plan: the orch slot is in the main tree, the rest in sibling worktrees
  if [ "$1" = orch ]; then echo "$EXPECT_MAIN"; else realpath -m "$PWD/../wt-$TAG-$1"; fi
}
expect_of() {                                           # take that slot's expected path under the fixed/restored $PATH_PLAN
  if [ "$PATH_PLAN" = worktree ]; then expect_wt_of "$1"; else echo "$EXPECT_MAIN"; fi
}

# ① Classify: a read-only judgement, writing nothing yet (classify first, paths second: a main-only task created before `git init` stays main-only on re-entry)
if [ -s "$TD/manifest.txt" ]; then
  CLASSIFY=reentry
elif [ -n "$(ls -A "$TD" 2>/dev/null)" ]; then
  echo "$TD is non-empty and has no manifest (leftovers, a lock from a concurrent launch, or another task): stop for manual confirmation, do not clean up automatically"; exit 1
else
  CLASSIFY=new                                          # empty directory → a legitimate new launch
fi
# ② Classification passed, only now take the exclusive lock
mkdir "$TD/launch.lock" 2>/dev/null || { echo "a launch flow is already running (launch.lock): no concurrent launches, stop"; exit 1; }

# ③ By classification: create the manifest / verify task identity
if [ "$CLASSIFY" = reentry ]; then
  # Re-entry: the manifest **unique parse** — required fields (tag / roster / project-root / created / bootstrap-id)
  # each exactly one line with a non-empty value; a field name appearing twice is a rejection (contradictory values do not count
  # as "consistent field by field");
  # the roster line first goes through the **same DSL source** (tabulated against ROSTER_DSL/ROSTER_NORM from the environment, compared by full value:
  # backend/model/effort/family all included — the same slot set with a swapped backend or model is still a rejection);
  # cwd line count == N, the cwd slot set == the roster line's slot set, and a per-slot realpath comparison against the **frozen plan**.
  # The frozen plan = the one fixed at creation time, recovered on re-entry from the manifest's set of cwd lines (both candidate
  # formulas — worktree / main-only — are passed in by the shell as **pure functions**, consulting no current git state): the cwd
  # lines must fall as a whole inside exactly one of them; the plan is never re-derived.
  # stdout emits N verified paths (fixed slot order) + a bootstrap-id line + a path-plan line, for the shell to read back line by line.
  _eargs=()                                             # plan name and slot/expected-path pairs go in as a **referenced array**: a path containing spaces stays one argv, never split
  for _pl in worktree main-only; do                     # one group per candidate plan: chosen at creation, claimed from the cwd lines on re-entry
    _eargs+=("$_pl")
    for s in $SLOTS; do
      if [ "$_pl" = worktree ]; then _eargs+=("$s" "$(expect_wt_of "$s")"); else _eargs+=("$s" "$EXPECT_MAIN"); fi
    done
  done
  _mout=$(python3 - "$TAG" "$ROOT" "$TD/manifest.txt" "$SLOTS" "$ROSTER_NORM" "${_eargs[@]}" <<'PY'
import os, sys, subprocess
tag, root, path, rnorm = sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[5]   # this run's full canonical roster is an **explicit argument** (no longer just the slot table)
slots = sys.argv[4].split()                             # $SLOTS arrives as a space-separated **string**: it must be split into a list, or list-vs-string never compares equal
rest = sys.argv[6:]
if not slots or len(set(slots)) != len(slots):
    sys.exit(f"槽位表非法(为空或有重复):{slots}")
dsl = os.environ.get("ROSTER_DSL", "")
if not dsl or not rnorm:
    sys.exit("环境里没有本次的 roster 校验器源码(ROSTER_DSL)/规范 roster 入参:fail-closed,不识别本任务")
def canon(rs):                                          # validate + normalize with the **same** DSL source; anything that fails stops right here
    r = subprocess.run([sys.executable, "-c", dsl, rs, tag], capture_output=True, text=True)
    if r.returncode != 0:
        sys.exit("roster 未过 DSL 校验:" + (r.stderr.strip() or r.stdout.strip()))
    return r.stdout.strip()
if canon(rnorm) != rnorm:
    sys.exit("本次规范 roster 入参不是规范形:fail-closed")
stride = 1 + 2 * len(slots)
if not rest or len(rest) % stride:
    sys.exit(f"路径方案组不是整组 N 对:槽位 {len(slots)} 个、参数 {len(rest)} 项")
groups = {}
for off in range(0, len(rest), stride):
    grp = rest[off:off + stride]
    plan, pairs = grp[0], grp[1:]
    if plan not in ("worktree", "main-only") or plan in groups:
        sys.exit(f"路径方案组名非法或重复:{plan!r}")
    want = dict(zip(pairs[0::2], pairs[1::2]))
    if set(want) != set(slots) or any(not p for p in want.values()):
        sys.exit(f"方案 {plan} 的槽位→预期路径映射不完整或含空值")
    groups[plan] = want
if set(groups) != {"worktree", "main-only"}:
    sys.exit(f"缺少路径方案组:实得 {sorted(groups)}")
seen = {}
for line in open(path, encoding="utf-8"):
    line = line.rstrip("\n")
    if not line:
        continue
    if ":" not in line:
        sys.exit(f"manifest 无法解析的行:{line!r}")
    k, v = line.split(":", 1)
    if not v.strip():
        sys.exit(f"manifest 字段 {k} 值为空")
    seen.setdefault(k, []).append(v.strip())
def one(k):                                             # a required field is exactly one line; a duplicate is a conflict
    vs = seen.get(k, [])
    if len(vs) != 1:
        sys.exit(f"manifest 字段 {k} 应恰好一行(实得 {len(vs)})")
    return vs[0]
if one("tag") != tag: sys.exit("tag 不符")
if one("project-root") != root: sys.exit("project-root 不符")
one("created")                                          # present, exactly one line, non-empty value
task_id = one("bootstrap-id")                           # task identity number: the sole basis for rebuilding the expected ACK
roster = one("roster")                                  # the roster line: required, exactly one line, non-empty
if canon(roster) != rnorm:
    sys.exit(f"manifest 的 roster 行与本次规范 roster 不同:{canon(roster)!r} ≠ {rnorm!r}(换了后端/模型/推理档/family 都算不同)")
cwds = seen.get("cwd", [])
if len(cwds) != len(slots):
    sys.exit(f"cwd 行数应 == N = {len(slots)}(实得 {len(cwds)})")
got = {}
for v in cwds:
    name, sep, p = v.partition(" ")
    if sep != " " or not p.strip(): sys.exit(f"cwd 行缺分隔符或路径为空:{v!r}")
    if not name.endswith("-" + tag):
        sys.exit(f"cwd 行成员名 {name!r} 不符 <slot>-{tag} 命名:非本任务")
    slot = name.rsplit("-" + tag, 1)[0]                 # the manifest records the full name <slot>-<tag>: normalize to the bare slot before comparing (differing conventions never match)
    if not slot: sys.exit(f"cwd 行成员名 {name!r} 的槽位为空")
    if slot in got: sys.exit(f"cwd 槽位 {slot} 重复")
    got[slot] = p.strip()
if set(got) != set(slots): sys.exit("cwd 槽位集合与 roster 行不符")
if [k for k in got] != slots: sys.exit("cwd 行序与槽位固定序不符")
def mism(plan):                                         # per-slot detail of what does not match under that plan
    return ", ".join(f"{s} cwd={got[s]} ≠ 方案预期 {groups[plan][s]}"
                     for s in slots if os.path.realpath(got[s]) != os.path.realpath(groups[plan][s]))
matched = [pl for pl in ("worktree", "main-only") if not mism(pl)]
if len(matched) != 1:                                   # 0 = belongs to no frozen plan; 2 = ambiguous; both fail closed
    sys.exit("cwd 行不落在唯一的冻结路径方案里(" + ("; ".join(mism(pl) for pl in ("worktree", "main-only")) if not matched
             else "两个方案同时成立,方案歧义") + ")")
print(f"manifest 唯一解析通过:字段无重复、值非空;roster 行过同一 DSL 且与本次全值一致;cwd 行属冻结方案 {matched[0]}", file=sys.stderr)
for name in slots:
    print(got[name])                                    # stdout emits exactly N verified paths (fixed slot order), for the shell to read back
print("bootstrap-id: " + task_id)                       # task identity number: the reuse check rebuilds the expected ACK from it
print("path-plan: " + matched[0])                       # the frozen plan: a re-entry never re-derives it from the current git state
PY
  ) || { echo "manifest does not match this run (duplicate field/empty value/project root/roster full value/frozen-plan path/missing path): not recognized as this task, stop for manual confirmation"; exit 1; }
  # The expected cwds were verified by the same parser with the same algorithm above (re-entry neither re-assembles the plan from the
  # current git state nor parses a second time — the awk second parse once drifted from the manifest line format, blanking all three
  # EXPECT values, 2026-10-09 N4); read them back line by line with r5's `IFS= read -r`, **no `eval`**: a path is data, not shell
  # source (it may contain spaces, backslashes, a literal $()), and read-back must place it into the array untouched; guard against
  # empty values and against too many/too few lines
  i=0; TASK_ID=""; PLAN_IN=""
  while IFS= read -r _line; do
    if [ "$i" -lt "$N" ]; then
      [ -n "$_line" ] || { echo "the re-entry read-back slot cwd is empty (line $((i+1))): stop for manual confirmation"; exit 1; }
      EXPECT_OF[${SLOT_ARR[$i]}]=$_line                 # placed line by line in fixed slot order (the generalization of r5's three EXPECT_* variables)
    else
      case "$_line" in
        "bootstrap-id: "*) [ -z "$TASK_ID" ] || { echo "re-entry read back a duplicate bootstrap-id line: stop for manual confirmation"; exit 1; }
                           TASK_ID=${_line#bootstrap-id: } ;;
        "path-plan: "*)    [ -z "$PLAN_IN" ] || { echo "re-entry read back a duplicate path-plan line: stop for manual confirmation"; exit 1; }
                           PLAN_IN=${_line#path-plan: } ;;
        *) echo "re-entry read back an unexpected line: $_line (fail-closed)"; exit 1 ;;
      esac
    fi
    i=$((i+1))
  done <<< "$_mout"
  [ "$i" = "$((N + 2))" ] || { echo "re-entry read back $i lines, expected $((N + 2)) (N paths + bootstrap-id + path-plan): stop for manual confirmation"; exit 1; }
  [ -n "$TASK_ID" ] || { echo "the re-entry read-back bootstrap-id is empty: fail-closed"; exit 1; }
  case "$PLAN_IN" in
    worktree|main-only) PATH_PLAN=$PLAN_IN ;;
    *) echo "the re-entry read-back path plan is illegal: $PLAN_IN (fail-closed)"; exit 1 ;;
  esac
  EXPECT_MAIN=${EXPECT_OF[orch]}                        # the orch slot's cwd is the main tree path (under the main-only plan every slot is it)
  echo "re-entry: manifest unique parse consistent (roster full value matches, cwd lines belong to the frozen plan $PATH_PLAN); reusing the existing taskboard, filling gaps only, never overwriting"
else
  # The path plan is fixed only inside the **new** branch, from the current git state; a re-entry always restores the frozen plan above
  # (classify first, paths second: a main-only task created before `git init` stays main-only on re-entry, it is not re-judged because the tree is now a repo)
  #   git repo → the orch slot's cwd = the main tree, and every non-orch slot gets a sibling worktree; non-git → all N cwds record the main tree (allowed only in attended mode)
  if git -C "$PWD" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    PATH_PLAN=worktree
  else
    PATH_PLAN=main-only
  fi
  CREATE_ID=$(date +%s)-$RANDOM                         # task identity number: taken only on a new launch, written into the manifest; the ACK expectation uses it too
  {
    echo "tag: $TAG"
    echo "roster: $ROSTER_NORM"
    echo "project-root: $ROOT"
    for s in $SLOTS; do                                  # N lines, fixed slot order (orch always first)
      echo "cwd: $s-$TAG $(expect_of "$s")"
    done
    echo "created: $(date -Is)"
    echo "bootstrap-id: $CREATE_ID"
  } > "$TD/manifest.txt"
  TASK_ID=$CREATE_ID
  for s in $SLOTS; do EXPECT_OF[$s]=$(expect_of "$s"); done   # same as the re-entry branch: the new-launch branch must fill the per-slot expected cwds too
                                                              # (r5 kept the three EXPECT_* in code shared by both branches; r6's generalization left them only in the re-entry branch, 2026-10-09 T001-d D4)
  echo "new: task directory initialized (path plan $PATH_PLAN; lock held, released once the ACK passes)"
fi
export TASK_ID                                          # the task identity number travels through the environment to the reuse check (needed to rebuild the expected ACK)
# ══ First-run flow block (end) ══

# Reuse check: JSON-exact match of the N full agent names; then each slot's cwd is checked against that slot's **expected cwd** (under the frozen path plan)
# (it is not "require every slot's cwd to be equal" — same tab but having wandered into another directory still cannot be reused);
# both sides are compared after realpath normalization — a path with `..` and a canonical cwd that point at the same directory
# must compare equal, or a normal re-entry would be misjudged;
# a missing tab_id/workspace_id exits non-zero (fail-closed); "it printed a conclusion" is not a pass;
# a registered member's actual backend kind (when the output carries that field) must equal the frozen roster's CLI for that slot — same slot, swapped backend = identity mismatch;
# hookless slots ($HOOKLESS, each item <slot>=<cli>; profiles that declare they are not in the herdr registry, e.g. genox) are exempt from the
# agent-list existence check and are verified three ways instead:
#   ① pane identity: that slot's mapping in panes.txt is **unique** (missing field / duplicate line / duplicate pane all rc=3), looked up
#      **by pane ID** (not by label) in pane list, checking label == <slot>-<tag>, same tab/workspace, and that pane's own cwd == expected;
#   ② process evidence from the same cell: pane-info-<slot>.json's process_info.pane_id must == the pane ID from ① (evidence from a
#      different pane may never be stitched in), the foreground process must be non-empty, its name == the CLI declared for that slot, and its cwd == expected (no process = missing, refill that one cell);
#   ③ on-disk ACK identity: **rebuild** the N+2-line expectation from the **already uniquely parsed task identity** (TASK_ID) + tag + the current N full member names,
#      then compare bootstrap-ack.md against it in full (an empty expectation / wrong tag / wrong bootstrap-id / wrong member order are all rejected; not an existence grep);
# Order: first check the identity of **every member on stage**, and any inconsistency takes precedence as rc=3; only then compute the missing set, and rc=2 reports only the missing slots.
# The missing list lands in $TD/missing-slots.txt (one space-separated line) for the "refill only" launch branch to read; the expected paths travel item by
# item as argv from a **referenced bash array** (a path with spaces is not split); the parser first asserts "slot count == path count, non-empty, unique" and only then compares.
herdr agent list > "$TD/agent-list.json"
herdr pane list  > "$TD/pane-list.json"
for _hs in $HOOKLESS; do
  s=${_hs%%=*}                                          # each $HOOKLESS item is <slot>=<cli>: take the slot name to look up panes.txt and to name the pane-info file
  p=""
  if [ -s "$TD/panes.txt" ]; then
    # that slot's mapping in panes.txt must be exactly one line with both fields non-empty: a missing field or a duplicate mapping fails closed (never pick one line and hope)
    p=$(awk -v s="$s" '$1 == s {n++; id=$2} END {if (n != 1) exit 1; print id}' "$TD/panes.txt") \
      || { echo "the pane mapping for slot $s in panes.txt is missing or duplicated (<slot> <pane_id> must be exactly one line): fail-closed, manual confirmation"; exit 3; }
  fi
  [ -n "$p" ] || continue                               # no pane record → let the parser report it as a missing member (a fresh launch has no panes yet)
  herdr pane process-info --pane "$p" > "$TD/pane-info-$s.json" \
    || { echo "cannot get pane process-info for hookless slot $s: fail-closed, manual confirmation"; exit 3; }
done
EXPECT_ARGS=()
for s in $SLOTS; do EXPECT_ARGS+=("${EXPECT_OF[$s]}"); done
python3 -c '
import json,os,sys
tag=sys.argv[1]; slots=sys.argv[2].split(); hookless=sys.argv[3].split()
td=sys.argv[4]; paths=sys.argv[5:]
taskid=os.environ.get("TASK_ID",""); rnorm=os.environ.get("ROSTER_NORM","")
def bad(msg):
    print(msg); sys.exit(3)
if not slots or len(set(slots))!=len(slots) or len(paths)!=len(slots) or any(not p for p in paths):
    bad("复用检查入参不完整:槽位 %d 个、路径 %d 个(fail-closed,不复用)"%(len(slots),len(paths)))
if not taskid or not rnorm:
    bad("复用检查缺任务身份/规范 roster(TASK_ID/ROSTER_NORM 环境变量):fail-closed,不复用")
cli={}
for part in rnorm.split("+"):
    seg=part.replace("=",":",1).split(":")
    if len(seg)<2 or not seg[0] or not seg[1]: bad("规范 roster 解析不过: "+part)
    cli[seg[0]]=seg[1]
if set(cli)!=set(slots): bad("规范 roster 的槽位集合与本次槽位表不符:fail-closed")
hook=dict(h.split("=",1) for h in hookless)
expect={s:p for s,p in zip(slots,paths)}
def norm(p):
    return os.path.realpath(p) if p else p
def load(name):
    f=os.path.join(td,name)
    if not os.path.exists(f): return None
    try:
        return json.load(open(f,encoding="utf-8"))
    except Exception as e:
        bad(name+" 读不出或不是合法 JSON: "+str(e)+":fail-closed,不复用")
def panes_txt():                                        # unique parse of panes.txt: a missing field / duplicate slot / duplicate pane is rc=3
    f=os.path.join(td,"panes.txt")
    if not os.path.exists(f): return {},False           # no panes.txt yet = a brand-new task, not an inconsistency
    m={}; ids={}
    for i,l in enumerate(open(f,encoding="utf-8"),1):
        l=l.rstrip("\n")
        if not l: bad("panes.txt 第 %d 行是空行:fail-closed"%i)
        parts=l.split()
        if len(parts)!=2: bad("panes.txt 第 %d 行不是 <slot> <pane_id> 两段: %r,fail-closed"%(i,l))
        s,p=parts
        if s in m: bad("panes.txt 里槽位 %s 有多条 pane 映射:fail-closed"%s)
        if p in ids: bad("panes.txt 里 pane %s 被多个槽位引用:fail-closed"%p)
        m[s]=p; ids[p]=s
    return m,True
al=load("agent-list.json"); pl=load("pane-list.json")
if al is None or pl is None: bad("agent-list.json / pane-list.json 缺一个:fail-closed,不复用")
agents=al.get("result",{}).get("agents"); panes=pl.get("result",{}).get("panes")
if not isinstance(agents,list) or not isinstance(panes,list): bad("agent list / pane list 结构不是列表:fail-closed")
got={}
for a in agents:
    n=a.get("name")
    if n in got: bad("agent list 里重名 %r:fail-closed"%n)
    got[n]=a
pane_by_id={}
for p in panes:
    pid=p.get("pane_id")
    if not pid: bad("pane list 有条目缺 pane_id:fail-closed")
    if pid in pane_by_id: bad("pane list 里 pane_id %s 重复:fail-closed"%pid)
    pane_by_id[pid]=p
ptxt, ptxt_ok = panes_txt()
tabs=set(); missing=[]; why={}; hook_live=False
for s in slots:                                         # order: every member on stage first, the missing set after (E2-4)
    if s in hook:
        pid=ptxt.get(s)
        if pid is None:
            if ptxt_ok: bad("panes.txt 在、却没有槽位 %s 的 pane 映射:不一致,fail-closed"%s)
            missing.append(s); why[s]="还没有 pane 记录(fresh)"; continue
        pane=pane_by_id.get(pid)
        if pane is None: bad("无钩子槽位 %s 的 pane %s 不在 pane list 里:fail-closed"%(s,pid))
        if pane.get("label")!=s+"-"+tag:
            bad("pane %s 的 label=%r ≠ %s:证据不是同一格,fail-closed"%(pid,pane.get("label"),s+"-"+tag))
        if not pane.get("tab_id") or not pane.get("workspace_id"): bad("pane %s 缺 tab_id/workspace_id:fail-closed"%pid)
        if norm(pane.get("cwd"))!=norm(expect[s]):
            bad("pane %s 的 cwd=%s ≠ 预期 %s(realpath 规范化后仍不同):fail-closed"%(pid,pane.get("cwd"),expect[s]))
        info=load("pane-info-"+s+".json")
        if info is None: bad("无钩子槽位 %s 缺 pane-info-%s.json(进程证据):fail-closed"%(s,s))
        pi=info.get("result",{}).get("process_info")
        if not isinstance(pi,dict): bad("pane-info-%s.json 里没有 process_info:fail-closed"%s)
        if pi.get("pane_id")!=pid:
            bad("槽位 %s 的进程证据来自 pane %s、panes.txt 记的是 %s:不许拼接不同 pane 的证据,fail-closed"%(s,pi.get("pane_id"),pid))
        procs=pi.get("foreground_processes") or []
        if not procs:
            missing.append(s); why[s]="pane 内无前台进程(在原 pane 上重起)"; continue
        p0=procs[0]; pname=p0.get("name") or ((p0.get("argv") or [""])[0])
        if pname!=hook[s]: bad("槽位 %s pane 前台进程=%s ≠ 该槽位 CLI=%s:fail-closed,人工确认"%(s,pname,hook[s]))
        if norm(p0.get("cwd"))!=norm(expect[s]):
            bad("槽位 %s pane 进程 cwd=%s ≠ 预期 %s(realpath 规范化后仍不同):fail-closed"%(s,p0.get("cwd"),expect[s]))
        tabs.add(pane.get("tab_id")); hook_live=True
    else:
        w=s+"-"+tag; a=got.get(w)
        if a is None:
            missing.append(s); why[s]="agent list 里没有"; continue
        if not a.get("tab_id") or not a.get("workspace_id"): bad(w+" 缺 tab_id/workspace_id,fail-closed:不复用")
        if norm(a.get("cwd"))!=norm(expect[s]): bad(w+" cwd="+str(a.get("cwd"))+" ≠ 预期 "+str(expect[s])+"(realpath 规范化后仍不同):不复用")
        k=a.get("kind")
        if k is not None and k!=cli[s]:
            bad(w+" 实际后端 kind="+str(k)+" ≠ 冻结 roster 的 CLI="+cli[s]+":身份不符,fail-closed")
        tabs.add(a.get("tab_id"))
if len(tabs)>1: bad("成员不在同一 tab: "+", ".join(sorted(str(t) for t in tabs)))
if hook_live:                                           # a hookless instance is on stage: the on-disk ACK must equal the **rebuilt** expectation
    want="bootstrap-id: "+taskid+"\ntag: "+tag+"\n"+"".join(s+"-"+tag+"\n" for s in slots)
    n_exp=len(slots)+2
    ef=os.path.join(td,"ack-expected")
    if os.path.exists(ef) and open(ef,encoding="utf-8").read()!=want:
        bad("落盘 ack-expected 与按任务身份重建的期望不同:身份没显式更新或被改过,fail-closed")
    af=os.path.join(td,"bootstrap-ack.md")
    ack=open(af,encoding="utf-8").read() if os.path.exists(af) else ""
    if ack!=want:
        bad("无钩子槽位复用的落盘 ACK 身份不成立:与按任务身份重建的 %d 行期望不全等(空/错 tag/错 bootstrap-id/错成员序都算):fail-closed,人工确认"%n_exp)
open(os.path.join(td,"missing-slots.txt"),"w",encoding="utf-8").write(" ".join(missing)+"\n")
if missing:
    print("缺: "+", ".join(s+"-"+tag+"("+why[s]+")" for s in missing)+":只补缺,只对这几格动手,已在场成员一个都不重启")
    print("missing-slots: "+" ".join(missing)); sys.exit(2)
print("%d 名成员齐全:同 tab,各自 cwd 与预期一致(realpath 规范化后);已注册成员后端与冻结 roster 一致;无钩子槽位 pane 身份链(panes.txt→pane list→同一 pane 的进程证据)已核,落盘 ACK 与重建期望全文全等"%len(slots))' \
  "$TAG" "$SLOTS" "$HOOKLESS" "$TD" "${EXPECT_ARGS[@]}"
# The reuse-check verdict → what this launch does: 0 = everything reusable (start nothing); 2 = refill only (iterate only $MISSING_SLOTS);
# anything else (3 = topology/directory/identity mismatch) = fail-closed, stop here for manual confirmation. Read the list back and check it once more
# (empty / an unknown slot / a duplicate all fail closed)
_reuse_rc=$?
REUSE=""
MISSING_SLOTS=""
case "$_reuse_rc" in
  0) REUSE=ok ;;
  2) REUSE=refill
     MISSING_SLOTS=$(cat "$TD/missing-slots.txt" 2>/dev/null) \
       || { echo "cannot read the missing-slot list $TD/missing-slots.txt: fail-closed, manual confirmation"; exit 3; }
     [ -n "$MISSING_SLOTS" ] || { echo "the missing-slot list is empty while the exit code is 2: self-contradictory, fail-closed"; exit 3; }
     for s in $MISSING_SLOTS; do
       case " $SLOTS " in *" $s "*) ;; *) echo "the missing-slot list contains $s, which is not in this round's slot table: fail-closed"; exit 3 ;; esac
     done
     [ "$(printf '%s\n' $MISSING_SLOTS | sort | uniq -d | wc -l)" = 0 ] \
       || { echo "the missing-slot list contains duplicates: fail-closed"; exit 3; }
     ;;
  *) echo "reuse check returned ${_reuse_rc} (identity/topology/directory mismatch): fail-closed, confirm by hand before launching"; exit 3 ;;
esac
export REUSE MISSING_SLOTS
# Exit codes: 0 = reusable (REUSE=ok); 2 = members missing (REUSE=refill, only the slots in $MISSING_SLOTS; a hookless slot counts as missing when no process runs in its pane);
# 3 = topology/directory mismatch, missing field, pane-evidence stitching, or a hookless slot's ACK identity failing (fail-closed, do not reuse automatically, rebuild after manual confirmation)
# Expected cwds always come from the manifest: on a new launch they are written per the path plan fixed in the "first-run flow block" (git → the main tree + N−1 sibling worktrees;
# non-git → all N cwds record the main tree, allowed only in attended mode), and on re-entry they are read back through the unique parse — here the N expected paths are passed to the reuse check
```

### Mode selection (after detection, before launch — mandatory)

**Launch parameters are the outcome of mode selection, not defaults; launching without having done mode selection is a process violation.** The task directory, the manifest (project root / each slot's expected cwd / roster / creation time / bootstrap ID) and the exclusive lock were already handled in the "detection" step above; this step checks the three unattended-mode enabling gates one by one, each documenting its evidence in `/tmp/$TAG/mode-selection.md` (evidence you cannot write down = that gate fails), and once the taskboard exists its pointer goes into the board:

| Gate | What is actually checked (not merely "a protection flag exists") | What the evidence says |
|---|---|---|
| 1 Push boundary | ① **The pushing identity itself has no bypass rights** (both admin direct pushes and protection-rule bypass lists must be checked); ② the target ref (main/master) is actually covered by a protection rule; ③ other branches that allow pushes must not allow force-push either | The actual branch-protection settings (e.g. `gh api repos/<owner>/<repo>/branches/main/protection` for `enforce_admins` and the rules), or credential-side read-only evidence; if it cannot be verified → fail |
| 2 Credential surface | **Verified per member against the process list**: each member's full agent name + process (argv/pid) + required credentials (only each CLI's own login state) + readable sensitive paths + isolation means (separate Unix user / container / no mounts) | The minimal verifiable isolation config + mount list; a container existing ≠ no sensitive data mounted, a separate user name ≠ files unreadable — write each one down |
| 3 Directory and permission scope | **Worktree availability and process permission scope are accepted separately**: a worktree isolates only working files and the index, and the same Unix user is not constrained by it when writing across directories (see "Known limitations" item 3). Under the git plan the worktree count = N−1 (orch uses the main tree, every non-orch slot gets one); non-git → all cwds = the main tree | ① It is a git repo and N−1 worktrees can be created; ② the process's write boundary: a restricted scope such as a container/separate user, **or the user explicitly choosing "same-user cross-directory access" as the authorized scope** (if chosen, restate unattended item 3 as that chosen scope and stop claiming cross-directory always confirms); no restricted scope and no explicit choice → fail |

All three gates pass → the **unattended parameter set**; any failure → the **attended parameter set**, and the taskboard states which gate failed and on what evidence. **The two rows of parameter sets as a per-CLI table live in "Backend profiles"**; each member's launch parameters are looked up there by that member's CLI, not hard-coded here.

The failure branch is not "launch with the permissive flags anyway and note it on the board":

- Gate 1 fails → **the push ban for every exec-capability member must be enforced as a capability**: that member's instance holds no push credential, or the remote grants restricted write access to line branches only; writing "push banned" on the board does not count;
- Gate 2 fails → **do not issue the bypass/auto launch command**; switch to the attended parameter set (given explicitly per CLI profile, see "Backend profiles");
- Gate 3 fails (non-git / no restricted permission scope) → same as above, fall back to attended mode, with the human as the backstop.

**The attended set means "explicit manual mode + acceptance", not "omit the flag":**

- Give it **explicitly** per that CLI's profile, and never accept inherited machine defaults (reasons and measurements are in the corresponding CLI entries under "Backend profiles"); after launch, **accept the real mode**: `herdr pane read <that slot's pane>` to see the status bar, or have that agent run `/status` to confirm it is currently manually approving (neither bypass nor auto), and record the evidence in `mode-selection.md`; **if the manual mode cannot be verified, that gate has failed — stop dispatching**, troubleshoot first or stop and relaunch, never dispatch into an unknown mode;
- **An existing session does not take launch parameters**: the permission mode is a property of the process — changing a shell variable or a taskboard note does not change an already running process. If the target agent is already running when a gate fails (especially already inside bypass/auto), you **must stop it first (exit/kill) and relaunch it in the manual mode**, and only resume dispatching after re-verifying the mode; "changed a variable, so consider it downgraded" is not allowed;
- The orch-slot exception is closed off (that "the current session acts as orchestrator" exception at the end of Step 0): the semantics of `--auto` is **Never Ask** — it does not interrupt the user, it executes automatically and decides automatically; the user sitting at the terminal = a human present, but **being present ≠ the user approving**, and rename only registers identity — it neither changes the CLI's permission mode nor makes the CLI wait for a nod before acting. When the current session is already running inside `--auto` and the three gates fail, only two options are allowed: **(a) documented narrowing (not equivalent to human approval)**: the user writes the authorized scope explicitly into `/tmp/$TAG/mode-selection.md` (which operations may run automatically, how supervision and stopping work), and that branch then runs characterized as "**authorized automatic execution + human supervision**", labelled truthfully, and never again claims equivalence to human approval; **(b) no such record → stop work**. If you want actions to genuinely wait for human approval beforehand, you need a separate external dispatch/execution approval gate (this skill has no built-in layer for that).

Create a separate tab and launch N **brand-new clean** agents per the layout rules (do not resume from old sessions). **First create each slot's worktree in the main tree, then create panes and start agents with the corresponding cwd** (a worktree isolates only working files and the index, not git metadata, and is not a security boundary — see "Known limitations"; when the project directory is **not a git repo**, everything falls back to launching in the main tree with the same cwd, the isolation guarantee drops, **and unattended mode automatically falls back to attended mode** — see gate 3 of the "Unattended mode" enabling gates, and note it on the board):

```bash
# 0) one worktree per non-orch slot (skipped if it already exists); the branch name carries both the tag and the slot name, so they never overlap.
#    Execute only the path plan fixed in the "first-run flow block": skip under the non-git fallback (main-only), where all N cwds are the main tree; on re-entry create none at all
if [ "$CLASSIFY" = new ] && [ "$PATH_PLAN" = worktree ]; then
  for s in $SLOTS; do
    [ "$s" = orch ] && continue
    git worktree add "$PWD/../wt-$TAG-$s" -b "trio-$TAG-$s"
  done
fi

declare -A PANE_OF                                       # slot → pane: both the start loop and the rename loop read it here, and each slot is asserted non-empty
START_SLOTS=""                                           # slots to start this round: a new launch = all of them; a re-entry refill = only the missing ones

if [ "$CLASSIFY" = new ]; then
  START_SLOTS=$SLOTS                                     # new launch: every slot must be started
  # 1) New tab: the orch slot stays in the main tree; the root pane P0 is parsed from the returned JSON
  P0=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "${EXPECT_OF[orch]}" --label <task name> --no-focus \
       | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["root_pane"]["pane_id"])')
  # 2) The pane-splitting loop (P0 = the orch position): non-orch members in declaration order M1…M(N−1);
  #    M1: P0 --direction right → P1; Mk (k≥2): P(k−1) --direction down → Pk.
  #    At N=3 this matches the current layout pane by pane; at N>3 the right column keeps extending downwards.
  #    Parse each pane_id from the returned JSON right after splitting — never copy it by hand; and write $TD/panes.txt (one `<slot> <pane_id>` per line).
  #    slot → pane is held in an **associative array** PANE_OF (no more assembling variable names from slot names): the start loop and the
  #    rename loop both read it, and each slot is asserted non-empty — writing PANE_OF_ORCH but reading PANE_ORCH blanks the pane, so stop right there.
  PANE_OF[orch]=$P0
  : > "$TD/panes.txt"; echo "orch $P0" >> "$TD/panes.txt"
  prev=$P0; first=1
  for s in $SLOTS; do
    [ "$s" = orch ] && continue
    if [ "$first" = 1 ]; then dir=right; first=0; else dir=down; fi
    p=$(herdr pane split "$prev" --direction "$dir" --cwd "$(expect_of "$s")" --no-focus \
        | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
    [ -n "$p" ] || { echo "pane_id for slot $s parsed empty: stop for manual confirmation"; exit 1; }
    PANE_OF[$s]=$p
    echo "$s $p" >> "$TD/panes.txt"
    prev=$p
  done
elif [ "${REUSE:-}" = refill ]; then
  # Re-entry refill: iterate **only** $MISSING_SLOTS; members already on stage are not touched at all (no agent start / pane run resent, no new pane/tab);
  # the pane is always the original one recorded in panes.txt at the first launch — a re-entry never splits a new pane (geometry is not protocol identity; the identity anchor is manifest/panes.txt)
  START_SLOTS=$MISSING_SLOTS
  for s in $START_SLOTS; do
    p=$(awk -v s="$s" '$1 == s {n++; id=$2} END {if (n != 1) exit 1; print id}' "$TD/panes.txt") \
      || { echo "the pane mapping for slot $s in panes.txt is missing or duplicated: fail-closed, manual confirmation"; exit 3; }
    [ -n "$p" ] || { echo "slot $s has no pane ID in panes.txt: fail-closed, manual confirmation"; exit 3; }
    PANE_OF[$s]=$p
  done
elif [ "${REUSE:-}" = ok ]; then
  :                                                      # everything reusable: start nothing and leave the panes alone
else
  echo "no usable reuse-check verdict (REUSE=${REUSE:-unset}, classification ${CLASSIFY}): never launch blindly, stop for manual confirmation"; exit 3
fi

# 3) Start loop: look up the "backend profile" per slot; the permission flags come from the "mode selection" arrays, while the
#    **model/effort are composed per slot** from the canonical roster;
#    hard-coding bypass/auto is a protocol violation, and so is hard-coding the model in a shared array (two slots on the same CLI would overwrite each other).
#    Branch on that slot's **cli** (not on the slot name): herdr agent start needs --kind (kimi/codex/claude etc.);
#    a member whose profile declares "no hook" (genox) never enters the registry and is started straight in its pane (`pane run genox`; the version
#    installed today, 3.0.1, reports `Missing command` for bare `genox` (measured, exit code 2), so `genox chat` is used for now; once the upstream PR
#    merges, the bare command is equivalent to it — that equivalence is **[not tested]**; see the genox entry under "Backend profiles"),
#    fake no kind registration, and never call it "missing" just because it is absent from agent list.
SLOT_ARGS=()                                            # composed output: cleared before every call
slot_argv_of() {                                        # $1 = slot: composed only from that slot's canonical roster declaration (gate ⑧ already blocks "declared but no flag" input)
  SLOT_ARGS=()
  case "${CLI_OF[$1]}" in
    kimi)   if [ -n "${MODEL_OF[$1]}" ]; then SLOT_ARGS+=(-m "${MODEL_OF[$1]}"); fi ;;
    codex)  if [ -n "${MODEL_OF[$1]}" ]; then SLOT_ARGS+=(-m "${MODEL_OF[$1]}"); fi
            if [ -n "${EFFORT_OF[$1]}" ]; then SLOT_ARGS+=(-c "model_reasoning_effort=${EFFORT_OF[$1]}"); fi ;;
    claude) if [ -n "${MODEL_OF[$1]}" ]; then SLOT_ARGS+=(--model "${MODEL_OF[$1]}"); fi
            if [ -n "${EFFORT_OF[$1]}" ]; then SLOT_ARGS+=(--effort "${EFFORT_OF[$1]}"); fi ;;
    *)      if [ -n "${MODEL_OF[$1]}${EFFORT_OF[$1]}" ]; then
              echo "CLI ${CLI_OF[$1]} of slot $1 has no registered model/effort composition rule: never drop it silently, stop and add the rule per the 'backend profile'"
              return 1
            fi ;;
  esac
  return 0
}
for s in $START_SLOTS; do
  pane=${PANE_OF[$s]}
  [ -n "$pane" ] || { echo "slot $s has no pane (the split loop never assigned it): stop for manual confirmation"; exit 1; }
  slot_argv_of "$s" || exit 1
  case "${CLI_OF[$s]}" in
    genox)  herdr pane run "$pane" "genox chat" ;;                    # hookless: the only usable mode = a human present; flags in "Backend profiles"
    kimi)   herdr agent start "$s-$TAG" --kind kimi   --pane "$pane" -- "${KIMI_ARGS[@]}"   "${SLOT_ARGS[@]}" ;;
    codex)  herdr agent start "$s-$TAG" --kind codex  --pane "$pane" -- "${CODEX_ARGS[@]}"  "${SLOT_ARGS[@]}" ;;
    claude) herdr agent start "$s-$TAG" --kind claude --pane "$pane" -- "${CLAUDE_ARGS[@]}" "${SLOT_ARGS[@]}" ;;
    *)      echo "CLI ${CLI_OF[$s]} of slot $s has no registered flag set in this loop: measure it against the 'backend profile' and add a case (do not reuse the values above), then launch"; exit 1 ;;
  esac
done
# With the default roster expanded, this is equivalent to (the permission arrays carry permission flags only; model/effort come from each slot's roster declaration):
#   herdr agent start "orch-$TAG" --kind kimi  --pane "${PANE_OF[orch]}" -- "${KIMI_ARGS[@]}"
#   herdr agent start "rev-$TAG"  --kind codex --pane "${PANE_OF[rev]}"  -- "${CODEX_ARGS[@]}"  -m gpt-6.1-sol -c model_reasoning_effort=high
#   herdr agent start "exec-$TAG" --kind claude --pane "${PANE_OF[exec]}" -- "${CLAUDE_ARGS[@]}"
# Each array element becomes its own argv: a path with spaces is not split, and an empty array contributes no argument; never fall back to string concatenation (see the measurements under "Mode selection")
# The model slug and the effort level must be passed separately: the slug carries no level, the level uses its own config key; `--settings` composes the path from the skill-directory variable, never a hard-coded username

# 4) Rename the panes started this round (new launch = all; re-entry refill = only the missing ones); the pane label and the agent name are two different objects
for s in $START_SLOTS; do
  pane=${PANE_OF[$s]}
  [ -n "$pane" ] || { echo "slot $s has no pane (the split loop never assigned it): stop for manual confirmation"; exit 1; }
  herdr pane rename "$pane" "$s-$TAG"
done

# 5) bootstrap: the task directory/manifest/lock were settled in the "detection" step → create the taskboard skeleton (only if missing; a re-entry never overwrites) →
#    fix the **task ID** (bootstrap-id) → wait until the ACK is byte-equal to the expectation (cmp) — that, and only that, is the "launch complete" criterion
TD="/tmp/$TAG"                             # the same task directory as the "first-run flow block"; reset here so this block can be lifted out and tested on its own
mkdir -p "$TD"                             # idempotent
SKILL_ABS=<absolute path of the skill, known to the sender>   # e.g. /home/<user>/.kimi-code/skills/trio-orchestrate; always an absolute path inside the package, never assume the peer expands ${KIMI_SKILL_DIR}
# ══ bootstrap ACK closure block (begin): lift this whole block out, set TD/TAG/SKILL_ABS/SLOTS, and it runs standalone (the peer's herdr can be stubbed) ══
CLASSIFY=${CLASSIFY:-new}                  # default for lifting this block out on its own: go as a "new launch" (never touches the manifest)
START_SLOTS=${START_SLOTS:-$SLOTS}         # same: default to every slot
# The task identity number has exactly **one** source, and three places must agree with it: the manifest's bootstrap-id line, the ACK expectation, and the sample handed to the peer.
#   new launch: the identity number = the CREATE_ID minted by the first-run flow block (already in the manifest); this block only reads it;
#   re-entry that really did start agents (refill/regroup): **explicitly mint a new identity number** — first rewrite the manifest's bootstrap-id line atomically, then rebuild the ACK expectation from it.
#     Never carry the old number over implicitly: the old ACK is exactly what the old number certifies, so keeping it means "two generations of members share one receipt", and a broken identity chain stays invisible.
#   re-entry that started nothing (REUSE=ok): keep the identity number from the manifest and do not resend bootstrap (everyone is on stage, the old ACK still holds).
if [ -n "$START_SLOTS" ] && [ "$CLASSIFY" = reentry ]; then
  TASK_ID=$(date +%s)-$RANDOM
  python3 - "$TD/manifest.txt" "$TASK_ID" <<'PY' || { echo "regrouping could not mint a new task identity number (manifest unwritable): fail-closed, manual confirmation"; exit 1; }
import os, sys
path, tid = sys.argv[1], sys.argv[2]
with open(path, encoding="utf-8") as f:
    lines = f.read().splitlines()
hit = [i for i, x in enumerate(lines) if x.startswith("bootstrap-id: ")]
if len(hit) != 1:
    sys.exit("manifest 的 bootstrap-id 行不是恰好一条:fail-closed,人工确认")
lines[hit[0]] = "bootstrap-id: " + tid
tmp = path + ".tmp"
with open(tmp, "w", encoding="utf-8") as f:
    f.write("\n".join(lines) + "\n")
os.replace(tmp, path)                      # rewritten in the same directory: either the whole old text or the whole new text, never a half-written file
PY
fi
[ -n "${TASK_ID:-}" ] || { echo "the task identity number is empty (neither a new launch nor a re-entry minted one): fail-closed, manual confirmation"; exit 1; }
[ -f "$TD/taskboard.md" ] || cat > "$TD/taskboard.md" <<'EOF'
# taskboard

## 待办(带验收线+冲突面)
## 在途(认领人+时刻)
## 待审
## 已验收(附证据路径)
EOF
# The line count follows N: the expected ACK = the bootstrap-id line + the tag line + N lines <slot>-<tag> (fixed slot order), with one trailing newline kept
NAMES=$(for s in $SLOTS; do echo "$s-$TAG"; done)      # the N full names, in fixed slot order
if [ -n "$START_SLOTS" ]; then
  rm -f "$TD/bootstrap-ack.md"             # drop the old ACK: only the receipt written this round counts (when the identity number changed, this also voids the old one)
  {
    echo "bootstrap-id: $TASK_ID"
    echo "技能文件:$SKILL_ABS/SKILL.md(绝对路径;先激活技能并读这个文件,不要依赖占位符自动展开)"
    echo "tag: $TAG"
    echo "看板:$TD/taskboard.md(骨架已建好)"
    echo "成员全名:$(echo $NAMES | tr ' ' '/')(编排 + 审阅 + 执行等;限 [机核] 的条目见看板规则)"
    echo "指令:读技能与看板后,把 ACK 写到 $TD/bootstrap-ack.md。ACK 必须与下面 $((N + 2)) 行**逐字节一致**(照抄、一行一项、不加别的字符、行尾不留空格、末尾保留一个换行):"
    echo "bootstrap-id: $TASK_ID"
    echo "tag: $TAG"
    for n in $NAMES; do echo "$n"; done
    echo "不要再执行拉起流程(不要建新 tab、不要起新代理)。"
  } > "$TD/bootstrap.md"
  # The full expected ACK (byte-identical to the $((N + 2)) lines handed to the peer above); acceptance is a full-text comparison:
  # an existence grep cannot stop "one field with two contradictory values / a duplicated line / an extra line" (2026-10-09 verdict), so the whole text is compared
  {
    echo "bootstrap-id: $TASK_ID"
    echo "tag: $TAG"
    for n in $NAMES; do echo "$n"; done
  } > "$TD/ack-expected"
  herdr agent prompt "orch-$TAG" "read $TD/bootstrap.md and follow its instructions" --wait --timeout 120000
  ack="$TD/bootstrap-ack.md"
  cmp -s "$TD/ack-expected" "$ack" || { echo "the ACK is not byte-equal to the expectation (contradictory/duplicated/extra/missing line): launch incomplete"; exit 1; }
else
  echo "everything reusable (no agent was started this round): bootstrap is not resent — the reuse check already verified the ACK in full under the same identity number"
fi
rmdir "$TD/launch.lock"                    # ACK passed = launch complete, release the exclusive lock; if it stopped midway, handle it by hand first, then delete the lock and re-enter
# ══ bootstrap ACK closure block (end) ══
```

The layout is always: the orch position on the left (main tree); non-orch members run down the right column in declaration order (N=3 means the reviewer top-right and the executor bottom-right, matching the existing layout pane by pane; right first, then down, independent of width). At N>5 the right column becomes hard to read, so splitting the task or adjusting panes by hand is advisable — **protocol identity anchors on the manifest, not on geometry**, and adjusting panes by hand does not affect the protocol.

- Model names follow each CLI's own `/model` list; **the model slug and the reasoning level must be passed separately** (e.g. codex `-m <slug> -c model_reasoning_effort=high`). Passing "GPT-6.1-Sol high" as a single model name is rejected by the API with a 400 (hit in practice 2026-10-09: `-m` did not complain at the time, and it only blew up on the first request); the slug follows this machine's `/model` list, and that model's default reasoning level may not be high, so it must be set explicitly; if `-m` is rejected, start the default model and pick the model + reasoning level manually with `/model` in the pane, and note it on the board — a manual rescue is fine, but it does not count as the normal "one-command / unattended" path;
- **Status semantics are unified throughout the document** (Step 0, Step 3 and the restart checkpoint use one set): **able to receive a new ticket = `idle` or `done`** (herdr's done is equally input-ready); `working` is never a criterion, only a supporting observation; `blocked` is a troubleshooting branch. **Members without hooks (such as genox) do not have this set of status semantics**, and use "no active output in pane read + the previous ticket's ACK already on disk" as the receiving condition, see "Backend profiles";
- Only a ready return from `agent start` counts; if it will not start, `herdr pane read` first to see where it is stuck (codex often sticks on an upgrade prompt — press Esc); **distinguish two kinds of launch failure**: parameter validation failure with no process created → fix the parameters and resend on the same shell pane; process alive but blocked/not-ready → troubleshoot first or wait for it to exit, never resend blindly;
- If the pane split succeeded but `agent start` failed (e.g. invalid_agent_name, in which case no process was created): the pane need not be rebuilt; fix the parameters and resend `agent start` on the same pane;
- Acceptance: `herdr pane read` each slot's pane; the review slot's status bar must show both `<model> <reasoning level>` segments, **then send one trivial test message to confirm no 400** before it counts as genuinely usable (looking at the displayed model name alone does not count); **claude-family** execution slots must show bypass permissions on (unattended parameter set only; other CLIs' unattended sets are the explicit parameter sets in their own profiles — "bypass" is claude's word, do not apply it to other backends) — **this only proves the permission mode, not that the deny list is mounted**. For **claude-family slots**, deny-list acceptance has two layers: the file existing and valid JSON is a **syntax check** (`python3 -c "import json;json.load(open('...'))"` verifies it in one line); the **effect check** is the orchestrator using `herdr pane read` or having that agent run `/status` (`/permissions`) and looking at the **settings sources and the effective rule list**, pasting the effective rule entries onto the board as evidence — a syntax check is not an effect check, they are two different things. For a **claude-family slot**, if no effective rules can be verified or the list has not taken effect → **pause dispatching**: stop that slot (exit/kill) and relaunch it with the attended set (explicit `--permission-mode default`), then re-verify mode and rules after the relaunch, and only resume dispatching once verified — changing a shell variable or writing a note on the board cannot touch the old process and does not count as a downgrade; for non-claude-family slots, verify the mode against the explicit parameters in their own profiles and what is actually displayed (e.g. codex's `-a on-request -c approvals_reviewer=user`), and stop if it cannot be verified;
- The completion criterion for bootstrap is **the ACK being byte-identical to `/tmp/$TAG/ack-expected` (`cmp -s`; contents = bootstrap ID, tag and the N member full names, N+2 lines)** — contradictory fields, duplicate fields, extra lines, missing lines and trailing spaces are all rejected, and it is not some status observation nor an existence grep: `agent get` is only a supporting observation — a short bootstrap is usually already idle/done by the time `--wait` returns, and waiting for `working` never arrives (an older version made working a hard condition, judged wrong on 2026-10-09); once the ACK is complete, tell the user to switch to the new tab and hand the task to the orch-slot member (that member is the subsequent orchestrator; this launching session only does the setup);
- Exception (**precondition: mode selection has passed; or a documented authorization was made per "Mode selection" — the user writing the authorized automatic-execution scope into `/tmp/$TAG/mode-selection.md`**; `--auto` = Never Ask, being present ≠ the user approving): if the user loaded this skill inside some kimi --auto session and wants that session itself to be the orchestrator, then do not create a new tab — **first register the agent identity with `herdr agent rename "$HERDR_PANE_ID" "orch-$TAG"`** (renaming a pane/tab is only a display label and is a different object from the agent name; changing the label alone does not register this session under a role name), then take the current pane as P0, split the remaining slots per the worktree/cwd rules above, start the remaining agents, and rename all panes as usual, **and also `herdr tab rename` the tab to the task name** (when that tab also holds unrelated panes, append `<TAG>` to the original name instead). This exception does not exempt mode selection: if the three gates fail and there is no documented authorization above, **disable the exception and stop work** (see "Mode selection"); rename only registers agent identity, does not change this session's CLI permission mode, and does not constitute user approval.

## Team assembly and substitutes (when the declared team is incomplete)

- **The orch slot is irreplaceable**: the skill runs inside kimi, but the binary check still has to run (being able to run in the current session ≠ there being a kimi on PATH that can start a new instance); substitute only when the missing capability is review/execution or similar;
- **Two-level availability check, both required**: ① `command -v <cli>` — the binary exists (a CLI on PATH that can start a new instance; being runnable in the current session does not count); ② `herdr integration status` shows `current` (the status hook is installed). A missing ② can be fixed with `herdr integration install <agent>`; if it cannot be fixed, downgrade to a spare tyre — it can start and run, but idle/working detection is unreliable, Step 3's `agent get` observation stops working, and only `pane read` by hand remains (**backends with no kind that herdr does not recognize, such as genox, are in this class — see "Backend profiles"**);
- **Candidates must be recorded**: each candidate states the CLI kind + provider/model + that slot's capability and permission boundary; "different backend" is judged on these two items and cannot be read off the binary name alone; the candidate pool is queried **by capability** (not by fixed role):

| Capability | First choice | Substitute order (examples; trust the two-level check as measured) |
|---|---|---|
| orch | kimi | none (irreplaceable) |
| rev | codex | copilot → opencode (install the hook first) → a second kimi |
| exec | claude | codex → copilot → opencode (install the hook first) → genox (attended set only) |

- **Hard constraint: rev-capability members and exec-capability members must not be same-origin** — a same-origin blind spot means a review by one of the other is no review at all. Same-origin judgement is in the pairwise constraint under "Slots and roster declaration" (different CLI and different model family);
- **Assemble by enumeration, not greedily**: give the candidate pool for each slot to be filled, enumerate **all combinations**, and take the first complete assignment satisfying [capability coverage + pairwise constraint + two-level health check + not the same live agent]. **The two-level health check branches by backend**: for a backend whose profile declares it **has** a status hook, check ① `command -v <cli>` binary and ② `herdr integration status` showing `current`; for a backend whose profile declares **no** hook (genox, see "Backend profiles"), ② is not required — check "that slot's pane is present, a process is alive inside it, and the on-disk ACK identity holds" instead, and **never call a genox slot unsolvable just because it lacks an integration**. Keep the existing counter-example lesson: "fixed first choice + greedy fallback down the list" stays banned — with only codex and kimi available, the legal solution is **kimi reviews + codex executes**; a greedy rule of "review has a fixed first choice, only step the execution choice down the list" would wrongly report no solution;
- Candidate health checks are at the same level as for a primary backend (binary + integration `current` + account/model availability); if enumeration still yields no legal team, **report clearly that the team cannot be assembled and stop; do not force one**;
- **Role names are unchanged, and they are slot names, not backend names**: `rev-<tag>`/`exec-<tag>` are slot names (protocol, taskboard and communication all unchanged), while the actual backend and kind are written into the board header; `--kind` takes the actual backend (the list herdr supports follows `herdr agent start --help`);
- Substitute CLIs have different no-confirmation parameters, so verify the mode after launch: generalized to **verify per CLI against its backend profile** (see "Backend profiles": one entry per CLI with the two-level health check, parameter sets, deny-mount capability and whether a status hook exists);
- **Deny-list mounting differs by CLI**: only the claude family has an equivalent mechanism for mounting a deny list via `--settings`; **when a non-claude execution slot has no equivalent mechanism, that slot's unattended capability is disabled along with it** — that slot switches to the attended parameter set, or gets a process-level sandbox of its own; **"just note on the board that the guard was dropped" and keep running as unattended is not allowed**.

## Step 1: put up the taskboard

`/tmp/<TAG>/taskboard.md` (TAG being the task tag fixed in Step 0), four columns: to-do (with acceptance line + conflict surface) / in flight (claimant + time) / awaiting review / accepted (with evidence paths). The rules go into the board header:

1. **Single writer: taskboard.md is written only by the orchestrator (the orch-slot member)**; working agents do not edit the board directly, they only write results into their own result file under `/tmp/<TAG>/` (e.g. `<task-id>-result.md`), and the orchestrator moves the column by pointer — several parties writing the same Markdown file overwrite each other;
2. Claiming writes your name, moving a column attaches evidence (test counts/hashes/file paths); words alone are not evidence;
3. **Evidence is bound to a code revision**: every ticket carries a task ID and a **base commit**; a receipt must carry the **tested commit** + artifact paths + checksums; review and merge check the same revision — if the code changes again (dirty worktree, HEAD moved) the old acceptance lapses and must be re-run and re-signed;
4. exec-capability members claim only tickets marked **[机核]** (machine-checkable acceptance line: test counts/hashes/gate scripts); their prose reports are not evidence;
5. rev-capability members pull from the "awaiting review" column and land their verdicts in /tmp with a pointer;
6. **Commit discipline (uniform across members)**: each line has its own index; `git add` only by task path, never `git add -A`; check `git diff --cached` before committing to confirm it contains only this task's changes, and never make a bare commit without looking at what is staged; the line prefix matches the actual slot: `[<slot>-<tag>]` (e.g. `[orch-g07]`, `[rev-g07]`, `[exec-g07]`); renames/moves must be completed atomically with the commit;
7. A full test suite is serialized on one slot; use `git worktree` double-buffering for multi-line construction;
8. A pre-registered number (write the expectation down before running) beats a post-hoc explanation;
9. **The composite slot `revtest` and the invalidity of self-review (taskboard routing)**: for a ticket the revtest member produced itself, **the review-signature column stays empty**, and the orch moves it straight to "accepted" after its own final verdict, annotating that line "**self-review invalid, orch final verdict**" — the single-writer rule is unchanged and the right to move columns stays with the orch alone.

**The "launch information" section in the board header (the roster table)**: besides the 9 rules above, the board header carries a roster table recording, per slot: slot / agent full name / backend CLI / provider+model / reasoning level / worktree path / parameter set (unattended or attended) — this is the single quick-reference table for re-entry, troubleshooting and "who is doing what".

## Unattended mode (pre-authorization for fully automatic collaboration)

When the user is absent and nobody clicks to confirm, modifications/deletions/git operations are **pre-authorized**. Separate three layers first: the **permission mode** (bypass) only saves confirmations, and **isolation** (a per-slot worktree) only separates working files and the index; neither is a **security boundary** (see "Known limitations"); the real hard boundaries are a process-level sandbox/container + restricted credentials + remote branch protection.

**Enabling gates (conditionally on, not on by default) — if any of the three is unmet, fall back to attended mode; do not start with the default permissive flags:**

| # | Must hold before enabling | Behaviour when it does not |
|---|---|---|
| 1 | Push hard boundary: remote main/master has **branch protection** set (or pushes use only restricted credentials) | **Ban pushes** for every exec-capability member; changes stay on local branches and the orchestrator pushes by hand |
| 2 | Credential surface: the runtime mounts no unnecessary credentials (or isolation via a separate Unix user/container) | Do not enable unattended mode |
| 3 | The directory is a git repo and worktrees can be created (a non-git directory has no usable isolation line) | **A non-git directory automatically falls back to attended mode** — this is a degraded path, not an equivalent substitute |

All three are **pre-work checks**, not after-the-fact explanations; if unverified, treat them as unmet. The per-item evidence and the landing of "all three gates pass → unattended parameter set / any failure → attended parameter set" are in Step 0's "**Mode selection**" — the launch parameters are chosen there and are not defaulted to permissive here. The attended set a failure falls back to **must be given explicitly per CLI profile** and its real mode verified: omitting bypass ≠ human approval, changing a variable in an existing session does not take effect either, and you must stop and relaunch.

**Deny-list mounting differs by CLI**: only the claude family has an equivalent mechanism for mounting a deny list via `--settings`; **when a non-claude execution slot has no equivalent mechanism, that slot's unattended capability is disabled unconditionally** (switch to the attended parameter set, or configure a process-level sandbox), and "just note on the board that the guard was dropped" and keep running as unattended is not allowed.

1. **One git worktree per non-orch slot** (a precondition only once gate 3 holds; created by the orchestrator in Step 0 before the agents start): each slot gets its own branch and index, so `commit -a`/`stash`/`reset` act only within that slot's working files — but see "Known limitations" item 3: git metadata is not separated;
2. Only orch-slot members merge/land into the main tree; other slots leave the main tree alone — this is a **disciplinary agreement** and cannot be enforced mechanically (the execution line has been measured modifying main-tree refs); the backstop is checking before committing plus acceptance;
3. Freedom to delete and modify is limited to your own worktree and under the `/tmp/<TAG>/` task directory; deleting or modifying anything outside still requires the user's confirmation while present — this is the **default authorized scope**; if the user explicitly chose "same-user cross-directory access" under gate 3 of "Mode selection", record it here as "the scope the user has chosen" and stop claiming simultaneously that cross-directory always confirms;
4. Push only your own slot branch and report it on the board; pushing to main/master and force-pushing are forever banned — but the **deny list is only a textual-prefix soft defence and can be bypassed by equivalent spellings** (measured forms are in "Known limitations" item 1); to make the ban real, set remote branch protection / use restricted credentials;
5. The deny list keeps only permanent bans (sudo/disk destruction/force-push/pushing main/reading secrets) — under unattended operation a long list only jams the batch; the real backstop is machine-checkable acceptance + worktree isolation + commit discipline (their boundaries are in "Known limitations").

## Known limitations (promises downgraded, stated honestly)

1. **The push ban is not a hard block**: `Bash(git push ...)` in `claude-exec-deny.json` is only a **textual prefix rule**, a soft defence that equivalent spellings can bypass; the bypass forms are distinguished by method (not lumped together as "measured") — `git push origin HEAD:main` (the local bare remote's main is actually rewritten) and `git push origin --force` are **real Git executions**, evidence `/tmp/skill-review/evidence.json`; `git push upstream main`, `+refspec` force-pushes, the `git -C <dir> push ...` wrapper and bare `git push` (which depends on upstream/refspec config) are only **text modelling of the prefix rule** (the command string compared against the rule list, with no real Git execution recorded); none of these forms is **measured inside the Claude permission engine**. `--force`, `-f` and `--force-with-lease` are three independent prefixes, and the list is incomplete. A genuine permanent ban can only come from **remote branch protection / restricted credentials / a controlled push entry point**, enforced on the parsed target ref and force semantics;
2. **Secret reading is covered only for direct reads inside cwd**: the `Read(./**/...)` rules cannot reach `~/.ssh/` or CLI credential directories, and cannot stop **indirect reads**: a subprocess (Python/Node) opening a file itself, or a search that names no file explicitly. Dangerous forms go into deny by **enumeration** (`.env.local`/`.env.production`/`.env.development`/`.env.staging`, `credentials.json`, `*.key`, `id_rsa`, etc.), which is why template names (`.env.example`/`.env.sample`/`.env.template`) are naturally not matched — permissions are evaluated deny → ask → allow, and **a broad deny beats a narrow allow, so do not count on a whitelist exemption**; enumeration is inherently incomplete, so an unattended environment **should not mount unnecessary credentials**; the read boundary must be enforced by process-level file permissions/a sandbox, and no configuration can claim to have enumerated "all secrets";
3. **A worktree is not a security boundary**: it isolates only working files and the index, while **git metadata (config/refs under the common dir) is shared with the main tree** — it was measured that the execution line's `git config` is read by the main tree, that `git update-ref refs/heads/main` can change the main tree's main ref, and that it can push to a local remote's main. So "only the orchestrator may touch the main tree" is a **disciplinary agreement that does not hold mechanically**; a hard boundary requires a process-level sandbox/container + a separate Unix user + restricted remote credentials, enabled after the user explicitly chooses that authorized scope;
4. **Substitute backends have no equivalent controls**: when claude is absent, `--settings` mounting is unavailable and substitute CLIs have no equivalent deny mechanism; that slot's unattended capability should then be disabled or switched to attended (see the last item of "Team assembly and substitutes");
5. **`/tmp` may be cleaned by the system**: the taskboard, ACKs and verdicts all live in `/tmp/<TAG>/`, and a reboot or a cleanup service will delete them; the key manifest, the final verdict and the recovery pointer must be saved elsewhere in a durable location inside the project (see item 1 of "Context/token discipline");
6. **A heartbeat cron belongs to its session**: scheduled tasks belong to the current session and are not inherited by a new session, so they must be recreated (see the last item of Step 3);
7. **Members without hooks have no status semantics**: neither herdr's agent kind list nor this machine's integration list contains genox (23 kinds / 17 integrations, triple-confirmed, evidence `/tmp/genox/T001-b-evidence/60-herdr-agent.stderr`, `/tmp/genox/T001-b-evidence/63-herdr-integration-install-help.stdout`, `/tmp/genox/T001-b-evidence/62-herdr-integration-status.stdout`), and `herdr agent read` on a genox pane reports `agent_not_found` outright (evidence `/tmp/genox/T001-b-evidence/87-agent-read.stdout`); only `herdr pane read` can read the screen (measured to work, evidence `/tmp/genox/T001-b-evidence/85-pane-read-visible.stdout`). A genox slot's idle/working is therefore a **screen inference**, weaker than a hook; the protocol's closed loop does not depend on it (it anchors on the on-disk ACK), but **dispatching timeliness is degraded**: glance at `pane read` before dispatching and before collecting the ACK, and let the on-disk ACK decide everything. `herdr pane wait-output --match/--regex --timeout` is a ready-made blocking primitive (measured `output_matched`, evidence `/tmp/genox/T001-b-evidence/89-wait-output.stdout`) that can be used to wait for the moment "it is time to move on".

Until the hard boundaries behind items 1–3 above (remote protection, credential isolation, worktree availability) are accepted, handle it per the **enabling gates** of "Unattended mode": fall back to attended mode, and do not default to permissive.

## Backend profiles

> One entry per backend CLI: **kind name / two-level health check / parameter sets (unattended, attended) / deny-mount capability / whether a status hook exists / entry-specific notes**. Step 0's "agent-start loop" looks parameters up here by the slot's cli; Step 0's "Mode selection" takes explicit parameters per CLI from the two-row table here. **CLI-specific experience with a raw CLI (such as network egress) is recorded here, while Step 3 keeps only general rules.**

**Parameter-set table (two rows — unattended / attended — × one column per profiled CLI)**:

| CLI | Unattended set (all three gates pass) | Attended set (any failure) | Deny mount | Status hook |
|---|---|---|---|---|
| kimi | `--auto` | no `--auto` (its semantics is Never Ask) | none | yes (see kimi in `integration status`) |
| codex | `--approve-for-me` | **explicit** `-a on-request -c approvals_reviewer=user` (inheritance is not accepted: omitting the flag ≠ human approval — this machine's user config is measured as `approval_policy = "never"`; the approver is separately determined by `approvals_reviewer`, so **on-request does not mean every command asks a human**) | none | yes |
| claude | `--dangerously-skip-permissions --settings ${KIMI_SKILL_DIR}/claude-exec-deny.json` | **explicit** `--permission-mode default` + `--settings` (deny only tightens, never loosens); from ≥2.1.283, not passing bypass may still inherit auto or the user's `permissions.defaultMode`, so **inheritance is not accepted**; verify the real mode after launch | **yes** (claude family only) | yes |
| ↑ note | when the roster contains **several claude-family slots** (e.g. `exec=claude` + `exec2=claude`), **every instance mounts the same `--settings` deny list**: the list is **mounted per instance**, not "bound to one particular slot" — the existing conclusion that it "binds only the `exec` slot and does not affect a user's own separately launched claude" is precisely the proof of per-instance mounting; when adding a claude-family slot, the launch loop still carries `--settings`, and none is left out | | | |
| ↑ note (codex) | **Network egress (limited to the author's environment, not a general fact)**: in the author's environment codex goes through a VPN while kimi/deepseek do not — so when only codex drops, check the VPN first; each person troubleshoots by analogy with their own egress situation (Step 3's re-delivery criterion uses this note) | | | |
| copilot / opencode / others | follow that CLI's `--help` as measured (**verify per CLI**, do not copy the values above) | same as left, explicit manual mode | none | present after `herdr integration install <agent>` |
| **genox** | **none** (see below) | interactive `genox chat` (the attended set is its only usable set) | **none** | **none** (herdr has no genox kind/integration) |

**Two-level health check (uniform across backends)**: ① `command -v <cli>` — the binary exists; ② `herdr integration status` shows `current`. **For a backend with no kind (genox), ② is necessarily missing**, so treat it as a spare tyre: `agent get` observation stops working and only `pane read` remains.

### The genox entry

**[Real execution] facts (all evidence under `/tmp/genox/T001-b-evidence/`, Genox 3.0.1)**:

- **Interactive command and flags**: the interactive launch command is written as bare **`genox`**; with the installed 3.0.1, bare `genox` reports `Missing command` (measured, exit code 2, evidence `05-genox-noargs.stderr`), so **use `genox chat` for now**; once the upstream PR lands, the bare command is equivalent to `genox chat` — **that equivalence is [not tested]**; flags only `--base-dir/-d`, `--verbose/-v`, `--help`; **chat has no no-confirmation parameter** (evidence `10-help-chat.stdout`);
- **The no-confirmation flags are on `genox run`**: `--approve-generated-plan` (generate a plan and approve it directly, without asking anyone) and `--plan FILE --approve-plan` (approve a given plan file); the source `run_preflight.py:292-318` makes clear it is `interactively_reviewed=False` and, with no TTY and no approval, reports "No TTY is available and the master plan is unconfirmed. Use --plan FILE --approve-plan or --approve-generated-plan." (evidence `72-approval-source-scan.txt`, `73-interactive-detection.txt`). **End-to-end unattended completion = [not tested]**: this machine has no external network, a TCP connect to `api.openai.com:443` immediately gives `Network is unreachable` (evidence `50-network-probe.txt`), and both paths die at the model call (evidence `41-run-approve-generated-plan.stdout`, `53-run-plan-file2.stdout`);
- **No-TTY behaviour**: `genox chat` with no TTY and `</dev/null` prints the banner and the `>` prompt, reads EOF and prints `Goodbye!`, **exit code 0, no hang** (evidence `30-chat-null-stdin.stdout`/`.exit`); in a pty it was measured reaching the interactive prompt and exiting cleanly on `/quit` (evidence `32-chat-pty.raw`); exit-code convention **0 normal / 2 usage error / 1 business rejection** (evidence `06-genox-badflag.exit`);
- **Process hygiene**: add **`--foreground`** to `genox run` to stop the coordinator from detaching (without it, it leaves the terminal); a failure may leave stale `.control.lock` metadata (measured: a lock file left by a dead pid, which **does not block subsequent starts** — three consecutive runs without unlock all started normally), and the cleaner is `genox unlock [slug] [--control]` (evidence `48-workspace-state.txt`, `49-lockpid-check.txt`, `10-help-unlock.stdout`);
- **Doctor posture**: `permission mode: workspace-write`, `allow_network: True`, `confirm_destructive: True`, bubblewrap available (mode auto), 76 built-in tools, mcp disabled; configuration follows **defaults + environment variables**, `~/.genox/` is cache only with no settings/permission config file (evidence `21-genox-doctor.stdout`, `20-genox-version.stdout`, `70-config-surface.txt`).

**Slot limits (insurance clauses while there is no status hook)**:

1. **It must not take the orch slot** (the exactly-one-orchestrator hard constraint): the orch collects ACKs, runs the three gates and dispatches, all relying on reliable status semantics, which genox does not meet;
2. **It may take rev / rev2 / exec / exec2…exec9 / revtest**, but:
   - its only parameter set is **attended** — **`genox run --approve-generated-plan` is recorded as a headless batch path [end-to-end not tested]**; backfill this profile once a usable path is measured;
   - **dispatching**: `herdr agent prompt --wait` is unavailable (no registered agent), so send a **short pointer** over the pane channel instead (the existing "panes carry only short pointers" rule already holds), and long text still lands on disk;
   - **status detection is degraded**: the orchestrator infers idle/working from the screen with `pane read`, **glancing once before dispatching and once before collecting the ACK**; `working` is never a criterion, and the on-disk ACK decides everything;
   - the "idle or done" part of the receiving condition lands, for a genox slot, as "**no active output in pane read + the previous ticket's ACK already on disk**", with no vagueness allowed;
3. **The heartbeat cron is hung for a genox slot as usual**, and deleted once the loop closes;
4. **Process hygiene**: genox runs in the foreground in a pane shell, so the restart sequence = exit inside the pane → **confirm with `pane read` that the shell prompt is back** → resend the command (the existing restart sequence's `herdr agent list` verification step does not apply to genox; confirm via pane read that the process is gone instead).

**Upgrade path (option b)**: write a genox integration hook for herdr (it requires understanding herdr's plugin mechanism and tracking upstream versions); once it lands upstream, this entry upgrades in place to the "has a hook" class, and the roster/naming/taskboard all stay unchanged. **Rejected option (c)**: mapping `agent start --kind` to the nearest kind — the kind determines herdr's driver protocol (prompt injection/status parsing), so a wrong kind breaks both send and receive, and it is a pseudo-registration that pollutes the registry.

## Step 2: layered dispatch (by judgement density, not by pane)

| Layer | Content | Dispatched to |
|---|---|---|
| A, judgement-dense (debugging/design/trade-offs) | the orchestrator's subagents (Agent tool, several in parallel) |
| B, implementation with a settled spec | the orchestrator's subagents, or exec-capability members + a strict spec |
| C, mechanical execution (running suites/regenerating/reading large files/table lookups) | exec-capability members |
| Review | rev-capability members |

**The criterion for dispatch tiering (fixed 2026-10-09)**: if the task brief still contains "how should this be designed" decisions (which implementation, how to split it, what threshold) = **layer A**, kept for the orchestrator's subagents — **it must not be marked [机核] and dispatched to exec-capability members**; [机核] marks only layer-C work whose method is already fixed and whose acceptance line is machine-checkable (commands, paths and expected values all complete, requiring no judgement from the executor). Writing design work as a [机核] order means making the executor take your design decisions, and no machine-checkable receipt can make up for that.

Quota discipline: reading large files and running long batches go to layer C. When a subagent reports 403, **split it by type** — quota wall / auth failure / network fault: on a quota wall, save progress first, back off and wait, then resume with `Agent(resume=...)` rather than restarting; **no infinite retries**; handle auth/network by their own causes; if the model/agent/parameters change, record it as a new execution condition and do not inherit the old acceptance automatically.

## Context/token discipline (context is a cache, the taskboard is the truth)

Automatic compaction only triggers near the ceiling (not moving at 70% is normal), and compaction itself costs quota — **accumulating less + externalizing state + restarting on schedule beats compaction**:

1. **Externalize all state**: task briefs, verdicts and batch results all land in `/tmp/<TAG>/`; any agent can be killed and restarted, and on restart it resumes from the **already-persisted checkpoint** (taskboard + pointer files + ACK/result files). Note that `/tmp` may be cleaned by the system: the key manifest, the final verdict and the recovery pointer must be saved elsewhere in a durable location inside the project;
2. **Write to disk in both directions**: not only dispatch orders, but agents' **outputs also land on disk** — a report may only return "conclusion + evidence path + key numbers", and pasting large files/logs back into the conversation is forbidden (paste once and it is carried along every later turn, burning quota);
3. **Slim prompts**: the pane carries only a short pointer + an explicit output format (conclusion first, a few lines at most), and however long the brief is, it is still just one path;
4. **Layer C is short-lived**: exec-capability members are **stateless executors** — restart after a batch of machine-checkable work (`/exit` then `agent start`), with no attachment to their context; their memory is the taskboard, not the session;
5. **Restart the review layer as needed**: rev-capability members' verdict history must live in the verdict files under `/tmp/<TAG>/` and resume from the persisted checkpoint on restart; restart after a long backlog or across tasks;
6. **Threshold patrol + restart state machine**: thresholds follow each CLI's current configuration (not written as a universal fact about all backends). The orchestrator's heartbeat checks the status bar with `pane read`; restart only at a safe checkpoint: confirm that agent is in a **receiving state (idle or done)** and that artifacts/uncommitted state are on disk (do not send `/compact` while working, do not interrupt a task) → send `/compact` (both claude and codex support it), and restart only if that fails. Restart sequence: `/exit` → `pane read` to confirm **the shell prompt is back** (this only means the screen is back at a shell, **not that the registry entry has been released**) → `herdr agent list` to verify the old name is gone from the registry → then `agent start`; when the process is alive but blocked/not-ready, troubleshoot first or wait for it to exit, and never resend blindly. **The difference for members without hooks (genox)**: there is no registry to check, so the `herdr agent list` step of the restart sequence becomes a `pane read` confirming the process has exited and the shell prompt is back (see clause 4 of the genox entry under "Backend profiles").

## Quota-exhaustion contingency (quota is a schedulable resource, not an incident)

Quota exhaustion (5h window/daily allowance) is a **routine external event** like a network drop; the principle is to absorb it, not to fight it:

1. **Detection goes into the heartbeat**: the orchestrator's `pane read` catches quota signals (codex status bar `5h limit: N% left · resets at HH:MM` — **this is an example from the author's environment, not a detection contract**: status-bar items are configurable and vary by version/auth method, so trust `/status` as measured; if it is not shown or cannot be read, mark it **unknown** and do not treat it as plentiful); a kimi 403 is only a candidate signal, and per the "quota discipline" item it is first split into "quota/auth/network" before entering the handling chain. One account may hold several limits at once (e.g. a 5h window plus a weekly one), so observe them separately. **<30% means noting it on the board and tightening dispatch** (dispatch only critical tickets; pause critical acceptance — this is our own scheduling policy, not a vendor-guaranteed safety threshold);
2. **Quota-pool independence must be verified, and cannot be inferred from the CLI name**: different CLIs may sit on one billing account (with a Claude API key or Copilot BYOK, two CLIs burn the same pool). Before substituting, check **provider + auth method + billing account/organization + quota bucket and window** item by item, and confirm the candidate is not bound by the exhausted bucket before calling it quota disaster recovery; switching clients alone does not count. The verification record contains no keys;
3. **Swapping brains works only because state is externalized**: with the taskboard + manifest + task IDs on disk, the substitute resumes from the **last confirmed checkpoint** — noting that the interruption scene may be "finished executing but not yet receipted", so before taking over, check the artifacts/processes/commits and confirm the old executor has stopped writing; necessary re-runs are registered on the board. The recovery promise is limited to "not repeating work already verified complete", and does not promise that any interruption is lossless;
4. **The contingency chain is by capability**: rev capability down → ① wait for the reset window ② borrow a review (from **another rev-capability member**, possibly across tasks/tabs, telling its orchestrator first and returning it on recall) ③ substitute review (labelled as degraded, **never as the final release signature**); exec capability down → swap the backend per "Team assembly and substitutes" (the rev/exec not-same-origin constraint still applies);
5. **The quality tier is not lowered to save quota**: final verdicts and critical gates keep their configured reasoning level; do not run critical acceptance on the last bus, and prefer waiting for a reset. (The author's single observation on 2026-10-09: at medium level with low remaining quota it still caught a deterministic bug — the bug's evidence is /tmp/skill-review/verdict-r5.md; the level and the remaining-quota number are what the orchestrator saw in the status bar at the time, **not archived and not independently verified, stated as-is**; this is a single observation, not proof that medium is generally sufficient; low has not been tested and is not used for final verdicts);
6. **Quota goes onto the board**: before dispatching a large ticket, record the current remaining amount, the quota bucket/window and the reset time (this is pre-registration); the consumption trend afterwards is retrospective data, and the two are recorded separately.

## Step 3: communication protocol (delivery and re-delivery criteria)

- **Every ticket carries a task ID** (T001/T002…, matching the board entry); before sending, confirm the **target slot's** agent is in a **receiving state: `idle` or `done`** (done is equally input-ready); do not send while `working` (that is the previous ticket running), and troubleshoot `blocked` first. **Members without hooks do not have this set of status semantics**, and their "receiving" is judged by "no active output in pane read + the previous ticket's ACK already on disk" (see the genox entry under "Backend profiles");
- Prefer sending with `herdr agent prompt <agent full name> "..." --wait --timeout <ms>` (with a bounded timeout); check the return for blocked/stalled/timeout. **Members without hooks (genox) do not take this path**; send a short pointer over the pane channel instead;
- **The criterion for acceptance/completion is the on-disk ACK/result file, and it must be bound to the task ID + that batch** (batch number/bootstrap ID); do not rely on a single status observation: `working` is only a **supporting observation** — a short task may finish before `agent get` (working is never seen), and when the other side was already working, seeing working does not prove the new ticket has started;
- Before re-delivering, check the ACK and the artifacts (result file, process, commit), and **the task must be idempotent** (safe to re-run), so a timeout re-delivery does not become duplicate execution;
- Long bodies/briefs **land on disk at /tmp/<TAG>/ and the path is sent**, with only a short pointer in the pane (the pane channel has been measured dropping long text);
- Troubleshoot network faults per each CLI profile's own note (e.g. the codex-over-VPN note is in "Backend profiles"); once it is confirmed to be a network problem and the previous ticket was definitely not executed, re-deliver per the flow above; if it is still unavailable → **borrow a review** (from another rev-capability member, telling its orchestrator first and returning it on recall);
- Hang a heartbeat cron for long tasks (avoiding :00/:30) and delete it once the loop closes; **a cron belongs to the current session and must be recreated in a new session**.

## Step 4: acceptance culture

- Fix-type changes require both pieces of evidence: red before, green after;
- The orchestrator personally re-checks the key numbers (re-running/re-computing), and does not relay them;
- "should be identical" ≠ "measured identical"; static claims and measurements are written separately;
- **The acceptance gate must contain behavioural probes**: for executable objects such as validation predicates, ACK comparisons and launch parameter sets, the gate must **construct adversarial samples and actually run them** — contradictory fields, empty values, duplicate lines, wrong paths, lock contention and leftover directories each need a sample that can be judged red/green; grepping a string does not count as acceptance (whether a string is present and whether a predicate judges correctly are two different things — the first four rounds of gates were all green on 2026-10-09, and the behavioural samples went red on the first run). The adversarial sample sets are in `/tmp/skill-review/` and `/tmp/skillfix/r*-evidence/`, ready to copy;
- **The gate script itself must also be verified**: once written, first prove it can go red and green; a pattern string starting with `-` must use `grep -qF -- "$3"` with a `--` separator, since misplaced arguments make the gate silently mis-check (the orchestrator made this mistake twice on 2026-10-09); after changing a gate, replay at least one sample that "the old implementation must fail";
- Words of your own that were overturned are kept as a correction record, honestly.

## Step 5: freezing and parallelism

- Freeze the repository during large compute batches (each fold spawns a new process reading the live tree); the orchestrator announces freezing/unfreezing;
- Real data scale exposes defects a demo cannot see (soft scoping, outer joins, typed chunking) — the acceptance evidence needs at least one copy at real-data scale.

## Launch script template

> Task: <one sentence>. I (the orch-slot member) orchestrate + accept; this team (roster): <list per slot: full name / backend / model+reasoning level / parameter set>.
> Taskboard: /tmp/<TAG>/taskboard.md. Discipline: prefixed commits, path-scoped add, serialized full suite, on-disk pointers.
> Tickets carry a task ID and a base commit; receipts are the on-disk ACK/result file that counts, with `working` only a supporting observation; members without hooks are judged by "pane read + on-disk ACK".

## Pre-release smoke checklist

Go through every item before sharing/releasing. Evidence has four tiers, replacing the vague [measured]: **[Real execution]** = the command/script was actually run, with input and result paths kept; **[Prefix modelling]** = the command string was only compared textually against the rule list/predicate, with no real run; **[Static inference]** = manual reasoning or a one-off enumeration, not fixed into a re-runnable record; **[Not tested]** = not yet run, merely a rule requirement. Protocol version: **2026-10-10-r6**.

> A row ending in **T001-d** means that row's probe is to be run by T001-d; **this table only states the requirement and does not claim execution before the evidence lands** — when you see T001-d and T001-d has not yet receipted, read it as [not tested].

| # | Check | Status and evidence |
|---|---|---|
| 1 | Parameter validation: **tag keeps its existing 8 cases**; **roster adds at least 6 cases**: empty string, containing a space, containing uppercase, containing a newline, illegal slot name, duplicate slot, N=1, out-of-range segment count — all rejected; and **the model gate runs ahead of any command containing $tag/$roster** | **[Real execution] T001-d** (tag's 8 cases reuse the 2026-10-09 evidence `/tmp/skillfix/r3-evidence/tag-validation.txt`, 1 accepted and 7 rejected; the roster's ≥6 cases and "the model gate comes first" are to be really run by T001-d). "The expanded value first goes through the model's reading judgement" is itself [not tested] (it needs a real call into the Kimi template engine fed an illegal `$tag`/`$roster`) |
| 2 | Re-entry with the same tag: **one reuse-check group each for N ∈ {2,3,4}**; r5's 6 sample criteria (realpath equality, missing-topology fail-closed, cross-tab, missing role) replayed sample by sample at N=2 and N=4 | **[Real execution] T001-d** (r5's original evidence `/tmp/skillfix/r4-evidence/reuse-realpath.txt`; exit-code criteria unchanged: 0 reusable / 2 missing members / 3 fail-closed. The replay after generalizing N is to be really run by T001-d); **partial team creation and failure rollback** [not tested] |
| 3 | Partial team-creation failure: a process not created can be resent on the same pane; a process alive but blocked/not-ready is troubleshot first | [Not tested] |
| 4 | No substitute solution: report no solution only after enumerating all candidate combinations | **[Real execution]** 2026-10-09 script: with review candidates = [codex,kimi] and execution candidates = [codex], the greedy rule wrongly reports no solution while enumeration yields kimi reviews + codex executes (script and output: `/tmp/skillfix/r3-evidence/role-pair-enum.txt`); the generalized enumeration **by slot candidate pool × all combinations** has not been exercised on a real machine [not tested] |
| 5 | deny negative case (B1): `git push origin HEAD:main`, `git push origin --force` | **[Real execution]** temporary Git commands really run: the local bare remote's main was actually rewritten (evidence `/tmp/skill-review/evidence.json`) |
| 5b | deny negative case (B1), remaining forms: `+refspec`, the `git -C` wrapper, `upstream main`, bare `push` | **[Prefix modelling]** the command string compared textually against the rule list, with no real Git execution recorded; an end-to-end negative case inside the permission engine is [not tested] |
| 6 | deny negative case (N1): the template name `.env.example` is matched by no deny rule (enumerate dangerous forms, do not rely on a whitelist) | **[Prefix modelling]** rule-intersection comparison (fnmatch, 2026-10-09); the final verdict inside the engine is [not tested] |
| 7 | Restart recovery: `/exit` → prompt → `herdr agent list` verifies the old name has been released → relaunch (**members without hooks use pane read to confirm the process has exited instead**) | [Not tested] |
| 8 | Model parameters: `-m <slug> -c model_reasoning_effort=high` takes effect (both status-bar segments + a test message that does not return 400) | Effective after the fix, pending verification [not tested]; the pitfall (the whole string as a model name → 400) **[Real execution]** 2026-10-09 |
| 9 | bootstrap: only an ACK **byte-identical (`cmp -s`)** to `/tmp/<TAG>/ack-expected` counts as "launch complete" — contradictory fields, duplicate fields, extra lines, missing lines and trailing spaces are all rejected; **green samples for N ∈ {2,3,4} plus red samples for each N**; task identity reads the manifest, and re-entry does a **unique parse**: each required field exactly one line with a non-empty value, a field name appearing twice being a rejection; adversarial samples for the manifest unique parse **on the roster line**: roster line missing / duplicated / mismatching the cwd slot set / cwd line count ≠ N; **replay ≥1 sample where "the old r5 parser must fail on a roster line"** | **[Real execution] T001-d** (r5's 8 sample criteria are kept and extended by N: 7 ACK samples `/tmp/skillfix/r4-evidence/cmp-ack.txt`, 8 manifest-unique-parse samples and 5 directory-safety samples `/tmp/skillfix/r5-evidence/manifest-unique-parse.txt`, `/tmp/skillfix/r5-evidence/dir-guard.txt`; the N generalization and the roster-line adversarial samples are to be really run by T001-d); a real-machine launch ACK is [not tested] |
| 10 | Dispatch status semantics: `done` can receive a ticket; the ACK is bound to the task ID + that batch; `working` is not a criterion; **a member without hooks has the receiving condition "no active output in pane read + the previous ticket's ACK already on disk"** | [Not tested] |
| 11 | Mode selection: three-gate evidence lands in `/tmp/<TAG>/mode-selection.md` → all pass uses the unattended parameter set, any failure uses the attended set (no bypass/auto sent to execution slots; **the attended set is given explicitly per CLI profile** — claude explicit `--permission-mode default`, codex explicit `-a on-request -c approvals_reviewer=user`, neither accepting an unaudited machine default by inheritance) | **[Static inference]** choosing between the two parameter sets by gate outcome is manual reasoning; expanding the parameter-set table per CLI is manual reasoning; launching once in each mode on a real machine is [not tested] |
| 12 | claude **effect check** of effective rules: `/status` (`/permissions`) to see settings sources and effective rules, with the entries pasted onto the board; if unverifiable → fall back to attended and **stop before relaunching with the explicit manual mode and re-verifying**; verify the real mode after launch (`pane read`/`/status`), and if the manual mode cannot be verified the gate has failed and dispatching stops | [Not tested] (the rule-intersection comparison in row 6 is [prefix modelling] and is not the same as effect inside the engine) |
| 13 | Parameter sets are bash arrays and `"${ARR[@]}"` expands to one argv each: a path containing spaces is not split and the Zsh scalar-no-word-splitting problem does not recur; an empty parameter set `=()` expands to nothing | **[Real execution]** 2026-10-09: bash 5.2.21 and zsh 5.9 × two modes × two paths (including `/tmp/skill space`), 8 samples in all — the array form is correct item by item on both sides; the old scalar string split into two items on a spaced path in bash and collapsed the whole group into one argument in zsh (evidence `/tmp/skillfix/r4-evidence/argv-arrays.txt`) |
| 14 | Attended-mode landing: give the manual set explicitly per CLI profile **and** verify the real manual mode after launch before resuming dispatch; codex's explicit approval policy/approver checked against the display with `pane read` (`on-request` only guarantees "asks when requested", not that every command asks a human); an existing session does not take launch parameters, so stop first and relaunch; the orch rename exception is narrowed to "mode acceptance has passed OR the user landed the authorized scope in `mode-selection.md`" (`--auto` = Never Ask, being present ≠ user approval) | [Not tested] (documentation basis: from claude `v2.1.283`, not passing bypass may still inherit auto or the user's `defaultMode`; this machine's codex user config `approval_policy = "never"` is a read-only narrow read) |
| 15 | **Classify before taking the lock** (N3): the full chain empty directory → classified "new" → lock taken → manifest written → ACK identical → released; **run the whole chain with an N=4 roster**; a leftover directory (non-empty, no manifest) is rejected before the lock is taken; a concurrent grab of the same TAG's lock is rejected; deleting the lock to force green is not allowed anywhere | **[Real execution] T001-d** (r5's original evidence: `/tmp/skillfix/r5-evidence/lock-flow.txt`, 4 branches + the gate-order probe; running the whole chain with an N=4 roster is to be really run by T001-d); a real agent launch is [not tested] |
| 16 | **Pairwise-constraint validator**: green = the current default trio; red = a same-CLI pair and a cross-CLI same-family pair; **the error wording matches the frozen text in the SKILL**; the gate script proves it can go red and green | **[Real execution] T001-d** (≥1 sample per class) |
| 17 | **Layout and naming**: static inference of the pane split order (right→down loop) and the rename loop for N=2/3/4 + **a pane-by-pane consistency check of N=3 against the r5 layout** | **[Static inference]** + a real-machine launch [not tested] |
| 18 | **genox slot launch**: the launch command really run; the no-confirmation path backfilled per `/tmp/genox/T001-b-result.md`; verifying that the "attended + pointer over the pane + on-disk ACK" loop works with a stub on the other side | **[Real execution] T001-d** (depends on T001-b); the part T001-b already established is in the genox entry under "Backend profiles"; **end-to-end unattended completion is still [not tested]** (this machine has no external network) |

## Appendix: the original author's experience (not universal facts; verify against your own environment)

> **Backend-specific experience (such as network egress) has moved into the "Backend profiles" section**, one entry per CLI.

- Quota: kimi/codex quota is tight, so reading large files and running long batches go to layer C as much as possible;
- Backend choice: which backend to hang on the exec slot is up to each user; a backend with unstable output gets only machine-checkable work;
- 2026-10-08 shared-index near-miss: `git add -A` swept away another line's staged changes;
- Measured pitfalls on 2026-10-09: an over-long tag reports `invalid_agent_name`; an earlier silent truncation made two tags collide (since removed); `-m "GPT-6.1-Sol high"` as a single model name — `-m` does not complain, and it only blows up with a 400 on the first request;
- The pane channel drops long text: the author's environment lost the same one four times in a day, hence the rule of always landing on disk and sending a pointer;
- r5 tasks wrap up under the r5 protocol and are not migrated to r6 naming (old manifests hold fixed names like `kimi-orch-*`, whereas from r6 they are `<slot>-<tag>`; re-entry would be judged fail-closed as "not this task" — that is the current semantics, with no special case).
