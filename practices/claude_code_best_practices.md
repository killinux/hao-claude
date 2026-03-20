# Claude Code 最佳实践指南（2026）

---

## 一、CLAUDE.md — 最重要的基础设施

2026 年社区共识：**CLAUDE.md 的重要性不亚于 `.gitignore`**，是必要基础设施而非可选文档。

```bash
# 先用 /init 自动生成初始版本
/init
```

CLAUDE.md 里应该包含：
- 常用构建命令（如 `npm run test`、`npm run build`）
- 代码风格规范（如"使用 ES modules 而非 CommonJS"）
- 关键文件和架构说明
- Claude 容易犯错的地方（而不是它能自己推断出来的）

**黄金法则**：对每一行都问自己 _"删掉这行会让 Claude 犯错吗？"_ 如果不会，就删掉。  
保持根目录 CLAUDE.md 在 **50~100 行**以内，复杂部分用 `@imports` 引入子文件。

### CLAUDE.md 放置位置

| 位置 | 作用 |
|------|------|
| `~/.claude/CLAUDE.md` | 适用所有项目的个人偏好、commit 风格等 |
| `./CLAUDE.md` | 项目级，提交到 git 共享给团队 |
| 子目录 `CLAUDE.md` | 按需加载，适合 monorepo |

### 示例结构

```markdown
# 项目名称

## Tech Stack
- Frontend: React 19, TypeScript, Vite
- Backend: Node.js, Express
- Testing: Vitest

## Commands
- `npm run dev`   -- 启动开发服务器
- `npm test`      -- 运行全部测试
- `npm run lint`  -- Lint 检查

## Code Style
- 使用 ES modules（import/export），禁止 CommonJS（require）
- 仅使用函数组件，禁止 class 组件
- 优先使用具名导出

## Workflow
- 改动前先建 feature branch
- 每次 commit 前运行测试
- PR 只聚焦一个关注点

## 常见错误（Claude 容易犯的）
- 永远不要使用 --foo-bar 参数，用 --baz 代替
- 状态管理统一用 Zustand，见 src/stores/
```

---

## 二、Context 管理 — 最核心的约束

> Context window 是最重要的资源。它会被整个对话、Claude 读过的所有文件、所有命令输出填满。随着 context 填充，LLM 性能会下降，Claude 可能开始"遗忘"早期指令或犯更多错误。

### 核心原则

- **每个任务开启新 session**，不同任务之间用 `/clear`
- **在 70% context 使用量时主动 `/compact`**，不要等到满了再压缩
- **将研究任务委托给 subagent**，保持主 context 聚焦
- **如果同一个问题纠正超过两次，清空重来**

### 常用命令

```bash
/clear        # 清空当前 context，开始新对话
/compact      # 压缩 context，保留关键信息
/rewind       # 撤销最近一步操作（也可用 Esc Esc）
```

### Commit 节奏

频繁 commit，每个任务完成后立即 commit，**至少每小时一次**。  
这既是安全网，也是 context 管理的一部分。

---

## 三、Plan Mode — 先规划再执行

> "Vibe coding" 只适合一次性 MVP，生产代码需要结构化思考、验证和文档。

### 标准工作流

```
1. Plan   → 进入 Plan Mode，让 Claude 分析并制定方案
2. Review → 用第二个 Claude 实例作为 Staff Engineer 审查计划
3. Code   → 切回 Normal Mode 执行实现
4. Verify → 运行测试，git diff 审查
5. Commit → 提交，保持原子性
```

### 何时使用 Plan Mode

| 使用 Plan Mode | 直接执行 |
|----------------|----------|
| 方案不确定时 | 明显的单行修复 |
| 改动涉及多个文件 | typo 修正 |
| 在不熟悉的代码库工作 | 简单配置变更 |
| 架构级决策 | |

---

## 四、给 Claude 验证手段 — 杠杆最高的实践

> 给 Claude 提供测试、截图或预期输出，让它能自我检验。这是单一最高杠杆的操作。

如果 Claude 能运行 `npm test` 并看到绿色，它会**自我纠正**。  
给它验证标准，它就能自己处理反馈循环。

### 实践方式

```bash
# 告诉 Claude 如何验证
"实现完成后运行 npm test，确保所有测试通过再告诉我"
"截图对比修改前后的 UI"
"跑一下 npm run lint，修复所有报错"
```

---

## 五、Subagents — 保持主 context 干净

用 subagent 将子任务卸载，保持主 context 聚焦。  
用 `"use subagents"` 指令把更多算力投入到复杂问题上。

### Subagent 配置示例

```markdown
---
name: reviewer
description: Use for thorough code reviews
model: sonnet
color: orange
---
You are an expert code reviewer.
Focus on security, performance, and maintainability.
```

### 常见 Subagent 角色

| 角色 | 用途 |
|------|------|
| reviewer | 代码审查，安全/性能/可维护性 |
| debugger | 专注 bug 分析，不动其他代码 |
| docs-writer | 文档生成，不涉及实现 |
| test-generator | 专门写测试 |

---

## 六、Hooks — 自动化质量门控

使用 `/permissions` 配合通配符语法，减少打断的同时保持控制：

```
Bash(npm run *)        # 允许所有 npm 脚本
Edit(/docs/**)         # 只允许编辑 docs 目录
```

### 常见 Hook 场景

| Hook 事件 | 用途 |
|-----------|------|
| `PreToolUse` | 拦截危险命令（如删除操作） |
| `PostToolUse` | 每次编辑后自动运行 lint/test |
| `Stop` | 任务完成后发桌面通知 |
| `UserPromptSubmit` | 注入额外上下文 |

使用 `/sandbox` 可以通过文件和网络隔离大幅减少权限提示（内部测试减少 84%）。

---

## 七、提示词技巧

### 基础技巧

- **引用具体文件**：用 Tab 补全加入文件路径，`@./src/user-service.js`
- **提供 URL**：粘贴 GitHub issue、文档链接，Claude 会自己去读
- **拖入截图**：做 UI 工作时直接提供设计稿或错误截图
- **给 URL 加权限**：`/permissions` allowlist 常用域名，避免每次询问

### 要具体，不要模糊

```bash
# ❌ 模糊
"加测试"

# ✅ 具体
"为 foo.py 写一个覆盖用户未登录边界情况的单元测试"
```

### 高级提示词技巧

```bash
# 挑战 Claude，提高代码质量
"帮我审问这些改动，在我通过你的测试之前不要提 PR"
"向我证明这个方案是正确的"

# 推倒重来
"结合你现在知道的一切，推倒重来，实现一个优雅的方案"

# 并行计算
"use subagents 来处理这个问题"

# 计划驱动
"先列出完整计划，等我确认后再开始写代码"
```

---

## 八、安全注意事项

研究显示 AI 生成的代码包含更多安全漏洞，常见问题包括：
- 不当的密码处理
- 不安全的直接对象引用
- 缺失的输入验证

**最佳实践**：
- 始终**手动审查** auth、支付和数据修改相关代码
- 使用 hooks 对每次编辑进行自动安全检查
- 不要使用 `dangerously-skip-permissions`，用通配符 permissions 代替

---

## 九、总结：工作流全景

```
┌─────────────────────────────────────────────┐
│              开始新任务                       │
│         (新 session / /clear)                │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│           CLAUDE.md 提供持久上下文            │
│    构建命令 / 代码风格 / 架构 / 常见错误       │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│              Plan Mode                       │
│      分析 → 制定方案 → Subagent 审查          │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│           Normal Mode 执行                   │
│         Hooks 自动质量门控                    │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│              验证                            │
│     测试 / 截图 / 预期输出 → Claude 自我纠正  │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│           Commit（频繁提交）                  │
│         /compact 保持 context 健康            │
└─────────────────────────────────────────────┘
```

**三个核心原则**：
1. **痴迷于 context 管理** — 这是主要失败模式
2. **编码前严格规划** — 技术债来自跳过这一步
3. **保持系统简单** — 复杂性让 LLM 调试难度指数级增加
