# trio-orchestrate

一键拉起并编排一组参数化编队的智能体协作(默认 kimi 编排 / codex 审阅 / claude 执行;**编排者后端可在已判绿名单 kimi/claude 中按 roster 指定**)。

## 1. 简介

trio-orchestrate 是一个多智能体协作技能:把一项任务分给若干**槽位**,各守其位——默认编队里 kimi 负责编排与验收,codex 负责独立审阅,claude 负责执行。

从 r6 起,成员不再由写死的三个名字固定,而是由调用时给出的 **roster 声明**决定:槽位名(协议身份)与后端 CLI(实际干活的程序)彻底解耦,编队可以是 2~N 名成员,可选成员里含 genox。从 r7 起,**编排者也不再写死成 kimi**——roster 里把 `orch` 指到**已判绿名单**(kimi/claude)内的任一后端即可;进名单要逐格跑过 `SKILL.md` "编排者档案"的能力矩阵(能收 ACK、能派活、能自己活着),"有状态钩子"或"能跑起来"都不够,kimi 额度耗尽时可直接换 claude 顶 orch。

| 槽位 | 能力 | 默认后端 | 职责 |
|---|---|---|---|
| `orch` | 编排 + 验收 | kimi(或名单内其它后端,如 claude) | 派活、复核关键数字、守看板;全 roster 唯一,不可替换;后端必须取自**已判绿名单**(kimi/claude,见 `SKILL.md` 的"编排者档案"),用它当 orch 时若该后端默认模型最弱(如本机 claude)必须显式声明 model |
| `rev` | 审阅 | codex | 独立审阅,裁决落盘并发指针 |
| `exec` | 执行 | claude | 只认领做法已写死、验收线机器可核的 [机核] 条目 |
| `rev2` / `revtest` / `exec2`…`exec9` | 扩展位 | 自定 | 第二条审阅线 / 审阅+测试复合位 / 更多执行线 |

实际代理名 = **`<槽位>-<任务tag>`**(如 `orch-g07`、`rev-g07`、`exec-g07`)。

技能依赖 herdr(终端多路复用)建面板、起代理、传话与盯状态:用户只需在**任一已判绿的编排者后端**会话里加载本技能(kimi 宿主 `/skill:trio-orchestrate`,claude 宿主 `/trio-orchestrate`),其余面板自动建好,布局为 orch 在左、其余成员沿右列自上而下(默认编队即左 `orch` | 右上 `rev` | 右下 `exec`)。

设计上有三条主张:一是**验收靠落盘证据**(ACK 全文全等比较、结果文件、复跑复算),不靠状态观测与口头汇报;二是**放权有条件**——是否采用无人值守参数组,由开工前的三道闸决定,不是默认值;三是**审的人不能和干的人同源**——roster 里审阅能力成员与执行能力成员不得同 CLI、不得同模型族。

## 2. 安装

把**整个 `trio-orchestrate` 目录**(含 `claude-exec-deny.json`)拷贝到**你要用来当编排者的那个后端**的技能目录:

```bash
cp -r trio-orchestrate ~/.kimi-code/skills/     # kimi 宿主
cp -r trio-orchestrate ~/.claude/skills/        # claude 宿主(或项目内 .claude/skills/)
```

拷完核对一下哈希,防旧副本混装(任务看板头也记这一份):

```bash
sha256sum ~/.kimi-code/skills/trio-orchestrate/SKILL.md   # 与看板头"技能版本"一行的 sha256 对比
```

`claude-exec-deny.json` 是执行线的永禁清单,必须与 `SKILL.md` 一同拷贝;缺了它,执行线就失去这层权限约束。**发布/换装 = 整目录拷到目标后端的技能目录,再核对哈希一致**;多个后端各装一份时,注意"这次任务是哪份技能文本拉起的"以看板头的 sha256 为准。

## 3. 前置依赖

- **本机任一已判绿的编排者后端**(必需):kimi 或 claude——技能本体跑在哪个后端里,就由那个后端顶 `orch`(默认编队的编排者是 kimi);判据=能力矩阵(能收 ACK、能派活、能自己活着),见 `SKILL.md` 的"编排者档案";本机 claude 默认模型是最弱档,**用 claude 当 orch 必须在 roster 里显式声明 model**;
- **herdr**(必需):负责标签页/面板、代理注册与状态侦测;`herdr integration status` 显示状态钩子已装,idle/working 侦测才可靠,否则只能人工 `pane read` 盯;
- **codex / claude**(默认编队的审阅线与执行线后端,可换):缺任一个时走"组队与替补"——按能力枚举候选池的全部组合,硬约束是**审阅与执行不得同 CLI 且不得同模型族**(同源盲点,审了等于没审),且每个 exec 成员都要有一个非同源的签字成员;`orch` 槽位不可替换(缺 orch 一律停下来等用户,不自动补);
- **genox**(可选):只能占 rev / exec 一类槽位,**不能占 orch**(不在已判绿名单内),且只有"有人在场"档可用(herdr 不认 genox,没有状态钩子),详见 `SKILL.md` 的"后端档案";
- **各自登录态**:使用前各 CLI 需在本机已登录。首次使用前请人工跑通一遍:各 CLI 登录、仓库 trust 确认、bypass 模式的首次确认,都不能留到无人值守时段做。

## 4. 快速开始

```
/skill:trio-orchestrate <tag> <roster>       # kimi 宿主
/trio-orchestrate <tag> <roster>             # claude 宿主
```

(claude 宿主靠技能正文的同名 `arguments:` 声明与文本替换拿到参数;无技能宿主——codex/copilot 等——由用户把 tag/roster 的**字面值**写进消息,编排者先写死字面值再执行;三条路与"取不到参数不许静默回落"的铁律见 `SKILL.md` 的"编排者档案"。)

两个参数都可省:`<tag>` 是任务标签(只允许 `[a-z0-9_-]`,不超过 24 字符),同时用作标签页名称与任务目录 `/tmp/<tag>/`;`<roster>` 是编队声明,省略则用默认三成员编队。

| 想拉起的编队 | 怎么写 |
|---|---|
| 默认三件套(kimi 编排 / codex 审 / claude 干) | `/trio-orchestrate g07`(roster 留空)或 `/trio-orchestrate g07 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude` |
| 2 成员:编排 + 审阅/执行合一 | `/trio-orchestrate hotfix orch=kimi+revtest=codex:gpt-6.1-sol:high` |
| 4 成员:双执行线,第二条挂 genox | `/trio-orchestrate bench4 orch=kimi+rev=codex:gpt-6.1-sol:high+exec=claude+exec2=genox` |
| 换后端顶 orch(如 kimi 额度耗尽) | `/trio-orchestrate tune2 orch=claude:claude-opus-5-5+revtest=codex:gpt-6.1-sol:high`(orch=claude **必须显式声明 model**) |

roster 是**单字符串**,成员之间用 `+` 连接,每个成员写成 `<槽位>=<cli>[:<模型>[:<推理档>[:<模型族>]]]`——例如 `rev=codex:gpt-6.1-sol:high` 表示审阅槽位跑 codex 的 `gpt-6.1-sol`、推理档 `high`;`exec=claude` 只给后端、其余用该 CLI 的默认值。

拉起流程概述:

1. **参数与组队校验(必做)**:tag 与 roster 先过模型闸与全字符串正则;roster 再逐个查槽位词表、成员数 ≥2、三种能力齐全、**orch 恰好一个且其后端在已判绿名单内**(orch=claude 还须显式声明 model)、**审阅与执行不同源**、**每个 exec 成员都有非同源签字成员**(冲突会按写死的措辞报错);任一不过即停;roster 留空才回落默认编队,且回落前会核默认编队的 orch 后端在本机可用;
2. **模式选择(必做)**:按三道闸逐项核验并把证据写进 `/tmp/<tag>/mode-selection.md`,据此选定无人值守或有人在场参数组;没做模式选择就启动 = 流程违规;
3. **建面板**:新开独立标签页,按布局循环裂出 N 个面板并在 `panes.txt` 落盘;git 仓库下先为每个非 orch 槽位各建一个 worktree,再按对应目录起代理;
4. **起代理**:按"后端档案"逐槽位取参数组启动,并全部重命名;启动后验收真实权限模式与模型/推理档;
5. **bootstrap 闭环**:编排者下发本次批次号的 bootstrap 文件,等 `orch` 回执,ACK 与预期文本**全文全等**(`cmp -s`)才判定"拉起完成",随即释放排他锁;
6. **派单**:看板 `/tmp/<tag>/taskboard.md` 四列(待办 / 在途 / 待审 / 已验收);每单带 task ID 与 base commit,回执以落盘文件为准。

## 5. 槽位与模式

默认编队三成员(实际代理名带任务 tag,如 `orch-g07`):

| 槽位 | 能力 | 默认后端 | 默认模式(无人值守参数组) |
|---|---|---|---|
| `orch` | 编排 + 验收 | kimi(或名单内其它后端 claude;见"编排者档案") | `--auto`(claude 当 orch 时:`--dangerously-skip-permissions` + 显式 `--model`) |
| `rev` | 审阅 | codex(模型 slug 与推理档分开设) | `--approve-for-me` |
| `exec` | 执行 | claude(各人自定) | `--dangerously-skip-permissions` + `--settings` 挂载 `claude-exec-deny.json` |

上表"默认模式"是**无人值守参数组**的取值。是否用它由开工前三道闸决定,任一不过就改用**有人在场参数组**(显式人工批准,并在启动后验收真实模式,不接受继承本机默认):

| 闸 | 核验什么 |
|---|---|
| 1 推送边界 | 执行身份本身无 bypass 权限;目标 main/master 被 branch protection 实际覆盖;其他允许推送的分支也不能 force-push |
| 2 凭据面 | 运行环境不挂载非必需凭据(或用独立 Unix 用户 / 容器隔离),并留下挂载清单 |
| 3 目录与权限范围 | 是 git 仓库且能建 worktree;进程写入边界有受限范围(容器 / 独立用户),或用户明确选定授权范围 |

各槽位的启动参数不在这里写死,一律到 `SKILL.md` 的"**后端档案**"按该槽位的 CLI 取;roster 里每个成员的后端各自查表。

## 6. 已知限制

边界如实写清(细节见 `SKILL.md` 的"已知限制"节):

- **push 永禁不是硬拦**:`claude-exec-deny.json` 里的是**文本前缀规则**,属软防线,等价写法可绕过(如 `git push origin HEAD:main`、`+refspec` 强推、`git -C` 包装、无参数 `git push`);要真永禁,得靠远端 branch protection 或权限受限凭据;
- **读机密只覆盖 cwd 内直接读**:`~/.ssh/`、CLI 凭据目录,以及子进程自己打开文件这类间接读取都管不到;危险形态靠**枚举**进清单,而枚举天然列不全——所以无人值守环境不应挂载非必需凭据;
- **worktree 不是安全边界**:它只隔离工作文件与索引,git 元数据(config/refs)与主树共享;"主树只有编排者能动"是纪律约定,机制上不成立;
- **替补后端没有等价控制**:claude 缺席时 `--settings` 挂载失效,替补 CLI 没有同等机制,该槽位的无人值守能力随之停用;
- **`/tmp` 可能被系统清理**:看板、ACK、裁决都在 `/tmp/<tag>/`;关键 manifest、最终裁决与恢复指针要另存项目内持久位置;
- **心跳机制归属当前会话**:各宿主的心跳机制不同名但同理(如 claude 宿主是会话内计划任务),新建会话不继承,需重建——不能当成跨会话可用的共享设施;
- **无钩子成员没有状态语义**:herdr 不认识 genox,`herdr agent read` 直接报 `agent_not_found`,只能 `herdr pane read` 读屏幕推断 idle/working,弱于钩子;协议闭环不依赖它(锚在落盘 ACK),但派单时效降级。

## 7. 文件清单

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 技能本体(中文):拉起协议 + 运行协议全文 |
| `SKILL.en.md` | 技能本体的英文全译,结构与 `SKILL.md` 逐节对齐 |
| `claude-exec-deny.json` | 执行线永禁清单(sudo / 毁盘 / force-push / 推 main / 读机密),经 `--settings` 挂载;roster 里每个 claude 系实例各挂一份 |
| `README.md` | 本文件:中文说明 |
| `README.en.md` | 英文说明,与 `README.md` 内容对齐 |
| `LICENSE` | MIT 许可 |

## 8. License

MIT License,Copyright (c) 2026 sheng luan。全文见 `LICENSE`。
