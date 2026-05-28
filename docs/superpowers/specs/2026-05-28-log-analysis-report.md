# Superpowers 日志分析报告

> **日期：** 2026-05-28
> **分析范围：** `~/.claude/superpowers-logs/` 下 2026-05-27 和 2026-05-28 两天日志
> **目的：** 验证 skill 组合、注入、到 workflow 中实际调用是否符合设计预期

## 1. 数据概览

| 指标 | 05-27 | 05-28 | 合计 |
|------|-------|-------|------|
| 工具调用总数 | 681 | 337 | 1,018 |
| 会话数 | 10 | 4 | 14 |
| Skill 调用 | 4 | 0 | **4** |
| Agent 调用 | 2 | 1 | 3 |
| Workflow 调用 | 0 | 0 | **0** |
| result_len=0 的记录 | ~200 | ~88 | **~288** |

### 唯一被调用的 Skills（全部 4 次）

| 时间 | Skill | 会话 |
|------|-------|------|
| 05-27 11:55 | `brainstorming` | test-session-123（测试数据） |
| 05-27 14:33 | `superpower-claude-plus:skill-router` | 425f6f8c |
| 05-27 14:42 | `ui-ux-pro-max:ui-ux-pro-max` | 425f6f8c |
| 05-27 17:40 | `ui-ux-pro-max:ui-ux-pro-max` | 4b04f70a |

### 从未被调用的 Skills

`orchestrator`, `writing-plans`, `executing-plans`, `subagent-driven-development`, `dispatching-parallel-agents`, `systematic-debugging`, `verification-before-completion`, `test-driven-development`, `finishing-a-development-branch`, `brainstorming`（正式会话中）

---

## 2. Skill 组合链路分析

### 设计预期的完整链路

```
brainstorming → orchestrator → skill-router → domain skills
                                    ↓
                    Workflow/Team/Agent (带 SKILL_INJECTIONS)
```

### 实际观察到的链路

#### 会话 425f6f8c（最佳案例，部分符合）

```
TaskUpdate 推进 brainstorming checklist
  → Skill: skill-router（传入 7 个子任务的详细描述）
    → 自动匹配到 ui-ux-pro-max 和 verification-before-completion
      → Skill: ui-ux-pro-max（手动调用，非自动注入）
      → 直接编码实现，无 Workflow/Team/Agent 模式
```

**评估：** skill-router 被正确调用了，而且传入了完整的子任务描述。但：
1. orchestrator 从未作为 Skill 被调用 —— 它是通过 TaskUpdate 检查清单模拟的
2. skill-router 的输出（SKILL_INJECTIONS）没有被注入到任何 Agent/Workflow 中
3. ui-ux-pro-max 是主会话中直接调用的，不是通过 agent prompt 注入触发的

#### 会话 4b04f70a（338 次工具调用）

```
直接进入编码（无 brainstorming）
  → 中途调用 Skill: ui-ux-pro-max（仅此一个 Skill）
  → 大量 Read/Edit/Bash 直接实现功能
  → 无 Workflow、无 Agent、无 skill-router
```

#### 会话 50ef14f4（294 次工具调用，0 Skill）

```
完整的 brainstorming checklist（通过 TaskUpdate 跟踪）
  → 直接编码（hooks、组件、样式）
  → 浏览器调试（chrome-devtools MCP）
  → 完全没有调用任何 Skill
```

**这是一个关键发现：** 该会话执行了完整的 brainstorming 流程（从 task 列表可以推断），但在转入"实现"阶段后没有调用 orchestrator、skill-router 或任何 domain skill。

---

## 3. Workflow 模式与 SKILL_INJECTIONS 分析

### 关键结论：Workflow 模式从未被使用

两天的 1,018 次工具调用中，**Workflow 工具调用次数为 0**。这意味着：

1. `SKILL_INJECTIONS` map 从未被构建
2. `agent()` 调用中的 `"IMPORTANT: Before starting, invoke the Skill tool..."` 注入从未通过 workflow 触发
3. `pipeline()` 和 `parallel()` 模式从未被使用
4. orchestrator 的 Phase 4（Execute）中的 Workflow/Team 模式从未执行

### 唯一含有 skill injection 文本的 Agent 调用

```
Agent(desc="Explore skill injection in workflow")
  prompt 中包含 "invoke Skill tool" 字样
  → 但这是一个研究/探索 agent，不是实现任务的 agent
```

这个 Agent 是在会话 2fbf62e8 中用于**探索 skill 系统结构**的，prompt 内容是让 agent 去读和理解 skill 文件，不是让 agent 去执行任务时加载 skill。这不是设计预期的 skill injection 使用方式。

---

## 4. 日志系统质量评估

### 4.1 tool-call-logger hook 整体工作状态

hook 已经正常工作并记录了 1,018 条日志，覆盖 14 个会话。基本的事件捕获是成功的。

### 4.2 字段覆盖问题

hook 实际通过 stdin 接收的完整字段（从 `_keys` 和 `.debug-raw` 确认）：

```
session_id, tool_name, tool_input, cwd, tool_response, tool_use_id,
hook_event_name, duration_ms, effort, permission_mode, transcript_path,
agent_id (部分), agent_type (部分)
```

但 JSONL 日志中只保留了：

```
ts, session_id, tool_name, tool_input, cwd, skill_name (仅Skill), result_len, result_preview, _keys
```

**丢失的有价值字段：**
- `duration_ms` — 无法分析工具调用耗时
- `tool_use_id` — 无法关联同一调用的请求和响应
- `hook_event_name` — 无法区分 PostToolUse 与其他事件（虽然目前只有 PostToolUse）
- `effort` / `permission_mode` — 无法分析配置对行为的影响
- `transcript_path` — 无法关联到完整会话记录

### 4.3 result_len=0 问题

**严重问题：约 28% 的日志条目 `result_len=0`。**

分布：
- Read: 148/249 条 result_len=0（59% 缺失）
- Bash: 77/163 条 result_len=0（47% 缺失）
- Edit: 14/47 条 result_len=0（30% 缺失）
- TaskUpdate: 35/74 条 result_len=0（47% 缺失）
- Skill: 2/4 条 result_len=0（50% 缺失）

**根因分析：**

hook 代码中 tool_result 提取逻辑：
```python
tool_result = (
    data.get('tool_result') or
    data.get('tool_response') or
    data.get('result') or
    data.get('output') or
    ''
)
```

从 `.debug-raw` 可以看到，stdin 中字段名是 `tool_response`（不是 `tool_result`）。而 `tool_response` 的值是一个复杂结构，例如 Read 工具的响应是：
```json
{"type": "text", "file": {"filePath": "...", "content": "...", ...}}
```

Python 中非空 dict 是 truthy 的，所以 `data.get('tool_response')` 应该能正确获取到值。**但** 如果 hook 在某些情况下接收到的 stdin 中确实没有 `tool_response`（例如工具调用被中断或取消），或者 `tool_response` 值为空 dict `{}`，则会 fallback 到 `''`。

另一种可能：异步 hook（`async: true`）在某些情况下 stdin 数据不完整。

### 4.4 _keys 字段不一致

736 条记录有 `_keys` 字段（非空数组），318 条记录 `_keys` 为空数组。有 `_keys` 的记录来自正式 hook 运行，无 `_keys` 的可能来自：
- 早期版本的 hook（v5.2.2 之前的测试数据）
- 人工注入的测试条目（如 test-session-123、test-456）

---

## 5. 设计预期 vs 实际行为差异总结

### 5.1 Skill 调用链路

| 设计预期 | 实际观察 | 符合度 |
|----------|----------|--------|
| brainstorming 完成后调用 orchestrator | orchestrator 从未被 Skill 调用 | **不符合** |
| orchestrator Phase 2 调用 skill-router | skill-router 被直接调用（绕过 orchestrator） | **部分符合** |
| skill-router 输出 SKILL_INJECTIONS map | 从未生成 SKILL_INJECTIONS map | **不符合** |
| agent() prompt 包含 skill injection 文本 | 从未通过 agent prompt 注入 | **不符合** |
| Workflow/Team 模式执行任务 | 从未使用 Workflow 或 Team 工具 | **不符合** |
| domain skill 在 agent 内部被调用 | domain skill 在主会话中直接调用 | **部分符合** |

### 5.2 最可能的偏差原因

1. **Claude 模型倾向于内联执行：** 面对多步骤任务，模型更倾向于在主会话中直接用 Read/Edit/Bash 完成，而不是通过 orchestrator 拆分并分发给 Workflow/Agent
2. **brainstorming 的"transition to orchestration"步骤未被执行：** brainstorming 的最后一步是调用 orchestrator skill，但实际中这个步骤经常被跳过——模型直接从设计文档跳到编码
3. **skill-router 的路由结果被"读取但未应用"：** skill-router 确实被调用了并且返回了匹配结果，但 SKILL_INJECTIONS 没有被应用到任何 agent prompt 中
4. **Workflow 工具的认知门槛：** 模型似乎不熟悉 Workflow 脚本的构建方式，倾向于用更基础的 TaskUpdate + Edit 循环代替

### 5.3 积极发现

1. **hook 基础设施工作正常：** PostToolUse hook 成功捕获了所有工具调用，JSONL 格式正确可解析
2. **skill-router 的语义匹配是有效的：** 传入 7 个子任务后正确匹配到 ui-ux-pro-max 和 verification-before-completion
3. **brainstorming checklist 机制在工作：** 从 TaskUpdate 序列可以看出，brainstorming 的 9 步 checklist 被严格遵循（Task 1-9 按序完成）
4. **domain skill 确实被使用了：** ui-ux-pro-max 在两个独立会话中被调用，说明 skill 发现和加载机制本身是工作的

---

## 6. 建议改进方向

### 6.1 日志系统改进（高优先级）

1. **补充丢失字段：** 在 JSONL 中记录 `duration_ms`、`tool_use_id`、`effort`
2. **修复 result_len=0：** 调查异步 hook 的 stdin 传递机制，确认 `tool_response` 何时缺失
3. **移除 `_keys` 调试字段：** 这只是开发期的诊断信息，不应进入生产日志

### 6.2 Skill 流程强制执行（中优先级）

1. **brainstorming → orchestrator 过渡：** 在 brainstorming SKILL.md 中加强对 orchestrator 调用的硬约束（已有 HARD-GATE 但显然被绕过）
2. **orchestrator 自动触发：** 考虑在 session-start hook 中注入更明确的 orchestrator 触发指令
3. **SKILL_INJECTIONS 验证：** 在 orchestrator Phase 4 的 Pre-flight Checklist 中增加自检机制

### 6.3 可观测性增强（低优先级）

1. **记录 Workflow/Team 生命周期：** 目前无法观察到 Workflow 脚本的执行过程
2. **Agent 子会话日志：** Agent 调用的子会话中的工具调用未被当前 hook 捕获（或者被捕获但无法关联到父会话）
3. **Skill 调用效果追踪：** 记录 skill 加载后模型的后续行为变化

---

## 附录：会话行为画像

### 会话 425f6f8c — "理想但残缺"的 skill 流程
- 唯一一个调用了 skill-router 的会话
- skill-router 正确匹配了 domain skills
- 但 orchestrator 未被调用，Workflow 未被执行
- 最终以主会话直接编码完成

### 会话 4b04f70a — "高效但无 skill"的大规模开发
- 338 次工具调用，跨足多个子系统
- 仅在中途调用了 ui-ux-pro-max 一个 skill
- 直接在主会话中完成了所有架构设计和实现

### 会话 50ef14f4 — "无 skill"的完整功能开发
- 294 次工具调用，从 brainstorming 到编码到浏览器调试
- 完全没有调用任何 Skill
- 这是一个"skill 系统——不生效"的典型案例

### 会话 2fbf62e8 — 纯研究/探索会话
- 62 次调用，以 Read 和 Agent（探索型）为主
- 在研究 skill 系统本身的结构和注入机制
