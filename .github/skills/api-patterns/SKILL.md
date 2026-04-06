---
name: api-patterns
description: >-
  Canonical API endpoint patterns for the Platform healthcare SaaS backend. Covers response shapes, status codes, filtering, three-user authorization, audit logging, transactions, and request.state usage.
  USE FOR: add endpoint, new endpoint, new route, API design, response format, status code, pagination, filtering, query params, list endpoint, detail endpoint, create endpoint, update endpoint, delete endpoint, PATCH endpoint, three-user auth, audit log, tenant_app_id validation, request.state, is_app, transaction, error handling, HTTPException, how to return data, response shape, bulk endpoint, file upload endpoint.
  DO NOT USE FOR: scaffolding a brand-new module from scratch (use new-module skill), database migrations (use migrations skill), code review (use code-review skill), creating mockups (use mockup-guidelines skill).
---

# API Endpoint Patterns — Platform Backend

> **Purpose:** Canonical patterns for designing and implementing API endpoints. Use this skill when adding new routes to **existing** modules, or when the `new-module` skill's basic CRUD template isn't enough for real-world complexity.
>
> The `new-module` skill covers scaffolding. **This skill covers design decisions** — response shapes, status codes, filtering, authorization branching, audit logging, and transaction management.

---

## 0. Before You Start

1. **Search existing controllers** for similar endpoints before writing new ones. Run `grep` or semantic search to find how comparable features are implemented.
2. **Identify the caller types** — does this endpoint serve Admin, Manufacturer, and/or Payer users? This drives authorization and data scoping.
3. **Identify whether `@is_app` is needed** — if the route is app-specific (ValueIQ/RebateIQ/AccessIQ), use `@is_app`. If platform-wide, omit it.

---

## 1. Route Decorator Stack

**Canonical order (never deviate):**

```python
@router.post("/path", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.PERMISSION.value)   # Always use .value
@is_app([PlatformApps.APP_NAME])                        # Optional — only for app-specific routes
@limiter.limit("30/minute")                              # Always last
def handler(request: Request, ...):
```

### Rules

| Rule                                    | Detail                                                                                                                                               |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dependencies=[Depends(authenticated)]` | Always on the **route decorator**, not as a function param                                                                                           |
| `@version(1)`                           | Always second                                                                                                                                        |
| `@has_permissions(...)`                 | OR logic for multiple permissions. Always use `.value` on the enum                                                                                   |
| `@is_app([...])`                        | Only for app-scoped routes. Accepts a list — can include multiple apps                                                                               |
| `@limiter.limit(...)`                   | Always last. Choose rate based on cost: `"30/minute"` (default), `"60/minute"` (reads), `"10/minute"` (expensive/bulk), `"120/minute"` (lightweight) |
| `def` not `async def`                   | Always sync. FastAPI runs sync handlers in a threadpool                                                                                              |
| `request: Request`                      | Always the first handler parameter (required by rate limiter and audit logging)                                                                      |

### Multiple Permissions (OR Logic)

```python
@has_permissions(
    BuiltInPermissions.VIEW_ALL_ORGANIZATIONS.value,
    BuiltInPermissions.VBC_INVITE_PAYER.value,
    BuiltInPermissions.INTELLIGENT_WORKSPACE_PAYER_INVITE.value,
)
```

Any **one** matching permission grants access.

### Multiple Apps

```python
@is_app([PlatformApps.INTELLIGENT_WORKSPACE, PlatformApps.VBC])
```

### DELETE Status Code

Set `status_code` in the route decorator for DELETE endpoints (not in the handler body):

```python
@router.delete(
    "/{item_id}",
    dependencies=[Depends(authenticated)],
    status_code=HTTPStatus.NO_CONTENT,
)
```

---

## 2. Request Parameters

### Header Parameters

Declare tenant headers as typed `Annotated` parameters:

```python
from typing import Annotated
from fastapi import Header

def handler(
    request: Request,
    x_tenant_id: Annotated[str, Header()],
    x_tenant_app_id: Annotated[int, Header()],
):
```

- `x_tenant_id` — always `str`. Declare it when you need it for audit logging or explicit tenant operations.
- `x_tenant_app_id` — always `int`. Declare it when the route filters by tenant app, or when validating against a request body.

### Path Parameters

Plain typed parameters. Always `int` for IDs:

```python
@router.get("/{plan_id}/stakeholders/{stakeholder_id}")
def get_stakeholder(request: Request, plan_id: int, stakeholder_id: int):
```

### Query Parameters — Filtering

**Simple filters** — use inline `Optional` params with defaults:

```python
def get_items(
    request: Request,
    x_tenant_app_id: Annotated[int, Header()],
    entity_type: str | None = None,
    party_name: str | None = None,
    status: str | None = None,
):
```

**With `Query()` for explicit defaults** — use when you want OpenAPI documentation or validation:

```python
from fastapi import Query
from typing import Optional

def get_plans(
    request: Request,
    tier: Optional[str] = Query(default=None),
    segment: Optional[str] = Query(default=None),
):
```

**List query parameters** — use `Annotated[list[T], Query()]`:

```python
from typing import Annotated
from fastapi import Query

def get_items(
    request: Request,
    file_ids: Annotated[list[int], Query()],
    baa_statuses: Annotated[list[BAAStatus] | None, Query()] = None,
):
```

**Filter object pattern** — use when there are 3+ filters with complex types:

```python
# In schema.py
class GetEntitiesFilter(BaseSchema):
    name: Optional[str] = Field(default=None, max_length=100)
    statuses: Optional[list[StatusEnum]] = []
    is_active: Optional[bool] = None

# In controller.py
def get_entities(request: Request, name: str | None = None, ...):
    entity_filter = GetEntitiesFilter(name=name, statuses=statuses, is_active=is_active)
    return crud_service.list_entities(filter=entity_filter)

# In crud_service.py
def list_entities(filter: GetEntitiesFilter | None = None):
    _, db = get_context_vars()
    query = db.query(Entity)
    if filter:
        if filter.name:
            query = query.filter(Entity.name.ilike(f"%{filter.name}%"))
        if filter.statuses and len(filter.statuses) > 0:
            query = query.filter(Entity.status.in_(filter.statuses))
        if filter.is_active is not None:
            query = query.filter(Entity.is_active == filter.is_active)
    return query.all()
```

**Reference:** `app/organization/controller.py`, `app/organization/crud_service.py`

### Body Parameters

Pydantic schema as type hint. Naming: `Create*Body`, `Post*Body` for creation; `Patch*Body` for partial update; `Put*Body` for full replacement:

```python
def create_item(request: Request, body: CreateItemBody):
def update_item(request: Request, item_id: int, body: PatchItemBody):
```

### tenant_app_id Body Validation

When a POST/PATCH body contains `tenant_app_id`, **always validate it matches the header**:

```python
def create_item(
    request: Request,
    body: CreateItemBody,
    x_tenant_app_id: Annotated[int, Header()],
):
    if body.tenant_app_id != x_tenant_app_id:
        raise HTTPException(
            status_code=HTTPStatus.BAD_REQUEST,
            detail="tenant_app_id in body does not match X-Tenant-App-Id header",
        )
```

### File Upload Parameters

```python
from fastapi import UploadFile, Form

# Single file
def upload(request: Request, file: UploadFile):

# Multiple files
def upload_bulk(request: Request, files: list[UploadFile]):

# File + form fields (no JSON body — use Form() for each field)
def upload_with_metadata(
    request: Request,
    file: UploadFile,
    entity_id: Annotated[int, Form()],
    description: Annotated[str | None, Form()] = None,
):
```

---

## 3. Response Patterns

### 3.1 List Endpoints

**Preferred pattern — `model_validate` loop:**

```python
@router.get("", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.MODULE_VIEW.value)
@is_app([PlatformApps.APP_NAME])
@limiter.limit("60/minute")
def get_items(request: Request, x_tenant_app_id: Annotated[int, Header()]):
    tenant_app = request.state.tenant_app
    items = crud_service.get_items(tenant_app_id=tenant_app.id)
    return [ItemSummarySchema.model_validate(i) for i in items]
```

**Why `model_validate`?** It controls exactly which fields are serialized, catches schema/model mismatches early, and makes return types explicit.

**Acceptable alternative — direct CRUD return** (when the CRUD function already returns the exact shape needed and the route has a `response_model` or typed return):

```python
def get_items(request: Request) -> list[ItemSchema]:
    return crud_service.get_items()
```

**Empty results:** Return an empty list, never `None`:

```python
if not items:
    return []
```

### 3.2 Detail Endpoints (GET by ID)

```python
@router.get("/{item_id}", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.MODULE_VIEW.value)
@is_app([PlatformApps.APP_NAME])
@limiter.limit("60/minute")
def get_item(request: Request, item_id: int):
    item = crud_service.get_by_id(item_id)
    if not item:
        raise HTTPException(
            status_code=HTTPStatus.NOT_FOUND,
            detail=f"Item {item_id} not found",
        )
    return ItemSchema.model_validate(item)
```

### 3.3 Create Endpoints (POST)

Set `HTTPStatus.CREATED` on the response object. Return the created entity via `model_validate`:

```python
from http import HTTPStatus

@router.post("", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.MODULE_CREATE.value)
@is_app([PlatformApps.APP_NAME])
@limiter.limit("30/minute")
def create_item(
    request: Request,
    response: Response,
    body: CreateItemBody,
    x_tenant_app_id: Annotated[int, Header()],
):
    if body.tenant_app_id != x_tenant_app_id:
        raise HTTPException(
            status_code=HTTPStatus.BAD_REQUEST,
            detail="tenant_app_id in body does not match X-Tenant-App-Id header",
        )

    current_user = get_current_user()
    org_id = get_current_user_organization_id()
    item = crud_service.create_item(body, current_user.email)

    create_audit_log(
        user_email=current_user.email,
        user_organization_id=org_id,
        entity_name=AuditEntityName.ENTITY_NAME,
        entity_action=AuditEntityAction.ADDED,
        entity_data=jsonable_encoder(ItemSchema.model_validate(item)),
        api_request_data=request,
    )

    response.status_code = HTTPStatus.CREATED
    return ItemSchema.model_validate(item)
```

### 3.4 Update Endpoints (PATCH)

PATCH for partial updates. Use `model_dump(exclude_unset=True)` to only update provided fields:

```python
@router.patch("/{item_id}", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.MODULE_EDIT.value)
@is_app([PlatformApps.APP_NAME])
@limiter.limit("30/minute")
def update_item(request: Request, item_id: int, body: PatchItemBody):
    current_user = get_current_user()
    item = crud_service.get_by_id(item_id)
    if not item:
        raise HTTPException(
            status_code=HTTPStatus.NOT_FOUND,
            detail=f"Item {item_id} not found",
        )

    for key, value in body.model_dump(exclude_unset=True).items():
        setattr(item, key, value)
    item.updated_by = current_user.email
    item.updated_at = datetime.now()
    crud_service.update_item(item)

    return ItemSchema.model_validate(item)
```

**Reference:** `app/intelligent_workspace/launch_planning/controller.py` (PATCH account plan, stakeholder, milestone)

### 3.5 Delete Endpoints

Use `HTTPStatus.NO_CONTENT` in the route decorator. No return body:

```python
@router.delete(
    "/{item_id}",
    dependencies=[Depends(authenticated)],
    status_code=HTTPStatus.NO_CONTENT,
)
@version(1)
@has_permissions(BuiltInPermissions.MODULE_DELETE.value)
@is_app([PlatformApps.APP_NAME])
@limiter.limit("30/minute")
def delete_item(request: Request, item_id: int):
    current_user = get_current_user()
    org_id = get_current_user_organization_id()
    item = crud_service.get_by_id(item_id)
    if not item:
        raise HTTPException(
            status_code=HTTPStatus.NOT_FOUND,
            detail=f"Item {item_id} not found",
        )

    crud_service.delete_item(item)

    create_audit_log(
        user_email=current_user.email,
        user_organization_id=org_id,
        entity_name=AuditEntityName.ENTITY_NAME,
        entity_action=AuditEntityAction.DELETED,
        entity_data=jsonable_encoder(ItemSchema.model_validate(item)),
        api_request_data=request,
    )
    # No return — 204 is set in the decorator
```

**Reference:** `app/intelligent_workspace/launch_planning/controller.py` (DELETE stakeholder, team member)

### 3.6 Action Endpoints (Non-CRUD)

For actions like accept-terms, resend-invite, or approve — return `"Success"` string:

```python
@router.post("/{participant_id}/accept-terms", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.MODULE_VIEW.value)
@limiter.limit("30/minute")
def accept_terms(request: Request, participant_id: int):
    # ... business logic ...
    return "Success"
```

### 3.7 Bulk Response

For endpoints that create multiple resources, return a wrapper schema:

```python
# schema.py
class BulkCreateResponse(BaseSchema):
    created_ids: list[int]
    errors: list[str] = []

# controller.py
return BulkCreateResponse(created_ids=ids, errors=errors)
```

---

## 4. Status Code Reference

| Operation                            | Status Code | How to Set                                                         |
| ------------------------------------ | ----------- | ------------------------------------------------------------------ |
| GET (list or detail)                 | `200`       | Implicit (default)                                                 |
| POST (create)                        | `201`       | `response.status_code = HTTPStatus.CREATED` in handler body        |
| PATCH/PUT (update)                   | `200`       | Implicit                                                           |
| DELETE                               | `204`       | `status_code=HTTPStatus.NO_CONTENT` in route decorator             |
| Action (accept-terms, resend-invite) | `200`       | Implicit                                                           |
| Client error (validation, mismatch)  | `400`       | `HTTPException(status_code=HTTPStatus.BAD_REQUEST, ...)`           |
| Authorization failure                | `403`       | `HTTPException(status_code=HTTPStatus.FORBIDDEN, ...)`             |
| Not found                            | `404`       | `HTTPException(status_code=HTTPStatus.NOT_FOUND, ...)`             |
| Internal error                       | `500`       | `HTTPException(status_code=HTTPStatus.INTERNAL_SERVER_ERROR, ...)` |

**Always use `HTTPStatus` enum** from `from http import HTTPStatus`, never bare integers (except for legacy `404`).

---

## 5. Error Handling

### HTTPException Patterns

```python
from http import HTTPStatus
from fastapi import HTTPException

# Not found
raise HTTPException(
    status_code=HTTPStatus.NOT_FOUND,
    detail=f"Contract with ID: {contract_id} not found",
)

# Validation / business rule violation
raise HTTPException(
    status_code=HTTPStatus.BAD_REQUEST,
    detail="tenant_app_id in body does not match X-Tenant-App-Id header",
)

# Authorization failure
raise HTTPException(
    status_code=HTTPStatus.FORBIDDEN,
    detail="Not authorized to access launch planning",
)
```

### Error Message Conventions

| Scenario             | Message Pattern                                                                          |
| -------------------- | ---------------------------------------------------------------------------------------- |
| Entity not found     | `f"<Entity> {id} not found"` or `f"<Entity> with ID: {id} not found"`                    |
| Tenant app not found | `"Tenant <AppName> app not found"`                                                       |
| Body mismatch        | `"tenant_app_id in body does not match X-Tenant-App-Id header"`                          |
| Authorization        | `"Not authorized to access <feature>"` or `"User does not have access to this <entity>"` |
| Duplicate            | `"<Entity> already exists for this <scope>"`                                             |

### Catching Service-Layer Errors

When calling service functions that may raise `ValueError`:

```python
try:
    result = service.complex_operation(data)
except ValueError as e:
    logger.error(f"Error in operation: {e}")
    raise HTTPException(
        status_code=HTTPStatus.BAD_REQUEST,
        detail=str(e),
    )
```

### Transaction Error Handling

For multi-step operations, use `begin_nested()` with explicit rollback:

```python
_, db = get_context_vars()
db.begin_nested()
try:
    entity_a = crud_service.create_a(data_a, in_transaction=True)
    entity_b = crud_service.create_b(data_b, in_transaction=True)
    db.commit()
except HTTPException as e:
    logger.error(e.detail)
    db.rollback()
    raise e
except Exception as e:
    logger.error(f"Error in transaction: {e}")
    db.rollback()
    raise HTTPException(
        status_code=HTTPStatus.INTERNAL_SERVER_ERROR,
        detail="Failed to complete operation",
    )
```

**Key:** Catch `HTTPException` separately and re-raise — this preserves intentional 400/404 errors from nested CRUD calls. Catch `Exception` for unexpected failures.

**Reference:** `app/organization/controller.py` (update app access), `app/chat/controller.py` (chat attachment upload)

---

## 6. `request.state.tenant_app` — Using the Loaded Tenant App

### What `@is_app` Does

The `@is_app` decorator:

1. Reads `X-Tenant-App-Id` from context
2. Loads the `TenantApp` model from DB (includes `.tenant` relationship)
3. Validates the app name is in the allowed list
4. **Stores it on `request.state.tenant_app`**

### DO: Use `request.state.tenant_app` After `@is_app`

```python
@router.get("", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.MODULE_VIEW.value)
@is_app([PlatformApps.APP_NAME])
@limiter.limit("30/minute")
def get_items(request: Request):
    tenant_app = request.state.tenant_app                  # Already loaded by @is_app
    tenant = request.state.tenant_app.tenant               # Tenant relationship
    items = crud_service.get_items(tenant_app_id=tenant_app.id)
    return [ItemSchema.model_validate(i) for i in items]
```

### DON'T: Re-fetch When `@is_app` Is Present

```python
# WRONG — redundant DB call, @is_app already loaded this
@is_app([PlatformApps.APP_NAME])
def get_items(request: Request, x_tenant_app_id: Annotated[int, Header()]):
    tenant_app = get_tenant_app_by_id(x_tenant_app_id)  # DON'T — use request.state.tenant_app
```

### Routes Without `@is_app` Must Fetch Themselves

If a route intentionally omits `@is_app` (e.g., cross-app or platform-level), it must load the tenant app:

```python
@router.get("", dependencies=[Depends(authenticated)])
@version(1)
@has_permissions(BuiltInPermissions.VBC_VIEW.value)
@limiter.limit("30/minute")
def get_items(request: Request, x_tenant_app_id: Annotated[int, Header()]):
    tenant_app = get_tenant_app_by_id(x_tenant_app_id)
    if not tenant_app:
        raise HTTPException(status_code=HTTPStatus.NOT_FOUND, detail="Tenant app not found")
    ...
```

---

## 7. Three-User Authorization

Every app-scoped route must consider three caller types: **Admin** (platform operator), **Manufacturer** (customer who owns the tenant), and **Payer** (participant organization).

### Pattern A: Helper Function (Preferred for Sub-Modules)

Extract authorization into a private helper when the same check applies to every route in a module:

```python
from http import HTTPStatus

from common.security import get_current_user, get_current_user_organization_id
from common.utils import is_customer_user
from fastapi import HTTPException, Request


def _verify_manufacturer_or_admin(request: Request):
    """Verify caller is Admin or the manufacturer. Raises 403 for payer users."""
    current_user = get_current_user()
    org_id = get_current_user_organization_id()
    tenant = request.state.tenant_app.tenant

    if current_user.is_admin_user:
        return
    if is_customer_user(current_user, tenant, org_id):
        return
    raise HTTPException(
        status_code=HTTPStatus.FORBIDDEN,
        detail="Not authorized to access this feature",
    )
```

Call as the first line in every handler:

```python
def create_item(request: Request, body: CreateItemBody):
    _verify_manufacturer_or_admin(request)
    # ... rest of handler ...
```

**Reference:** `app/intelligent_workspace/launch_planning/controller.py`

### Pattern B: Inline Check (For One-Off Routes)

For routes where only one or two need the check:

```python
current_user = get_current_user()
org_id = get_current_user_organization_id()
tenant = request.state.tenant_app.tenant

if not (current_user.is_admin_user or is_customer_user(current_user, tenant, org_id)):
    raise HTTPException(
        status_code=HTTPStatus.FORBIDDEN,
        detail="User does not have access to this resource",
    )
```

**Reference:** `app/intelligent_workspace/controller.py`

### Pattern C: Full Three-Way Branching with Data Scoping

When Admin, Manufacturer, and Payer all have access but see **different data**:

```python
current_user = get_current_user()
org_id = get_current_user_organization_id()
tenant_app = request.state.tenant_app

if current_user.is_admin_user or is_customer_user(current_user, tenant_app.tenant, org_id):
    # Admin or Manufacturer: full access to all data
    items = crud_service.get_all_items(tenant_app_id=tenant_app.id)
else:
    # Payer: scoped to their participant's data
    if not participant_id:
        raise HTTPException(
            status_code=HTTPStatus.BAD_REQUEST,
            detail="Payer id is required",
        )
    participant = get_participant_by_id(participant_id)
    if not participant or participant.organization_id != org_id:
        raise HTTPException(
            status_code=HTTPStatus.FORBIDDEN,
            detail="User does not have access to this participant",
        )
    items = crud_service.get_items_for_participant(participant_id=participant.id)
```

**Reference:** `app/vbc/controller.py`

### When to Use Which

| Scenario                                               | Pattern                                                                |
| ------------------------------------------------------ | ---------------------------------------------------------------------- |
| All routes in a sub-module are manufacturer/Admin only | **A** — helper function                                                |
| One-off route needs the check                          | **B** — inline                                                         |
| Route returns different data per caller type           | **C** — full branching                                                 |
| Route explicitly blocks Admin (e.g., chat messaging)   | Custom: `if current_user.is_admin_user: raise HTTPException(403, ...)` |

### Key Functions

| Function                                   | From                     | Returns    | Purpose                                                                                             |
| ------------------------------------------ | ------------------------ | ---------- | --------------------------------------------------------------------------------------------------- |
| `current_user.is_admin_user`               | Property on `User` model | `bool`     | `True` if email domain is `@1admin.com` or `@platform.ai`                                             |
| `is_customer_user(user, tenant, org_id)`   | `common.utils`           | `bool`     | `True` if user's org matches the tenant's owning org (manufacturer)                                 |
| `is_valid_user_organization(user, org_id)` | `common.utils`           | Raises 401 | Side-effect: raises `HTTPException(401)` if non-Admin user doesn't match org. Use for invite routes |
| `get_current_user_organization_id()`       | `common.security`        | `int`      | From `X-Organization-Id` header                                                                     |

---

## 8. Audit Logging

### When to Audit

| Operation     | Audit? | Action Enum                                                                     |
| ------------- | ------ | ------------------------------------------------------------------------------- |
| Create        | Yes    | `AuditEntityAction.ADDED`                                                       |
| Update        | Yes    | `AuditEntityAction.UPDATED`                                                     |
| Delete        | Yes    | `AuditEntityAction.DELETED`                                                     |
| Invite        | Yes    | `AuditEntityAction.INVITED`                                                     |
| Accept terms  | Yes    | `AuditEntityAction.VBC_TERMS_ACCEPTED` / `INTELLIGENT_WORKSPACE_TERMS_ACCEPTED` |
| Resend invite | Yes    | `AuditEntityAction.RESENT_INVITE`                                               |
| Approve       | Yes    | `AuditEntityAction.APPROVED`                                                    |
| GET / List    | No     | —                                                                               |

### Audit Log Template

```python
from audit_log.audit_service import create_audit_log
from audit_log.constants import AuditEntityAction, AuditEntityName
from fastapi.encoders import jsonable_encoder

create_audit_log(
    user_email=current_user.email,
    user_organization_id=org_id,
    tenant_id=x_tenant_id,              # Optional — omit if not declared as param
    tenant_app_id=tenant_app.id,        # Optional — omit if not available
    entity_name=AuditEntityName.ENTITY_NAME,
    entity_action=AuditEntityAction.ADDED,
    entity_data=jsonable_encoder(ItemSchema.model_validate(item)),
    api_request_data=request,
)
```

**`entity_data` options:**

- `jsonable_encoder(Schema.model_validate(entity))` — serialized entity (preferred for creates)
- `body.model_dump()` — raw request body (useful for updates to capture what changed)
- `None` — omit for simple actions like accept-terms

**Background audit log** — for streaming responses where the handler returns before logging completes:

```python
from fastapi import BackgroundTasks

def handler(request: Request, background_tasks: BackgroundTasks):
    # ... streaming response setup ...
    background_tasks.add_task(
        create_audit_log,
        user_email=current_user.email,
        user_organization_id=org_id,
        entity_name=AuditEntityName.ENTITY_NAME,
        entity_action=AuditEntityAction.ADDED,
        entity_data=entity_data,
        api_request_data=request,
    )
```

**New entity names:** If the entity doesn't exist in `AuditEntityName`, add it to `app/audit_log/constants.py`.

---

## 9. Transaction Management

### Default: Let CRUD Handle It

Most routes don't need explicit transaction management. CRUD functions call `db.flush()` then `db.commit()` internally:

```python
def create_item(request: Request, body: CreateItemBody):
    item = crud_service.create_item(body)  # flush + commit happens inside
    return ItemSchema.model_validate(item)
```

### Multi-Step Operations: `in_transaction=True`

When a route calls **multiple CRUD functions that must succeed or fail together**, use `begin_nested()` and pass `in_transaction=True`:

```python
_, db = get_context_vars()
db.begin_nested()
try:
    parent = crud_service.create_parent(data, in_transaction=True)
    for child_data in children:
        crud_service.create_child(child_data, parent.id, in_transaction=True)
    db.commit()
except HTTPException as e:
    db.rollback()
    raise e
except Exception as e:
    logger.error(f"Transaction failed: {e}")
    db.rollback()
    raise HTTPException(
        status_code=HTTPStatus.INTERNAL_SERVER_ERROR,
        detail="Failed to complete operation",
    )
```

**Rules:**

- `in_transaction=True` tells CRUD to `flush()` but NOT `commit()`
- The controller is responsible for calling `db.commit()` on success
- Always `db.rollback()` in the `except` block
- Catch `HTTPException` separately and re-raise to preserve intentional 400/404 errors

### Update Pattern: Controller Mutates, CRUD Persists

The preferred update flow is: controller fetches entity, mutates fields, passes to CRUD for persistence:

```python
# In controller
item = crud_service.get_by_id(item_id)
for key, value in body.model_dump(exclude_unset=True).items():
    setattr(item, key, value)
item.updated_by = current_user.email
item.updated_at = datetime.now()
crud_service.update_item(item)

# In crud_service
def update_item(item: MyModel, in_transaction: bool = False):
    _, db = get_context_vars()
    db.add(item)
    db.flush()
    if not in_transaction:
        db.commit()
    return item
```

---

## 10. Schema Design Conventions

### Naming

| Suffix                               | Purpose                                   | Example                                                 |
| ------------------------------------ | ----------------------------------------- | ------------------------------------------------------- |
| `*Schema`                            | Full response/detail                      | `AccountPlanSchema`, `ChatSchema`                       |
| `*SummarySchema` / `*ListItemSchema` | Lightweight list view (omits nested data) | `AccountPlanSummarySchema`, `GTNScenarioListItemSchema` |
| `Basic*Schema`                       | Minimal embedding in other schemas        | `BasicOrganizationSchema`                               |
| `Create*Body` / `Post*Body`          | POST request body                         | `CreateAccountPlanBody`, `PostChatBody`                 |
| `Patch*Body`                         | Partial update body (all fields Optional) | `PatchStakeholderBody`, `PatchMilestoneBody`            |
| `Put*Body`                           | Full replacement of sub-resource          | `PutGTNScenarioPeriodBody`                              |
| `Get*Filter`                         | Filter object for list queries            | `GetOrganizationsFilter`                                |

### Field Conventions

```python
from common.base_schema import BaseSchema
from datetime import datetime
from decimal import Decimal
from typing import Optional
from pydantic import Field

class ItemSchema(BaseSchema):
    # Standard metadata — always present on response schemas
    id: int
    created_by: str
    created_at: datetime                        # datetime, not str
    updated_by: Optional[str] = None
    updated_at: Optional[datetime] = None

    # Business fields
    name: str
    status: str                                 # Enum fields: str in response schemas
    amount: Decimal                             # Money: always Decimal, never float
    children: list[ChildSchema] = []            # Nested lists: default to empty list


class CreateItemBody(BaseSchema):
    tenant_app_id: int = Field(gt=0)            # Required with validation
    name: str = Field(min_length=1, max_length=255)
    amount: Decimal = Field(ge=0, decimal_places=2)
    status: ItemStatusEnum                      # Enum type in request bodies


class PatchItemBody(BaseSchema):
    name: Optional[str] = Field(default=None, min_length=1, max_length=255)
    amount: Optional[Decimal] = None            # ALL fields Optional for PATCH
    status: Optional[ItemStatusEnum] = None
```

### Field Validators

For constrained-value fields where an enum exists but you want a custom error:

```python
from pydantic import field_validator

_VALID_TIERS = {t.value for t in TierEnum}

class CreateItemBody(BaseSchema):
    tier: Optional[str] = None

    @field_validator("tier")
    @classmethod
    def validate_tier(cls, v):
        if v is not None and v not in _VALID_TIERS:
            raise ValueError(f"Invalid tier: {v}. Must be one of {sorted(_VALID_TIERS)}")
        return v
```

**Reference:** `app/intelligent_workspace/launch_planning/schema.py`

### BaseSchema Enum Gotcha (CRITICAL)

`BaseSchema` has `use_enum_values=True`. After Pydantic parses a request body, **enum fields are strings**, not enum instances:

```python
# Schema field: granularity: MyEnum

# WRONG — body.granularity is already a string, not an enum
if body.granularity == MyEnum.OPTION_A:
    ...

# CORRECT
if body.granularity == MyEnum.OPTION_A.value:
    ...
```

---

## 11. CRUD Service Patterns

### Query Building with Conditional Filters

Build queries incrementally. Only join when the filter is actually provided:

```python
def get_items(tenant_app_id: int, tier: str | None = None, segment: str | None = None):
    _, db = get_context_vars()
    query = db.query(Item).filter(Item.tenant_app_id == tenant_app_id)

    if tier:
        query = query.join(Item.participant).filter(Participant.priority_tier == tier)
    if segment:
        if not tier:  # Avoid double-join
            query = query.join(Item.participant)
        query = query.filter(Participant.segment == segment)

    return query.all()
```

### Text Search Filters

Use `.ilike()` for case-insensitive partial matching (Azure SQL is CI by default, but `ilike` ensures portability):

```python
if name and len(name) > 0:
    query = query.filter(Entity.name.ilike(f"%{name}%"))
```

### Eager Loading — List vs. Detail

```python
# List view — only load what's needed for summary/hybrid_property
def get_all(tenant_app_id: int):
    _, db = get_context_vars()
    return db.query(Parent).options(
        joinedload(Parent.milestones),   # Needed for hybrid_property
    ).filter(Parent.tenant_app_id == tenant_app_id).all()

# Detail view — load all children
def get_by_id(item_id: int):
    _, db = get_context_vars()
    return db.query(Parent).options(
        joinedload(Parent.stakeholders),
        joinedload(Parent.team_members),
        joinedload(Parent.milestones),
        joinedload(Parent.notes),
    ).filter(Parent.id == item_id).first()
```

### Not-Found Handling in CRUD

Two patterns — pick based on usage:

1. **Return `None`** — let the controller decide what to do (most `get_*_by_id` functions):

   ```python
   def get_by_id(item_id: int) -> Item | None:
       _, db = get_context_vars()
       return db.query(Item).filter(Item.id == item_id).first()
   ```

2. **Raise directly** — for prerequisite/ownership checks (`verify_*` functions):

   ```python
   def verify_item(item_id: int, tenant_app_id: int) -> Item:
       _, db = get_context_vars()
       item = db.query(Item).filter(Item.id == item_id, Item.tenant_app_id == tenant_app_id).first()
       if not item:
           raise HTTPException(status_code=HTTPStatus.NOT_FOUND, detail=f"Item {item_id} not found")
       return item
   ```

---

## 12. Sub-Module Router Wiring

### Registration

Sub-module routers register on the **parent module's router**, not in `main.py`:

```python
# In parent controller.py
from feature_module.sub_module.controller import sub_module_router

feature_module_router.include_router(sub_module_router)
```

### Tag Convention

Sub-module routers use a **two-element tag list** — parent tag first:

```python
sub_module_router = APIRouter(
    prefix="/sub-module",
    tags=["Parent Module", "Sub Module"],
)
```

This groups endpoints in Swagger under both the parent and sub-module.

### URL Nesting

Prefixes compose automatically:

- Parent: `prefix="/intelligent-workspace"`
- Sub-module: `prefix="/launch-planning"`
- Result: `/v1/intelligent-workspace/launch-planning/...`

### `request.state` in Sub-Modules

The `@is_app` decorator on the parent route or sub-module route stores `tenant_app` on `request.state`. Sub-module handlers access it identically:

```python
tenant_app = request.state.tenant_app
tenant = request.state.tenant_app.tenant
```

This works because the same `Request` object flows through the entire middleware/decorator chain.

---

## 13. Don'ts

- **Don't use `async def`** for route handlers — the app uses sync SQLAlchemy.
- **Don't re-fetch `tenant_app`** from DB when `@is_app` is present — use `request.state.tenant_app`.
- **Don't return bare integers** for status codes — use `HTTPStatus.NOT_FOUND` not `404`.
- **Don't return `None`** from list endpoints — return `[]`.
- **Don't mix `HTTPException` and bare string returns** in the same error path.
- **Don't skip `body.tenant_app_id` validation** on POST/PATCH routes that accept it.
- **Don't forget `in_transaction=True`** when calling multiple CRUD functions in a single transaction.
- **Don't catch `Exception` without re-raising** — at minimum, raise `HTTPException(500)`.
- **Don't use `JSONResponse` with `jsonable_encoder`** for normal responses — let FastAPI serialize via `model_validate`. The `JSONResponse` pattern is legacy.
- **Don't log sensitive data** (Bearer tokens, PII) — audit logging already redacts the Authorization header.
- **Don't compare Pydantic enum fields with enum instances** — `BaseSchema` has `use_enum_values=True`, so parsed values are strings.
