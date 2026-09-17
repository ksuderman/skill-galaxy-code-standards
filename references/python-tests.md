# Galaxy Python Test Patterns

Source: `gx-review-test-types` and `py-challenge-patches` in gx-arch-review.

## Test type decision tree

```
Does test need running Galaxy server?
├─ NO → Unit test (test/unit/ or doctests)
└─ YES → Does test need web browser?
         ├─ NO → Does test need custom Galaxy config?
         │       ├─ NO → Is it testing only tools or workflows?
         │       │       ├─ Tools → Framework test (test/functional/tools/)
         │       │       ├─ Workflows → Workflow Framework test (lib/galaxy_test/workflow/)
         │       │       └─ Neither → API test (lib/galaxy_test/api/)
         │       └─ YES → Integration test (test/integration/)
         └─ YES → Does test need custom config?
                  ├─ NO → Selenium test (lib/galaxy_test/selenium/)
                  └─ YES → Selenium Integration test (test/integration_selenium/)
```

## API test (`lib/galaxy_test/api/`)

Use when ALL are true:
- Needs a running server
- No custom Galaxy configuration
- No direct access to internals (`self._app`, database models)
- Could run against any external Galaxy server

Shape: inherits `ApiTestCase`; uses `DatasetPopulator`, `WorkflowPopulator`,
`DatasetCollectionPopulator`; uses `self._get()`, `self._post()`, `self._put()`,
`self._delete()`; exercises the API only.

## Integration test (`test/integration/`)

Use when ANY is true:
- Needs custom config via `handle_galaxy_config_kwds`
- Needs features off by default (quotas, specific object stores, job runners)
- Needs `self._app` or database models
- Needs external services the test starts (Docker containers and similar)
- Sets options such as `enable_quotas`, `job_config_file`, `object_store_config_file`,
  `metadata_strategy`, `ftp_upload_dir`, custom auth settings

Shape: inherits `IntegrationTestCase`; overrides `handle_galaxy_config_kwds(cls, config)`
and calls `super()` first; may set `require_admin_user`, `framework_tool_and_types`; each
class spins up its own server, so do not create one without need.

```python
class TestQuotaIntegration(IntegrationTestCase):
    require_admin_user = True

    @classmethod
    def handle_galaxy_config_kwds(cls, config):
        super().handle_galaxy_config_kwds(config)
        config["enable_quotas"] = True
```

## Red flags reviewers look for

In an API test (means it belongs in integration):
- `handle_galaxy_config_kwds` present (definite)
- `self._app` access (definite)
- `galaxy.model` imports with direct queries (likely)
- A comment such as "requires X to be enabled" with no config setup (likely)
- `@skip_unless_docker()`, `@skip_unless_postgres()`, `@skip_unless_amqp()` (likely)

In an integration test (means it belongs in API):
- No `handle_galaxy_config_kwds` override, or an override that only calls `super()`
- No `self._app` or model access
- Only populators and HTTP assertions

Also flagged:
- Direct database modification when the API would suffice
- Integration tests mixing config-dependent and plain API cases; split them
- Missing `super().handle_galaxy_config_kwds(config)`
- Unit tests with external dependencies lacking `@external_dependency_management`

## Eliminating mocks and patches

The reviewer finds every mock and patch, asks whether the test only tests the mock, and
asks what abstraction would remove it. Write the test so the answer is already "none needed".

1. Dependency injection over patching. Pass collaborators in explicitly. Galaxy's
   constructor-injected managers make this natural: build the manager with test collaborators.

2. Fakes over mocks. Implement the interface with plain data structures.
   - Databases: in-memory dict or SQLite
   - Queues: lists or deques
   - Caches: dict with TTL logic

3. Real objects with controlled inputs.
   - Temporary directories (`tmp_path`)
   - Local SQLite instead of Postgres
   - Real serializers and parsers with fixed inputs

4. Ports and adapters. Put the side effect behind a minimal interface and substitute a
   fake adapter in tests.

5. State verification over interaction verification.

   ```python
   # Avoid
   mock.assert_called_once_with(x)

   # Prefer
   assert result.status == "sent"
   assert repo.items[id].email_sent is True
   ```

   If the effect matters, assert the effect, not the call.

6. Contract tests for external services: recorded fixtures (VCR style), golden JSON files,
   stub HTTP servers with real parsing and validation.

7. Encapsulate time, randomness, IDs.
   - `datetime.now()` becomes an injected Clock
   - `uuid.uuid4()` becomes an injected IdGenerator
   - `random.*` becomes a seeded generator

If a patch is truly unavoidable, keep it narrow, and make sure the assertions check real
behavior of the code under test rather than the return value the mock was told to give.
