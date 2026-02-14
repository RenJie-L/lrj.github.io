---
title: AI 编码最佳实践 2.0
date: 2026-02-14 14:00:20
tags: [学习, AI]
---

# AI 编码最佳实践 2.0

> 基于 [Cursor Agent 官方指南](https://cursor.com/cn/blog/agent-best-practices) 的总结

---

## 一、了解 Agent Harness

Agent 建立在三个组件之上：

1. **Instructions**：引导 agent 行为的 system prompt 和规则  
2. **Tools**：文件编辑、代码库搜索、终端执行等工具  
3. **User messages**：你用来指挥工作的提示词和后续交互  

Cursor 会为每个模型专门调整 instructions 和工具。不同模型对同一提示响应方式不同（有的更偏好 grep，有的需要明确要求在编辑后调用 linter），harness 帮你处理这些差异，让你专注构建软件本身。

---

## 二、从规划开始

> 你能做出的最大改变：**在编写代码之前先进行规划**。

芝加哥大学研究发现，有经验的开发者更倾向于在生成代码之前先规划。规划让你更清晰自己要构建什么，并为 Agent 提供明确目标。

### 2.1 使用 Plan 模式

在 agent 输入框按 **`Shift+Tab`** 切换 Plan 模式。Agent 不会立即写代码，而是：

1. 分析代码库以查找相关文件  
2. 就你的需求提出澄清问题  
3. 创建包含文件路径和代码引用的详细实现计划  
4. 在开始构建前**等待你的确认**  

计划以 Markdown 打开，可直接编辑、删减、补充。

**提示**：点击 "Save to workspace" 可保存到 `.cursor/plans/`，方便团队协作和中断后续写。

**不需要计划的场景**：简单小改动、已做过多次的任务，直接交给 agent 即可。

### 2.2 从计划重新开始

若 Agent 生成的内容不符合预期，**不要反复修补提示**，而应：

1. 回滚改动  
2. 重新细化计划，把需求描述得更具体  
3. 再执行一次  

这通常比「修一个进行中的 agent」更快、结果更干净。

---

## 三、管理上下文

随着 Agent 编写代码越来越多，你的主要工作变成：**为每个 agent 提供完成其任务所需的上下文**。

### 3.1 让 Agent 自行获取上下文

不必在提示里手动 @ 每一个文件。Agent 具备 grep 和语义搜索，会按需查找。

**例如**：问 "the authentication flow"（身份验证流程）时，Agent 会搜索到相关文件，即使你的提示里没有这些词。

**原则**：
- 知道确切文件 → 在提示中引用  
- 不知道 → 交给 Agent 去查  
- 包含不相关文件 → 会让 Agent 弄不清重点  

### 3.2 善用 @ 引用

- **`@Branch`**：让 Agent 了解当前工作，如  
  - "Review the changes on this branch"  
  - "What am I working on?"  

- **`@Past Chats`**：新对话时引用历史，Agent 可有选择地读取，只提取所需上下文，比复制整段对话更高效  

![通过引用以往的聊天记录，将之前对话的上下文带入当前会话](https://cursor.com/marketing-static/_next/image?url=https%3A%2F%2Fptht05hbb1ssoooe.public.blob.vercel-storage.com%2Fassets%2Fblog%2Fpast-chats.jpg&w=1920&q=70)

### 3.3 何时开新对话 vs 继续当前对话

| 开始新对话 | 继续当前对话 |
|------------|--------------|
| 切换到不同任务或功能 | 对同一功能迭代 |
| Agent 困惑、重复犯同样错误 | Agent 需要先前讨论的上下文 |
| 完成一个逻辑完整的工作单元 | 正在调试 Agent 刚构建的内容 |

**注意**：过长对话会积累噪音，Agent 易分散注意力。若效果下降，应及时开新对话。

---

## 四、扩展 Agent：Rules 与 Skills

### 4.1 Rules：静态上下文

在 `.cursor/rules/` 创建 Markdown 规则，**每次对话都会加载**。

**示例**（聚焦要点：命令、模式、示例引用）：

```markdown
# Commands

- `npm run build`: Build the project
- `npm run typecheck`: Run the typechecker
- `npm run test`: Run tests (prefer single test files for speed)

# 代码风格

- 使用 ES 模块(import/export)，而非 CommonJS(require)
- 尽可能使用解构导入：`import { foo } from 'bar'`
- 参考 `components/Button.tsx` 了解标准组件结构

# Workflow

- Always typecheck after making a series of code changes
- API routes go in `app/api/` following existing patterns
```

**应避免**：
- 复制整份风格指南（交给 linter）  
- 记录所有可能的命令（agent 已了解常用工具）  
- 为极少适用的边缘情况添加指令  

**提示**：从简单开始，只有 agent 反复犯同样错误时再新增规则。将规则提交到 Git，团队可共享。

### 4.2 Skills：动态能力

Skills 在 agent **认为相关时**才加载，可包含：

- **Custom commands**：用 `/` 触发的可复用工作流  
- **Hooks**：在 agent 动作前后运行的脚本  
- **Domain knowledge**：按需调用的任务指令  

### 4.3 示例：长时间运行 Agent 循环

用 Hooks 让 Agent 持续迭代直到达成目标（如测试全通过）。

**`.cursor/hooks.json`**：

```json
{
  "version": 1,
  "hooks": {
    "stop": [{ "command": "bun run .cursor/hooks/grind.ts" }]
  }
}
```

**`.cursor/hooks/grind.ts`**（从 stdin 接收上下文，返回 `followup_message` 驱动下一轮）：

```typescript
import { readFileSync, existsSync } from "fs";

interface StopHookInput {
  conversation_id: string;
  status: "completed" | "aborted" | "error";
  loop_count: number;
}

const input: StopHookInput = await Bun.stdin.json();
const MAX_ITERATIONS = 5;

if (input.status !== "completed" || input.loop_count >= MAX_ITERATIONS) {
  console.log(JSON.stringify({}));
  process.exit(0);
}

const scratchpad = existsSync(".cursor/scratchpad.md")
  ? readFileSync(".cursor/scratchpad.md", "utf-8")
  : "";

if (scratchpad.includes("DONE")) {
  console.log(JSON.stringify({}));
} else {
  console.log(JSON.stringify({
    followup_message: `[迭代 ${input.loop_count + 1}/${MAX_ITERATIONS}] 继续工作。完成后在 .cursor/scratchpad.md 中更新为 DONE。`
  }));
}
```

适用于：一直运行直到测试通过、反复迭代 UI 直到与设计稿一致、任何目标可验证的任务。

---

## 五、常见工作流

### 5.1 测试驱动开发（TDD）

1. 让 Agent 根据**预期输入/输出**编写测试（明确说明在做 TDD，避免 mock 实现）  
2. 运行测试，**确认失败**（此阶段不写实现）  
3. 提交测试  
4. 让 Agent 编写能通过测试的代码，**不修改测试**，持续迭代直到全部通过  
5. 提交实现  

Agent 有清晰迭代目标时表现最好，测试让它能不断评估结果并改进。

### 5.2 理解代码库

像向同事请教一样提问，Agent 会 grep + 语义搜索查找答案。例如：

- "这个项目里的日志是如何运作的？"  
- "我该如何添加一个新的 API endpoint？"  
- "`CustomerOnboardingFlow` 处理了哪些边界情况？"  
- "为什么第 1738 行用的是 `setUser()` 而不是 `createUser()`？"  

### 5.3 Git 工作流：自定义命令

将常用流程写成命令，放在 `.cursor/commands/`，用 `/` 触发。

**`/pr` 示例**：

```
为当前更改创建 Pull Request。

1. 使用 `git diff` 查看已暂存和未暂存的更改
2. 根据更改内容编写清晰的提交信息
3. 提交并推送到当前分支
4. 使用 `gh pr create` 创建 Pull Request，并提供标题和描述
5. 完成后返回 PR URL
```

**其他常用命令**：
- `/fix-issue [number]`：获取 issue 详情、查找相关代码、修复并开 PR  
- `/review`：运行 linter，检查常见问题，总结需关注内容  
- `/update-deps`：检查过时依赖，逐个更新，每次更新后跑测试  

---

## 六、其他能力

### 6.1 包含图片

- **从设计到代码**：粘贴设计稿，Agent 能识别布局、颜色、间距；可用 Figma MCP  
- **可视化调试**：对错误或异常 UI 截图，比文字描述更高效  
- **浏览器**：Agent 可控制浏览器截屏、测试应用、验证界面变化  

### 6.2 代码审查

- **生成中**：实时看 Diff，方向不对按 **Escape** 中断  
- **Agent Review**：完成后点击 Review → Find Issues，逐行分析、标记潜在问题  
- **Bugbot**：推送后为 PR 自动做代码审查  
- **架构图**：提示 "Create a Mermaid diagram showing the data flow for our authentication system, including OAuth providers, session management, and token refresh."  

### 6.3 并行运行 Agent

- Cursor 会为并行 agent 自动创建 **git worktrees**，各 agent 隔离运行  
- 同一提示可**同时跑多个模型**，对比结果，挑选最优  
- 适合：棘手问题、比较不同模型、发现边缘情况  

![多代理评估会显示 Cursor 推荐的解决方案](https://cursor.com/marketing-static/_next/image?url=https%3A%2F%2Fptht05hbb1ssoooe.public.blob.vercel-storage.com%2Fassets%2Fchangelog%2Fchangelog-2-2-judge.jpg&w=1920&q=70)

### 6.4 云端 Agent

适合「丢进待办」的任务：bug 修复、重构、补测试、文档更新。可从 cursor.com/agents、编辑器或手机启动，在远程沙箱运行，完成后开 PR、发通知。

**流程**：描述任务 → Agent 克隆仓库并建分支 → 自主工作 → 完成后开 PR → 审查并合并。

![Cursor Agents 执行编码和调研任务的看板视图](https://cursor.com/marketing-static/_next/image?url=https%3A%2F%2Fptht05hbb1ssoooe.public.blob.vercel-storage.com%2Fassets%2Fblog%2Fweb-and-mobile.jpg&w=1920&q=70)

### 6.5 Debug 模式

针对难以定位的 bug，Debug 模式会：

1. 生成多个可能出错原因的假设  
2. 用日志语句埋点  
3. 要求你在收集数据时复现 bug  
4. 分析实际行为定位根因  
5. 基于证据做针对性修复  

**适合**：能复现但搞不清原因、竞争条件、性能问题、回归 bug。你描述得越具体，Agent 添加的埋点越有价值。

![Agent 下拉菜单中的 Debug Mode](https://cursor.com/marketing-static/_next/image?url=https%3A%2F%2Fptht05hbb1ssoooe.public.blob.vercel-storage.com%2Fassets%2Fchangelog%2Fchangelog-2-2-debug-dropdown.jpg&w=1920&q=70)

---

## 七、打造你的工作流

### 7.1 写具体的提示

指令越具体，成功率越高。

**对比**：
- ❌ "add tests for auth.ts"  
- ✅ "Write a test case for auth.ts covering the logout edge case, using the patterns in `__tests__/` and avoiding mocks."  

### 7.2 推荐与避免

**✅ 推荐**
- 规划优先、写具体提示、认真 Review  
- 提供可验证目标：强类型、linter、测试  
- 把 Agent 当有能力的协作者：要计划、要解释、敢于质疑  

**❌ 避免**
- 一次性塞入过多复杂需求  
- 缺少项目背景和上下文  
- 期望 AI 理解所有业务细节  
- 在效果下降时仍坚持长对话修补  
- 在真正理解自己的模式之前过度优化规则  
