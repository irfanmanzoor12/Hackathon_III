---
id: 003
title: Convert Playbooks to Skills with MCP Code Execution
stage: general
date: 2026-02-07
surface: agent
model: claude-opus-4-6
feature: none
branch: master
user: irfan
command: convert playbooks to skills
labels: ["skills", "mcp-code-execution", "hackathon-iii", "restructure"]
links:
  spec: null
  ticket: null
  adr: null
  pr: null
files:
  - .claude/skills/kafka-k8s-setup/SKILL.md
  - .claude/skills/kafka-k8s-setup/scripts/deploy.sh
  - .claude/skills/kafka-k8s-setup/scripts/verify.py
  - .claude/skills/kafka-k8s-setup/REFERENCE.md
  - .claude/skills/postgres-k8s-setup/SKILL.md
  - .claude/skills/postgres-k8s-setup/scripts/deploy.sh
  - .claude/skills/postgres-k8s-setup/scripts/verify.py
  - .claude/skills/postgres-k8s-setup/REFERENCE.md
  - .claude/skills/agents-md-gen/SKILL.md
  - .claude/skills/agents-md-gen/scripts/generate.sh
  - .claude/skills/fastapi-dapr-agent/SKILL.md
  - .claude/skills/fastapi-dapr-agent/scripts/create_app.sh
  - .claude/skills/fastapi-dapr-agent/scripts/deploy.sh
  - .claude/skills/fastapi-dapr-agent/scripts/verify.py
  - .claude/skills/fastapi-dapr-agent/REFERENCE.md
  - .claude/skills/mcp-code-execution/SKILL.md
  - .claude/skills/mcp-code-execution/scripts/create_server.sh
  - .claude/skills/mcp-code-execution/scripts/deploy.sh
  - .claude/skills/mcp-code-execution/scripts/verify.py
  - .claude/skills/mcp-code-execution/REFERENCE.md
  - .claude/skills/nextjs-k8s-deploy/SKILL.md
  - .claude/skills/nextjs-k8s-deploy/scripts/create_app.sh
  - .claude/skills/nextjs-k8s-deploy/scripts/deploy.sh
  - .claude/skills/nextjs-k8s-deploy/scripts/verify.py
  - .claude/skills/nextjs-k8s-deploy/REFERENCE.md
  - .claude/skills/docusaurus-deploy/SKILL.md
  - .claude/skills/docusaurus-deploy/scripts/create_site.sh
  - .claude/skills/docusaurus-deploy/scripts/deploy.sh
  - .claude/skills/docusaurus-deploy/scripts/verify.py
  - .claude/skills/docusaurus-deploy/REFERENCE.md
  - docs/skill-development-guide.md
tests:
  - SKILL.md word count check (all under ~125 words)
  - File structure verification (all skills have SKILL.md + scripts/)
---

## Prompt

check how much we done and what left?
(followed by full hackathon requirements paste and request to start converting)

User identified that existing playbooks were not in the required Skills format for Hackathon III. The hackathon requires `.claude/skills/` with SKILL.md (~100 tokens) + scripts/ (executable, 0 tokens) + REFERENCE.md (on-demand). Existing playbooks were single large markdown files in `playbooks/` directory — the Hackathon 2 approach, not the required Hackathon 3 MCP Code Execution pattern.

## Response snapshot

Performed gap analysis showing: (1) playbooks are wrong format, (2) mcp-code-execution skill missing entirely, (3) LearnFlow app (Repo 2) not built. Then converted all 8 playbooks into 7 proper Skills:

1. kafka-k8s-setup: SKILL.md + deploy.sh + verify.py + REFERENCE.md
2. postgres-k8s-setup: SKILL.md + deploy.sh + verify.py + REFERENCE.md (built from scratch - playbook was incomplete)
3. agents-md-gen: SKILL.md + generate.sh
4. fastapi-dapr-agent: SKILL.md + create_app.sh + deploy.sh + verify.py + REFERENCE.md
5. mcp-code-execution: SKILL.md + create_server.sh + deploy.sh + verify.py + REFERENCE.md (NEW)
6. nextjs-k8s-deploy: SKILL.md + create_app.sh + deploy.sh + verify.py + REFERENCE.md
7. docusaurus-deploy: SKILL.md + create_site.sh + deploy.sh + verify.py + REFERENCE.md

Plus docs/skill-development-guide.md explaining the pattern.

## Outcome

- ✅ Impact: All 7 required skills created in correct .claude/skills/ format with MCP Code Execution pattern
- 🧪 Tests: SKILL.md word counts verified (89-124 words, all near ~100 token target)
- 📁 Files: 30 new files created across 7 skill directories + 1 guide
- 🔁 Next prompts: Build LearnFlow app (Repo 2), set up skills-library as separate repo
- 🧠 Reflection: The key insight was that Hackathon 3 requires executable scripts (not just documentation). The playbooks had all the knowledge but in the wrong format.

## Evaluation notes (flywheel)

- Failure modes observed: postgres playbook was incomplete (only 22 lines), had to build from other sources
- Graders run and results (PASS/FAIL): PASS - all skills created with correct structure
- Prompt variant (if applicable): N/A
- Next experiment (smallest change to try): Test skills on actual Minikube cluster to validate deploy.sh and verify.py work end-to-end
