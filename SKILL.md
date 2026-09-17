---
name: galaxy-code-standards
description: Coding standards for the Galaxy (galaxyproject/galaxy) codebase, derived from the gx-arch-review code review plugin. Use BEFORE and WHILE writing, modifying, or planning any code in a Galaxy checkout, including Python backend code (lib/galaxy, API endpoints, managers, services, models, Celery tasks, Alembic migrations), Python tests (test/unit, lib/galaxy_test, test/integration), and client code (client/src Vue components, TypeScript, vitest tests). Ensures generated code passes /gx-arch-review:gx-review.
---

# Galaxy Code Standards

Rules for writing Galaxy code that passes the `gx-arch-review` review commands
(https://github.com/jmchilton/gx-arch-review). Each section below maps to one review
command. Apply the rules while writing code, not as an afterthought, then run the
self-review in the last section before declaring the work done.

These rules also apply to planning documents: a plan for Galaxy work must describe code
that satisfies them (for example, it must say a migration goes in its own commit).

## Step 1: Decide which rules apply

| The change contains | Apply | Review command it must pass |
|---|---|---|
| Any Python code | Python structure, Dependency injection | `py-review-code-structure`, `gx-review-di` |
| A new or modified API endpoint | FastAPI layer, Business logic layers | `gx-fastapi-review`, `gx-review-business-logic-organization` |
| Python tests | Test placement, Mocks and patches | `gx-review-test-types`, `py-challenge-patches` |
| An Alembic migration | Migrations | `gx-review-migration` |
| A new database model | Models | `gx-review-model` |
| Vue components | UI components | `gx-review-ui-components` |
| Client tests (`*.test.ts`, `*.test.js`) | Vitest tests | `gx-vitest-review` |

Read the matching reference file when a section applies and you need the full patterns:

- `references/python-backend.md`: layers, DI, FastAPI, migrations, models, code structure
- `references/python-tests.md`: test type decision tree, red flags, patch elimination techniques
- `references/client.md`: G-component replacements and prop mappings, vitest practices

## Python structure

- Annotate parameters and return types on public methods and on any function with a
  non-obvious signature. Annotate complex types (dicts, lists of objects, unions). Do not
  over-annotate trivial local helpers.
- All imports at the top of the file. An inline import is only allowed with a trailing
  comment giving the reason, for example `# circular import`, `# lazy load`, `# conditional`.
- Do not reorder import groups by hand; isort owns that.
- Docstrings describe the caller-facing contract only: purpose, parameters, return value,
  raised exceptions, durable behavior. No local control flow, internal collaborators, or
  implementation choices in docstrings. Necessary rationale goes in a short inline comment
  next to the statement it explains; omit it when the code is clear.

## Dependency injection

Galaxy uses Lagom type-based DI. The `app` god object is being retired.

- Inject dependencies through type-annotated constructor parameters. Never pull them off
  `app` (`app.model`, `app.security`, `app.some_manager`).
- Never construct a collaborator inside a component (`self.hda_manager = HDAManager(app)`).
- Do not add new properties to `app` when DI can supply the object.
- If an app reference is unavoidable, type it `StructuredApp`, never `UniverseApplication`.
  Best is to inject only what is needed (`GalaxyModelMapping`, `IdEncodingHelper`, specific managers).
- The dependency graph must be a DAG. No cycles.
- Controllers use Galaxy's `depends(Type)`, never FastAPI's `Depends(factory)`:
  `manager: TagsManager = depends(TagsManager)`. This holds for `@cbv(router)` FastAPI
  classes and for legacy `BaseGalaxyAPIController` classes.
- Celery tasks stack `@galaxy_task` under `@celery_app.task` and receive managers as
  type-annotated parameters.

## FastAPI layer

- Never call `include_router` or otherwise register a router. Galaxy discovers routers
  automatically.
- Keep the endpoint thin and use type-based DI (see the other sections).

## Business logic layers

Controllers (`lib/galaxy/webapps/galaxy/api/`) sit on optional Services
(`lib/galaxy/webapps/galaxy/services/`), which sit on Managers (`lib/galaxy/managers/`),
which sit on Models (`lib/galaxy/model/`).

- Controllers: route definition, parameter extraction, response formatting, HTTP status
  codes. Nothing else. No database queries, no business validation, no complex
  conditionals, no multi-step operations. The ideal endpoint body is one delegating call:
  `return self.manager.create(trans, payload)`.
- Services: optional and thin. API and web processing details only, shielding application
  logic from FastAPI. Skipping this layer is fine. A thick service is an anti-pattern.
- Managers: all business logic, validation rules, permission checks, coordination across
  models, side effects. No HTTP handling or request/response formatting. A manager that
  only passes through to a model while the logic lives in the controller is an anti-pattern.
- Models: columns, relationships, simple derived properties, basic serialization. No
  business logic, no cross-model coordination, no external service calls.

## Migrations

- Put the Alembic migration in its own commit, separate from the code that uses it. When
  writing a plan, state this explicitly in the plan.
- Before writing one, read several recent migrations under `lib/galaxy/model/migrations/`
  and match their patterns. Use the existing migration util abstractions rather than raw
  Alembic operations, and add a new util when a new reusable operation is needed.

## Models

The upstream `gx-review-model` command is currently empty, so there are no model-specific
rules beyond the Models bullet under Business logic layers. Follow the declarative mapping
conventions already used in `lib/galaxy/model/__init__.py`, and pair any schema change
with a migration that follows the Migrations section.

## Test placement

Placing a test in the wrong location is the most common review failure. Decide the type
before creating the file:

| Needs | Type | Location | Base class |
|---|---|---|---|
| No running server | Unit | `test/unit/` or doctests | none |
| Server, default config, tools only | Framework | `test/functional/tools/` | |
| Server, default config, workflows only | Workflow framework | `lib/galaxy_test/workflow/` | |
| Server, default config, API only | API | `lib/galaxy_test/api/` | `ApiTestCase` |
| Server plus custom config or internals | Integration | `test/integration/` | `IntegrationTestCase` |
| Browser, default config | Selenium | `lib/galaxy_test/selenium/` | |
| Browser plus custom config | Selenium integration | `test/integration_selenium/` | |

- An API test must be runnable against any external Galaxy with default config. It uses
  populators (`DatasetPopulator`, `WorkflowPopulator`, `DatasetCollectionPopulator`) and
  `self._get/_post/_put/_delete` only.
- The test is an integration test if it needs ANY of: `handle_galaxy_config_kwds`,
  `self._app`, direct `galaxy.model` database queries, features that are off by default
  (quotas, object stores, job runners), or external services such as Docker containers.
- Do not write an integration test that leaves config unmodified and only makes API calls.
  That is an API test.
- Always call `super().handle_galaxy_config_kwds(config)` in an override.
- Do not touch the database directly when the API would do.
- Unit tests with external dependencies carry the `@external_dependency_management` marker.

## Mocks and patches

- A test must never merely test its own mock. Before adding `mock.patch`, look for a way
  to avoid it, in this order of preference: inject the collaborator (DI), use an
  in-memory fake, use the real object with a constrained environment (`tmp_path`, SQLite),
  or extract a thin port/adapter around the side effect.
- Assert state and results, not interactions. Prefer `assert result.status == "sent"` over
  `mock.assert_called_once_with(x)`.
- Wrap time, randomness, and ID generation behind injectable abstractions (Clock,
  IdGenerator, seeded generator) instead of patching `datetime.now()` or `uuid.uuid4()`.
- Test external service adapters with recorded fixtures, golden JSON, or stub HTTP servers
  that still exercise real parsing and validation.

## UI components

No new Bootstrap-Vue usage in new or modified Vue code, in imports or in template tags.

| Do not use | Use |
|---|---|
| `BButton`, `b-button` | `GButton` with `color` (not `variant`), `size="small"` (not `"sm"`) |
| `BLink`, `b-link` | `GLink` |
| `BModal`, `b-modal` | `GModal` with `v-model:show` (not `v-model`), `confirm` (not `ok-only`) |
| `BCard`, `b-card`, hand-rolled card markup | `GCard` |
| `v-b-tooltip` directive | `tooltip` prop plus `title` |

Colors are `grey`, `blue`, `green`, `yellow`, `orange`, `red`. Sizes are `small`,
`medium`, `large`. Read the component source in `client/src/components/BaseComponents/`
before using one; see `references/client.md` for mappings.

## Vitest tests

- Test user-visible behavior, not implementation. No `vi.spyOn(wrapper.vm, ...)` and no
  assertions on `wrapper.vm`; use `mount()` and assert on rendered output and emitted events.
- Test Pinia stores in isolation with `setActivePinia(createPinia())` in `beforeEach`.
- `await flushPromises()` after async work. `vi.clearAllMocks()` in `beforeEach` and
  `vi.restoreAllMocks()` in `afterEach`.
- One behavior per test, with a descriptive name such as
  `displays error banner when API returns 500`. Never `works correctly`.
- Mock external services (API, router) only. Exercise component logic with real data.
- Do not over-test: no tests of Vue or Pinia themselves, no test per method or computed
  property. Cover error states, empty data, and boundaries. Write TypeScript.
- Every test must be seen red. After writing a test, break the code under test (or the
  assertion) and confirm the test fails, then restore it. Run with `yarn test` from `client/`.
- Extract repeated setup into helpers instead of duplicating it.

## Step 2: Self-review before finishing

Walk this list against the diff. Fix every hit before reporting the work complete.

Python
- [ ] Public functions and methods annotated, including return types
- [ ] No unexplained inline imports
- [ ] Docstrings hold contract only
- [ ] No `app.<something>` dependency access, no internal construction of collaborators, no new `app` properties
- [ ] No `UniverseApplication` type annotations; `depends()` not `Depends()`
- [ ] Celery tasks use `@galaxy_task` with typed parameters
- [ ] No `include_router`
- [ ] Endpoint bodies only delegate; logic is in a manager; models hold no business logic
- [ ] Migration follows existing util patterns and sits in its own commit

Python tests
- [ ] Each test file is in the location its content requires
- [ ] `super().handle_galaxy_config_kwds(config)` called
- [ ] No test that only verifies a mock; state assertions over call assertions

Client
- [ ] No Bootstrap-Vue imports, tags, or `v-b-tooltip` in touched code
- [ ] `color`, `size="small"`, `v-model:show`, `tooltip` props used correctly
- [ ] Tests assert behavior, avoid `wrapper.vm`, and were each seen failing once

If the `gx-arch-review` plugin is loaded in the session, finish by running
`/gx-arch-review:gx-review` on the working directory and resolve its findings.
