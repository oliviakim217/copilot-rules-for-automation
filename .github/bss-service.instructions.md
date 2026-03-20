---
applyTo: "**"
---

# BSS Service — Project-Specific Rules
# Version: 1.0.0

## Project Overview

- **Name**: BSS Service
- **Purpose**: Web UI that lets users create Jira stories and service requests with one click. 
- **Input**: Dropdowns (project, issue type, priority) + 8-digit staff ID + summary/description.
- **Output**: Jira issue key (e.g. `EQDTRG-1234`) as a clickable link.
- **Phase**: Starting with Jira story creation. Designed to expand to other integrations.
- **Users**: Internal staff. Deployed on company internal network.
- **Entry point**: `server3.py` at root level (platform deployment constraint).


## YAML Config Schema

Every `bss_config.yaml` must follow this schema as closely as possible.
Adding a new project or dropdown value = edit YAML only. Zero code changes.
```yaml
version: "1.0.0"
environment: "dev"     # must match APP_ENV

jira:
  project_keys:
    - key: "EQDTRG"
      display_name: "Equipment Tracking"
      default_issue_type: "Story"
    - key: "BSSCORE"
      display_name: "BSS Core"
      default_issue_type: "Story"

  issue_types:
    - "Story"
    - "Bug"
    - "Task"
    - "Epic"

  priority_levels:
    - "Highest"
    - "High"
    - "Medium"
    - "Low"

  field_defaults:
    priority: "Medium"
    story_points: 3
    labels:
      - "bss-service"

  field_mappings:
    summary: "summary"
    description: "description"
    issue_type: "issuetype"
    priority: "priority"
    story_points: "story_points"
    reporter_staff_id: "reporter"

ui:
  staff_id_length: 8
  max_description_chars: 2000

app:
  rate_limit_per_minute: 30
  log_retention_days: 90
  cors_allowed_origins:
    - "http://localhost:8000"
```

## Config Loader Rules

- Config loaded ONCE at app startup. Injected via dependency injection — not re-read per request.
- Environment selected by `APP_ENV` env var → loads `configs/{APP_ENV}/bss_config.yaml`.
- If config file is missing or schema is invalid: fail fast with a `CRITICAL` log on startup.
- `backend/config/config_loader.py` is the ONLY place that reads YAML and environment variables.
- Business logic never calls `os.environ` directly.

## Jira Integration

- **API Reference**: Jira Cloud REST API v3
  `https://developer.atlassian.com/cloud/jira/platform/rest/v3/`
- **Create issue**: `POST {JIRA_BASE_URL}/rest/api/3/issue`
- **Authentication**: Basic Auth — `base64(JIRA_USER_EMAIL:JIRA_API_TOKEN)` in Authorization header.
- Credentials from env vars only: `JIRA_USER_EMAIL`, `JIRA_API_TOKEN`, `JIRA_BASE_URL`.

### Jira Client Rules (`backend/modules/jira/client.py`)

1. This is the ONLY file that makes HTTP calls to Jira. No exceptions.
2. Use `httpx.AsyncClient` with explicit timeout (default 30s, configurable via config).
3. Retry on `503` and `429` with exponential backoff — max 3 retries.
4. Never log the Authorization header or API token.
5. Return a typed result dataclass — never return raw `httpx.Response` outside this module.

### Jira Result Dataclass
```python
@dataclass
class JiraCreateResult:
    success: bool
    jira_issue_key: str = ""       # e.g. "EQDTRG-1234"
    jira_issue_id: str = ""
    jira_issue_url: str = ""
    error_message: str = ""
    http_status_code: int = 0
```

## Input Validation Rules

- `staff_id`: must match `/^\d{8}$/`. Length read from `cfg_ui.staff_id_length` (config, not hardcoded).
- `project_key`: must be in `cfg_jira.project_keys` list loaded from config.
- `issue_type`: must be in `cfg_jira.issue_types` list loaded from config.
- `priority`: must be in `cfg_jira.priority_levels` or fall back to `cfg_jira.field_defaults.priority`.
- `summary`: non-empty, stripped of whitespace, under `cfg_ui.max_description_chars`.

### Validation Result Dataclass
```python
@dataclass
class ValidationResult:
    is_valid: bool
    error_message: str = ""
    error_field: str = ""
    error_code: str = ""
```

Always return a `ValidationResult` — never a bare `bool`. The error field and code are required
to give the user and the logs enough context to understand exactly what failed.

## Frontend Rules

1. Served by FastAPI using `StaticFiles` or Jinja2 templates.
2. Plain HTML + vanilla JS only. No frontend framework, no build pipeline.
3. Dropdowns populated dynamically from `GET /api/v1/config/options` — never hardcoded in HTML.
4. Staff ID input: 8-digit numeric validation on client side AND server side.
5. "Generate" button disabled while request is in-flight (prevent double-submit).
6. On success: display `jira_issue_key` as a clickable link using `jira_issue_url`.
7. On error: display the `message` field from the response envelope clearly to the user.
8. No credentials or tokens anywhere in frontend code or HTML source.

## Environment Variables (`.env.example`)
```
# Jira
JIRA_BASE_URL=https://your-company.atlassian.net
JIRA_USER_EMAIL=your-service-account@company.com
JIRA_API_TOKEN=your_api_token_here

# Application
APP_ENV=dev
BSS_LOG_LEVEL=INFO
BSS_CONFIG_PATH=configs/dev/bss_config.yaml

# Server
BSS_HOST=0.0.0.0
BSS_PORT=8000
```

## PowerShell Test Command
```powershell
Invoke-RestMethod -Method POST -Uri "http://localhost:8000/api/v1/stories" `
  -ContentType "application/json" `
  -Body '{"staff_id":"12345678","project_key":"EQDTRG","issue_type":"Story","summary":"Test story from BSS Service"}'
```