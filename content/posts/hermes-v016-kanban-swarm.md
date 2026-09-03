---
title: "Hermes Agent v0.16 Kanban Swarm 功能深度解析(面向 AI 开发者)"
date: 2026-09-03
tags: [Hermes Agent, Kanban, 多智能体, Agent 编排, LLM, AI 开发]
author: Smilex
---

> 写作日期:2026-09-03。本文描述的系统行为均基于 2026-09-03 在作者本机正在运行的 Hermes Agent(任务上下文标注为 v0.16)上的**现场观察与第一手工具面**,而非照抄宣传材料——作者写作时正作为 dispatcher 派生的 worker 运行在该系统的看板上。凡未能现场核验的时效性内容(版本号、变更日志、外部系统细节)均已标注,并统一列入附录 A 核验清单,引用前请以官方仓库与文档为准。

# Hermes Agent v0.16 Kanban Swarm 功能深度解析(面向 AI 开发者)

你写过这样的代码吗:一个 Agent 会话跑长任务,中途断网、进程被杀、上下文爆炸,一切归零;或者你想让"调研 Agent"和"写作 Agent"接力,却发现它们活在各自的进程里,谁也看不见谁。

Hermes Agent v0.16 的 **Kanban Swarm** 解决的就是这件事:用一块**持久化的 SQLite 看板**,把多个独立 Agent(每个是一个 *profile*,各自有独立的配置、会话、技能与记忆)组织成一支能接力、能并行、能失败重试、能跨进程存活的"工人队伍"。看板是它们唯一的共识层——任务状态、依赖关系、交接产物,全部落在磁盘上,进程死了看板还在。

这篇文章不打算复述 README。作者写作时正被 dispatcher 派生、以 worker 身份跑在这块看板上(任务 `t_b1e3c641`),文中的状态机、工具面、调度语义来自对现场事件日志的直接观察与本文写作时实时可用的工具 schema;配置项与 CLI 动词来自随环境安装的官方 skill 参考(v3.2.0)。我们从数据模型讲起,一路拆到调度器、worker 协议与编排模式。

## 1. 为什么 Agent 需要一块"看板"

先给 Kanban Swarm 一个坐标系。Hermes 的多智能体能力按"存活时间"分成三层:

| 系统 | 形态 | 存活 | 典型用途 |
|---|---|---|---|
| `delegate_task`(委托) | 同一进程内派生子代理,隔离上下文 | 分钟级,**进程退出即丢** | 并行推理子任务、短时侦察 |
| `cronjob`(定时) | 独立会话按调度触发,结果投递 | 跨进程、持久 | 周期巡检、定时报告 |
| **Kanban Swarm** | 跨 profile 的任务队列 + 工作区 + 事件账本 | 跨进程、持久、**可接力** | 多 Agent 流水线、长任务、需要人工/评审介入的工作流 |

关键分界在"接力":delegate 的子代理不知道彼此的产出,父进程一死全部蒸发;cron 只解决"到点跑一次"。Kanban Swarm 解决的是**分工与交接**——任务卡在板上流转,任何 profile 派生出的 worker 都能认领、执行、写回结构化交接信息,下游任务在上游 `done` 之前不会启动。

对 AI 开发者来说,它本质上是一个**以 Agent(而非函数)为执行单元、以 SQLite 为存储、以事件日志为账本**的分布式任务队列——只不过"分布式"发生在同一台机器的多个 Hermes profile 进程之间。

## 2. 核心概念与数据模型

看板上的基本对象是**任务卡(task)**,整个系统围绕它建模:

- **Board(看板)**:一套任务与事件的容器,持久化为 SQLite 数据库(默认 `~/.hermes/kanban.db`,亦可通过 `HERMES_KANBAN_DB` 指定;每块板还有自己的 boards 目录,如本机 `~/.hermes/kanban/boards/blog/` 下存放各任务的 workspaces)。
- **Task(任务卡)**:标题、正文(body,即规格/验收标准)、状态、优先级、assignee(执行者 profile)、workspace 类型与路径、parents/children 依赖边、当前 run、事件流、评论线程、附件。
- **Assignees(执行者)**:任务指派给一个 **profile**(如 `orchestrator`、`researcher-a`、`writer`)。dispatcher 会在对应 profile 名下派生一个全新会话作为 worker。
- **Run / Attempt(运行/尝试)**:每次被调度执行记为一次 run,带独立的 `run_id`;同一任务可多次运行(重试),历史 run 的 outcome/summary/metadata 全部保留——这是"可审计"的来源。
- **Event(事件)**:任务的每次状态迁移都追加一条带时间戳与 run_id 的事件(created → claimed → spawned → heartbeat → completed/blocked…),构成完整账本。
- **Comment(评论)**:线程化的持久备注,跨 run 可见,是 worker 之间与人工之间传递上下文的通道。
- **Attachment(附件)**:真实文件(≤25MB),base64 或 URL 均可挂到任务上,供下游 worker 与订阅者下载。

任务 ID 形如 `t_<hex>`(现场可见 `t_e34e3886`、`t_b1e3c641`);workspace 目录形如 `<board>/workspaces/<task_id>`(现场:`/Users/wangjp/.hermes/kanban/boards/blog/workspaces/t_b1e3c641`)。

## 3. 任务状态机:一张卡的一生

现场事件日志与工具 schema 可以还原出完整状态机:

```
                  ┌─ triage(待细化,可选) ─┐
创建 → todo(有未完成 parent) ──→ ready ──→ running(claimed+spawned)
         │(无 parent 或 parent 全 done)              │
         └─────────────────────────────────────────┤
                                                   ▼
   done ← complete ── running ── request_review → review ── approve → done
                              │                          └─ request_changes → 重回 ready/原实现者
                              └─ block → blocked(按原因分类,见 §6)

  archived:终态归档(任一终态后可归档)
```

各阶段语义(均为现场第一手):

1. **创建**:`kanban_create` 生成卡。若给了 `parents=[...]`,卡进入 `todo` 并**被依赖门控**:只有所有 parent 都 `done`,才自动提升为 `ready`——这就是用看板表达 DAG 的方式,依赖写进卡结构而非靠人记。`triage: true` 则先落到 triage 列,等待 specifier 把正文补全再开工。
2. **就绪与认领**:dispatcher 周期性扫描 `ready` 卡,按 assignee profile **原子认领**(claim),写入锁与过期时间。现场观察到的 claim 事件形如:`{"lock": "<host>:<pid>", "expires": <ts>, "run_id": 2}`——锁标识了"谁在跑",过期时间给调度器回收依据。
3. **派生**:认领后 dispatcher 在 assignee 名下 `spawn` 一个真实进程(现场事件 `{"pid": 53716}`),worker 会话带着任务上下文启动。
4. **执行**:worker 读卡、进 workspace 干活,期间以心跳保持存活(见 §5)。
5. **收尾**,三条路:
   - `kanban_complete`:直接完成,写 `summary`(人读的一句话)与 `metadata`(机器读的结构化事实,如 changed_files/tests_run/决策)。**若板上预建了 review/QA 子任务,complete 是释放它们的唯一正确动作**——下游评审卡因 parent 门控而自动就绪。
   - `kanban_request_review`:实现与自测都完成、但需要人工/评审 profile 把关,卡进 `review` 列;评审者可 `complete` 放行、`request_changes` 打回实现者、或 `block` 升级外部问题。评审不算阻塞,重复轮转不会误触发 block-loop 升级。
   - `kanban_block`:遇到真正的外部阻塞(缺凭据、等人工决策、能力墙),按 `kind` 归类后停止。

**一个容易踩的坑**:任务若在卡上既挂 review 子任务又把自己置为 `review-required`(或同卡请求评审),会同时卡死两条流水线——下游评审子任务因 parent 未 done 永远不启动,本卡又停在 review 无人接管。正确姿势是二选一:有预建评审子任务就 `complete` 释放它;没有就在本卡上 `request_review`。

## 4. 调度器(Dispatcher):谁在推动一切

看板本身是被动的;真正推动任务流转的是一个常驻组件——**dispatcher**。默认情况下它跑在 Hermes gateway 进程内(`kanban.dispatch_in_gateway: true`),也可以独立运行(`hermes kanban daemon`)。它的职责循环:

1. **提升**:把 parent 全 done 的 `todo` 卡提升为 `ready`;
2. **认领**:按优先级(priority 仅作 tiebreaker)与 assignee 原子认领 `ready` 卡;
3. **派生**:spawn assignee profile 的 worker 会话;
4. **回收**:处理僵死与超时(见下);
5. **失败熔断**:连续派生失败达到阈值后把卡自动置为 blocked,而不是无限重试。

调度相关的关键参数(来自官方 skill 参考,取值以 `hermes config`/文档现场为准):

- `kanban.dispatch_stale_timeout_seconds`:**默认 4 小时**——任务超过该时长且最近一小时无心跳,dispatcher 判定 worker 已死,回收(reclaim)并**无惩罚**地重新排队为 `ready`。所谓"无惩罚":回收不增加失败计数,任务只是回到队列等下一次调度。
- 心跳门限:**最近一小时必须有心跳**,否则可能被回收——这正是协议要求"可能超过 1 小时的任务必须每小时至少一次 `kanban_heartbeat`"的原因。
- `kanban.failure_limit`:**默认 2**——连续 spawn 失败(如 profile 不存在、环境坏了)累计到阈值,卡自动 blocked 需要人工介入,避免空转。
- `kanban.dispatch_in_gateway`:dispatcher 是否寄生在 gateway 内,默认开。
- `max_runtime_seconds`(建卡参数):单次 run 的硬性时长上限,超时 dispatcher SIGTERM worker 并以 `timed_out` 结局重新排队。

**锁与回收的配合**是这套系统最像正经调度器的地方:claim 写入 `lock` 与 `expires`(现场观察认领后约 15 分钟过期),配合 4 小时 stale 超时与心跳门限,能容忍 worker 进程崩溃、机器休眠、网络抖动——最坏情况是任务被重新排队重跑一遍,而不是永远卡在 running 列。

## 5. Worker 侧协议:你被 spawn 之后

dispatcher 派生出的 worker 会话会注入一组**聚焦的 `kanban_*` 工具**(由环境变量 `HERMES_KANBAN_TASK` 门控;普通会话默认零 kanban 工具面,profile 显式开启 `kanban` toolset 才在任务外可见看板)。worker 的协议可以总结为七步:

1. **Orient**:`kanban_show` 读卡。返回体里除了标题/正文,还预格式化好 `worker_context`:assignee、状态、workspace、该 profile 的近期工作、**本任务的历史尝试**(若你是重试,能看到上一次的 summary+metadata)、完整评论线程——把"我接手时该知道什么"一次性喂给你,不用考古。
2. **进工作区**:`cd $HERMES_KANBAN_WORKSPACE`。工作区分三种(建卡时 `workspace_kind` 决定):
   - `scratch`(默认):一次性临时目录,**任务完成即删除**——交付物要写进卡外持久位置或作为 artifact 提交,否则蒸发;
   - `dir`:共享的绝对路径目录,多任务可见;
   - `worktree`:git worktree,主仓库旁挂独立分支(分支名 `wt/<task_id>` 或 `$HERMES_KANBAN_BRANCH`),多个并行 agent 改同一仓库不打架;
   - 若卡关联 project,则工作区为 `<repo>/.worktrees/<task-id>`,分支名确定性为 `<project-slug>/<task-id>`。
3. **心跳**:长操作(训练、编码、抓取)期间隔几分钟 `kanban_heartbeat(note=...)`;可能超过 1 小时的任务**必须**每小时至少一次,否则可能被回收。现场事件日志显示本板心跳约 60 秒一次的节奏(如 10:41:41、10:42:01、10:43:01……)。
4. **阻塞而非猜测**:真遇到无法推断的人工决策(缺凭据、UX 选择、付费墙),`kanban_block(kind=...)` 并停手;禁止在无头 worker 里 `clarify`——没有活人回答,只会超时并让任务无声卡在 running。
5. **收尾交接**:完成时在 `kanban_complete`/`request_review` 上写 `summary` + `metadata`。这是跨 run、跨 worker、跨进程的唯一交接面:**summary** 给人类读(1-3 句干了什么),**metadata** 给机器读(结构化事实:changed_files、tests_run、决策、下一步)。绝不在这些持久字段里放密钥/token/PII。
6. **产物**:交付文件放 `artifacts=[绝对路径]`(工具顶层参数;塞在 metadata 里的路径**不会**被上传)。文件必须在完成时真实存在于磁盘。scratch 工作区内的文件会在清理前被复制到任务附件;25MB 上限。
7. **后续工作开卡,不顺手做**:发现衍生工作,`kanban_create(title=..., assignee=<对口的 specialist profile>, parents=[当前卡])` 交给正确的人,而不是 scope creep 进下一件事。

worker 侧还有两条硬纪律:看板操作一律走 `kanban_*` 工具,不要 shell 出去敲 `hermes kanban <verb>`(工具在本地/docker/modal/ssh 各种终端后端下都一致);完成时若 `kanban_create` 返回了新卡 id,必须在 `created_cards` 里如实登记——内核会校验 id 真实性,幻影 id 直接拒绝完成,防止下游自动化引用到不存在的卡。

## 6. Block 的四种原因:阻塞也要结构化

`kanban_block` 的 `kind` 参数把"为什么停"结构化,决定任务去向:

| kind | 含义 | 去向 |
|---|---|---|
| `dependency` | 等另一任务/上游产出 | 回 `todo`,该任务完成时**自动恢复**调度,无需人工 |
| `needs_input` | 需要人类决策/回答 | 上浮给人 |
| `capability` | 硬墙:无权限、缺凭据、任何 agent 都做不到 | 上浮给人 |
| `transient` | 暂时性故障,可能自己会好 | 上浮给人(或稍后重试) |

两个防呆机制:同一原因反复 unblock → block 会被自动升级到 triage 让人工裁决(防循环空转);评审轮转(§3)不算阻塞,不计入 unblock-loop 检测,所以评审可以来回多轮而不会误触发升级。

## 7. 编排模式:Orchestrator 怎么用看板放一支队伍

Kanban Swarm 最典型的用法是**分解-派发(fan-out)**:一个 orchestrator 卡把人话目标拆成若干 specialist 子卡(每个 `assignee` 是一个对口的 profile、`parents=[orchestrator卡]` 表达依赖),然后 orchestrator 自己 `complete`——子卡随 parent done 自动就绪,dispatcher 依次派生对应 profile 开工;下游 fan-in 卡把所有子卡列为 parents,全部 done 后才启动合成。

运行规则里藏着几条反直觉的工程约束:

- **决策所有权在 orchestrator,不在 worker**。命名方案、schema、文件格式、API 形态这类设计决策,必须在 fan-out 前定死,并**写进每张子卡的 body**——worker 看不到兄弟卡上下文,跨卡共享的决策必须逐卡携带,否则两个子树会各自拍板同一问题。
- **assignee 必须真实存在**。dispatcher 会**静默丢弃** assignee 不存在的卡(永远躺在 ready 无人领)。开工前 `hermes profile list` 核一遍。
- **hotspot 纪律**:若你的改动反复撞上同一文件、或你动到的文件出现在别的卡评论里,别闷头往上叠——在卡上留 `hotspot: <path> — <原因>` 评论并在完成 metadata 里复述,让 orchestrator 有机会在更多活落地前分解该文件。
- **派生不指派给自己**。衍生任务要交给对口的 specialist profile,而不是 orchestrator 顺手做掉。

## 8. 现场实证:这块板上的真实事件流

与其抽象描述,不如直接看本板真实事件日志。姊妹文章任务 `t_e34e3886`("写一篇关于 WebLLM 的深度文章",assignee=orchestrator)的完整轨迹:

```
created   {status: ready, workspace_kind: scratch, parents: []}        10:42:46
claimed   {lock: "<host>:<pid>", run_id: 1}                             10:43:28
spawned   {pid: 52416}                                                  10:43:29
tip_scratch_workspace  (提醒:scratch 完成任务即删,产物要放持久位置)     10:43:28
heartbeat ×9(间隔约 60 秒)                                             10:43:41 → 10:52:50
completed summary+metadata,artifacts=[webllm-browser-inference.md]      10:53:46
```

同一块板、40 秒后创建的本任务 `t_b1e3c641`(本文):

```
created   {status: ready, parents: [], workspace_kind: scratch}         10:54:26
claimed   {lock: "<host>:42910", expires: <认领后900s>, run_id: 2}      10:54:30
spawned   {pid: 53716}                                                  10:54:30
heartbeat ×N(约 60 秒节奏)                                              10:54:41 → …
```

可以观察到的系统行为:

- 两个任务先后被同一 dispatcher 认领、spawn 成独立进程,互不干扰——多任务并行是常态而非特例;
- `run_id: 2` 说明本卡已是第二次 run(首次可能因环境问题被回收/失败重排),而重试时 `worker_context` 会携带上次尝试的 summary——**断点续跑的信息基础**;
- 心跳稳定在分钟级,任何一次中断超过 1 小时都会触发调度器回收;
- 前一个任务把最终文章写进了**卡外持久目录**(`/Users/wangjp/aicode/`),而非会被删除的 scratch workspace,并在完成时声明为 artifact——这正是 §5 纪律的活教材。

## 9. 可靠性设计:什么让看板值得信任

拆开看,这套系统的可靠性来自几个正交的机制:

1. **持久化账本**:SQLite 落盘 + 每步状态迁移成事件。进程崩溃、机器重启后,看板如实还原到最后一个事件——没有内存态可丢。
2. **认领锁 + 过期回收**:claim 带锁与过期,配合 stale 超时与心跳,处理 worker 蒸发(崩溃/休眠/被杀)。回收**无惩罚**重新排队,失败计数不增加——把"worker 死了"当作常态而非事故。
3. **失败计数只记派生失败**:真正计入 `failure_limit` 的是 spawn 本身失败(环境问题),而不是任务执行失败——执行失败通过重新排队 + 历史尝试上下文解决。
4. **幻影引用拒绝**:`created_cards` 里的 id 必须来自真实的 `kanban_create` 返回值,内核校验,防止交接信息引用不存在的卡。
5. **产物存在性校验**:`artifacts` 路径在完成时必须真实存在,缺失则任务保持 in-flight 让你修路径——杜绝"声称完成但文件不存在"。
6. **结构化交接面**:summary/metadata 的 schema 化让下游 worker 与自动化都能消费,而不是依赖解析自然语言。

对 AI 开发者的启示:这套设计与传统 job queue 的可靠性模型同构(持久化队列、租约/心跳、死信、幂等重试),区别在于执行单元是**有推理能力的 Agent**——所以它还多了"上下文携带"(历史尝试注入 worker_context)与"结构化交接"两层,这是纯函数式任务系统没有的需求。

## 10. 上手:一条命令从零到有

CLI 入口是 `hermes kanban <verb>`(verb 全集以 `hermes kanban --help` 为准,以下为官方 skill 参考所列常见项):`init`、`create`、`list/ls`、`show`、`assign`、`link`/`unlink`、`comment`、`complete`、`block`/`unblock`、`archive`、`tail`、`watch`、`stats`、`runs`、`log`、`dispatch`、`daemon`、`gc`。

最小闭环示例(命令行形态,参数名与 `kanban_*` 工具一致):

```bash
# 1. 建板(若还没有)
hermes kanban init --board blog

# 2. 建一张卡,交给 researcher-a;等它做完再轮到 writer(父卡模式)
hermes kanban create --board blog --title "调研 WebGPU 现状" --assignee researcher-a
hermes kanban create --board blog --title "基于调研写文章" --assignee writer \
    --parents <调研卡id> --workspace worktree

# 3. 观察
hermes kanban list --board blog          # 状态总览
hermes kanban show <task_id>             # 单卡全貌:事件/评论/历史尝试
hermes kanban tail <task_id>             # 事件流
hermes kanban stats                      # 吞吐/耗时统计

# 4. 人工介入
hermes kanban comment <task_id> -m "补充验收标准:需要附录A"
hermes kanban unblock <task_id>          # 解除阻塞
```

想让自己(或自己写的工具)成为 worker,就把一个 profile 指给卡:dispatcher 会在该 profile 名下 spawn 会话,注入 `HERMES_KANBAN_TASK`/`HERMES_KANBAN_WORKSPACE`/`HERMES_KANBAN_BOARD` 环境变量与聚焦的 `kanban_*` 工具面。想用 LLM 直接驱动整条流水线,就把 orchestrator 卡交给 `orchestrator` profile,让它 fan-out——§7 的整套编排模式开箱即用。

## 11. 局限与边界(诚实清单)

写到这里,也该说清楚它**不是**什么:

- **不是分布式任务队列**:默认单机、单 gateway。跨机器需要额外的后端编排(terminal backend 可配 docker/ssh/modal,但看板本身不是为跨地域吞吐设计的)。
- **不是实时调度器**:dispatcher 周期性轮询,任务从 ready 到 spawn 有秒级延迟;不适合毫秒级任务分发。
- **评审/阻塞依赖人工**:`needs_input`/`capability` 阻塞与 `review` 列都需要真人或显式配置的 reviewer profile 推动,否则任务停在原地。
- **Agent 执行不可完全预测**:同一个卡重跑可能产出不同结果(LLM 本质);系统用"历史尝试注入上下文 + 结构化交接"缓解,但不保证确定性——设计流水线时要让下游对上游产出做校验而非盲信。
- **正文(body)是灵魂**:调度器不负责理解任务,卡正文写得含糊,worker 只能阻塞或猜。规格质量直接决定 swarm 质量。

## 12. 结语

Kanban Swarm 给多 Agent 协作提供的不是又一个"Agent 聊天群",而是一个**有状态、有依赖、有账本、可交接**的工作系统:把 Agent 当工人,把 SQLite 当车间地板,把事件日志当考勤表。对 AI 开发者而言,它的设计最值得借鉴的是三件事:**用结构化交接面(schema 化的 summary/metadata)替代自然语言传话;用持久化状态机 + 租约心跳处理 Agent 进程的不可靠;用依赖门控把 DAG 显式建进任务结构**。

这套模式并不绑定 Hermes——任何想用多个 LLM Agent 可靠地协作完成长任务的系统,都可以照这个蓝图搭自己的看板。

---

## 附录 A:发布前核验清单

本文写作环境**无网络、无 shell**,以下时效性内容未能现场核验,发布前请逐项确认(标注处为写作时依据的来源):

1. **版本号与功能归属**:文中"v0.16"取自任务上下文标注;Kanban 在官方文档中的规范名称与 v0.16 release notes 请核对 https://github.com/NousResearch/hermes-agent 的 Releases 与 `hermes --version`。
2. **官方 Kanban 文档页**:https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban (写作时该 URL 来自随环境安装的官方 skill 参考 v3.2.0,内容未抓取核验)。
3. **文档索引**:https://hermes-agent.nousresearch.com/docs/llms.txt —— 一行一个功能的完整索引,查"kanban"可定位最新页面。
4. **CLI 动词与参数**:以本机 `hermes kanban --help`、`hermes kanban <verb> --help` 为准;配置键(dispatch_stale_timeout_seconds=4h、failure_limit=2、dispatch_in_gateway=true)以 `hermes config` 与文档为准。
5. **默认值**:心跳门限"1 小时"、"stale 4 小时"、"failure_limit 2"、"附件 25MB" 均来自官方 skill 参考与本文运行环境的系统协议文本,未对源码逐行核对。
6. **数据路径**:默认 `~/.hermes/kanban.db`、boards 目录布局来自本机现场(boards/blog/workspaces/…),不同安装版本可能不同。
7. **术语核对**:官方文档对该功能的称呼(本文按任务标题使用 "Kanban Swarm";skill 参考中称 "Kanban(multi-agent work queue)")——发布前统一口径。
8. **外部对照**:§1 对比表中 delegate_task/cronjob 的行为边界来自官方 skill 参考,未抓取文档页复核。

## 附录 B:术语表

| 术语 | 含义 |
|---|---|
| profile | 一套独立的 Hermes 实例配置(模型/会话/技能/记忆),worker 的身份 |
| dispatcher | 常驻调度器:提升、认领、spawn、回收、熔断 |
| worker | dispatcher 在 assignee profile 名下派生的执行会话 |
| claim | 调度器对 ready 卡的原子认领,带锁与过期 |
| run | 一次调度执行;`run_id` 全局递增 |
| parents / children | 任务依赖边;child 在全部 parent done 前停留在 todo |
| workspace_kind | scratch / dir / worktree:worker 的落盘环境 |
| heartbeat | worker 的存活信号,超时门限约 1 小时 |
| reclaim | 回收僵死 run,无惩罚重新排队 |
| artifact | 完成任务时声明上传的交付文件(绝对路径,须真实存在) |
| review run | 一次评审轮转:request_review → complete / request_changes / block |
