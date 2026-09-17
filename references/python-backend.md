# Galaxy Python Backend Patterns

Source: `gx-review-di`, `gx-review-business-logic-organization`, `gx-fastapi-review`,
`gx-review-migration`, `py-review-code-structure` in gx-arch-review.

## Contents

1. Layer architecture
2. Dependency injection patterns
3. FastAPI specifics
4. Migrations
5. Code structure
6. Code paths to read for examples

## 1. Layer architecture

```
API Controllers   lib/galaxy/webapps/galaxy/api/        request/response handling
Services          lib/galaxy/webapps/galaxy/services/   API processing details (thin, optional)
Managers          lib/galaxy/managers/                  business logic
Models            lib/galaxy/model/                     database interaction (SQLAlchemy ORM)
```

Bad, business logic in the controller:

```python
@router.post("/api/histories")
def create_history(payload: CreateHistoryPayload):
    history = model.History()
    history.name = payload.name
    history.user = trans.user
    session.add(history)
    session.commit()
    # validation, side effects...
    return history
```

Good, thin controller delegating to the manager:

```python
@router.post("/api/histories")
def create_history(self, trans: ProvidesHistoryContext, payload: CreateHistoryPayload):
    return self.manager.create(trans, payload)
```

Where things belong:

| Concern | Controller | Manager | Model |
|---|---|---|---|
| Route definitions, parameter extraction, response formatting, status codes | yes | no | no |
| Business logic, validation rules, permission checks | no | yes | no |
| Coordinating multiple models, side effects (notifications, audit logs) | no | yes | no |
| Database queries | no | yes | yes |
| Column definitions, relationships, simple derived properties, basic serialization | no | no | yes |
| External service calls | no | yes | no |

Anti-patterns reviewers flag:

1. Fat controllers: queries, complex validation, or multi-step operations in a controller.
2. Anemic managers: managers that pass straight through to models while logic sits in controllers.
3. Smart models: business logic, external service coordination, or complex validation in models.
4. Missing manager layer: controller talking straight to models for a complex operation.

Existing managers to extend before creating new ones: `histories.py`, `hdas.py`,
`users.py`, `workflows.py`, `collections.py`. Helpers: `base.py` (base classes),
`context.py` (transaction context), `deletable.py`, `sharable.py`.

## 2. Dependency injection patterns

Why: every component holding `app` produced circular imports (a manager imports
`UniverseApplication`, which constructs that manager), untestable code, and brittle
construction order. Lagom resolves dependencies from type annotations, and the graph must
be a DAG.

App typing, worst to best:

```python
class MyManager:                      # BAD: circular dependency
    def __init__(self, app: UniverseApplication): ...

class MyManager:                      # ACCEPTABLE: interface breaks the cycle
    def __init__(self, app: StructuredApp): ...

class MyManager:                      # BEST: inject only what is needed
    def __init__(self, model: GalaxyModelMapping, security: IdEncodingHelper): ...
```

Manager, bad:

```python
class DatasetCollectionManager:
    def __init__(self, app):
        self.model = app.model
        self.security = app.security
        self.hda_manager = hdas.HDAManager(app)  # internal construction
        self.history_manager = histories.HistoryManager(app)
```

Manager, good:

```python
class DatasetCollectionManager:
    def __init__(
        self,
        model: GalaxyModelMapping,
        security: IdEncodingHelper,
        hda_manager: HDAManager,
        history_manager: HistoryManager,
    ):
        self.model = model
        self.security = security
        self.hda_manager = hda_manager
        self.history_manager = history_manager
```

FastAPI controller, bad then good:

```python
def get_tags_manager() -> TagsManager:
    return TagsManager()

@cbv(router)
class FastAPITags:
    manager: TagsManager = Depends(get_tags_manager)  # BAD: FastAPI style
```

```python
@cbv(router)
class FastAPITags:
    manager: TagsManager = depends(TagsManager)  # GOOD: Galaxy style
```

Legacy WSGI controller, bad then good:

```python
class TagsController(BaseAPIController):
    def __init__(self, app):
        super().__init__(app)
        self.manager = TagsManager()  # BAD: manual construction
```

```python
class TagsController(BaseGalaxyAPIController):
    manager: TagsManager = depends(TagsManager)
```

Celery task:

```python
@celery_app.task(ignore_result=True)
@galaxy_task
def purge_hda(hda_manager: HDAManager, hda_id):
    hda = hda_manager.by_id(hda_id)
    hda_manager._purge(hda)
```

Managers come first as typed parameters and are injected; plain arguments follow.

## 3. FastAPI specifics

Galaxy auto-detects routers. Defining `router = Router(tags=[...])` in a module under
`lib/galaxy/webapps/galaxy/api/` is all the registration required. Any `include_router`
call or manual registration fails review.

## 4. Migrations

- Read recent files in `lib/galaxy/model/migrations/` first and mirror them.
- Use Galaxy's migration util abstractions where they exist. If an operation is reusable
  and no util exists, add one rather than inlining raw Alembic calls.
- Commit the migration alone. When producing a plan instead of code, the plan must say the
  migration is committed separately.

## 5. Code structure

Type annotations:
- Required: public methods and functions, non-obvious signatures, complex parameter types.
- Not required: obvious local helpers and simple internal methods.

Imports:
- Top of file after the module docstring.
- Inline import allowed only with a reason comment:

```python
def build():
    from galaxy.tools import Tool  # circular import
```

Docstrings and comments:

```python
def purge(self, hda: HDA) -> None:
    """Permanently remove the dataset's files and mark it purged.

    Raises ItemAccessibilityException if the user does not own the dataset.
    """
    # Quota must be adjusted before the files disappear, size is read from disk.
    self._adjust_quota(hda)
```

The docstring states the contract. The ordering rationale is an inline comment. A
docstring saying "first calls `_adjust_quota`, then loops over..." fails review.

## 6. Code paths to read for examples

- `lib/galaxy/di/`: DI container setup
- `lib/galaxy/structured_app.py`: `StructuredApp` interface
- `lib/galaxy/app.py`: `UniverseApplication`
- `lib/galaxy/managers/`: managers using DI
- `lib/galaxy/webapps/galaxy/api/`: FastAPI controllers using DI
- `lib/galaxy/webapps/galaxy/services/`: service layer
- `lib/galaxy/celery/tasks.py`: Celery tasks using DI
- `lib/galaxy/model/__init__.py`: model definitions
- `lib/galaxy/model/migrations/`: Alembic migrations
