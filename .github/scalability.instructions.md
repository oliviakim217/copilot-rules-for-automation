---
applyTo: "**"
---

# Scalability and Extension Contract
# Version: 1.0.0
# This file defines HOW to extend BSS Service without modifying existing code.
# All teams adding features must follow these rules.

## Core Principle

The code itself is the LAST thing that changes.
Every extension point below requires config changes or new files — not edits to existing modules.

## Extension Patterns

### Add a new Jira project
→ Edit `configs/dev/bss_config.yaml` and `configs/prod/bss_config.yaml`
→ Add entry under `jira.project_keys`
→ Zero code changes required
→ The `/api/v1/config/options` endpoint returns the new project automatically

### Add a new dropdown option (issue type, priority)
→ Edit the relevant list in YAML config only
→ Zero code changes required

### Add a new field to the Jira story form
→ Add field to `CreateStoryInput` Pydantic model
→ Add field mapping to `bss_config.yaml` under `jira.field_mappings`
→ Add validation rule to `backend/modules/jira/validator.py`
→ Add transformation logic to `backend/modules/jira/transformer.py`
→ No changes to `client.py`, `server3.py`, `routes.py`, or `story_service.py`

### Add a new external integration (e.g. Confluence, ServiceNow, GitHub)
→ Create new folder: `backend/modules/<service_name>/`
→ Add these files inside it:
   - `client.py`       — HTTP calls to that service only
   - `validator.py`    — input validation for that service
   - `transformer.py`  — maps form input → that service's payload shape
   - `enricher.py`     — adds defaults and metadata
→ Create a new service: `backend/services/<feature>_service.py`
→ Create a new endpoint: `backend/api/v1/endpoints/<feature>.py`
→ Register the router in `backend/api/v1/routes.py`
→ Add a config section to `bss_config.yaml`
→ Do NOT modify any existing module, client, or service

### Add a new API version (breaking change)
→ Create `backend/api/v2/` with new endpoints
→ Keep `v1` intact and unchanged
→ Both versions run simultaneously during transition

## Rules for Teams Adding Code

1. Your module goes under `backend/modules/<your_service_name>/`. Do not put code elsewhere.
2. Your client file is the ONLY file that calls your external API.
3. Your config section goes under a new top-level key in `bss_config.yaml`.
4. Your env vars use a prefix matching your service: `CONFLUENCE_`, `SNOW_`, `GITHUB_`, etc.
5. Follow the same pipeline: Validate → Transform → Enrich → Call API → Return typed result.
6. Follow all naming conventions in `naming-rules.instructions.md`.
7. Your endpoint returns the same standard response envelope as every other endpoint.
8. Write test data files in `tests/test_data/` covering your integration scenarios.
9. Do not modify `server3.py`, `constants.py`, `config_loader.py`, or another team's modules.
10. Never rename or move `server3.py`. It is a platform deployment constraint, not a design choice.

## Code Generation Checklist

Before finalizing any generated code, verify:

- [ ] Folder placement follows the structure in `project-structure.instructions.md`
- [ ] All config values come from YAML — nothing is hardcoded
- [ ] Every function has type hints, a docstring, and a try/except where appropriate
- [ ] Logging uses `BEGIN:/END:/ERROR:` markers with `duration_ms` on external calls
- [ ] Naming follows `naming-rules.instructions.md`
- [ ] Jira HTTP calls exist only in `backend/modules/jira/client.py`
- [ ] API response uses the standard envelope `{ status, message, data }`
- [ ] Input goes through full pipeline: Pydantic → business validator → transform → enrich
- [ ] Code is testable without modifying `server3.py`
- [ ] No `print()` statements anywhere
- [ ] No secrets, tokens, or URLs hardcoded anywhere
- [ ] `requests` library is not used — `httpx` only