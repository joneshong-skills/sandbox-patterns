[English](README.md) | [繁體中文](README.zh.md)

# sandbox-patterns

Codified patterns and best practices for `sandbox_execute` MCP tool usage.

## 說明

Sandbox Patterns is a knowledge skill providing recipes, anti-patterns, and decision rules for when and how to use `sandbox_execute` — covering parallel execution, batching, and common use cases.

## 功能特色

- Decision matrix: when to use sandbox vs. direct tool calls
- Parallel execution patterns for independent operations
- Batch data processing recipes (read → transform → write)
- Common anti-patterns and how to avoid them
- SDK helper reference (http_get/post, read_file/write_file, output)
- Real examples from production skill implementations

## 使用方式

透過以下觸發語句呼叫 Claude Code 來使用此技能：

- "sandbox patterns"
- "sandbox best practices"
- "how to use sandbox"
- "沙盒模式"
- "沙盒最佳實踐"

## 相關技能

- [`skill-publisher`](https://github.com/joneshong-skills/skill-publisher)
- [`skill-catalog`](https://github.com/joneshong-skills/skill-catalog)
- [`executor`](https://github.com/joneshong-skills/executor)

## 安裝

將技能目錄複製到 Claude Code 技能資料夾：

```
cp -r sandbox-patterns ~/.claude/skills/
```

放置在 `~/.claude/skills/` 的技能會被 Claude Code 自動發現，無需額外註冊。
