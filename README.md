# trio-orchestrate

一键拉起并编排 kimi / codex / claude 三智能体协作的技能。

## 1. 简介

trio-orchestrate 是一个多智能体协作技能:把一项任务分给三个角色,各守其位——kimi 负责编排与验收,codex 负责独立审阅,claude 负责执行。三者由固定名字标识,不随任务变化。

| 角色 | 固定名 | 职责 |
|---|---|---|
| 编排 + 验收 | `kimi-orch` | 派活、复核关键数字、守看板 |
| 审阅 | `codex-review` | 独立审阅,裁决落盘并发指针 |
| 执行 | `claude-exec` | 只认领做法已写死、验收线机器可核的 [机核] 条目 |

技能依赖 herdr(终端多路复用)建面板、起代理、传话与盯状态:用户只需启动 kimi 并加载本技能,其余面板自动建好,布局固定为左 `kimi-orch` | 右上 `codex-review` | 右下 `claude-exec`。

设计上有两条主张:一是**验收靠落盘证据**(ACK 全文全等比较、结果文件、复跑复算),不靠状态观测与口头汇报;二是**放权有条件**——是否采用无人值守参数组,由开工前的三道闸决定,不是默认值。

## 2. 安装

把**整个 `trio-orchestrate` 目录**(含 `claude-exec-deny.json`)拷贝到 kimi 的技能目录:

```bash
cp -r trio-orchestrate ~/.kimi-code/skills/
```

`claude-exec-deny.json` 是执行线的永禁清单,必须与 `SKILL.md` 一同拷贝;缺了它,执行线就失去这层权限约束。

## 3. 前置依赖

- **kimi CLI**(必需):技能本体跑在 kimi 里,编排者就是它;
- **herdr**(必需):负责标签页/面板、代理注册与状态侦测;`herdr integration status` 显示状态钩子已装,idle/working 侦测才可靠,否则只能人工 `pane read` 盯;
- **codex / claude**(可选):审阅线与执行线的默认后端。缺任一个时走"替补组队"——按可用 CLI 枚举候选对,硬约束是**审阅与执行不得同 CLI 且不得同模型族**(同源盲点,审了等于没审);`kimi-orch` 不可替换;
- **各自登录态**:使用前各 CLI 需在本机已登录。首次使用前请人工跑通一遍:各 CLI 登录、仓库 trust 确认、bypass 模式的首次确认,都不能留到无人值守时段做。

## 4. 快速开始

```
/skill:trio-orchestrate <tag>
```

`<tag>` 是任务标签(只允许 `[a-z0-9_-]`,不超过 19 字符),同时用作标签页名称与任务目录 `/tmp/<tag>/`。拉起流程概述:

1. **模式选择(必做)**:按三道闸逐项核验并把证据写进 `/tmp/<tag>/mode-selection.md`,据此选定无人值守或有人在场参数组;没做模式选择就启动 = 流程违规;
2. **建面板**:新开独立标签页,按固定布局裂出三个面板;git 仓库下先为审阅线与执行线各建一个 worktree,再按对应目录起代理;
3. **起代理**:三件套按选定的参数组启动,并全部重命名;启动后验收真实权限模式与模型/推理档;
4. **bootstrap 闭环**:编排者下发本次批次号的 bootstrap 文件,等 `kimi-orch` 回执,ACK 与预期文本**全文全等**(`cmp -s`)才判定"拉起完成",随即释放排他锁;
5. **派单**:看板 `/tmp/<tag>/taskboard.md` 四列(待办 / 在途 / 待审 / 已验收);每单带 task ID 与 base commit,回执以落盘文件为准。

## 5. 角色与模式

三个固定角色(实际代理名带任务 tag,如 `kimi-orch-g07`):

| 角色 | 固定名 | 默认后端 | 默认模式 |
|---|---|---|---|
| 编排 + 验收 | `kimi-orch` | kimi | `--auto` |
| 审阅 | `codex-review` | codex(模型 slug 与推理档分开设) | `--approve-for-me` |
| 执行 | `claude-exec` | claude(各人自定) | `--dangerously-skip-permissions` + `--settings` 挂载 `claude-exec-deny.json` |

上表"默认模式"是**无人值守参数组**的取值。是否用它由开工前三道闸决定,任一不过就改用**有人在场参数组**(显式人工批准,并在启动后验收真实模式,不接受继承本机默认):

| 闸 | 核验什么 |
|---|---|
| 1 推送边界 | 执行身份本身无 bypass 权限;目标 main/master 被 branch protection 实际覆盖;其他允许推送的分支也不能 force-push |
| 2 凭据面 | 运行环境不挂载非必需凭据(或用独立 Unix 用户 / 容器隔离),并留下挂载清单 |
| 3 目录与权限范围 | 是 git 仓库且能建 worktree;进程写入边界有受限范围(容器 / 独立用户),或用户明确选定授权范围 |

## 6. 已知限制

边界如实写清(细节见 `SKILL.md` 的"已知限制"节):

- **push 永禁不是硬拦**:`claude-exec-deny.json` 里的是**文本前缀规则**,属软防线,等价写法可绕过(如 `git push origin HEAD:main`、`+refspec` 强推、`git -C` 包装、无参数 `git push`);要真永禁,得靠远端 branch protection 或权限受限凭据;
- **读机密只覆盖 cwd 内直接读**:`~/.ssh/`、CLI 凭据目录,以及子进程自己打开文件这类间接读取都管不到;危险形态靠**枚举**进清单,而枚举天然列不全——所以无人值守环境不应挂载非必需凭据;
- **worktree 不是安全边界**:它只隔离工作文件与索引,git 元数据(config/refs)与主树共享;"主树只有编排者能动"是纪律约定,机制上不成立;
- **替补后端没有等价控制**:claude 缺席时 `--settings` 挂载失效,替补 CLI 没有同等机制,该线的无人值守能力随之停用;
- **`/tmp` 可能被系统清理**:看板、ACK、裁决都在 `/tmp/<tag>/`;关键 manifest、最终裁决与恢复指针要另存项目内持久位置;
- **心跳 cron 归属当前会话**:新建会话不继承,需重建。

## 7. 文件清单

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 技能本体(中文):拉起协议 + 运行协议全文 |
| `SKILL.en.md` | 技能本体的英文全译,结构与 `SKILL.md` 逐行一致 |
| `claude-exec-deny.json` | 执行线永禁清单(sudo / 毁盘 / force-push / 推 main / 读机密),经 `--settings` 挂载 |
| `README.md` | 本文件:中文说明 |
| `README.en.md` | 英文说明,与 `README.md` 内容对齐 |
| `LICENSE` | MIT 许可 |

## 8. License

MIT License,Copyright (c) 2026 sheng luan。全文见 `LICENSE`。
