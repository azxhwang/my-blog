# ACP、acpx、A2A：三层协议，三个世界

在当前的 agent 生态里，`ACP`、`acpx`、`A2A` 这几个名字经常一起出现，但它们其实处在完全不同的层：有的是协议规范，有的是实现与后端插件，有的则是跨系统互操作标准。

本文尝试把这三者放在同一张图里讲清楚：

- **ACP**：Agent Client Protocol——"客户端 ↔ agent runtime"的会话协议
- **acpx**：OpenClaw 生态中的 ACP backend / 插件与 CLI 实现
- **A2A**：Agent2Agent Protocol——"agent ↔ agent"的开放互操作协议

> 适合读者：正在搭建或使用 agent 系统的开发者，对 agent 协议生态感兴趣的技术人员。

---

## 目录

- [背景：为什么到处都是"XX 协议"？](#背景为什么到处都是xx-协议)
- [三个概念一览](#三个概念一览)
- [前置知识：什么是 Agent Runtime？](#前置知识什么是-agent-runtime)
- [ACP：Agent Client Protocol，是"会话线"](#acpagent-client-protocol是会话线)
- [acpx：OpenClaw 里的 ACP backend 与 CLI 实现](#acpxopenclaw-里的-acp-backend-与-cli-实现)
- [A2A：Agent2Agent Protocol，是"agent 互联语言"](#a2aagent2agent-protocol是agent-互联语言)
- [放在一起：谁和谁在说话？](#放在一起谁和谁在说话)
- [实际场景中的边界感](#实际场景中的边界感)
- [一个简单的心智模型](#一个简单的心智模型)
- [总结](#总结)
- [参考资料](#参考资料)

---

## 背景：为什么到处都是"XX 协议"？

随着 agent 系统越来越复杂，大家发现光有"一个大模型"远远不够，还需要标准化：

- 人类或 IDE 要怎样和 agent runtime 对话，才能稳定地建会话、发指令、收流式输出？
- 一个 agent 系统要怎样和另一个完全独立的 agent 系统协作，而不是硬写一堆私有 HTTP API？
- 具体某个框架（比如 OpenClaw）内部，又需要怎样的后端插件，把这些协议真正"跑起来"？

ACP、acpx、A2A 就分别对应了这三层问题：**会话协议、实现后端、多 agent 互操作标准**。

---

## 三个概念一览

先用一句话快速对齐三个词：

- **ACP**：面向"客户端 ↔ agent runtime"的协议和会话模型，定义如何 spawn / prompt / cancel / load 一个 agent 会话。
- **acpx**：在 OpenClaw 生态里落地 ACP 的具体实现，是 ACP backend/plugin + CLI 工具，用来驱动 Codex、Claude Code、Gemini CLI 等外部 coding harness。
- **A2A**：一个独立的开放标准（Agent2Agent Protocol），定义"不同组织、不同框架的 agent 系统之间如何发现彼此、协商交互方式、协作完成任务"。

可以粗暴地记：**ACP 解决"一个 agent 怎么被用"，A2A 解决"多个 agent 怎么互相用"，acpx 负责在 OpenClaw 里把 ACP 真实跑起来。**

---

## 前置知识：什么是 Agent Runtime？

在深入 ACP 之前，需要先理解一个核心概念：**Agent Runtime**。

### Runtime = 运行时环境

类比其他技术栈：

| 场景 | Runtime |
|------|---------|
| Java 程序 | JVM |
| JavaScript | Node.js / 浏览器 |
| Python 脚本 | Python 解释器 |
| **Agent** | **Agent Runtime** |

Agent runtime 就是 **agent 程序的执行环境**，负责让 agent "跑起来"并管理它的整个生命周期。

### 为什么 Agent 需要 Runtime？

Agent 和普通程序不同，它不是"输入→计算→输出"就结束了，而是一个持续的循环：

```
用户指令 → 理解 → 规划 → 调用工具 → 观察结果 → 调整计划 → 继续执行 → ...
```

这个循环可能持续很久，中间需要：

- **会话状态管理**：记住上下文、历史对话、中间结果
- **工具调用**：执行 shell 命令、读写文件、调用 API
- **流式输出**：边思考边返回，而不是等全部完成
- **中断与恢复**：用户随时可能取消，或者暂停后继续
- **权限控制**：哪些文件能读、哪些命令能执行

这些都不是"调用一次大模型 API"就能解决的，需要一个专门的运行环境来管理——这就是 **agent runtime**。

### Agent Runtime 的内部结构

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Runtime                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              会话管理器                          │    │
│  │   - sessionId 管理                              │    │
│  │   - 会话恢复、持久化                              │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │              工具执行器                          │    │
│  │   - shell 命令                                   │    │
│  │   - 文件操作                                      │    │
│  │   - 外部 API 调用                                │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │              状态与记忆                           │    │
│  │   - 对话历史                                     │    │
│  │   - 中间结果                                     │    │
│  │   - 执行轨迹                                     │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │              LLM 调用层                           │    │
│  │   - 模型 API 调用                                │    │
│  │   - 流式响应处理                                  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 实际产品中的例子

| 产品 | Agent Runtime |
|------|---------------|
| OpenClaw | OpenClaw Gateway |
| Claude Code | 内置的执行环境 |
| Cursor | IDE 内的 agent 运行时 |
| LangGraph | 应用级别的 runtime 框架 |

### ACP 和 Agent Runtime 的关系

理解了 runtime，ACP 的定位就清晰了：**ACP 是客户端和 agent runtime 之间的通信协议**。

```
┌──────────┐                    ┌─────────────────┐
│   IDE    │                    │  Agent Runtime  │
│   CLI    │◄────── ACP ───────►│  (OpenClaw)     │
│  Client  │   spawn/prompt/    │                 │
│          │   cancel/close     │  管理会话状态    │
└──────────┘                    │  执行工具调用    │
                                │  调用 LLM       │
                                └─────────────────┘
```

ACP 定义的是：
- 怎么告诉 runtime "创建一个新会话"（spawn）
- 怎么发送指令（prompt）
- 怎么取消执行（cancel）
- 怎么关闭会话（close）
- 怎么接收流式输出

**Runtime 是"干活的人"，ACP 是"指挥它的语言"。**

---

## ACP：Agent Client Protocol，是"会话线"

在 OpenClaw 文档里，ACP 主要扮演两种角色：

1. **作为 runtime**：`sessions_spawn` 时指定 `runtime: "acp"`，让某个会话通过 ACP backend 去运行"外部 coding harness"（比如 Codex、Claude Code）。
2. **作为桥接协议**：`openclaw acp` 子命令可以跑在本地，把 IDE、CLI 这类 ACP 客户端通过 stdio 连接起来，再转成 WebSocket 发送到 OpenClaw Gateway。

从协议视角看，ACP 管的就是：

- 如何创建 / 标记 / 恢复会话（sessionId、loadSession）
- 如何发送 prompt、取消执行、关闭与清理会话（prompt / cancel / close）
- 如何做流式输出与状态跟踪（progress、intermediate steps）

你可以把 ACP 想成一条"IDE ↔ agent runtime 的专用会话线"：不管底层是哪个模型、哪个框架，沿着这条线说标准的话，大家就能安全地创建、恢复和关闭一个 agent 会话。

---

## acpx：OpenClaw 里的 ACP backend 与 CLI 实现

**acpx 不是另一个协议**，而是 OpenClaw 生态中用来"承载 ACP"并驱动外部 coding harness 的实现组件。

在 OpenClaw 文档中，你会看到类似配置：

```yaml
gateway:
  acp:
    enabled: true
    backend: acpx
    acpx:
      default_harness: claude-code
      timeout: 300s
```

这表示：

- Gateway 开启了 ACP 能力（可以处理 ACP 会话）
- 具体怎么把 ACP 请求落到真实的执行环境上，由名为 `acpx` 的 backend 插件负责

acpx 在整个链路里承担的是：

- 把 "spawn 一个 session、发送 prompt、cancel、close" 这些 ACP 指令，翻译成对应的外部 coding harness 操作（例如 Codex / Claude Code 的 CLI 或本地 server 调用）
- 处理权限策略、工作目录（cwd）、运行模式（run / session）、超时等运行时参数
- 在一些模式下，反向作为 client，通过 ACP 把 `openclaw` 暴露出去，让其他 ACP-aware 工具把 OpenClaw 当"远端 agent runtime"

一句话：**ACP 是"语言"，acpx 是"会讲这种语言并干活的人"——在 OpenClaw 世界里目前最主要的那个人。**

---

## A2A：Agent2Agent Protocol，是"agent 互联语言"

A2A 则完全是另一条线：它不关心 IDE 如何连到某个 agent，而是关心一个 agent 系统如何连到另一个 agent 系统。

根据官方定义，Agent2Agent Protocol 的几个关键点包括：

- **Agent Cards**：描述一个 agent 的能力、连接方式、支持的交互模态等，用于"发现与注册"
- **JSON-RPC 2.0 over HTTP(S)**：做请求-响应，并辅以流式输出与异步通知机制，用于复杂任务与长时会话
- **多 agent 设计**：专门为"多个、各自独立的 agent 服务"设计，而不是为单用户 / 单 IDE 会话设计

### Agent Cards 示例

Agent Cards 是 A2A 的核心概念，一个简化的示例如下：

```json
{
  "name": "OrderAgent",
  "description": "处理订单相关请求的 Agent",
  "capabilities": ["place_order", "check_status", "cancel_order"],
  "endpoints": {
    "jsonrpc": "https://api.example.com/a2a"
  },
  "authentication": {
    "type": "bearer"
  }
}
```

通过 Agent Cards，一个 agent 可以"告诉"其他 agent 自己能做什么、怎么调用。

在典型示例里，一个"采购 agent"可以通过 A2A 向"审批 agent""物流 agent""风控 agent"发出请求与协作，而这些服务可能分别运行在不同团队、不同技术栈上，只要都说 A2A 这门"语言"。

所以如果说 ACP 像"一个 IDE 对一个 agent runtime 的私密热线电话"，那 A2A 就像"跨公司的通用业务接口标准"，核心关注点在互联互通与协作。

---

## 放在一起：谁和谁在说话？

为了不混淆，最直接的方式是看"通信两端是谁"：

- **ACP**：客户端（IDE、CLI、bridge） ↔ 单个 agent runtime / 会话
- **acpx**：OpenClaw Gateway ↔ 外部 coding harness；或者 acpx ↔ `openclaw` ACP 端点（当 acpx 作为 client 时）
- **A2A**：独立的 agent 系统 A ↔ 独立的 agent 系统 B（通常是两个服务或产品）

### 架构图

```
┌─────────────┐                      ┌─────────────────┐
│    IDE      │                      │   OpenClaw      │
│    CLI      │◄────── ACP ─────────►│    Gateway      │
│   (Client)  │      (会话协议)        └────────┬────────┘
└─────────────┘                               │
                                              │ acpx (实现层)
                                              ▼
                                    ┌─────────────────┐
                                    │  Codex          │
                                    │  Claude Code    │
                                    │  Gemini CLI     │
                                    └─────────────────┘

┌─────────────────┐                      ┌─────────────────┐
│   Agent A       │                      │   Agent B       │
│  (采购系统)      │◄────── A2A ─────────►│  (审批系统)      │
└─────────────────┘    (互操作协议)        └─────────────────┘
```

### 对比表

| 维度 | ACP | acpx | A2A |
|------|-----|------|-----|
| 本质 | 会话协议 | 协议实现 | 互操作标准 |
| 通信双方 | IDE/CLI ↔ Agent | Gateway ↔ Harness | Agent ↔ Agent |
| 核心关注 | 会话生命周期 | 执行与运维 | 能力发现与协作 |
| 典型场景 | IDE 连接 agent runtime | 驱动外部 coding harness | 多 agent 系统互联 |

---

## 实际场景中的边界感

当你在问：

> "我的 IDE 要怎么连到 OpenClaw / 这个 agent 服务？"

你关心的是客户端和单个会话之间怎么说话，这属于 **ACP** 的问题；在 OpenClaw 里，Gateway 会把 ACP 请求转给某个 backend（比如 acpx）。

当你在问：

> "OpenClaw 实际上是怎么把 Codex / Claude Code 这种外部工具跑起来的？"

你在谈的是实现与运维，这属于 **acpx** 的范畴：ACL、cwd、mode、超时、进程管理这些事情，都不在"协议规范"里，而在具体 backend 如何落地。

当你在问：

> "我们公司里已经有好几个 agent 系统，怎么让它们互相发现和协作？"

你遇到的是 agent-to-agent 的互操作问题，这一层更接近 **A2A** 的设计目标，而不是 IDE ↔ agent 的 ACP 问题。

---

## 一个简单的心智模型

最后给一个可以"背下来"的心智模型：

- **ACP**：一条"从客户端打进来的热线电话"，负责"怎么打、怎么挂、怎么转接、怎么听录音"
- **acpx**：一位"接电话并真正去跑任务的值班工程师"，负责"怎么拉起服务、怎么配权限、怎么清理现场"
- **A2A**：一个"跨公司的业务协作标准"，负责"不同公司的机器人怎么打招呼、怎么对齐接口、怎么一起干项目"

只要记住：**ACP & acpx ≈ 单系统里的"会话线 + 实现层"，A2A ≈ 多系统之间的"互联语言"**，一般就不容易再把这三个名词混在一起了。

---

## 总结

| 协议/组件 | 解决的问题 | 一句话定位 |
|-----------|-----------|-----------|
| **ACP** | 客户端怎么连 agent | 会话层协议 |
| **acpx** | 协议怎么落地执行 | 实现层组件 |
| **A2A** | agent 之间怎么协作 | 互操作标准 |

三者分层协作，互不冲突：

- ACP 定义"怎么说"
- acpx 负责"怎么干"
- A2A 解决"怎么一起干"

---

## 参考资料

1. [OpenClaw Docs: ACP Agents](https://docs.openclaw.ai/tools/acp-agents)
2. [OpenClaw Docs: `acp` CLI 子命令](https://docs.openclaw.ai/cli/acp)
3. [A2A Protocol 官方网站](https://a2a-protocol.org/latest/)
4. [Agent2Agent (A2A) GitHub 仓库](https://github.com/a2aproject/A2A)
5. [Codelab: Getting Started with Agent2Agent (A2A) Protocol](https://codelabs.developers.google.com/intro-a2a-purchasing-concierge)