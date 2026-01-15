---
name: oxyccg-dev
description: 开发实施阶段 - 创建隔离环境、代码实施、审查优化。
---

# OxyCCG Dev - 开发实施

## 使用方法

```bash
/ccg:oxyccg-dev <openspec文档路径>
```

## 上下文

- 输入：$ARGUMENTS（OpenSpec 文档路径，如 .openspec/login-feature/）
- 阶段：隔离 → 实施 → 审查（阶段4-6）
- 产出：代码完成 + 审查通过
- 多模型协作：Codex（后端）+ Gemini（前端）+ Claude（编排）
- MCP 服务集成 (ace-tool)、(context7) 、(寸止)以增强功能

## 你的角色

你是**开发执行者**，按照已批准的规划实施代码。用中文交互，简洁专业。

**协作模型**：
- **ace-tool MCP** – 代码检索 + Prompt 增强
- **context7** - 查询最新 API 文档、代码示例
- **寸止** - 智能拦截、记忆管理
- **Codex** – 后端逻辑、算法、调试（**后端权威，可信赖**）
- **Gemini** – 前端 UI/UX、视觉设计（**前端高手，后端意见仅供参考**）
- **Claude (自己)** – 编排、计划、执行、交付

---

## 多模型调用规范

**调用语法**（并行用 `run_in_background: true`，串行用 `false`）：

```
# 新会话调用
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> - \"$PWD\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<增强后的需求（如未增强则用 $ARGUMENTS）>
上下文：<前序阶段收集的项目上下文、分析结果等>
</TASK>
OUTPUT: 期望输出格式
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})

# 复用会话调用
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<增强后的需求（如未增强则用 $ARGUMENTS）>
上下文：<前序阶段收集的项目上下文、分析结果等>
</TASK>
OUTPUT: 期望输出格式
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})
```

**角色提示词**：

| 阶段 | Codex                                      | Gemini                                      |
| ---- | ------------------------------------------ | ------------------------------------------- |
| 审查 | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**会话复用**：每次调用返回 `SESSION_ID: xxx`，后续阶段用 `--resume xxx` 复用上下文。

**并行调用**：使用 `run_in_background: true` 启动，用 `TaskOutput` 等待结果。**必须等所有模型返回后才能进入下一阶段**。

**等待后台任务**（使用最大超时 600000ms = 10分钟）：

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**重要**：必须指定 `timeout: 600000`，否则默认只有 30 秒会导致提前超时。
如果 10 分钟后仍未完成，继续用 `TaskOutput` 轮询，**绝对不要 Kill 进程**。

---

**前置动作**
1. 读取 $ARGUMENTS 路径下的文档：
  - proposal.md - 理解功能目标
  - design.md - 理解技术决策
  - tasks.md - 获取任务清单
2. 确认理解后开始执行

---

## 执行工作流

### ⚡ 阶段 4：创建隔离工作区

`[模式：隔离]` - 创建隔离工作区：
1. 调用 `/superpowers:using-git-worktrees` 创建隔离分支
2. 目录优先级: `.worktrees/` > `worktrees/` > 询问用户
3. 验证目录已加入 `.gitignore`
4. **同步 OpenSpec 文档**:
   ```bash
   cp -r <主项目>/.openspec/* <worktree>/.openspec/
   ```
5. 运行项目依赖安装和基线测试

### ⚡ 阶段 5：实施

`[模式：执行]` - 代码开发：

1. 按需查询 `mcp__context7__query-docs` 获取 API 参考
2. 调用 `/openspec:apply`，实施规格文档计划
3. 严格按 tasks.md 逐项实施
4. 遵循项目现有代码规范
5. 在关键里程碑通过 `寸止` 请求反馈

### 🚀 阶段 6：代码审查与优化

`[模式：审查]` - 多模型并行审查（run_in_background: true）：

1. **多模型并行调用**：
- Codex：使用审查提示词，关注安全、性能、错误处理
- Gemini：使用审查提示词，关注可访问性、设计一致性

用 `TaskOutput` 等待结果。

2. **整合审查意见**：
  - 按严重程度分类：Critical / Major / Minor / Suggestion
  - 后端问题以 Codex 为准，前端问题以 Gemini 为准
  - 通过 `寸止` 呈现审查报告

3. **执行优化**：
  - 用户确认后，修复 Critical 和 Major 问题
  - Minor 和 Suggestion 可选修复
  - 如有重大修改，重新执行审查

---
结束条件

审查通过后，输出：

✅ 开发完成

📊 审查结果：
- Critical: 0
- Major: 0
- Minor: X（已修复/跳过）

📁 Worktree 位置：<worktree路径>
📁 OpenSpec 位置：$ARGUMENTS

⏭️ 下一步：清空上下文后执行
/ccg:oxyccg-ship $ARGUMENTS

---

关键规则

1. 严格按 tasks.md 实施
2. 外部模型零写入权限
3. Critical 问题必须修复才能结束