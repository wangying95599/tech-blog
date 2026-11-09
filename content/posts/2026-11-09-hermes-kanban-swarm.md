---
title: "Hermes Kanban Swarm：用一块看板调度一群 Agent"
date: 2026-11-09
tags: ["Hermes Agent", "Kanban", "多智能体", "架构设计", "AI 开发"]
author: wangying95599
---

## 一、单 Agent 的三个死结

让一个 Agent 干完一个长任务，迟早会撞上三堵墙。

**第一堵是上下文。** 需求分析、资料检索、初稿、审校、发布，每一步都会往对话里堆东西。等做到第 5 步，第 1 步的关键约束已经被压缩算法吞了，Agent 开始自信地违背自己十分钟前写下的规则。

**第二堵是可恢复性。** 整个任务是一个进程里的一个循环。进程崩了、模型超时了、token 用光了，前面两小时的工作只剩一堆 JSONL。你没法说“从审校那步接着做”，只能重跑，而重跑的代价和不确定性都随任务变长而上升。

**第三堵是串行。** 三个互不依赖的子任务（查资料、写示例代码、整理配图）在一个 Agent 里只能排队。总时长等于三者之和，而不是三者最大值。

`delegate_task` 能缓解第三堵：它开一个独立上下文跑子任务，只把结论回传。但子任务是父亲一轮工具调用里的东西——父亲还在跑，子任务就活不了；父亲上下文爆了，子任务一起没。它解决的是“别把子任务的中间过程塞进我的上下文”，不是“这件事要在我不在的时候也能推进”。

Kanban Swarm 换了个思路：**把任务从进程内存里搬到磁盘上。**

## 二、它到底是什么

它不是让几个 LLM 在一个群里互相聊天。聊天式多智能体最大的问题是：状态在对话历史里，没有权威副本，谁都能改，谁都能忘。

Kanban Swarm 的组成只有三样东西：

- **一块持久化的工单板。** 一个 SQLite 数据库，存任务、状态、依赖、交接摘要、评论事件。板是**硬边界**：worker 进程的环境变量里 `HERMES_KANBAN_BOARD` 被固定住，它看不到别的板。多项目隔离靠“多开板”，而不是靠命名空间。
- **一个派发器（dispatcher）。** 默认跑在 gateway 进程里，不额外起服务。
- **若干 worker 进程。** 每个 worker 是一个按 profile 配置启动的 Hermes 实例，有自己独立的 config、会话、技能、记忆。

三者之间的通信只有一条：**卡片状态 + 结构化交接**。worker 之间零共享上下文。这不是妥协，是刻意设计——正因为不共享，一个 worker 的上下文爆炸不会传染给另一个，成本也才是可预测的（每张卡的成本只取决于那张卡的体量）。

## 三、六个核心抽象

### 1. 卡片与状态机

一张卡是一个 id（形如 `t_b66ebdfe`）+ 标题 + 正文 + assignee + 状态。状态不是“待办/进行中/完成”三档，而是一条带分支的流水线：

```
triage → todo → ready → running → review → done
                  ↑                    ↓
               blocked ←───────────────┘

scheduled（挂起，等某个时刻再进 ready）
review → running（request_changes：打回重做）
```

关键点在于 `todo` 和 `ready` 是分开的。`todo` 是“存在但还不能开工”（依赖没满足），`ready` 是“可以被认领”。派发器只在两者之间搬运，它不猜你的意图。这个区分看起来琐碎，实际是整个系统能自动化的前提：**“能不能开工”变成一个可以计算的状态，而不是需要人去判断的事。**

### 2. assignee = profile

卡片的执行者是一个 profile 名，不是一个模型名。这是我认为最被低估的设计。profile 是隔离的运行环境：独立 config、独立会话库、独立技能集、独立记忆。于是“研究员”和“审稿人”不是两个 prompt，而是两个不同的 Agent 实例——一个装了检索类技能、模型选贵的；一个装了审校流程、模型选稳的。派发器做的事很朴素：看到卡上写着 `writer`，就 spawn 一个 `writer` profile 的进程，把卡 id 塞进它的环境变量。

### 3. workspace 三种形态

worker 要动文件，就得回答“在哪动”。三种答案：

| kind | 含义 | 适用 |
|---|---|---|
| `scratch` | 每张卡一个全新临时目录 | 只产出结论/文本，不需要留痕 |
| `dir` | 一个已存在的共享绝对路径 | 多张卡要改同一批文件 |
| `worktree` | 从主仓库开一个 git worktree + 确定性分支 | 并行改代码，互不踩脚 |

代价要记住：**`scratch` 在任务结束时会被删掉。** 所以凡是“产物”（报告、图表、生成的文件），必须显式声明成 artifact，系统会在清理前把它们复制到任务附件的持久位置；否则你会在完成之后发现文件没了。这是新手最常踩的第一个坑。

### 4. 依赖图：扇出与扇入

卡与卡之间可以有父子边。语义很硬：**子卡在所有父卡变成 done 之前，不会进 `ready`。** 于是两种拓扑自然长出来：

- **扇出**：orchestrator 卡扇成 N 张专家卡，彼此无依赖，同时 ready，同时跑。
- **扇入**：一张汇总卡把 N 张研究卡都写进 parents，等它们全完再开工，读它们的结构化交接。

这比“在 prompt 里说‘等你做完再’”可靠得多——依赖是数据库里的边，不是自然语言里的承诺。

### 5. 结构化交接，而不是聊天记录

worker 结束时交回的不是一段散文，而是 `summary`（给人看的 1–3 句）+ `metadata`（给机器看的 dict：改了哪些文件、跑了哪些测试、有哪些决策）。下游 worker 的上下文里，父卡的这两样东西是**原样注入**的，它读不到父卡的对话过程。

这个约束很强，也很健康：它逼你把“下一棒真正需要的信息”从过程里提炼出来。顺带一提，正因为这些字段是持久化且会注入下游的，**密钥、token、原始 PII 一律不许写进去**——一句话写错，它就跟着卡片活到项目结束。

### 6. 生命周期 API

worker 手里的工具就是它的一生：

```
kanban_show           认领时先读：标题/正文/父卡交接/历史尝试/评论
kanban_heartbeat      长任务打卡，证明自己还活着
kanban_comment        留给人的持久备注（不是临时思考）
kanban_create         发现后续工作 → 派给别的专家，而不是自己做
kanban_link           事后补一条依赖边
kanban_complete       完成，带 summary + metadata + artifacts
kanban_block          卡住，必须说明卡在哪一类
kanban_request_review 实现完了，交给审阅
kanban_request_changes 审阅后打回（审阅者用）
```

注意这里没有“发送消息”的方法。worker 之间不通信，只通过卡片状态交接。（同属工具集、但不在这条主链上的还有 `kanban_list`、`kanban_attach` / `kanban_attach_url` / `kanban_attachments` 和 `kanban_unblock`。）

## 四、派发循环，逐拍拆解

派发器每一拍（默认 60 秒）做四件事，顺序不能乱：

1. **回收**。找出认领超时、且最近没有心跳的任务，把锁还回池子。这一步的判据是 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时）+ 最近心跳时间；认领 TTL 过期、进程已不在的卡也在这步收回。
2. **提升**。把父任务全部完成、且不再阻塞的 `todo` 卡推成 `ready`。
3. **原子认领**。在同一个事务里把 `ready` 改成 `running` 并写锁，防止两个派发器抢同一张卡。
4. **启动**。按 assignee 找到 profile，起子进程，注入卡 id 和工作区。

连续启动失败会累计，超过 `kanban.failure_limit`（默认 2）后自动把卡 block 掉——不然一张写错的卡会无限重启，烧钱。

心跳的实操建议很直白：**预计超过 1 小时的任务，至少每小时打一次心跳。** 否则同时满足“超过 stale 阈值”和“最近一小时没心跳”就会被回收。心跳超时的回收不算失败（不累加失败计数），但**你会丢掉这一轮的全部进度**——它是“重新排队”，不是“续跑”。

## 五、把“卡住”分类，是整个系统的精髓

`kanban_block` 强制你选一个 kind，这不是形式主义：

| kind | 含义 | 系统怎么处理 |
|---|---|---|
| `dependency` | 我在等另一张卡 | 回到 `todo`，等父卡 done 后**自动**恢复，不需要人 |
| `needs_input` | 需要人拍板 | 浮到看板上等人，解锁后重新派发 |
| `capability` | 硬墙：没权限、没凭据、没有 agent 能做的事 | 直接升级给人 |
| `transient` | 抖了一下，可能会自己好 | 仍会浮到看板上，但通常重试一次就过 |

四种“卡住”混成一种，结果就是人被迫处理本来可以自动恢复的等待，最后对看板脱敏、不再看它。分开之后，人的注意力只花在 `needs_input` 和 `capability` 上——`dependency` 自己会走，`transient` 重试即可，真正要人做判断的只有这两种。

同一逻辑延伸到审阅：**review 不是 block。** 审阅单独走一条通道（`kanban_request_review` / `kanban_request_changes`），因为它会反复来回，而反复的 block/unblock 会被系统当成“卡住循环”做升级告警。把正常的工作节奏塞进告警通道，等于教系统对噪音报警。

## 六、orchestrator 与 worker：谁决定什么

这是多智能体系统最容易糊掉的地方，Kanban 的处理很明确：

**orchestrator 决定设计，worker 只做实现。** 命名方案、数据结构、文件格式、接口形状——这些必须在扇出之前定死，并且**写进每一张子卡的正文**。原因是硬性的：worker 看不到兄弟卡的上下文，看不到 orchestrator 心里的方案。两张子卡各自“自己决定一下”，就会产出两套不兼容的东西，然后需要第三张卡来擦屁股。

反过来，worker 有一条纪律：**发现后续工作就开卡派出去，不许自己扩范围。** 因为 scope creep 在单 Agent 里只是“多写了几百行”，在多智能体里会变成“三张卡同时改了同一个文件”。

关于扇出的一个现实约束：分派前先确认 assignee 真的存在。**派发器对不存在的 assignee 是静默跳过的**——卡会安安静静躺在 `ready` 里，不报错、不 block，只在事件流里留一条 `skipped_nonspawnable`（要 `hermes kanban show` 或 `hermes kanban dispatch --dry-run` 才看得见）。这是我在实战里见过最安静的失败。

## 七、这篇文章自己就是被这套系统跑出来的

这不是比喻。本文的写作任务就是以一张真实的卡片存在的：

```
卡片 id : t_b66ebdfe
看板    : blog
run     : 1
工作区  : scratch @ .../kanban/boards/blog/workspaces/t_b66ebdfe
```

它的事件流（从卡片记录里直接读出来）：

```
created              → 卡片建好，状态 ready
claimed              → 某个进程拿到锁，lock 有过期时间
tip_scratch_workspace→ 系统提示：scratch 工作区在完成时会被删除
spawned              → 起了一个 worker 进程
owner_admitted       → 该进程登记为卡的 owner（带 session_id）
worker_bound         → worker 绑定到这次的 claim_lock
heartbeat            → 心跳
```

这条链上有意思的地方：**卡片比进程活得久。** 进程挂了，卡片还在 `running`；锁过期，派发器把它捞回 `ready`；下一个 worker `kanban_show` 一次，就继承了父卡交接、历史尝试和全部评论——它不需要重读任何对话历史，因为它压根没有。

这也是为什么这一节能写出来：我不需要相信我自己记忆，我只需要读卡片。

## 八、八个坑（按踩到的频率排序）

1. **`scratch` 工作区的产物会消失。** 要留的东西必须进 `artifacts`，或者一开始就用 `worktree` / `dir`。
2. **assignee 拼错 = 静默死卡。** 卡不会报错，只会不动。
3. **长任务不打心跳。** 被回收后进度清零，且没有告警。
4. **把 review 当 block。** 反复 block/unblock 会被升级成“卡住循环”，正常的往返变成告警。
5. **把密钥写进 summary/metadata。** 这两个字段会被注入下游 worker，并且在数据库里长期存在。
6. **一张卡塞多个交付物。** 交付物混淆 → 审阅者无法判断“这条通过了吗”。
7. **orchestrator 不在卡里写决策。** 兄弟卡不可见，各做各的，最后拼不起来。
8. **以为并发是免费的。** N 张卡同时 ready 就是 N 个并发的模型调用。扇出前先算钱。

## 九、上手：最小可跑的三卡流水线

```bash
# 1. 建板（不建就落在 default 板上）
hermes kanban boards create blog --switch

# 2. 建卡（assignee 必须是已存在的 profile）
hermes kanban create "调研 Hermes Kanban 的实现细节" --assignee researcher
hermes kanban create "写文章初稿" --assignee writer

# 3. 连依赖：初稿等调研完成才 ready
hermes kanban link <research-id> <draft-id>

# 4. 看板与详情
hermes kanban list
hermes kanban show <draft-id>

# 5. 派发（gateway 内默认自动派发，也可手动触发一拍）
hermes kanban dispatch

# 6. 或者一条命令建出整张 Swarm 拓扑：root 黑板 → N 个并行 worker → verifier → synthesizer
hermes kanban swarm "调研并成稿：Hermes Kanban 实现细节" \
  --workers researcher,researcher-b --verifier reviewer --synthesizer writer
```

配置片段（`config.yaml`）：

```yaml
kanban:
  dispatch_in_gateway: true          # 派发器跑在 gateway 进程内，不用额外起服务
  failure_limit: 2                   # 连续启动失败 N 次后自动 block
  dispatch_stale_timeout_seconds: 14400   # 4 小时无心跳的认领会被回收
```

然后记住两条纪律：父卡全部 `done` 子卡才会 `ready`；worker 结束时必须给出 `summary` + `metadata`，散文式的“我做得差不多了”在下一棒那里等于零信息。

## 十、什么时候不该用它

Kanban Swarm 的每个优点都有代价：状态要落盘、进程要重启、交接要序列化。所以：

- **一次性、十分钟内能跑完的活**，用 `delegate_task` 就够了，它共享进程、零启动成本。
- **需要小时级、可断点续跑、多专家协作**的活，才值得上板。
- **需要实时对话协商的活**（几个 Agent 反复讨价还价），它不适合——它刻意去掉了通信通道。

判断标准可以简化成一句：**如果这个任务失败后你需要“从某一步接着做”，它就该在看板上。**

## 附录：事实核对

- Kanban 官方文档：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban （源文件 `website/docs/user-guide/features/kanban.md`，“Kanban Swarm” 这一表述即出自该页）
- 相关文档页：`/docs/user-guide/features/kanban-tutorial`、`/docs/user-guide/features/kanban-worker-lanes`、`/docs/user-guide/features/kanban-multi-gateway`、`/docs/user-guide/features/goals`、`/docs/user-guide/features/deliverable-mode`
- 源码对应模块：`hermes_cli/kanban.py`、`hermes_cli/kanban_swarm.py`、`hermes_cli/kanban_parser.py`、`gateway/kanban_watchers_dispatcher.py`（仓库 `NousResearch/hermes-agent`）
- `kanban.dispatch_stale_timeout_seconds`、`kanban.dispatch_in_gateway`、`kanban.failure_limit` 三个配置项均见于上述 kanban 文档页（Dispatcher 段与 config 片段）
- 本文第七节的事件流、状态名、工作区形态取自该卡片的真实运行记录
- **版本说明**：官方文档站没有版本化的 changelog 页，因此本文不绑定版本号；文中机制按 `/docs/user-guide/features/kanban` 逐条核对（核对时该页对应 main 分支，发布版为 v0.23.0）。
