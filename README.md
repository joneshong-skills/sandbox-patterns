[English](README.md) | [繁體中文](README.zh.md)

# sandbox-patterns

Codified patterns and best practices for `sandbox_execute` MCP tool usage.

## Description

Sandbox Patterns is a knowledge skill providing recipes, anti-patterns, and decision rules for when and how to use `sandbox_execute` — covering parallel execution, batching, and common use cases.

## Features

- Decision matrix: when to use sandbox vs. direct tool calls
- Parallel execution patterns for independent operations
- Batch data processing recipes (read → transform → write)
- Common anti-patterns and how to avoid them
- SDK helper reference (http_get/post, read_file/write_file, output)
- Real examples from production skill implementations

## Usage

Invoke by asking Claude Code with trigger phrases such as:

- "sandbox patterns"
- "sandbox best practices"
- "how to use sandbox"
- "沙盒模式"
- "沙盒最佳實踐"

## Related Skills

- [`skill-publisher`](https://github.com/joneshong-skills/skill-publisher)
- [`skill-catalog`](https://github.com/joneshong-skills/skill-catalog)
- [`executor`](https://github.com/joneshong-skills/executor)

## Install

Copy the skill directory into your Claude Code skills folder:

```
cp -r sandbox-patterns ~/.claude/skills/
```

Skills placed in `~/.claude/skills/` are auto-discovered by Claude Code. No additional registration is needed.
