---
applyTo: "**"
---

# Project Structure and FastAPI Patterns
# Version: 1.0.0

## Folder Structure

Follow the established hierarchy as closely as possible. 
```
bss-service/
├── .github/
│   ├── copilot-instructions.md
│   └── instructions/
│       ├── global-rules.instructions.md
│       ├── naming-rules.instructions.md
│       ├── python-rules.instructions.md
│       ├── project-structure.instructions.md
│       ├── bss-service-specific.instructions.md
│       └── scalability-extension.instructions.md
├── server3.py                         # REQUIRED filename. REQUIRED at root level.
│                                      # Platform deployment constraint — do not rename or move.
│                                      # Wires the FastAPI app only. Zero business logic.
├── backend/
│   ├── api/
│   │   └── v1/
│   │       ├── routes.py              # Registers all routers
│   │       └── endpoints/
│   │           └── jira_stories.py    # Jira story endpoint (api_ prefixed handlers)
│   ├── modules/
│   │   └── jira/
│   │       ├── client.py              # Jira HTTP calls only
│   │       ├── validator.py           # Input validation logic
│   │       ├── transformer.py         # Maps form input → Jira payload shape
│   │       └── enricher.py            # Adds defaults, timestamps, metadata
│   ├── services/
│   │   └── story_service.py           # Orchestrates the full pipeline
│   ├── config/
│   │   └── config_loader.py           # Loads YAML, validates schema, exposes typed config
│   ├── utils/
│   │   ├── logger.py                  # Centralized logger configuration
│   │   └── http_client.py             # Shared httpx session with retry logic
│   └── constants.py                   # UPPER_CASE_SNAKE_CASE constants only
├── configs/
│   ├── dev/
│   │   └── bss_config.yaml            # Dev environment YAML config
│   └── prod/
│       └── bss_config.yaml            # Prod environment YAML config
├── frontend/
│   ├── index.html
│   ├── static/
│   │   ├── css/
│   │   └── js/
│   └── templates/                     # Jinja2 templates if used
├── tests/
│   ├── test_data/
│   │   ├── valid_story_input.json
│   │   ├── invalid_staff_id.json
│   │   └── missing_required_fields.json
│   ├── unit/
│   └── integration/
├── logs/                              # Gitignored. Log files only.
├── .env                               # Gitignored. Secrets and URLs only.
├── .env.example                       # Committed. All required env var names, no values.
├── requirements.txt                   # Pinned dependency versions
└── README.md
└── references/
```

## Responsibility Rules

- `server3.py`: REQUIRED filename at root level (platform deployment constraint).
  Creates the FastAPI app, registers routers, configures CORS and middleware. Nothing else.
  Do NOT rename. Do NOT move. All business logic belongs in `backend/`.

- `backend/api/v1/endpoints/`: HTTP handlers only. Call service layer. Return response envelope.

- `backend/services/`: Orchestration only. Calls validator → transformer → enricher → client.
  No direct API calls. No raw HTTP.

- `backend/modules/<integration>/client.py`: The ONLY place that makes HTTP calls to that
  external API. Nothing outside this file calls the external API directly.

- `backend/config/config_loader.py`: The ONLY place that reads YAML files and environment
  variables. Business logic never calls `os.environ` directly.

- `configs/dev/` and `configs/prod/`: YAML data files only. Not Python code.
  These are separate from `backend/config/` which contains Python loader code.

- `backend/constants.py`: The ONLY place for app-wide constant values.

## FastAPI App Setup
```python
# server3.py
# REQUIRED filename. REQUIRED at root level. Platform deployment constraint.
# Do not rename. Do not move. Do not add business logic here.

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from backend.api.v1.routes import bss_v1_router
from backend.config.config_loader import load_config

cfg_app = load_config()

app = FastAPI(title="BSS Service", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=cfg_app.cors_allowed_origins,
    allow_methods=["GET", "POST"],
    allow_headers=["Content-Type"],
)

app.include_router(bss_v1_router, prefix="/api/v1")
```

## Pydantic Request/Response Models
```python
from pydantic import BaseModel
from typing import Optional, Any

class CreateStoryInput(BaseModel):
    staff_id: str
    project_key: str
    issue_type: str
    summary: str
    description: Optional[str] = None
    priority: Optional[str] = None

class BssApiResponse(BaseModel):
    status: str          # "success" | "error"
    message: str
    data: Optional[Any] = None
```

## Standard Response Envelope

Every API response — success or error — must use this exact shape:
```json
{
  "status": "success | error",
  "message": "Human-readable outcome description",
  "data": {}
}
```

Success example:
```json
{
  "status": "success",
  "message": "Jira story created successfully",
  "data": {
    "jira_issue_key": "EQDTRG-1234",
    "jira_issue_url": "https://your-company.atlassian.net/browse/EQDTRG-1234",
    "jira_issue_id": "10042"
  }
}
```

Error example:
```json
{
  "status": "error",
  "message": "Validation failed: staff_id must be exactly 8 digits",
  "data": {
    "field": "staff_id",
    "provided_value": "1234",
    "error_code": "INVALID_STAFF_ID"
  }
}
```

## HTTP Status Codes

- `200` — success
- `400` — validation error (bad user input)
- `422` — Pydantic model validation failure (automatic)
- `500` — internal or Jira API error
- `503` — Jira API unreachable


## API Versioning

- All endpoints live under `/api/v1/`. No exceptions.
- Breaking changes go in `/api/v2/` — never modify existing versioned routes.

## CORS

- Dev: allow all origins.
- Prod: restrict to `cfg_app.cors_allowed_origins` list loaded from config.