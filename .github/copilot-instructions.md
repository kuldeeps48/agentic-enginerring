# Platform Backend - AI Coding Agent Instructions

## System Overview

Multi-tenant healthcare SaaS backend supporting **ValueIQ** (VBC), **RebateIQ** (Rebate), and **AccessIQ** (Intelligent Workspace). FastAPI + SQL Server + Azure deployment via GitLab CI/CD + Terraform.

**Architecture note:** The app uses synchronous `def` route handlers with synchronous SQLAlchemy and pyodbc. FastAPI handles this correctly by running sync handlers in a threadpool. CORS `allow_origins=["*"]` in `main.py` is intentional — CORS restrictions are enforced at the Azure Web App infrastructure level, not in application code.

**Multi-worker deployment:** The app runs with multiple uvicorn workers via `WEB_CONCURRENCY` env var. Each worker is a separate OS process with its own memory. Do NOT store mutable cross-request state in module-level globals — it won't be shared across workers. Tenant schema mappings are handled by `TenantSchemaMap` (`app/common/tenant_schema_map.py`) which auto-refreshes from DB on cache miss.

> **Golden Rule:** Before creating anything new, search existing code for similar patterns. Use `grep` or semantic search to find how similar features are implemented.

## Project Structure

- `app/` - Application code (modules, services, models)
- `_migrations/` - SQL migration files
- `_terraform/` - Infrastructure as code
- `_app_configurations/` - Per-app feature configs
- `documents/` - Project documentation, organized by feature area (e.g., `documents/rebate/`, `documents/accessiq-launch-planning/`)
  - **Business/design docs:** `*-business-document.md` (e.g., `rebate-business-document.md`)
  - **Technical docs:** `*-technical-document.md` (e.g., `rebate-technical-document.md`)
  - `*` = feature/topic name matching the folder
  - A folder may contain multiple documents with different topic prefixes
- `mockups/` - HTML mockups organized by feature area
- `.github/skills/` - Agent Skills (auto-discovered from `SKILL.md` frontmatter; no need to register here)
- `tests/` - Pytest test suite
- `test/` - Scratch/sample files (NOT part of project)

## Skills Reference

For task-specific work, load the appropriate skill for full procedures, templates, and checklists:

| Task                       | Skill                                                                                    | What it covers                                                                                |
| -------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Writing API endpoints      | [api-patterns](.github/skills/api-patterns/SKILL.md)                                     | Response shapes, status codes, filtering, pagination, error handling, audit logging templates |
| Creating migrations        | [migrations](.github/skills/migrations/SKILL.md)                                         | Naming, DDL conventions, permission inserts, tables.py sync procedure                         |
| Scaffolding a new module   | [new-module](.github/skills/new-module/SKILL.md)                                         | Directory structure, file templates, wiring, permissions                                      |
| Reviewing code             | [code-review](.github/skills/code-review/SKILL.md)                                       | Review categories, checklists, severity levels, output format                                 |
| Creating HTML mockups      | [mockup-guidelines](.github/skills/mockup-guidelines/SKILL.md)                           | Design tokens, layout anatomy, component patterns, sample data rules                          |
| Frontend task generation   | [frontend-repo-task-generator](.github/skills/frontend-repo-task-generator/SKILL.md)     | Frontend-specific task files with React/TS types, Redux, routing, permissions baked in        |
| AI service task generation | [ai-service-repo-task-generator](.github/skills/ai-service-repo-task-generator/SKILL.md) | AI service task files with pipelines, prompts, pgvector, event queue, PII masking baked in    |
| Updating knowledge         | [update-knowledge](.github/skills/update-knowledge/SKILL.md)                             | Persist session learnings to memory, skills, commands, and instruction files                  |

The sections below provide the **always-on essentials**. Skills contain the deep-dive details.

## Critical Patterns

### Request Context (NEVER pass as parameters)

```python
from common.security import get_context_vars, get_current_user, get_current_user_organization_id, get_current_tenant_app_id

tenant_id, db = get_context_vars()  # Always available after middleware
current_user = get_current_user()   # Set by AuthMiddleware from Bearer token
org_id = get_current_user_organization_id()  # From X-Organization-Id header
tenant_app_id = get_current_tenant_app_id()  # From X-Tenant-App-Id header (used by @is_app decorator)
```

Context set by `ContextVarMiddleware` from headers: `X-Tenant-Id`, `X-Organization-Id`, `X-Tenant-App-Id`.

### Module Structure (Mandatory)

```
app/<module>/
  controller.py      # Routes: @version(1), @authenticated, @has_permissions
  crud_service.py    # Database layer (repository pattern) — ALL db.query/db.add/db.delete calls go here
  models.py          # SQLAlchemy models with __table_args__ = {"schema": None}
  schema.py          # Pydantic schemas inheriting BaseSchema
  constants.py       # Enums
  service.py         # Optional: cross-module business logic (NO direct DB access — call crud_service)
```

**Reference modules:** `app/chat/`, `app/vbc/`, `app/organization/`

#### Sub-Module Structure

Larger modules can have nested sub-modules, each with the full module structure. Sub-modules register their routers on the **parent module's router** (not in `main.py`):

```
app/intelligent_workspace/
  controller.py            # Parent router + include_router() for sub-routers
  models.py
  launch_planning/         # Sub-module
    controller.py          # launch_planning_router
    crud_service.py
    models.py
    schema.py
    constants.py
  payer_workflow/           # Sub-module
    controller.py          # payer_workflow_router
    ...
```

```python
# In parent controller.py:
from intelligent_workspace.launch_planning.controller import launch_planning_router
intelligent_workspace_router.include_router(launch_planning_router)
```

Resulting URL paths nest automatically: `/v1/intelligent-workspace/launch-planning/...`

**Reference:** `app/intelligent_workspace/controller.py`

### Multi-Tenancy (CRITICAL)

- **Platform tables:** `PLATFORM` schema (users, roles, organizations, audit logs)
- **Tenant tables:** Dynamic schema per tenant (e.g., `TENANT_ABC123`)
- **All tenant models MUST have:** `__table_args__ = {"schema": None}` - schema is set dynamically at runtime
- **Tenant schema map:** `tenant_schema_map` in `app/common/tenant_schema_map.py` maps `tenant_id → schema_name`. Bulk-loaded at startup, auto-refreshes from DB on cache miss. Use `tenant_schema_map.put()` after creating a new tenant for immediate local visibility.
- **Cross-tenant validation:** `CurrentUserMiddleware._user_can_access_tenant()` validates `X-Tenant-Id` against the authenticated user's org memberships. Three paths: Admin (full access), Manufacturer (org owns tenant), Payer/Delegate (active `OrganizationTenantAppAccess` record). Returns 403 on mismatch. Only triggered when `X-Tenant-Id` header is present — platform-scoped apps (PRM, TPA) are unaffected.
- **Tenant-scoped vs platform-scoped apps:** VBC and AccessIQ are tenant-scoped (per-tenant schemas, require `X-Tenant-Id` + `OrganizationTenantAppAccess`). PRM and TPA are platform-scoped (`PLATFORM` schema only, no `X-Tenant-Id`).

```python
class VBCParticipant(Base):
    __tablename__ = "VBC_PARTICIPANT"
    __table_args__ = {"schema": None}  # Schema injected by middleware
```

**Hybrid properties:** Use `@hybrid_property` on models for computed fields derived from relationships (e.g., deriving a current stage from a list of milestones). These are useful for lightweight list-view schemas without extra queries:

```python
from sqlalchemy.ext.hybrid import hybrid_property

class MyModel(Base):
    ...
    items = relationship("ChildModel", lazy="select")

    @hybrid_property
    def computed_field(self) -> Optional[str]:
        # Requires items to be loaded (use joinedload in query)
        return some_derivation(self.items)
```

### Migrations (CRITICAL - Read Before Any Schema Change)

> **Full procedures:** Load the [migrations skill](.github/skills/migrations/SKILL.md) for naming conventions, DDL rules, permission insert patterns, and the tables.py sync procedure.

Key rules (always apply):

1. **Never edit applied migrations** — always create new file
2. **Naming:** `YYYY_N_SEQ_REV.sql` — always verify next SEQ from `_migrations/`
3. **Platform tables:** `PLATFORM.TABLE_NAME`. **Tenant tables:** `{schema_name}` placeholder
4. **ALWAYS update `app/tenant/tables.py`** when adding/modifying tenant tables — if out of sync, new tenants get different schema than existing ones

### Route Decorators (Order Matters)

> **Full patterns:** Load the [api-patterns skill](.github/skills/api-patterns/SKILL.md) for complete decorator stack, response shapes, error handling, and filtering patterns.

```python
from fastapi import Depends, Request
from fastapi_versioning import version
from common.authentication import authenticated  # NOT from common.security!
from common.authorization import has_permissions
from common.decorators.is_app import is_app  # Optional
from role.constants import BuiltInPermissions
from tenant.constants import PlatformApps

@router.post("/endpoint", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.VBC_READ.value)  # Use .value! OR logic for multiple
@is_app([PlatformApps.VBC])  # Optional: restrict to specific app
@limiter.limit("30/minute")  # Optional: rate limiting
def handler(request: Request):
    current_user = get_current_user()
    _, db = get_context_vars()
```

### Three-User Authorization Pattern

All apps support three caller types: **Admin** (platform operator), **Manufacturer** (customer), and **Payer** (participant). Routes must branch on caller type for ownership validation and data scoping.

- `current_user.is_admin_user` — full access, no ownership check
- `is_customer_user(current_user, tenant, org_id)` — from `common.utils`, manufacturer-scoped access
- Payer — validate `participant.organization_id` against `get_current_user_organization_id()`

> **Full pattern with code:** Load the [api-patterns skill](.github/skills/api-patterns/SKILL.md) § "Three-User Authorization".

### Header-Based Tenant App Scoping

Many routes (especially in AccessIQ) use `x_tenant_app_id` from the request header to scope data:

```python
from typing import Annotated
from fastapi import Header

def handler(
    request: Request,
    x_tenant_id: Annotated[str, Header()],
    x_tenant_app_id: Annotated[int, Header()],
):
```

For POST/PATCH routes with a body containing `tenant_app_id`, validate that `body.tenant_app_id == x_tenant_app_id` (raise 400 on mismatch). GET routes filter by `x_tenant_app_id` from the header.

### Transaction Pattern

```python
def create_entity(data, in_transaction=False):
    _, db = get_context_vars()
    db.add(entity)
    db.flush()  # Always flush to get IDs
    if not in_transaction:
        db.commit()  # Only commit if not in a larger transaction
```

**Atomicity rule for background tasks:** When a background task performs multiple DB operations that must succeed or fail together (e.g., bulk invites + status record), wrap them in a single `db.begin_nested()` / `db.commit()` savepoint. The SUCCESS status record must use `in_transaction=True` so it commits with the data. On failure, `db.rollback()` undoes everything, then the ERROR status is recorded in a separate commit. Reference: `app/vbc/health_plan/health_plan_bulk_upload_service.py`.

**SQLAlchemy 2.0 savepoint trap:** `session.commit()` is a session-level operation that releases ALL savepoints and commits the outermost transaction — it does NOT just release the innermost savepoint. If function A opens a savepoint with `db.begin_nested()` and calls function B which also has `db.begin_nested()` + `db.commit()`, B's commit ends A's savepoint too. When reusing an orchestrator function (one that manages its own transaction) inside another savepoint, it MUST accept `in_transaction` to suppress its `begin_nested`/`commit`/`rollback` and return deferred side effects to the caller instead of executing them.

**Exception:** `get_context_vars()` in service files is allowed when the service manages a transaction boundary (e.g., background task wrappers). The "no DB access in service files" rule applies to _business logic queries_, not transaction control.

### Pydantic Schemas

```python
from common.base_schema import BaseSchema  # NOT pydantic.BaseModel!

class MySchema(BaseSchema):  # Pre-configured: from_attributes, strip_whitespace, use_enum_values
    field: str
```

**BaseSchema Quirks (IMPORTANT):**

- **`use_enum_values=True`**: Enums are converted to their `.value` string on parsing. Compare with `MyEnum.OPTION_A.value`, not `MyEnum.OPTION_A`.
- **`from_attributes=True`**: Create schemas from SQLAlchemy models via `MySchema.model_validate(db_object)`
- Use `Decimal` for money/percentage fields, not `float`. Use `datetime` for timestamps, not `str`.

### Audit Logging

Use `create_audit_log()` from `audit_log.audit_service` for all state-changing actions. Entity names and actions are enums in `audit_log.constants`.

> **Full template with code:** Load the [api-patterns skill](.github/skills/api-patterns/SKILL.md) § "Audit Logging".

## Development Commands

```bash
# Setup
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt && pre-commit install

# macOS Apple Silicon ODBC fix
pip install --no-binary :all: pyodbc

# Run server (single worker)
cd app && uvicorn main:app --reload  # Docs: http://localhost:8000/v1/docs

# Run server (multi-worker, mimics production)
cd app && WEB_CONCURRENCY=4 DB_POOL_SIZE=5 DB_MAX_OVERFLOW=10 uvicorn main:app --host 0.0.0.0 --port 8000

# Pre-commit (required before commit)
pre-commit run --all-files  # autoflake, black, isort, pip-audit, pytest

# Dependency vulnerability scan (also runs in pre-commit and CI)
pip-audit --strict --desc
```

## CI/CD Branch Mapping

`main`→DEV, `qa`→QA, `sbx`→SBX, `beta`→BETA, `production`→PROD

## Environment & Integrations

Settings via Pydantic from `.env` (see `app/common/config.py`):

- **DB Pool:** `DB_POOL_SIZE` (default 30), `DB_MAX_OVERFLOW` (default 100), `DB_POOL_RECYCLE` (default 3600). Tune per environment: `pool_size × WEB_CONCURRENCY` must not exceed Azure SQL tier concurrent worker limit (S0=60, P1=200).
- **Auth:** Azure AD B2C + Managed Identity for SQL Server
- **Storage:** Azure Blob Storage, SharePoint (document management)
- **Monitoring:** Application Insights, PostHog analytics (frontend-only — PostHog events are tracked client-side; the backend does NOT call PostHog APIs or log events to PostHog)
- **Email:** SendGrid
- **Reports:** Power BI embedded

## AI Service Integration

AI features are per-app configurable. Always check before implementing:

```python
from ai_service.constants import is_ai_enabled_for_app

if not is_ai_enabled_for_app(tenant_app.config):
    raise HTTPException(status_code=403, detail="AI not enabled")
```

**Key patterns:**

- **AI Event Queue:** Async processing via `AIEventQueue` model - insert events with `insert_ai_event_in_queue()`
- **AI Chat:** Create sessions via `create_new_ai_chat()`, get responses via `get_ai_chat_message_response()`
- **Feature names:** Use `AIEventQueueFeatureName` enum (`GOLDEN_POLICY`, `REVIEW_HEALTH_PLAN_POLICY`, etc.)
- **External service:** AI logic lives in separate microservice at `AI_MICROSERVICE_URL`

Reference: `app/ai_service/service.py`, `app/ai_service/constants.py`

## Rule Engine (Claims Evaluation)

Dynamic business rule evaluation for VBC claims using `rule_engine` library with custom builtins.

```python
from common.services.rule_engine import execute_rule, validate_rule

# Evaluate a rule against claim data
result = execute_rule("CLAIM.DIAGNOSIS_CODE == 'D57.1' and within_days(CLAIM.SERVICE_DATE, 30)", claim_data)

# Validate rule syntax before saving
validate_rule(rule_string)
```

**Custom builtins available:** `within_days()`, `within_months()`, `consecutive_within_days()`, `any_in()`, `any_code_in()`, `count_in()`, `extract()`, `convert()`, `quarter_date()`, `quarter_end_date()`, `anchor_claim()`, `calculate_rebate_fees()`

**Key files:**

- `app/common/services/rule_engine.py` - Core execution context and custom builtins
- `app/common/services/rule_engine_helpers.py` - Custom function implementations
- `app/vbc/rule_engine/form_to_rule_service.py` - UI form → rule formula conversion

## Testing

Tests in `tests/` directory. Run with pytest from project root.

> **Note:** The `test/` directory at repo root contains scratch/sample files and is NOT part of the project.

```bash
# Run all tests
pytest tests/

# Run specific test file
pytest tests/platform_patient_id/test_platform_patient_id_generation_service.py -v
```

**Test patterns:**

- Use fixtures from `tests/conftest.py` (`mock_db_session`, `mock_context_vars`)
- Mock `get_context_vars()` and CRUD functions, don't hit real DB
- Name test files `test_<module>_<feature>.py`

```python
@pytest.fixture
def mock_context_vars(mock_db_session):
    return ("test_tenant", mock_db_session)

# In tests, patch context:
with patch("module.get_context_vars", return_value=mock_context_vars):
    result = function_under_test()
```

## Key Files

| File                                        | Purpose                                                                                                          |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `app/main.py`                               | Middleware stack, router registration, startup/shutdown hooks                                                    |
| `app/tenant/tables.py`                      | **Critical:** All tenant table DDL (must sync with migrations)                                                   |
| `app/common/tenant_schema_map.py`           | Tenant ID → schema name cache (DB-backed, auto-refresh on miss)                                                  |
| `app/common/config.py`                      | All settings including DB pool tuning (`DB_POOL_SIZE`, etc.)                                                     |
| `app/common/security.py`                    | Context variable accessors (`get_context_vars`, `get_current_user`)                                              |
| `app/common/authentication.py`              | `authenticated` dependency for routes                                                                            |
| `app/common/middlewares/auth_middleware.py` | Cross-tenant validation (`_user_can_access_tenant`), user context setup                                          |
| `app/common/authorization.py`               | `has_permissions` decorator for RBAC                                                                             |
| `app/common/base_schema.py`                 | Base Pydantic schema with project defaults                                                                       |
| `app/role/constants.py`                     | `BuiltInPermissions` and `BuiltInSystemRoles` enums                                                              |
| `app/auth/login/registration_service.py`    | User registration flow: signup, org assignment, delegate/TPA/admin/user branching, multi-org pending invitations |
| `_migrations/`                              | SQL migrations (check folder for latest)                                                                         |
| `_app_configurations/`                      | Per-app feature configs (AI enablement, etc.)                                                                    |

## Security Notes

### JWT Token Verification (`app/common/security.py`)

JWT signature verification is intentionally disabled (`verify_signature: False`) in `get_decoded_token()`. This is **by design**, not an oversight:

- The access token is issued for **Microsoft Graph** (scope: `graph.microsoft.com/.default`), not for this app. Graph tokens are signed with Microsoft-internal keys not available at the tenant JWKS endpoint.
- The same token is also used for **SharePoint API calls** via the user's delegated permissions, so switching to an app-specific scope would require a separate login flow for SharePoint.

**Mitigations in place:**

- Algorithm restricted to RS256 only (prevents algorithm confusion attacks)
- Audience validated against `AD_CLIENT_ID`
- Issuer validated against `AD_TENANT_ID`
- Token expiry (`exp`) and issued-at (`iat`) are validated by PyJWT
- Required claims (`email` or `upn`, `exp`, `iat`) are enforced
- User is looked up in DB after decode — forged tokens with non-existent users fail at the DB lookup stage

**To enable full signature verification:** Change MSAL scopes from `https://graph.microsoft.com/.default` to an app-specific scope (e.g., `api://{client_id}/.default`) so tokens are issued for this app. This would also require a separate auth flow for SharePoint.

> **Do NOT** add signature verification without changing the MSAL scope — it will break authentication since Graph token signing keys are not publicly available.

## Common Gotchas

1. **App naming:** Code uses `VBC`/`REBATE`/`INTELLIGENT_WORKSPACE` internally → user-facing is ValueIQ/RebateIQ/AccessIQ. All apps support three user contexts: **Admin** (platform operator), **Manufacturer** (customer), and **Payer** (participant). Use `is_admin_user`, `is_customer_user()`, and payer ownership validation to branch controller logic
2. **Forgetting `tables.py`:** New tenant provisioning will fail silently
3. **Missing `__table_args__`:** Multi-tenancy breaks - queries go to wrong schema
4. **Passing context as params:** Use `get_context_vars()` - never thread context through functions
5. **Permissions `.value`:** Always use `BuiltInPermissions.PERM_NAME.value` in decorators
6. **AI features:** Check `enable_ai: true` in tenant app config before implementing
7. **Wrong import for `authenticated`:** Import from `common.authentication`, NOT `common.security`
8. **Module-level mutable globals:** With multi-worker deployment, each worker has its own memory. Do not use module-level dicts/lists for cross-request state — use DB or `TenantSchemaMap` pattern instead
9. **Azure SQL is case-insensitive:** Default collation is `SQL_Latin1_General_CP1_CI_AS` (CI = Case Insensitive). No need for `func.lower()` in SQLAlchemy filters for string comparisons — the DB handles it. Existing `func.lower()` calls are harmless but redundant
10. **FastAPI route ordering:** Parameterized routes like `/{id}` act as catch-alls and must be registered **after** literal routes on the same HTTP method. If `GET /{request_id}` is defined before `GET /gaps`, FastAPI tries to parse `"gaps"` as the integer `request_id` → 422 error. Always define literal paths (`/gaps`, `/gaps/detect`) before parameterized paths (`/{request_id}`, `/{request_id}/status`) on the same router
11. **Imports at module top:** Always place imports at the top of the file, never inside functions. Inline/lazy imports make dependencies hard to trace and violate Python conventions. Only use inline imports if there is a genuine circular-import issue (and document why)
12. **DB queries outside `crud_service.py`:** All database access (`db.query`, `db.add`, `db.delete`, `db.flush`, `db.commit`, `get_context_vars()` for DB session) must live in `crud_service.py`. Service files (`service.py`, `*_service.py`) must call CRUD functions — never import `get_context_vars` or use `db` directly
13. **Identifying recent changes:** When asked about recent changes, always use `git log` and `git diff` to identify actual commits and diffs — never guess from current code alone. When summarizing changes for other teams (e.g., frontend), only include what's NEW or CHANGED, not existing/unchanged API contracts
14. **Behavior over rules:** When implementing a feature modeled on an existing reference, understand _why_ the reference does something before applying coding rules that might contradict it. Trace the full behavior (transaction boundaries, commit/rollback paths, error flows) — don't just pattern-match against structural rules. A rule like "no DB access in service files" does not override the need for a service to manage a savepoint in a background task
15. **Background task atomicity:** Background tasks that write multiple related records must wrap them in `db.begin_nested()`. If invites commit but the status record doesn't (or vice versa), the system enters an inconsistent state. Always verify: "if any step after commit fails, is the DB consistent?"
16. **SQLAlchemy 2.0 `session.commit()` releases all savepoints:** Do NOT call a function that has its own `db.begin_nested()` + `db.commit()` from inside another `begin_nested()` block — the inner `commit()` will release ALL savepoints and commit the outermost transaction. When promoting an orchestrator function to work inside a caller's savepoint, add `in_transaction=False` parameter and: (a) skip `begin_nested`/`commit`/`rollback` when `True`, (b) return deferred side effects (AAD, email) instead of executing them — the caller runs them after its own commit. Reference: `app/vbc/invite_participant_service.py` `bulk_invite_payers_to_vbc()`
17. **Variable scoping in try/except:** Variables referenced in `except` blocks must be initialized before the `try` statement. If a variable is declared inside `try` and an exception occurs before that line, the `except` handler gets `UnboundLocalError`. Common with `invites`, `errors`, or similar lists used in background task error reporting
18. **Background task exception differentiation:** Background tasks calling helpers that raise `HTTPException` for validation must catch `HTTPException` separately from generic `Exception`. Preserve `HTTPException.detail` (structured validation data) in the error record. Generic exceptions should store a safe message like `"An unexpected error occurred while processing the file"` — never `str(e)` which can leak SQL errors, connection strings, or stack details. Reference: `app/vbc/payer_bulk_invite_upload_service.py`
19. **Truncate strings before DB insert:** When storing user-generated or error-derived strings into bounded VARCHAR columns, always truncate to the column limit before inserting (e.g., `error_message[:8000]`). Without truncation, joined validation errors or raw exception strings can exceed the limit, causing a secondary DB failure when recording the error — leaving no record of what went wrong. Reference: `app/common/crud_service.py` `add_payer_bulk_invite_upload_status()`
20. **Participant removal must clean up `OrganizationTenantAppAccess`:** When implementing participant removal for VBC CE payers or AccessIQ participants, you MUST also delete/disable their `OrganizationTenantAppAccess` records (2 per payer — ADMIN + APP_ADMIN groups). Otherwise removed participants retain tenant access via stale records. Only VBC delegates currently clean up properly (via `remove_tenant_app_access_from_organization()`). Reference: `app/vbc/health_plan/invite_delegate_service.py`
21. **`OrganizationTenantAppAccess.disabled` is dead code:** The `disabled` column exists and the middleware reads it, but no code path ever sets it to `True`. Access is revoked by hard-deleting records. If adding soft-disable, update both the revocation code AND verify middleware handles it

## New Feature Checklist

- [ ] Which app? (ValueIQ/RebateIQ/AccessIQ)
- [ ] Tenant-scoped or platform-scoped data?
- [ ] New permission? Add to migration AND `BuiltInPermissions` enum in `app/role/constants.py`
- [ ] New table? Create migration AND update `app/tenant/tables.py`
- [ ] AI integration? Verify `enable_ai` in app config

## Post-Implementation Self-Review (Mandatory)

After completing any implementation task (new feature, bugfix, migration, or refactor), **automatically perform a code review** of all files you created or modified — do not wait for the user to ask. Load the [code-review skill](.github/skills/code-review/SKILL.md) and report findings with the standard severity table. This catches issues in the same session while full context is available.
