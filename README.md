# claude-daily-report

根据 Claude Code 的 session 历史自动生成每日工作日报。

## 功能

自动读取当天所有 session，无需手动输入 session ID，生成一份人类能看懂的日报，包含：

- **Token 消耗**：今天花了多少 tokens，哪些 session 最耗
- **重复工作分析**：哪些文件/指令被反复修改，效率瓶颈在哪
- **人机协同优化建议**：可以改进的协作模式
- **今日完成事项**：做了什么、做到什么程度、结果如何
- **明日计划**：按 P0/P1/P2 排序的推荐待办
- **一句话总结**：一天干了啥，一句话交代清楚

## 怎么用

把这个 skill 放到你的 Claude Code skills 目录中：

```bash
# macOS/Linux
cp -r claude-daily-report ~/.claude/skills/

# Windows (Git Bash / WSL)
cp -r claude-daily-report ~/.claude/skills/
```

然后在对话里说：
- "写个日报"
- "今天做了什么"
- "日报"
- "今日总结"

它会自动读取当天的所有 session，直接出报告。

## 项目结构

```
claude-daily-report/
├── SKILL.md      # Claude Code skill 定义文件
└── README.md     # 说明文档
```

## 报告结构

```
Token 消耗
  ↓
重复工作分析
  ↓
人机协同优化建议
  ↓
今日做了什么（进度 + 结果）
  ↓
明日计划（P0/P1/P2）
  ↓
一句话总结
```

## 数据从哪来

Claude Code 的 session 历史存在：
```
~/.claude/projects/{workspace}/<session-id>.jsonl
```

这个技能直接解析这些文件，提取用户提示词、工具调用、文件修改记录、token 消耗等数据，然后汇总成日报。

## 注意事项

- 只支持当前 workspace 的 session 历史
- 跨天的 session（如从昨晚持续到凌晨）会被归到文件修改日期对应的天
- 如果今天没有 session，会明确告知而不是输出空报告
