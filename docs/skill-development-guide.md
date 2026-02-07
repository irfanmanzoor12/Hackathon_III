# Skill Development Guide

## Skills with MCP Code Execution Pattern

This guide explains how to create Skills that follow the MCP Code Execution pattern for maximum token efficiency and cross-agent compatibility.

## The Problem: Token Bloat

When MCP servers connect directly to an agent, all tool definitions load into context at startup:

| MCP Servers | Token Cost Before Conversation |
|-------------|-------------------------------|
| 1 server (5 tools) | ~10,000 tokens |
| 3 servers (15 tools) | ~30,000 tokens |
| 5 servers (25 tools) | ~50,000+ tokens |

With 5 MCP servers, you consume 25% of context before typing a single prompt.

## The Solution: Skills + Code Execution

Instead of loading tools into context, wrap them in Skills that execute scripts:

```
SKILL.md → Agent loads (~100 tokens)
scripts/ → Code executes outside context (0 tokens)
REFERENCE.md → Deep docs loaded only if needed (0 tokens)
```

**Result**: ~110 tokens vs 50,000+ tokens.

## Directory Structure

```
.claude/skills/<skill-name>/
├── SKILL.md          # ~100 tokens - what agent loads
├── REFERENCE.md      # 0 tokens - loaded on-demand
└── scripts/
    ├── deploy.sh     # 0 tokens - executed
    ├── verify.py     # 0 tokens - executed
    └── create_app.sh # 0 tokens - executed (if applicable)
```

## Creating a New Skill

### Step 1: Create SKILL.md (~100 tokens)

```yaml
---
name: my-skill
description: What this skill does and when Claude should use it
---

# My Skill

## When to Use
- Trigger condition 1
- Trigger condition 2

## Instructions
1. Run: `bash .claude/skills/my-skill/scripts/deploy.sh`
2. Verify: `python3 .claude/skills/my-skill/scripts/verify.py`

## Validation
- [ ] Check 1
- [ ] Check 2

See [REFERENCE.md](./REFERENCE.md) for details.
```

**Rules**:
- Keep under 100 tokens (~50-80 words)
- Include clear trigger conditions in "When to Use"
- Instructions should reference scripts only
- Link to REFERENCE.md for details

### Step 2: Create scripts/

**deploy.sh** pattern:
```bash
#!/bin/bash
set -e

echo "=== My Skill Deployment ==="

# 1. Check prerequisites
echo "Checking prerequisites..."
# ... checks ...
echo "✓ Prerequisites OK"

# 2. Deploy
echo "Deploying..."
# ... deployment commands ...
echo "✓ Deployed"

# Only this enters agent context:
echo "=== ✓ Deployment complete ==="
```

**verify.py** pattern:
```python
#!/usr/bin/env python3
import subprocess, json, sys

def run(cmd):
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    return result.stdout.strip() if result.returncode == 0 else None

def main():
    passed = 0
    total = 3

    # Check 1
    print("1. Checking...")
    # ... validation ...
    passed += 1

    # Summary - only this enters context
    print(f"\n=== {passed}/{total} checks passed ===")
    sys.exit(0 if passed == total else 1)

if __name__ == "__main__":
    main()
```

**Rules**:
- Scripts must be executable (`chmod +x`)
- Output should be minimal (just status lines)
- Exit code 0 = success, 1 = failure
- Never print large data dumps

### Step 3: Create REFERENCE.md

Put everything detailed here:
- Configuration options
- API documentation
- Common pitfalls
- Agent hints
- Production configuration
- Cleanup commands
- External references

This file is **never loaded** unless the agent explicitly needs it.

### Step 4: Make scripts executable

```bash
chmod +x .claude/skills/my-skill/scripts/*.sh
chmod +x .claude/skills/my-skill/scripts/*.py
```

## Token Budget

| Component | Tokens | Loaded When |
|-----------|--------|-------------|
| SKILL.md | ~100 | Skill triggered |
| REFERENCE.md | 0 | Only if agent reads it |
| scripts/*.sh | 0 | Executed, never loaded |
| scripts/*.py | 0 | Executed, never loaded |
| Script output | ~10-20 | After execution |

**Total per skill invocation**: ~110-120 tokens

## Cross-Agent Compatibility

Skills in `.claude/skills/` work with:
- **Claude Code**: Natively discovers and loads SKILL.md
- **Goose**: Reads `.claude/skills/` directory for recipes
- **Codex**: Supports the same Skills format

No transpilation or conversion needed.

## Existing Skills in This Repository

| Skill | Purpose | Scripts |
|-------|---------|---------|
| `kafka-k8s-setup` | Deploy Kafka on K8s | deploy.sh, verify.py |
| `postgres-k8s-setup` | Deploy PostgreSQL on K8s | deploy.sh, verify.py |
| `agents-md-gen` | Generate AGENTS.md | generate.sh |
| `fastapi-dapr-agent` | FastAPI + Dapr microservice | create_app.sh, deploy.sh, verify.py |
| `mcp-code-execution` | MCP server with code execution | create_server.sh, deploy.sh, verify.py |
| `nextjs-k8s-deploy` | Next.js frontend on K8s | create_app.sh, deploy.sh, verify.py |
| `docusaurus-deploy` | Docusaurus docs on K8s | create_site.sh, deploy.sh, verify.py |

## Best Practices

1. **SKILL.md under 100 tokens** — if it's longer, move details to REFERENCE.md
2. **Scripts return minimal output** — just pass/fail status lines
3. **Scripts are idempotent** — safe to re-run (`--dry-run=client`, `helm upgrade --install`)
4. **No hardcoded secrets** — use K8s Secrets and env vars
5. **Verify before proceeding** — always run verify.py after deploy.sh
6. **Prerequisites check** — every deploy.sh starts with checks
