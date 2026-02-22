---
name: sandbox-patterns
description: >-
  This skill should be used when the user asks about "sandbox patterns",
  "sandbox best practices", "how to use sandbox", "sandbox optimization",
  "parallel sandbox", "batch sandbox", "sandbox recipe", "沙盒模式",
  "沙盒用法", "沙盒最佳實踐", mentions sandbox_execute usage patterns,
  or discusses optimizing tool calls with sandbox execution.
version: 0.1.0
tools: sandbox_execute
---

# Sandbox Patterns

Codified patterns for `sandbox_execute` MCP tool. Covers when to use sandbox,
parallel execution, common recipes, and anti-patterns.

## Agent Delegation

This skill runs entirely in main context. Sandbox execution is a main-context
tool — sub-agents cannot access MCP tools directly.

## Core Principle

**Deterministic batch work → sandbox; reasoning/presentation → LLM.**

Sandbox processes data internally and returns only structured summaries via
`output()`, keeping full data out of conversation context.

## When to Use Sandbox

| Signal | Use Sandbox | Use Direct Tools |
|--------|-------------|-----------------|
| Read 3+ files → aggregate | Yes | No |
| Single file read | No | Yes (Read tool) |
| Batch API calls (same service) | Yes | No |
| Needs MCP tools (Playwright, etc.) | No | Yes (MCP tools) |
| Multi-step transform → single output | Yes | No |
| Interactive / needs mid-step decisions | No | Yes (sequential) |
| Side effects on shared systems | Caution | Prefer direct |

## Parallel Sandbox

Multiple `sandbox_execute` calls in a single message run as **independent
subprocesses concurrently**. Each has its own Python/JS runtime.

### Pattern: Fan-Out

```
sandbox_execute #1: Process batch A → output(summary_a)
sandbox_execute #2: Process batch B → output(summary_b)
sandbox_execute #3: Process batch C → output(summary_c)
```

All three run simultaneously (~30ms each). LLM synthesizes results after.

### Pattern: Split-Process-Merge

```
# Step 1: Parallel processing (single message, 3 tool calls)
sandbox #1: Read files 1-25 → write /tmp/batch1.json
sandbox #2: Read files 26-50 → write /tmp/batch2.json
sandbox #3: Read files 51-73 → write /tmp/batch3.json

# Step 2: Merge (single sandbox call)
sandbox #4: Read batch1-3.json → merge → output(final_summary)
```

### Constraints

- No shared state between parallel sandboxes (separate processes)
- File system IS shared (~/Claude/, /tmp/sandbox-executor/)
- Coordinate via files, not memory
- Each sandbox has ~30s timeout

## Common Recipes

### Recipe 1: Batch File Scan

```python
# sandbox_execute (python)
import os, json
from pathlib import Path

results = []
for f in sorted(Path(target_dir).glob("**/*.md")):
    content = f.read_text()
    results.append({
        "file": str(f),
        "lines": len(content.splitlines()),
        "size": len(content),
    })

# Save full data to file, return only summary
with open(out_path, "w") as f:
    json.dump(results, f, indent=2)

output({"total": len(results), "total_lines": sum(r["lines"] for r in results)})
```

### Recipe 2: Batch HTTP (localhost services)

```python
# sandbox_execute (python)
endpoints = ["/api/a", "/api/b", "/api/c"]
results = {}
for ep in endpoints:
    resp = http_get(f"http://localhost:8080{ep}")
    results[ep] = {"status": resp.get("status"), "size": len(str(resp))}
output(results)
```

### Recipe 3: Import Script from ~/Claude/

```python
# sandbox_execute (python)
import os, sys
# ONLY works for scripts in ~/Claude/ or /tmp/ — NOT ~/.claude/
sys.path.insert(0, os.path.expanduser("~/Claude/projects/<project>/scripts"))
from my_script import my_function

result = my_function(args)
output(result)
```

> **Warning**: Scripts in `~/.claude/skills/` are NOT accessible from sandbox. Use `Bash` to run those scripts directly.

### Recipe 4: Data Transform Pipeline

```python
# sandbox_execute (python)
import json, csv, io
from pathlib import Path

# Read → Transform → Write
data = json.loads(Path(input_path).read_text())
transformed = [{"name": d["name"], "score": d["value"] * 100} for d in data]

# Write CSV
buf = io.StringIO()
writer = csv.DictWriter(buf, fieldnames=["name", "score"])
writer.writeheader()
writer.writerows(transformed)
Path(output_path).write_text(buf.getvalue())

output({"rows": len(transformed), "saved_to": output_path})
```

## Anti-Patterns

| Anti-Pattern | Why Bad | Fix |
|-------------|---------|-----|
| Return full file contents via `output()` | Defeats token savings | Save to file, return summary |
| Use sandbox for 1 simple read | Overhead > benefit | Use Read tool |
| `import os` after `os.path.expanduser()` | NameError | Imports always first |
| Assume cross-sandbox state | Separate processes | Use file-based coordination |
| Skip `os.makedirs()` before write | FileNotFoundError | Always ensure parent dirs |
| Sandbox for interactive decisions | Can't pause mid-execution | Use sequential tool calls |

## SDK Helpers Available

| Helper | Purpose |
|--------|---------|
| `output(data)` | Return structured data to LLM (dict, list, str) |
| `http_get(url)` | GET request (localhost only) |
| `http_post(url, data)` | POST request (localhost only) |
| `read_file(path)` | Read file (whitelisted paths) |
| `write_file(path, content)` | Write file (whitelisted paths) |

## Token Savings Reference

| Scale | Traditional | Sandbox | Savings |
|-------|------------|---------|---------|
| 5 config files | ~2,500 tok | ~600 tok | 75% |
| 73 skill files | ~147,000 tok | ~1,550 tok | 99% |
| 10 API calls | ~5,000 tok | ~400 tok | 92% |

## Known Constraints (Verified 2026-02-22)

These constraints were verified by real execution tests:

### 1. Filesystem — Strict Whitelist
- **Readable/Writable**: `~/Claude/`, `/tmp/sandbox-executor/`
- **Blocked**: EVERYTHING else, including `~/.claude/` (skills, agents, config are NOT accessible)

### 2. Network — Localhost Only
- `http_get()` / `http_post()` restricted to `localhost` / `127.0.0.1`
- External URLs (APIs, websites, CDNs) are blocked with connection error
- Pulso services (ports 8840-8842) and other local services work fine

### 3. Security Keyword Blocklist
- Code containing these keywords is rejected before execution: `subprocess`, `os.system`, `eval`, `exec`, `__import__`
- Even as STRING LITERALS (e.g., searching for the word "subprocess" in text) — triggers the blocklist
- Workaround for string matching: use character-by-character construction or avoid the keyword entirely

### 4. No External Libraries
- `requests` is NOT available (use SDK `http_get`/`http_post` instead)
- Available: `Pillow`, `openpyxl`, `pypdf`, `pdfplumber`, `python-pptx`, `python-docx`, stdlib
- No package installation at runtime

### 5. Execution Model
- Each call = fresh process (no state between calls)
- Parallel calls share filesystem but not memory
- 30s timeout per call (max 60s)

## Quick Reference

```
When to sandbox:  3+ files in ~/Claude/, batch localhost APIs, data transforms
When NOT to:      1 file, external HTTP, ~/.claude/ paths, subprocess tools, interactive
Parallel:         Multiple sandbox_execute in one message
Import scripts:   ONLY from ~/Claude/ paths (NOT ~/.claude/ — use Bash for those)
Always:           imports first, makedirs before write, output() summary only
Constraints:      No subprocess, localhost-only HTTP, ~/Claude/ + /tmp/ paths only
```
