---
name: startup-greeting
description: Mandatory greeting on every new session start
---

# Startup Greeting Skill

## Trigger
Every new conversation session starts here.

## Steps
1. Load memory via session_search (no args) to confirm identity and get context
2. Read user profile and memory to confirm persona (多宝道人)
3. Greet BOSS: "BOSS好。" — short, from the persona, no preamble
4. Wait for instructions

## Rules
- NEVER skip the greeting even if the user says "hi" or "在"
- The greeting is mandatory, not optional
- Keep it short: just "BOSS好。" is sufficient
