---
name: trio-orchestrate
description: "一键拉起并编排一组参数化编队的智能体协作(默认三角色 kimi 编排+验收 / codex 审阅 / claude 执行,编队由 roster 声明,可 2~N 成员,可选成员含 genox)。当用户要开始一项新的多智能体任务,或说'安排三方协作/开工/拉起评审执行/立看板'时使用。可带参数:/skill:trio-orchestrate <任务tag> <roster>;第一参数是任务 tag,第二参数是编队声明。依赖 herdr(终端多路复用)建面板、起代理、传话、盯状态。用户只需启动 kimi 并加载本技能,其余面板自动建好。"
arguments:
  - tag
  - roster
---

# 参数化编队协作:一键拉起 + 编排(默认 roster = kimi 编排 · codex 审 · claude 干)

> 作者:sheng luan
> 协议版本:**2026-10-10-r6**
> 依赖:kimi CLI + herdr;roster 里声明的其它 CLI 缺了走"组队与替补"。分享方式:整个 `trio-orchestrate/` 目录(含 `claude-exec-deny.json`)拷到对方 `~/.kimi-code/skills/` 即可。
> 本技能同时是**拉起协议**与**运行协议**。分享出去后若担心普通对话误触发建 tab/切模式,可按需在 frontmatter 加 `disableModelInvocation: true`,改为只由显式命令调用。

## 槽位与 roster 声明(角色由 roster 参数决定)

### 槽位词表(封闭,可枚举、可校验)

| 槽位 | 能力集 | 说明 |
|---|---|---|
| `orch` | {orch} | 编排+验收,**全 roster 唯一,硬约束** |
| `rev` | {rev} | 审阅 |
| `rev2` | {rev} | 第二审阅线(扩展) |
| `revtest` | {rev, exec} | 审阅+测试复合槽位(扩展) |
| `exec` | {exec} | 执行(C 层) |
| `exec2`…`exec9` | {exec} | 扩展执行线 |

**规则**:

1. **覆盖约束**:每个 roster 必须 orch ≥1、rev ≥1、exec ≥1;成员数 **N ≥ 2**(单人不成队)。缺能力即拒,**报错点名缺哪种能力**。
2. **必须单编排**(硬约束)。理由全部是现行纪律的直接推论,任一放松都会撕裂协议:看板单写者要求唯一写者,双 orch = 双写者;launch.lock 归属、manifest 创建/唯一解析、分类裁决需要一个唯一权威,双 orch 会出两种分类结论;模式选择是"唯一决策者"的三闸裁决,双 orch 会出现两套参数组结论;分层派活的 A 层裁决与派单权、ACK 收集与验收权都锚在唯一编排者上。**不设"多编排例外条款"**:现行"当前会话当编排者"例外只是 orch **实例来源**的例外(不新建 tab),不是 orch 数量的例外,r6 保持这个口径。
3. **复合槽位 revtest 的自审无效**:允许"审阅+测试"由同一成员承担,但该成员**对自己执行产出的审阅不计数**——该类任务的审阅签字必须来自异槽 rev 能力成员,或升格由 orch 终审(标注"自审无效、orch 终审")。orch 终审权本来就在(编排+验收),本条款只是把"自审无效"写明。

### roster DSL

调用入口:`/skill:trio-orchestrate <tag> <roster>`。roster 是**单字符串 DSL**,成员之间用 `+` 连接(不用空格,规避参数分词不确定性),每成员:

```
<slot>=<cli>[:<model>[:<effort>[:<family>]]]
```

- `slot`:槽位 ID,取自上面封闭词表;
- `cli`:后端 CLI 名(kimi/codex/claude/copilot/opencode/genox…),即 herdr `--kind` 或后端档案名;
- `model`:模型 slug,可选(kimi/claude 可省略用默认);
- `effort`:推理档(high/medium/low),可选;**与 model 分开传**的现行约定保留;
- `family`:模型族显式声明,可选,**缺省 = `model` 原串**(供同源判定,见下)。

**默认声明**:roster 为空 → 默认三成员编队

```
orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude
```

这是现行三件套的能力等价物,**保证旧调用方式行为不变**。

**完整示例**:

2 成员(编排+审阅/执行合一的最小队,适用于轻任务):
```
/skill:trio-orchestrate hotfix orch=kimi+revtest=codex:gpt-6.1-sol:high
```
展开:orch=kimi(主树) | revtest=codex gpt-6.1-sol high(独立 worktree;能力={rev,exec},自产出不可自审,终审归 orch)

3 成员(现行默认的 roster 写法):
```
/skill:trio-orchestrate genox orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude
```

4 成员(双执行线):
```
/skill:trio-orchestrate bench4 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude+exec2=genox
```
展开:orch=kimi | rev=codex gpt-6.1-sol high | exec=claude | exec2=genox(genox 槽位参数组与限制见"后端档案")

### 成对约束(声明式)

- **同源定义**(推广现行"不同后端"):成员 i、j 同源 ⇔ `cli_i == cli_j` **或** `family_i == family_j`(family 缺省 = model 原串)。即现行"不同后端 = 不同 CLI 且不同模型族"的等价形式,语义不变。
- **约束**:∀(i ∈ rev 能力成员, j ∈ exec 能力成员, i ≠ j):i 与 j **不得同源**。槽位维度按**能力**取——revtest 这类复合槽位以其 rev 能力参与 rev 集、以其 exec 能力参与 exec 集;i=j(自配对)不进来,由上面"自审无效"条款兜。
- **冲突报错措辞(写死,保证 B/C/D 三层口径一致)**:

  > 组队冲突:槽位 <slotA>(<cliA>/<modelA>) 与槽位 <slotB>(<cliB>/<modelB>) 同源(共享 <CLI 名|模型族 X>):审了等于没审。请更换其中一端的 CLI 或模型族,或在声明中给两端标注不同 family。

### 命名规则(多 tab 并存关键)

实际 agent 名 = **`<slot>-<任务tag>`**,如 `orch-g07tune`。原因:herdr 的 **agent 名是全局注册表,跨 tab/workspace 唯一**(实测撞名报 `agent_name_taken`;pane label 只是显示标签,不作身份);合法字符仅 `[a-z0-9_-]`、≤32。同一 tab 的成员共享同一 tag;**pane label 与 agent 名保持一致**。

**长度**:按 roster 实际槽位**逐个校验** `len(slot)+1+len(tag) ≤ 32` 且全名匹配 `^[a-z0-9_-]{1,32}$`;同时给默认词表的静态上限:词表最长槽位 `revtest`(7 字符)→ **tag ≤ 24**
(tag 上限 = 32(agent 名上限) − 1(连字符) − 7(最长槽位名 `revtest`);**超长一律拒绝,永不静默截断**——截断会让两个 tag 撞名,2026-10-09 实测)。

**复用检查必须用 JSON 精确匹配 N 个完整 agent 名**,不能用 `grep -- "-<tag>"` 这类子串匹配:`herdr agent list` 返回紧凑单行 JSON,grep 会把 `g070` 当成 `g07`、把别的任务整行算进来(2026-10-09 实测)。示例见"第 0 步"。别的 tab 的成员不算数,绝不跨任务复用。

## 第 0 步:拉起面板(全新面板组,独立 tab)

> 以下代码块是**概念流程**,不是可整段粘贴的脚本:逐步执行、**每步检查返回码**,任一前置失败立即停止(fail-fast)。面板 id 一律从命令返回的 JSON 里解析,不要手抄;示例见下。首次使用前先人工跑通一遍:各 CLI 登录、仓库 trust 确认、bypass 模式的首次确认,都不能放到无人值守时段做。**代码块语法一律按 Bash 写**(参数组用数组,`"${ARR[@]}"` 逐项成 argv;Zsh 对未引用标量不分词,行为不同,不要直接照搬到 Zsh)。

```bash
command -v herdr >/dev/null || { echo "未安装 herdr,停下告知用户"; exit 1; }
[ "${HERDR_ENV:-}" = 1 ] || { echo "不在 herdr 内运行,停下告知用户"; exit 1; }
herdr integration status           # 每个 agent 的状态钩子装没装:binary 在但钩子缺 → 状态侦测不可靠

# ── 模型闸(第一道,先于任何含 $tag/$roster 的命令,由编排者=模型执行)──────────
# $tag 与 $roster 在技能加载时已被文本展开进本文——下面那两行赋值本身就是 shell 源码:
# 校验写在赋值之后,挡不住"赋值之前"就随 $tag/$roster 进入源码的文本。真正第一道是
# 阅读判断:读到展开值后,先核对 TAG 整串符合 ^[a-z0-9_-]{1,24}$、ROSTER 整串符合下面
# 那条 DSL 正则(空成员/空段/::/大写/空格/换行一律非法);非法就停下找用户重取——不敲下面
# 任何一行,也不把非法值写进任何命令。这是模型闸,不是 shell 闸;不许"先赋值、再补校验"。
# roster 未传到(引擎只回传首参数,行为[未实测])→ 按 R2 降级:当面问用户要 roster 声明,
# 拿到的字符串过同一套模型闸 + shell 校验再继续,两个来源一视同仁,协议无分叉。
TAG=$tag                           # 调用参数:/skill:trio-orchestrate <tag> <roster>(必须带 skill: 前缀);为空则等用户下达任务时点名。定后整局不变,同时用作 tab 标签与看板目录 /tmp/<TAG>/
ROSTER=$roster                     # 第二参数;为空 → 用默认声明(见"槽位与 roster 声明")
[ -n "$ROSTER" ] || ROSTER='orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude'
# 二次防线(shell 层,挡的是赋值之后的继续使用;不能替代上面的模型闸):全字符串校验
# (2026-10-09 实测:$'g07\n../outside' 能穿过旧 grep -qE 的行锚定;bash [[ =~ ]] 全字符串则拒绝)
[[ "$TAG" =~ ^[a-z0-9_-]{1,24}$ ]] || { echo "TAG 不合法(空/含 [a-z0-9_-] 以外字符/超 24 字符/含换行),拒绝并要求用户重取"; exit 1; }
[[ "$ROSTER" =~ ^[a-z0-9]+=[a-z0-9_.-]+(:[a-z0-9_.-]+){0,3}(\+[a-z0-9]+=[a-z0-9_.-]+(:[a-z0-9_.-]+){0,3})*$ ]] \
  || { echo "ROSTER 不合法(空成员/空段/::/大写/空格/换行),拒绝并要求用户重取"; exit 1; }
# 该正则拒空成员(`+`连写)、拒空段(`=`后无值)、拒 `::`、拒大写与空格与换行;每成员最多 4 段(slot=cli:model:effort:family)

# ── roster 结构校验(解析后,python 一步做完,失败即停)────────────────────
# 依次:① 槽位名 ∈ 词表、同 roster 内不重复;② N ≥ 2;③ 能力覆盖 orch≥1/rev≥1/exec≥1;
# ④ 单编排(orch 恰好 1);⑤ 成对约束枚举(rev 能力 × exec 能力,不得同源);⑥ 每成员全名 ^[a-z0-9_-]{1,32}$ 且 ≤32;
# ⑦ cli ∈ 本机后端档案表(见"后端档案";未知 CLI → "无档案,先补后端档案再拉起")。
# 校验通过后输出规范化 roster 串(槽位序固定:orch 恒第一,其余按声明序),写进 manifest 的 roster: 行与 bootstrap,
# 保证重入时预期集合可重建。
ROSTER_NORM=$(python3 - "$ROSTER" "$TAG" <<'PY'
import re, sys
CAPS = {"orch": {"orch"}, "rev": {"rev"}, "rev2": {"rev"}, "revtest": {"rev", "exec"}, "exec": {"exec"}}
for _i in range(2, 10):
    CAPS[f"exec{_i}"] = {"exec"}                       # exec2…exec9 十个扩展执行线
KNOWN_CLI = {"kimi", "codex", "claude", "copilot", "opencode", "genox"}   # 本机后端档案表;扩充档案时同步这里
roster, tag = sys.argv[1], sys.argv[2]
full = re.compile(r"^[a-z0-9_-]{1,32}$")
members = []
for part in roster.split("+"):
    seg = part.replace("=", ":", 1).split(":")      # 先拆一次 `=`(seg[0] 里装的是 `slot=cli` 整串,不是 slot),再按 `:` 分段
    slot, cli = seg[0], seg[1]
    model = seg[2] if len(seg) > 2 else ""
    effort = seg[3] if len(seg) > 3 else ""
    family = seg[4] if len(seg) > 4 else model      # family 缺省 = model 原串
    if slot not in CAPS:                            # 未知槽位:点名拒绝(中文报错,不让 KeyError 栈回溯冒出来)
        sys.exit(f"未知槽位名 {slot!r}:槽位取自封闭词表 orch/rev/rev2/revtest/exec/exec2…exec9")
    members.append(dict(slot=slot, cli=cli, model=model, effort=effort, family=family, caps=CAPS[slot]))
# ① 槽位不重复
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
# ④ 单编排
if sum(1 for m in members if "orch" in m["caps"]) != 1:
    sys.exit("必须单编排:orch 槽位恰好一个")
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
) || { echo "roster 结构校验不过:停下,按上面报错整改后再拉起"; exit 1; }
echo "规范化 roster:$ROSTER_NORM"

# ── 任务目录:先分类后取锁、先定路径方案;不无条件覆盖 ──────────────────────
# 次序(2026-10-09 裁决 N3):建目录 → 目录/文件安全核验 → 定路径方案 → 分类(新建/重入/恢复)
#   → 分类通过才取排他锁 → 建/核对 manifest。
# 不能先取锁再分类:新取的锁自己让目录非空,合法新建会被"非空且无 manifest"判死(自阻塞)。
# 排他:一个 TAG 同时只许一个拉起流程(并发共用 ACK 文件本身不允许);锁在拉起完成(ACK 通过)后释放,
# 中途失败停下的先别删锁——人工处理完再删锁重入,防误重入
# ══ 首次流程块(起;协议版本 2026-10-10-r6):把本块整段抽出去、设好 TAG/ROSTER/ROSTER_NORM 与 PWD(项目根 cwd)即可单独跑 ══
TD="/tmp/$TAG"                                          # 任务目录(状态目录);本块一律用 $TD
mkdir -p "$TD"                                          # 幂等

# 目录/文件安全:读写的每个路径都查——不是符号链接(`[ -L ]`)、属主是自己;不存在的视为可创建
guard() {
  [ -e "$1" ] || [ -L "$1" ] || return 0
  [ -L "$1" ] && { echo "拒绝:$1 是符号链接(可能指向任务目录外部)"; return 1; }
  [ "$(stat -c %u -- "$1" 2>/dev/null)" = "$(id -u)" ] || { echo "拒绝:$1 属主不是当前用户"; return 1; }
  return 0
}
for p in "$TD" "$TD/manifest.txt" "$TD/taskboard.md" "$TD/bootstrap.md" "$TD/bootstrap-ack.md" "$TD/ack-expected" "$TD/mode-selection.md" "$TD/panes.txt"; do
  guard "$p" || { echo "任务目录/文件安全核验不过:停下人工确认"; exit 1; }
done

ROOT=$(realpath -m "$PWD")                              # 项目根规范绝对路径:任务身份的锚
# 槽位序固定:orch 恒第一,其余按 roster 声明序 —— manifest 的 cwd 行、ACK 行、panes.txt 一律照此序
SLOTS=$(python3 -c 'import sys;print(" ".join(p.split("=")[0].split(":")[0] for p in sys.argv[1].split("+")))' "$ROSTER_NORM")
N=$(echo $SLOTS | wc -w)
SLOT_ARR=($SLOTS)                                       # 槽位固定序,可按下标取(槽位名只含 [a-z0-9_],分词安全)
declare -A CLI_OF                                       # 槽位 → 后端 CLI:起代理循环按 CLI 分流(无钩子后端走 pane 直起)
while read -r _s _c; do CLI_OF[$_s]=$_c; done < <(python3 - "$ROSTER_NORM" <<'PY'
import sys
for part in sys.argv[1].split("+"):
    seg = part.replace("=", ":", 1).split(":")
    print(seg[0], seg[1])
PY
)
declare -A EXPECT_OF                                    # 槽位 → 预期 cwd:新建/重入两条路都必须填满,复用检查与裂 pane 一律从这里取
NO_HOOK_CLI="genox"                                     # 档案声明"无状态钩子"的后端:不进 herdr agent 注册表(见"后端档案")
HOOKLESS=""
for s in $SLOTS; do
  case " $NO_HOOK_CLI " in *" ${CLI_OF[$s]} "*) HOOKLESS="$HOOKLESS $s=${CLI_OF[$s]}";; esac
done
HOOKLESS=${HOOKLESS# }                                  # 无钩子槽位(槽位固定序),每项记成 <slot>=<cli>:复用检查要拿该槽位的 CLI 名核对 pane 前台进程名,只记槽位名不够
# 路径方案在生成 manifest 之前定死(不是先算兄弟 worktree、再拒正文允许的非 git 降级):
#   git 仓库 → orch 槽位 cwd = 主树,每个非 orch 槽位各一条兄弟 worktree;非 git → 全部 N 个 cwd 都记主树(只允许在有人在场模式下)
if git -C "$PWD" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  PATH_PLAN=worktree
else
  PATH_PLAN=main-only
fi
# 逐槽位预期路径:git 方案下 orch=主树、其余 ../wt-$TAG-<slot>;非 git 方案下全部=主树
EXPECT_MAIN=$(realpath -m "$PWD")
expect_of() {                                            # $1 = slot
  if [ "$PATH_PLAN" = worktree ] && [ "$1" != orch ]; then
    realpath -m "$PWD/../wt-$TAG-$1"
  else
    echo "$EXPECT_MAIN"
  fi
}

# ① 分类:只读判断,先不写任何东西
if [ -s "$TD/manifest.txt" ]; then
  CLASSIFY=reentry
elif [ -n "$(ls -A "$TD" 2>/dev/null)" ]; then
  echo "$TD 非空且无 manifest(残留、并发拉起中的锁、或别的任务):停下人工确认,不自动清理"; exit 1
else
  CLASSIFY=new                                          # 空目录 → 合法新建
fi
# ② 分类通过,才取排他锁
mkdir "$TD/launch.lock" 2>/dev/null || { echo "已有拉起流程在跑(launch.lock):不并发拉起,停下"; exit 1; }

# ③ 按分类建 manifest / 核对任务身份
if [ "$CLASSIFY" = reentry ]; then
  # 重入:manifest **唯一解析**——必需字段(tag / roster / project-root / created / bootstrap-id)
  # 每条恰好一行、值非空;同名字段出现两次即拒(矛盾值不算"逐字段一致");
  # cwd 行数 == N、cwd 槽位集合 == roster 行的槽位集合、逐槽位 realpath 比对。
  # N 从 roster 行读出(不用 shell 二次解析);预期路径由本进程按 path plan 逐槽位算出后传入,
  # 两侧都过 realpath 规范化再比。stdout 只出 N 条已验路径,按槽位固定序,供 shell 逐行取回。
  _eargs=()                                             # 槽位/预期路径成对进**引用的数组**:含空格的路径整条是一个 argv,不被拆开
  for s in $SLOTS; do _eargs+=("$s" "$(expect_of "$s")"); done
  _mout=$(python3 - "$TAG" "$ROOT" "$TD/manifest.txt" "$SLOTS" "${_eargs[@]}" <<'PY'
import os, sys
tag, root, path = sys.argv[1:4]
slots = sys.argv[4].split()                             # $SLOTS 传进来是空格分隔的**字符串**:必须 split 成 list,否则 list 比 string 恒不相等
rest = sys.argv[5:]
if not slots or len(set(slots)) != len(slots):
    sys.exit(f"槽位表非法(为空或有重复):{slots}")
if len(rest) != 2 * len(slots):
    sys.exit(f"槽位与预期路径不是成对的 N 组:槽位 {len(slots)} 个、参数 {len(rest)} 项")
want = dict(zip(rest[0::2], rest[1::2]))
if set(want) != set(slots) or any(not p for p in want.values()):
    sys.exit("槽位→预期路径映射不完整或含空值")
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
def one(k):                                             # 必需字段恰好一行;重复即冲突
    vs = seen.get(k, [])
    if len(vs) != 1:
        sys.exit(f"manifest 字段 {k} 应恰好一行(实得 {len(vs)})")
    return vs[0]
if one("tag") != tag: sys.exit("tag 不符")
if one("project-root") != root: sys.exit("project-root 不符")
one("created"); one("bootstrap-id")                     # 存在、恰好一行、值非空
roster = one("roster")                                  # roster 行:必需、恰好一行、非空
rslots = [p.split("=")[0].split(":")[0] for p in roster.split("+")]
if rslots[0] != "orch": sys.exit(f"roster 行槽位序不合法(orch 必须第一):{roster!r}")
if rslots != slots:
    sys.exit(f"roster 行槽位集合与本次不符:{rslots} ≠ {slots}")
cwds = seen.get("cwd", [])
if len(cwds) != len(slots):
    sys.exit(f"cwd 行数应 == N = {len(slots)}(实得 {len(cwds)})")
got = {}
for v in cwds:
    name, sep, p = v.partition(" ")
    if sep != " " or not p.strip(): sys.exit(f"cwd 行缺分隔符或路径为空:{v!r}")
    if not name.endswith("-" + tag):
        sys.exit(f"cwd 行成员名 {name!r} 不符 <slot>-{tag} 命名:非本任务")
    slot = name.rsplit("-" + tag, 1)[0]                 # manifest 记的是全名 <slot>-<tag>:归一成裸槽位再比对(两侧口径不同会恒判不符)
    if not slot: sys.exit(f"cwd 行成员名 {name!r} 的槽位为空")
    if slot in got: sys.exit(f"cwd 槽位 {slot} 重复")
    got[slot] = p.strip()
if set(got) != set(slots): sys.exit("cwd 槽位集合与 roster 行不符")
if [k for k in got] != slots: sys.exit("cwd 行序与槽位固定序不符")
for name in slots:
    if os.path.realpath(got[name]) != os.path.realpath(want[name]):
        sys.exit(f"{name} cwd={got[name]} ≠ 预期 {want[name]}(realpath 规范化后仍不同)")
print("manifest 唯一解析通过:字段无重复、值非空,roster 行与 N 个槽位 cwd 与本次一致", file=sys.stderr)
for name in slots:
    print(got[name])                                    # stdout 只出 N 条已验路径(槽位固定序),供 shell 逐行取回
PY
  ) || { echo "manifest 与本次不符(字段重复/空值/项目根/roster/槽位路径/缺路径):不识别为本任务,停下人工确认"; exit 1; }
  # 预期 cwd 由上面同一解析器同一套算法验过(重入不重新拼路径方案、也不二次解析——awk 二次解析的字段口径
  # 曾与 manifest 行格式脱节,三条 EXPECT 被清空,2026-10-09 N4);逐行取回,r5 同款 IFS= read -r,**禁用 eval**:
  # 路径是数据不是 shell 源码(可含空格、反斜杠、字面 $()),回读只许原样进数组;并防空值、防多行/少行
  i=0
  while IFS= read -r _line; do
    [ -n "$_line" ] || { echo "重入取回的槽位 cwd 为空(第 $((i+1)) 行):停下人工确认"; exit 1; }
    [ "$i" -lt "$N" ] || { echo "重入取回的路径多于 N=$N 行:停下人工确认"; exit 1; }
    EXPECT_OF[${SLOT_ARR[$i]}]=$_line                   # 槽位固定序逐行落位(r5 的三个 EXPECT_* 的泛化位)
    i=$((i+1))
  done <<< "$_mout"
  [ "$i" = "$N" ] || { echo "重入取回的路径 $i 行、应 $N 行:停下人工确认"; exit 1; }
  EXPECT_MAIN=${EXPECT_OF[orch]}                        # orch 槽位 cwd 即主树路径(非 git 方案下全部槽位都是它)
  echo "重入:manifest 唯一解析一致,复用已有看板,只补缺、不覆盖"
else
  CREATE_ID=$(date +%s)-$RANDOM                         # 建目录批次号:任务身份的一部分,只在新建时取
  {
    echo "tag: $TAG"
    echo "roster: $ROSTER_NORM"
    echo "project-root: $ROOT"
    for s in $SLOTS; do                                  # N 条,槽位序固定(orch 恒第一)
      echo "cwd: $s-$TAG $(expect_of "$s")"
    done
    echo "created: $(date -Is)"
    echo "bootstrap-id: $CREATE_ID"
  } > "$TD/manifest.txt"
  for s in $SLOTS; do EXPECT_OF[$s]=$(expect_of "$s"); done   # 与重入分支同款:新建分支也要把逐槽位预期 cwd 填满
                                                              # (r5 把三个 EXPECT_* 写在两分支共用处;r6 泛化时只留在重入分支,2026-10-09 T001-d D4)
  echo "新建:任务目录已初始化(路径方案 $PATH_PLAN;锁已上,ACK 通过后释放)"
fi
# ══ 首次流程块(止)══

# 复用检查:JSON 精确匹配 N 个完整 agent 名;再逐槽位核对其 cwd == 该槽位**预期 cwd**(git 方案=该槽位 worktree;
# 非 git 降级=主树)——预期值一律来自上面 manifest
# (不是要求各槽位 cwd 相等——同 tab 但错跑到别的目录,一样不能复用);
# 两侧都过 realpath 规范化后比较——带 .. 的拼法与规范 cwd 指向同一目录时必须判等,否则正常重入会被误判;
# tab_id/workspace_id 任一缺失即非零退出(fail-closed),不拿"能打印出结论"当通过
# 无钩子槽位($HOOKLESS,每项 <slot>=<cli>;档案声明不进 herdr 注册表,如 genox)豁免 agent-list 存在性,改验三件:
#   ① pane 身份:panes.txt 记的 pane 仍在,label == <slot>-<tag>,同 tab,cwd 与预期一致;
#   ② pane 进程存活:前台进程非空,且进程名 == 该槽位声明的 CLI(不在 = 缺,只补这一格);
#   ③ 落盘 ACK 身份:bootstrap-ack.md 与本次 ack-expected 全文全等(不是本任务的 ACK 不认,防跨任务复用)
# 预期路径用**引用的 bash 数组**逐项成 argv 传(含空格的路径不被拆);解析器先断言"槽位数 == 路径数、非空、唯一"再比对。
herdr agent list > "$TD/agent-list.json"
herdr pane list  > "$TD/pane-list.json"
for _hs in $HOOKLESS; do
  s=${_hs%%=*}                                          # $HOOKLESS 每项是 <slot>=<cli>:取槽位名去查 panes.txt 与落 pane-info
  p=$( [ -s "$TD/panes.txt" ] && awk -v s="$s" '$1 == s {print $2}' "$TD/panes.txt" )
  [ -n "$p" ] || continue                               # 没有 pane 记录 → 交给解析器按"缺成员"报(新建时本来就没有 pane)
  herdr pane process-info --pane "$p" > "$TD/pane-info-$s.json" \
    || { echo "无钩子槽位 $s 的 pane process-info 取不到:fail-closed,人工确认"; exit 3; }
done
EXPECT_ARGS=()
for s in $SLOTS; do EXPECT_ARGS+=("${EXPECT_OF[$s]}"); done
python3 -c '
import json,os,sys
tag=sys.argv[1]; slots=sys.argv[2].split(); hookless=sys.argv[3].split()
td=sys.argv[4]; paths=sys.argv[5:]
if not slots or len(set(slots))!=len(slots) or len(paths)!=len(slots) or any(not p for p in paths):
    print("复用检查入参不完整:槽位 %d 个、路径 %d 个(fail-closed,不复用)"%(len(slots),len(paths))); sys.exit(3)
hook=dict(h.split("=",1) for h in hookless)
agents=json.load(open(os.path.join(td,"agent-list.json")))["result"]["agents"]
panes=json.load(open(os.path.join(td,"pane-list.json")))["result"]["panes"]
expect={s:p for s,p in zip(slots,paths)}
def norm(p):
    return os.path.realpath(p) if p else p
def bad(msg):
    print(msg); sys.exit(3)
def pinfo(s):
    f=os.path.join(td,"pane-info-"+s+".json")
    if not os.path.exists(f): return {}
    return json.load(open(f)).get("result",{}).get("process_info",{})
got={a.get("name"):a for a in agents}
missing=[s for s in slots if s not in hook and (s+"-"+tag) not in got]
if missing:
    print("缺: "+", ".join(s+"-"+tag for s in missing)); sys.exit(2)
by_label={p.get("label"):p for p in panes}
tabs=set()
for s in slots:
    if s in hook: continue
    w=s+"-"+tag; a=got[w]
    if not a.get("tab_id") or not a.get("workspace_id"):
        bad(w+" 缺 tab_id/workspace_id,fail-closed:不复用")
    if norm(a.get("cwd"))!=norm(expect[s]):
        bad(w+" cwd="+str(a.get("cwd"))+" ≠ 预期 worktree="+str(expect[s])+"(realpath 规范化后仍不同):不复用")
    tabs.add(a.get("tab_id"))
for s in sorted(hook):
    w=s+"-"+tag; pane=by_label.get(w)
    if pane is None:
        print("缺: "+w+"(无钩子槽位:panes.txt 记的 pane 不在 list 里,按只补缺在该 pane 重起)"); sys.exit(2)
    procs=pinfo(s).get("foreground_processes") or []
    if not procs:
        print("缺: "+w+"(无钩子槽位:pane 内无前台进程,按只补缺在该 pane 重起)"); sys.exit(2)
    p0=procs[0]; pname=p0.get("name") or ((p0.get("argv") or [""])[0])
    if pname!=hook[s]:
        bad(w+" pane 前台进程="+str(pname)+" ≠ 该槽位 CLI="+hook[s]+":fail-closed,人工确认")
    if norm(p0.get("cwd"))!=norm(expect[s]):
        bad(w+" pane 进程 cwd="+str(p0.get("cwd"))+" ≠ 预期 worktree="+str(expect[s])+"(realpath 规范化后仍不同):不复用")
    if not pane.get("tab_id") or not pane.get("workspace_id"):
        bad(w+" pane 缺 tab_id/workspace_id,fail-closed:不复用")
    tabs.add(pane.get("tab_id"))
if len(tabs)!=1:
    bad("成员不在同一 tab: "+", ".join(sorted(str(t) for t in tabs)))
if hook:
    ack=os.path.join(td,"bootstrap-ack.md"); want=os.path.join(td,"ack-expected")
    ok=os.path.exists(ack) and os.path.exists(want) and open(ack,"rb").read()==open(want,"rb").read()
    if not ok:
        bad("无钩子槽位复用的落盘 ACK 身份不成立(bootstrap-ack.md 与本次 ack-expected 不全等):fail-closed,人工确认")
print("%d 名成员齐全:同 tab,各自 cwd 与预期 worktree 一致(realpath 规范化后);无钩子槽位 pane 进程存活且落盘 ACK 身份已核"%len(slots))' \
  "$TAG" "$SLOTS" "$HOOKLESS" "$TD" "${EXPECT_ARGS[@]}"
# 退出码:0 = 可复用;2 = 缺成员(只补缺的;无钩子槽位 = pane 内无进程);
# 3 = 拓扑/目录不一致、缺字段,或无钩子槽位 ACK 身份不成立(fail-closed,不自动复用,人工确认后重建)
# 预期 cwd 一律取自 manifest:新建时按"首次流程块"定好的路径方案写死(git→主树+N−1 条兄弟 worktree;
# 非 git→N 个 cwd 都记主树,只允许在有人在场模式下),重入时经唯一解析取出——这里把 N 个预期路径传给复用检查
```

### 模式选择(检测之后、拉起之前,必做)

**启动参数是模式选择的结果,不是默认值;没做模式选择就启动 = 流程违规。** 任务目录、manifest(项目根/各槽位预期 cwd/roster/创建时间/bootstrap ID)与排他锁已在上面"检测"一步办妥;本步按"无人值守模式"启用闸三条逐项核验,每条都在 `/tmp/$TAG/mode-selection.md` 落证据(写不清证据 = 该闸不过),看板建好后把该文件指针贴进去:

| 闸 | 核验对象(不是只核"有个保护标志") | 证据写什么 |
|---|---|---|
| 1 推送边界 | ① **执行身份本身无 bypass 权限**(远端管理员直推、保护规则的 bypass 名单都要核);② 目标 ref(main/master)被保护规则实际覆盖;③ 其他允许推送的分支也不能 force-push | branch protection 实际设置(如 `gh api repos/<owner>/<repo>/branches/main/protection` 看 `enforce_admins` 与规则),或凭据侧只读证据;核不实 → 不过 |
| 2 凭据面 | **按成员进程列表逐成员核验**:每个成员 agent 全名 + 进程(argv/pid)+ 必需凭据(只留各 CLI 自身登录态)+ 可读敏感路径 + 隔离手段(独立 Unix 用户/容器/不挂载) | 最小可核对的隔离配置 + 挂载清单;容器存在 ≠ 没挂敏感数据,独立用户名 ≠ 文件不可读,逐项写 |
| 3 目录与权限范围 | **worktree 可用性与进程权限范围分开验收**:worktree 只隔离工作文件与索引,同一 Unix 用户跨目录写不受它约束(见"已知限制"第 3 条)。git 方案下 worktree 数 = N−1(orch 用主树,每个非 orch 槽位一条);非 git → 全部 cwd=主树 | ① 是 git 仓库且能建 N−1 条 worktree;② 进程写入边界:有容器/独立用户等受限范围,**或用户明确选择"允许同用户跨目录访问"这一授权范围**(选了就把无人值守第 3 条改述为该已选范围,不再宣称跨目录必确认);核不到受限范围又没明确选择 → 不过 |

三闸全过 → **无人值守参数组**;任一不过 → **有人在场参数组**,并在看板写明是哪条闸、什么证据不过。**两行参数组按 CLI 排列的实体表在"后端档案"节**;roster 里每个成员的启动参数按该成员的 CLI 去那张表取,不在这里写死:

失败分支不是"照旧带放权参数、只在看板记一句":

- 闸 1 不过 → **每个 exec 能力成员禁 push 要用能力落实**:该成员实例不持有 push 凭据,或远端只给线分支受限写权限;只在看板写"禁 push"不算;
- 闸 2 不过 → **不发出 bypass/auto 的启动指令**,改用有人在场参数组(按各 CLI 档案显式给出,见"后端档案");
- 闸 3 不过(非 git / 无受限权限范围)→ 同上,退回有人在场模式,由人兜底。

**有人在场组是"显式人工模式 + 验收",不是"省掉 flag":**

- 按该 CLI 档案的有人在场参数组**显式给出**,不接受继承本机默认(理由与实测见"后端档案"对应 CLI 条);启动后**验收真实模式**:`herdr pane read <该槽位 pane>` 看状态栏或让该 agent 跑 `/status` 确认当前是人工批准(不是 bypass、也不是 auto),证据落 `mode-selection.md`;**验不出人工模式 = 闸失败,停止派单**,先排障或先停再重来,不许带未知模式派活;
- **既有会话不适用启动参数**:权限模式是进程级属性——改 shell 变量、改看板记录都不改变已在跑的进程。闸失败时若目标代理已在跑(尤其已在 bypass/auto 里),**必须先停(退出/杀掉)再按人工模式重拉**,重验模式后才恢复派单,不许"改个变量就当降级了";
- orch 槽位例外条款收口(第 0 步末条"当前会话当编排者"那个例外):`--auto` 的语义是 **Never Ask**——不打断用户、自动执行并自动决定;用户坐在终端前 = 有人在场,但**在场≠用户批准**,rename 只注册身份,既不改 CLI 权限模式、也不让 CLI 在动作前等人点头。当前会话已在 `--auto` 里跑、又过不了三闸时,只允许二选一:**(a) 记录式收窄(不是等价人工审批)**:用户把授权范围明确写进 `/tmp/$TAG/mode-selection.md`(授权哪些操作可自动执行、监督与止损方式),该分支按"**已授权自动执行 + 人类监督**"定性执行并如实标注,不再自称与人工审批等价;**(b) 没有这份记录 → 停工**。想让动作前真的等人批准,得另设外部派单/执行批准闸(本技能没有内建这一层)。

建独立 tab 并按布局规则拉起 N 个**全新干净**代理(不从旧会话续)。**先在主树建好各槽位 worktree,再按对应 cwd 建面板、起代理**(worktree 只隔离工作文件与索引,不隔 git 元数据,不是安全边界,见"已知限制";项目目录**非 git** 时全部退回主树同 cwd 启动,隔离保证降低,**且无人值守自动退回有人在场模式**——见"无人值守模式"启用闸第 3 条,并在看板记一笔):

```bash
# 0) 非 orch 槽位各一条 worktree(已存在则跳过);分支名带 tag 与槽位名,互不重叠。
#    只按"首次流程块"定下的路径方案执行:非 git 降级(main-only)跳过,N 个 cwd 都用主树
if [ "$PATH_PLAN" = worktree ]; then
  for s in $SLOTS; do
    [ "$s" = orch ] && continue
    git worktree add "$PWD/../wt-$TAG-$s" -b "trio-$TAG-$s"
  done
fi

# 1) 新 tab:orch 槽位留主树;根面板 P0 从返回 JSON 解析
P0=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "${EXPECT_OF[orch]}" --label <任务名> --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["root_pane"]["pane_id"])')
# 2) 裂 pane 循环(P0 = orch 位):非 orch 成员按声明序 M1…M(N−1);
#    M1:P0 --direction right → P1;Mk(k≥2):P(k−1) --direction down → Pk。
#    N=3 时与现行布局逐 pane 一致;N>3 时右列继续向下竖排延伸。
#    每裂一格即从返回 JSON 解析 pane_id,不要手抄;并落盘 $TD/panes.txt(`<slot> <pane_id>` 每行一条)。
#    槽位 → pane 用**关联数组** PANE_OF 记(不再按槽位名拼变量名):起代理与 rename 循环都从这里取,
#    且逐槽位断言非空——写进 PANE_OF_ORCH、读成 PANE_ORCH 这类名字错位会让 pane 取空,必须当场停。
declare -A PANE_OF
PANE_OF[orch]=$P0
: > "$TD/panes.txt"; echo "orch $P0" >> "$TD/panes.txt"
prev=$P0; first=1
for s in $SLOTS; do
  [ "$s" = orch ] && continue
  if [ "$first" = 1 ]; then dir=right; first=0; else dir=down; fi
  p=$(herdr pane split "$prev" --direction "$dir" --cwd "$(expect_of "$s")" --no-focus \
      | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
  [ -n "$p" ] || { echo "槽位 $s 的 pane_id 解析为空:停下人工确认"; exit 1; }
  PANE_OF[$s]=$p
  echo "$s $p" >> "$TD/panes.txt"
  prev=$p
done

# 3) 起代理循环:按槽位查"后端档案",参数取自"模式选择"的数组;写死 bypass/auto 属流程违规。
#    分流按该槽位的 **cli**(不是槽位名):herdr agent start 需要 --kind(kimi/codex/claude 等);
#    档案声明"无钩子"的成员(genox)不进注册表,改由 pane 直起(`pane run genox chat`,见"后端档案"genox 条),
#    不伪造 kind 注册、也不因它没进 agent list 就判"缺成员"。
for s in $SLOTS; do
  pane=${PANE_OF[$s]}
  [ -n "$pane" ] || { echo "槽位 $s 没有 pane(裂 pane 循环未赋值):停下人工确认"; exit 1; }
  case "${CLI_OF[$s]}" in
    genox)  herdr pane run "$pane" "genox chat" ;;                    # 无钩子:唯一可用档=有人在场;参数见"后端档案"
    kimi)   herdr agent start "$s-$TAG" --kind kimi   --pane "$pane" -- "${KIMI_ARGS[@]}" ;;
    codex)  herdr agent start "$s-$TAG" --kind codex  --pane "$pane" -- "${CODEX_ARGS[@]}" ;;
    claude) herdr agent start "$s-$TAG" --kind claude --pane "$pane" -- "${CLAUDE_ARGS[@]}" ;;
    *)      echo "槽位 $s 的 CLI ${CLI_OF[$s]} 未在本循环登记参数组:按'后端档案'实测补一条 case(不套用上面的值)后再拉起"; exit 1 ;;
  esac
done
# 默认 roster 展开后等价于:
#   herdr agent start "orch-$TAG" --kind kimi  --pane "${PANE_OF[orch]}" --
#   herdr agent start "rev-$TAG"  --kind codex --pane "${PANE_OF[rev]}"  -- -m gpt-6.1-sol -c model_reasoning_effort=high
#   herdr agent start "exec-$TAG" --kind claude --pane "${PANE_OF[exec]}" -- "${CLAUDE_ARGS[@]}"
# 数组元素逐项成为独立 argv,含空格路径也不拆、空数组不产生参数;不要退回字符串拼接(见"模式选择"的实测)
# 模型 slug 与推理档必须分开传:slug 不带档位,档位用独立配置键;`--settings` 用技能目录变量拼路径、不写死用户名

# 4) N 个面板全部重命名(必做,漏了代理名与显示标签不一致);pane label 与 agent 名是两个对象
for s in $SLOTS; do
  pane=${PANE_OF[$s]}
  [ -n "$pane" ] || { echo "槽位 $s 没有 pane(裂 pane 循环未赋值):停下人工确认"; exit 1; }
  herdr pane rename "$pane" "$s-$TAG"
done

# 5) bootstrap:任务目录/manifest/锁已在"检测"一步办妥 → 建看板骨架(仅缺时;重入不覆盖)→
#    取本次拉起批次号 bootstrap ID → 等 ACK 与预期全文全等(cmp)——这才是"拉起完成"的判据
TD="/tmp/$TAG"                             # 与"首次流程块"同一个任务目录;此处重设,本块可单独抽取自测
mkdir -p "$TD"                             # 幂等
SKILL_ABS=<发送方已知的技能绝对路径>        # 如 /home/<user>/.kimi-code/skills/trio-orchestrate;包内一律写绝对路径,不假设对端会展开 ${KIMI_SKILL_DIR}
# ══ bootstrap ACK 闭环块(起):整段抽出去、设好 TD/TAG/SKILL_ABS/SLOTS 即可单独跑(对端 herdr 可用桩替)══
# 秒+随机尾只是降低撞车概率,不承诺并发唯一;并发排他靠 launch.lock——一个 TAG 同时只许一个拉起流程
BOOT_ID=$(date +%s)-$RANDOM                # 本次拉起批次号:旧 ACK、失败残留靠它区分
[ -f "$TD/taskboard.md" ] || cat > "$TD/taskboard.md" <<'EOF'
# taskboard

## 待办(带验收线+冲突面)
## 在途(认领人+时刻)
## 待审
## 已验收(附证据路径)
EOF
# 行数随 N 变:ACK 预期 = bootstrap-id 行 + tag 行 + N 行 <slot>-<tag>(槽位固定序),末尾保留一个换行
NAMES=$(for s in $SLOTS; do echo "$s-$TAG"; done)      # N 个全名,槽位固定序
rm -f "$TD/bootstrap-ack.md"               # 清掉旧 ACK:只认本次写出的回执
{
  echo "bootstrap-id: $BOOT_ID"
  echo "技能文件:$SKILL_ABS/SKILL.md(绝对路径;先激活技能并读这个文件,不要依赖占位符自动展开)"
  echo "tag: $TAG"
  echo "看板:$TD/taskboard.md(骨架已建好)"
  echo "成员全名:$(echo $NAMES | tr ' ' '/')(编排 + 审阅 + 执行等;限 [机核] 的条目见看板规则)"
  echo "指令:读技能与看板后,把 ACK 写到 $TD/bootstrap-ack.md。ACK 必须与下面 $((N + 2)) 行**逐字节一致**(照抄、一行一项、不加别的字符、行尾不留空格、末尾保留一个换行):"
  echo "bootstrap-id: $BOOT_ID"
  echo "tag: $TAG"
  for n in $NAMES; do echo "$n"; done
  echo "不要再执行拉起流程(不要建新 tab、不要起新代理)。"
} > "$TD/bootstrap.md"
# 预期 ACK 全文(与上面给对方的 $((N + 2)) 行逐字节一致),验收用全等比较:
# 存在性 grep 挡不住"同一字段两个矛盾值 / 重复行 / 多余行"(2026-10-09 裁决),必须整文比
{
  echo "bootstrap-id: $BOOT_ID"
  echo "tag: $TAG"
  for n in $NAMES; do echo "$n"; done
} > "$TD/ack-expected"
herdr agent prompt "orch-$TAG" "读 $TD/bootstrap.md 并按其指令执行" --wait --timeout 120000
ack="$TD/bootstrap-ack.md"
cmp -s "$TD/ack-expected" "$ack" || { echo "ACK 与预期全文不一致(矛盾/重复/多余行或缺行):拉起未完成"; exit 1; }
rmdir "$TD/launch.lock"                    # ACK 通过 = 拉起完成,释放排他锁;中途失败停下的,人工处理完再删锁重入
# ══ bootstrap ACK 闭环块(止)══
```

布局恒为:orch 位在左(主树);非 orch 成员按声明序沿右列自上而下(N=3 即 reviewer 在右上、executor 在右下,与现行布局逐 pane 一致;先 right 后 down,与宽窄无关)。N>5 时右列竖排可读性差,建议拆任务或人工调整面板——**协议身份锚在 manifest 不在几何**,人工调面板不影响协议。

- 模型名以各 CLI 内 `/model` 列表为准;**模型 slug 与推理档必须分开传**(如 codex `-m <slug> -c model_reasoning_effort=high`)。把 "GPT-6.1-Sol high" 整体当模型名会被 API 以 400 拒掉(2026-10-09 实测踩中:`-m` 当时不报错,首次发请求才炸);slug 以本机 `/model` 列表为准,该模型的默认推理档可能不是 high,必须显式设置;`-m` 被拒就起默认模型后在面板里 `/model` 手动选模型+推理档,并在看板记一笔——人工补救可以,但不算"一键/无人值守"的正常路径;
- **状态语义全文统一**(第 0 步、第 3 步、重启检查点同一套):**可接收新单 = `idle` 或 `done`**(herdr 的 done 同样可输入);`working` 永不作为判据,只是辅助观测;`blocked` 是排查分支。**无钩子成员(如 genox)没有这组状态语义**,改用"pane read 未见活动输出 + 上一单 ACK 已落盘"作收单条件,见"后端档案";
- `agent start` 返回 ready 才算数;起不来先 `herdr pane read` 看卡哪(codex 常卡升级提示,Esc 按掉);**启动失败先分清两种**:参数校验失败、进程根本没创建 → 可复用该 shell pane 修参数重发;进程已在但 blocked/not-ready → 先排障或等它退出,不要盲目重发;
- 裂 pane 成功但 `agent start` 失败(如 invalid_agent_name,此时进程未创建):面板不用重建,修好参数对同一 pane 重发 `agent start` 即可;
- 验收:`herdr pane read` 各槽位面板,审阅槽位状态栏须见 `<模型> <推理档>` 两段,**再发一条无关紧要的测试消息确认不弹 400** 才算真可用(只看模型展示名不算数);**claude 系**执行槽须见 bypass permissions on(仅无人值守参数组;其他 CLI 的无人值守档是各自档案里那组显式参数——"bypass" 只是 claude 的说法,不要拿它去套别的后端)——**这只证明权限模式,不证明 deny 清单已挂载**。**claude 系槽位**的 deny 清单验收分两层:文件存在且 JSON 合法是**语法检查**(`python3 -c "import json;json.load(open('...'))"` 一行验);**效果检查**是编排者 `herdr pane read` 或让该 agent 跑 `/status`(`/permissions`),看 **settings sources 与生效规则列表**,把生效规则条目贴进看板当证据——语法检查 ≠ 效果检查,两回事。**claude 系槽位**验不到有效规则/清单未生效 → **暂停派单**:已在跑的该槽位先停(退出/杀掉)再按有人在场组(显式 `--permission-mode default`)重拉,重拉后重验模式与规则,验过才恢复派单——只改 shell 变量或只看板记一笔都动不了旧进程,不算降级;非 claude 系槽位的模式验收按各自档案的显式参数与实际显示逐条核对(如 codex 的 `-a on-request -c approvals_reviewer=user`),验不出即停;
- bootstrap 的完成判据是 **ACK 与 `/tmp/$TAG/ack-expected` 全文全等(`cmp -s`;内容 = bootstrap ID、tag、N 个成员全名共 N+2 行)**——矛盾字段、重复字段、多余行、缺行、行尾空格全拒,不是某个状态观测,也不是存在性 grep:`agent get` 只作辅助观测——短 bootstrap 在 `--wait` 返回后通常已是 idle/done,等不到 working(旧版曾把 working 当硬条件,2026-10-09 判定为错误);ACK 齐全后再告知用户切到新 tab 把任务下达给 orch 槽位成员(它就是后续编排者;本条拉起会话只负责搭建);
- 例外(**前置:模式选择已过;或按"模式选择"做过记录式授权——用户把授权自动执行的范围写进 `/tmp/$TAG/mode-selection.md`**;`--auto` = Never Ask、在场≠用户批准):若用户就是在某个 kimi --auto 会话里加载本技能、且要它本人当编排者,则不建新 tab——**先用 `herdr agent rename "$HERDR_PANE_ID" "orch-$TAG"` 注册 agent 身份**(pane/tab 的 rename 只是显示标签,与 agent 名是两个不同对象;只改标签不会把本会话注册进角色名),再以当前面板为 P0、按上面的 worktree/cwd 规则裂其余槽位、起其余代理、全部面板照旧重命名,**tab 也 `herdr tab rename` 为任务名**(该 tab 还混有其他无关 pane 时,改为在原名后追加 `<TAG>`)。该例外不豁免模式选择:三闸不过、且没有上面的记录式授权时,**停用例外并停工**(见"模式选择");rename 只注册 agent 身份,不改变本会话的 CLI 权限模式,也不构成用户批准。

## 组队与替补(编队成员不齐时)

- **orch 槽位不可替**:技能就跑在 kimi 里,但二进制检测仍要走(当前会话能跑 ≠ PATH 里有可新起实例的 kimi);缺的是审阅/执行等其它能力才走本节;
- **可用性两级检测,缺一不可**:① `command -v <cli>` 二进制在(PATH 里有能新起实例的 CLI;当前会话里能跑不算);② `herdr integration status` 显示 `current`(状态钩子已装)。缺②可 `herdr integration install <agent>` 补装;补不了则降级为备胎——能起跑但 idle/working 侦测不可靠,通信协议第 3 步 `agent get` 的观测失效,只能 `pane read` 人工盯(**无 kind、herdr 不认的后端如 genox 属这一类,见"后端档案"**);
- **候选要记录**:每个候选写明 CLI kind + provider/模型 + 该槽位能力与权限边界;"不同后端"靠这两项判定,不能只看二进制名;候选池按**能力**查(不是按固定角色):

| 能力 | 首选 | 替补顺序(示例,以两级检测实测为准) |
|---|---|---|
| orch | kimi | 无(不可替) |
| rev | codex | copilot → opencode(先补钩子)→ 第二个 kimi |
| exec | claude | codex → copilot → opencode(先补钩子)→ genox(有人在场档) |

- **硬约束:rev 能力成员与 exec 能力成员不得同源**——同源盲点,审了等于没审。同源判定见"槽位与 roster 声明"的成对约束(不同 CLI 且不同模型族);
- **组队用枚举,不用贪心**:对每个待填槽位给出候选池,枚举**全部组合**,取第一个满足 [能力覆盖 + 成对约束 + 两级健康检查 + 非同一 live agent] 的完整赋值。**两级健康检查按后端分流**:档案声明**有**状态钩子的后端核 ① `command -v <cli>` 二进制 + ② `herdr integration status` 见 `current`;档案声明**无**钩子的后端(genox,见"后端档案")不要求 ②,改核"该槽位 pane 在场、pane 内进程存活、落盘 ACK 身份",**不得因缺 integration 就把 genox 槽位判成无解**。保留现行反例教训:仍禁"固定首选 + 贪心顺位下移"——只有 codex 与 kimi 可用时,合法解是 **kimi 审 + codex 干**;若按"审阅固定首选、只把执行顺位下移"的贪心,会误判成无解;
- 候选健康检查与主后端同级(二进制 + integration `current` + 账号/模型可用性);枚举完仍无合法组队,**明确报告无法组队并停,不硬组**;
- **角色名不变,是槽位名不是后端名**:`rev-<tag>`/`exec-<tag>` 是槽位名(协议、看板、通信全部不动),实际后端与 kind 写进看板头;`--kind` 传实际后端(herdr 支持列表以 `herdr agent start --help` 为准);
- 替补 CLI 的免确认参数各异、启动后验收模式:推广为**按后端档案逐 CLI 验收**(见"后端档案":每 CLI 一条,含两级健康检查、参数组、deny 挂载能力、状态钩子有无);
- **deny 清单挂载按 CLI 区分**:只有 claude 系有 `--settings` 挂 deny 清单的等价机制;**非 claude 执行槽位没有等价机制时,该槽位的无人值守能力随之一律停用**——该槽位改用有人在场参数组,或另配进程级沙箱;**不许"只把撤防记一笔"继续按无人值守跑**。

## 第 1 步:立看板

`/tmp/<TAG>/taskboard.md`(TAG 即第 0 步定下的任务 tag),四列:待办(带验收线+冲突面)/ 在途(认领人+时刻)/ 待审 / 已验收(附证据路径)。规则写进看板头:

1. **单写者:taskboard.md 只由编排者(orch 槽位成员)写**;工作代理不直接编辑看板,只把结果写到 `/tmp/<TAG>/` 下自己的结果文件(如 `<task-id>-result.md`),由编排者按指针移列——多方共写同一份 Markdown 会互相覆盖;
2. 认领写名,移列附证据(测试数/哈希/文件路径),口说无凭;
3. **证据绑代码版本**:每张单带 task ID 与 **base commit**;回执必须带 **tested commit** + 产物路径 + 校验值;审阅与合并核对同一 revision——代码再变(工作树脏了、HEAD 移动),旧验收即失效,要重跑重签;
4. exec 能力成员只认领标 **[机核]** 的条目(验收线机器可核:测试数/哈希/闸脚本);它的文字汇报不是证据;
5. rev 能力成员拉"待审"列,裁决落盘 /tmp 并发指针;
6. **提交纪律(全成员统一)**:每线独立索引;`git add` 只按任务路径,不用 `git add -A`;提交前核对 `git diff --cached`,确认只含本任务改动,不做不看 staged 内容的裸提交;线标前缀与实际槽位一致:`[<slot>-<tag>]`(如 `[orch-g07]`、`[rev-g07]`、`[exec-g07]`);改名/移动必须随提交原子完成;
7. 全量套件串行占坑;多线施工用 `git worktree` 双缓冲;
8. 预登记数(先写死预期再跑)优先于事后解释;
9. **复合槽位 `revtest` 的自审无效(看板流转落款)**:revtest 成员自产出的单子,**审阅签字列留空**,由 orch 终审后直接移"已验收",并在该行附标注"**自审无效、orch 终审**"——单写者规则不变,移列权仍在 orch 一人手上。

**看板头的"拉起信息"段(roster 表)**:除上面 9 条规则外,看板头写一张 roster 表,逐槽位记:槽位 / agent 全名 / 后端 CLI / provider+模型 / 推理档 / worktree 路径 / 参数组(无人值守或有人在场)——这是重入、排障与"谁在干什么"的唯一速查表。

## 无人值守模式(全自动协作的预授权)

用户不在场、没人点确认时,修改/删除/git 操作**预授权放行**。先分清三层:**权限模式**(bypass)只省确认,**隔离**(每槽位 worktree)只分离工作文件与索引,两者都**不是安全边界**(见"已知限制");真正的硬边界是进程级沙箱/容器 + 受限凭据 + 远端 branch protection。

**启用闸(条件开,不是默认开)——三条任一不满足,就退回有人在场模式,不许带默认放权开工:**

| # | 启用前必须满足 | 不满足时的行为 |
|---|---|---|
| 1 | push 硬边界:远端 main/master 已设 **branch protection**(或推送只用权限受限凭据) | 每个 exec 能力成员**禁 push**,改动只留本地分支,由编排者人工推送 |
| 2 | 凭据面:运行环境不挂载非必需凭据(或用独立 Unix 用户/容器隔离) | 不启用无人值守 |
| 3 | 目录是 git 仓库、能建 worktree(非 git 目录没有可用的隔离线) | **非 git 目录自动退回有人在场模式**——这是降级路径,不是等价替代 |

三条都是**开工前的检查项**,不是事后说明;没验过就按不满足处理。三条的逐项证据写法与"三闸全过用无人值守参数组 / 任一不过用有人在场参数组"的落地见第 0 步"**模式选择**"——启动参数由那里选定,不在这里默认放权。失败退回的有人在场组**必须按各 CLI 档案显式给出**并验收真实模式:不传 bypass ≠ 人工批准,既有会话改变量也不生效,得先停再重拉。

**deny 清单挂载按 CLI 区分**:只有 claude 系有 `--settings` 挂 deny 清单的等价机制;**非 claude 执行槽位没有等价机制时,该槽位的无人值守能力一律停用**(改用有人在场参数组,或另配进程级沙箱),不许"只把撤防记一笔"继续按无人值守跑。

1. **每个非 orch 槽位一个 git worktree**(启用闸第 3 条满足后才是前置条件,由编排者在第 0 步先建好再启动各代理):各槽位独立分支、独立索引,`commit -a`/`stash`/`reset` 的作用范围限于本槽位工作文件——但见"已知限制"第 3 条,git 元数据不隔;
2. 主树只由 orch 槽位成员合并/落盘;其它槽位不动主树——这是**纪律约定**,机制上拦不住(执行线实测能改主树引用),靠提交前核对与验收兜底;
3. 删改自由限于自己 worktree 和 `/tmp/<TAG>/` 任务目录内;跨出去删改仍需用户在场确认——这是**默认授权范围**;若按"模式选择"第 3 闸由用户明确选定了"同用户跨目录访问",在此改记为"用户已选定的授权范围",不再同时宣称跨目录必确认;
4. push 只准推各槽位分支并随看板报告;推 main/master、force-push 永禁——但 **deny 清单只是文本前缀软防线,可被等价写法绕过**(实测形态见"已知限制"第 1 条);要真永禁,在远端设 branch protection / 用权限受限凭据;
5. deny 清单只留永禁项(sudo/毁盘/force-push/推 main/读机密)——无人值守下长清单只会卡死跑批;真兜底是机核验收+worktree 隔离+提交纪律(三者的边界见"已知限制")。

## 已知限制(承诺降级,老实写清)

1. **push 永禁不是硬拦**:`claude-exec-deny.json` 里的 `Bash(git push ...)` 只是**文本前缀规则**,是软防线,可被等价写法绕过;各绕过形态按方法区分(不笼统说"已实测")——`git push origin HEAD:main`(本地 bare remote 的 main 被实际改写)与 `git push origin --force` 是**真实 Git 执行**,证据 `/tmp/skill-review/evidence.json`;`git push upstream main`、`+refspec` 强推、`git -C <目录> push ...` 包装、无参数 `git push`(看 upstream/refspec 配置)只是**前缀规则文本建模**(命令串与规则清单比对,无真实 Git 执行记录);以上形态在 **Claude 权限引擎内均未实测**。`--force`、`-f`、`--force-with-lease` 是三条独立前缀,列不全。真正永禁只能靠**远端 branch protection / 受限凭据 / 受控推送入口**,按解析后的目标 ref 与 force 语义执行;
2. **读机密只覆盖 cwd 内直接读**:`Read(./**/...)` 规则管不到 `~/.ssh/`、CLI 凭据目录,也管不住**间接读取**:子进程(Python/Node)自己打开文件、没有显式列出文件名的搜索。危险形态用**枚举**方式进 deny(`.env.local`/`.env.production`/`.env.development`/`.env.staging`、`credentials.json`、`*.key`、`id_rsa` 等),模板名(`.env.example`/`.env.sample`/`.env.template`)因此天然不被命中——权限按 deny → ask → allow 评估,**宽 deny 压得住窄放行,别指望白名单豁免**;枚举天然列不全,无人值守环境**不应挂载非必需凭据**;读取边界要靠进程级文件权限/沙箱落实,任何配置都不能声称枚举完了"所有机密";
3. **worktree 不是安全边界**:它只隔离工作文件与索引,**git 元数据(common dir 下的 config/refs)与主树共享**——实测执行线 `git config` 会被主树读到、`git update-ref refs/heads/main` 能改主树 main 引用、也能推本地远端 main。所以"主树只有编排者能动"是**纪律约定,机制上不成立**;要硬边界得用进程级沙箱/容器 + 独立 Unix 用户 + 受限远端凭据,并由用户明确选择授权范围后启用;
4. **替补后端没有等价控制**:claude 缺席时 `--settings` 挂载失效,替补 CLI 没有同等 deny 机制;此时该槽位无人值守能力应停用或改为有人在场(见"组队与替补"末条);
5. **`/tmp` 可能被系统清理**:看板、ACK、裁决都在 `/tmp/<TAG>/`,重启或清理服务会删;关键 manifest、最终裁决、恢复指针要另存项目内持久位置(见"上下文/token 纪律"第 1 条);
6. **心跳 cron 归属会话**:计划任务属于当前会话,新建会话不继承,需重建(见第 3 步末条);
7. **无钩子成员没有状态语义**:herdr 的 agent kind 表与本机 integration 列表都不含 genox(23 kinds / 17 integrations 三重确认,证据 `/tmp/genox/T001-b-evidence/60-herdr-agent.stderr`、`/tmp/genox/T001-b-evidence/63-herdr-integration-install-help.stdout`、`/tmp/genox/T001-b-evidence/62-herdr-integration-status.stdout`),`herdr agent read` 对 genox 面板直接报 `agent_not_found`(证据 `/tmp/genox/T001-b-evidence/87-agent-read.stdout`);只能 `herdr pane read` 读屏幕(实测可行,证据 `/tmp/genox/T001-b-evidence/85-pane-read-visible.stdout`)。所以 genox 槽位的 idle/working 是**屏幕推断**,弱于钩子;协议闭环不依赖它(锚在落盘 ACK),但**调度时效降级**:派单前与收 ACK 前各 `pane read` 看一眼,一切以落盘 ACK 为准。`herdr pane wait-output --match/--regex --timeout` 是现成的阻塞原语(实测 `output_matched`,证据 `/tmp/genox/T001-b-evidence/89-wait-output.stdout`),可用它等"该往下走了"这个时点。

以上第 1–3 条对应的硬边界(远端保护、凭据隔离、worktree 可用)没有验收之前,按"无人值守模式"的**启用闸**处理:退回有人在场模式,不默认放权。

## 后端档案(backend profile)

> 每个后端 CLI 一条档案:**kind 名 / 两级健康检查 / 参数组(无人值守、有人在场) / deny 挂载能力 / 状态钩子有无 / 专属注记**。第 0 步"起代理循环"按槽位的 cli 来这里取参数;第 0 步"模式选择"按这里的两行表逐 CLI 取显式参数。**raw CLI 的 CLI 专属经验(如网络出口)记在这里,第 3 步正文只留通用规则。**

**参数组表(无人值守/有人在场 两行 × 每个档案化 CLI 一列)**:

| CLI | 无人值守参数组(三闸全过) | 有人在场参数组(任一不过) | deny 挂载 | 状态钩子 |
|---|---|---|---|---|
| kimi | `--auto` | 不加 `--auto`(其语义是 Never Ask) | 无 | 有(`integration status` 见 kimi) |
| codex | `--approve-for-me` | **显式** `-a on-request -c approvals_reviewer=user`(不接受继承:省略 flag ≠ 人工批准——本机用户 config 实测 `approval_policy = "never"`;审批者另由 `approvals_reviewer` 决定,**on-request 不等于每条命令都问人**) | 无 | 有 |
| claude | `--dangerously-skip-permissions --settings ${KIMI_SKILL_DIR}/claude-exec-deny.json` | **显式** `--permission-mode default` + `--settings`(deny 只收紧不放松);≥2.1.283 不传 bypass 也可能继承 auto 或用户 `permissions.defaultMode`,**不接受继承**;启动后验收真实模式 | **有**(仅 claude 系) | 有 |
| ↑ 注 | roster 里出现**多个 claude 系槽位**(如 `exec=claude` + `exec2=claude`)**各实例逐条挂载同一份 `--settings` deny 清单**:清单是**按实例挂载**的,不是"只绑某一个槽位"——现行"只绑 `exec` 槽位、不影响用户单开 claude"的结论正是按实例挂载的证明;新增 claude 系槽位时,启动循环里照样带 `--settings`,一个都不落 | | | |
| ↑ 注(codex) | **网络出口(作者环境限定,不是通用事实)**:作者环境 codex 走 VPN、kimi/deepseek 不走——只有 codex 掉线时先查 VPN;各人按自己的出口情况类比排查(第 3 步的重递判据用这条) | | | |
| copilot / opencode / 其它替补 | 以该 CLI `--help` 实测为准(**逐 CLI 验收**,不套用上面的值) | 同左,显式人工档 | 无 | 补 `herdr integration install <agent>` 后有 |
| **genox** | **无**(见下) | 交互式 `genox chat`(有人在场档为其唯一可用档) | **无** | **无**(herdr 无 genox kind/integration) |

**两级健康检查(所有后端统一)**:① `command -v <cli>` 二进制在;② `herdr integration status` 显示 `current`。**无 kind 的后端(genox)②必然缺失**,按备胎处理:`agent get` 观测失效,只能 `pane read`。

### genox 条目

**[真实执行] 事实(证据均在 `/tmp/genox/T001-b-evidence/`,Genox 3.0.1)**:

- **交互命令与 flag**:`genox chat`,flag 仅 `--base-dir/-d`、`--verbose/-v`、`--help`;**chat 无免确认参数**(证据 `10-help-chat.stdout`);
- **免确认 flag 在 `genox run`**:`--approve-generated-plan`(生成计划并直接批准,不问人)与 `--plan FILE --approve-plan`(批准给定计划文件);源码 `run_preflight.py:292-318` 明确 `interactively_reviewed=False` 且无 TTY 且未批准时报 "No TTY is available and the master plan is unconfirmed. Use --plan FILE --approve-plan or --approve-generated-plan."(证据 `72-approval-source-scan.txt`、`73-interactive-detection.txt`)。**端到端无人值守跑通 = [未实测]**:本机无外网,TCP 连 `api.openai.com:443` 立即 `Network is unreachable`(证据 `50-network-probe.txt`),两条路径都死在模型调用(证据 `41-run-approve-generated-plan.stdout`、`53-run-plan-file2.stdout`);
- **无 TTY 行为**:`genox chat` 无 TTY + `</dev/null` → 打 banner 与 `>` 提示符后读到 EOF 即打印 `Goodbye!`,**退出码 0,不挂死**(证据 `30-chat-null-stdin.stdout`/`.exit`);pty 里实测到达交互提示符、`/quit` 干净退出(证据 `32-chat-pty.raw`);退出码约定 **0 正常 / 2 用法错误 / 1 业务拒绝**(证据 `06-genox-badflag.exit`);
- **进程卫生**:`genox run` 加 **`--foreground`** 防协调进程脱离(不加会脱离终端);失败可能留陈旧 `.control.lock` 元数据(实测留下已死 pid 的锁文件,**不卡后续启动**——连续跑 3 次未 unlock 均正常),清道夫是 `genox unlock [slug] [--control]`(证据 `48-workspace-state.txt`、`49-lockpid-check.txt`、`10-help-unlock.stdout`);
- **doctor 姿态**:`permission mode: workspace-write`、`allow_network: True`、`confirm_destructive: True`、bubblewrap 可用(mode auto)、builtin tools 76、mcp disabled;配置走**默认值 + 环境变量**,`~/.genox/` 仅缓存、无 settings/permission 配置文件(证据 `21-genox-doctor.stdout`、`20-genox-version.stdout`、`70-config-surface.txt`)。

**槽位限制(无状态钩子期间的保险条款)**:

1. **不得承担 orch**(单一编排硬约束):orch 要收 ACK、跑三闸、派单,全靠可靠状态语义,genox 不达标;
2. **可承担 rev / rev2 / exec / exec2…exec9 / revtest**,但:
   - 参数组只有**有人在场**档——**`genox run --approve-generated-plan` 记为 headless 批处理路径 [端到端未实测]**,实测出可用路径后再回填本档案;
   - **派单方式**:`herdr agent prompt --wait` 不可用(未注册 agent),改 pane 通道发**短指针**(现行"窗格只发短指针"本来就成立),长文照旧落盘;
   - **状态侦测降级**:编排者 `pane read` 盯屏推断 idle/working,**派单前 + 收 ACK 前各看一眼**;`working` 永不作判据,一切以下盘 ACK 为准;
   - **收单条件**里"idle 或 done"对 genox 槽位的落地 = "**pane read 未见活动输出 + 上一单 ACK 已落盘**",不许含糊;
3. **心跳 cron 对 genox 槽位照常挂**,闭环即删;
4. **进程卫生**:genox 由 pane shell 前台运行,重启序列 = pane 内退出 → **pane read 确认回 shell 提示符** → 重发命令(现行 restart 序列的 `herdr agent list` 核对步骤对 genox 不适用,改为 pane read 确认进程已退)。

**升级路径(备选 b)**:为 herdr 写 genox 集成钩子(需懂 herdr 插件机制并跟进上游版本),进上游后本条目就地升级为"有钩子"档,roster/命名/看板全部不动。**否掉的路径 (c)**:用 `agent start --kind` 映射到最近 kind——kind 决定 herdr 的驱动协议(prompt 注入/状态解析),错 kind = 收发全坏,且属伪注册、污染注册表。

## 第 2 步:分层派活(按判断密度,不按面板)

| 层 | 内容 | 派给 |
|---|---|---|
| A 判断密集(调试/设计/取舍) | 编排者的子代理(Agent 工具,可并行多开) |
| B 规格已死的实现 | 编排者子代理,或 exec 能力成员 + 严规格 |
| C 机械执行(跑套件/重生成/读大文件/对表) | exec 能力成员 |
| 审阅 | rev 能力成员 |

**派活分级的判据(2026-10-09 固化)**:任务书里还含"该怎么设计"的方案决策(选哪种实现、怎么拆、阈值定多少)= **A 层**,留给编排者的子代理——**不许标 [机核] 派给 exec 能力成员**;[机核] 只标做法已写死、验收线机器可核的 C 层活(命令、路径、期望值齐全,不需要执行者做任何判断)。把设计活写成 [机核] 下达,等于让执行者替你做设计决策,回执再"机器可核"也补不回来。

额度纪律:大文件阅读、长任务跑批放 C 层。子代理报 403 先**按类型分**——配额墙 / 鉴权失败 / 网络故障:配额墙先保存进度、退避等待,恢复后再用 `Agent(resume=...)` 续跑,不重开;**不做无限重试**;鉴权/网络按各自原因处理;换了模型/代理/参数,记为新的执行条件,不自动继承旧验收。

## 上下文/token 纪律(上下文是缓存,看板是真相)

自动压缩在接近上限才触发(70% 不动是正常的),且压缩本身也要花额度——**少积累+状态外置+到期重启比压缩更有效**:

1. **状态全外置**:任务书、裁决、批次结果全部落 `/tmp/<TAG>/`;任何 agent 可杀可重启,重启后按**已持久化的检查点**(看板+指针文件+ACK/结果文件)恢复。注意 `/tmp` 可能被系统清理:关键 manifest、最终裁决、恢复指针要另存项目内持久位置;
2. **双向落盘**:不只下发落盘,agent 的**产出也落盘**——汇报只许回"结论+证据路径+关键数字",禁止把大段文件/日志贴回对话(贴一次,后面每轮都带着它烧);
3. **prompt 瘦身**:窗格只发短指针+明确输出格式(结论先行、几行封顶),任务书再长也是一个路径;
4. **C 层短生命周期**:exec 能力成员是**无状态执行器**,一批机核活干完就重启(`/exit` 后 `agent start` 重拉),不留恋上下文;它的记忆在看板,不在会话;
5. **审阅层按需重启**:rev 能力成员的裁决历史必须在 `/tmp/<TAG>/` 的裁决文件里,重启按已持久化检查点恢复;跨任务/积压久了就重启;
6. **阈值巡检 + 重启状态机**:阈值以各 CLI 当前配置为准(不写成所有后端的通用事实)。编排者心跳 `pane read` 看状态栏;重启只在安全检查点做:确认该 agent 处于**可接收状态(idle 或 done)**、产物/未提交状态已落盘(working 中不发 `/compact`、不打断任务)→ 发 `/compact`(claude/codex 均支持),无效才重启。重启序列:`/exit` → `pane read` 确认**回到 shell 提示符**(这只说明屏幕回到 shell,**不等于注册状态已释放**)→ `herdr agent list` 核对旧名已从注册表消失 → 再 `agent start`;进程在但 blocked/not-ready 时先排障或等它退出,不盲目重发。**无钩子成员(genox)的差异**:没有注册表可核对,重启序列的 `herdr agent list` 一步改为 `pane read` 确认进程已退、回到 shell 提示符(见"后端档案"genox 条第 4 款)。

## 额度耗尽预案(quota 是可调度资源,不是事故)

额度(5h 窗/日配额)耗尽与网络断连一样是**常态外部事件**,处理原则是吸收不是硬扛:

1. **检测进心跳**:编排者 `pane read` 抓额度信号(codex 状态栏 `5h limit: N% left · resets at HH:MM`——**这是作者环境的示例,不是检测契约**:状态栏项可配置、版本/认证方式各异,以 `/status` 实测为准;未显示或取不到就标**未知**,不当成充足);kimi 403 只是候选信号,按"额度纪律"条先分"配额/鉴权/网络"再进处理链。同一账户可能同时有多个限制(如 5h 窗+周限),分别观察。**<30% 记看板并收紧派单**(只派关键单;关键验收暂停——这是本方调度政策,不是厂商保证的安全阈值);
2. **额度池独立性要核实,不能从 CLI 名推**:不同 CLI 可能挂同一计费账户(Claude 用 API key、Copilot BYOK 时,两个 CLI 烧的是同一池)。替补前按 **provider + 认证方式 + 计费账户/组织 + 额度桶与窗口** 逐项核对,确认候选不受原耗尽桶约束才算额度容灾;换客户端本身不算。核实记录不含密钥;
3. **换脑恢复的前提是状态外置**:看板+manifest+task ID 在盘上,替补按**最后已确认检查点**恢复——注意中断现场可能是"执行完但未回执",接手前先核对产物/进程/提交并确认旧执行者已停写;必要重跑登记进看板。恢复承诺限于"不重复已核实完成的工作",不承诺任意中断都无损;
4. **预案链按能力**:rev 能力断 → ①等重置窗口 ②备援审阅(向**另一 rev 能力成员**借调,可跨任务/跨 tab,先知会其编排者,召回即还)③替补审阅(降级标注,**不作最终放行签**);exec 能力断 → 按"组队与替补"换后端(rev/exec 不得同源的约束仍适用);
5. **质量档不为省额度而降**:终审/关键闸保持既定推理档;额度末班车上不跑关键验收,宁可等重置。(作者 2026-10-09 单次观察:medium 档低余量下仍抓出确定性 bug——bug 本身证据 /tmp/skill-review/verdict-r5.md;档位与余量数字是编排者当时 `pane read` 状态栏的观测,**未留档、未独立核实,照实标注**;这是单次观察,不是 medium 普遍充分的证明;low 未验过,终审不用它);
6. **额度入看板**:大单派出前记录当前余量、额度桶/窗口与重置时刻(这是预登记);事后的消耗趋势是复盘数据,两者分开记。

## 第 3 步:通信协议(送达与重递判据)

- **每张单带 task ID**(T001/T002…,与看板条目一致);发送前确认**目标槽位**的 agent 处于**可接收状态:`idle` 或 `done`**(done 同样可输入);`working` 时不发(那是上一单在跑),`blocked` 先排查。**无钩子成员没有这组状态语义**,其"可接收"按"pane read 未见活动输出 + 上一单 ACK 已落盘"判定(见"后端档案"genox 条);
- 发送优先 `herdr agent prompt <agent 全名> "..." --wait --timeout <ms>`(带有限超时);检查返回里的 blocked/stalled/timeout。**无钩子成员(genox)不走这条**,改 pane 通道发短指针;
- **受理/完成的判据是落盘的 ACK/结果文件,且必须绑 task ID + 当次批次**(批次号/bootstrap ID);不看单次状态观测:`working` 只是**辅助观测**——短任务可能在 `agent get` 之前就跑完(永远看不到 working);对端原本就在 working 时,看到 working 也不能证明新单已开始;
- 重递前先核对 ACK 与产物(结果文件、进程、提交),**任务必须幂等**(可安全重跑),避免超时重递变成重复执行;
- 长正文/任务书**落盘 /tmp/<TAG>/ 发路径**,窗格只发短指针(窗格通道实测会丢长文);
- 网络故障按各 CLI 档案的专属注记排查(如 codex 走 VPN 的注记见"后端档案");确认是网络问题、且上一单确未被执行,再按上面的重递流程重递;仍不可用 → **备援审阅**(向另一 rev 能力成员借调,先知会其编排者,召回即还);
- 长任务挂心跳 cron(避开 :00/:30),闭环即删;**cron 归属当前会话,新建会话需重建**。

## 第 4 步:验收文化

- 修复类改动必须"改前红、改后绿"双证据;
- 编排者亲自复核关键数字(复跑/复算),不转述;
- "应该相同"≠"实测相同";静态声明与实测分开写;
- **验收闸必须含行为探针**:校验谓词、ACK 比较、启动参数组这类可执行物,闸里要**构造对抗样本真跑**——矛盾字段、空值、重复行、错路径、抢锁、残留目录,每类都得有能判红/判绿的样本;只 grep 字符串不算验收(字符串在不在,与谓词判得对不对是两件事——2026-10-09 前四轮闸全绿、行为样本一跑就红)。对抗样本集在 `/tmp/skill-review/` 与 `/tmp/skillfix/r*-evidence/`,可抄;
- **闸脚本本身也要被验**:写完闸先自证它能红能绿;模式串以 `-` 开头必须用 `grep -qF -- "$3"` 这样加 `--` 分隔,参数错位会让闸静默错检(2026-10-09 编排者自犯两次);改闸后至少回放一个"旧实现必红"的样本;
- 被推翻的自己说过的话照实留更正记录。

## 第 5 步:冻结与并行

- 大计算批次(每折新进程读活树)期间**冻结仓库**;冻结/解冻由编排者宣布;
- 真实数据规模会暴露 demo 看不见的缺陷(软作用域、外连接、类型分块)——验收证据要有真数据尺度的至少一份。

## 启动话术模板

> 任务:<一句话>。我(orch 槽位成员)编排+验收;本次编队(roster):<逐槽位列出 全名 / 后端 / 模型+推理档 / 参数组>。
> 看板:/tmp/<TAG>/taskboard.md。纪律:线标提交、路径 add、串行全量、落盘发指针。
> 派单带 task ID 与 base commit;回执以落盘 ACK/结果文件为准,`working` 只当辅助观测;无钩子成员按"pane read + 落盘 ACK"判定。

## 发布前 smoke checklist

分享/发布前逐项过。证据分四档,弃用笼统的 [已实测]:**[真实执行]** = 实际运行过命令/脚本,留有输入与结果路径;**[前缀建模]** = 只把命令串与规则清单/谓词做文本比对,没有真实运行;**[静态推演]** = 手工推理或一次性枚举,未固化成可重跑的记录;**[未实测]** = 尚未运行,只是规则要求。协议版本:**2026-10-10-r6**。

> 行末带 **T001-d** 的,表示该行的探针由 T001-d 真跑;**本表只写要求,落证前不宣称已执行**——见到 T001-d 而 T001-d 尚未回执时,按 [未实测] 读。

| # | 检查项 | 状态与证据 |
|---|---|---|
| 1 | 参数校验:**tag 照旧 8 例**;**roster 追加至少 6 例**:空串、含空格、含大写、含换行、非法槽位名、槽位重复、N=1、越界段数 —— 全拒;且**模型闸先于任何含 $tag/$roster 的命令** | **[真实执行] T001-d**(tag 的 8 例沿用 2026-10-09 证据 `/tmp/skillfix/r3-evidence/tag-validation.txt`,1 放行 7 拒绝;roster 的 ≥6 例与"模型闸先行"由 T001-d 真跑)。"展开值先经模型阅读判断"这条本身[未实测](需真调 Kimi 模板引擎喂非法 `$tag`/`$roster`) |
| 2 | 同 tag 重入:**N ∈ {2,3,4} 各跑一组**复用检查;r5 的 6 样本口径(realpath 判等、缺拓扑 fail-closed、跨 tab、缺角色)逐样本重放到 N=2 与 N=4 | **[真实执行] T001-d**(r5 原证据 `/tmp/skillfix/r4-evidence/reuse-realpath.txt`;退出码口径不变:0 可复用 / 2 缺成员 / 3 fail-closed。N 泛化后的重放由 T001-d 真跑);**补建与失败回滚**[未实测] |
| 3 | 部分建队失败:进程未创建可同 pane 重发;进程在但 blocked/not-ready 先排障 | [未实测] |
| 4 | 替补无解:枚举所有候选组合后才报无解 | **[真实执行]** 2026-10-09 脚本:审阅候选=[codex,kimi]、执行候选=[codex] 时,贪心误报无解、枚举得 kimi 审+codex 干(脚本与输出:`/tmp/skillfix/r3-evidence/role-pair-enum.txt`);**按槽位候选池 × 全组合**的泛化枚举实机演练[未实测] |
| 5 | deny 负例(B1):`git push origin HEAD:main`、`git push origin --force` | **[真实执行]** 临时 Git 命令真跑:本地 bare remote 的 main 被实际改写(证据 `/tmp/skill-review/evidence.json`) |
| 5b | deny 负例(B1)其余形态:`+refspec`、`git -C` 包装、`upstream main`、无参数 `push` | **[前缀建模]** 命令串与规则清单文本比对,无真实 Git 执行记录;权限引擎内端到端负例[未实测] |
| 6 | deny 负例(N1):模板名 `.env.example` 不被任何 deny 规则命中(枚举危险形态,不靠白名单) | **[前缀建模]** 规则交集比对(fnmatch,2026-10-09);引擎内最终判定[未实测] |
| 7 | 重启恢复:`/exit` → 提示符 → `herdr agent list` 核对旧名已释放 → 重拉(**无钩子成员改为 pane read 确认进程已退**) | [未实测] |
| 8 | 模型参数:`-m <slug> -c model_reasoning_effort=high` 生效(状态栏两段 + 测试消息不弹 400) | 修复后生效待验[未实测];踩坑(整串当模型名 → 400)**[真实执行]** 2026-10-09 |
| 9 | bootstrap:ACK 与 `/tmp/<TAG>/ack-expected` **全文全等(`cmp -s`)**才算"拉起完成"——矛盾字段、重复字段、多余行、缺行、行尾空格全拒;**N ∈ {2,3,4} 的判绿样本 + 每 N 的判红样本**;任务身份看 manifest,重入**唯一解析**:必需字段各恰好一行、值非空、同名字段两次即拒;manifest 唯一解析**在 roster 行上的对抗样本**:roster 缺行 / 重复行 / 与 cwd 槽位集合不符 / cwd 行数 ≠ N;**回放"旧 r5 解析器遇 roster 行必红"样本 ≥1** | **[真实执行] T001-d**(r5 的 8 样本口径沿用并扩 N:ACK 7 样本 `/tmp/skillfix/r4-evidence/cmp-ack.txt`、manifest 唯一解析 8 样本与目录安全 5 样本 `/tmp/skillfix/r5-evidence/manifest-unique-parse.txt`、`/tmp/skillfix/r5-evidence/dir-guard.txt`;N 泛化与 roster 行对抗样本由 T001-d 真跑);实机拉起 ACK[未实测] |
| 10 | 派单状态语义:`done` 可收单;ACK 绑 task ID + 当次批次;`working` 不作判据;**无钩子成员的收单条件 = pane read 未见活动输出 + 上一单 ACK 已落盘** | [未实测] |
| 11 | 模式选择:三闸证据落 `/tmp/<TAG>/mode-selection.md` → 全过用无人值守参数组,任一不过用有人在场参数组(不对执行槽发 bypass/auto;**有人在场按各 CLI 档案显式给出**——claude 显式 `--permission-mode default`、codex 显式 `-a on-request -c approvals_reviewer=user`,都不接受继承本机未审默认) | **[静态推演]** 两参数组按闸结果二选一为人工推演;参数组表按 CLI 展开为人工推演;实机按两模式各拉起一次[未实测] |
| 12 | claude 有效规则**效果检查**:`/status`(`/permissions`)看 settings sources 与生效规则,条目贴看板;验不到 → 退回有人在场并**先停再按显式人工模式重拉重验**;启动后验收真实模式(`pane read`/`/status`),验不出人工模式 = 闸失败停止派单 | [未实测](第 6 行的规则交集比对是[前缀建模],不等于引擎内生效) |
| 13 | 参数组为 bash 数组、`"${ARR[@]}"` 逐项成 argv:含空格路径不被拆、Zsh 标量不分词问题不复现;空参数组 `=()` 展开为空 | **[真实执行]** 2026-10-09:bash 5.2.21 与 zsh 5.9 × 两模式 × 两路径(含 `/tmp/skill space`)共 8 样本——数组形式两边都逐项正确;旧标量串在 bash 空格路径被拆成两项、在 zsh 整组变成一个参数(证据 `/tmp/skillfix/r4-evidence/argv-arrays.txt`) |
| 14 | 有人在场落地:按各 CLI 档案显式给出人工档 **且**启动后验到真实人工模式才恢复派单;codex 显式审批策略/审批者并 `pane read` 核对显示(`on-request` 只保证"请求时才问",不等于每条命令都问人);既有会话不适用启动参数、先停再拉;orch rename 例外收窄为"模式验收已过 或 用户把授权范围落盘 `mode-selection.md`"(`--auto`=Never Ask,在场≠用户批准) | [未实测](文档依据:claude `v2.1.283` 起不传 bypass 也可能继承 auto 或用户 `defaultMode`;本机 codex 用户 config `approval_policy = "never"` 为只读窄读) |
| 15 | **先分类后取锁**(N3):空目录 → 分类"新建" → 取锁 → 建 manifest → ACK 全等 → 释放的完整链;**用 N=4 roster 整段抽跑**;残留目录(非空无 manifest)在取锁前被拒;同 TAG 并发抢锁被拒;全程不许删锁换绿 | **[真实执行] T001-d**(r5 原证据:`/tmp/skillfix/r5-evidence/lock-flow.txt`,4 分支 + 闸的顺序探针;N=4 roster 的整段抽跑由 T001-d 真跑);真代理拉起[未实测] |
| 16 | **成对约束校验器**:判绿 = 现行默认 trio;判红 = 同 CLI 对、跨 CLI 同 family 对;**报错措辞与 SKILL 写死文本一致**;闸脚本自证能红能绿 | **[真实执行] T001-d**(每类 ≥1 样本) |
| 17 | **布局与命名**:N=2/3/4 的 pane 裂序(right→down 循环)与 rename 循环静态推演 + **N=3 与 r5 布局逐 pane 一致性核对** | **[静态推演]** + 真机拉起[未实测] |
| 18 | **genox 槽位拉起**:启动命令真跑;免确认路径按 `/tmp/genox/T001-b-result.md` 回填;验证"有人在场 + pane 发指针 + 落盘 ACK"闭环可用桩对端跑通 | **[真实执行] T001-d**(依赖 T001-b);T001-b 已落证部分见"后端档案"genox 条;**端到端无人值守跑通仍 [未实测]**(本机无外网) |

## 附录:原作者经验(不是通用事实,按自己的环境验证)

> **后端专属经验(如网络出口)已移入"后端档案"节**,按 CLI 各记一条。

- 额度:kimi/codex 额度较紧,大文件阅读、长任务跑批尽量放 C 层;
- 后端选择:exec 槽位挂什么后端各人自定,输出不稳的后端只配机核活;
- 2026-10-08 共享索引险情:`git add -A` 卷走了别线 staged 的改动;
- 2026-10-09 实测踩中:tag 超长报 `invalid_agent_name`;早期的静默截断让两个 tag 撞名(已删);`-m "GPT-6.1-Sol high"` 整体当模型名,`-m` 不报错、首次发请求才炸 400;
- 窗格通道丢长文:作者环境一天丢过四次同一条,所以一律落盘发指针;
- r5 任务按 r5 协议收尾,不迁移到 r6 命名(旧 manifest 里是 `kimi-orch-*` 一类固定名,r6 起为 `<slot>-<tag>`,重入会被 fail-closed 判"非本任务"——这是现行语义,不特例)。
