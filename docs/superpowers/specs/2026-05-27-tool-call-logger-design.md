# PostToolUse 全工具调用日志设计

> **日期：** 2026-05-27
> **状态：** 设计完成，待实现

## 目标

为 superpowers 插件增加可观测性，记录 Claude Code 会话中每一次工具调用（包括 Skill 调用），以 JSONL 格式持久化到本地磁盘。解决当前"skill 注入后无法验证是否被正确使用"的黑盒问题。

## 背景与动机

当前 superpowers 插件通过 hooks 注入 skill 内容到会话上下文，但：
- 没有任何机制记录哪些 skill 被调用、何时调用、调用顺序如何
- 无法验证 orchestrator 的 skill-router 是否正确匹配和注入了子 agent 所需的 skills
- 遇到"明明注入了 ui-ux-pro skill 但产出界面仍然很弱"时，无法判断是 skill 没被调用还是被调用但没被遵循
- 项目 hook 系统已有 SessionStart、PostToolUse(Skill)、PreToolUse(Skill) 三个 hook，但没有任何日志产出

## 方案概述

利用 Claude Code 的 `PostToolUse` hook 事件，新增一个 matcher 为空（匹配所有工具）的 hook，在每次工具调用完成后将关键信息追加写入按天分割的 JSONL 日志文件。

## 架构

### 新增文件

```
hooks/
  post-tool-logger        # 新的 bash hook 脚本
  hooks.json              # 添加 PostToolUse[*] hook 注册
  hooks-cursor.json       # 同步更新
```

### 日志文件位置

```
~/.claude/superpowers-logs/
  .enabled                    # 标记文件：存在即启用日志
  2026-05-27.jsonl            # 按天分割的 JSONL 日志
  2026-05-28.jsonl
  ...
```

### Hook 注册（hooks.json 新增条目）

在 `hooks.json` 的 `PostToolUse` 数组中新增一个 matcher 为空（匹配所有工具）的条目：

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "command",
      "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" post-tool-logger",
      "async": true
    }
  ]
}
```

使用 `async: true` 确保日志记录不阻塞主流程。

### Hook 脚本设计

`post-tool-logger` 接收 Claude Code 通过 stdin 传入的 PostToolUse JSON 输入。

**stdin 输入格式**（Claude Code 提供）：
```json
{
  "tool_name": "Skill",
  "tool_input": { "skill": "brainstorming" },
  "tool_result": "...",
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "hook_event_name": "PostToolUse"
}
```

**日志输出格式**（JSONL，每行一条）：
```json
{
  "ts": "2026-05-28T14:30:00+08:00",
  "session_id": "abc123",
  "tool_name": "Skill",
  "skill_name": "brainstorming",
  "tool_input": { "skill": "brainstorming" },
  "result_len": 42,
  "result_preview": "...",
  "cwd": "E:\\AIDemos\\my-project",
  "plugin_version": "5.2.3",
  "duration_ms": 85
}
```

### 字段说明

| 字段 | 来源 | 说明 |
|------|------|------|
| `ts` | 脚本生成 | ISO 8601 时间戳 |
| `session_id` | stdin | 会话 ID |
| `tool_name` | stdin | 工具名称（Skill、Read、Edit、Bash 等） |
| `skill_name` | 从 tool_input 提取 | 仅 tool_name 为 Skill 时存在，顶层冗余字段方便 grep |
| `tool_input` | stdin | 工具输入参数（完整保留） |
| `result_len` | 从 tool_response 计算 | tool_response 的字符长度 |
| `result_preview` | 从 tool_response 截取 | tool_response 的前 500 字符（截断处理） |
| `cwd` | stdin | 当前工作目录 |
| `plugin_version` | plugin.json | 插件版本号（如 "5.2.3"），用于区分版本变更导致的行为差异 |
| `duration_ms` | stdin | 工具调用耗时（毫秒），仅当 hook stdin 提供时存在 |

### tool_result 截断策略

`tool_result` 可能包含完整文件内容（如 Read 工具）或大量数据，直接写入会导致日志文件膨胀。处理方式：

- 不保留完整 `tool_result`
- 记录 `result_len`（原始长度）和 `result_preview`（截断预览）
- `result_preview` 策略：总长度 <= 500 则完整保留；否则取前 400 字符 + `...[truncated, total=N]`
- 截断在字符级别操作（python3 `str[:n]`），不会破坏多字节 UTF-8 字符

### 开关控制

通过标记文件 `~/.claude/superpowers-logs/.enabled` 控制：
- 文件存在 → 记录日志
- 文件不存在 → 脚本立即 exit 0，不做任何 I/O
- 首次安装时由 hook 脚本自动创建该标记文件和目录

### 跨平台兼容

复用现有的 `run-hook.cmd` polyglot wrapper，与 `session-start`、`post-skill-reminder` 等脚本使用相同的调用模式。

### 错误容忍

脚本内部所有操作包裹在错误处理中：
- JSON 解析失败 → exit 0（静默跳过）
- 目录/文件写入失败 → exit 0（静默跳过）
- 不向 stdout 输出任何内容（避免干扰 hook 系统）
- 绝不影响正常的工具调用流程

### 并发安全

- 使用追加模式（`>>`）写入文件，单次 write 在大多数 OS 上是原子的
- JSONL 格式天然支持并发追加（每行独立完整 JSON）
- 不使用文件锁，避免引入额外依赖和性能开销

## 使用场景

### 1. 验证 skill 调用链

```bash
# 查看今天所有 Skill 调用
cat ~/.claude/superpowers-logs/2026-05-27.jsonl | grep '"tool_name":"Skill"'
```

### 2. 检查 skill-router 注入是否生效

```bash
# 查看 skill 调用中是否包含预期的 skill
cat ~/.claude/superpowers-logs/2026-05-27.jsonl | grep '"skill_name":"ui-ux-pro-max"'
```

### 3. 分析工具使用分布

```bash
# 统计各工具调用次数
cat ~/.claude/superpowers-logs/2026-05-27.jsonl | jq -r '.tool_name' | sort | uniq -c | sort -rn
```

### 4. 追踪会话行为

```bash
# 查看某个会话的完整工具调用序列
cat ~/.claude/superpowers-logs/2026-05-27.jsonl | jq 'select(.session_id=="abc123") | {ts, tool_name, skill_name}'
```

## 不做的事

- **不做实时终端输出** — 仅写文件，不影响用户终端体验
- **不做分析工具或仪表盘** — 只生成日志，后续分析由用户自行处理
- **不做 PreToolUse 捕获** — 单一 PostToolUse 足够，避免双倍 hook 开销
- **不做日志轮转** — 按天分割，用户自行清理旧日志
- **不做敏感信息过滤** — tool_input 可能包含文件内容或命令，由用户自行管理日志文件的安全性
