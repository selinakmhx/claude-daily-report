---
name: claude-daily-report
description: Generate a daily work report from Claude Code session history. Automatically reads all today's sessions — no manual input needed. Use when the user says "write my daily report", "summarize today's Claude sessions", "what did I do today", "give me a daily summary", "日报", "今日总结". Also trigger when the user wants to know what they accomplished in Claude Code today.
---

# Claude Daily Report

Generates a daily work report from Claude Code session logs. The key principle: **write like a human**, not a machine. Group by actual work/tasks, not by session. Use natural language. Make it readable and traceable.

## Data source

Claude Code stores conversation history in:
- `~/.claude/projects/{workspace}/<session-id>.jsonl` — full conversation per session
- `~/.claude/sessions/` — session metadata (start time, cwd, working directory)

Each `.jsonl` file contains one JSON object per line with types:
- `ai-title` — session title/topic
- `user` — user messages (the actual prompts)
- `assistant` — AI responses (contain tool calls, text output, usage stats)
- `queue-operation` — enqueue/dequeue events (timing markers)

## Workflow

### Step 1: Extract data

Write and execute a Python script to parse all `*.jsonl` files for today's date (check file modification date). Extract from each session:

**Session metadata:**
- `ai-title` → session topic
- First/last timestamps → duration
- CWD → working directory

**Token usage (from assistant `message.usage`):**
- `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
- Sum across all assistant messages per session for session totals
- Grand total across all sessions

**User prompts:**
- Filter for `type: "user"` with non-empty content
- Skip: system prompt injections (content containing "Base directory for this skill" or "You are Claude Code"), empty messages, `task-notification` messages, `<ide_` tags

**Tool calls (from assistant messages):**
- File edits/creates: `tool_use` with name Edit/Write/NotebookEdit → extract `input.file_path`
- Git commits: Bash tool calls containing `git commit` → extract commit info
- Agent delegations: Agent tool calls → extract `input.description` or name

**Then synthesize:** group sessions by what they were actually working on. Multiple sessions about the same topic = one task. Merge the file edits. Merge the timeline.

### Step 2: Analyze patterns

From the extracted data, identify:

**重复修改分析:**
- Count file edit frequency across sessions (by filename, not full path)
- Files edited 5+ times across multiple sessions are "重复修改重灾区"
- For each hot-spot file, list which sessions touched it and how many times

**重复指令/任务模式:**
- Look for similar user prompt intents across sessions (e.g., "审核流程", "帮我看看", "优化")
- Identify patterns where the same type of task was done in multiple separate sessions

**人机协同效率问题:**
- "试错式"调试：同一文件短时间内被反复修改（如 10+ 次 Edit 到同一文件），说明边试边改而非先规划
- 重复审核：同一类审核请求在不同 session 中反复出现
- 草稿反复调：笔记/文档生成后 step 文件被大量修改，说明一次性生成质量不够

### Step 3: Generate the report

Language: mixed Chinese and English (技术术语保留英文，其余用中文).

The report has these sections in order:

```markdown
# 工作日报 - YYYY-MM-DD

## Token 消耗

| 指标 | 数值 |
|-----|------|
| API 调用次数 | X 次 |
| 总 token 数 | XX.XM |
| 估算成本 | ~$XX（Sonnet 定价） |
| 单次调用平均 | XXK tokens |

**Token 消耗 TOP 5 的 session：**

| Session | 时间 | Token | 说明 |
|---------|------|-------|------|
| ... | ... | ... | ... |

笔记生成类（X 篇）合计约 XXM tokens，占总消耗 XX%。

## 重复工作分析

### 1. [最高频的重复修改]

[文件名/任务] 被 [修改次数] 次。[简要描述这个重复模式]。

**可优化**：[一句话建议]

### 2. [第二高频]
...

## 人机协同优化建议

| 问题 | 现状 | 建议 |
|------|------|------|
| [问题名称] | [当前怎么做的] | [怎么改] |
| ... | ... | ... |

## 今日做了什么、到什么程度、结果如何

### 任务 1：[任务名称]（HH:MM ~ HH:MM，约 Xh）

| 子任务 | 进度 | 结果 |
|--------|------|------|
| [子任务 A] | ✅ 完成 / ⚠️ 部分完成 | [具体结果] |
| [子任务 B] | ✅ 完成 | [具体结果] |

### 任务 2：[任务名称]

[同上格式]

## 明日计划

| 优先级 | 任务 | 说明 |
|--------|------|------|
| P0 | [最紧急的] | [为什么紧急] |
| P1 | [重要的] | [具体做什么] |
| P2 | [可以稍后的] | [说明] |

## 一句话总结

今天 [花了多久/做了什么]，核心成果是 [1-2 个最重要的结果]，遗留问题是 [还没搞定的事]。
```

### Section-specific guidance

**Token 消耗:**
- Calculate cost using Sonnet pricing: $3/M input tokens, $15/M output tokens
- Group note-generation sessions together if they're all about producing the same type of content
- Highlight the % of tokens spent on generation vs. debugging/optimization

**重复工作分析:**
- Use actual counts from the parsed data — don't guess
- Focus on the TOP 3-4 patterns, not every minor repetition
- Each pattern should have: what was repeated, how many times, why it's wasteful, how to fix

**人机协同优化建议:**
- Only include patterns that appeared 3+ times (significant, not one-off)
- The "建议" column should be actionable, not vague
- Examples: "先用 plan 文档规划再批量改"、"Audit 结果固化为 checklist"

**今日做了什么:**
- Group by task/project, NOT by session
- Use ✅ for fully done, ⚠️ for partially done (needs follow-up)
- "结果" column should be concrete — what files changed, what was produced

**明日计划:**
- P0 = must do tomorrow (usually unfinished work from today)
- P1 = important but not blocking
- P2 = nice to have, maintenance tasks
- Limit to 5 items max

**一句话总结:**
- One sentence. Really. Not a paragraph.
- Cover: time spent + core achievement + biggest unresolved issue
- Example: "今天花了 6 小时 debug 和迭代小红书笔记技能，修复了 4 个 bug 并生成了 3 篇笔记，但 workflow 的 fix 还没做完整回归测试，是明天 P0。"

## Parsing tips

- Empty user messages (content is empty string or whitespace only) should be skipped
- System prompt chunks (`type: "user"` with huge content starting with "Base directory for this skill") are not real user prompts — skip them
- `task-notification` messages in user content are system events, not user prompts — skip them
- `ai-title` is usually a good summary of session intent
- To determine if a jsonl file is from today: check the file's modification date, and also the latest timestamp inside the file
- For timing: use the first and last `timestamp` fields from user/assistant messages (not queue operations)
- Use `Agent` subagent task titles/names to understand what was delegated
- Token usage is in `message.usage` of assistant messages, NOT in user messages
- For file edit frequency, compare by filename (basename) not full path — same file name from different sessions = same file being touched repeatedly

## Edge cases

- If a session is still active, include it but mark as "进行中"
- If no sessions from today, tell the user clearly rather than producing empty output
- If a session has very few interactions (under 5 exchanges), include it but mark as "brief session"
- For large jsonl files (>5MB), use targeted reading (search for key message types) rather than loading everything into memory at once
- If multiple sessions are about the same topic, merge them into one task entry
