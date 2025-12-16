# AGENTS.md

## Repository Purpose
This repository is part of Hackathon III: Reusable Intelligence and Cloud-Native Mastery.

The goal is to create vendor-neutral CAPS playbooks that compile into:
- Claude Code Skills
- Goose Recipes

---

## Agent Operating Rules

### Before Taking Action
1. Read PROJECT_STATUS.md (single source of truth for project state)
2. Read DECISIONS.md (architectural and process decisions)
3. Read .specify/memory/constitution.md (code quality and standards)

### During Execution
- Follow spec-driven development (spec → plan → tasks → implementation)
- Prefer CAPS playbooks over manual code
- Optimize for autonomous execution (single prompt → deployment)
- Use available tools in .specify/scripts/bash/ for automation
- Execute slash commands from .claude/commands/ when appropriate

---

## Key Directories

### Source of Truth
- `playbooks/` - CAPS playbooks (vendor-neutral agent instructions)
- `DECISIONS.md` - Architectural and process decisions
- `PROJECT_STATUS.md` - Current project state

### Agent Configuration
- `.claude/commands/` - Claude Code slash commands (sp.* commands)
- `.specify/memory/` - Project constitution and standards
- `.specify/scripts/bash/` - Automation scripts for development workflows

### Future Output (Not Yet Generated)
- `.claude/skills/` - Generated Claude Code skills (from CAPS)
- `recipes/` - Generated Goose recipes (from CAPS)

---

## Standards
- Follow Agentic AI Foundation (AAIF) principles
- Kubernetes-first, cloud-native design
- Event-driven architecture where applicable
- Maximum autonomy: Single prompt → running system
- Local-first development (WSL, Minikube, Docker)

---

## Workflow for Both Claude Code and Goose
1. Read PROJECT_STATUS.md to understand current state
2. Read DECISIONS.md for architectural context
3. If working on a feature: check playbooks/ for CAPS playbook
4. If CAPS playbook exists: execute it, don't write manual code
5. If creating new capability: write CAPS playbook first
6. Validate dual-agent compatibility (both Claude Code and Goose must work)
