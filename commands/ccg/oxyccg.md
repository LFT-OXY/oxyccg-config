---
name: oxyccg
description: 完整开发工作流 - 整合 Superpowers + OpenSpec + Codex/Gemini 多模型协作。用于任何新功能开发、重构、或复杂任务。
---

# OxyCCG - 完整开发工作流

使用质量把关、MCP 服务和多模型协作执行结构化开发的完整工作流。

## 使用方法

```bash
/oxyccg <任务描述>
```

## 上下文

- 要开发的任务：$ARGUMENTS
- 带质量把关的结构化 8 阶段工作流
- 多模型协作：Codex（后端）+ Gemini（前端）+ Claude（编排）
- MCP 服务集成 (ace-tool)、(context7) 、(寸止)以增强功能

## 你的角色

你是**编排者**，协调多模型协作系统（研究 → 构思 → 计划 → 隔离 → 执行 → 审查 → 验收 → 交付），用中文协助用户，面向专业程序员，交互应简洁专业，避免不必要解释。

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

| 阶段 | Codex | Gemini |
|------|-------|--------|
| 分析 | `~/.claude/.ccg/prompts/codex/analyzer.md` | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| 规划 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/architect.md` |
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

## 执行工作流

**任务描述**：$ARGUMENTS

### 🔍 阶段 1：研究与分析

`[模式：研究]` - 理解需求并收集上下文：

1. **Prompt 增强**：调用 `mcp__ace-tool__enhance_prompt`，**用增强结果替代原始 $ARGUMENTS，后续调用 Codex/Gemini 时传入增强后的需求**
2. **上下文检索**：调用 `mcp__ace-tool__search_context`
3. **探索意图，需求澄清**：调用 `/superpowers:brainstorming` 和 `/interview`，通过 `寸止`澄清
4. **需求完整性评分**（0-10 分）：
   - 目标明确性（0-3）、预期结果（0-3）、边界范围（0-2）、约束条件（0-2）
   - ≥7 分：继续 | <7 分：⛔ 停止，提出补充问题
5. **不保存superpowers的独立设计文档，直接进入下一阶段**



### 💡 阶段 2：方案构思

`[模式：构思]` - 多模型并行分析：

**并行调用**（`run_in_background: true`）：
- Codex：使用分析提示词，输出技术可行性、方案、风险
- Gemini：使用分析提示词，输出 UI 可行性、方案、体验

用 `TaskOutput` 等待结果。**📌 保存 SESSION_ID**（`CODEX_SESSION` 和 `GEMINI_SESSION`）。

交叉验证两者输出：
- 整合各方思路，进行迭代优化
- 执行逻辑推演和优劣势互补
- 生成无逻辑漏洞的 Step-by-step 实施计划



### 📋 阶段 3：详细规划

`[模式：计划]` - 多模型协作规划：

**并行调用**（复用会话 `resume <SESSION_ID>`）：
- Codex：使用规划提示词 + `resume $CODEX_SESSION`，输出后端架构
- Gemini：使用规划提示词 + `resume $GEMINI_SESSION`，输出前端架构

用 `TaskOutput` 等待结果。

**Claude 综合规划**：采纳 Codex 后端规划 + Gemini 前端规划，用户批准后存入。

1. 调用 `/openspec:proposal` 创建变更提案
2. 文档结构（以下所有文档都必须创建）：
   - `proposal.md` - 包含 brainstorming 的设计结论
   - `design.md` - 技术决策
   - `tasks.md` - 实施步骤
   - `specs/` - 需求规格

3. **强制阻断**: 通过 `寸止` 输出方案摘要并询问 **"已经整合为统一方案，并完成提案。并询问还有要补充的吗？(Y/N)"**。
4. **如果用户说Y，必须执行这个操作**：调用 `/interview` 深度访谈，通过 `寸止` 完善方案细节（技术/UI/边缘/风险/架构）
5. 如果用户说N,就算是确认结束，执行下一个阶段。



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

`[模式：审查]` - 多模型并行审查：

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



### ✅ 阶段 7：功能验收

`[模式：验收]` - 验证功能完整性：

1. **完成前验证**：
    调用 `/superpowers:verification-before-completion`
  - **必须运行验证命令并查看输出**
  - **禁止无证据声称完成**

2. **计划对照检查**：
  - 逐项对照 OpenSpec tasks.md 检查完成情况

  - 标记：✅ 已完成 / ⚠️ 部分完成 / ❌ 未完成

3. **测试验证**：
  - 运行完整测试套件
  - 确保无回归问题

4. **验收报告**：
通过 `寸止` 呈现验收报告：
  - 已完成功能列表
  - 遗留问题（如有）
  - 后续优化建议（可选）
  - 请求用户最终确认，确认后进入交付阶段




### ✅ 阶段 8：交付

`[模式：交付]` - 最终交付：

1. 调用 `/superpowers:finishing-a-development-branch`
   - 验证测试通过
   - 提供选项: 合并/创建PR/保留/丢弃
2. 执行用户选择的交付方式
2. 调用 `/openspec:archive` 归档文档到 specs/
3. 通过 `寸止` 确认交付完成

---
结束条件

交付完成后，输出：

✅ 交付完成

📋 验收结果：
- 已完成：X 项
- 部分完成：X 项
- 未完成：X 项

🚀 交付方式：<合并/PR/保留/丢弃>
📁 归档位置：specs/<功能名>/

🎉 任务结束

---

## 关键规则

1. 阶段顺序不可跳过（除非用户明确指令）
2. 外部模型对文件系统**零写入权限**，所有修改由 Claude 执行
3. 评分 <7 或用户未批准时**强制停止**
4. 用户未确认前禁止结束
5. 严格按 tasks.md 实施
6. Critical 问题必须修复才能结束
4. 验收未通过禁止交付
5. 必须有测试证据
6. 用户确认后才能执行交付动作

---