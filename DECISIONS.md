# Architectural & Process Decisions

## Decision 001: CAPS as Source of Truth
All agent instructions are written in CAPS format and compiled into
Claude Code Skills and Goose Recipes.

## Decision 002: Dual-Agent Validation
Every capability must work with:
- Claude Code
- Goose

## Decision 003: Local-First Development
Development and testing happen locally using:
- WSL
- Minikube
- Docker

## Decision 004: Maximum Autonomy
Success is defined as:
Single prompt → running system with no manual steps.
