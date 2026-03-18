---
description: How to CLI-ify and automate any Electron Desktop Application via CDP
---

# CLI-ifying Electron Applications (Skill Guide)

Based on the successful extraction and automation of **Antigravity** and **OpenAI Codex** desktop apps, this guide serves as the standard operating procedure (SOP) for adapting ANY Electron-based application into an OpenCLI adapter.

## 核心原理 (Core Concept)
Electron 应用本质上是运行在本地的 Chromium 浏览器实例。只要在启动应用时暴露了调试端口（CDP，Chrome DevTools Protocol），我们就可以利用 Playwright MCP 直接穿透其 UI 层，获取并操控包括 React/Vue 组件、Shadow DOM 等在内的所有底层状态，实现“从应用外挂入自动化脚本”。

### 启动 Target App 
要在本地操作任何 Electron 应用，必须先要求用户使用以下参数注入调试端点：
```bash
/Applications/AppName.app/Contents/MacOS/AppName --remote-debugging-port=9222
```

## 标准适配模式：The 5-Command Pattern

适配一个新的 App，必须在 `src/clis/<app_name>/` 下实现这 5 个标准化指令：

### 1. `status.ts` (连接测试)
负责确认应用监听正确。
- **机制**: 直接 `export const statusCommand = cli({...})`
- **核心代码**: 获取 `window.location.href` 与 `document.title`。
- **注意**: 必须指明 `domain: 'localhost'` 和 `browser: true`。

### 2. `dump.ts` (逆向工程核心)
很多现代 App DOM 极其庞大且混淆。**千万不要直接猜选择器**。
首先编写 dump 脚本，将当前页面的 DOM 与 Accessibility Tree 导出到 `/tmp/`，方便用 AI (或者 `grep`) 提取精确的容器名称和 Class。
```typescript
const dom = await page.evaluate('document.body.innerHTML');
fs.writeFileSync('/tmp/app-dom.html', dom);
const snap = await page.snapshot({ interactive: false });
fs.writeFileSync('/tmp/app-snapshot.json', JSON.stringify(snap, null, 2));
```

### 3. `send.ts` (高级注入技巧)
Electron 应用常常使用极端复杂的富文本编辑器（如 Monaco, Lexical, ProseMirror）。直接修改元素的 `value` 常常会被 React 状态机忽略。
- **最佳实践**: 使用 `document.execCommand('insertText')` 完美模拟真实的人类复制粘贴输入流，完全穿透 React state。
```javascript
// 寻路机制：优先尝试寻找 contenteditable
let composer = document.querySelector('[contenteditable="true"]');
composer.focus();
document.execCommand('insertText', false, "你好");
```
- **提交快捷键**: `await page.pressKey('Enter')`。

### 4. `read.ts` (上下文解析)
不要提取整个页面的文本。应该利用 `dump.ts` 抓取出来的特征寻找真正的“对话容器”。
- **技巧**: 检查带有语义化结构的数据，例如 `[role="log"]`、`[data-testid="conversation"]` 或是 `[data-content-search-turn-key]`。
- **格式化**: 拼接抓取出的文本转粗暴渲染成 Markdown 返回，这样不仅你和人类能读懂，LLM 后续作为 Agent 也能精准切分。

### 5. `new.ts` / Action Macros (底层事件模拟)
许多图形界面操作难以找到按钮实例，但它们通常响应原生快捷键。
- **最佳实践**: 模拟系统级快捷键直接驱动 `(Meta+N / Control+N)`。
```typescript
const isMac = process.platform === 'darwin';
await page.pressKey(isMac ? 'Meta+N' : 'Control+N');
await page.wait(1); // 等待重渲染
```

## 全局环境变量
为了让 Core Framework 挂载到我们指定的端口，必须在执行指令前（或在 README 中指导用户）注入目标环境端口：
```bash
export OPENCLI_CDP_ENDPOINT="http://127.0.0.1:9222"
```

## 踩坑避雷 (Pitfalls & Gotchas)
1. **端口占用 (EADDRINUSE)**: 确保同一时间只能有一个 App 占据一个端口。如果同时测试 Antigravity (9224) 且你要测试别的 App (9222)，要将 CDP Endpoint 分配开来。
2. **TypeScript 抽象**: OpenCLI 内部封装了 `IPage` 类型（`src/types.ts`），不是原生的 Playwright Page。要用 `page.pressKey()` 和 `page.evaluate()`，而非 `page.keyboard.press()`。
3. **延时等待**: DOM 发生剧烈变化后，一定要加上 `await page.wait(0.5)` 到 `1.0` 给框架反应的时间。不要立刻 return 导致连接 prematurely 阻断。
