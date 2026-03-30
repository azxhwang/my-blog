# Harness Engineering：AI 时代软件开发的范式转移

> **摘要**：OpenAI 和 Anthropic 近期发布的三篇博客揭示了 AI 驱动软件开发的新范式——Harness Engineering。本文深入解析这一概念，对比三家实践，并探讨工程师角色如何从"写代码"转向"设计约束"。

---

## 一、为什么需要 Harness？

### 1.1 单 Agent 的局限性

在传统的 AI 辅助编程场景中，开发者通常采用"单 Agent"模式：

```
开发者提问 → Agent 生成代码 → 开发者审查 → 手动测试 → 迭代
```

这种模式存在几个根本性问题：

| 问题         | 表现                   | 后果         |
| ---------- | -------------------- | ---------- |
| **上下文限制**  | Agent 无法记住之前的会话内容    | 复杂任务无法连续完成 |
| **自我评估偏差** | Agent 倾向于高估自己代码的质量   | 过早宣布任务完成   |
| **缺乏反馈闭环** | Agent 无法自动验证代码是否真正工作 | 需要人工介入测试   |
| **架构熵增**   | 多次迭代后代码结构逐渐混乱        | 技术债快速积累    |

### 1.2 Harness 的定义

**Harness（约束框架）** 是一套结构化的工作流、工具和约束系统，用于：

- 约束 Agent 的行为边界
- 提供自动化的反馈循环
- 确保长期代码质量和架构一致性
- 桥接多个会话之间的状态和上下文

用 Martin Fowler 的话说：

> "Harness Engineering 是 AI 赋能软件开发的关键框架，包含上下文工程、架构约束和垃圾回收机制。"

---

## 二、三篇核心博客深度对比

2025 年底至 2026 年初，OpenAI 和 Anthropic 相继发布了三篇关于 Harness Engineering 的重磅文章：

| 文章                                                                                                                               | 发布方       | 时间      | 核心场景         |
| -------------------------------------------------------------------------------------------------------------------------------- | --------- | ------- | ------------ |
| [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Anthropic | 2025.11 | 长周期 Agent 开发 |
| [Harness Engineering](https://openai.com/index/harness-engineering/)                                                             | OpenAI    | 2026.02 | 百万行代码规模      |
| [Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)                   | Anthropic | 2026.03 | 全栈应用开发       |

### 2.1 Anthropic V1：双 Agent 模式（2025.11）

**核心架构：**

```
┌──────────────┐         ┌──────────────┐
│ Initializer  │  ──→    │   Coding     │
│    Agent     │         │    Agent     │
└──────────────┘         └──────────────┘
   • 环境设置                • 增量开发
   • 依赖安装                • 单功能提交
   • 服务启动                • 状态记录
```

**关键实践：**

1. **状态持久化**
   
   - `claude-progress.txt`：记录 Agent 行动和会话摘要
   - `feature_list.json`：结构化需求列表（JSON 优于 Markdown，防止误编辑）
   - Git 历史：作为会话间的上下文桥梁

2. **增量主义**
   
   - 每次会话只完成一个功能
   - 会话结束时代码必须可合并
   - 提交信息必须描述性清晰

3. **自动化测试**
   
   - 使用 Puppeteer MCP 进行浏览器自动化
   - 模拟真实用户行为进行端到端验证

**局限性：**

- 架构约束较弱
- 评估机制依赖单一 Agent 自我判断

---

### 2.2 OpenAI：规模化 Harness（2026.02）

**核心成果：**

- 5 个月内部实验
- **100 万行代码，零手写源码**
- 小团队通过 PR 和 CI 工作流引导 Agent

**核心架构：**

```
┌─────────────────────────────────────────────────────────┐
│                    Codex Agents Suite                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐          │
│  │ Code │ │ Test │ │ Obs. │ │ PR   │ │ CI   │          │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘          │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    约束系统                              │
│  • docs/ 目录：单一事实源                                │
│  • 依赖层规则：Types→Config→Repo→Service→Runtime→UI     │
│  • Linter + CI：机械执行架构边界                         │
│  • 垃圾回收 Agent：定期查找不一致和违规                  │
└─────────────────────────────────────────────────────────┘
```

**关键创新：**

1. **文档即事实源**
   
   - `docs/` 目录包含架构地图、执行计划、设计规范
   - 文档交叉链接并通过 CI 验证一致性

2. **架构约束机械化**
   
   ```
   依赖流向：Types → Config → Repo → Service → Runtime → UI
   
   规则：上层可依赖下层，下层不可依赖上层
   执行：结构测试 + Linter 自动验证
   ```

3. **熵减机制**
   
   - 专门的"垃圾回收"Agent 定期运行
   - 查找文档不一致和架构违规
   - 自动修复或报告问题

**核心洞见：**

> "Harness 将脚手架、反馈循环、文档和架构约束编码为机器可读的工件。"

---

### 2.3 Anthropic V2：生成器 - 评估器模式（2026.03）

**核心架构：**

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Planner    │ →  │  Generator   │ →  │  Evaluator   │
└──────────────┘    └──────────────┘    └──────────────┘
       ↑                                       │
       └────────────── 反馈循环 ───────────────┘
```

**关键创新：**

1. **生成器 - 评估器分离**
   
   - 灵感来自 GAN（生成对抗网络）
   - 解决模型"自我赞美偏差"
   - Generator 专注写代码，Evaluator 专注挑毛病

2. **主观质量量化**
   将难以衡量的"代码质量"转化为可评分标准：
   
   | 维度             | 权重  | 评估内容      |
   | -------------- | --- | --------- |
   | Design Quality | 30% | 设计美感、一致性  |
   | Originality    | 30% | 创意、非通用方案  |
   | Craft          | 20% | 代码工艺、细节处理 |
   | Functionality  | 20% | 功能完整性     |

3. **Sprint 合同**（V1）
   
   - Generator 和 Evaluator 在编码前约定"完成"定义
   - 避免后续对需求理解不一致

4. **Harness 演化**
   
   - V1（Opus 4.5）：使用 Sprint 分解 + 上下文重置
   - V2（Opus 4.6）：移除 Sprint，连续会话 + 自动压缩
   - **核心原则**：Harness 复杂度应随模型能力提升而降低

**性能对比：**

| 模式         | 时间     | 成本   | 结果            |
| ---------- | ------ | ---- | ------------- |
| Solo Run   | 20 min | $9   | 核心功能失败        |
| Harness V1 | 6 hr   | $200 | 功能完整，高质量      |
| Harness V2 | ~4 hr  | $124 | 功能完整，成本降低 38% |

**关键发现：**

- 默认 Evaluator 过于宽容，需要专门调优使其"挑剔"
- 显式提示 AI 功能集成，产品会更先进
- 分离"做事的 Agent"和"评判的 Agent"是质量控制的关键

---

## 三、工程师角色的范式转移

### 3.1 从"写代码"到"设计约束"

```
传统工程师                    Harness 时代工程师
    │                              │
    ▼                              ▼
┌──────────┐                  ┌──────────────┐
│ 写业务代码 │                  │ 设计约束系统  │
│ 手动调试   │       →          │ 定义反馈循环  │
│ 编写测试   │                  │ 构建验证工具  │
│ 代码审查   │                  │ 架构治理     │
└──────────┘                  └──────────────┘
```

### 3.2 价值转移矩阵

| 低价值（逐渐自动化） | 高价值（更需要人类） |
| ---------- | ---------- |
| 样板代码编写     | 问题定义和需求澄清  |
| 重复性调试      | 架构设计和约束定义  |
| 手动测试编写     | 评估标准设计     |
| 文档维护       | 工具和环境设计    |
| 代码审查（风格级）  | 复杂决策和权衡判断  |

### 3.3 新的核心能力

**1. 架构设计能力**

OpenAI 的依赖层设计不是 Agent 完成的，而是工程师预先定义的。如果架构设计错误：

- Agent 生成的代码会违反模块化原则
- 技术债会以更快的速度积累（因为生成速度更快）

**2. 约束系统设计能力**

```python
# 不再是写业务逻辑
def calculate_price(items):
    return sum(item.price for item in items)

# 而是写约束规则
@enforce_architecture
def validate_dependency(source: Module, target: Module):
    """确保依赖流向符合架构规则"""
    allowed_flows = {
        'UI': ['Runtime'],
        'Runtime': ['Service'],
        'Service': ['Repo', 'Config'],
        # ...
    }
    return target in allowed_flows.get(source.layer, [])
```

**3. 反馈循环设计能力**

Anthropic V2 的关键发现：评估器需要专门调优。默认模型太"友好"，无法有效发现问题。

```
好的反馈循环：
Generator 生成代码 → Evaluator 严格审查 → 发现问题 → 返回修复 → 重新验证

坏的反馈循环：
Generator 生成代码 → Evaluator 友好认可 → 问题遗漏 → 技术债积累
```

**4. 工具链建设能力**

让 Agent 能够有效工作的工具：

- MCP 服务器（浏览器自动化、数据库访问等）
- 自动化测试框架
- 可观测性系统（日志、指标、追踪）

---

## 四、关键结论与建议

### 4.1 核心结论

1. **Harness 是必需的，不是可选的**
   
   - 单 Agent 模式无法完成复杂、长周期任务
   - 结构化工作流显著提升代码质量和功能完整性

2. **Harness 需要与模型共同演进**
   
   - Anthropic 从 V1 到 V2 的简化证明：模型能力提升后，部分 scaffolding 成为冗余
   - 目标是找到实现预期结果的**最简 Harness**

3. **架构约束是规模化的关键**
   
   - OpenAI 百万行代码案例证明：没有机械化约束，架构会快速熵增
   - "垃圾回收"机制是对抗熵增的必要手段

4. **评估器需要专门设计**
   
   - 默认模型过于宽容
   - 分离生成和评估是质量控制的核心

### 4.2 实践建议

**对于新项目：**

1. 从零开始设计 Harness，而非事后 retrofit
2. 优先建设：文档结构、架构约束、自动化测试
3. 采用生成器 - 评估器模式，确保代码质量

**对于遗留项目：**

1. 逐步引入约束：先加 Linter，再加结构测试
2. 评估 retrofit 成本，某些情况下重构可能更经济
3. 优先在新增功能模块应用 Harness 模式

**对于工程师个人：**

1. **保持编码手感**：理解代码才能设计好的约束
2. **提升抽象层级**：从代码行转向系统设计
3. **学习新工具**：MCP、Agent SDK、自动化测试框架
4. **培养架构思维**：这是目前最难被替代的能力

### 4.3 适用场景边界

| 场景        | 适用度    | 说明                            |
| --------- | ------ | ----------------------------- |
| 新项目从零开始   | ✅ 高    | 可设计完整 Harness                 |
| 遗留系统改造    | ⚠️ 中   | Martin Fowler 指出可能难以 retrofit |
| 探索性/研究型代码 | ⚠️ 中   | 需求不清晰，难以定义约束                  |
| 高可靠性系统    | ⚠️ 待验证 | OpenAI 案例是 beta 产品            |

---

## 五、未来展望

### 5.1 技术趋势

1. **Harness 模板化**
   
   - 可能出现标准化的"黄金路径"服务模板
   - 内置 Linter、结构测试、上下文文档

2. **技术栈收敛**
   
   - 组织可能倾向于采用更易 Harness 的技术栈
   - "AI 友好度"成为技术选型新维度

3. **多 Agent 协作**
   
   - 专业化 Agent：测试 Agent、QA Agent、清理 Agent
   - 从单领域（代码）扩展到多领域（科研、金融）

### 5.2 工程师的机遇

Harness Engineering 不是要取代工程师，而是**重新定位工程师的价值**：

> "我们构建 Harness 是为了提供一致可靠的方式运行大规模 AI 工作负载，让团队专注于研究和产品开发，而非基础设施编排。"
> — Ryan Lopopolo, OpenAI

**未来的工程师：**

- 不是"写更少代码"，而是"写不同的代码"
- 不是"被 AI 取代"，而是"用 AI 放大能力"
- 不是"放弃技术深度"，而是"在更高层级发挥深度"

---

## 参考资源

1. [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) - Anthropic, Nov 2025
2. [Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/) - OpenAI, Feb 2026
3. [Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) - Anthropic, Mar 2026
4. [Harness Engineering - Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)

---
