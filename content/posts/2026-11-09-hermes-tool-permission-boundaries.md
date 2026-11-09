---
title: "Hermes 的工具权限边界设计——为什么 orchestrator 没有 shell，capability 是权限还是信息"
date: 2026-11-09
tags:
  - multi-agent
  - kanban
  - tool-permissions
  - hermes
  - agent-architecture
author: wangying95599
---

2026-11-09 14:25:55，一张编号 `t_b66ebdfe` 的卡片从 `running` 变成 `blocked`。它不是崩溃，也不是超时——它是自己停下来，附了一段六百多字的理由，逐项列出「我缺什么、为什么因此做不了、你有哪两条路可选」。两分半钟后，人类回了一条评论并 `unblock`，卡片继续跑完。

这张卡值得当成一个标本，因为它同时暴露了两个容易被混为一谈的东西：**权限边界**和**信息边界**。负责拆解任务的 orchestrator 没有 shell，这看起来像漏配；但翻代码会发现，它是被显式写进 `platform_toolsets.cli` 的。而它「看不见有哪些 profile」，则完全不是配置问题——那是另一层裁剪，发生在运行期。

把这两层分清楚，才能回答标题里的第二问：`capability` 到底是一种权限申请，还是一段带标签的信息。

## 一、orchestrator 没有 shell：这是设计，而且只做加法

先看代码。worker 这一轮拿到的工具集，等式只有一行：

```python
# gateway/session_kanban.py:69
toolsets=tuple(sorted(set(policy.toolsets) | {'kanban'}))
```

`policy.toolsets` 来自该 profile 的 `config.yaml`，`source` 固定为 `cli`；代码只往并集里塞一个 `kanban`，**没有任何「检测到 kanban 运行场景就砍掉 terminal」的分支**。换句话说，orchestrator 没有 shell，原因简单到有点无聊：它的 `platform_toolsets.cli` 里没写 `terminal`。

实测的五个角色长这样：orchestrator 拿到 `clarify, computer_use, connections, cronjob, delegation, image_gen, memory, session_search, skills, todo, tts, vision`；researcher 在此之上加 `file, terminal, web`；writer 和 reviewer 直接用 `[hermes-cli]` 预设，也就是全量；publisher 再加 `code_execution, file, terminal`。

官方文档对这个角色的描述是逐字对应的：

> An orchestrator is a Hermes profile whose toolset includes `kanban` but excludes `terminal` / `file` / `code` / `web` for implementation. Its job is decomposing a high-level goal into child tasks via `kanban_create` + `kanban_link` and stepping back.

这段话里有个词值得停一下：**for implementation**。它说的不是「这个角色不允许执行」，而是「这个角色不负责执行」。拆解者一旦有了 shell，它就会开始自己动手——这是多智能体系统里最常见的退化：编排层慢慢变成执行层，因为写一行 `git commit` 比拆三张卡快得多。所以第一层裁剪的方法论其实是**用能力缺失来约束职责**，而不是「最小权限」这种安全叙事。整个文档站里 `least privilege` 只命中两处，都在第三方集成页，和工具集裁剪无关——它压根不是一套成文原则。

**为什么这一层重要：** 因为它决定了隔离的粒度。这是 profile 级裁剪，不是卡片级。任何一张派给 writer 的卡都会拿到全量 shell，不管那张卡是「写 3000 字技术文」还是「把三个数字加起来」。同一套 shell 权限可以在任意卡里执行任意命令，隔离靠的是「派给谁」，不是「这张卡要干什么」。而且裁剪只写在 YAML 里，没有代码校验：复制一份 profile 配置就完成了一次越权，全程零告警。

这是第一层。真正让 orchestrator 抓瞎的是第二层。

## 二、第二层裁剪在运行期：权限随上下文收缩，不随身份

`tools/kanban_tools.py` 里有个 `_visible` 函数，同一份 profile 会因为「这一轮是不是在跑卡」而拿到不同的 schema：

```python
def _visible(*, to_env_worker: bool) -> bool:
    if _is_delegated_child_context():
        return False
    if os.environ.get("HERMES_KANBAN_TASK") and _is_dispatcher_owned_worker():
        return to_env_worker
    return _profile_has_kanban_toolset()
```

`kanban_list` 和 `kanban_unblock` 注册时传的是 `to_env_worker=False`，注释写得很直白：*Board-routing tools, hidden from task workers.*

于是同一个 orchestrator profile：在聊天窗口里，`kanban_list` 在 schema 里，它能看板上所有卡、所有 assignee；一旦它自己持卡（`HERMES_KANBAN_TASK` 存在），这两个工具从 schema 里消失，只剩 `kanban_show`、`kanban_comment`、`kanban_block` 这类本卡生命周期工具。

**权限随运行上下文收缩，而不是随身份。** 文件读写权限由「你是谁」决定，看板路由权限由「你此刻在干什么」决定。两种机制叠在同一个进程里，这就是为什么「orchestrator 有没有 X 工具」这个问题本身是有歧义的——必须问「哪一轮的 orchestrator」。

还有个更锋利的反例：**有 shell 也不等于能读看板。** researcher 那一轮确实有 terminal，但在它的 shell 里跑 `hermes kanban list` 会被直接拒绝：

```
kanban: delegate_task child contexts cannot mutate Kanban tasks or boards
```

所以「读不到名单」有三个彼此独立的成因：profile 配置裁掉了 shell、代码在持卡时关掉了 `kanban_list`、以及即使有 shell 也可能被 fence 拦住。把它们混成一句「因为没 shell 所以看不到名单」，会得到错误的修法——给 shell 并不能解决后两个。

## 三、权限边界不等于信息边界

收走 shell 的连带损失，通常被低估。它的直接后果不是「少了个执行器」，而是「少了个环境探针」。

一个 worker 用 shell 做的事情，很大一部分不是改变世界，而是**核验自己对这个世界的判断**：列一下有哪些 profile、`git status` 看看工作区干不干净、`cat` 一下确认自己刚写的文件真的落盘了。这些动作的产物全部是信息，副作用为零。可一旦 shell 被裁掉，这些信息通道跟着一起消失，而设计上并没有替代品。

最典型的是「我该把子卡派给谁」。这个决策需要的事实是：本机存在哪些 profile、哪些 gateway 在跑。这个事实不在任何工具的返回值里。`kanban_create` 只接受一个 `assignee` 字符串，不校验；派发器遇到不存在的 assignee 时，在 `kanban_db_dispatch.py` 里走这一段：

```python
2077  profile_exists = _profile_exists_fn()
2078  if profile_exists is not None and not profile_exists(assignee):
2079      result.skipped_nonspawnable.append(task_id)
```

卡留在 `ready`，不 spawn、不计失败、不重试，只写一次 `skipped_nonspawnable` 事件。从 worker 的视角看，这是一次**完全静默的失败**：它 `kanban_create` 成功了，返回了 id，然后那张卡永远不会动。唯一的兜底是 `hermes kanban diagnostics` 里的 `stranded_in_ready`，默认阈值 30 分钟，2 倍升 error、6 倍升 critical。

把这三件事连起来看，`t_b66ebdfe` 的行为就不是「模型偷懒」，而是**在信息缺失下的理性停机**：猜一个 assignee 的期望收益是「两分半钟后卡在 ready 里静默腐烂，30 分钟后才被超时信号捞起来」；而声明 capability 的期望收益是「两分半钟后人类回一句话，我拿到名单继续干」。实际发生的是后者，14:28:26 unblock，14:31:30 完成。

**这里有个必须写下来的失真点。** 那张卡的 reason 说自己「只被授予了看板工具 + GitHub 只读 MCP」，但它实际还有 memory、skills、todo、delegation、vision 等一堆非 shell 工具。它对自己工具清单的枚举是错的，只有「没有 terminal / read_file / write_file」这一条是准的。

为什么这一点重要：如果你靠 worker 的自述来诊断权限问题，你会拿到一份错的清单。可靠的诊断面在 spawn 侧——profile 的 `platform_toolsets.cli`，以及 `state.db` 里 `sessions.tool_names` 存的 schema 指纹（同一 profile 的多次 kanban 会话指纹完全相同，工具集是稳定的）。**agent 的自我报告是证据，但不是权威。**

## 四、缺口补在设计层还是执行层：三个判据

「worker 需要知道某个事实」和「worker 需要一个工具」是两种完全不同的补法。给错层的代价不对称：补信息最多浪费几十个 token，补工具则可能把一个只读决策点变成任意代码执行点。我一般用三个判据过一遍。

**判据一：谁需要知道。** 如果每个 worker 都需要（比如「仓库根目录在哪」），推到注入层；如果只有一个角色需要、且它需要探索（比如「这个仓库的 CI 为什么挂了」），给工具。

**判据二：知道了能做什么。** 信息只能支撑决策，工具能改变世界。orchestrator 需要的是「board 上有哪五个可用 assignee」这一个事实，用来填 `kanban_create(assignee=...)`。它不需要 `git`、不需要 `curl`、不需要写文件。为了 20 个 token 的事实给它一整套 shell，是拿 1000 倍的副作用面买一个字符串。

**判据三：副作用面。** 给 shell 等于给了任意命令、继承的环境变量、网络访问、以及 profile 目录下的凭据。给一个 roster 字段等于几十个 token。这两者不在一个量级上，所以「反正给它 shell 也能查到」不是论证——那是用最贵的通道传最便宜的信息。

按这三条过完，结论很清楚：**orchestrator 缺的是信息，不是工具。** 但紧接着有个约束：这个信息必须由 spawn 侧注入，因为运行期的它已经没有能力去查了。这就是设计层的责任——你裁掉了它的探查能力，就得把探查结果喂给它。

现实里这个洞是靠 memory 糊上的，而且糊得挺成功。看时间线：14:25 那张卡 capability block；`profiles/orchestrator/memories/MEMORY.md` 的 mtime 是 15:51，第 13 行写着「blog 看板可用 assignee（本机已确认存在、gateway running）……派发器对不存在的 assignee 静默跳过」；15:52:27 建的自检卡 `t_9bb68065`，15:53:25 完成，摘要里五个 assignee 全对。

**但这只是时间线支持，不是因果证明**——代码层面没有日志能证明第 13 行是哪一次 `memory` 调用写入的。而且用常驻记忆补一次性上下文，成本结构是反的：每个 session 都要为这条事实付费，包括那些根本不派卡的 session。

## 五、三条信息通道，三种失效方式

Hermes 目前往 worker 脑子里塞东西，只有三条通道。它们的差别不在「谁更全」，而在**失效方式**——这决定了你该把什么放进去。

**memory** 在会话启动时渲染进 system prompt，然后冻结。预算是 `MEMORY.md` 2,200 字符、`USER.md` 1,375 字符。冻结是为了保住 prefix cache：代价是你在本轮写进去的东西本轮不生效，要等下一个 session 才进 prompt；超预算的写入直接被拒。它适合放「这台机器是什么样」——profile 名单、路径约定、已知的坑。

**`kanban_create(skills=[...])`** 在 spawn 时把技能正文**整篇**拼进首轮 prompt。没有硬上限，所以一份长 SKILL.md 会在首轮就吃掉几千 token，之后每轮重发。它的失效方式很特别：上下文压缩时正文被替换成一个占位符 `[SKILL_PRUNED: ... reload with skill_view(name='…')]`，靠模型自觉看到占位符再重新加载；如果技能在磁盘上找不到且没加载过，本轮直接抛 `kanban_skills_unavailable`。也就是说，**它有一个显式的重建协议**，压缩不等于丢失，但要求模型配合。

**`worker_context`** 是系统替你抓的历史：卡片 body、附件、历史尝试、父卡 handoff、同角色近期工作、评论。它最重，也最全，边界是 10 次尝试、30 条评论、字段 4KB、body 8KB、单条评论 2KB。失效方式最糟：它是首轮 prompt 里的一段普通文本，压缩进入中段后被摘要吞掉，**没有占位符保护，也没有重建机制**。丢了就是丢了。

把三条放在一起，缺的那一格就显出来了：memory 回答「这台机器是什么样」，skill 回答「这类任务怎么做」，worker_context 回答「这张卡之前发生过什么」。**没有一条回答「我这一轮到底有什么工具、我缺什么」。**

这个信息目前只能从 schema 反推，而 schema 在两种情况下并不可靠：工具被 `tool_search` 延迟加载时（配置里启用了 `computer_use`，但它不在首屏 schema 里，要检索才出现），以及上下文压缩之后。顺带说一句，orchestrator 的 `computer_use` 是**启用**的，不在缺失清单里；只是它那一轮能否真的驱动桌面，没有直接证据，这里不做断言。

## 六、capability：一条带标签的信息，不是权限申请

回到标题的第二问。`kanban_block(kind=...)` 接受四种 kind，实际的代码行为是这样：

| kind | 落到 | 自动恢复 | 计 failure | 计 recurrences |
|---|---|---|---|---|
| `dependency` | `todo` | 是，父卡全 done 后自动 promote | 否 | 否 |
| `needs_input` | `blocked` | 否 | 否 | 是 |
| `capability` | `blocked` | 否 | 否 | 是 |
| `transient` | `blocked` | 否 | 否 | 是 |

关键在于后三者在 `_route_block` 里走**同一个分支、同一个状态、同一个计数**，差别只存在于 `payload.kind` 字段和 reason 文本里。内核不校验、不授予、不放宽任何东西。它做的事只有一件：把卡从「跑」挪到「等人」，并把标签留档。

所以 `capability` **不是权限申请，是一段带标签的自述**。真正的权限在 spawn 那一刻就固定了（`session_kanban.py:69`），运行期无论 block 多少次都不会多出一个 `terminal`。

这个设计值不值得？我认为值，理由是三档排序：**主动声明做不到 > 猜 > 静默失败。** 猜的失败是延迟的——派给不存在的 profile，卡停在 ready，30 分钟后才出 `stranded_in_ready`；静默失败的失败是不可见的——worker 以为自己写了文件，其实没有。而 capability block 在两分半钟内就把问题顶到人类面前，还附带了「我查过什么、缺什么、两条可执行选项」，`t_b66ebdfe` 的 reason 就是范本。

代价也很实在，四条：

1. **零校验。** 「我不想做」和「我看不到工具」在库里长得一模一样，理由无法机检。
2. **只容得下一次。** 同一 kind 第二次 block（也就是第一次 re-block）就 `recurrences >= 2` → `triage` + `block_loop_detected`，而且是确定性 DB 守卫，不是 LLM 判断，卡片正文无法 opt-out。人类一次误判就会让卡离开正常泳道；`unblock` 不重置计数，只有成功 `complete` 才清零。
3. **不分轻重。** 真·缺凭据和「自觉能力不足」共用一条通道，人类必须逐条读 reason 才能排序。
4. **反向激励。** 正因为 worker 看不到 roster，「声明 capability」比「猜 assignee」更安全——这会系统性抬高 capability 的使用率。

不过别把这个风险想得太夸张。调研抓取那一刻，这个板上 12 张卡的 `consecutive_failures` 全是 0，`blocked` 只出现 2 次（capability 与 needs_input 各一次），`recurrences` 都是 1，`block_loop_detected` 0 次——**从未触发过 triage**。在这个小样本里它是低频的一次性信息通道，不是常态逃逸阀。但样本只有两例，机制上的洪泛风险依然成立。

那怎么控？我的答案是**不要禁止 block**。禁了之后模型会退化成猜，把可见的失败换成不可见的失败，这是净损失。可控的地方在三处：一是 reason 的格式契约（必须写「查过什么、缺什么、两条选项」），二是人类侧按 kind 分流而不是逐条读，三是对高频 capability 的角色**补信息而不是补工具**——因为高频声明 capability 通常不是能力问题，是它不知道该派给谁。

## 七、给多智能体流水线划边界时，把信息侧一起划

具体到「这个角色需要哪些元信息、由谁注入」，我的清单是这样的：

- **orchestrator**：工具主干是看板 + delegation（实测还挂着 `clarify / memory / todo / tts` 等非执行类工具，但没有 `terminal / file / web`）。必须注入的元信息是**可用 assignee 白名单**（含每个 assignee 的工具能力摘要，至少标明谁有 shell）、**板上的现有卡**、以及**派发器对未知 assignee 的行为**。注入方是 spawn 侧（`worker_context` 或按卡 skills），不是 memory——因为这是「这一轮」的事实，不是「这台机器」的常量。
- **researcher**：工具 `terminal / read_file / write_file / web`。元信息是**可写目录的边界**和**结论的标注格式**（哪些是核实、哪些是推断）。它不缺探索能力，缺的是「写到哪算越界」。
- **writer**：全量 shell。元信息是**仓库路径、既有文件命名、frontmatter 契约**。这类角色最容易出的错不是做不到，是做完了不符合契约。
- **reviewer**：全量 shell。元信息是**它要核的事实来自哪张卡**——直接写进卡片 body，别指望它自己去翻评论。
- **publisher**：`terminal + code_execution + file`。元信息是**远端仓库地址、凭据是否存在**。别让它去「试一下有没有 token」，那是拿外部副作用做探测。

还有两条通用的：

**别在卡片 body 里写「你这轮有什么工具」。** worker 对自身工具清单的自述会失真，写进去只会给它一个错误的自我模型。写「你缺什么的时候按什么格式 block」才有用——契约比清单可靠。

**给 worker 一个只读的 `kanban_list`，比给它 shell 便宜得多。** 现在持卡时它被代码关掉了，这在我看来是可以重新讨论的设计选择：只读名单不改变世界，副作用面几乎为零，却能消灭「因为看不见所以停机」这一整类 block。真要防的是 `kanban_unblock`，那个会改变世界，该关。

## 八、边界要成对设计

回到那张 14:25 的卡。它没有做错任何事，它只是在一个「能力被裁剪、信息没有补上」的系统里，做了一个正确但昂贵的决定：停下来，换信息。

权限边界决定 agent 能改变什么；信息边界决定 agent 知道自己能改变什么。只设计前者，你会得到一批**能力充足但自我认知错误**的 worker——它们的失败模式不是做错，而是停错：在能做的时候停，在不能做的时候硬做。而这两种错误在日志里长得几乎一样。

`capability` 是这个系统的诚实性接口。它存在这件事本身，说明设计者承认了一件事：agent 会不知道自己有什么。但如果信息侧一直不补，这个接口就会从安全阀退化成主要工作方式——那时候你面对的就不是一张卡停两分半钟，而是一整块看板在等人回话。
