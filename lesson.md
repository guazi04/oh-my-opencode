# Oh My OpenCode 项目学习指南

> 面向 Python 专家的 TypeScript 项目深度解读。从零精通 Oh My OpenCode 的架构、代码模式和核心模块。

---

## 目录

- [前言：TS vs Python 核心概念对照](#前言ts-vs-python-核心概念对照)
- [项目架构总览](#项目架构总览)
- [第 1 课：配置系统 — src/config/schema.ts](#第-1-课配置系统--srcconfigschemats)
- [第 2 课：插件入口 — src/index.ts](#第-2-课插件入口--srcindexts)
- [第 3 课：工具定义 — src/tools/grep/](#第-3-课工具定义--srctoolsgrep)
- [第 4 课：Hook 系统 — src/hooks/](#第-4-课hook-系统--srchooks)
- [第 5 课：Agent 定义 — src/agents/](#第-5-课agent-定义--srcagents)
- [第 6 课：任务委派 — src/tools/delegate-task/](#第-6-课任务委派--srctoolsdelegate-task)
- [第 7 课：后台任务调度 — BackgroundManager](#第-7-课后台任务调度--backgroundmanager)
- [第 8 课：技能系统 — Skill Loader](#第-8-课技能系统--skill-loader)
- [第 9 课：MCP 协议集成](#第-9-课mcp-协议集成)
- [总结与速查表](#总结与速查表)

---

## 前言：TS vs Python 核心概念对照

| Python | TypeScript | 说明 |
|--------|-----------|------|
| `dict` | `Record<string, T>` 或 `{ key: type }` | TS 用接口/类型定义 dict 结构 |
| `typing.Optional[str]` | `string \| undefined` | 联合类型 |
| `pydantic.BaseModel` | `zod.object({...})` | 本项目用 Zod 做运行时校验，类似 Pydantic |
| `async def` / `await` | `async function` / `await` | 几乎一样 |
| `decorator` | 无直接等价，用高阶函数 | TS 的装饰器不常用 |
| `pip install` | `bun install` (或 `npm install`) | 本项目用 Bun 运行时 |
| `pytest` | `bun test` | Bun 内置测试框架 |
| `from x import y` | `import { y } from './x'` | ESM 模块系统 |
| `**kwargs` | 解构 `{ ...rest }` | 展开运算符 |
| `list comprehension` | `.map()`, `.filter()` | 函数式方法链 |

---

## 项目架构总览

```
┌─────────────────────────────────────────────┐
│  src/index.ts  —— 插件入口（主函数）          │
├─────────────────────────────────────────────┤
│  src/agents/   —— 8个 AI 智能体定义          │
│  (Sisyphus主控, Oracle调试, Librarian搜索...)  │
├─────────────────────────────────────────────┤
│  src/tools/    —— 15+ 工具（grep, LSP, AST） │
│  src/hooks/    —— 25+ 钩子（消息变换管道）     │
├─────────────────────────────────────────────┤
│  src/features/ —— 复杂功能模块               │
│  (后台Agent管理, MCP协议, 技能加载器...)       │
├─────────────────────────────────────────────┤
│  src/shared/   —— 工具函数库                 │
│  src/config/   —— Zod 配置 schema            │
└─────────────────────────────────────────────┘
```

各层的 Python 类比：

- `index.ts` = Flask/FastAPI 的 `app.py`，注册所有路由（工具+钩子）
- `agents/` = 策略模式的各个策略类
- `tools/` = API 端点/命令
- `hooks/` = 中间件 (middleware)
- `features/` = 业务逻辑服务层
- `config/schema.ts` = Pydantic models

---

## 第 1 课：配置系统 — `src/config/schema.ts`

这个文件是整个项目的"说明书"。读懂它，就知道项目能做什么。

### Zod ≈ Pydantic

```python
# Python (Pydantic)                      # TypeScript (Zod)
class Permission(str, Enum):             # const PermissionValue = z.enum(
    ask = "ask"                          #   ["ask", "allow", "deny"]
    allow = "allow"                      # )
    deny = "deny"                        #

class AgentPermission(BaseModel):        # const AgentPermissionSchema = z.object({
    edit: Optional[Permission] = None    #   edit: PermissionValue.optional(),
    bash: Optional[Permission] = None    #   bash: BashPermission.optional(),
    ...                                  # })
```

关键对照：

- `z.object({...})` = `BaseModel` 定义结构
- `.optional()` = `Optional[X] = None`
- `z.enum([...])` = `Enum`
- `z.string()`, `z.number()`, `z.boolean()` = `str`, `int`, `bool`
- `z.array(...)` = `list[...]`
- `z.record(z.string(), X)` = `dict[str, X]`
- `z.union([A, B])` = `Union[A, B]`

### 根配置结构

文件从小到大构建 schema，最终汇聚到第 366 行的 `OhMyOpenCodeConfigSchema`：

```
OhMyOpenCodeConfig（项目总开关）
├── disabled_mcps / disabled_agents / disabled_hooks  → 禁用列表
├── agents: AgentOverridesSchema        → 覆盖各 Agent 的模型/prompt/权限
├── categories: CategoriesConfigSchema  → 任务分类配置（如 ultrabrain, quick）
├── skills: SkillsConfigSchema          → 技能系统配置
├── background_task                     → 后台任务并发数
├── ralph_loop                          → 迭代循环功能
├── tmux                                → Tmux 多窗格布局
├── sisyphus                            → Sisyphus Agent 专属配置
├── experimental                        → 实验性功能（如动态上下文裁剪）
└── ...其他子模块配置
```

### 5 大子系统

| 行号 | Schema | 子系统 | 作用 |
|------|--------|--------|------|
| 19-29 | `BuiltinAgentNameSchema` | **Agent系统** | 9个内置智能体 |
| 56-91 | `HookNameSchema` | **Hook系统** | 25个消息中间件 |
| 31-36 | `BuiltinSkillNameSchema` | **技能系统** | 可插拔能力（playwright等） |
| 187-195 | `BuiltinCategoryNameSchema` | **分类系统** | 7类任务（ultrabrain等） |
| 296-302 | `BackgroundTaskConfigSchema` | **后台任务** | 并发调度 |

### 类型导出

```typescript
export type OhMyOpenCodeConfig = z.infer<typeof OhMyOpenCodeConfigSchema>
```

`z.infer<typeof X>` 是 Zod 的精髓——**一份 schema，同时得到运行时校验和编译时类型检查**。Python 的 Pydantic 天然做到这一点，TS 需要这个额外步骤。

**要点**：这个文件没有任何逻辑，纯粹是数据结构定义。它是整个项目的"契约"——所有模块都依赖这些类型。

---

## 第 2 课：插件入口 — `src/index.ts`

700 行，整个项目的"主函数"。结构非常清晰——就是一个大工厂函数。

### Python 等价

```python
def create_plugin(ctx) -> dict:
    # 第1步：读配置
    config = load_config(ctx.directory)

    # 第2步：按配置创建各种中间件（hooks）
    hooks = []
    if "comment-checker" not in config.disabled_hooks:
        hooks.append(create_comment_checker())
    # ... 25+ 个 hook

    # 第3步：创建工具（tools）
    tools = {**builtin_tools, "delegate_task": ..., "skill": ...}

    # 第4步：返回插件接口
    return {
        "tool": tools,
        "chat.message": handler,        # 消息到来时
        "tool.execute.before": handler,  # 工具执行前
        "tool.execute.after": handler,   # 工具执行后
        "event": handler,               # 全局事件
        "config": config_handler,
    }
```

### 文件的 4 个区域

#### 区域 1：导入（第 1-84 行）

从各个模块导入工厂函数。注意命名规律：

- `createXxxHook` → 钩子工厂
- `createXxxTool` → 工具工厂
- 类名如 `BackgroundManager`, `SkillMcpManager` → 有状态的管理器

#### 区域 2：初始化（第 86-378 行）

核心模式是**条件创建**：

```typescript
const isHookEnabled = (hookName: HookName) => !disabledHooks.has(hookName);

const commentChecker = isHookEnabled("comment-checker")
  ? createCommentCheckerHooks(pluginConfig.comment_checker)
  : null;
```

Python 等价：

```python
comment_checker = (
    create_comment_checker(config.comment_checker)
    if "comment-checker" not in disabled_hooks
    else None
)
```

这个模式重复了 25+ 次——每个 hook 一次。

#### 区域 3：工具与技能组装（第 294-377 行）

技能加载的流程（第 341-354 行）：

```typescript
// 并行加载 4 个来源的技能
const [userSkills, globalSkills, projectSkills, opencodeProjectSkills] =
  await Promise.all([...]);

// 合并成最终技能列表
const mergedSkills = mergeSkills(builtin, config, user, global, project, opencode);
```

Python 等价：

```python
user, global_, project, opencode = await asyncio.gather(
    discover_user_skills(),
    discover_global_skills(),
    discover_project_skills(),
    discover_opencode_project_skills(),
)
merged = merge_skills(builtin, config, user, global_, project, opencode)
```

#### 区域 4：返回插件接口（第 385-698 行）

定义了 5 个钩子点：

```
返回的插件对象
├── tool: { ... }                          # 注册所有工具
├── "chat.message": async (input, output)  # 用户发消息时触发
├── "tool.execute.before": async (...)     # 工具执行前拦截
├── "tool.execute.after": async (...)      # 工具执行后拦截
├── "event": async (input)                 # 全局事件处理
└── config: configHandler                  # 配置变更处理
```

### 关键设计模式

**责任链模式**（第 597-675 行的 `tool.execute.before`）：

```typescript
"tool.execute.before": async (input, output) => {
  await claudeCodeHooks["tool.execute.before"](input, output);
  await nonInteractiveEnv?.["tool.execute.before"](input, output);
  await commentChecker?.["tool.execute.before"](input, output);
  // ... 依次传递
}
```

每个 hook 拿到同一个 `(input, output)`，可以修改 `output` 来影响后续行为。等同于 Django/FastAPI 的中间件链。

**可选链 `?.`**：

```typescript
await xxx?.["tool.execute.before"]?.(input, output)
// Python 等价：
# if xxx is not None and hasattr(xxx, "tool_execute_before"):
#     await xxx.tool_execute_before(input, output)
```

### 数据流全貌

```
用户输入 ──→ chat.message（25个hook处理）
              │
              ▼
          AI 决定调用工具
              │
              ▼
         tool.execute.before（10个hook拦截）
              │
              ▼
         执行 tool（grep/delegate_task/skill...）
              │
              ▼
         tool.execute.after（12个hook后处理）
              │
              ▼
          返回结果给 AI
```

---

## 第 3 课：工具定义 — `src/tools/grep/`

项目中最好的教学案例——4 个文件，各司其职，每个都很短。

### 文件结构

```
src/tools/grep/
├── tools.ts    →  工具定义（对外接口）    40 行
├── types.ts    →  类型定义              39 行
├── cli.ts      →  底层执行逻辑          229 行
└── utils.ts    →  输出格式化            53 行
```

### `types.ts` — 数据结构

```typescript
export interface GrepMatch {
  file: string
  line: number
  column?: number
  text: string
}

export interface GrepResult {
  matches: GrepMatch[]
  totalMatches: number
  filesSearched: number
  truncated: boolean
  error?: string
}
```

Python 等价：

```python
@dataclass
class GrepMatch:
    file: str
    line: int
    column: int | None = None
    text: str = ""

@dataclass
class GrepResult:
    matches: list[GrepMatch]
    total_matches: int
    files_searched: int
    truncated: bool
    error: str | None = None
```

TS 的 `interface` 只在编译时存在，运行时消失。类似 Python 的 `TypedDict`。

### `tools.ts` — 工具定义（核心 40 行）

```typescript
export const grep: ToolDefinition = tool({
  description: "...",
  args: {
    pattern:  tool.schema.string().describe("..."),
    include:  tool.schema.string().optional().describe("..."),
    path:     tool.schema.string().optional().describe("..."),
  },
  execute: async (args) => {
    const result = await runRg({ pattern: args.pattern, ... })
    return formatGrepResult(result)
  },
})
```

Python 等价：

```python
@tool(description="Fast content search tool...")
async def grep(
    pattern: Annotated[str, "The regex pattern to search for"],
    include: Annotated[str | None, "File pattern to include"] = None,
    path: Annotated[str | None, "The directory to search in"] = None,
) -> str:
    result = await run_rg(pattern=pattern, globs=[include], paths=[path])
    return format_grep_result(result)
```

`tool()` 是一个工厂函数，接受 `{description, args, execute}` 返回一个 `ToolDefinition` 对象。

### `cli.ts` — 超时控制（TS 异步经典模式）

```typescript
const timeoutPromise = new Promise<never>((_, reject) => {
  const id = setTimeout(() => {
    proc.kill()
    reject(new Error(`Search timeout after ${timeout}ms`))
  }, timeout)
  proc.exited.then(() => clearTimeout(id))
})

const stdout = await Promise.race([
  new Response(proc.stdout).text(),
  timeoutPromise
])
```

`Promise.race` 同时运行两个 Promise，谁先完成就用谁的结果。Python 等价：

```python
async def run_with_timeout(coro, timeout):
    return await asyncio.wait_for(coro, timeout=timeout)
```

### `utils.ts` — 按文件分组

```typescript
const byFile = new Map<string, GrepMatch[]>()
for (const match of result.matches) {
  const existing = byFile.get(match.file) || []
  existing.push(match)
  byFile.set(match.file, existing)
}
```

Python 等价：

```python
from collections import defaultdict
by_file = defaultdict(list)
for match in result.matches:
    by_file[match.file].append(match)
```

---

## 第 4 课：Hook 系统 — `src/hooks/`

### 核心概念：Hook = Python 中间件

```python
# Django 中间件                           # TS Hook
class CommentCheckerMiddleware:           # createCommentCheckerHooks()
    def process_request(self, req):        #   "tool.execute.before": (input, output) => {}
    def process_response(self, req, res):  #   "tool.execute.after": (input, output) => {}
```

### Hook 的统一结构

```typescript
export function createXxxHook(ctx, config?) {
  // 闭包状态（等价于 Python 类的 __init__）
  const someState = new Map()

  return {
    "chat.message": async (input, output) => { ... },
    "tool.execute.before": async (input, output) => { ... },
    "tool.execute.after": async (input, output) => { ... },
    event: async (input) => { ... },
  }
}
```

Python 等价：

```python
def create_xxx_hook(ctx, config=None):
    some_state = {}

    async def on_chat_message(input, output): ...
    async def on_tool_before(input, output): ...
    async def on_tool_after(input, output): ...

    return {
        "chat.message": on_chat_message,
        "tool.execute.before": on_tool_before,
        "tool.execute.after": on_tool_after,
    }
```

用**闭包+工厂函数**代替 class 来保持状态。

### 3 个 Hook 实例

#### 1. `comment-checker` — before/after 协作

检测 AI 写的代码里是否有占位注释。

```
tool.execute.before ──→ 记录调用信息存入 pendingCalls Map
                        ↓
Write/Edit 实际执行
                        ↓
tool.execute.after  ──→ 从 pendingCalls 取出 → 调用 CLI 检查 → 追加警告到 output
```

before 存数据，after 取数据，用 `callID` 关联同一次工具调用。

#### 2. `tool-output-truncator` — 最简单的 Hook

如果工具输出太长（>50k token），截断它：

```typescript
"tool.execute.after": async (input, output) => {
  if (!TRUNCATABLE_TOOLS.includes(input.tool)) return
  const { result, truncated } = await truncator.truncate(output.output, { targetMaxTokens })
  if (truncated) {
    output.output = result  // ← 直接修改 output
  }
}
```

通过直接修改 output 对象来影响行为——和 Python 中间件修改 `response` 完全一样。

#### 3. `keyword-detector` — 监听 chat.message

检测用户消息中的关键词（如"ultrawork"），自动切换模式：

```typescript
"chat.message": async (input, output) => {
  let detectedKeywords = detectKeywordsWithType(cleanText, currentAgent)
  if (hasUltrawork) {
    output.message.variant = "max"  // ← 切换到高性能模式
  }
}
```

### Hook 生命周期全景图

```
用户发消息
  ├─→ chat.message hooks
  ▼
AI 决定调用工具
  ├─→ tool.execute.before hooks（可修改参数）
  ▼
工具执行
  ├─→ tool.execute.after hooks（可修改输出）
  ▼
结果返回给 AI
  ├─→ event hooks（处理生命周期事件）
```

---

## 第 5 课：Agent 定义 — `src/agents/`

### 核心认知：Agent 不是"类"，是"配置对象"

Agent 没有复杂的行为逻辑。它就是一个**配置包**——告诉 OpenCode SDK："用这个模型、这个 prompt、这些工具限制去运行"。

### 3 层结构

```
types.ts   →  Agent 的类型/接口
oracle.ts  →  一个具体 Agent 的定义
utils.ts   →  Agent 工厂：组装所有 Agent，处理覆盖/回退
```

### `types.ts` — Agent 的"蓝图"

```typescript
export type AgentFactory = (model: string) => AgentConfig
export type AgentCategory = "exploration" | "specialist" | "advisor" | "utility"
export type AgentCost = "FREE" | "CHEAP" | "EXPENSIVE"

export interface AgentPromptMetadata {
  category: AgentCategory
  cost: AgentCost
  triggers: DelegationTrigger[]   // 什么情况下触发委派
  useWhen?: string[]
  avoidWhen?: string[]
}
```

`AgentPromptMetadata` 告诉 Sisyphus（主控）什么时候应该把任务委派给这个 Agent。

### `oracle.ts` — 一个完整的 Agent

两件事：声明元数据 + 工厂函数。

```typescript
export const ORACLE_PROMPT_METADATA: AgentPromptMetadata = {
  category: "advisor",
  cost: "EXPENSIVE",
  triggers: [
    { domain: "Architecture decisions", trigger: "Multi-system tradeoffs" },
    { domain: "Hard debugging", trigger: "After 2+ failed fix attempts" },
  ],
}

export function createOracleAgent(model: string): AgentConfig {
  const restrictions = createAgentToolRestrictions(["write", "edit", "task", "delegate_task"])
  const base = {
    description: "Read-only consultation agent...",
    mode: "subagent",
    model,
    temperature: 0.1,
    ...restrictions,
    prompt: ORACLE_SYSTEM_PROMPT,
  }
  if (isGptModel(model)) {
    return { ...base, reasoningEffort: "medium" }
  }
  return { ...base, thinking: { type: "enabled", budgetTokens: 32000 } }
}
```

Python 等价：

```python
def create_oracle_agent(model: str) -> AgentConfig:
    restrictions = create_tool_restrictions(["write", "edit", "task", "delegate_task"])
    base = AgentConfig(
        description="Read-only consultation agent...",
        mode="subagent", model=model, temperature=0.1,
        **restrictions, prompt=ORACLE_SYSTEM_PROMPT,
    )
    if is_gpt_model(model):
        return base.copy(update={"reasoningEffort": "medium"})
    return base.copy(update={"thinking": {"type": "enabled", "budgetTokens": 32000}})
```

`{ ...base, key: val }` 展开语法——复制并覆盖，等价于 Pydantic 的 `model.copy(update={...})`。

### `utils.ts` — Agent 总装车间

`createBuiltinAgents()` 的流程：

```
遍历每个 Agent
  ├─ 跳过被禁用的
  ├─ 解析模型（支持回退链）
  ├─ 构建基础配置：buildAgent(source, model)
  ├─ 应用分类覆盖：applyCategoryOverride()
  └─ 应用用户覆盖：mergeAgentConfig()
```

配置优先级（从低到高）：

```
工厂默认值 → 分类配置 → 模型回退链 → 用户覆盖
```

**类型守卫** `source is AgentFactory`：

```typescript
function isFactory(source: AgentSource): source is AgentFactory {
  return typeof source === "function"
}
if (isFactory(source)) {
  source(model)  // TS 在 if 内自动收窄类型
}
```

类似 Python 3.10+ 的 `TypeGuard`。

---

## 第 6 课：任务委派 — `src/tools/delegate-task/`

整个项目**最复杂的单个文件**（1127 行），也是核心引擎。

### 功能

Sisyphus（主控 Agent）通过这个工具把任务分配给子 Agent 执行。类比：项目经理调用 `delegate_task(category="visual-engineering")` 把活儿分出去。

### 3 条执行路径

```
execute()
  │
  ├── 路径1：session_id 存在 → 继续已有会话
  ├── 路径2：category 指定   → 通过分类创建任务（用 sisyphus-junior）
  └── 路径3：subagent_type   → 直接指定 Agent（如 oracle）
```

### 同步执行的核心模式

```typescript
// 1. 创建子会话
const createResult = await client.session.create({
  body: { parentID: ctx.sessionID, title: `Task: ${args.description}` },
})

// 2. 发送 prompt 给子 Agent
await client.session.prompt({
  path: { id: sessionID },
  body: {
    agent: agentToUse,
    system: systemContent,
    tools: { task: false, delegate_task: false },
    parts: [{ type: "text", text: args.prompt }],
  },
})

// 3. 轮询等待完成（稳定性检测）
while (Date.now() - pollStart < MAX_POLL_TIME_MS) {
  await new Promise(resolve => setTimeout(resolve, POLL_INTERVAL_MS))

  const currentMsgCount = msgs.length
  if (currentMsgCount === lastMsgCount) {
    stablePolls++
    if (stablePolls >= STABILITY_POLLS_REQUIRED) break
  } else {
    stablePolls = 0
    lastMsgCount = currentMsgCount
  }
}

// 4. 提取最后一条 assistant 消息返回
```

Python 等价：

```python
# 1. 创建子会话
session = await client.session.create(parent_id=ctx.session_id)

# 2. 发送 prompt
await client.session.prompt(session.id, agent=agent, prompt=prompt)

# 3. 稳定性检测
stable_polls = 0
while time.time() - start < MAX_POLL_TIME:
    await asyncio.sleep(POLL_INTERVAL)
    msgs = await client.session.messages(session.id)
    if len(msgs) == last_count:
        stable_polls += 1
        if stable_polls >= REQUIRED:
            break
    else:
        stable_polls = 0
        last_count = len(msgs)
```

**稳定性检测**——不是简单等 `status == "idle"`，而是要求消息数量连续多次不变，避免"暂停思考"被误判为完成。

### 关键设计

| 决策 | 原因 |
|------|------|
| `category` 和 `subagent_type` 互斥 | 分类走 junior，直接指定走具体 Agent |
| 不稳定模型强制后台 | Gemini 等可能卡住，后台可监控 |
| 子 Agent 禁用 `task`/`delegate_task` | 防止递归委派 |
| 技能注入到 `system` | 让子 Agent 获得特定能力 |

### 条件展开模式

```typescript
{
  ...(categoryModel ? { model: categoryModel } : {}),
  ...(categoryModel?.variant ? { variant: categoryModel.variant } : {}),
}
```

Python 等价：

```python
body = {}
if category_model:
    body["model"] = category_model
if category_model and category_model.get("variant"):
    body["variant"] = category_model["variant"]
```

---

## 第 7 课：后台任务调度 — BackgroundManager

整个项目中**唯一真正用到 `class` 的核心模块**（1419 行）。

### Python 类比

```python
class BackgroundManager:
    """管理多个 AI Agent 并行执行后台任务"""

    def __init__(self, client, config):
        self.tasks: dict[str, BackgroundTask] = {}
        self.notifications: dict[str, list] = {}
        self.pending_by_parent: dict[str, set] = {}
        self.concurrency_manager = ConcurrencyManager()
        self.queues_by_key: dict[str, list] = {}

    async def launch(self, input) -> BackgroundTask: ...
    async def resume(self, input) -> BackgroundTask: ...
    def handle_event(self, event): ...
    def shutdown(self): ...
```

### 核心架构图

```
                    launch(input)
                        │
                        ▼
              ┌─────────────────┐
              │   Task Queue    │  ← 按 provider/model 分组排队
              └────────┬────────┘
                       │
            concurrencyManager.acquire(key)
                       │
                       ▼
              ┌─────────────────┐
              │   startTask()   │  ← 创建子会话，发送 prompt
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Polling Loop   │  ← 每 2 秒轮询
              │  三重守卫:       │
              │  1. session idle│
              │  2. 有输出?      │
              │  3. todo完成?    │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ tryCompleteTask │  ← 释放并发槽 + 通知父会话
              └─────────────────┘
```

### 5 个核心方法

#### `launch()` — 启动任务

```typescript
async launch(input): Promise<BackgroundTask> {
  const task = { id: `bg_${crypto.randomUUID().slice(0, 8)}`, status: "pending", ... }
  this.tasks.set(task.id, task)

  const key = this.getConcurrencyKeyFromInput(input)  // 如 "anthropic/claude-sonnet"
  queue.push({ task, input })

  this.processKey(key)  // fire-and-forget，不 await！
  return task
}
```

**fire-and-forget**——调用 async 函数但不 await。Python 等价：`asyncio.create_task(self.process_key(key))`

#### `processKey()` — 队列消费

```typescript
private async processKey(key) {
  if (this.processingKeys.has(key)) return  // 手动互斥锁
  this.processingKeys.add(key)
  try {
    while (queue.length > 0) {
      await this.concurrencyManager.acquire(key)
      await this.startTask(item)
      queue.shift()
    }
  } finally {
    this.processingKeys.delete(key)
  }
}
```

`processingKeys` 是手动互斥锁——JS/TS 单线程，不需要真正的锁，只需 Set 做标记。

#### `startTask()` — 实际启动

创建子会话 → 更新状态为 running → 启动轮询 → fire-and-forget 发送 prompt。

#### `pollRunningTasks()` — 轮询检测完成

三重守卫确保不误判：

```
轮询循环（每 2 秒）
├─ pruneStaleTasksAndNotifications()  → 清理超时任务（30分钟）
├─ checkAndInterruptStaleTasks()      → 中止卡死任务（3分钟无活动）
└─ 对每个 running 任务：
    ├─ 守卫1：session.status == "idle"？
    ├─ 守卫2：validateSessionHasOutput()
    ├─ 守卫3：checkSessionTodos()
    └─ 全部通过 → tryCompleteTask()
```

#### `shutdown()` — 优雅关闭

停止轮询 → 中止所有运行中的会话 → 释放并发槽 → 清空状态 → 注销信号处理。

### 通知机制

```typescript
await this.client.session.prompt({
  path: { id: task.parentSessionID },
  body: {
    noReply: !allComplete,  // 单个完成静默，全部完成才触发 AI
    parts: [{ type: "text", text: notification }],
  },
})
```

5 个后台任务，前 4 个完成发静默通知（`noReply: true`），第 5 个全部完成时才触发 AI 处理。

---

## 第 8 课：技能系统 — Skill Loader

### 核心概念

技能（Skill）= **一段 Markdown 指令，教会 AI 做某件特定的事**。

类比 `pytest` 的插件，但不是代码插件，而是"提示词插件"——一个 `.md` 文件，AI 读了就会。

### 一个技能文件

```markdown
---
name: playwright
description: Browser automation using Playwright MCP
model: anthropic/claude-sonnet-4-5
argument-hint: "What to test in the browser"
allowed-tools: Read Write Bash
mcp:
  playwright:
    command: npx
    args: ["@playwright/mcp"]
---

You are a browser automation expert using Playwright.
When the user asks you to test something in the browser:
1. Navigate to the URL
2. Take a screenshot
3. Report results
```

YAML 前置元数据 + Markdown 正文。

### 3 个文件

```
types.ts   →  数据结构（LoadedSkill, SkillScope...）
loader.ts  →  从文件系统发现和加载技能
merger.ts  →  合并多来源技能，处理优先级
```

### 技能来源与优先级

从 **4 个位置** 并行搜索：

```
~/.claude/skills/            → user（用户全局）
./.claude/skills/            → project（项目级）
~/.config/opencode/skills/   → opencode（OpenCode 全局）
./.opencode/skills/          → opencode-project（项目级 OpenCode）
```

优先级（从低到高）：

```
builtin (1) → config (2) → user (3) → opencode (4) → project (5) → opencode-project (6)
```

### 合并流程

```
第1步：加载所有内置技能
  ↓
第2步：应用配置文件中的定义（可覆盖/新增/删除）
  ↓
第3步：加载文件系统技能（高优先级覆盖低优先级）
  ↓
第4步：再次应用配置覆盖（配置拥有最终决定权）
  ↓
第5步：应用 disable 列表
  ↓
第6步：应用 enable 白名单
```

Python 等价核心逻辑：

```python
def merge_skills(builtin, config, user_claude, user_opencode, proj_claude, proj_opencode):
    skill_map = {}

    for s in builtin:
        skill_map[s.name] = s

    for name, entry in config.entries.items():
        if entry is False or entry.get("disable"):
            continue
        skill_map[name] = config_entry_to_loaded(name, entry)

    for skill in [*user_claude, *user_opencode, *proj_claude, *proj_opencode]:
        existing = skill_map.get(skill.name)
        if not existing or SCOPE_PRIORITY[skill.scope] > SCOPE_PRIORITY[existing.scope]:
            skill_map[skill.name] = skill

    for name in config.disable:
        skill_map.pop(name, None)

    return list(skill_map.values())
```

### 技能模板包装

```
<skill-instruction>
Base directory for this skill: /path/to/skill/
File references (@path) in this skill are relative to this directory.

{Markdown 正文}
</skill-instruction>

<user-request>
$ARGUMENTS
</user-request>
```

`$ARGUMENTS` 在运行时被用户的具体指令替换。

---

## 第 9 课：MCP 协议集成

### MCP 是什么？

**Model Context Protocol** = AI 工具的标准接口协议。

类比 **WSGI/ASGI** 让任何 Web 框架对接任何 Web 服务器，MCP 让任何 AI 模型对接任何外部工具。

### 架构分 3 层

```
┌──────────────────────────────────────────┐
│  skill-mcp-manager/manager.ts            │  ← 管理层：连接池、生命周期
│  SkillMcpManager class                   │
├──────────────────────────────────────────┤
│  MCP SDK                                 │  ← 传输层：stdio / HTTP
│  StdioClientTransport                    │
│  StreamableHTTPClientTransport           │
├──────────────────────────────────────────┤
│  mcp-oauth/                              │  ← 认证层：OAuth 2.1
│  discovery → provider → storage          │
└──────────────────────────────────────────┘
```

### 两种连接方式

**本地进程（stdio）**：

```json
{ "playwright": { "command": "npx", "args": ["@playwright/mcp"] } }
```

启动子进程，通过 stdin/stdout 通信。

**远程服务（HTTP）**：

```typescript
export const context7 = {
  type: "remote",
  url: "https://mcp.context7.com/mcp",
  headers: { Authorization: `Bearer ${process.env.CONTEXT7_API_KEY}` },
  oauth: false,
}
```

连接类型推断：

```typescript
function getConnectionType(config): ConnectionType | null {
  if (config.url) return "http"      // 有 URL → HTTP
  if (config.command) return "stdio"  // 有命令 → 本地进程
  return null
}
```

### SkillMcpManager — 连接池

```typescript
export class SkillMcpManager {
  private clients: Map<string, ManagedClient> = new Map()         // 连接池
  private pendingConnections: Map<string, Promise<Client>> = new Map()  // 防并发重连
  private authProviders: Map<string, McpOAuthProvider> = new Map()      // OAuth 缓存
  private readonly IDLE_TIMEOUT = 5 * 60 * 1000                        // 5分钟空闲断开
}
```

连接池的 key 是 `sessionID:skillName:serverName`，确保每个组合一个连接。

设计特点：**懒连接 + 空闲清理**——只在首次调用时建立连接，5 分钟没使用自动断开。

### OAuth 2.1 认证（4 步）

```
discovery.ts  →  第1步：GET /.well-known/oauth-authorization-server
                  得到 authorization_endpoint, token_endpoint

dcr.ts        →  第2步：动态客户端注册（获取 client_id）

provider.ts   →  第3步：PKCE 授权流程
                  1. generateCodeVerifier() → 随机字符串
                  2. generateCodeChallenge() → SHA256(verifier)
                  3. openBrowser(authUrl) → 用户授权
                  4. startCallbackServer() → localhost:19877 接收回调
                  5. 用 code + verifier 换 token

storage.ts    →  第4步：保存 token 到 ~/.opencode/mcp-oauth.json
```

Python 等价的 PKCE 核心：

```python
import hashlib, secrets, base64

verifier = secrets.token_urlsafe(32)
challenge = base64.urlsafe_b64encode(
    hashlib.sha256(verifier.encode()).digest()
).rstrip(b"=").decode()
```

---

## 总结与速查表

### 项目全景图

```
Oh My OpenCode 插件
│
├── 配置层 (config/schema.ts)
│   └── Zod schema 定义一切可配置项
│
├── 入口层 (index.ts)
│   └── 组装 tools + hooks + features → 返回插件接口
│
├── 工具层 (tools/)
│   ├── grep, glob, lsp_*     → 代码搜索/分析
│   ├── delegate_task          → 任务委派（核心引擎）
│   ├── skill, skill_mcp       → 技能执行
│   └── background_*           → 后台任务控制
│
├── 钩子层 (hooks/)
│   └── 25 个中间件，拦截 chat.message / tool.before / tool.after / event
│
├── Agent 层 (agents/)
│   └── 9 个 AI 智能体，各有 prompt + 元数据 + 工具限制
│
└── 特性层 (features/)
    ├── BackgroundManager      → 并发任务调度
    ├── Skill Loader           → 多来源技能发现与合并
    ├── MCP OAuth              → OAuth 2.1 + PKCE 认证
    └── Skill MCP Manager      → MCP 连接池管理
```

### TS 核心语法速查

| TS | Python | 课程 |
|----|--------|------|
| `z.object({...})` | `pydantic.BaseModel` | 第1课 |
| `z.infer<typeof X>` | 自动（Pydantic 天然） | 第1课 |
| `interface X {}` | `@dataclass` / `TypedDict` | 第3课 |
| `x?: T` | `x: T \| None = None` | 第3课 |
| `x ?? default` | `x if x is not None else default` | 第3课 |
| `{ ...obj, key: val }` | `{**obj, "key": val}` | 第5课 |
| `const { a, ...rest } = obj` | 解构赋值 | 第5课 |
| `source is Type` (类型守卫) | `TypeGuard` (3.10+) | 第5课 |
| `Promise.race/all` | `asyncio.wait/gather` | 第3课 |
| `await new Promise(r => setTimeout(r, ms))` | `await asyncio.sleep(s)` | 第6课 |
| `?.` 可选链 | `getattr(x, y, None)` | 第2课 |
| `Map<K,V>` | `dict[K, V]` | 第4课 |
| `private` / `static` | `_` 约定 / 类变量 | 第7课 |
| `Record<string, T>` | `dict[str, T]` | 第5课 |
| `Partial<T>` | 所有字段变 Optional | 第5课 |
| `.unref()` | 无等价（让定时器不阻止进程退出） | 第7课 |
| `export * from "./x"` | `from .x import *` | 第8课 |
| `crypto.randomUUID()` | `uuid.uuid4()` | 第7课 |
| `process.on("SIGINT", fn)` | `signal.signal(SIGINT, fn)` | 第7课 |
| `switch/case` | `match/case` (3.10+) | 第6课 |
| `` `template ${var}` `` | `f"template {var}"` | 第6课 |
| `60_000` | `60_000` | 第4课 |
| `array.map(fn)` | `[fn(x) for x in array]` | 第3课 |
| `array.reduce(fn, init)` | `functools.reduce(fn, array, init)` | 第3课 |
| `Object.entries(obj)` | `obj.items()` | 第5课 |
| `new URL(str)` | `urllib.parse.urlparse(str)` | 第9课 |

### 下一步建议

1. **跑测试**：`bun test` 看测试怎么写的（项目里有大量 `.test.ts` 文件）
2. **尝试改东西**：挑一个简单 hook（如 `comment-checker`），加个小功能
3. **读 Sisyphus 的 prompt**：`src/agents/prometheus-prompt.ts`，看主控 Agent 的完整系统提示词
4. **调试**：设置 `COMMENT_CHECKER_DEBUG=1` 等环境变量，看日志输出
