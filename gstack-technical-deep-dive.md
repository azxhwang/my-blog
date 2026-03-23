# gstack 技术深度解析：AI 原生软件工程的架构革命

> 当 Garry Tan 在 60 天内产出 60 万行生产代码时，我们看到的不仅是效率提升，更是软件开发范式的根本转变。

## 引言：从"写代码"到"指挥 AI"

2025 年，AI 编程助手已经成为开发者的标配。但大多数人仍然停留在"让 AI 帮我写函数"的阶段。Y Combinator CEO Garry Tan 开源的 **gstack** 展示了一种全新的可能性：**一个人通过系统化的 AI 编排，可以替代整个软件工程团队**。

这不是简单的"用 AI 写更多代码"，而是重新定义了软件开发的**生产关系**。本文将从技术架构角度，深入解析 gstack 的设计哲学和实现细节。

---

## 一、核心架构：三层分离设计

gstack 的架构可以用一张图概括：

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 1: 角色层（Skills）                   │
│  /office-hours  /plan-eng-review  /qa  /ship  /cso...       │
│  └─ 28 个 Markdown 文件，定义 AI 角色的"剧本"                  │
├─────────────────────────────────────────────────────────────┤
│                    Layer 2: 编排层（Conductor）                │
│  conductor.json                                             │
│  └─ 多会话并行编排，流程自动化                                  │
├─────────────────────────────────────────────────────────────┤
│                    Layer 3: 执行层（Browser）                  │
│  browse/dist/browse (Bun 编译的二进制)                         │
│  └─ 持久化 Chromium 守护进程，提供亚秒级浏览器自动化              │
└─────────────────────────────────────────────────────────────┘
```

这种分层设计的精妙之处在于：**每一层都可以独立演进**。

### 1.1 角色层：Markdown 即代码

传统软件开发中，流程文档和实际执行是分离的。gstack 的创新在于将**流程本身编码为 Markdown**。

以 `/review` 技能为例，其 `SKILL.md` 不是简单的使用说明，而是包含：

```markdown
## Preamble
```bash
# 更新检查、会话追踪、遥测初始化...
```

## Workflow
1. 读取代码变更
2. 执行静态分析
3. 检查测试覆盖率
4. 验证 API 设计规范
5. 输出审查报告

## Completion Protocol
- DONE: 全部通过
- DONE_WITH_CONCERNS: 通过但有问题
- BLOCKED: 无法继续（需人工介入）
```

**技术洞察**：这实际上是一种**声明式编程**。开发者声明"审查应该做什么"，而不是写代码"如何审查"。Claude Code 解析这些指令后，自主决定具体执行步骤。

### 1.2 编排层：并行冲刺的秘密

`conductor.json` 允许定义多会话并行工作流：

```json
{
  "workspaces": [
    { "name": "design-review", "skill": "/plan-design-review" },
    { "name": "qa-testing", "skill": "/qa" },
    { "name": "feature-dev", "skill": "/implement" }
  ]
}
```

**实现细节**：每个 workspace 是独立的 Claude Code 进程，通过文件系统同步状态。这种设计避免了单进程的状态冲突，同时利用了现代操作系统的进程隔离特性。

### 1.3 执行层：持久化浏览器的技术突破

这是 gstack 最硬核的技术创新。让我们深入其实现：

#### 守护进程架构

```typescript
// browse/src/server.ts 核心逻辑
const server = Bun.serve({
  port: randomPort(), // 10000-60000 随机选择
  routes: {
    "/command": async (req) => {
      const { cmd, args, token } = await req.json();
      // Token 验证（Bearer UUID）
      // 命令分发到 BrowserManager
      // 返回纯文本结果
    }
  }
});

// BrowserManager 维护持久化 Chromium 实例
class BrowserManager {
  private browser: Browser;
  private refMap: Map<string, RefEntry>; // @e1, @e2 映射
  
  async execute(command: string, args: string[]) {
    // 100-200ms 响应时间
  }
}
```

#### 为什么不用 MCP？

gstack 明确拒绝了 MCP（Model Context Protocol）协议，选择简单的 HTTP + 纯文本：

| 方案 | 延迟 | 调试难度 | 复杂度 |
|------|------|----------|--------|
| MCP | 高（JSON Schema 开销） | 难 | 高 |
| HTTP + 纯文本 | 低 | 易（curl 即可） | 低 |

**架构决策**：在 AI 工具链中，**简单性优于完备性**。HTTP 的无状态特性使得故障排查变得 trivial——直接用 curl 就能复现问题。

---

## 二、Ref 系统：AI 操作网页的"指针"

让 AI "点击按钮"是浏览器自动化的核心挑战。传统方案是让 AI 生成 CSS 选择器：

```javascript
// 脆弱且容易出错
document.querySelector(".btn-primary:nth-child(3)")
```

gstack 的 Ref 系统是一种**间接寻址**机制：

### 2.1 技术实现

```typescript
// browse/src/snapshot.ts
async function generateRefs(page: Page): Promise<RefMap> {
  // 1. 获取无障碍树（ARIA Tree）
  const snapshot = await page.accessibility.snapshot();
  
  // 2. 遍历树，为每个可交互元素分配 Ref
  const refs = new Map<string, RefEntry>();
  let counter = 1;
  
  traverseTree(snapshot, (node) => {
    if (isInteractive(node)) {
      const ref = `@e${counter++}`;
      // 3. 创建 Playwright Locator（不修改 DOM）
      const locator = page.getByRole(node.role, { 
        name: node.name 
      }).nth(node.index);
      
      refs.set(ref, {
        role: node.role,
        name: node.name,
        locator // 存储 Locator 实例
      });
    }
  });
  
  return refs;
}
```

### 2.2 为什么用 Locator 而非 DOM 注入？

| 方案 | CSP 兼容 | 框架兼容 | Shadow DOM |
|------|----------|----------|------------|
| DOM 注入（data-ref） | ❌ 容易被拦截 | ❌ React 会清理 | ❌ 无法穿透 |
| Playwright Locator | ✅ 外部查询 | ✅ 无 DOM 修改 | ✅ 支持 |

**关键设计**：Locator 基于 Chromium 内部的**无障碍树**，而非 DOM 树。这避开了前端框架的调和机制（Reconciliation），也不会触发 CSP 策略。

### 2.3 Staleness 检测

SPA（单页应用）的 DOM 变化不会触发页面导航，导致 Ref 失效。gstack 的解决方案：

```typescript
async function resolveRef(refId: string): Promise<Locator> {
  const entry = refMap.get(refId);
  // 快速检查：元素是否还存在？
  const count = await entry.locator.count();
  
  if (count === 0) {
    throw new Error(
      `Ref ${refId} is stale — element no longer exists. ` +
      `Run 'snapshot' to get fresh refs.`
    );
  }
  
  return entry.locator;
}
```

**性能**：`count()` 检查只需 ~5ms，相比 Playwright 默认的 30 秒超时，实现了**快速失败**（Fail Fast）。

---

## 三、Cookie 导入：安全与便利的平衡

测试需要登录态，但让 AI 知道你的密码显然不合理。gstack 的解决方案是**从本地浏览器导入 Cookies**。

### 3.1 技术实现

```typescript
// browse/src/cookies.ts
async function importCookiesFromBrowser(
  browserName: 'Chrome' | 'Arc' | 'Brave'
): Promise<Cookie[]> {
  // 1. 找到浏览器 Cookie 数据库路径
  const dbPath = getCookieDBPath(browserName);
  
  // 2. 复制到临时文件（避免 SQLite 锁冲突）
  const tempPath = `/tmp/gstack-cookies-${Date.now()}.db`;
  await Bun.write(tempPath, await Bun.file(dbPath).arrayBuffer());
  
  // 3. 从 macOS Keychain 获取加密密钥
  const keychainPassword = await getKeychainPassword(browserName);
  const aesKey = deriveKey(keychainPassword); // PBKDF2 + AES-128-CBC
  
  // 4. 解密 Cookie 值
  const db = new Database(tempPath, { readonly: true });
  const encryptedCookies = db.query("SELECT * FROM cookies").all();
  
  return encryptedCookies.map(cookie => ({
    ...cookie,
    value: decrypt(cookie.value, aesKey)
  }));
}
```

### 3.2 安全设计

1. **只读访问**：从不修改原始浏览器数据库
2. **内存解密**：Cookie 值只在内存中解密，不落盘
3. **Keychain 授权**：首次访问触发 macOS 系统对话框，需用户明确授权
4. **会话缓存**：密钥只缓存在内存中，服务停止即清除

---

## 四、技能模板系统：文档即代码

gstack 如何解决"文档与代码不同步"的问题？

### 4.1 模板引擎

```
SKILL.md.tmpl          # 人类编写的模板（包含占位符）
    ↓
gen-skill-docs.ts      # 构建时从源码提取元数据
    ↓
SKILL.md               # 生成的最终文档（提交到仓库）
```

模板示例：

```markdown
## Command Reference
{{COMMAND_REFERENCE}}  <!-- 从 commands.ts 自动生成 -->

## Snapshot Flags
{{SNAPSHOT_FLAGS}}     <!-- 从 snapshot.ts 自动生成 -->
```

### 4.2 三层测试体系

| 层级 | 内容 | 成本 | 速度 |
|------|------|------|------|
| Tier 1 | 静态验证：解析所有 `$B` 命令，验证语法 | 免费 | <2s |
| Tier 2 | E2E：真实 Claude 会话运行每个技能 | ~$3.85 | ~20min |
| Tier 3 | LLM 评估：Sonnet 评分文档质量 | ~$0.15 | ~30s |

**设计哲学**：95% 的问题用免费测试捕获，LLM 只用于判断性任务。

---

## 五、"Boil the Lake"：AI 时代的工程哲学

gstack 的文档中反复出现"Boil the Lake"（煮沸湖泊）原则，这是其核心理念：

> "当 AI 让完整性的边际成本趋近于零时，永远选择完整的方案。"

### 5.1 成本对比表

| 任务类型 | 人工团队 | CC + gstack | 压缩比 |
|----------|----------|-------------|--------|
| 脚手架代码 | 2 天 | 15 分钟 | ~100x |
| 测试编写 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |

### 5.2 技术实现：Completeness Protocol

每个技能结束时必须报告状态：

```typescript
type CompletionStatus = 
  | 'DONE'              // 全部完成
  | 'DONE_WITH_CONCERNS' // 完成但有问题
  | 'BLOCKED'           // 无法继续
  | 'NEEDS_CONTEXT';    // 需要更多信息
```

配合**升级规则**：
- 尝试 3 次未成功 → 升级（Escalate）
- 安全敏感变更不确定 → 升级
- 超出验证能力范围 → 升级

---

## 六、实际应用：如何验证产品想法

让我们走一遍用 gstack 验证产品的完整流程：

### 6.1 从零开始

```bash
mkdir my-idea && cd my-idea
claude
```

```
User: /office-hours

Claude: 让我们通过 6 个 YC 式问题来定义你的产品...

1. 你在解决什么问题？
2. 谁有这个问题？
3. 为什么现在做？
4. 你的独特洞察是什么？
5. 如何验证？
6. 可能失败的原因？

[经过多轮对话后生成 Product Definition Document]
```

### 6.2 并行审查

```
User: /autoplan

Claude: 我将并行运行三个审查...

[CEO Review]     挑战需求范围...
[Eng Review]     绘制架构图...
[Design Review]  评估设计可行性...

生成综合报告：
- 建议范围：保持当前（不要扩展）
- 技术风险：中等（依赖第三方 API）
- 设计评分：7/10（需要优化移动端）
```

### 6.3 快速原型

```
User: /design-consultation

Claude: 基于审查结果，我将生成完整设计系统...

- Color palette
- Typography scale
- Component library
- Responsive breakpoints
```

### 6.4 持续验证

```
User: /qa

Claude: 启动浏览器，测试用户流程...

$B goto http://localhost:3000
$B snapshot -i
$B click @e3  // 点击"开始"按钮
$B fill @e5 "test@example.com"
...

发现 Bug：表单验证失败时无错误提示
自动修复：添加错误状态显示
生成回归测试：test/form-validation.spec.ts
```

---

## 七、局限性与未来方向

### 7.1 当前局限

1. **平台限制**：Cookie 导入仅支持 macOS（依赖 Keychain）
2. **iframe 支持**：Ref 系统尚未支持跨 iframe 操作
3. **多用户**：设计为单用户本地使用，无多租户支持
4. **成本**：大规模使用需要预算（Claude API + E2E 测试）

### 7.2 可能的演进

1. **云端浏览器池**：支持并行多浏览器会话
2. **技能市场**：社区共享自定义技能
3. **IDE 集成**：VS Code 插件替代命令行
4. **多模态**：支持截图理解、视频测试

---

## 结语：范式转移的征兆

gstack 不仅仅是一个工具集，它代表了软件开发从**"手工劳动"向"知识工作"**的转移。

在传统模式中，开发者是**工匠**——亲手编写每一行代码。在 gstack 模式中，开发者是**指挥家**——定义流程、做出关键决策、审查结果。

这种转变的底层逻辑是：**AI 的边际成本趋近于零，使得"做完整的事"比"做 shortcuts"更经济**。

当 Garry Tan 说"我不认为自 12 月以来我写过一行代码"时，他指的不是"我不工作了"，而是**工作的本质变了**——从编码转向思考、从实现转向验证、从个人产出转向系统编排。

这或许就是软件工程的未来。

---

## 参考资源

- **GitHub**: https://github.com/garrytan/gstack
- **核心文档**: [SKILL.md](https://github.com/garrytan/gstack/blob/main/SKILL.md)、[ARCHITECTURE.md](https://github.com/garrytan/gstack/blob/main/ARCHITECTURE.md)
- **相关阅读**: [Boil the Ocean](https://garryslist.org/posts/boil-the-ocean) (Garry Tan 的博客)

---

*本文基于 gstack v1.1.0 版本分析，架构可能随版本迭代而变化。*
