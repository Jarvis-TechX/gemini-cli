# Mac VSCode 调试步骤（图文版）

## 🚀 快速开始（3 分钟上手）

### 步骤 1：打开终端并构建项目

在 VSCode 中：

1. 按 `Control + ~`（波浪号键）打开集成终端
2. 确认路径：

```bash
pwd
# 应该显示：/Users/mingyuye/AIDevelop/gemini-cli/gemini-cli
```

3. 构建项目：

```bash
npm run build
```

等待构建完成（大约 10-30 秒）

---

### 步骤 2：设置断点

1. **打开文件管理器**：点击左侧文件图标（或按 `Cmd + Shift + E`）

2. **导航到文件**：

   ```
   packages
     └── cli
         └── src
             └── gemini.tsx
   ```

3. **设置断点**：
   - 找到第 50 行左右（`export const Gemini` 函数内部）
   - 点击行号左边的灰色区域
   - 出现红色圆点 🔴 表示断点已设置

**示例位置**：

```typescript
export const Gemini = ({ args }: GeminiProps) => {
  // 👈 在这行设置断点
  const [state, setState] = useState(initialState);
  // ...
};
```

---

### 步骤 3：打开调试面板

**方法 A（推荐）**：

- 点击左侧活动栏的 **调试图标** 🐛

**方法 B（快捷键）**：

- 按 `Cmd + Shift + D`

你会看到左侧出现调试面板。

---

### 步骤 4：选择调试配置

在调试面板顶部，你会看到一个下拉菜单：

```
▼ Build & Launch CLI    ▶️
```

确保选中 **"Build & Launch CLI"**

如果看不到，点击下拉菜单，选择这个配置。

---

### 步骤 5：启动调试

**方法 A（快捷键）**：

```
按 F5
（如果不工作，试试 fn + F5）
```

**方法 B（点击）**：

- 点击绿色播放按钮 ▶️

**方法 C（菜单）**：

- 顶部菜单 → 运行 → 启动调试

---

### 步骤 6：观察调试启动

你会看到：

1. **底部状态栏变橙色**：表示正在调试
2. **顶部出现调试工具栏**：
   ```
   ⏸️ 继续  ⏯️ 单步跳过  ⏭️ 单步进入  ⏮️ 单步跳出  🔄 重启  ⏹️ 停止
   ```
3. **终端输出构建信息**
4. **程序在断点处暂停**（如果执行到了你的断点）

---

### 步骤 7：使用调试功能

#### 查看变量

- 左侧调试面板显示：
  - **变量**：当前作用域的所有变量
  - **监视**：你添加的监视表达式
  - **调用堆栈**：函数调用链

#### 鼠标悬停查看值

- 把鼠标放在代码中的变量上，会显示当前值

#### 控制执行流程

- `F5` 或 ▶️ **继续**：运行到下一个断点
- `F10` 或 ⏯️ **单步跳过**：执行当前行，不进入函数
- `F11` 或 ⏭️ **单步进入**：进入函数内部
- `Shift + F11` 或 ⏮️ **单步跳出**：跳出当前函数

#### 在调试控制台执行代码

- 打开调试控制台（底部面板 → "调试控制台" 标签）
- 输入表达式查看值：
  ```javascript
  > args
  > state.currentMessage
  ```

---

## 🎓 实战演练

### 练习 1：调试 CLI 启动流程

**目标**：观察 CLI 启动时的参数

1. 在 `packages/cli/src/gemini.tsx` 第 50 行设置断点：

```typescript
export const Gemini = ({ args }: GeminiProps) => {
  // 👈 断点在这里
  const [state, setState] = useState(initialState);
```

2. 按 `F5` 启动调试

3. 程序暂停后：
   - 在左侧"变量"面板查看 `args` 的值
   - 在调试控制台输入 `args` 查看详细信息
   - 按 `F10` 单步执行几次，观察执行流程

4. 按 `F5` 继续运行

---

### 练习 2：调试 API 调用

**目标**：查看发送给 Gemini API 的请求

1. 在 `packages/core/src/core/geminiChat.ts` 找到 `sendMessage` 函数

```typescript
async sendMessage(message: string) {
  // 👈 在这里设置断点
  const response = await this.client.generateContent({
```

2. 按 `F5` 启动调试

3. 在 CLI 中输入一个问题

4. 程序会在断点处暂停：
   - 查看 `message` 参数
   - 按 `F11` 进入 `generateContent` 函数
   - 查看完整的 API 请求对象

---

### 练习 3：调试测试

**目标**：调试单元测试

1. 打开测试文件：`packages/cli/src/gemini.test.tsx`

2. 在某个测试中设置断点：

```typescript
it('should render correctly', () => {
  // 👈 在这里设置断点
  const { lastFrame } = render(<Gemini args={mockArgs} />);
```

3. 按 `F5` → 选择 **"Debug Test File"**

4. 输入文件路径或直接回车使用默认值

5. 观察测试执行过程

---

## 🔥 进阶技巧

### 1. 条件断点

**适用场景**：只想在特定条件下暂停

**设置方法**：

1. 右键点击红色断点
2. 选择 "编辑断点"
3. 输入条件，例如：
   ```javascript
   args.prompt && args.prompt.includes('debug');
   ```
4. 现在只有当条件为 true 时才会暂停

---

### 2. 日志点（不暂停，只输出）

**适用场景**：想看变量值但不想暂停执行

**设置方法**：

1. 右键点击行号区域
2. 选择 "添加日志点"
3. 输入表达式，例如：
   ```javascript
   args.prompt: {args.prompt}
   ```
4. 运行时会在调试控制台输出，但不暂停

---

### 3. 监视表达式

**适用场景**：持续监视某个变量的变化

**设置方法**：

1. 在左侧调试面板找到 "监视" 区域
2. 点击 "+" 号
3. 输入表达式，例如：
   ```javascript
   state.isLoading;
   args.model;
   this.client.apiKey;
   ```
4. 调试时会实时显示这些值

---

### 4. 异常断点

**适用场景**：程序抛出错误时自动暂停

**设置方法**：

1. 在左侧调试面板找到 "断点" 区域
2. 勾选 "Uncaught Exceptions"（未捕获的异常）
3. 可选勾选 "Caught Exceptions"（已捕获的异常）

---

## ⚡ Mac 专用快捷键

| 操作         | Mac 快捷键          | Windows 快捷键      |
| ------------ | ------------------- | ------------------- |
| 启动调试     | `fn + F5` 或 `F5`   | `F5`                |
| 打开调试面板 | `Cmd + Shift + D`   | `Ctrl + Shift + D`  |
| 切换断点     | `fn + F9` 或 `F9`   | `F9`                |
| 单步跳过     | `fn + F10` 或 `F10` | `F10`               |
| 单步进入     | `fn + F11` 或 `F11` | `F11`               |
| 单步跳出     | `fn + Shift + F11`  | `Shift + F11`       |
| 重启调试     | `Cmd + Shift + F5`  | `Ctrl + Shift + F5` |
| 停止调试     | `Shift + F5`        | `Shift + F5`        |
| 打开终端     | `Control + ~`       | `Ctrl + ~`          |
| 命令面板     | `Cmd + Shift + P`   | `Ctrl + Shift + P`  |

**注意**：某些 Mac 键盘需要按 `fn` 键配合功能键。

---

## 🐛 常见问题解决

### Q1: 按 F5 后没反应

**解决方案 1**：检查 Mac 功能键设置

```bash
# 按这个组合键试试
fn + F5
```

**解决方案 2**：使用菜单

- 顶部菜单 → 运行 → 启动调试

**解决方案 3**：检查调试配置

1. 点击左侧调试图标
2. 确保下拉菜单显示 "Build & Launch CLI"

---

### Q2: 断点显示灰色空心圆，不是红色实心圆

**原因**：源代码和编译后代码不匹配

**解决方案**：

```bash
# 重新构建
npm run build

# 然后重启调试
# 按 Shift + F5 停止
# 按 F5 重新启动
```

---

### Q3: 调试时看不到变量值

**原因**：代码被优化或 source map 未生成

**解决方案**：

1. 检查 `tsconfig.json` 中 `"sourceMap": true`
2. 重新构建：
   ```bash
   npm run clean
   npm install
   npm run build
   ```

---

### Q4: 终端报错 "Cannot find module"

**解决方案**：

```bash
# 清理并重装依赖
npm run clean
npm install

# 重新构建
npm run build
```

---

### Q5: 想调试但沙箱干扰

**解决方案**：临时禁用沙箱

```bash
# 在启动前设置环境变量
export GEMINI_SANDBOX=false
npm run build
```

或修改 `.vscode/launch.json` 中的配置（已经默认禁用了）。

---

## 📚 推荐的学习路径

### 第 1 天：熟悉基础

- ✅ 设置断点并启动调试
- ✅ 使用 F10 单步跳过
- ✅ 查看变量面板

### 第 2 天：深入调试

- ✅ 使用 F11 单步进入函数
- ✅ 在调试控制台执行表达式
- ✅ 查看调用堆栈

### 第 3 天：高级技巧

- ✅ 设置条件断点
- ✅ 使用监视表达式
- ✅ 调试单元测试

### 第 4 天：专业调试

- ✅ 使用开发追踪（Dev Tracing）
- ✅ 结合 React DevTools
- ✅ 性能分析

---

## 🎯 下一步

现在你已经知道如何调试了！试试这个：

1. **打开终端**：`Control + ~`
2. **构建项目**：`npm run build`
3. **设置断点**：在 `packages/cli/src/gemini.tsx` 第 50 行
4. **启动调试**：按 `F5`

🎉 开始你的调试之旅吧！
