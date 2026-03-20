---
applyTo: "**"
---

# Global Rules
# Version: 1.0.0

## General

1. Always follow programming best practices.
2. Write code with production in mind at all times.
3. Think in layers — separate concerns: data, business logic, presentation.
4. Always validate inputs. Never trust data from the user or an external API.
5. Apply a ceiling to anything that can grow infinitely:
   - Log files older than `cfg_log_retention_days` days must be rotated or deleted.
   - API rate limiting enforced via config (`cfg_rate_limit_per_minute`).
6. Code must work in both `dev` and `prod` environments.
7. Use GitHub for version control. Write meaningful commit messages.

## Production Considerations

1. **Error Handling**: Handle all errors gracefully. No unhandled exceptions that crash the app.
2. **Security**: No credentials, URLs, or secrets in code. Use environment variables if possible.
3. **Performance**: No infinite loops. Handle timeouts. Avoid blocking calls in async context.
4. **Monitoring**: Log all important operations, errors, and durations.
5. **Reliability**: Handle edge cases, network failures, missing config, unexpected inputs.
6. **Scalability**: Adding a new feature should not require rewriting existing modules.
7. **Documentation**: Every module has a docstring explaining its purpose.

## Logging

1. Use Python's `logging` module only. Never use `print()`.
2. Log level controlled by `BSS_LOG_LEVEL` environment variable. Default: `INFO`.
3. Log format must include: timestamp, log level, module name, message.
4. Use structured key=value pairs in messages: `staff_id=****1234 project_key=EQDTRG`
5. Required markers for all endpoint and external API operations:
   - `logger.info("BEGIN:<operation> <context key=value>")`
   - `logger.info("END:<operation> duration_ms=<ms>")`
   - `logger.error("ERROR:<operation> error=<message> duration_ms=<ms>")`
6. `END:` and `ERROR:` must be inside a `finally` block — they must always execute.
7. Always measure and log `duration_ms` for any external API call.
8. NEVER log: passwords, API tokens, Authorization headers, secrets, full payloads.
9. Mask staff IDs in logs — show last 4 digits only: `****1234`
10. Log retention: rotate or delete logs older than `cfg_log_retention_days` (from config).

## Error Handling

1. Always use `try/except` for any I/O, network call, or file operation.
2. Catch specific exceptions first, then `Exception` as fallback.
3. Never expose internal error details (stack traces, system paths) in API responses.
4. Return a meaningful, user-safe error message in the standard response envelope.
5. Use `finally` blocks for cleanup and guaranteed logging.

## Validation Pipeline (non-negotiable order)
```
Receive Input → Pydantic Validation → Business Validation →
Transform → Enrich → Call External API → Build Response
```

Never skip or reorder these stages.

## Config Files

1. Use YAML for all application config.
2. YAML data files live in `configs/dev/` or `configs/prod/` at the project root.
3. The Python module that reads YAML lives in `backend/config/config_loader.py`.
   These are two different things — do not confuse them.
4. Every config file must have a `version` field. Increment when the schema changes.
5. Environment is selected by `APP_ENV` env var — never hardcoded.
6. Business rules, field mappings, dropdown values → YAML config.
7. Credentials, secrets, URLs → `.env` file only. Never in YAML.

## Testing

1. Test data goes in `tests/test_data/`. Use realistic but fake data.
2. Always test: happy path, missing fields, invalid formats, API failure/timeout, edge cases.
3. Code must be testable without modifying `server3.py`.
4. Use dependency injection so external clients (Jira) can be mocked.