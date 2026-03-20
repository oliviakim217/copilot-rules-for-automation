---
applyTo: "**/*.py"
---

# Naming Conventions
# Version: 1.0.0

## Functions

1. Use `snake_case`. Functions are verbs or verb phrases.
2. FastAPI endpoint handlers: prefix with `api_`
   - `api_create_story()`, `api_health_check()`, `api_get_config_options()`
3. Jira client functions: clean domain names, no `api_` prefix
   - `create_issue()`, `fetch_project_metadata()`
4. Internal/private functions: prefix with `_`
   - `_validate_staff_id()`, `_build_issue_payload()`
5. Service layer orchestrators: verb + noun
   - `submit_story_request()`, `process_story_creation()`
6. Domain prefixes for grouped functions:
   - `jira_` — Jira-related helpers
   - `bss_` — BSS Service business logic
   - `cfg_` — config value handling

## Variables

1. Use `snake_case`. Always descriptive — no `data`, `tmp`, `x`, `result`, `obj`.
2. Domain-specific naming:
   - `jira_issue_key` not `key`
   - `staff_id_raw` / `staff_id_clean`
   - `cfg_project_keys`, `cfg_issue_types`
   - `env_jira_base_url`, `env_app_env`
   - `bss_story_payload`
3. Booleans read like conditions: `is_valid`, `has_errors`, `should_retry`, `is_prod_env`
4. Never shadow Python builtins: `list`, `dict`, `id`, `type`, `input`
5. Use `.get()` for safe dict access: `cfg_value = config_dict.get("key", "default")`
6. IMPORTANT: Do not use generic names. Be specific so that it's easy to maintain and is scalable. Consistency is key.

## Constants

1. `UPPER_CASE_SNAKE_CASE` only.
2. Defined in `constants.py` only — never inline in logic.
3. Examples: `STAFF_ID_LENGTH = 8`, `DEFAULT_PRIORITY = "Medium"`, `JIRA_API_VERSION = "3"`

## Classes

1. `PascalCase`. Class names are nouns, not verbs.
2. Apply consistent suffixes:
   - `Client` — external API access: `JiraClient`
   - `Service` — business orchestration: `StoryService`
   - `Validator`, `Transformer`, `Enricher`
   - `Config` — config models: `BssConfig`
   - `Input`, `Response` — Pydantic models: `CreateStoryInput`, `CreateStoryResponse`
3. Pydantic models reflect data shape, not behavior.
4. One class = one clear responsibility.

## Files and Modules

1. `snake_case` for all file names.
2. File name reflects the responsibility of its contents:
   - `jira_client.py`, `story_service.py`, `config_loader.py`, `jira_stories.py`
3. No generic names: avoid `helpers.py`, `utils2.py`, `misc.py`

## Environment Variables

1. `UPPER_CASE` with project/service prefix to avoid collisions.
2. Jira: `JIRA_BASE_URL`, `JIRA_USER_EMAIL`, `JIRA_API_TOKEN`
3. App: `APP_ENV`, `BSS_LOG_LEVEL`, `BSS_CONFIG_PATH`, `BSS_HOST`, `BSS_PORT`

## API and JSON Fields

1. Internal Python models: `snake_case`
2. Match external Jira API field names exactly when required.
3. Request/response schemas are explicit — no generic keys like `data` or `result`
   at the field level.