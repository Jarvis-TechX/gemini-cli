# Gemini CLI 技术指南 (Technical Guide)

> 完整的技术架构、数据流、核心原理和代码路径参考文档

**版本**: Based on codebase as of 2025-10-31
**目标读者**: 开发者、架构师、贡献者

---

## 目录 (Table of Contents)

1. [项目概览](#1-项目概览-project-overview)
2. [完整数据流](#2-完整数据流-complete-data-flow)
3. [核心架构设计](#3-核心架构设计-core-architecture)
4. [Agent调度系统](#4-agent调度系统-agent-scheduling)
5. [Prompt处理机制](#5-prompt处理机制-prompt-processing)
6. [Tool调度系统](#6-tool调度系统-tool-scheduling)
7. [流式处理架构](#7-流式处理架构-streaming-architecture)
8. [关键技术原理](#8-关键技术原理-key-technical-principles)
9. [代码路径索引](#9-代码路径索引-code-path-index)

---

## 1. 项目概览 (Project Overview)

### 1.1 Monorepo 结构

Gemini CLI 采用 npm workspaces 管理的 monorepo 架构，包含 4 个主要包:

```
gemini-cli/
├── packages/
│   ├── core/              # @google/gemini-cli-core - 核心库
│   ├── cli/               # @google/gemini-cli - CLI 应用
│   ├── a2a-server/        # @google/gemini-cli-a2a-server - Agent-to-Agent 服务器
│   └── test-utils/        # @google/gemini-cli-test-utils - 测试工具
├── package.json           # Workspace 根配置
└── tsconfig.json          # TypeScript 配置
```

**依赖关系**:

- CLI → Core (file:../core)
- A2A Server → Core (file:../core)
- 所有包要求 Node.js >= 20

**参考文件**: `/package.json`

### 1.2 核心技术栈

| 技术              | 用途                 | 版本要求        |
| ----------------- | -------------------- | --------------- |
| **TypeScript**    | 主要开发语言         | ^5.0            |
| **React + Ink**   | 终端 UI 框架         | React 19, Ink 5 |
| **Vitest**        | 单元测试             | ^2.0            |
| **@google/genai** | Gemini API SDK       | ^0.3.0          |
| **Express**       | A2A Server HTTP 框架 | ^4.19           |
| **Zod**           | Schema 验证          | ^3.23           |
| **Commander**     | CLI 参数解析         | ^12.1           |

### 1.3 核心架构模式

1. **Provider/Factory Pattern** - 依赖注入
2. **Event-Driven Architecture** - 事件驱动
3. **Policy Engine** - 策略引擎
4. **Registry Pattern** - 注册表模式
5. **State Machine** - 状态机 (Tool Call Status)
6. **Async Generators** - 流式处理

---

## 2. 完整数据流 (Complete Data Flow)

### 2.1 从用户输入到 API 调用的完整链路

```
┌─────────────────────────────────────────────────────────────────┐
│ 第 1 层: CLI 入口 (Entry Point)                                   │
└─────────────────────────────────────────────────────────────────┘
   packages/cli/index.ts:14
   ├─> #!/usr/bin/env node
   └─> main() 函数调用

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 2 层: React UI 初始化 (UI Initialization)                     │
└─────────────────────────────────────────────────────────────────┘
   packages/cli/src/gemini.tsx:50
   ├─> export const Gemini = ({ args }: GeminiProps)
   ├─> render(<App />, { ... })
   └─> Ink 渲染终端 UI

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 3 层: 用户输入捕获 (Input Capture)                             │
└─────────────────────────────────────────────────────────────────┘
   packages/cli/src/ui/components/InputPrompt.tsx:359
   ├─> handleInput(char, key) - 键盘事件处理
   ├─> 构建用户消息
   └─> 触发 onFinalSubmit(userMessage)

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 4 层: 消息提交处理 (Message Submit)                            │
└─────────────────────────────────────────────────────────────────┘
   packages/cli/src/ui/AppContainer.tsx:713
   ├─> handleFinalSubmit(userMessage, attachments)
   ├─> addMessage(userMessage) - 添加到历史
   └─> 调用 useGeminiStream hook

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 5 层: 消息队列管理 (Message Queue)                             │
└─────────────────────────────────────────────────────────────────┘
   packages/cli/src/ui/hooks/useMessageQueue.ts
   ├─> addMessage(message) - 加入队列
   └─> 序列化处理防止并发问题

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 6 层: 查询路由 (Query Routing)                                 │
└─────────────────────────────────────────────────────────────────┘
   packages/cli/src/ui/hooks/useGeminiStream.ts:805
   ├─> submitQuery(message, signal)
   ├─> 判断消息类型:
   │   ├─> /command → 斜杠命令
   │   ├─> @command → At 命令
   │   └─> 普通文本 → Prompt
   └─> 调用相应处理器

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 7 层: Prompt 组装 (Prompt Assembly)                            │
└─────────────────────────────────────────────────────────────────┘
   packages/core/src/core/client.ts:393
   ├─> startChat(extraHistory) - 初始化聊天
   ├─> getInitialChatHistory() - 环境上下文
   ├─> getCoreSystemPrompt() - 系统提示词
   ├─> getIdeContextParts() - IDE 上下文
   └─> 创建 GeminiChat 实例

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 8 层: Turn 执行 (Turn Execution)                               │
└─────────────────────────────────────────────────────────────────┘
   packages/core/src/core/turn.ts:229
   ├─> Turn.run(model, request, signal)
   ├─> 调用 chat.sendMessageStream()
   └─> 返回事件流 (async generator)

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 9 层: GeminiChat 消息发送 (Message Send)                       │
└─────────────────────────────────────────────────────────────────┘
   packages/core/src/core/geminiChat.ts:226
   ├─> sendMessageStream(model, params, prompt_id)
   ├─> 记录用户消息到历史
   ├─> 添加到 this.history
   └─> 调用底层 API

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 10 层: API 调用 (API Call)                                     │
└─────────────────────────────────────────────────────────────────┘
   packages/core/src/core/geminiChat.ts:367
   ├─> chat.generateContentStream({
   │     contents: this.history,
   │     systemInstruction,
   │     tools: [functionDeclarations],
   │     generationConfig: { temperature, ... }
   │   })
   └─> 返回流式响应

                         ↓

┌─────────────────────────────────────────────────────────────────┐
│ 第 11 层: SDK 调用云端 (SDK to Cloud)                             │
└─────────────────────────────────────────────────────────────────┘
   @google/genai SDK
   ├─> contentGenerator.ts - ContentGenerator 接口
   ├─> 根据认证类型选择 API:
   │   ├─> OAuth (LOGIN_WITH_GOOGLE)
   │   ├─> API Key (USE_GEMINI)
   │   └─> Vertex AI (USE_VERTEX_AI)
   └─> HTTP 请求到 Gemini API
```

### 2.2 关键断点位置 (Debugging Breakpoints)

| 层级           | 文件路径                                             | 行号 | 作用                    |
| -------------- | ---------------------------------------------------- | ---- | ----------------------- |
| **入口**       | `packages/cli/index.ts`                              | 14   | main() 函数入口         |
| **UI初始化**   | `packages/cli/src/gemini.tsx`                        | 50   | React 组件渲染          |
| **输入捕获**   | `packages/cli/src/ui/components/InputPrompt.tsx`     | 359  | handleInput()           |
| **消息提交**   | `packages/cli/src/ui/AppContainer.tsx`               | 713  | handleFinalSubmit()     |
| **查询路由**   | `packages/cli/src/ui/hooks/useGeminiStream.ts`       | 805  | submitQuery()           |
| **命令执行**   | `packages/cli/src/ui/hooks/slashCommandProcessor.ts` | 339  | command.action()        |
| **Chat初始化** | `packages/core/src/core/client.ts`                   | 393  | startChat()             |
| **API调用**    | `packages/core/src/core/geminiChat.ts`               | 367  | generateContentStream() |

**参考文档**: `DEBUG-BREAKPOINTS.md`, `DATA-FLOW-GUIDE.md`

---

## 3. 核心架构设计 (Core Architecture)

### 3.1 模块职责划分

#### Core Package 模块

| 模块                  | 职责                           | 关键文件                                            |
| --------------------- | ------------------------------ | --------------------------------------------------- |
| **config/**           | 配置管理、存储、模型管理       | `config.ts`, `storage.ts`, `models.ts`              |
| **core/**             | LLM 客户端、内容生成、聊天逻辑 | `client.ts`, `contentGenerator.ts`, `geminiChat.ts` |
| **tools/**            | 工具定义和注册表               | `tool-registry.ts`, `tools.ts`                      |
| **agents/**           | Agent 定义和注册表             | `registry.ts`, `executor.ts`, `types.ts`            |
| **mcp/**              | Model Context Protocol 集成    | `oauth-provider.ts`, `oauth-token-storage.ts`       |
| **services/**         | Git、文件发现、Shell 执行      | `gitService.ts`, `fileDiscoveryService.ts`          |
| **confirmation-bus/** | 工具确认工作流                 | `message-bus.ts`, `types.ts`                        |
| **policy/**           | 策略引擎                       | `policy-engine.ts`, `types.ts`                      |
| **telemetry/**        | 事件日志和可观测性             | 各种 logger 和事件类型                              |

#### CLI Package 模块

| 模块             | 职责                 | 关键文件                                       |
| ---------------- | -------------------- | ---------------------------------------------- |
| **ui/commands/** | 斜杠命令和交互功能   | `aboutCommand.ts`, `authCommand.ts` 等         |
| **ui/hooks/**    | React hooks 状态管理 | `useGeminiStream.ts`, `useAuth.ts` 等          |
| **services/**    | 命令加载和提示处理   | `CommandService.ts`, `BuiltinCommandLoader.ts` |
| **config/**      | 扩展管理、设置、策略 | `extension-manager.ts`, `settings.ts`          |

#### A2A Server Package 模块

| 模块             | 职责                     | 关键文件                              |
| ---------------- | ------------------------ | ------------------------------------- |
| **http/**        | Express 应用和 HTTP 端点 | `app.ts`, `server.ts`, `endpoints.ts` |
| **agent/**       | Agent 执行逻辑           | `executor.ts`, `task.ts`              |
| **config/**      | 服务器配置和扩展         | `config.ts`, `settings.ts`            |
| **persistence/** | GCS 存储集成             | `gcs.ts`                              |

### 3.2 关键抽象接口

#### 3.2.1 ContentGenerator (内容生成器)

```typescript
// File: packages/core/src/core/contentGenerator.ts
export interface ContentGenerator {
  generateContent(request, userPromptId): Promise<GenerateContentResponse>;
  generateContentStream(
    request,
    userPromptId,
  ): AsyncGenerator<GenerateContentResponse>;
  countTokens(request): Promise<CountTokensResponse>;
  embedContent(request): Promise<EmbedContentResponse>;
}
```

**实现**:

- `RecordingContentGenerator` - 记录所有请求/响应
- `LoggingContentGenerator` - 日志记录 API 交互
- `FakeContentGenerator` - 测试用 mock 响应

**工厂方法**: `createContentGenerator(authType, config)`

#### 3.2.2 ToolInvocation (工具调用)

```typescript
// File: packages/core/src/tools/tools.ts
export interface ToolInvocation<TParams, TResult> {
  params: TParams;
  getDescription(): string;
  toolLocations(): ToolLocation[];
  shouldConfirmExecute(
    abortSignal,
  ): Promise<ToolCallConfirmationDetails | false>;
  execute(signal, updateOutput?, config?): Promise<TResult>;
}
```

**基类**: `BaseToolInvocation<TParams, TResult>`

- 集成 MessageBus 策略决策
- 实现 `shouldConfirmExecute()` 确认逻辑
- 30 秒策略决策超时

#### 3.2.3 DeclarativeTool (声明式工具)

```typescript
// File: packages/core/src/tools/tools.ts
export interface DeclarativeTool<TParams, TResult> {
  name: string;
  displayName: string;
  description: string;
  kind: Kind; // Read, Edit, Delete, Move, Search, Execute, Think, Fetch, Other
  schema: FunctionDeclaration;
  build(params: TParams): Promise<ToolInvocation<TParams, TResult>>;
}
```

**基类**: `BaseDeclarativeTool<TParams, TResult>`

- 自动 JSON Schema 验证
- `buildAndExecute()` 便捷方法

### 3.3 依赖注入模式

#### 3.3.1 Constructor-based DI

```typescript
// 示例: CommandService
export class CommandService {
  static async create(
    loaders: ICommandLoader[],  // 注入加载器
    signal: AbortSignal
  ): Promise<CommandService> { ... }
}

// 示例: BuiltinCommandLoader
export class BuiltinCommandLoader implements ICommandLoader {
  constructor(private readonly config: Config) { }  // 注入配置
}
```

#### 3.3.2 Factory Pattern

```typescript
// File: packages/core/src/core/contentGenerator.ts
export function createContentGenerator(authType, config): ContentGenerator {
  switch (authType) {
    case AuthType.LOGIN_WITH_GOOGLE:
      return new GoogleOAuthContentGenerator(config);
    case AuthType.USE_GEMINI:
      return new ApiKeyContentGenerator(config);
    case AuthType.USE_VERTEX_AI:
      return new VertexAIContentGenerator(config);
  }
}
```

#### 3.3.3 Service Locator Pattern

```typescript
// Singleton 实例
IdeClient.getInstance();
ideContextStore; // 全局 IDE 上下文
coreEvents; // 全局 EventEmitter
```

### 3.4 扩展架构 (Plugin/Extension)

#### ExtensionManager

**文件**: `packages/cli/src/config/extension-manager.ts`

**功能**:

- 扩展安装 (GitHub source / releases)
- 扩展启用/禁用
- 设置和同意流程
- 环境变量解析

**配置存储**:

- `.gemini/extensions.json` - 扩展配置
- `.gemini/extensions/[name]/install-metadata.json` - 安装元数据

#### MCP Server 集成

- `Model Context Protocol` 服务器作为扩展加载
- `MCPOAuthProvider` - OAuth 启用的 MCP 工具
- `McpPromptLoader` - MCP prompt 发现

#### 命令扩展

- `FileCommandLoader` - 基于文件的命令加载
- `BuiltinCommandLoader` - 内置 CLI 命令
- `CommandService` - 聚合加载器，冲突解决

---

## 4. Agent调度系统 (Agent Scheduling)

### 4.1 Agent 定义和注册

#### AgentDefinition 接口

**文件**: `packages/core/src/agents/types.ts`

```typescript
interface AgentDefinition<TOutput extends z.ZodTypeAny = z.ZodUnknown> {
  name: string; // 唯一标识符
  displayName?: string; // 显示名称
  description: string; // 功能描述
  promptConfig: PromptConfig; // Prompt/查询配置
  modelConfig: ModelConfig; // LLM 模型参数
  runConfig: RunConfig; // 执行约束
  toolConfig?: ToolConfig; // 可用工具
  outputConfig?: OutputConfig<TOutput>; // 输出 Schema
  inputConfig: InputConfig; // 输入参数
  processOutput?: (output: z.infer<TOutput>) => string;
}
```

**子配置**:

- `PromptConfig`: 系统提示词、初始消息、查询模板 (支持 `${variable}` 模板)
- `ModelConfig`: 模型选择、temperature、top_p、thinking budget
- `RunConfig`: max_time_minutes、max_turns 执行限制
- `InputConfig`: 参数定义和类型验证
- `OutputConfig`: Zod schema 输出验证

#### AgentRegistry (Agent 注册表)

**文件**: `packages/core/src/agents/registry.ts`

```typescript
export class AgentRegistry {
  async initialize(): Promise<void>;
  private loadBuiltInAgents(): void;
  protected registerAgent<TOutput>(definition: AgentDefinition<TOutput>): void;
  getDefinition(name: string): AgentDefinition | undefined;
  getAllDefinitions(): AgentDefinition[];
}
```

**内置 Agent**:

- `CodebaseInvestigatorAgent` - 代码库调查 Agent

**配置驱动**: 遵循配置中的 agent 设置 (模型、thinking budget、超时)

### 4.2 AgentExecutor (执行引擎)

**文件**: `packages/core/src/agents/executor.ts` (772 行)

#### 核心架构

```typescript
export class AgentExecutor<TOutput extends z.ZodTypeAny> {
  readonly definition: AgentDefinition<TOutput>;
  private readonly agentId: string; // 唯一运行时 ID
  private readonly toolRegistry: ToolRegistry; // 隔离工具注册表
  private readonly runtimeContext: Config;
  private readonly onActivity?: ActivityCallback; // 活动事件回调
}
```

#### 关键方法

**1. 工厂方法 - `create()`** (lines 78-122)

- 创建和验证 `AgentExecutor` 实例
- 构建隔离的 `ToolRegistry`
- 验证所有工具适合非交互式使用
- 保留父级 prompt ID 上下文

**2. 主执行循环 - `run()`** (lines 156-249)

```typescript
async run(inputs: AgentInputs, signal: AbortSignal): Promise<OutputObject<TOutput>> {
  // 1. 初始化 GeminiChat with 模板化 prompts
  // 2. 准备工具列表
  // 3. 循环:
  //    - 检查终止条件 (timeout/max_turns)
  //    - 调用 LLM with 当前消息和工具
  //    - 提取 function calls 和 thoughts
  //    - 并行处理所有 function calls
  //    - 更新对话历史
  //    - 检查 complete_task 调用
  // 4. 返回 OutputObject with 结果和终止原因
}
```

**执行流程**:

```
初始化 GeminiChat
  ↓
准备工具列表 (从 toolConfig)
  ↓
循环开始:
  ├─> 检查终止条件 (timeout/max_turns)
  ├─> 调用 LLM with 当前消息 + tools
  ├─> 提取 function calls 和 thoughts
  ├─> 并行处理所有 function calls (Promise.all)
  ├─> 更新对话历史
  └─> 检查 complete_task → 退出
  ↓
返回 OutputObject
```

**3. Function Call 处理 - `processFunctionCalls()`** (lines 374-592)

- 处理特殊的 `complete_task` 工具
- 使用 Zod 验证输出 schema
- 并行执行授权工具
- 收集同步响应

**4. 工具验证 - `validateTools()`** (lines 707-731)

- 允许列表: LS, READ_FILE, GREP, GLOB, READ_MANY_FILES, MEMORY, WEB_SEARCH
- 阻止交互式工具 (shell, write file)

#### 终止模式 (AgentTerminateMode)

```typescript
enum AgentTerminateMode {
  GOAL = 'goal', // 成功调用 complete_task
  TIMEOUT = 'timeout', // 超过 max_time_minutes
  MAX_TURNS = 'max_turns', // 超过 max_turns
  ERROR = 'error', // 停止调用工具但未完成
  ABORTED = 'aborted', // 用户取消
}
```

### 4.3 Agent as Tool (Agent作为工具)

#### SubagentToolWrapper

**文件**: `packages/core/src/agents/subagent-tool-wrapper.ts`

```typescript
export class SubagentToolWrapper extends BaseDeclarativeTool<
  AgentInputs,
  ToolResult
> {
  // 将 Agent 包装为标准 DeclarativeTool
  // 从 InputConfig 动态生成 JSON schema
  // 支持 Zod schema 强类型
  // 调用时创建 SubagentInvocation
}
```

#### SubagentInvocation

**文件**: `packages/core/src/agents/invocation.ts`

```typescript
export class SubagentInvocation<
  TOutput extends z.ZodTypeAny,
> extends BaseToolInvocation<AgentInputs, ToolResult> {
  async execute(
    signal: AbortSignal,
    updateOutput?: (output: string | AnsiOutput) => void,
  ): Promise<ToolResult> {
    // 1. 验证输入参数
    // 2. 创建 AgentExecutor 实例
    // 3. 桥接 agent 活动事件到工具输出流
    // 4. 格式化最终结果为 ToolResult
    // 5. 优雅处理错误
  }
}
```

### 4.4 Agent 活动事件 (Activity Events)

```typescript
interface SubagentActivityEvent {
  isSubagentActivityEvent: true;
  agentName: string;
  type: 'TOOL_CALL_START' | 'TOOL_CALL_END' | 'THOUGHT_CHUNK' | 'ERROR';
  data: Record<string, unknown>;
}
```

**事件流**:

- Agent 通过 `ActivityCallback` 发出事件
- `SubagentInvocation` 桥接到工具输出流
- Thoughts 流式显示为: `🤖💭 {thought text}`

### 4.5 内置 Agent 示例: CodebaseInvestigatorAgent

**文件**: `packages/core/src/agents/codebase-investigator.ts`

**特性**:

- **Input**: 单个 `objective` 参数 (详细调查目标)
- **Output**: 结构化 JSON 报告:
  - SummaryOfFindings
  - ExplorationTrace (步骤记录)
  - RelevantLocations (文件、推理、关键符号)
- **Tools**: 只读工具 (LS, READ_FILE, GLOB, GREP)
- **执行**: 最大 5 分钟, 最大 15 turns
- **Temperature**: 0.1 (确定性)
- **Thinking**: 启用深度推理 (thinkingBudget: -1)

**Prompt 工程**:

- 详细系统提示词和调查方法论
- Scratchpad 模式用于推理
- 分层清单跟踪
- 明确终止条件 (Questions 列表为空)

---

## 5. Prompt处理机制 (Prompt Processing)

### 5.1 系统 Prompt 生成

#### getCoreSystemPrompt

**文件**: `packages/core/src/core/prompts.ts`

```typescript
export function getCoreSystemPrompt(
  config: Config,
  userMemory?: string,
): SystemInstruction {
  // 1. 读取自定义 system.md (如果存在)
  //    环境变量: GEMINI_SYSTEM_MD
  //    默认路径: ~/.gemini/system.md

  // 2. 使用默认系统提示词:
  //    - 核心要求 (约定、库、风格、注释)
  //    - 主要工作流 (软件工程任务、新应用)
  //    - 操作指南 (shell 工具、tone/style、安全)
  //    - Git 仓库指导

  // 3. 附加用户记忆 (如果提供)

  return systemInstruction;
}
```

**压缩 Prompt**: `getCompressionPrompt()`

- 指示模型作为状态管理器
- 要求 XML 结构化快照和 scratchpad 推理
- 用于历史压缩

### 5.2 上下文组装

#### 环境上下文

**文件**: `packages/core/src/utils/environmentContext.ts`

```typescript
export function getInitialChatHistory(
  config: Config,
  extraHistory?: Content[],
): Content[] {
  // 获取环境上下文
  const envContext = getEnvironmentContext(config);

  // 包含:
  // - 当前日期/时间
  // - 操作系统信息
  // - 工作目录上下文
  // - 文件夹结构

  // 与 extraHistory 组合
  return [envContext, ...extraHistory];
}
```

**目录上下文**: `getDirectoryContextString(config)`

- 从 `config.getWorkspaceContext()` 获取工作目录
- 通过 `getFolderStructure()` 构建文件夹结构
- 生成可读的目录序言

#### IDE 上下文

**文件**: `packages/core/src/ide/ideContext.ts`

**IDE Context Store**:

- 维护 `IdeContext` 状态和工作区状态
- 跟踪打开的文件属性: path, timestamp, isActive, cursor, selectedText
- 应用截断限制:
  - `IDE_MAX_OPEN_FILES` - 限制显示的打开文件数
  - `IDE_MAX_SELECTED_TEXT_LENGTH` - 截断选中文本

**上下文 Delta 跟踪** (`client.ts`):

```typescript
// File: packages/core/src/core/client.ts
getIdeContextParts(forceFullContext?: boolean): Part[] {
  // 完整上下文模式: 发送所有 active/other 打开文件的完整 JSON
  // Delta 模式: 仅发送变化的文件 (opened/closed/modified)
  // 包含: active file path, cursor position, selected text
  // 跟踪 lastSentIdeContext 以优化 token 使用
}
```

### 5.3 Chat 初始化

**文件**: `packages/core/src/core/client.ts`

```typescript
startChat(extraHistory?: Content[]): GeminiChat {
  // 1. 获取初始历史
  const history = getInitialChatHistory(config, extraHistory);

  // 2. 获取系统指令
  const systemInstruction = getCoreSystemPrompt(config, userMemory);

  // 3. 创建 GeminiChat
  return new GeminiChat({
    systemInstruction,
    tools: [{ functionDeclarations }],
    thinkingConfig,
    history
  });
}
```

### 5.4 消息组装

#### 用户消息捕获

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
sendMessageStream(model, params, prompt_id) {
  // 1. 从 params.message 捕获用户输入 (PartListUnion 类型)
  // 2. 通过 ChatRecordingService 记录消息
  // 3. 添加到历史: this.history.push(userContent)
  // 4. 使用 createUserContent() 规范化输入
  // 5. 过滤 functionResponse parts (单独存储)
}
```

**用户消息规范化** 通过 `toParts()` 转换:

- 处理 string → {text} part 转换
- 过滤无效/null parts
- 特殊处理 thought parts (为兼容性剥离)

### 5.5 历史管理和压缩

#### 历史策展

**文件**: `packages/core/src/services/chatCompressionService.ts`

```typescript
extractCuratedHistory(comprehensiveHistory: Content[]): Content[] {
  // 过滤有效 turns
  // - 移除无效/空的模型输出 (安全过滤器 artifacts)
  // - 保留有效的 user-model 对
  // - 使用 isValidContent() 验证
}
```

#### 压缩策略

**阈值**:

- `COMPRESSION_TOKEN_THRESHOLD = 0.7` - 达到限制的 70% 时压缩
- `COMPRESSION_PRESERVE_THRESHOLD = 0.3` - 保留最后 30% 的历史

**`findCompressSplitPoint(contents, fraction)`** - 确定压缩边界

- 在用户消息边界分割 (不在对话中间)
- 确保安全压缩点
- 永不在 tool-call 中间压缩

**压缩 Prompt**: 生成状态快照 XML:

- `<overall_goal>` - 单句用户目标
- `<key_knowledge>` - 关键事实和约束
- `<file_system_state>` - 修改/创建的文件
- `<recent_actions>` - Tool call 摘要
- `<current_plan>` - 逐步进度

### 5.6 消息格式化和 Part 转换

#### Part-to-String 转换

**文件**: `packages/core/src/utils/partUtils.ts`

```typescript
partToString(value: PartListUnion, options?: { verbose?: boolean }): string {
  // 处理 strings, arrays, Part 对象
  // Verbose 模式摘要非文本 parts: [File Data], [Function Call: name], [Thought: text]
  // 返回文本内容或空字符串
}

getResponseText(response: GenerateContentResponse): string {
  // 从 API 响应提取文本
  // 获取第一个候选的 content parts
  // 过滤为纯文本 parts (排除 function calls, file data)
  // 连接多个文本 parts
}
```

#### Content/Part 转换器

**文件**: `packages/core/src/code_assist/converter.ts`

```typescript
// 规范化 content 数组
toContents(contents: ContentListUnion): Content[]

// 转换为 Content 对象
toContent(content: ContentUnion): Content
// 处理: string, Part[], Content 对象
// 默认规范化 role 为 'user'

// 转换为 Part 数组
toParts(parts: PartUnion[]): Part[]
// toPart(part: PartUnion) - 单个 part 转换
//   String → {text: string}
//   Thought parts → 转换为文本格式 (API 兼容)
//   剥离无效 parts 用于 token 计数 API
```

### 5.7 Agent 系统 Prompt

**文件**: `packages/core/src/agents/executor.ts`

```typescript
buildSystemPrompt(inputs: AgentInputs): string {
  // 1. 模板化系统 prompt with inputs
  //    通过 templateString() 注入 agent inputs

  // 2. 附加目录/环境上下文
  //    通过 getDirectoryContextString()

  // 3. 添加非交互式执行规则

  // 4. 引用 complete_task 工具作为完成机制

  return systemPrompt;
}
```

**模板字符串处理**:

- `templateString(template, inputs)` - 用输入值替换 `${key}`
- 支持动态 prompts 的变量替换

---

## 6. Tool调度系统 (Tool Scheduling)

### 6.1 CoreToolScheduler 架构

**文件**: `packages/core/src/core/coreToolScheduler.ts` (1396 行)

#### 工具调用状态机

```
validating
    ↓
(if valid) → scheduled
    ↓
(if confirmation needed) → awaiting_approval
    ↓
executing
    ↓
success / error / cancelled
```

**状态转换**:

1. **validating**: 初始状态
   - 验证参数 against 工具 schema
   - 构建 invocation 实例

2. **scheduled**: 工具准备执行
   - 可以进入执行或确认

3. **awaiting_approval**: 等待用户确认
   - 持有确认详情 (type, title, prompt)
   - 与 MessageBus 策略引擎集成

4. **executing**: 工具正在运行
   - 通过回调流式输出
   - 跟踪开始时间和进程 ID
   - 可被用户取消

5. **success/error/cancelled**: 终止状态
   - 记录持续时间和结果
   - 存储响应和结果显示

#### 调度方法

**1. `schedule(request, signal)`** (lines 668-708)

- 调度工具调用的公共入口
- 实现并发调用的请求队列
- 如果调度器忙，队列请求稍后处理
- 否则调用 `_schedule()`

**2. `_schedule(request, signal)`** (lines 741-809)

- 创建状态为 'validating' 的 ToolCall 对象
- 在处理前验证所有请求
- 填充工具调用队列
- 通过 `_processNextInQueue()` 启动处理

**3. `_processNextInQueue(signal)`** (lines 811-900+)

- 一次处理一个工具 (顺序)
- 步骤间检查取消
- 验证期间处理错误情况
- 完成后移动到队列中的下一个

#### 状态管理

```typescript
setStatusInternal(): // 重载方法用于状态转换
  // - 不可变更新工具调用状态
  // - 为终止状态计算持续时间
  // - 通知观察者变化

notifyToolCallsUpdate(): // 向 UI 发布更新

checkAndNotifyCompletion(): // 完成时最终化 batch
```

### 6.2 工具定义模式

#### 三层架构

**文件**: `packages/core/src/tools/tools.ts`

**1. ToolInvocation Interface** (lines 25-66)

- 表示已验证、准备执行的工具调用
- 参数已预验证
- 方法:
  - `getDescription()` - 执行前描述
  - `toolLocations()` - 工具影响的文件
  - `shouldConfirmExecute()` - 确认决策
  - `execute()` - 实际执行 with abort signal & output callback

**2. BaseToolInvocation Class** (lines 71-234)

- 所有工具 invocations 的基类
- MessageBus 集成用于策略决策 (lines 92-112)
- 实现 `shouldConfirmExecute()` with 策略引擎支持
- 处理遗留确认流程回退
- 30 秒策略决策超时 (line 204)

**3. DeclarativeTool Interface** (lines 244-289)

- 工具元数据 (name, displayName, description, kind)
- Schema 定义 (FunctionDeclaration)
- `build(params)` - 创建已验证的 invocations
- 工具分类 with `Kind` enum

#### Tool Kind 枚举

```typescript
enum Kind {
  Read = 'read', // 文件读取、搜索
  Edit = 'edit', // 文件修改
  Delete = 'delete', // 文件/目录删除
  Move = 'move', // 文件/目录移动
  Search = 'search', // 内容搜索
  Execute = 'execute', // Shell/外部命令
  Think = 'think', // 推理/分析
  Fetch = 'fetch', // Web 抓取
  Other = 'other', // 其他操作
}
```

### 6.3 Tool Registry

**文件**: `packages/core/src/tools/tool-registry.ts` (489 行)

```typescript
export class ToolRegistry {
  // 所有可用工具的中央注册表
  // 管理内置和发现的工具

  registerTool(tool: DeclarativeTool): void;
  getTool(name: string): DeclarativeTool | undefined;
  getAllTools(): DeclarativeTool[]; // 按 display name 排序
  getAllToolNames(): string[];
  getFunctionDeclarations(): FunctionDeclaration[]; // Gemini function schemas

  // 发现
  discoverAllTools(): Promise<void>; // 从命令 + MCP 服务器发现
  discoverMcpTools(): Promise<void>; // 仅 MCP 服务器发现
  removeMcpToolsByServer(serverName): void;
  discoverToolsForServer(serverName): Promise<void>;
}
```

#### 工具发现流程

```
1. 从配置解析发现命令 (line 308)
2. 执行发现命令 (line 314)
3. 解析 JSON 数组输出 (10MB 大小限制 - line 320)
4. 提取 function_declarations 或 functionDeclarations (lines 388-391)
5. 注册每个为 DiscoveredTool
```

### 6.4 MCP 集成

#### MCP Client Manager

**文件**: `packages/core/src/tools/mcp-client-manager.ts`

```typescript
export class McpClientManager {
  // 管理多个 MCP 客户端的生命周期

  discoverAllMcpTools(): Promise<void>; // 跨服务器并行发现 (line 39)
  stop(): Promise<void>; // 清理和断开 (line 92)

  // 发现状态跟踪: NOT_STARTED → IN_PROGRESS → COMPLETED
}
```

#### MCP Client

**文件**: `packages/core/src/tools/mcp-client.ts`

```typescript
export class McpClient {
  // 单个 MCP 服务器连接管理

  // 状态: DISCONNECTED → CONNECTING → CONNECTED → DISCONNECTING

  connect(): Promise<void>; // 建立传输 (SSE, Stdio, HTTP) (line 104)
  discover(): Promise<void>; // 发现工具 & prompts (line 141)
  disconnect(): Promise<void>; // 清理 (line 159)

  // 支持多种传输类型: SSE, Stdio, StreamableHTTP
  // OAuth 认证支持 via 配置 providers
  // 10 分钟默认 MCP 超时 (line 48)
}
```

#### MCP Tool

**文件**: `packages/core/src/tools/mcp-tool.ts`

```typescript
export class DiscoveredMCPToolInvocation extends BaseToolInvocation {
  // 工具调用包装器
  // 复合命名: serverName__toolName 用于策略检查 (line 80)
  // MCP 确认详情 with 服务器感知 (lines 100-114)
  // 服务器 & 工具 allowlisting (lines 86-98)
  // 调用 mcpTool.callTool() with function call (line 167)
  // 从 MCP 响应检测错误 (lines 118-137)
  // 内容转换 for LLM 兼容性 (line 195)
}

export class DiscoveredMCPTool extends BaseDeclarativeTool {
  // 包装来自 MCP 服务器的工具
  // 服务器名称和工具名称跟踪
  // 签名 MCP 服务器的信任标志
  // 确认结果 for allowlisting:
  //   - ProceedAlwaysServer - 来自服务器的所有工具
  //   - ProceedAlwaysTool - 仅特定工具
}
```

### 6.5 内置工具示例

#### ReadFileTool

**文件**: `packages/core/src/tools/read-file.ts`

```typescript
export class ReadFileTool extends BaseDeclarativeTool<
  ReadFileToolParams,
  ToolResult
> {
  // 参数: absolute_path, offset (optional), limit (optional)
  // 基于行的截断 with offset/limit 支持大文件
  // 遥测日志 (FileOperationEvent)
  // MIME 类型检测
  // 编程语言检测
}
```

#### ShellTool

**文件**: `packages/core/src/tools/shell.ts`

```typescript
export class ShellToolInvocation extends BaseToolInvocation {
  // 命令 allowlisting with 根命令提取
  // 目录感知执行
  // 通过回调实时输出流
  // 进程 PID 跟踪
  // 大输出的摘要生成
  // 内存使用格式化
  // 退出代码处理
  // ANSI 输出支持
}
```

#### EditTool

**文件**: `packages/core/src/tools/edit.ts`

```typescript
export class EditTool extends ModifiableDeclarativeTool {
  // 编辑器修改的 ModifiableDeclarativeTool 支持
  // Diff 生成和预览
  // 多次出现替换处理
  // IDE 集成 for 确认文件更改
  // 实现 getModifyContext() for 编辑器流程
}
```

### 6.6 工具执行流程

```
用户输入
  ↓
GEMINI 模型响应 (with function calls)
  ↓
GeminiChat 处理响应
  ↓
CoreToolScheduler.schedule(ToolCallRequestInfo[], signal)
  ├─ 为每个请求创建 ToolCall 状态
  ├─ 验证工具存在于 registry
  ├─ 通过 tool.build(args) 构建 invocation
  │  └─ 验证参数
  ↓
_schedule() → _processNextInQueue() (顺序)
  ├─ 调用 invocation.shouldConfirmExecute(signal)
  ↓
确认决策
  ├─ 不需要确认 → scheduled
  ├─ 自动批准 (YOLO, allowlist) → scheduled
  ├─ 需要手动批准 → awaiting_approval
  │  └─ 用户 approves/rejects/modifies
  ↓
状态: scheduled → executing
  ├─ 调用 invocation.execute(signal, outputCallback, config)
  ├─ 通过回调流式实时输出
  ↓
执行完成
  ├─ Success: 转换结果为 FunctionResponse
  ├─ Error: 捕获错误 with ToolErrorType
  ├─ Cancelled: 取消消息
  ↓
状态: executing → success/error/cancelled
  ↓
checkAndNotifyCompletion()
  ├─ 将完成的调用移至 batch
  ├─ 处理队列中的下一个工具
  ├─ Batch 完成 → onAllToolCallsComplete callback
  ↓
结果返回给模型
  └─ Part[] with FunctionResponse for LLM 历史
```

---

## 7. 流式处理架构 (Streaming Architecture)

### 7.1 核心流处理

#### 主流编排

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// sendMessageStream() (lines 226-343)
async function* sendMessageStream(
  model: string,
  params: { message: PartListUnion },
  prompt_id: string,
): AsyncGenerator<StreamEvent> {
  // 1. 记录用户消息
  // 2. 添加到历史
  // 3. 重试循环 (最多 2 次尝试)
  //    - 调用 API
  //    - 处理流响应
  //    - 重试时调整 temperature (增加到 1)
  //    - yield RETRY 事件通知 UI
  // 4. 返回流式事件
}
```

**重试选项**: `INVALID_CONTENT_RETRY_OPTIONS`

- 最多 2 次尝试
- Temperature 调整以获得多样性

#### Turn-based 处理

**文件**: `packages/core/src/core/turn.ts`

```typescript
// Turn.run() (lines 229-362)
async function* run(
  model: string,
  request: GeminiStreamRequest,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent> {
  // 高级 async generator 处理流事件
  // 转换低级 API chunks 为语义事件
  // 实现基于 turn 的 agentic 循环
  // 通过 AbortSignal 检查处理用户取消 (lines 249-251)
}
```

### 7.2 流事件类型

#### StreamEventType

```typescript
// packages/core/src/core/geminiChat.ts
enum StreamEventType {
  CHUNK, // 常规 API 响应 chunks
  RETRY, // 信号即将重试 (UI 应丢弃部分内容)
}
```

#### GeminiEventType

```typescript
// packages/core/src/core/turn.ts (lines 49-65)
enum GeminiEventType {
  Content, // 文本响应
  Thought, // 模型推理
  ToolCallRequest, // 工具调用请求
  ToolCallResponse, // 工具响应
  ToolCallConfirmation, // 工具确认
  UserCancelled, // 用户取消
  Error, // 错误
  ChatCompressed, // 历史压缩
  Finished, // 响应完成
  LoopDetected, // 检测到循环
  Citation, // 引用
  Retry, // 重试
  ContextWindowWillOverflow, // 上下文窗口溢出
  InvalidStream, // 无效流
}
```

### 7.3 Chunk 解析

#### 流响应处理

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// processStreamResponse() (lines 491-585)
async function* processStreamResponse(
  streamResponse: AsyncGenerator<GenerateContentResponse>,
): AsyncGenerator<GenerateContentResponse> {
  // 1. 使用 for await (const chunk of streamResponse) 迭代 chunks
  // 2. 使用 isValidResponse() 验证响应 (lines 73-82)
  // 3. 累积文本 parts 并合并连续文本 chunks (lines 534-545)
  // 4. 检测 thoughts, tool calls, finish reasons
  // 5. 从每个 chunk 记录 token 使用元数据 (lines 521-528)
  // 6. 如果流结束没有必要条件抛出 InvalidStreamError
}
```

#### Chunk 解析

**文件**: `packages/core/src/core/turn.ts`

```typescript
// Chunk 解析 (lines 260-318)
// - 通过 getResponseText(resp) 提取文本内容
// - 通过 resp.functionCalls 检测 function calls
// - 通过 getCitations() 提取引用 (lines 392-401)
// - 检查 finish reason 和 usage 元数据
// - 从 chunks 创建语义事件
```

### 7.4 实时 UI 更新

#### useGeminiStream Hook

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// useGeminiStream() (lines 90-1290)
// 管理流状态的中央 hook

// processGeminiStreamEvents() (lines 717-803)
async function processGeminiStreamEvents(
  stream: AsyncGenerator<ServerGeminiStreamEvent>,
) {
  // 1. 使用 for await (const event of stream) 迭代流事件
  // 2. 按事件类型分派到特定处理器
  // 3. 收集 tool call 请求并调度它们
}
```

#### 事件处理器

**handleContentEvent()** (lines 477-533)

```typescript
// - 缓冲传入的文本内容
// - 为性能分割大消息 (lines 500-529)
// - 使用 findLastSafeSplitPoint() 避免分割 mid-character/word
// - 维护 pending history item for 实时显示
// - 通过 setPendingHistoryItem() 立即更新 UI
```

**handleThoughtEvent()**: 设置 thought 状态以显示思考

**handleFinishedEvent()** (lines 609-650): 处理 finish reasons 和警告

**handleCitationEvent()** (lines 594-607): 实时显示引用

**handleErrorEvent()** (lines 570-592): 向用户显示错误消息

**handleChatCompressionEvent()** (lines 652-674): 通知压缩

#### 流状态管理

```typescript
// packages/cli/src/ui/contexts/StreamingContext.tsx
enum StreamingState {
  Idle, // 未流式传输
  Responding, // 正在接收响应
  WaitingForConfirmation, // 等待工具批准
}

// 基于工具状态计算 (lines 234-255)
// - 从 isResponding, toolCalls 状态派生
// - 管理批准工作流
```

### 7.5 错误处理

#### 重试机制

**文件**: `packages/core/src/utils/retry.ts`

```typescript
// retryWithBackoff() (lines 89-215)
// - 默认: 3 次尝试, 5s 初始延迟, 30s 最大
// - 指数退避 with jitter: currentDelay * 2 每次尝试
// - 区分瞬态 (429, 5xx) 和永久错误
// - 配额错误的特殊处理 with 模型回退
```

#### 流验证和错误恢复

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// InvalidStreamError (lines 167-175)
// - 类型: NO_FINISH_REASON, NO_RESPONSE_TEXT
// - 如果流缺少 finish reason 或有空文本触发重试 (lines 570-582)

// sendMessageStream() 中的重试逻辑 (lines 265-325)
// - 线性退避重试: initialDelayMs * (attempt + 1) (line 317)
```

#### 错误事件传播

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// handleErrorEvent() 将错误存储在历史中 (line 570-592)
// - 使用 parseAndFormatApiError() 获取用户友好消息
// - 单独处理 UnauthorizedError for auth 流程
// - submitQuery() 中的 Catch block (lines 937-955) 处理流错误
```

#### 用户取消

```typescript
// Abort signal 传播 (lines 833, 1043-1050)
// - 在 abortControllerRef 中管理 AbortController
// - Escape 键触发 cancelOngoingRequest() (lines 339-350)
// - 通过 cancelAllToolCalls() 取消工具 (line 305)
// - User cancelled 事件停止处理 (lines 741, 1341-1342)
```

### 7.6 Backpressure 和流控制

#### AbortSignal-based 流控制

**文件**: `packages/core/src/utils/retry.ts`

```typescript
// 在重试前/期间检查 signal abort 状态 (lines 93-95, 124-126)
// 如果取消抛出 abort 错误 (line 125)
// 传递 signal 到 delay 函数以进行可取消等待
```

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// abortControllerRef 持有 active abort controller
// 为每个查询新创建 (line 832)
// Signal 传递到 sendMessageStream() (line 879)
// Signal 传递到 processGeminiStreamEvents() (line 885)
```

#### 工具调用调度 with Backpressure

```typescript
// packages/cli/src/ui/hooks/useReactToolScheduler.ts
// - 工具执行被调度，而非立即
// - 基于队列的处理防止压垮系统
// - onComplete 回调在所有工具完成时触发 (lines 68-69)
// - 状态转换: scheduled → validating → awaiting_approval → executing → success/error
```

#### 顺序消息处理

**文件**: `packages/core/src/core/geminiChat.ts`

```typescript
// sendPromise (line 187): 确保消息顺序处理
// 新消息等待前一个: await this.sendPromise (line 231)
// Resolver promises 链请求 (lines 233-237)
```

#### 内容缓冲和分割

**文件**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

```typescript
// geminiMessageBuffer 累积内容
// findLastSafeSplitPoint() 防止 mid-word 分割
// 在静态和动态渲染之间分割大消息 (lines 500-529)
// 防止过度重新渲染和浏览器滞后
```

### 7.7 流输出格式化

**文件**: `packages/core/src/output/stream-json-formatter.ts`

```typescript
export class StreamJsonFormatter {
  // JSONL (newline-delimited JSON) 输出

  formatEvent(event): string {
    // 转换事件为 JSON with newline (lines 20-22)
  }

  emitEvent(event): void {
    // 直接写入 stdout (lines 28-30)
  }

  // 非交互模式的实时事件流
}
```

**流事件类型** (`output/types.ts`):

- INIT, MESSAGE, TOOL_USE, TOOL_RESULT, ERROR, RESULT

---

## 8. 关键技术原理 (Key Technical Principles)

### 8.1 Async Generators 模式

#### 核心抽象

```typescript
async function* streamingFunction(): AsyncGenerator<Event> {
  for await (const chunk of apiStream) {
    const event = processChunk(chunk);
    yield event;
  }
}

// 消费
for await (const event of streamingFunction()) {
  handleEvent(event);
}
```

**优势**:

- 懒加载评估 (lazy evaluation)
- 内存效率 (不需要缓冲整个响应)
- 可组合 (generators 可以链式)
- 取消支持 (通过 AbortSignal)

**使用位置**:

- `geminiChat.ts`: `sendMessageStream()`, `processStreamResponse()`
- `turn.ts`: `Turn.run()`
- `useGeminiStream.ts`: `processGeminiStreamEvents()`

### 8.2 事件驱动架构

#### MessageBus 模式

**文件**: `packages/core/src/confirmation-bus/message-bus.ts`

```typescript
export class MessageBus {
  // 发布-订阅模式

  subscribe<T>(type: MessageBusType, handler: (message: T) => void): void;

  publish<T>(message: T & { type: MessageBusType }): void;

  // 支持请求-响应模式 with correlation IDs
}
```

**消息类型**:

- `TOOL_CONFIRMATION_REQUEST` - 请求策略决策
- `TOOL_CONFIRMATION_RESPONSE` - 策略决策响应
- `TOOL_POLICY_REJECTION` - 策略拒绝
- `TOOL_EXECUTION_SUCCESS` / `TOOL_EXECUTION_FAILURE` - 执行结果
- `UPDATE_POLICY` - 更新工具策略

**工作流**:

```
CoreToolScheduler
  ↓
发布 TOOL_CONFIRMATION_REQUEST
  ↓
PolicyEngine 订阅
  ↓
评估策略 (ALLOW, DENY, ASK_USER)
  ↓
发布 TOOL_CONFIRMATION_RESPONSE
  ↓
CoreToolScheduler 处理响应
  ↓
执行或等待用户确认
```

### 8.3 策略引擎

**文件**: `packages/core/src/policy/policy-engine.ts`

```typescript
export class PolicyEngine {
  evaluatePolicy(toolName: string, context: EvaluationContext): PolicyDecision {
    // ALLOW - 自动执行
    // DENY - 拒绝执行
    // ASK_USER - 委托给用户
  }
}
```

**策略层级**:

1. **DEFAULT** - 应用级默认策略
2. **USER** - 用户覆盖
3. **ADMIN** - 管理员强制策略

**策略存储**: `.gemini/policies.json`

### 8.4 Token 限制管理

#### 上下文窗口跟踪

**文件**: `packages/core/src/core/tokenLimits.ts`

```typescript
// 监控估计的 token 使用
// 超过阈值时触发压缩
// 管理最大 turns (每会话 100)
// 处理上下文窗口溢出事件
```

#### 压缩触发

**文件**: `packages/core/src/core/client.ts`

```typescript
startTurn() {
  // 1. 如果需要构建 IDE 上下文 parts
  // 2. 发送前附加到用户消息
  // 3. 管理 token 估计
  // 4. 如果需要触发压缩
}
```

**压缩阈值**:

- 当达到模型限制的 70% 时压缩
- 保留最后 30% 的历史
- 在用户消息边界分割

### 8.5 状态机模式

#### Tool Call 状态机

```
┌──────────────┐
│  validating  │ ◄── 初始状态
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  scheduled   │ ◄── 准备执行
└──────┬───────┘
       │
       ├──────────────────┐
       │                  │
       ▼                  ▼
┌──────────────┐   ┌─────────────────┐
│  executing   │   │ awaiting_approval│
└──────┬───────┘   └────────┬─────────┘
       │                    │
       │                    │ (用户批准)
       │◄───────────────────┘
       │
       ├────────────┬──────────────┐
       │            │              │
       ▼            ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│ success  │ │  error   │ │  cancelled   │
└──────────┘ └──────────┘ └──────────────┘
```

**转换逻辑**: `CoreToolScheduler.setStatusInternal()`

### 8.6 Registry 模式

#### 可扩展注册表

**共同接口**:

```typescript
interface Registry<T> {
  register(item: T): void;
  get(name: string): T | undefined;
  getAll(): T[];
}
```

**实现**:

- `ToolRegistry` - 工具注册
- `AgentRegistry` - Agent 注册
- `PromptRegistry` - MCP prompt 注册
- `CommandService` - 命令注册

**优势**:

- 动态发现
- 插件支持
- 冲突解决
- 类型安全

### 8.7 模板系统

#### 字符串模板

**文件**: `packages/core/src/agents/utils.ts`

```typescript
export function templateString(
  template: string,
  inputs: Record<string, string>,
): string {
  // 替换 ${key} with inputs[key]
  // 验证所有必需键存在
  // 缺少输入时抛出
}
```

**使用**:

- Agent system prompts
- Agent query templates
- Agent initial messages
- Dynamic prompt 生成

---

## 9. 代码路径索引 (Code Path Index)

### 9.1 核心流程文件

| 功能            | 文件路径                                             | 行数     | 关键内容                                     |
| --------------- | ---------------------------------------------------- | -------- | -------------------------------------------- |
| **CLI 入口**    | `packages/cli/index.ts`                              | 14       | main() 函数                                  |
| **React UI**    | `packages/cli/src/gemini.tsx`                        | 50       | Gemini 组件                                  |
| **输入捕获**    | `packages/cli/src/ui/components/InputPrompt.tsx`     | 359      | handleInput()                                |
| **消息提交**    | `packages/cli/src/ui/AppContainer.tsx`               | 713      | handleFinalSubmit()                          |
| **查询路由**    | `packages/cli/src/ui/hooks/useGeminiStream.ts`       | 805      | submitQuery()                                |
| **命令处理**    | `packages/cli/src/ui/hooks/slashCommandProcessor.ts` | 339      | command.action()                             |
| **Chat 初始化** | `packages/core/src/core/client.ts`                   | 393      | startChat()                                  |
| **Turn 执行**   | `packages/core/src/core/turn.ts`                     | 229      | Turn.run()                                   |
| **API 调用**    | `packages/core/src/core/geminiChat.ts`               | 226, 367 | sendMessageStream(), generateContentStream() |

### 9.2 Prompt 处理文件

| 功能            | 文件路径                                               | 关键函数                                          |
| --------------- | ------------------------------------------------------ | ------------------------------------------------- |
| **系统 Prompt** | `packages/core/src/core/prompts.ts`                    | getCoreSystemPrompt(), getCompressionPrompt()     |
| **环境上下文**  | `packages/core/src/utils/environmentContext.ts`        | getInitialChatHistory(), getEnvironmentContext()  |
| **IDE 上下文**  | `packages/core/src/ide/ideContext.ts`                  | ideContextStore, getIdeContextParts()             |
| **历史压缩**    | `packages/core/src/services/chatCompressionService.ts` | extractCuratedHistory(), findCompressSplitPoint() |
| **Part 转换**   | `packages/core/src/code_assist/converter.ts`           | toContents(), toParts(), toContent()              |
| **Part 工具**   | `packages/core/src/utils/partUtils.ts`                 | partToString(), getResponseText()                 |

### 9.3 Agent 系统文件

| 功能             | 文件路径                                            | 行数 | 关键内容                         |
| ---------------- | --------------------------------------------------- | ---- | -------------------------------- |
| **Agent 类型**   | `packages/core/src/agents/types.ts`                 | 169  | AgentDefinition 接口             |
| **Agent 注册表** | `packages/core/src/agents/registry.ts`              | 102  | AgentRegistry 类                 |
| **Agent 执行器** | `packages/core/src/agents/executor.ts`              | 772  | AgentExecutor 类                 |
| **Agent 调用**   | `packages/core/src/agents/invocation.ts`            | 138  | SubagentInvocation 类            |
| **Tool 包装器**  | `packages/core/src/agents/subagent-tool-wrapper.ts` | 79   | SubagentToolWrapper 类           |
| **Schema 工具**  | `packages/core/src/agents/schema-utils.ts`          | 91   | convertInputConfigToJsonSchema() |
| **模板工具**     | `packages/core/src/agents/utils.ts`                 | 44   | templateString()                 |
| **内置 Agent**   | `packages/core/src/agents/codebase-investigator.ts` | -    | CodebaseInvestigatorAgent        |

### 9.4 Tool 系统文件

| 功能            | 文件路径                                        | 行数 | 关键内容                        |
| --------------- | ----------------------------------------------- | ---- | ------------------------------- |
| **Tool 调度器** | `packages/core/src/core/coreToolScheduler.ts`   | 1396 | CoreToolScheduler 类            |
| **Tool 接口**   | `packages/core/src/tools/tools.ts`              | 716  | ToolInvocation, DeclarativeTool |
| **Tool 注册表** | `packages/core/src/tools/tool-registry.ts`      | 489  | ToolRegistry 类                 |
| **Tool 名称**   | `packages/core/src/tools/tool-names.ts`         | -    | 常量定义                        |
| **Tool 错误**   | `packages/core/src/tools/tool-error.ts`         | -    | ToolErrorType enum              |
| **MCP Manager** | `packages/core/src/tools/mcp-client-manager.ts` | 112  | McpClientManager                |
| **MCP Client**  | `packages/core/src/tools/mcp-client.ts`         | 300+ | McpClient 类                    |
| **MCP Tool**    | `packages/core/src/tools/mcp-tool.ts`           | -    | DiscoveredMCPTool               |

### 9.5 内置工具文件

| 工具           | 文件路径                                     | 功能                       |
| -------------- | -------------------------------------------- | -------------------------- |
| **ReadFile**   | `packages/core/src/tools/read-file.ts`       | 文件读取 with offset/limit |
| **Shell**      | `packages/core/src/tools/shell.ts`           | Shell 命令执行             |
| **Edit**       | `packages/core/src/tools/edit.ts`            | 文件编辑 with diff 预览    |
| **Glob**       | `packages/core/src/tools/glob.ts`            | 文件模式匹配               |
| **Grep**       | `packages/core/src/tools/grep.ts`            | 内容搜索                   |
| **LS**         | `packages/core/src/tools/ls.ts`              | 目录列表                   |
| **WriteFile**  | `packages/core/src/tools/write-file.ts`      | 文件写入                   |
| **WebSearch**  | `packages/core/src/tools/web-search.ts`      | Google 搜索                |
| **WebFetch**   | `packages/core/src/tools/web-fetch.ts`       | URL 内容抓取               |
| **ReadMany**   | `packages/core/src/tools/read-many-files.ts` | 批量文件读取               |
| **WriteTodos** | `packages/core/src/tools/write-todos.ts`     | Todo 列表管理              |
| **Memory**     | `packages/core/src/tools/memory.ts`          | 持久内存/上下文            |

### 9.6 流式处理文件

| 功能              | 文件路径                                            | 行数 | 关键内容                                       |
| ----------------- | --------------------------------------------------- | ---- | ---------------------------------------------- |
| **GeminiChat 流** | `packages/core/src/core/geminiChat.ts`              | 654  | sendMessageStream(), processStreamResponse()   |
| **Turn 处理**     | `packages/core/src/core/turn.ts`                    | 402  | Turn.run()                                     |
| **UI 流 Hook**    | `packages/cli/src/ui/hooks/useGeminiStream.ts`      | 1290 | useGeminiStream(), processGeminiStreamEvents() |
| **重试逻辑**      | `packages/core/src/utils/retry.ts`                  | 283  | retryWithBackoff()                             |
| **流格式化**      | `packages/core/src/output/stream-json-formatter.ts` | 63   | StreamJsonFormatter                            |
| **输出类型**      | `packages/core/src/output/types.ts`                 | 104  | 流事件类型                                     |
| **流上下文**      | `packages/cli/src/ui/contexts/StreamingContext.tsx` | 23   | StreamingContext                               |

### 9.7 配置和扩展文件

| 功能                   | 文件路径                                                  | 关键内容             |
| ---------------------- | --------------------------------------------------------- | -------------------- |
| **Config**             | `packages/core/src/config/config.ts`                      | Config 类 (主配置)   |
| **Storage**            | `packages/core/src/config/storage.ts`                     | 配置存储             |
| **Models**             | `packages/core/src/config/models.ts`                      | 模型定义             |
| **Extension Manager**  | `packages/cli/src/config/extension-manager.ts`            | ExtensionManager 类  |
| **Extension Loader**   | `packages/core/src/utils/extensionLoader.ts`              | ExtensionLoader 接口 |
| **Extension Settings** | `packages/cli/src/config/extensions/extensionSettings.ts` | 扩展设置             |
| **GitHub Integration** | `packages/cli/src/config/extensions/github.ts`            | GitHub 扩展源        |
| **Policy**             | `packages/cli/src/config/policy.ts`                       | 策略加载             |
| **PolicyEngine**       | `packages/core/src/policy/policy-engine.ts`               | PolicyEngine 类      |

### 9.8 服务文件

| 服务                 | 文件路径                                               | 功能       |
| -------------------- | ------------------------------------------------------ | ---------- |
| **Git Service**      | `packages/core/src/services/gitService.ts`             | Git 操作   |
| **File Discovery**   | `packages/core/src/services/fileDiscoveryService.ts`   | 文件发现   |
| **Shell Execution**  | `packages/core/src/services/shellExecutionService.ts`  | Shell 执行 |
| **Chat Recording**   | `packages/core/src/services/chatRecordingService.ts`   | 聊天记录   |
| **Chat Compression** | `packages/core/src/services/chatCompressionService.ts` | 历史压缩   |
| **Model Router**     | `packages/core/src/services/modelRouterService.ts`     | 模型路由   |

### 9.9 A2A Server 文件

| 功能                | 文件路径                                     | 关键内容           |
| ------------------- | -------------------------------------------- | ------------------ |
| **Express App**     | `packages/a2a-server/src/http/app.ts`        | A2AExpressApp      |
| **Server**          | `packages/a2a-server/src/http/server.ts`     | HTTP 服务器        |
| **Endpoints**       | `packages/a2a-server/src/http/endpoints.ts`  | API 端点           |
| **Agent Executor**  | `packages/a2a-server/src/agent/executor.ts`  | CoderAgentExecutor |
| **Task**            | `packages/a2a-server/src/agent/task.ts`      | Task 类 (600+ 行)  |
| **Config**          | `packages/a2a-server/src/config/config.ts`   | 服务器配置         |
| **GCS Persistence** | `packages/a2a-server/src/persistence/gcs.ts` | GCS 存储           |

### 9.10 关键工具和实用程序

| 功能                  | 文件路径                                            | 用途                    |
| --------------------- | --------------------------------------------------- | ----------------------- |
| **Events**            | `packages/core/src/utils/events.js`                 | coreEvents EventEmitter |
| **MessageBus**        | `packages/core/src/confirmation-bus/message-bus.ts` | 事件总线                |
| **Telemetry**         | `packages/core/src/telemetry/`                      | 日志和遥测              |
| **Error Parsing**     | `packages/core/src/utils/errorParsing.ts`           | API 错误解析            |
| **File Utils**        | `packages/core/src/utils/fileUtils.ts`              | 文件操作                |
| **Path Utils**        | `packages/core/src/utils/pathUtils.ts`              | 路径处理                |
| **Content Generator** | `packages/core/src/core/contentGenerator.ts`        | LLM 接口                |
| **Base LLM Client**   | `packages/core/src/core/baseLlmClient.ts`           | 实用 LLM 客户端         |

---

## 附录: 快速参考

### A. 常见调试位置

| 场景           | 文件                       | 行号 | 断点说明              |
| -------------- | -------------------------- | ---- | --------------------- |
| **用户输入**   | `InputPrompt.tsx`          | 359  | 键盘输入处理          |
| **消息提交**   | `AppContainer.tsx`         | 713  | handleFinalSubmit     |
| **命令路由**   | `useGeminiStream.ts`       | 805  | submitQuery 路由      |
| **斜杠命令**   | `slashCommandProcessor.ts` | 339  | command.action()      |
| **API 调用**   | `geminiChat.ts`            | 367  | generateContentStream |
| **Tool 调度**  | `coreToolScheduler.ts`     | 668  | schedule()            |
| **Agent 执行** | `agents/executor.ts`       | 156  | run()                 |

### B. 配置文件位置

| 配置            | 路径                        | 说明             |
| --------------- | --------------------------- | ---------------- |
| **系统 Prompt** | `~/.gemini/system.md`       | 自定义系统提示词 |
| **扩展配置**    | `~/.gemini/extensions.json` | 扩展列表         |
| **策略**        | `~/.gemini/policies.json`   | 工具执行策略     |
| **设置**        | `~/.gemini/settings.json`   | 用户设置         |
| **Auth**        | `~/.gemini/auth.json`       | 认证令牌         |

### C. 环境变量

| 变量                 | 默认值      | 用途                   |
| -------------------- | ----------- | ---------------------- |
| `GEMINI_SYSTEM_MD`   | -           | 自定义 system.md 路径  |
| `GEMINI_SYSTEM_DIR`  | `~/.gemini` | 配置目录               |
| `GEMINI_SANDBOX`     | -           | 启用/禁用沙箱          |
| `GEMINI_DEV_TRACING` | -           | 启用开发追踪           |
| `NODE_ENV`           | -           | development/production |

### D. 关键命令

| 命令                                   | 用途           |
| -------------------------------------- | -------------- |
| `npm run build`                        | 构建所有包     |
| `npm run test`                         | 运行测试       |
| `npm run debug`                        | 调试模式       |
| `npm run telemetry -- --target=genkit` | 启动 Genkit UI |
| `npm run telemetry -- --target=local`  | 启动 Jaeger    |

---

**文档维护**: 此文档应随代码变化更新。最后更新: 2025-10-31

**贡献**: 欢迎提交 PR 更新此文档以反映架构变化。

**反馈**: 如有问题或建议，请在 GitHub Issues 中提出。
