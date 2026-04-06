---
name: code-review
description: >-
  Thorough code review for the Platform healthcare SaaS backend. Checks for logical flaws, inconsistencies, missing pieces, security issues, and adherence to project patterns.
  USE FOR: code review, review code, review my changes, review this PR, review pull request, check my code, find bugs, find issues, audit code, review implementation, sanity check, look for problems, what did I miss, review diff, review branch.
  DO NOT USE FOR: writing new code from scratch, generating features, creating mockups (use mockup-guidelines skill).
---

# Code Review — Platform Backend

> **Purpose:** Perform a thorough, critical code review. Your job is to find problems, not to approve code. Be constructive but direct.

---

## 0. Before You Start — Clarifying Questions

**Always ask clarifying questions before diving in.** At minimum, confirm:

1. **What is the scope?** A specific file, a module, recent changes, or the full codebase?
2. **What changed and why?** Feature, bugfix, refactor, or migration?
3. **Which module/app?** (ValueIQ, RebateIQ, AccessIQ, or platform-level?)
4. **Any areas of concern?** Performance, security, correctness, or general review?

If the user says "review my changes," use `get_changed_files` or ask which files to look at. Do not review blindly.

> **Recent-change reviews:** Always use `git log` and `git diff` to identify actual commits and diffs. Never guess what changed from current code alone.

---

## 1. Review Process

### Step 1: Understand the Context

- Read the files being reviewed **in full** — do not skim.
- Identify which module the code belongs to and check related files (controller, crud_service, models, schema, constants).
- Look at how similar features are implemented elsewhere in the codebase. Search for patterns before flagging something as wrong.

### Step 2: Check Each Category

Work through **every category** in Section 2 below. Do not skip categories even if the change seems small — small changes can have large ripple effects.

### Step 3: Report Findings

Organize findings by severity:

| Severity     | Meaning                                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------------------------- |
| **CRITICAL** | Will cause bugs, data corruption, security vulnerabilities, or production failures. Must fix.              |
| **MAJOR**    | Significant logic issues, missing error handling, pattern violations that will cause problems. Should fix. |
| **MINOR**    | Style inconsistencies, minor improvements, redundant code. Nice to fix.                                    |
| **NOTE**     | Observations, suggestions, or questions for the author. Not necessarily wrong.                             |

For each finding, provide:

- **File and line** (with link)
- **What's wrong** (be specific)
- **Why it matters**
- **Suggested fix** (show the corrected code when possible)

---

## 2. Review Categories

### 2.1 Logical Correctness

- Does the code do what it claims to do?
- Are there off-by-one errors, wrong comparisons, or inverted conditions?
- Are edge cases handled? (empty lists, None values, zero, negative numbers)
- Are return values used correctly by callers?
- Do loops terminate correctly? Are break/continue used properly?
- Are there race conditions in multi-step operations?

### 2.2 Project Pattern Adherence

Check against the established patterns in `copilot-instructions.md`:

- **Request context:** Is `get_context_vars()` / `get_current_user()` / `get_current_user_organization_id()` called directly, or is context incorrectly passed as function parameters?
- **Module structure:** Does the code follow `controller.py` / `crud_service.py` / `models.py` / `schema.py` / `constants.py` split?
- **DB access isolation:** Are all database queries (`db.query`, `db.add`, `db.delete`, `get_context_vars()` for DB session) confined to `crud_service.py`? Service files (`service.py`, `*_service.py`) must NOT access the DB directly — they should call CRUD functions. **Exception:** `get_context_vars()` is allowed in service files that manage transaction boundaries (e.g., background task wrappers with `db.begin_nested()`).
- **Route decorators:** Correct order? `dependencies=[Depends(authenticated)]` on route, `@version(1)`, `@has_permissions(BuiltInPermissions.X.value)` (is `.value` used?), then optionally `@is_app([PlatformApps.X])`, then `@limiter.limit(...)`. Are all decorators in this order?
- **Authentication import:** Is `authenticated` imported from `common.authentication` (NOT `common.security`)?
- **Schemas:** Do they inherit from `BaseSchema` (NOT `pydantic.BaseModel`)?
- **Models:** Do tenant models have `__table_args__ = {"schema": None}`?
- **Transaction pattern:** Is `flush()` used before `commit()`? Is `in_transaction` parameter respected?
- **Background task atomicity:** Do background tasks that write multiple related records (e.g., data + status) wrap them in `db.begin_nested()`? The SUCCESS status must use `in_transaction=True` so it commits with the data. On failure, `db.rollback()` undoes everything, then ERROR status is recorded in a separate commit. Ask: "if any step after commit fails, is the DB consistent?" Reference: `app/vbc/health_plan/health_plan_bulk_upload_service.py`.
- **Nested savepoint trap (SQLAlchemy 2.0):** If a `begin_nested()` block calls a function that itself does `begin_nested()` + `session.commit()`, the inner commit releases ALL savepoints and commits the outermost transaction. Verify: does every function called inside a `begin_nested()` either (a) accept `in_transaction` and skip its own commit when `True`, or (b) never call `session.commit()`? Also check that deferred side effects (AAD, email) are returned, not executed, when `in_transaction=True`.
- **Behavior over rules:** When reviewing code modeled on an existing reference, understand _why_ the reference does something before flagging it as a pattern violation. Trace the full behavior (transaction boundaries, commit/rollback paths, error flows). A structural rule like "no DB access in service files" does not override the need for transaction control.
- **Enum comparisons:** Since `BaseSchema` has `use_enum_values=True`, are enum fields compared with `.value` strings, not enum instances?
- **Three-user authorization:** For routes that handle participant-scoped data, does the controller branch on caller type (`current_user.is_admin_user`, `is_customer_user()`, payer ownership check)? Is payer ownership validated by comparing `participant.organization_id` against `get_current_user_organization_id()`? Does Admin bypass ownership checks? Are manufacturer-only routes blocked for payer users (and vice versa)?
- **FastAPI route ordering:** Are literal routes (e.g., `/gaps`, `/detect`) registered **before** parameterized routes (e.g., `/{request_id}`) on the same HTTP method? Parameterized routes act as catch-alls and cause 422 errors if they shadow literal ones.
- **Imports at module top:** Are all imports at the top of the file? Inline/lazy imports inside functions make dependencies hard to trace. Only acceptable for genuine circular-import issues (and must be documented).
- **Azure SQL case-insensitivity:** Are there unnecessary `func.lower()` calls in SQLAlchemy string filters? Azure SQL default collation is case-insensitive — `func.lower()` is harmless but redundant.
- **Header/body tenant_app_id validation:** For POST/PATCH routes with `tenant_app_id` in the request body, is `body.tenant_app_id == x_tenant_app_id` validated (raise 400 on mismatch)? GET routes should filter by `x_tenant_app_id` from the header.
- **Module-level mutable globals:** Are there module-level dicts/lists used for cross-request state? With multi-worker deployment, each uvicorn worker has its own memory — module-level mutable state won't be shared. Use DB or `TenantSchemaMap` pattern instead.

### 2.3 Multi-Tenancy

- Are new tables tenant-scoped or platform-scoped? Is the choice correct?
- Do tenant models have `__table_args__ = {"schema": None}`?
- If a new tenant table was added, was `app/tenant/tables.py` updated too?
- If a migration adds a tenant table, does it use `{schema_name}` placeholder?
- Are there any cross-tenant data leaks? (queries without proper schema isolation)

### 2.4 Migrations

- Is the migration file named correctly? (`YYYY_N_SEQ_REV.sql`)
- Is the sequence number correct? (check `_migrations/` for the highest current SEQ)
- Does it use `PLATFORM.TABLE_NAME` for platform tables?
- Does it use `{schema_name}` for tenant tables?
- Is `app/tenant/tables.py` in sync with the migration?
- Are there `IF NOT EXISTS` guards where appropriate?
- New permissions: inserted into `ROLE_PERMISSION` with correct role IDs?
- **FK ordering:** When creating multiple tables with FK dependencies, are parent tables created before child tables? Does the same order hold in `tables.py`?

### 2.5 Missing Pieces

This is where most bugs hide. Actively look for things that **should exist but don't:**

- Missing error handling — what if the DB query returns None?
- Missing validation — are inputs validated before use?
- Missing permissions — should this endpoint have `@has_permissions`?
- Missing audit logging — should this state-changing action be logged? If audit logging is present, does it use `create_audit_log()` from `audit_log.audit_service` with enum-backed entity names and actions from `audit_log.constants`?
- Missing AI feature gating — do AI-related endpoints check `is_ai_enabled_for_app()` before proceeding? AI features must be gated per tenant app config.
- Missing `@is_app` decorator — should this route be restricted to a specific app?
- Missing rate limiting on sensitive endpoints?
- Incorrect rate limit values? (guideline: `"60/minute"` for reads, `"30/minute"` default, `"10/minute"` for expensive/bulk, `"120/minute"` for lightweight)
- Missing `db.commit()` or `db.rollback()` in error paths?
- Missing atomicity — are multiple writes in a background task wrapped in `db.begin_nested()`? If the data commits but the status record doesn't, the system is inconsistent.
- Nested savepoint violations — does a `begin_nested()` block call any function that itself calls `session.commit()` without an `in_transaction` guard? (SQLAlchemy 2.0 trap: inner commit releases all savepoints)
- Boolean parsing in file uploads — are boolean-like values from spreadsheet columns parsed with a positive allowlist (`in ("yes", "true", "1", "y")`) so unknowns default to False? Negative allowlists cause blanks/typos to default to True.
- Variable scoping in try/except — are variables referenced in `except` blocks initialized before the `try` statement? Common with `invites`, `errors`, or similar lists in background tasks.
- Exception differentiation in background tasks — do background tasks catch `HTTPException` separately from generic `Exception`? Business validation errors should preserve structured detail; generic exceptions should store a safe message, never `str(e)`.
- String truncation before DB insert — are user-generated or error-derived strings truncated to the VARCHAR column limit before inserting? Without truncation, joined errors or raw exception strings can exceed the limit and cause a secondary DB failure.
- File upload row limits — when a file upload path constructs an API body schema with `max_length` on a list field, does the upload service check row count before constructing the body? Otherwise users get opaque Pydantic validation errors.
- New enum values not added to `constants.py`?
- New permissions not added to `BuiltInPermissions` in `app/role/constants.py`?
- Schema fields that exist in the model but not in the response schema (or vice versa)?
- Missing tests — does this change have corresponding test coverage in `tests/`? Are new functions, edge cases, and error paths tested?

### 2.6 Inconsistencies

Look for things that contradict each other or break expectations:

- Model field names vs. schema field names — do they match?
- Column types in migration vs. SQLAlchemy model types — do they align?
- Business logic that contradicts what the route name/docstring promises
- Inconsistent naming (e.g., `user_id` in one place, `userId` in another)
- Inconsistent error response formats compared to other endpoints
- Constants defined but not used, or magic strings instead of constants

### 2.7 Security

- Are there any SQL injection risks? (raw SQL without parameterization)
- Is user input sanitized before use in queries or file paths?
- Are permissions checks present on all endpoints that need them?
- Is sensitive data (passwords, tokens, PII) being logged?
- Are there any IDOR (Insecure Direct Object Reference) risks — can a user access another user's/org's data by manipulating IDs?
- Is the `current_user` or `org_id` used to scope data access?
- **Participant removal flows:** If removing/disabling a VBC CE payer or AccessIQ participant, are the corresponding `OrganizationTenantAppAccess` records also deleted? (2 per payer — ADMIN + APP_ADMIN groups). Stale records grant tenant access via middleware. Reference: delegate cleanup in `app/vbc/health_plan/invite_delegate_service.py`

### 2.8 Error Handling

- Are exceptions caught at the right level?
- Are error messages helpful without leaking internal details?
- Are HTTP status codes correct? (400 for client errors, 404 for not found, 403 for forbidden, 500 should be rare)
- Are there bare `except:` or `except Exception:` blocks that swallow errors silently?
- Do error paths clean up resources (rollback transactions)?

### 2.9 Performance

- Are there N+1 query patterns? (querying in a loop)
- Are there missing DB indexes for frequently-queried columns?
- Are large result sets loaded entirely into memory when they could be paginated?
- Are there unnecessary `db.commit()` calls inside loops?

### 2.10 Code Quality

- Dead code, unused imports, commented-out blocks
- Functions that are too long (>50 lines) or do too many things
- Duplicated logic that should be extracted
- Naming: are variables and functions clearly named for their purpose?
- Type hints: are they present and correct?

---

## 3. Don'ts

- **Don't rubber-stamp.** If you find nothing wrong, say so explicitly, but also explain what you checked. An empty review is not a good review.
- **Don't guess.** If you're unsure whether something is a bug, read the related code to confirm. Use search tools to find how similar code works elsewhere.
- **Don't only find problems.** If something is done well or cleverly, call it out briefly.
- **Don't flag things that are documented exceptions.** For instance, `allow_origins=["*"]` in CORS is intentional (enforced at infra level). JWT `verify_signature: False` is by design. Check `copilot-instructions.md` before flagging.
- **Don't suggest changes you can't justify.** Every finding needs a "why it matters."

---

## 4. Review Summary Template

End every review with a summary:

```
## Review Summary

**Scope:** [what was reviewed]
**Verdict:** [CRITICAL issues found / Looks good with minor items / Clean]

### Findings
| # | Severity | File | Issue |
|---|----------|------|-------|
| 1 | CRITICAL | file.py:L42 | Description |
| 2 | MAJOR    | file.py:L88 | Description |
| ...| ... | ... | ... |

### What Looks Good
- [Positive observations]

### Recommendations
- [Optional broader suggestions]
```
