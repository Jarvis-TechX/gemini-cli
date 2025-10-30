# Gemini CLI 调试指南

## 快速开始

### 1. 最简单的调试方式（推荐新手）

1. 在代码中设置断点（点击行号左侧）
2. 按 `F5` → 选择 **"Build & Launch CLI"**
3. 程序会在断点处暂停，可以查看变量、单步执行

### 2. 调试快捷键

| 快捷键          | 功能      | 说明                           |
| --------------- | --------- | ------------------------------ |
| `F5`            | 启动/继续 | 开始调试或继续执行到下一个断点 |
| `F9`            | 切换断点  | 在当前行添加/删除断点          |
| `F10`           | 单步跳过  | 执行当前行，不进入函数内部     |
| `F11`           | 单步进入  | 进入函数内部调试               |
| `Shift+F11`     | 单步跳出  | 跳出当前函数                   |
| `Ctrl+Shift+F5` | 重启调试  | 重新开始调试会话               |
| `Shift+F5`      | 停止调试  | 终止调试会话                   |

## 调试场景

### 场景 1：调试 CLI 主程序

**目标**：调试用户输入、命令处理、UI 渲染等

```typescript
// 在 packages/cli/src/gemini.tsx 中设置断点
export const Gemini = ({ args }: GeminiProps) => {
  // 在这里设置断点，查看 args 参数
  console.log('Debug: args =', args);
  // ...
};
```

**启动方式**：

- VS Code: `F5` → "Build & Launch CLI"
- 命令行: `npm run debug`（然后使用 "Attach" 配置）

### 场景 2：调试 API 调用和 Tool 执行

**目标**：调试 Gemini API 交互、工具调度

```typescript
// 在 packages/core/src/core/coreToolScheduler.ts 中设置断点
export class CoreToolScheduler {
  async executeTool(toolName: string, args: unknown) {
    // 在这里设置断点，查看工具执行流程
    debugger; // 代码中的硬断点
    // ...
  }
}
```

### 场景 3：调试单元测试

**目标**：调试测试失败的原因

1. 打开测试文件（如 `packages/cli/src/gemini.test.tsx`）
2. 在测试中设置断点：

```typescript
it('should render correctly', () => {
  // 设置断点
  const { lastFrame } = render(<Gemini args={mockArgs} />);
  expect(lastFrame()).toMatchSnapshot();
});
```

3. `F5` → "Debug Test File"

**快捷方式**：

```bash
# 调试单个测试
npx vitest run --inspect-brk=9229 packages/cli/src/gemini.test.tsx

# 调试匹配的测试
npx vitest run --inspect-brk=9229 -t "should render correctly"
```

### 场景 4：调试集成测试

**目标**：端到端调试完整流程

```bash
# 设置环境变量禁用沙箱（方便调试）
export GEMINI_SANDBOX=false

# 调试特定集成测试
npx vitest run --root ./integration-tests --inspect-brk=9229 file-system.test.ts
```

### 场景 5：调试工具（Tool）实现

**目标**：调试文件操作、Shell 命令等工具

```typescript
// 在 packages/core/src/core/tools/readFile.ts 中设置断点
export async function readFile(path: string) {
  // 设置断点
  const content = await fs.readFile(path, 'utf-8');
  return content;
}
```

## 高级调试技巧

### 1. 条件断点

右键点击断点 → "编辑断点" → 添加条件：

```javascript
// 只在特定条件下暂停
args.length > 0;
// 或
toolName === 'readFile';
```

### 2. 日志点（Logpoint）

右键点击行号 → "添加日志点"：

```javascript
console.log('Variable value:', myVar);
```

不会暂停执行，只输出日志。

### 3. 调用堆栈查看

在调试模式下，左侧"调用堆栈"面板显示函数调用链：

```
gemini.tsx:45
  ← client.ts:123
    ← coreToolScheduler.ts:67
```

点击任意层级可以查看该层的变量。

### 4. 监视表达式

在"监视"面板添加表达式，实时查看值的变化：

```javascript
args.prompt;
this.state.currentTool;
response.data.length;
```

### 5. 调试控制台

在暂停时，可以在调试控制台执行代码：

```javascript
// 查看变量
> args
> this.props

// 执行函数
> this.formatResponse(data)

// 修改变量（小心使用）
> args.verbose = true
```

## 开发追踪（Dev Tracing）

### 使用 Genkit UI 可视化调试

```bash
# 终端 1：启动 Genkit UI
npm run telemetry -- --target=genkit
# 访问 http://localhost:4000

# 终端 2：启动 CLI 并启用追踪
GEMINI_DEV_TRACING=true npm start
```

**优势**：

- 可视化查看 API 调用流程
- 查看完整的请求/响应数据
- 追踪工具执行时间线

### 使用 Jaeger 追踪

```bash
# 终端 1：启动 Jaeger
npm run telemetry -- --target=local
# 访问 http://localhost:16686

# 终端 2：运行 CLI
GEMINI_DEV_TRACING=true npm start
```

## 调试 React UI（Ink）

### 使用 React DevTools

```bash
# 终端 1：启动开发模式
DEV=true npm start

# 终端 2：启动 React DevTools
npx react-devtools@4.28.5
```

### 查看组件树和 Props

在 React DevTools 中可以：

- 查看组件层级结构
- 实时查看/修改 Props 和 State
- 追踪组件重新渲染

## 常见问题

### Q1: 断点不生效？

**解决方案**：

1. 确保启用了 source maps：`tsconfig.json` 中 `"sourceMap": true`
2. 重新构建：`npm run build`
3. 清理缓存：`npm run clean && npm install`

### Q2: 无法附加到进程？

**解决方案**：

```bash
# 确保使用正确的调试端口
npm run debug  # 默认 9229 端口

# 检查端口是否被占用
lsof -i :9229
```

### Q3: 在 Sandbox 中调试？

**解决方案**：

```bash
# 方法 1：禁用沙箱
GEMINI_SANDBOX=false npm start

# 方法 2：在沙箱内调试
DEBUG=1 gemini
# 然后使用 "Attach" 配置，remoteRoot 会自动映射
```

### Q4: 测试调试时变量显示 undefined？

**解决方案**：

- 检查是否使用了 `vi.mock()` 模拟了模块
- 在 `beforeEach` 中确保 mock 正确设置
- 使用 `console.log()` 确认执行流程

## 推荐的调试工作流

### 日常开发调试流程

1. **启动 Watch 模式**（自动重新编译）：

```bash
# 在一个终端保持运行
npm run build -- --watch
```

2. **在 VS Code 中调试**：
   - 设置断点
   - `F5` 启动
   - 修改代码后 `Ctrl+Shift+F5` 重启

3. **使用开发追踪**（复杂问题）：

```bash
# 启动追踪
npm run telemetry -- --target=genkit

# 在另一终端
GEMINI_DEV_TRACING=true npm start
```

### Bug 修复流程

1. **编写失败的测试**：

```typescript
it('should handle edge case', () => {
  // 设置断点
  const result = myFunction(edgeCaseInput);
  expect(result).toBe(expectedOutput);
});
```

2. **调试测试**：
   - `F5` → "Debug Test File"
   - 单步执行找到问题

3. **修复代码**：
   - 在源代码中设置断点
   - 验证修复

4. **运行完整测试套件**：

```bash
npm run test
```

## 性能调试

### 使用 Chrome DevTools

```bash
# 启动带 inspect 的 CLI
node --inspect dist/index.js

# 在 Chrome 打开
chrome://inspect
```

可以使用：

- Performance 面板分析性能
- Memory 面板检查内存泄漏
- Profiler 分析 CPU 使用

### 使用 VS Code Profiler

1. 启动调试
2. 点击调试工具栏的 "Take Performance Profile"
3. 执行操作
4. 停止录制，查看火焰图

## 参考资源

- [VS Code 调试文档](https://code.visualstudio.com/docs/editor/debugging)
- [Node.js 调试指南](https://nodejs.org/en/docs/guides/debugging-getting-started/)
- [Vitest 调试](https://vitest.dev/guide/debugging.html)
- [React DevTools](https://react.dev/learn/react-developer-tools)
