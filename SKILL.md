---
name: trio-orchestrate
description: "一键拉起并编排 kimi(编排+验收)/codex(审阅)/claude(执行)三智能体协作。当用户要开始一项新的多智能体任务,或说'安排三方协作/开工/拉起评审执行/立看板'时使用。可带参数:/skill:trio-orchestrate <任务tag>,参数直接用作任务 tag。依赖 herdr(终端多路复用)建面板、起代理、传话、盯状态。用户只需启动 kimi 并加载本技能,其余面板自动建好。"
arguments:
  - tag
---

# 三智能体协作:一键拉起 + 编排(kimi 编排 · codex-review 审 · claude-exec 干)

> 作者:sheng luan
> 依赖:kimi CLI + herdr;codex/claude 缺了走"替补组队"。分享方式:整个 `trio-orchestrate/` 目录(含 `claude-exec-deny.json`)拷到对方 `~/.kimi-code/skills/` 即可。
> 本技能同时是**拉起协议**与**运行协议**。分享出去后若担心普通对话误触发建 tab/切模式,可按需在 frontmatter 加 `disableModelInvocation: true`,改为只由显式命令调用。

## 角色与固定名字(不再随任务变)

| 角色 | 固定名(代理名+面板标签) | 后端 | 模式 |
|---|---|---|---|
| 编排+验收 | `kimi-orch` | kimi | `--auto` |
| 审阅 | `codex-review` | codex,**模型与推理档分开设**(如模型 slug `gpt-6.1-sol` + 推理档 `high`;具体型号以各人 codex `/model` 列表为准) | `--approve-for-me` |
| 执行(C 层) | `claude-exec` | claude(挂什么后端各人自定,把 CLI kind 与 provider/模型记进看板) | `--dangerously-skip-permissions` + 启动时 `--settings` 挂技能目录的 `claude-exec-deny.json` 永禁清单(sudo/毁盘/force-push/推 main/读机密;只绑 claude-exec,不影响用户单开 claude;其余全放行)。**该清单只是文本前缀软防线,不是安全边界——能绕过哪些形态见"已知限制"** |

> 上表"模式"列是**无人值守参数组**的取值;用不用它们由第 0 步"**模式选择**"的三闸结果决定——任一闸不过就改用**有人在场参数组**,不是默认值。

**命名规则(多 tab 并存关键)**:实际 agent 名 = `<角色>-<任务tag>`,如 `codex-review-g07tune`。原因:herdr 的 **agent 名是全局注册表,跨 tab/workspace 唯一**(实测撞名报 `agent_name_taken`;pane label 只是显示标签,不作身份);合法字符仅 `[a-z0-9_-]`、≤32。同一 tab 的三件套共享同一 tag;pane label 与 agent 名保持一致。**复用检查必须用 JSON 精确匹配三个完整 agent 名**,不能用 `grep -- "-<tag>"` 这类子串匹配:`herdr agent list` 返回紧凑单行 JSON,grep 会把 `g070` 当成 `g07`、把别的任务整行算进来(2026-10-09 实测)。示例见"第 0 步"。别的 tab 的三件套不算数,绝不跨任务复用。

## 第 0 步:拉起面板(全新三面板,独立 tab)

> 以下代码块是**概念流程**,不是可整段粘贴的脚本:逐步执行、**每步检查返回码**,任一前置失败立即停止(fail-fast)。P0/P1/P2 一律从命令返回的 JSON 里解析,不要手抄;示例见下。首次使用前先人工跑通一遍:各 CLI 登录、仓库 trust 确认、bypass 模式的首次确认,都不能放到无人值守时段做。**代码块语法一律按 Bash 写**(参数组用数组,`"${ARR[@]}"` 逐项成 argv;Zsh 对未引用标量不分词,行为不同,不要直接照搬到 Zsh)。

```bash
command -v herdr >/dev/null || { echo "未安装 herdr,停下告知用户"; exit 1; }
[ "${HERDR_ENV:-}" = 1 ] || { echo "不在 herdr 内运行,停下告知用户"; exit 1; }
herdr integration status           # 每个 agent 的状态钩子装没装:binary 在但钩子缺 → 状态侦测不可靠
for a in kimi codex claude; do command -v $a >/dev/null && echo "$a ✓" || echo "缺 $a → 走替补组队"; done

# ── 模型闸(第一道,先于任何含 TAG 的命令,由编排者=模型执行)──────────────
# $tag 在技能加载时已被文本展开进本文——下面那行赋值本身就是 shell 源码:
# 校验写在赋值之后,挡不住"赋值之前"就随 $tag 进入源码的文本。真正第一道是
# 阅读判断:读到 $tag 的展开值后,先核对整串符合 ^[a-z0-9_-]{1,19}$(空/空格/
# 大写/超长/换行一律非法);非法就停下找用户重取——不敲下面任何一行,也不把
# 非法值写进任何命令。这是模型闸,不是 shell 闸;不许"先赋值、再补校验"。
TAG=$tag                           # 调用参数:/skill:trio-orchestrate <tag>(必须带 skill: 前缀);为空则等用户下达任务时点名。定后整局不变,同时用作 tab 标签与看板目录 /tmp/<TAG>/
# 二次防线(shell 层,挡的是赋值之后的继续使用;不能替代上面的模型闸):全字符串校验
# (2026-10-09 实测:$'g07\n../outside' 能穿过旧 grep -qE 的行锚定;bash [[ =~ ]] 全字符串则拒绝)
[[ "$TAG" =~ ^[a-z0-9_-]{1,19}$ ]] || { echo "TAG 不合法(空/含 [a-z0-9_-] 以外字符/超 19 字符/含换行),拒绝并要求用户重取"; exit 1; }
# 19 = 32(agent 名上限) − 13(codex-review- 前缀长);超长一律拒绝,不做静默截断(截断会让两个 tag 撞名,2026-10-09 实测)

# ── 任务目录:先分类后取锁、先定路径方案;不无条件覆盖 ──────────────────────
# 次序(2026-10-09 裁决 N3):建目录 → 目录/文件安全核验 → 定路径方案 → 分类(新建/重入/恢复)
#   → 分类通过才取排他锁 → 建/核对 manifest。
# 不能先取锁再分类:新取的锁自己让目录非空,合法新建会被"非空且无 manifest"判死(自阻塞)。
# 排他:一个 TAG 同时只许一个拉起流程(并发共用 ACK 文件本身不允许);锁在拉起完成(ACK 通过)后释放,
# 中途失败停下的先别删锁——人工处理完再删锁重入,防误重入
# ══ 首次流程块(起):把本块整段抽出去、设好 TAG 与 PWD(项目根 cwd)即可单独跑 ══
TD="/tmp/$TAG"                                          # 任务目录(状态目录);本块一律用 $TD
mkdir -p "$TD"                                          # 幂等

# 目录/文件安全:读写的每个路径都查——不是符号链接(`[ -L ]`)、属主是自己;不存在的视为可创建
guard() {
  [ -e "$1" ] || [ -L "$1" ] || return 0
  [ -L "$1" ] && { echo "拒绝:$1 是符号链接(可能指向任务目录外部)"; return 1; }
  [ "$(stat -c %u -- "$1" 2>/dev/null)" = "$(id -u)" ] || { echo "拒绝:$1 属主不是当前用户"; return 1; }
  return 0
}
for p in "$TD" "$TD/manifest.txt" "$TD/taskboard.md" "$TD/bootstrap.md" "$TD/bootstrap-ack.md" "$TD/ack-expected" "$TD/mode-selection.md"; do
  guard "$p" || { echo "任务目录/文件安全核验不过:停下人工确认"; exit 1; }
done

ROOT=$(realpath -m "$PWD")                              # 项目根规范绝对路径:任务身份的锚
# 路径方案在生成 manifest 之前定死(不是先算兄弟 worktree、再拒正文允许的非 git 降级):
#   git 仓库 → 三角色 cwd = 主树 + 两条兄弟 worktree;非 git → 三个 cwd 都记主树(只允许在有人在场模式下)
if git -C "$PWD" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  PATH_PLAN=worktree
  EXPECT_ORCH=$(realpath -m "$PWD")
  EXPECT_REVIEW=$(realpath -m "$PWD/../wt-$TAG-review")
  EXPECT_EXEC=$(realpath -m "$PWD/../wt-$TAG-exec")
else
  PATH_PLAN=main-only
  EXPECT_ORCH=$(realpath -m "$PWD"); EXPECT_REVIEW="$EXPECT_ORCH"; EXPECT_EXEC="$EXPECT_ORCH"
fi

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
  # 重入:manifest **唯一解析**——必需字段(tag / project-root / 三条 cwd / created / bootstrap-id)
  # 每条恰好一行、值非空;同名字段出现两次即拒(矛盾值不算"逐字段一致");cwd 三角色各一条、realpath 比对
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
cwds = seen.get("cwd", [])
if len(cwds) != 3: sys.exit(f"cwd 行应恰好三条(实得 {len(cwds)})")
got = {}
for v in cwds:
    name, sep, p = v.partition(" ")
    if sep != " " or not p.strip(): sys.exit(f"cwd 行缺分隔符或路径为空:{v!r}")
    if name in got: sys.exit(f"cwd 角色 {name} 重复")
    got[name] = p.strip()
if set(got) != set(want_cwd): sys.exit("cwd 角色集合与本次不符")
for name, exp in want_cwd.items():
    if os.path.realpath(got[name]) != os.path.realpath(exp):
        sys.exit(f"{name} cwd={got[name]} ≠ 预期 {exp}(realpath 规范化后仍不同)")
print("manifest 唯一解析通过:字段无重复、值非空,三角色 cwd 与本次一致", file=sys.stderr)
for _n in (f"kimi-orch-{tag}", f"codex-review-{tag}", f"claude-exec-{tag}"):
    print(got[_n])                                  # stdout 只出三条已验路径(顺序固定),供 shell 逐行取回
PY
  ) || { echo "manifest 与本次不符(字段重复/空值/项目根/角色路径/缺路径):不识别为本任务,停下人工确认"; exit 1; }
  # 预期 cwd 由上面同一解析器直接输出(重入不重新拼路径,也不二次解析——awk 二次解析的字段口径曾与 manifest 行格式脱节,三条 EXPECT 被清空,2026-10-09 N4);逐行取回并防空值
  { IFS= read -r EXPECT_ORCH; IFS= read -r EXPECT_REVIEW; IFS= read -r EXPECT_EXEC; } <<< "$_mout"
  [ -n "$EXPECT_ORCH" ] && [ -n "$EXPECT_REVIEW" ] && [ -n "$EXPECT_EXEC" ] || { echo "重入取回的预期 cwd 有空值:停下人工确认"; exit 1; }
  echo "重入:manifest 唯一解析一致,复用已有看板,只补缺、不覆盖"
else
  CREATE_ID=$(date +%s)-$RANDOM                         # 建目录批次号:任务身份的一部分,只在新建时取
  cat > "$TD/manifest.txt" <<EOF
tag: $TAG
project-root: $ROOT
cwd: kimi-orch-$TAG $EXPECT_ORCH
cwd: codex-review-$TAG $EXPECT_REVIEW
cwd: claude-exec-$TAG $EXPECT_EXEC
created: $(date -Is)
bootstrap-id: $CREATE_ID
EOF
  echo "新建:任务目录已初始化(路径方案 $PATH_PLAN;锁已上,ACK 通过后释放)"
fi
# ══ 首次流程块(止)══

# 复用检查:JSON 精确匹配三个完整 agent 名;再逐角色核对其 cwd == 该角色**预期 cwd**(git 方案=该线 worktree;
# 非 git 降级=主树)——预期值一律来自上面 manifest
# (不是要求三者 cwd 相等——三者同 tab 但错跑到别的目录,一样不能复用);预期路径取自上面 manifest,
# 两侧都过 realpath 规范化后比较——带 .. 的拼法与规范 cwd 指向同一目录时必须判等,否则正常重入会被误判;
# tab_id/workspace_id 任一缺失即非零退出(fail-closed),不拿"能打印出结论"当通过
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
    print("缺: "+", ".join(missing)); sys.exit(2)
def bad(msg):
    print(msg); sys.exit(3)
for w,exp in expect.items():
    a=got[w]
    if not a.get("tab_id") or not a.get("workspace_id"):
        bad(w+" 缺 tab_id/workspace_id,fail-closed:不复用")
    cwd=a.get("cwd")
    if norm(cwd)!=norm(exp):
        bad(w+" cwd="+str(cwd)+" ≠ 预期 worktree="+str(exp)+"(realpath 规范化后仍不同):不复用")
tabs={got[w].get("tab_id") for w in expect}
if len(tabs)!=1:
    bad("三件套不在同一 tab: "+", ".join(w+"@"+str(got[w].get("tab_id")) for w in expect))
print("三件套齐全:同 tab,且各自 cwd 与预期 worktree 一致(realpath 规范化后)")' "$TAG" "$EXPECT_ORCH" "$EXPECT_REVIEW" "$EXPECT_EXEC"
# 退出码:0 = 可复用;2 = 缺角色(只补缺的);3 = 拓扑/目录不一致或缺字段(fail-closed,不自动复用,人工确认后重建)
# 预期 cwd 一律取自 manifest:新建时按"首次流程块"定好的路径方案写死(git→主树+两条兄弟 worktree;
# 非 git→三个 cwd 都记主树,只允许在有人在场模式下),重入时经唯一解析取出——这里把三个预期路径传给复用检查
```

### 模式选择(检测之后、拉起之前,必做)

**启动参数是模式选择的结果,不是默认值;没做模式选择就启动 = 流程违规。** 任务目录、manifest(项目根/各角色预期 cwd/创建时间/bootstrap ID)与排他锁已在上面"检测"一步办妥;本步按"无人值守模式"启用闸三条逐项核验,每条都在 `/tmp/$TAG/mode-selection.md` 落证据(写不清证据 = 该闸不过),看板建好后把该文件指针贴进去:

| 闸 | 核验对象(不是只核"有个保护标志") | 证据写什么 |
|---|---|---|
| 1 推送边界 | ① **执行身份本身无 bypass 权限**(远端管理员直推、保护规则的 bypass 名单都要核);② 目标 ref(main/master)被保护规则实际覆盖;③ 其他允许推送的分支也不能 force-push | branch protection 实际设置(如 `gh api repos/<owner>/<repo>/branches/main/protection` 看 `enforce_admins` 与规则),或凭据侧只读证据;核不实 → 不过 |
| 2 凭据面 | 记录**哪个执行实例**(claude-exec-$TAG 这个进程)、**哪些凭据是必需的**(只留各 CLI 自身登录态)、**敏感路径清单**(挂载点、可读目录),以及隔离手段(独立 Unix 用户/容器/不挂载) | 最小可核对的隔离配置 + 挂载清单;容器存在 ≠ 没挂敏感数据,独立用户名 ≠ 文件不可读,逐项写 |
| 3 目录与权限范围 | **worktree 可用性与进程权限范围分开验收**:worktree 只隔离工作文件与索引,同一 Unix 用户跨目录写不受它约束(见"已知限制"第 3 条) | ① 是 git 仓库且能建 worktree;② 进程写入边界:有容器/独立用户等受限范围,**或用户明确选择"允许同用户跨目录访问"这一授权范围**(选了就把无人值守第 3 条改述为该已选范围,不再宣称跨目录必确认);核不到受限范围又没明确选择 → 不过 |

三闸全过 → **无人值守参数组**;任一不过 → **有人在场参数组**,并在看板写明是哪条闸、什么证据不过:

| 参数组 | kimi-orch | codex-review | claude-exec |
|---|---|---|---|
| 无人值守参数组(三闸全过) | `--auto` | `--approve-for-me` | `--dangerously-skip-permissions --settings ${KIMI_SKILL_DIR}/claude-exec-deny.json` |
| 有人在场参数组(任一不过) | 不加 `--auto`(其语义是 Never Ask,见下) | **显式** `-a on-request -c approvals_reviewer=user`(不接受继承:省略 flag ≠ 人工批准——本机用户 config 实测 `approval_policy = "never"`;审批者另由 `approvals_reviewer` 决定) | **显式** `--permission-mode default`(≥2.1.283 不传 bypass 也可能继承 auto 或用户在设置里的 `permissions.defaultMode`,**不接受继承**)+ `--settings`(deny 只收紧不放松);启动后验收真实模式 |

```bash
# 参数组用 Bash 数组(本技能代码块一律限定 Bash;禁 eval——引号字符不得塞进字符串期待二次解析。
# 字符串拼 argv 是错的:Bash 遇空格路径会分词,把 --settings 的路径拆成两项;
# Zsh 更糟——标量根本不自动分词,整组变成一个参数。2026-10-09 实测,两模式两 shell 八样本)
# (a) 无人值守参数组——三闸全过
KIMI_ARGS=(--auto)
CODEX_ARGS=(--approve-for-me)
CLAUDE_ARGS=(--dangerously-skip-permissions --settings "${KIMI_SKILL_DIR}/claude-exec-deny.json")
# (b) 有人在场参数组——任一不过;把上面三行换成(codex 显式策略+审批者:见"模式选择"的验收条):
# KIMI_ARGS=(); CODEX_ARGS=(-a on-request -c approvals_reviewer=user); CLAUDE_ARGS=(--permission-mode default --settings "${KIMI_SKILL_DIR}/claude-exec-deny.json")
# 空数组 "${KIMI_ARGS[@]}" 展开为空(安全;不会变成空串参数)
```

失败分支不是"照旧带放权参数、只在看板记一句":

- 闸 1 不过 → **执行线禁 push 要用能力落实**:执行实例不持有 push 凭据,或远端只给线分支受限写权限;只在看板写"禁 push"不算;
- 闸 2 不过 → **不发出 bypass/auto 的启动指令**,改用有人在场参数组(claude **显式** `--permission-mode default`,codex **显式** `-a on-request -c approvals_reviewer=user`);
- 闸 3 不过(非 git / 无受限权限范围)→ 同上,退回有人在场模式,由人兜底。

**有人在场组是"显式人工模式 + 验收",不是"省掉 flag":**

- claude ≥2.1.283 不传 bypass 也可能继承 auto 或用户设置的默认模式——所以必须显式选人工批准;启动后**验收真实模式**:`herdr pane read "$P2"` 看状态栏或让 claude 跑 `/status` 确认当前是人工批准(不是 bypass、也不是 auto),证据落 `mode-selection.md`;**验不出人工模式 = 闸失败,停止派单**,先排障或先停再重来,不许带未知模式派活;
- codex **必须显式给审批策略与审批者**:`-a on-request -c approvals_reviewer=user`——省略 flag 是继承本机用户 config,不等于人工批准(本机实测用户级 `.codex/config.toml` 里就是 `approval_policy = "never"`:不向用户请求批准;审批者另由 `approvals_reviewer` 决定,user 才是问人);**on-request 不等于每条命令都问人**——它是"模型主动请求时才问",要让每批动作先由人放行,得另设派单闸(编排者逐单等用户点头再发);启动后 `herdr pane read` 核对状态栏/会话头显示的审批策略与审批者,证据落 `/tmp/$TAG/mode-selection.md`,验不出就停、不派单;
- **既有会话不适用启动参数**:权限模式是进程级属性——改 shell 变量、改看板记录都不改变已在跑的进程。闸失败时若目标代理已在跑(尤其已在 bypass/auto 里),**必须先停(退出/杀掉)再按人工模式重拉**,重验模式后才恢复派单,不许"改个变量就当降级了";
- kimi-orch 例外条款收口(第 0 步末条"当前会话当编排者"那个例外):`--auto` 的语义是 **Never Ask**——不打断用户、自动执行并自动决定;用户坐在终端前 = 有人在场,但**在场≠用户批准**,rename 只注册身份,既不改 CLI 权限模式、也不让 CLI 在动作前等人点头。当前会话已在 `--auto` 里跑、又过不了三闸时,只允许二选一:**(a) 记录式收窄(不是等价人工审批)**:用户把授权范围明确写进 `/tmp/$TAG/mode-selection.md`(授权哪些操作可自动执行、监督与止损方式),该分支按"**已授权自动执行 + 人类监督**"定性执行并如实标注,不再自称与人工审批等价;**(b) 没有这份记录 → 停工**。想让动作前真的等人批准,得另设外部派单/执行批准闸(本技能没有内建这一层)。

建独立 tab 并按固定布局拉起三个**全新干净**代理(不从旧会话续)。**先在主树建好各线 worktree,再按对应 cwd 建面板、起代理**(worktree 只隔离工作文件与索引,不隔 git 元数据,不是安全边界,见"已知限制";项目目录**非 git** 时退回主树同 cwd 启动,隔离保证降低,**且无人值守自动退回有人在场模式**——见"无人值守模式"启用闸第 3 条,并在看板记一笔):

```bash
# 0) 两线 worktree:审阅线 / 执行线各一(已存在则跳过);分支名带 tag,互不重叠。
#    只按"首次流程块"定下的路径方案执行:非 git 降级(main-only)跳过,三个 cwd 都用主树
if [ "$PATH_PLAN" = worktree ]; then
  git worktree add "$PWD/../wt-$TAG-review" -b "trio-$TAG-review"
  git worktree add "$PWD/../wt-$TAG-exec"   -b "trio-$TAG-exec"
fi

# 1) 新 tab:kimi 位留主树;根面板 P0 从返回 JSON 解析
P0=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$EXPECT_ORCH" --label <任务名> --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["root_pane"]["pane_id"])')
# 2) P0 右裂 → P1(审阅位);P1 下裂 → P2(执行位)——cwd 用 manifest 里的预期值(非 git 降级时=主树)
P1=$(herdr pane split "$P0" --direction right --cwd "$EXPECT_REVIEW" --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
P2=$(herdr pane split "$P1" --direction down --cwd "$EXPECT_EXEC" --no-focus \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')

# 3) 起代理:参数取自"模式选择"的数组;写死 bypass/auto 属流程违规
herdr agent start "kimi-orch-$TAG"    --kind kimi   --pane "$P0" -- "${KIMI_ARGS[@]}"
herdr agent start "codex-review-$TAG" --kind codex  --pane "$P1" -- -m gpt-6.1-sol -c model_reasoning_effort=high "${CODEX_ARGS[@]}"
herdr agent start "claude-exec-$TAG"  --kind claude --pane "$P2" -- "${CLAUDE_ARGS[@]}"
# 数组元素逐项成为独立 argv,含空格路径也不拆、空数组不产生参数;不要退回字符串拼接(见"模式选择"的实测)
# 模型 slug 与推理档必须分开传:slug 不带档位,档位用独立配置键;`--settings` 用技能目录变量拼路径、不写死用户名(展开后本机 = ~/.kimi-code/skills/trio-orchestrate),分享后随安装位置变化

# 4) 三个面板全部重命名(必做,漏了代理名与显示标签不一致);pane label 与 agent 名是两个对象
herdr pane rename "$P0" kimi-orch-$TAG
herdr pane rename "$P1" codex-review-$TAG
herdr pane rename "$P2" claude-exec-$TAG

# 5) bootstrap:任务目录/manifest/锁已在"检测"一步办妥 → 建看板骨架(仅缺时;重入不覆盖)→
#    取本次拉起批次号 bootstrap ID → 等 ACK 与预期全文全等(cmp)——这才是"拉起完成"的判据
TD="/tmp/$TAG"                             # 与"首次流程块"同一个任务目录;此处重设,本块可单独抽取自测
mkdir -p "$TD"                             # 幂等
SKILL_ABS=<发送方已知的技能绝对路径>        # 如 /home/<user>/.kimi-code/skills/trio-orchestrate;包内一律写绝对路径,不假设对端会展开 ${KIMI_SKILL_DIR}
# ══ bootstrap ACK 闭环块(起):整段抽出去、设好 TD/TAG/SKILL_ABS 即可单独跑(对端 herdr 可用桩替)══
# 秒+随机尾只是降低撞车概率,不承诺并发唯一;并发排他靠 launch.lock——一个 TAG 同时只许一个拉起流程
BOOT_ID=$(date +%s)-$RANDOM                # 本次拉起批次号:旧 ACK、失败残留靠它区分
[ -f "$TD/taskboard.md" ] || cat > "$TD/taskboard.md" <<'EOF'
# taskboard

## 待办(带验收线+冲突面)
## 在途(认领人+时刻)
## 待审
## 已验收(附证据路径)
EOF
rm -f "$TD/bootstrap-ack.md"               # 清掉旧 ACK:只认本次写出的回执
cat > "$TD/bootstrap.md" <<EOF
bootstrap-id: $BOOT_ID
技能文件:$SKILL_ABS/SKILL.md(绝对路径;先激活技能并读这个文件,不要依赖占位符自动展开)
tag: $TAG
看板:$TD/taskboard.md(骨架已建好)
角色全名:kimi-orch-$TAG(编排+验收)/ codex-review-$TAG(审阅)/ claude-exec-$TAG(执行,限 [机核])
指令:读技能与看板后,把 ACK 写到 $TD/bootstrap-ack.md。ACK 必须与下面五行**逐字节一致**(照抄、一行一项、不加别的字符、行尾不留空格、末尾保留一个换行):
bootstrap-id: $BOOT_ID
tag: $TAG
kimi-orch-$TAG
codex-review-$TAG
claude-exec-$TAG
不要再执行拉起流程(不要建新 tab、不要起新代理)。
EOF
# 预期 ACK 全文(与上面给对方的五行逐字节一致),验收用全等比较:
# 存在性 grep 挡不住"同一字段两个矛盾值 / 重复行 / 多余行"(2026-10-09 裁决),必须整文比
cat > "$TD/ack-expected" <<EOF
bootstrap-id: $BOOT_ID
tag: $TAG
kimi-orch-$TAG
codex-review-$TAG
claude-exec-$TAG
EOF
herdr agent prompt kimi-orch-$TAG "读 $TD/bootstrap.md 并按其指令执行" --wait --timeout 120000
ack="$TD/bootstrap-ack.md"
cmp -s "$TD/ack-expected" "$ack" || { echo "ACK 与预期全文不一致(矛盾/重复/多余行或缺行):拉起未完成"; exit 1; }
rmdir "$TD/launch.lock"                    # ACK 通过 = 拉起完成,释放排他锁;中途失败停下的,人工处理完再删锁重入
# ══ bootstrap ACK 闭环块(止)══
```

布局恒为:左 kimi-orch | 右上 codex-review | 右下 claude-exec(先 right 后 down,与宽窄无关)。

- 模型名以 codex 内 `/model` 列表为准;**模型 slug 与推理档必须分开传**:`-m <slug> -c model_reasoning_effort=high`。把 "GPT-6.1-Sol high" 整体当模型名会被 API 以 400 拒掉(ChatGPT 账号下 high 是推理档、不是模型名的一部分——2026-10-09 实测踩中:`-m` 当时不报错,首次发请求才炸);slug 以本机 `/model` 列表为准,该模型的默认推理档可能不是 high,必须显式设置;`-m` 被拒就起默认模型后在面板里 `/model` 手动选模型+推理档,并在看板记一笔——人工补救可以,但不算"一键/无人值守"的正常路径;
- **状态语义全文统一**(第 0 步、第 3 步、重启检查点同一套):**可接收新单 = `idle` 或 `done`**(herdr 的 done 同样可输入);`working` 永不作为判据,只是辅助观测;`blocked` 是排查分支;
- `agent start` 返回 ready 才算数;起不来先 `herdr pane read` 看卡哪(codex 常卡升级提示,Esc 按掉);**启动失败先分清两种**:参数校验失败、进程根本没创建 → 可复用该 shell pane 修参数重发;进程已在但 blocked/not-ready → 先排障或等它退出,不要盲目重发;
- 裂 pane 成功但 `agent start` 失败(如 invalid_agent_name,此时进程未创建):面板不用重建,修好参数对同一 pane 重发 `agent start` 即可;
- 验收:`herdr pane read "$P1" | tail` 状态栏须见 `<模型> <推理档>` 两段,**再发一条无关紧要的测试消息确认不弹 400** 才算真可用(只看模型展示名不算数);P2 须见 bypass permissions on(仅无人值守参数组)——**这只证明权限模式,不证明 deny 清单已挂载**。清单验收分两层:文件存在且 JSON 合法是**语法检查**(`python3 -c "import json;json.load(open('...'))"` 一行验);**效果检查**是编排者 `herdr pane read "$P2"` 或让 claude 跑 `/status`(`/permissions`),看 **settings sources 与生效规则列表**,把生效规则条目贴进看板当证据——语法检查 ≠ 效果检查,两回事。验不到有效规则/清单未生效 → **暂停派单**:已在跑的 claude 先停(退出/杀掉)再按有人在场组(显式 `--permission-mode default`)重拉,重拉后重验模式与规则,验过才恢复派单——只改 shell 变量或只看板记一笔都动不了旧进程,不算降级;
- 第 5 步的完成判据是 **ACK 与 `/tmp/$TAG/ack-expected` 全文全等(`cmp -s`;内容 = bootstrap ID、tag、三角色全名五行)**——矛盾字段、重复字段、多余行全拒,不是某个状态观测,也不是存在性 grep:`agent get` 只作辅助观测——短 bootstrap 在 `--wait` 返回后通常已是 idle/done,等不到 working(旧版曾把 working 当硬条件,2026-10-09 判定为错误);ACK 齐全后再告知用户切到新 tab 把任务下达给 kimi-orch(它就是后续编排者;本条拉起会话只负责搭建);
- 例外(**前置:模式选择已过;或按"模式选择"做过记录式授权——用户把授权自动执行的范围写进 `/tmp/$TAG/mode-selection.md`**;`--auto` = Never Ask、在场≠用户批准):若用户就是在某个 kimi --auto 会话里加载本技能、且要它本人当编排者,则不建新 tab——**先用 `herdr agent rename "$HERDR_PANE_ID" "kimi-orch-$TAG"` 注册 agent 身份**(pane/tab 的 rename 只是显示标签,与 agent 名是两个不同对象;只改标签不会把本会话注册进角色名),再以当前面板为 P0、按上面的 worktree/cwd 规则裂 P1/P2、起两个代理、三个面板照旧全部重命名,**tab 也 `herdr tab rename` 为任务名**(该 tab 还混有其他无关 pane 时,改为在原名后追加 `<TAG>`)。该例外不豁免模式选择:三闸不过、且没有上面的记录式授权时,**停用例外并停工**(见"模式选择");rename 只注册 agent 身份,不改变本会话的 CLI 权限模式,也不构成用户批准。

## 替补组队(三件套不齐时)

- **kimi-orch 不可替**:技能就跑在 kimi 里,但二进制检测仍要走(当前会话能跑 ≠ PATH 里有可新起实例的 kimi);缺的是审阅/执行才走本节;
- **可用性两级检测,缺一不可**:① `command -v <cli>` 二进制在(PATH 里有能新起实例的 CLI;当前会话里能跑不算);② `herdr integration status` 显示 `current`(状态钩子已装)。缺②可 `herdr integration install <agent>` 补装;补不了则降级为备胎——能起跑但 idle/working 侦测不可靠,通信协议第 3 步 `agent get` 的观测失效,只能 `pane read` 人工盯;
- **候选要记录**:每个候选写明 CLI kind + provider/模型 + 角色能力与权限边界;"不同后端"靠这两项判定,不能只看二进制名;替补顺序(通用优先级;每台机器以两级检测实测为准,下面标注仅为示例):

| 角色 | 首选 | 替补顺序 |
|---|---|---|
| 审阅 | codex | copilot → opencode(先补钩子)→ 第二个 kimi |
| 执行 | claude | codex → copilot → opencode(先补钩子) |

- **硬约束:审阅与执行不得同后端**——同源盲点,审了等于没审。这里的"不同后端"= **不同 CLI 且不同模型族**(同 CLI 换模型、或不同 CLI 连同一个 provider/模型,都算同源);
- **选角用枚举,不用贪心**:把审阅候选 × 执行候选列成对,取**第一个满足"不同 CLI 且不同模型族"的合法对**(两个角色也不能是同一个 live agent)。反例:只有 codex 与 kimi 可用时,合法解是 **kimi 审 + codex 干**;若按"审阅固定首选、只把执行顺位下移"的贪心,会误判成无解;
- 候选健康检查与主后端同级(二进制 + integration `current` + 账号/模型可用性);枚举完仍无合法对,**明确报告无法组队并停,不硬组**;
- 角色名不变:`codex-review-<tag>`/`claude-exec-<tag>` 是**槽位名不是后端名**(协议、看板、通信全部不动),实际后端与 kind 写进看板头;`--kind` 传实际后端(herdr 支持列表以 `herdr agent start --help` 为准);
- 替补 CLI 的免确认参数各异(codex `--approve-for-me`、claude `--dangerously-skip-permissions`,其他以该 CLI `--help` 实测为准);拉起后 `herdr pane read` 核对权限模式,并验收实际生效的模型/推理档,记看板;
- claude 缺席时 `claude-exec-deny.json` 的 `--settings` 挂载自动失效——替补后端没有等价机制,**该线的无人值守能力随之停用**(改为有人在场,或另配进程级沙箱),不能只把撤防记一笔就继续沿用无人值守纪律。

## 第 1 步:立看板

`/tmp/<TAG>/taskboard.md`(TAG 即第 0 步定下的任务 tag),四列:待办(带验收线+冲突面)/ 在途(认领人+时刻)/ 待审 / 已验收(附证据路径)。规则写进看板头:

1. **单写者:taskboard.md 只由编排者(kimi-orch)写**;工作代理不直接编辑看板,只把结果写到 `/tmp/<TAG>/` 下自己的结果文件(如 `<task-id>-result.md`),由编排者按指针移列——多方共写同一份 Markdown 会互相覆盖;
2. 认领写名,移列附证据(测试数/哈希/文件路径),口说无凭;
3. **证据绑代码版本**:每张单带 task ID 与 **base commit**;回执必须带 **tested commit** + 产物路径 + 校验值;审阅与合并核对同一 revision——代码再变(工作树脏了、HEAD 移动),旧验收即失效,要重跑重签;
4. claude-exec 只认领标 **[机核]** 的条目(验收线机器可核:测试数/哈希/闸脚本);它的文字汇报不是证据;
5. codex-review 拉"待审"列,裁决落盘 /tmp 并发指针;
6. **提交纪律(全角色统一)**:每线独立索引;`git add` 只按任务路径,不用 `git add -A`;提交前核对 `git diff --cached`,确认只含本任务改动,不做不看 staged 内容的裸提交;线标前缀与实际角色一致:`[kimi-orch]`、`[codex-review]`、`[claude-exec]`;改名/移动必须随提交原子完成;
7. 全量套件串行占坑;多线施工用 `git worktree` 双缓冲;
8. 预登记数(先写死预期再跑)优先于事后解释。

## 无人值守模式(全自动协作的预授权)

用户不在场、没人点确认时,修改/删除/git 操作**预授权放行**。先分清三层:**权限模式**(bypass)只省确认,**隔离**(每线 worktree)只分离工作文件与索引,两者都**不是安全边界**(见"已知限制");真正的硬边界是进程级沙箱/容器 + 受限凭据 + 远端 branch protection。

**启用闸(条件开,不是默认开)——三条任一不满足,就退回有人在场模式,不许带默认放权开工:**

| # | 启用前必须满足 | 不满足时的行为 |
|---|---|---|
| 1 | push 硬边界:远端 main/master 已设 **branch protection**(或推送只用权限受限凭据) | 执行线**禁 push**,改动只留本地分支,由编排者人工推送 |
| 2 | 凭据面:运行环境不挂载非必需凭据(或用独立 Unix 用户/容器隔离) | 不启用无人值守 |
| 3 | 目录是 git 仓库、能建 worktree(非 git 目录没有可用的隔离线) | **非 git 目录自动退回有人在场模式**——这是降级路径,不是等价替代 |

三条都是**开工前的检查项**,不是事后说明;没验过就按不满足处理。三条的逐项证据写法与"三闸全过用无人值守参数组 / 任一不过用有人在场参数组"的落地见第 0 步"**模式选择**"——启动参数由那里选定,不在这里默认放权。失败退回的有人在场组**必须显式给 claude `--permission-mode default` 并验收真实模式**:不传 bypass ≠ 人工批准(≥2.1.283 可能继承 auto 或用户默认模式),既有会话改变量也不生效,得先停再重拉。

1. **每线一个 git worktree**(启用闸第 3 条满足后才是前置条件,由编排者在第 0 步先建好再启动各代理):各线独立分支、独立索引,`commit -a`/`stash`/`reset` 的作用范围限于本线工作文件——但见"已知限制"第 3 条,git 元数据不隔;
2. 主树只由 kimi-orch 合并/落盘;claude-exec/codex-review 不动主树——这是**纪律约定**,机制上拦不住(执行线实测能改主树引用),靠提交前核对与验收兜底;
3. 删改自由限于自己 worktree 和 `/tmp/<TAG>/` 任务目录内;跨出去删改仍需用户在场确认——这是**默认授权范围**;若按"模式选择"第 3 闸由用户明确选定了"同用户跨目录访问",在此改记为"用户已选定的授权范围",不再同时宣称跨目录必确认;
4. push 只准推各线分支并随看板报告;推 main/master、force-push 永禁——但 **deny 清单只是文本前缀软防线,可被等价写法绕过**(实测形态见"已知限制"第 1 条);要真永禁,在远端设 branch protection / 用权限受限凭据;
5. deny 清单只留永禁项(sudo/毁盘/force-push/推 main/读机密)——无人值守下长清单只会卡死跑批;真兜底是机核验收+worktree 隔离+提交纪律(三者的边界见"已知限制")。

## 已知限制(承诺降级,老实写清)

1. **push 永禁不是硬拦**:`claude-exec-deny.json` 里的 `Bash(git push ...)` 只是**文本前缀规则**,是软防线,可被等价写法绕过;各绕过形态按方法区分(不笼统说"已实测")——`git push origin HEAD:main`(本地 bare remote 的 main 被实际改写)与 `git push origin --force` 是**真实 Git 执行**,证据 `/tmp/skill-review/evidence.json`;`git push upstream main`、`+refspec` 强推、`git -C <目录> push ...` 包装、无参数 `git push`(看 upstream/refspec 配置)只是**前缀规则文本建模**(命令串与规则清单比对,无真实 Git 执行记录);以上形态在 **Claude 权限引擎内均未实测**。`--force`、`-f`、`--force-with-lease` 是三条独立前缀,列不全。真正永禁只能靠**远端 branch protection / 受限凭据 / 受控推送入口**,按解析后的目标 ref 与 force 语义执行;
2. **读机密只覆盖 cwd 内直接读**:`Read(./**/...)` 规则管不到 `~/.ssh/`、CLI 凭据目录,也管不住**间接读取**:子进程(Python/Node)自己打开文件、没有显式列出文件名的搜索。危险形态用**枚举**方式进 deny(`.env.local`/`.env.production`/`.env.development`/`.env.staging`、`credentials.json`、`*.key`、`id_rsa` 等),模板名(`.env.example`/`.env.sample`/`.env.template`)因此天然不被命中——权限按 deny → ask → allow 评估,**宽 deny 压得住窄放行,别指望白名单豁免**;枚举天然列不全,无人值守环境**不应挂载非必需凭据**;读取边界要靠进程级文件权限/沙箱落实,任何配置都不能声称枚举完了"所有机密";
3. **worktree 不是安全边界**:它只隔离工作文件与索引,**git 元数据(common dir 下的 config/refs)与主树共享**——实测执行线 `git config` 会被主树读到、`git update-ref refs/heads/main` 能改主树 main 引用、也能推本地远端 main。所以"主树只有编排者能动"是**纪律约定,机制上不成立**;要硬边界得用进程级沙箱/容器 + 独立 Unix 用户 + 受限远端凭据,并由用户明确选择授权范围后启用;
4. **替补后端没有等价控制**:claude 缺席时 `--settings` 挂载失效,替补 CLI 没有同等 deny 机制;此时该线无人值守能力应停用或改为有人在场(见"替补组队"末条);
5. **`/tmp` 可能被系统清理**:看板、ACK、裁决都在 `/tmp/<TAG>/`,重启或清理服务会删;关键 manifest、最终裁决、恢复指针要另存项目内持久位置(见"上下文/token 纪律"第 1 条);
6. **心跳 cron 归属会话**:计划任务属于当前会话,新建会话不继承,需重建(见第 3 步末条)。

以上第 1–3 条对应的硬边界(远端保护、凭据隔离、worktree 可用)没有验收之前,按"无人值守模式"的**启用闸**处理:退回有人在场模式,不默认放权。

## 第 2 步:分层派活(按判断密度,不按面板)

| 层 | 内容 | 派给 |
|---|---|---|
| A 判断密集(调试/设计/取舍) | kimi 子代理(Agent 工具,可并行多开) |
| B 规格已死的实现 | kimi 子代理,或 claude-exec+严规格 |
| C 机械执行(跑套件/重生成/读大文件/对表) | claude-exec |
| 审阅 | codex-review |

**派活分级的判据(2026-10-09 固化)**:任务书里还含"该怎么设计"的方案决策(选哪种实现、怎么拆、阈值定多少)= **A 层**,留给 kimi 子代理——**不许标 [机核] 派给 claude-exec**;[机核] 只标做法已写死、验收线机器可核的 C 层活(命令、路径、期望值齐全,不需要执行者做任何判断)。把设计活写成 [机核] 下达,等于让执行者替你做设计决策,回执再"机器可核"也补不回来。

额度纪律:大文件阅读、长任务跑批放 C 层。子代理报 403 先**按类型分**——配额墙 / 鉴权失败 / 网络故障:配额墙先保存进度、退避等待,恢复后再用 `Agent(resume=...)` 续跑,不重开;**不做无限重试**;鉴权/网络按各自原因处理;换了模型/代理/参数,记为新的执行条件,不自动继承旧验收。

## 上下文/token 纪律(上下文是缓存,看板是真相)

自动压缩在接近上限才触发(70% 不动是正常的),且压缩本身也要花额度——**少积累+状态外置+到期重启比压缩更有效**:

1. **状态全外置**:任务书、裁决、批次结果全部落 `/tmp/<TAG>/`;任何 agent 可杀可重启,重启后按**已持久化的检查点**(看板+指针文件+ACK/结果文件)恢复。注意 `/tmp` 可能被系统清理:关键 manifest、最终裁决、恢复指针要另存项目内持久位置;
2. **双向落盘**:不只下发落盘,agent 的**产出也落盘**——汇报只许回"结论+证据路径+关键数字",禁止把大段文件/日志贴回对话(贴一次,后面每轮都带着它烧);
3. **prompt 瘦身**:窗格只发短指针+明确输出格式(结论先行、几行封顶),任务书再长也是一个路径;
4. **C 层短生命周期**:claude-exec 是**无状态执行器**,一批机核活干完就重启(`/exit` 后 `agent start` 重拉),不留恋上下文;它的记忆在看板,不在会话;
5. **审阅层按需重启**:codex-review 的裁决历史必须在 `/tmp/<TAG>/` 的裁决文件里,重启按已持久化检查点恢复;跨任务/积压久了就重启;
6. **阈值巡检 + 重启状态机**:阈值以各 CLI 当前配置为准(不写成所有后端的通用事实)。编排者心跳 `pane read` 看状态栏;重启只在安全检查点做:确认该 agent 处于**可接收状态(idle 或 done)**、产物/未提交状态已落盘(working 中不发 `/compact`、不打断任务)→ 发 `/compact`(claude/codex 均支持),无效才重启。重启序列:`/exit` → `pane read` 确认**回到 shell 提示符**(这只说明屏幕回到 shell,**不等于注册状态已释放**)→ `herdr agent list` 核对旧名已从注册表消失 → 再 `agent start`;进程在但 blocked/not-ready 时先排障或等它退出,不盲目重发。

## 额度耗尽预案(quota 是可调度资源,不是事故)

额度(5h 窗/日配额)耗尽与网络断连一样是**常态外部事件**,处理原则是吸收不是硬扛:

1. **检测进心跳**:编排者 `pane read` 抓额度信号(codex 状态栏 `5h limit: N% left · resets at HH:MM`——**这是作者环境的示例,不是检测契约**:状态栏项可配置、版本/认证方式各异,以 `/status` 实测为准;未显示或取不到就标**未知**,不当成充足);kimi 403 只是候选信号,按"额度纪律"条先分"配额/鉴权/网络"再进处理链。同一账户可能同时有多个限制(如 5h 窗+周限),分别观察。**<30% 记看板并收紧派单**(只派关键单;关键验收暂停——这是本方调度政策,不是厂商保证的安全阈值);
2. **额度池独立性要核实,不能从 CLI 名推**:不同 CLI 可能挂同一计费账户(Claude 用 API key、Copilot BYOK 时,两个 CLI 烧的是同一池)。替补前按 **provider + 认证方式 + 计费账户/组织 + 额度桶与窗口** 逐项核对,确认候选不受原耗尽桶约束才算额度容灾;换客户端本身不算。核实记录不含密钥;
3. **换脑恢复的前提是状态外置**:看板+manifest+task ID 在盘上,替补按**最后已确认检查点**恢复——注意中断现场可能是"执行完但未回执",接手前先核对产物/进程/提交并确认旧执行者已停写;必要重跑登记进看板。恢复承诺限于"不重复已核实完成的工作",不承诺任意中断都无损;
4. **预案链按角色**:审阅断 → ①等重置窗口 ②备援审阅(借调,先知会其编排者)③替补审阅(降级标注,**不作最终放行签**);执行断 → 按替补组队换后端(审阅/执行不同后端约束仍适用);
5. **质量档不为省额度而降**:终审/关键闸保持既定推理档;额度末班车上不跑关键验收,宁可等重置。(作者 2026-10-09 单次观察:medium 档低余量下仍抓出确定性 bug——bug 本身证据 /tmp/skill-review/verdict-r5.md;档位与余量数字是编排者当时 `pane read` 状态栏的观测,**未留档、未独立核实,照实标注**;这是单次观察,不是 medium 普遍充分的证明;low 未验过,终审不用它);
6. **额度入看板**:大单派出前记录当前余量、额度桶/窗口与重置时刻(这是预登记);事后的消耗趋势是复盘数据,两者分开记。

## 第 3 步:通信协议(送达与重递判据)

- **每张单带 task ID**(T001/T002…,与看板条目一致);发送前确认对端处于**可接收状态:`idle` 或 `done`**(done 同样可输入);`working` 时不发(那是上一单在跑),`blocked` 先排查;
- 发送优先 `herdr agent prompt <名> "..." --wait --timeout <ms>`(带有限超时);检查返回里的 blocked/stalled/timeout;
- **受理/完成的判据是落盘的 ACK/结果文件,且必须绑 task ID + 当次批次**(批次号/bootstrap ID);不看单次状态观测:`working` 只是**辅助观测**——短任务可能在 `agent get` 之前就跑完(永远看不到 working);对端原本就在 working 时,看到 working 也不能证明新单已开始;
- 重递前先核对 ACK 与产物(结果文件、进程、提交),**任务必须幂等**(可安全重跑),避免超时重递变成重复执行;
- 长正文/任务书**落盘 /tmp/<TAG>/ 发路径**,窗格只发短指针(窗格通道实测会丢长文);
- codex 后端偶发 connection failed:先按自己的网络出口排查(作者原环境见附录);确认是网络问题、且上一单确未被执行,再按上面的重递流程重递;仍不可用 → **备援审阅**(向另一线的审阅后端借调,先知会其编排者,召回即还);
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

> 任务:<一句话>。我(kimi-orch)编排+验收;已建 codex-review(高推理档)与 claude-exec(限 [机核])。
> 看板:/tmp/<TAG>/taskboard.md。纪律:线标提交、路径 add、串行全量、落盘发指针。
> 派单带 task ID 与 base commit;回执以落盘 ACK/结果文件为准,`working` 只当辅助观测。

## 发布前 smoke checklist

分享/发布前逐项过。证据分四档,弃用笼统的 [已实测]:**[真实执行]** = 实际运行过命令/脚本,留有输入与结果路径;**[前缀建模]** = 只把命令串与规则清单/谓词做文本比对,没有真实运行;**[静态推演]** = 手工推理或一次性枚举,未固化成可重跑的记录;**[未实测]** = 尚未运行,只是规则要求。协议版本:**2026-10-09-r5**。

| # | 检查项 | 状态与证据 |
|---|---|---|
| 1 | 参数校验:空 / 超长 / 非法字符 / **多行注入**全被拒;且**模型闸先于任何含 TAG 的命令** | **[真实执行]** 2026-10-09:bash `[[ =~ ]]` 全字符串 8 例逐例跑过,1 放行 7 拒绝(证据 `/tmp/skillfix/r3-evidence/tag-validation.txt`);"展开值先经模型阅读判断"这条本身[未实测](需真调 Kimi 模板引擎喂非法 $tag) |
| 2 | 同 tag 重入:三名字精确匹配 + 同 tab + **逐角色 cwd == 预期 worktree(两边 `os.path.realpath()` 规范化后比较,带 `..` 的拼法与规范路径指向同一目录须判等)**;缺 `tab_id`/`workspace_id` fail-closed | **[真实执行]** 2026-10-09:6 样本——**新正例**预期带 `..`、实际 cwd 为规范路径 → 0;同 tab 且 cwd 全对 → 0;cwd 全错 → 3;缺拓扑字段 → 3;跨 tab → 3;缺角色 → 2(证据 `/tmp/skillfix/r4-evidence/reuse-realpath.txt`;r3 拓扑 5 样本 `/tmp/skillfix/r3-evidence/reuse-topology.txt`);**补建与失败回滚**[未实测] |
| 3 | 部分建队失败:进程未创建可同 pane 重发;进程在但 blocked/not-ready 先排障 | [未实测] |
| 4 | 替补无解:枚举所有 (审阅, 执行) 候选对后才报无解 | **[真实执行]** 2026-10-09 脚本:审阅候选=[codex,kimi]、执行候选=[codex] 时,贪心误报无解、枚举得 kimi 审+codex 干(脚本与输出:`/tmp/skillfix/r3-evidence/role-pair-enum.txt`);实机演练[未实测] |
| 5 | deny 负例(B1):`git push origin HEAD:main`、`git push origin --force` | **[真实执行]** 临时 Git 命令真跑:本地 bare remote 的 main 被实际改写(证据 `/tmp/skill-review/evidence.json`) |
| 5b | deny 负例(B1)其余形态:`+refspec`、`git -C` 包装、`upstream main`、无参数 `push` | **[前缀建模]** 命令串与规则清单文本比对,无真实 Git 执行记录;权限引擎内端到端负例[未实测] |
| 6 | deny 负例(N1):模板名 `.env.example` 不被任何 deny 规则命中(枚举危险形态,不靠白名单) | **[前缀建模]** 规则交集比对(fnmatch,2026-10-09);引擎内最终判定[未实测] |
| 7 | 重启恢复:`/exit` → 提示符 → `herdr agent list` 核对旧名已释放 → 重拉 | [未实测] |
| 8 | 模型参数:`-m <slug> -c model_reasoning_effort=high` 生效(状态栏两段 + 测试消息不弹 400) | 修复后生效待验[未实测];踩坑(整串当模型名 → 400)**[真实执行]** 2026-10-09 |
| 9 | bootstrap:ACK 与 `/tmp/<TAG>/ack-expected` **全文全等(`cmp -s`)**才算"拉起完成"——矛盾字段、重复字段、多余行、缺行、行尾空格全拒;任务身份看 manifest(项目根绝对路径 + 三角色预期 cwd + 创建时间 + bootstrap ID),重入**唯一解析**:必需字段每条恰好一行、值非空、同名字段出现两次即拒;读写任务目录/文件前核验非符号链接(`[ -L ]`)且属主是自己;一个 TAG 同时只许一个拉起流程(launch.lock) | **[真实执行]** ACK 7 样本 r4 不变(证据 `/tmp/skillfix/r4-evidence/cmp-ack.txt`);**r5 新谓词重测**:manifest 唯一解析 8 样本 + 目录安全 5 样本(证据 `/tmp/skillfix/r5-evidence/manifest-unique-parse.txt`、`/tmp/skillfix/r5-evidence/dir-guard.txt`,旧 r4 manifest 4 样本由新证据取代);实机拉起 ACK[未实测] |
| 10 | 派单状态语义:`done` 状态可收单;ACK 绑 task ID + 当次批次;`working` 不作判据 | [未实测] |
| 11 | 模式选择:三闸证据落 `/tmp/<TAG>/mode-selection.md` → 全过用无人值守参数组,任一不过用有人在场参数组(不对执行线发 bypass/auto;**有人在场时 claude 显式 `--permission-mode default`、codex 显式 `-a on-request -c approvals_reviewer=user`,都不接受继承本机未审默认**) | **[静态推演]** 两参数组按闸结果二选一为人工推演;实机按两模式各拉起一次[未实测] |
| 12 | claude 有效规则**效果检查**:`/status`(`/permissions`)看 settings sources 与生效规则,条目贴看板;验不到 → 退回有人在场并**先停再按显式人工模式重拉重验**;启动后验收真实模式(`pane read`/`/status`),验不出人工模式 = 闸失败停止派单 | [未实测](第 6 行的规则交集比对是[前缀建模],不等于引擎内生效) |
| 13 | 参数组为 bash 数组、`"${ARR[@]}"` 逐项成 argv:含空格路径不被拆、Zsh 标量不分词问题不复现;空参数组 `=()` 展开为空 | **[真实执行]** 2026-10-09:bash 5.2.21 与 zsh 5.9 × 两模式 × 两路径(含 `/tmp/skill space`)共 8 样本——数组形式两边都逐项正确;旧标量串在 bash 空格路径被拆成两项、在 zsh 整组变成一个参数(证据 `/tmp/skillfix/r4-evidence/argv-arrays.txt`) |
| 14 | 有人在场落地:显式 `--permission-mode default` **且**启动后验到真实人工模式才恢复派单;codex 显式审批策略/审批者并 `pane read` 核对显示(`on-request` 只保证"请求时才问",不等于每条命令都问人);既有会话不适用启动参数、先停再拉;kimi-orch rename 例外收窄为"模式验收已过 或 用户把授权范围落盘 `mode-selection.md`"(`--auto`=Never Ask,在场≠用户批准) | [未实测](文档依据:claude `v2.1.283` 起不传 bypass 也可能继承 auto 或用户 `defaultMode`;本机 codex 用户 config `approval_policy = "never"` 为只读窄读) |
| 15 | **先分类后取锁**(N3):空目录 → 分类"新建" → 取锁 → 建 manifest → ACK 全等 → 释放的完整链;残留目录(非空无 manifest)在取锁前被拒;同 TAG 并发抢锁被拒;全程不许删锁换绿 | **[真实执行]** 2026-10-09:"首次流程块"整段从 SKILL 抽出真跑(真实文件系统;对端 ACK 用桩,未起代理),4 分支 + 闸的顺序探针(分类行号 < 取锁行号),证据 `/tmp/skillfix/r5-evidence/lock-flow.txt`;真代理拉起[未实测] |

## 附录:原作者经验(不是通用事实,按自己的环境验证)

- 额度:kimi/codex 额度较紧,大文件阅读、长任务跑批尽量放 C 层;
- 后端选择:claude-exec 挂什么后端各人自定,输出不稳的后端只配机核活;
- 2026-10-08 共享索引险情:`git add -A` 卷走了别线 staged 的改动;
- 2026-10-09 实测踩中:tag 超长报 `invalid_agent_name`;早期的静默截断让两个 tag 撞名(已删);`-m "GPT-6.1-Sol high"` 整体当模型名,`-m` 不报错、首次发请求才炸 400;
- 窗格通道丢长文:作者环境一天丢过四次同一条,所以一律落盘发指针;
- 网络:作者环境 codex 走 VPN、kimi/deepseek 不走——只有 codex 掉线时先查 VPN;各人按自己的出口情况类比排查。
