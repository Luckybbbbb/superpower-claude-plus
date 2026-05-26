# Orchestrator + Skill Router 设计文档

> **日期：** 2026-05-26
> **状态：** 设计完成，待实现

## 目标

为 superpowers 插件增加智能编排能力，使其能够：
1. 根据任务复杂度自动选择 Agent / Workflow / Team 执行模式
2. 通过语义扫描自动匹配并注入本地 skills（如 UI 任务自动调用 ui-ux-pro-max）
3. 处理任务间的依赖关系和上下文传递，避免纯串行 subagent 的局限

## 背景与动机

当前 superpowers 的任务执行模式：
- `subagent-driven-development`：串行 dispatch Agent，每个任务独立执行
- `dispatching-parallel-agents`：并行但无协调机制

痛点：
- 任务间无法传递上下文，下游 agent 缺乏上游的信息
- 无法根据任务语义自动组装本地 skills
- 简单任务和复杂任务用同一套执行模式，效率不高

## 架构概述

### 新增 Skills

```
skills/
  orchestrator/
    SKILL.md              # 主编排 skill（四阶段流程）
    routing-logic.md      # 智能路由决策逻辑参考
    templates/            # Workflow 脚本模板
      pipeline-with-deps.js
      team-collaborative.js
  skill-router/
    SKILL.md              # 语义扫描 skill
```

### 四阶段流程

```
阶段 1：需求梳理（对话式，参考 brainstorming 模式）
   │  通过多轮对话理解：
   │  - 任务目标、约束、成功标准
   │  - 子任务之间的依赖关系
   │  - 涉及的领域（UI、后端、测试...）
   │
   ▼
阶段 2：Skill 路由（自动）
   │  skill-router 根据梳理结果扫描匹配：
   │  - 分析每个子任务的关键词和语义
   │  - 与所有已注册 skill 的 description 比对
   │  - 返回每个子任务的 skill 注入列表
   │
   ▼
阶段 3：编排决策（自动）
   │  根据任务特征选择执行模式：
   │  - 1-2 个简单独立任务 → Agent
   │  - 有流水线阶段 → Workflow pipeline
   │  - 多角色 + 需要实时协作 → Team
   │
   ▼
阶段 4：执行（自动）
   │  按选定模式执行
   │  - 依赖图（TaskCreate blockedBy）控制顺序
   │  - 匹配的 skill 作为前置指令注入每个 agent
   │  - 上下文通过 pipeline prevResult / SendMessage 传递
```

## Skill 1：orchestrator

### 触发条件

```yaml
name: orchestrator
description: Use when facing a complex task that requires multi-step coordination, task dependency management, or automatic skill composition. Triggers when task involves multiple domains (UI + backend, design + implementation), has clear dependency chains, or needs more than simple sequential agent dispatch.
```

### 阶段 1：需求梳理

内化 brainstorming skill 的核心对话模式（不直接调用，太重）：

1. **探索上下文** — 检查当前项目状态、文件、相关代码
2. **逐一提问** — 一次一个问题，理解：
   - 任务的整体目标是什么？
   - 有哪些子任务？它们之间的关系？
   - 涉及哪些技术领域？
   - 有什么约束或非目标？
3. **确认理解** — 用自己的话复述需求，获取用户认可

**例外**：如果用户的需求涉及"从零设计一个新功能"，建议先走完整的 `superpowers:brainstorming` → `superpowers:writing-plans` 流程。

### 阶段 2：Skill 路由

调用 `superpowers:skill-router`（通过 Skill 工具），传入阶段 1 梳理出的子任务列表。

### 阶段 3：编排决策

根据 `routing-logic.md` 中的决策逻辑选择执行模式：

**决策树：**

```
任务数量 ≤ 2 且完全独立？
  → Agent 模式
    直接 dispatch Agent，注入匹配的 skill

有明确的阶段划分，阶段间有数据流？
  → Workflow 模式
    使用 pipeline() 串行阶段 + parallel() 并行子任务

需要 3+ 个专业角色，且需要实时通信？
  → Team 模式
    TeamCreate + TaskCreate(blockedBy) + SendMessage

混合场景？
  → Workflow 编排外层，内部节点可 dispatch Team agent
```

### 阶段 4：执行

根据选定模式执行，关键机制：

**Agent 模式：**
- dispatch 单个 Agent
- prompt 中注入 skill-router 匹配的 skill 调用指令

**Workflow 模式：**
- 使用预定义模板（`templates/pipeline-with-deps.js` 等）
- pipeline 的每个 stage callback 接收 `(prevResult, originalItem, index)`
- 上游 agent 的输出自然流入下游 agent 的 prompt
- 用 `phase()` 标记阶段进度

**Team 模式：**
- TeamCreate 创建团队
- TaskCreate 创建任务，用 `addBlockedBy` 声明依赖
- 每个任务通过 TaskUpdate 设置 owner
- 完成后通过 SendMessage 传递上下文给下游
- team-lead 监控全局进度

## Skill 2：skill-router

### 触发条件

```yaml
name: skill-router
description: Use when orchestrator needs to identify which local skills match a given task. Scans all registered skill frontmatter and returns a prioritized list of matching skills with injection instructions.
```

### 扫描流程

1. **收集 skill 列表** — 读取所有已注册 skill 的 frontmatter：
   - `~/.claude/skills/*/SKILL.md`
   - 已安装插件的 `skills/*/SKILL.md`
   - 重点关注 `name` 和 `description` 字段

2. **分析任务关键词** — 从任务描述中提取：
   - 领域词（UI、调试、测试、游戏引擎...）
   - 操作词（设计、修复、重构、编译...）
   - 技术词（React、Lua、Godot、Cocos...）

3. **语义匹配** — 将任务关键词与每个 skill 的 description 比对
   - 返回匹配度排序的 skill 列表
   - 每个匹配项包含：skill 名称、匹配原因、建议的调用方式

4. **生成注入指令** — 对每个匹配的 skill 生成：
   ```
   "执行此任务前，必须先通过 Skill 工具加载 {skill-name}"
   ```
   编排器将此指令追加到 agent 的 prompt 中

### 匹配规则示例

| 任务描述关键词 | 匹配的 Skill | 原因 |
|---|---|---|
| UI、界面、交互、布局、样式 | ui-ux-pro-max:ui-ux-pro-max | description 匹配 UI/UX 设计 |
| 设计系统、组件库、Design Token | ui-ux-pro-max:design-system | description 匹配设计系统 |
| 调试、bug、报错、崩溃 | superpowers:systematic-debugging | description 匹配调试场景 |
| Prefab、Lua 绑定、Cocos | jx3-prefab-mcp-luaui | description 匹配 Cocos/Lua |
| 相机、镜头、跟随 | closeup-camera-system | description 匹配相机系统 |

## Workflow 模板

### pipeline-with-deps.js

用于有依赖关系的流水线任务：

```javascript
export const meta = {
  name: 'pipeline-with-deps',
  description: 'Execute tasks in pipeline with dependency-aware ordering and context flow',
  phases: [] // 动态生成
}

// 由 orchestrator 动态构建：
// 1. 解析依赖图，确定执行顺序
// 2. 为每个阶段生成 phase() 和 agent() 调用
// 3. 用 parallel() 处理同一层级的并行任务
// 4. pipeline 的 prevResult 传递上下文
```

### team-collaborative.js

用于多角色协作：

```javascript
export const meta = {
  name: 'team-collaborative',
  description: 'Multi-role team collaboration with task dependencies',
  phases: [
    { title: 'Setup' },
    { title: 'Execute' },
    { title: 'Review' },
    { title: 'Integrate' }
  ]
}

// Setup: TeamCreate + TaskCreate with blockedBy
// Execute: Agent dispatches with skill injection
// Review: Cross-review between roles
// Integrate: Merge results, verify
```

## 与现有 Skills 的集成

### using-superpowers（更新）

在 skill 列表中增加：
- `orchestrator`：复杂任务的智能编排入口
- `skill-router`：语义扫描匹配本地 skills

在 Red Flags 表中增加：
```
| "这个任务太复杂不适合直接做" | 复杂任务应该用 orchestrator 编排 |
| "我需要同时处理多个子系统" | 多子系统任务应该用 orchestrator 路由 |
```

### brainstorming（不变）

- orchestrator 的阶段 1 内化其对话模式，不直接调用
- 但"从零设计新功能"仍建议走完整 brainstorming → writing-plans

### subagent-driven-development（共存）

- 保留给已写好 plan 的串行执行场景
- orchestrator 的 Agent 模式是其增强版（加了 skill 路由）

### dispatching-parallel-agents（共存）

- 保留给已知需要并行的简单场景
- orchestrator 的 Workflow/Team 模式是其增强版

## 设计决策记录

1. **为什么是 skill 而不是 agent 定义？**
   - 与 superpowers 现有架构一致
   - Skill 可以被 orchestrator、brainstorming 等多个 skill 引用
   - 不需要额外的 agent 定义文件

2. **为什么用依赖图 + 消息传递混合？**
   - 依赖图（TaskCreate blockedBy）解决执行顺序问题
   - 消息传递（SendMessage / pipeline prevResult）解决上下文流转问题
   - 两者互补，单独使用都不够

3. **为什么不直接调用 brainstorming？**
   - brainstorming 有完整的 checklist 和文档流程，对大多数任务太重
   - orchestrator 只需要其对话模式的精华（逐一提问、确认理解）
   - 但"从零设计"类需求仍然走完整 brainstorming 流程

4. **skill-router 为什么不维护映射表？**
   - 映射表需要手动维护，新 skill 需要手动注册
   - 语义扫描自动适配新安装的 skill，零配置
   - 代价是匹配精度略低，但可通过 description 质量弥补
