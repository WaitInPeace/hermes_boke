---
title: "Hermes Agent v0.21 Kanban Swarm 功能深度解析（面向 AI 开发者）"
date: 2026-10-08T00:00:00+08:00
author: WaitInPeace
tags: [hermes, kanban, multi-agent, swarm, ai-developers]
draft: false
---

> 来源标注说明：本文每条技术断言保留 [源: …] 行内标注（代码文件:行号 / 官方文档 / 实测）。术语首次出现附英文。

## 一、一个 agent 不够，一队 agent 怎么管？

场景：深夜，你让 agent 重构一个模块。它跑到一半，会话崩了，上下文压缩了，你重开窗口凭记忆复述进度——"上次改到一半"。或者你想并行：三个专家 agent 同时开工，你开三个终端、三个会话，交接靠复制粘贴，最后谁改了什么、为什么改，全在你日渐模糊的记忆里。

单次会话的 agent 有三个天生短板：会话易失（crash、压缩、超时都会丢状态）、并行困难（一个会话一条主线）、跨会话交接靠嘴（无审计、无持久记录）。Kanban Swarm 的回答是：把"一队 agent 的协作"建模成一块持久的工作队列——看板（board）。任务是有状态的行，agent 是认领（claim）行的工人（profile），调度是网关里的一段循环，协作的全部状态写在可审计的 SQLite 行里。人可以在任何时刻介入：评论、改状态、重派。

顺带一提：你正在读的这篇文章，正是由这样一块看板流水线（orchestrator → researcher → writer → reviewer → publisher）产出的——此刻 blog 板的 dispatcher 正派发着这条流水线的发布卡 [源: 实测]。

## 二、架构解剖：SQLite 看板 + 网关内嵌 dispatcher

先建立整体图景：没有独立服务，没有消息中间件。每个 board 一个 SQLite 数据库文件（default 板在 ~/.hermes/kanban.db，其余板在 boards/<slug>/kanban.db），workspace 与日志同目录分放——看板即目录，可以整体打包迁移 [源: docs/kanban.md:144-151]。

数据库共 7 张表：tasks、task_links、task_comments、task_events、task_runs、task_attachments、kanban_notify_subs [源: kanban_db.py:947-1142]。tasks 表是核心，除标题、正文、状态外还带一排工程字段：claim_lock/expires（认领锁）、worker_pid、worker_started_at 指纹（boot epoch 加启动时间，防止 PID 回收导致误杀新进程）、consecutive_failures（连续失败计数）、block_kind、goal_mode/max_turns、max_runtime_seconds、completion_contract（迁移添加列，见 kanban_db_connect.py:833）等 [源: kanban_db.py:947-1048; kanban_db_connect.py:833]。task_runs 表每次 claim 写一行，status 枚举 running|done|blocked|crashed|timed_out|failed|released——每一次尝试都有行可查 [源: kanban_db.py:947-1142]。

为什么是 SQLite 而不是消息队列？因为多个 profile 是多个跨进程的 agent，需要共享状态；SQLite 的 WAL（write-ahead log）模式加上加长的 busy timeout，天然支持多写者串行化——多个进程同时读写同一个看板文件而不互相踩踏 [源: kanban_db_connect.py:49]。一板一库还带来两个免费属性：隔离即隔离（删库即清场），备份即备份（拷文件即迁移）。

调度器（dispatcher）内嵌在网关进程里（dispatch_in_gateway 默认 true），不单起服务。它每 tick 默认 60 秒枚举所有板（下限 1 秒），空闲一 tick 仅约 300µs——成本低到不值得为它单开进程 [源: gateway/kanban_watchers_common.py:113; config_defaults.py:1899]。单例性靠排他非阻塞文件锁保证：同机两个网关不会变成两个调度器抢任务。

认领（claim）是并发安全的核心原子操作：在 write_txn 事务内做 CAS（compare-and-swap）——ready 态的卡翻成 running，同时校验父卡是否全部 done，不满足就降回 todo 并记录 claim_rejected；claim 的 TTL 为 900 秒，过期自动释放 [源: kanban_db.py:2341; kanban_parser.py:278]。实测本卡的 run 行里 claim_lock 形如 "LAPTOP-BGPEGOBP:50500"（host:pid）[源: 实测]。

组件图（文字版）：

```text
┌──────────────┐   tick(60s)    ┌──────────────────────────────┐
│ 网关 dispatcher ├──────────────▶│ SQLite kanban.db（每板一库）    │
└──────┬───────┘   claim(CAS)   │ tasks/links/comments/events/  │
       │ 派发，注入               │ runs/attachments/notify_subs │
       │ HERMES_KANBAN_TASK/BOARD└──────────────┬───────────────┘
       ▼                                        │ WAL 多写者串行化
┌──────────────────────────────┐                │
│ worker 进程（profile:writer）◀────────────────┘
│ 用 kanban_* 工具读写本卡与黑板 │
└──────────────────────────────┘
```

这套架构把"多 agent 协作"降解成了一个 SQLite 事务问题——认领的 CAS、蜂群的原子建图、父卡门控，全是可验证的数据库操作而非黑盒调度。这是它与框架内多 agent 方案最本质的区别。代价同样明确：派发是轮询不是事件推送，60 秒级的延迟是简单性的账单。

工人（worker）的视角同样刻意收窄：运行时给 worker 的是 15 个 kanban_* 工具——show/list/complete/block/schedule/request_review/request_changes/heartbeat/comment/create/link/attach/attach_url/attachments/unblock [源: tools/kanban_tools_schemas.py:46-549]。其中 kanban_list 与 kanban_unblock 只有 orchestrator 可用——普通 worker 看不到全板，信息获取只能靠 kanban_show 读自己的卡（含父卡交接与评论线程）[源: tools/kanban_tools.py:698,1264]。这是刻意的权力分离：worker 全权写自己的卡，但被围栏挡在"动别人的卡"之外；全板视角是编排者的特权，而非功能缺失。

## 三、生命周期与状态机：九种状态与四类中断

状态全集是九个：triage、todo、scheduled、ready、running、blocked、review、done、archived；建卡的 --initial-status 参数仅允许 running 与 blocked 两种取值 [源: kanban_db.py:106-107]。其中 running 意为"正常入流"：实际落库状态由父卡决定——无父卡或父卡全 done 落 ready，有未完成父卡落 todo；blocked 则直接停靠等人 [源: kanban_db.py:1327-1345]。流转规则如下：

- 依赖门控：todo → ready 由每 tick 的 recompute_ready 自动推进，条件是父卡全部 done；claim 时二次校验，不满足当场降级 [源: docs/kanban.md:132; kanban_db.py:2354]。
- block 是"有类型的中断"，共四类 kind：dependency（等父卡，回 todo，无需人介入）；needs_input（等人给答案）与 capability（硬墙：无凭据、无人能做的动作）落 blocked；transient（疑似偶发故障）。同一 kind 反复 block→unblock→block 达到 2 次，改道 triage——防止 cron 之类的外部驱动把自动化变成死循环；计数只在 complete 时清零 [源: kanban_db.py:109-114; kanban_parser.py:319; docs/kanban.md:421]。
- unblock 的落点取决于父卡：父卡齐→ready（若从 review 起源则回 review），未齐→todo [源: docs/kanban.md:414]。
- review 经由 kanban_request_review 进入，是正向流转，不占 block 预算；reviewer 用 request_changes 打回或 complete 放行 [源: kanban_parser.py:335]。
- scheduled 是"等时间"的停放态，供外部定时驱动在指定时刻唤醒 [源: kanban_parser.py:326]。

主干流转可概括为：triage（待细化）→ todo（排队）→ ready（可认领）→ running → review（待审）→ done；blocked 是从任意工作态的旁路，archived 是终态冷库。

可逆性（done 可重开、block 可 unblock、review 可打回）与防震荡护栏（block_recurrences）之间的张力，是这套状态机最见功力的地方——可逆性方便人机协作，护栏防止自动化把可逆性变成振荡。另外注意，triage 一处三用：既接收人工草稿，也承接 specify/decompose 的自动细化，还是防循环护栏的落点。

## 四、Swarm 拓扑：模板化的任务图

Swarm 不是第二个调度器，而是把"并行—验证—汇总"这个高频模式固化成一个任务图模板。源码第一行就把立场写死了："Deliberately no second scheduler"——一个写入既有 Kanban kernel 的小型任务图 [源: kanban_swarm.py:1-14]：

```text
root（立即完成，常驻黑板）
 ├─ worker₁（并行）
 ├─ worker₂（并行）
 ├─ ……
 └─ verifier（父 = 全部 worker）
      └─ synthesizer（父 = verifier）
```

建图是一次原子写：create_swarm 在单个事务里建完全图——root 以 blocked 出生，事务内 CAS blocked→done 激活它，提交后才 recompute_ready 放行 worker [源: kanban_swarm.py:70-162]。每个 worker 的 body 会自动追加一段 "Swarm protocol"：root id、黑板约定、目标——工人无需知道自己在蜂群里，协议随卡下发 [源: kanban_swarm.py:61-67]。

两个 gate 卡被设计成"固定技能的通用能力"：verifier 固定 skills=["requesting-code-review"]，只有以 metadata {"gate":"pass"} complete 才算放行，否则 block 并指明缺什么；synthesizer 固定 skills=["humanizer"] [源: kanban_swarm.py:226-254]。也就是说，"审查"与"润色"被当作可复用的通用能力，而非每次手写提示词。

黑板（blackboard）是蜂群的共享内存：以 [swarm:blackboard] 为前缀的结构化 JSON 评论挂在 root 卡上，{key, value} 形式，同 key 后写覆盖，_authors 字段记录胜出值的作者以便溯源 [源: kanban_swarm.py:26,261-291]。合并规则是逐条扫描 root 评论线程、按序覆盖，最后附 _authors——示意（结构取自源码，非实际回显）：

```text
评论1: [swarm:blackboard] {"key":"verdict","value":"方案A可行"}
评论2: [swarm:blackboard] {"key":"verdict","value":"方案B更优"}
合并:  {"verdict":"方案B更优","_authors":{"verdict":"worker-2"}}
```

幂等性也从黑板来：带同一 idempotency-key 再跑一次，不重建图，而是从黑板里的 topology 记录恢复全部 id [源: kanban_swarm.py:201-209]。

为什么黑板挂 root？root 立即完成、不会再被派发，但它的评论线程永续保留——天然成为蜂群的共享内存与审计锚点；dashboard、notifier、slash command、dispatcher 全都无需为新服务适配 [源: kanban_swarm.py:1-14]。

CLI 建蜂群的真实语法（以代码为准）：

```bash
hermes kanban swarm "目标描述" \
  --worker researcher:调研卡 \
  --worker analyst:分析卡 \
  --verifier reviewer \
  --synthesizer writer
```

--worker 格式为 PROFILE:TITLE[:SKILL,SKILL]，可重复；--verifier 与 --synthesizer 必填 [源: kanban_parser.py:222-233]。注意：官方文档示例写作 --workers 逗号列表，与代码不符——以代码为准 [源: docs/kanban.md:1064]。

流水线与蜂群的区别值得写清楚：流水线是线性 parents 链（本文自身就是四卡流水线），蜂群是"root 黑板 + 并行 worker + 双层 gate"，适合同质并行研究。

## 五、隔离与工作区：三层边界

隔离分三层，适用口诀："一个业务用 tenant，多个项目用 board。"

- board = 硬边界：不同板是物理不同的 DB、workspace、日志目录；dispatcher 派发时注入 HERMES_KANBAN_BOARD/DB 环境变量，使 worker 物理上不可见他板；跨板 link 被禁止 [源: docs/kanban.md:149; docs/reference/environment-variables.md:130]。板的解析顺序：--board 参数 → 环境变量 → current 文件 → default [源: kanban_parser.py:467]。
- tenant = 软命名空间：字符串过滤，隔离靠 workspace 路径与 memory key 前缀 [源: docs/kanban.md:139]。
- workspace = 生命周期隔离，三态：scratch（默认，完成即删，仅声明过的 artifacts 幸存）、dir:<绝对路径>（保留；相对路径在派发时被拒）、worktree（.worktrees/<id>，--branch 命名分支）[源: docs/kanban.md:134-137; kanban_parser.py:169-172; kanban_db_workspace.py:779-785]。worktree 分支名是确定性的：普通任务 wt/<task-id>，挂到项目的任务 project-slug/<task-id>——两个 worker 永远不可能撞到同一分支 [源: 实测派发环境]。

scratch"完成即删"是反直觉但正确的默认——它强制交付物显式化（artifacts 声明机制：scratch 清理前把声明的文件复制进持久附件存储，声明缺失会阻止完成），杜绝工作区残留变成隐式状态 [源: docs/kanban.md:135]。一个耐人寻味的细节：相对路径 dir: 在派发时被拒，源码注释明说这是 confused-deputy（混乱代理人）逃逸向量——CLI 与 DB 两层都在防这件事。

## 六、可靠性工程：熔断、协议、心跳

可靠性不是重试堆叠，而是三件套：每个失败有退出码语义、每个回收有事件、每次尝试都落在 run 行里。

- 心跳桥：worker 每分钟镜像一次进程活性；预计超过 1 小时的任务必须每小时至少一次 heartbeat，否则 dispatcher 会回收 [源: docs/kanban.md:567]。
- stale 回收：running 超过 dispatch_stale_timeout_seconds（默认 14400 = 4 小时）且 3600 秒无心跳 → 回 ready，outcome='stale'，先按 pid+指纹杀掉可能还活着的 worker，且不计入失败次数 [源: kanban_db_dispatch.py:743; config_defaults.py:1952]。
- 熔断：连续 spawn_failed/timed_out/crashed 达到 failure_limit（默认 2）自动 block，等人工处置；--max-retries 可按卡覆盖 [源: config_defaults.py:1909]。
- 协议违约：进程退出码 0 但卡仍是 running → 记为 protocol_violation，连续默认 3 次 block；退出码 75（EX_TEMPFAIL，限流）不计失败；78（EX_CONFIG）首犯即 sticky block [源: docs/kanban.md:572-627]。
- goal_mode：同一 session 内由 judge 对照卡的 title/body 验收，judge 认为没完成就继续跑；goal_max_turns（默认 20）耗尽则 block 供人审——一张卡就是一个微型 /goal 循环 [源: kanban_db.py:1018; docs/kanban.md:716]。
- max_runtime_seconds：超时先 SIGTERM 再 SIGKILL，然后重排队 [源: kanban_parser.py:184]。
- completion_contract：local-only（默认）/ OWNER-REPO（PR 发布）/ 精确 PR URL（须 CI 通过）三档 [源: kanban_parser.py:206]。

心跳桥说明"活性"被显式建模——系统从不假设进程还活着，一切以行的状态为准，这是 crash 后能干净回收的根本原因。顺带提一个易混点：reclaimed 与 stale 是两种回收——前者是 claim 被释放/收回，后者是运行超时回收，排障时别混用。

## 七、CLI 与配置速查

CLI 动词超过 50 个，覆盖 init/boards/create/swarm/assign/claim/link/unlink/list/show/comment/complete/block/unblock/archive/tail/watch/stats/runs/dispatch/specify/decompose/repair/request-review/request-changes 等 [源: kanban_parser.py:97-454]。设计上有个关键决策：/kanban slash 与 CLI 共用同一棵 argparse 树——每加一个动词，CLI、聊天会话与全部 gateway 平台同时获得该能力 [源: docs/kanban.md:1072]。另注意 daemon 子命令已 DEPRECATED：dispatcher 已并入网关，daemon 与网关并存会造成两个调度器对同一 DB 抢 claim，必然双跑 [源: kanban_parser.py:371; docs/kanban.md:370]。

常用动词速查（语法均经源码核对）：

```bash
hermes kanban init                                    # 建库（幂等）
hermes kanban create TITLE --assignee PROFILE \
  --parent ID --skill NAME --priority N \
  --idempotency-key KEY --json                        # 建卡
hermes kanban swarm GOAL --worker P:T[:S] \
  --verifier P --synthesizer P                        # 建蜂群
hermes kanban link PARENT CHILD                       # 加依赖
hermes kanban unlink PARENT CHILD                     # 去依赖
hermes kanban assign TASK PROFILE                     # 改派
hermes kanban request-review TASK --summary "..."     # 交审
hermes kanban request-changes TASK "改哪里"            # 打回
hermes kanban tail TASK --interval 1.0                # 跟事件流
hermes kanban dispatch --dry-run --max N              # 手动试派发
```

配置键（kanban.* 段，默认即最佳实践）：dispatch_in_gateway:true、notify_in_gateway:true、review_dispatch:true、dispatch_interval_seconds:60、failure_limit:2、dispatch_stale_timeout_seconds:14400、max_in_progress（内存推导 clamp 到 [2,8]，小机器自动收敛并发；Windows/macOS 读不到内存则无上限）、dispatch_profiles（fail-closed 白名单，None=不限）、auto_decompose:true、done_sub_retention_days:30 [源: config_defaults.py:1891-1960]。实测本机 config.yaml 只有一条 review_dispatch:true，其余全靠出货默认 [源: 实测]。环境变量面：HERMES_KANBAN_HOME/BOARD/DB/WORKSPACES_ROOT/DISPATCH_IN_GATEWAY；HERMES_KANBAN_TASK 由派发时注入，不要手设 [源: docs/reference/environment-variables.md:128-133,832]。

对 AI 开发者最实用的两个 flag 是 --json 与 --idempotency-key——机器可读输出加去重创建，构成 webhook 或定时任务接入看板的最小闭环。配套的附件面同样为机器交互设计：kanban_attach 走 base64 内联，kanban_attach_url 走服务端下载，上限 25MB，blob 落盘在 attachments_root/<task_id>/ 下——跨容器场景里 worker 不便内联大文件时，传一个 URL 即可 [源: tools/kanban_tools_schemas.py:343; kanban_db.py:1106]。

## 八、动手教程：三条命令跑起一条流水线

目标：建一条"研究 → 写作 → 审核"流水线，全自动派发，审核通过收尾。以下命令语法均经源码核对（见第七节来源）；输出以文字描述——本 worker 会话被围栏禁止执行写动词（原因见下），故不附伪造的终端回显。

```bash
hermes kanban init                                     # 建库（幂等，重复无害）
hermes kanban boards create blog --name "博客流水线"    # 可选：独立板
hermes kanban create "调研 v0.21 Kanban 事实底座" \
  --assignee researcher --skill arxiv --json           # 记下返回的 id，称 R
hermes kanban create "撰写深度解析初稿" \
  --assignee writer --parent R --json                  # 记下 id，称 W
hermes kanban create "审核初稿" \
  --assignee reviewer --parent W --json                # 记下 id，称 V
```

三张卡创建后，dispatcher 在一个 tick 内看到它们：R 无父卡直接 ready 被派发；W、V 因父卡未完成保持 todo，父卡完成时由 recompute_ready 自动晋升 [源: docs/kanban.md:132]。worker 进程开场会 kanban_show 读自己的卡——包括父卡交接（summary+metadata）与评论线程——然后干活 [源: docs/kanban.md:565]。期间你可以 hermes kanban tail W 实时看事件流，或随时 hermes kanban comment W "……" 插话。

writer 完成后经工具层 kanban_request_review 交审，卡进 review；reviewer 两种结局：request-changes 打回重写（不占 block 预算，可反复循环），或 complete 放行 [源: kanban_parser.py:335]。最后一卡 done，流水线收尾；done 的卡默认保留 30 天（done_sub_retention_days），审计随时可查 [源: config_defaults.py:1891-1960]。

蜂群版一行替换：

```bash
hermes kanban swarm "论证 X 方案并产出报告" \
  --worker researcher:方案调研 \
  --worker analyst:数据分析 \
  --verifier reviewer --synthesizer writer --json
```

注意三点：一是这些是"人/编排者"的动词——worker 进程里执行会被围栏拦截，实测返回 "delegate_task child contexts cannot mutate Kanban tasks or boards"；卡与卡之间的协作靠 kanban_* 工具（show/comment/complete/block），而不是靠卡自己跑 CLI [源: 实测; docs/kanban.md:570]。二是派发是轮询，创建后最多等一个 tick（60 秒）才见 worker 启动；急的话 hermes kanban dispatch 手动触发。三是需要"人先动手才能开工"的卡（比如 R3 审批门），建卡时用 --initial-status blocked，直接停在等人状态，跳过 running→blocked 的短暂流转 [源: kanban_parser.py:216-219]。

## 九、对比与取舍：什么时候用看板，什么时候不用

对比的核心不是"谁更强"，而是"状态存在哪里"：进程内存还是 SQLite 行。

vs delegate_task：函数调用 vs 持久队列；阻塞等待 vs 创建即忘；匿名子代理 vs 具名 profile；不可恢复 vs block/unblock/重试；无人工介入点 vs 任意评论；审计随上下文压缩丢失 vs SQLite 永久留存；层级嵌套 vs 同侪流水线。官方场景建议：要答案用 delegate_task，要协作/持久/审计用 Kanban [源: docs/kanban.md:97-122]。

vs cron：cron 是定时触发器，与队列正交；scheduled 状态专门管"等到某时刻"的卡，两者可以叠加 [源: kanban_parser.py:326]。

vs Claude Code subagents / AutoGen / crewAI：进程内/框架内的多 agent 协作，缺少跨进程持久队列、人工可介入的状态机与 SQLite 审计。此条为结构性通用知识，具体版本行为未经本树验证 [未验证]。

取舍框架用三个问题即可：状态存在哪里？人能不能中途介入？审计会不会丢？答完这三个问题，选型自然清晰。看板的代价也要明说：60 秒轮询延迟不适合实时对话式协作；单机 SQLite 意味着没有跨机器分布式调度（除非自己搬库）；50+ 动词的学习曲线不低。最后一条内部取舍：流水线（pipeline）与蜂群（swarm）都是 parents 图，差别在形态——串行依赖选流水线，同质并行加汇总选蜂群；后者多出的成本是 root 黑板卡与 verifier 双层 gate 的等待时间。

## 十、结语：本文的元叙事

回到开头的问题：一队 agent 怎么管？Hermes 的回答是——用一块看板当持久工作队列，用 SQLite 行当共享内存，用状态机当协作语言，用熔断与心跳当安全网。

而本文本身就是证据：它由 blog 板上一条流水线产出——orchestrator 拆解大纲，researcher 产出 55 条带逐条来源的断言简报，writer（本卡，正是此刻）据此成文，reviewer 将打回或放行，publisher 最终发表。派发、认领、心跳、交接，每一步都是 tasks/task_events 表里可查的行 [源: 实测]。

如果你正卡在"一个 agent 不够用"的阶段，不妨先跑 `hermes kanban init` 建一块板——第一张卡，就是你现在脑中的那个任务。
