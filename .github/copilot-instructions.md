# BSS Service — Copilot Instructions Index
# Version: 1.0.0
# This file is the entry point. GitHub Copilot always loads this file.
# All detailed rules are split into focused instruction files under .github/instructions/

## Project Summary
BSS Service is a Python/FastAPI web application that allows users to create Jira stories
via a simple UI form (dropdowns + 8-digit staff ID) with one click.

## Primary Goal
Scalable, config-driven, modular architecture.
The code itself should be the LAST thing that changes when adding features.

## Instruction Files (always apply all of these)
- .github/instructions/global-rules.instructions.md         — production mindset, logging, error handling
- .github/instructions/naming-rules.instructions.md         — all naming conventions
- .github/instructions/python-rules.instructions.md         — Python-specific coding standards
- .github/instructions/project-structure.instructions.md    — folder layout, FastAPI patterns, API design
- .github/instructions/bss-service-specific.instructions.md — Jira integration, config schema, UI rules
- .github/instructions/scalability-extension.instructions.md — how to extend without modifying existing code

## Non-Negotiable Rules (summary)
1. Entry point is `server3.py` at the root level. This name and location are fixed by the
   deployment platform and must never be changed. All logic lives in `backend/` —
   `server3.py` only wires the app together.
2. Config-driven: business rules, dropdowns, field mappings live in YAML — not in code.
3. Layered pipeline: Input → Validate → Transform → Enrich → API Call → Response
4. Jira client is the ONLY place that makes HTTP calls to Jira.
5. All API responses use the standard envelope: { status, message, data }
6. All endpoints are versioned under /api/v1/
7. Credentials and URLs come from .env only — never from YAML or hardcoded.
8. Guardrail: generate only what is asked. No speculative abstractions. Ask me to clarify if unsure.