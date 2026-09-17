# galaxy-code-standards

A Claude Code skill that makes Claude write code for the
[Galaxy](https://github.com/galaxyproject/galaxy) codebase that passes Galaxy's
architecture code reviews on the first try.

## What it does

The [gx-arch-review](https://github.com/jmchilton/gx-arch-review) plugin provides slash
commands that review Galaxy contributions after the code is written. This skill takes the
criteria from those review commands and turns them into rules Claude applies while
writing or planning the code, so the review has nothing left to find.

When the skill triggers, Claude:

1. Works out which rule sets apply to the change (Python, API endpoint, tests, migration,
   model, Vue component, client tests).
2. Follows those rules while writing the code, loading the detailed reference files as needed.
3. Runs a self-review checklist against the diff before reporting the work complete.
4. Runs `/gx-arch-review:gx-review` as a final check if the plugin is loaded.

The rules also apply to planning documents, since the review commands accept plans as input.

## Rules covered

| Area | Summary | Review command |
|---|---|---|
| Python structure | Type annotations on public APIs, imports at top of file, docstrings describe the contract only | `py-review-code-structure` |
| Dependency injection | Constructor injection via Lagom, no `app` access, `StructuredApp` over `UniverseApplication`, Galaxy `depends()` over FastAPI `Depends()`, `@galaxy_task` for Celery | `gx-review-di` |
| FastAPI layer | No `include_router`; Galaxy detects routers automatically | `gx-fastapi-review` |
| Business logic layers | Thin controllers, optional thin services, logic in managers, models limited to database concerns | `gx-review-business-logic-organization` |
| Migrations | Follow existing migration utils, migration in its own commit | `gx-review-migration` |
| Models | Upstream command is currently empty; existing model conventions apply | `gx-review-model` |
| Test placement | Unit, framework, API, integration, and Selenium tests in the correct location with the correct base class | `gx-review-test-types` |
| Mocks and patches | DI, fakes, and real objects over patching; assert state, not calls | `py-challenge-patches` |
| UI components | `GButton`, `GLink`, `GModal`, `GCard` instead of Bootstrap-Vue | `gx-review-ui-components` |
| Vitest tests | Test behavior not implementation, isolate Pinia stores, every test seen failing once | `gx-vitest-review` |

## Layout

```
galaxy-code-standards/
├── SKILL.md                      # Rule selection table, rules per area, self-review checklist
├── README.md
└── references/
    ├── python-backend.md         # Layers, DI patterns, FastAPI, migrations, code structure
    ├── python-tests.md           # Test type decision tree, red flags, patch elimination
    └── client.md                 # G-component prop mappings, vitest practices
```

`SKILL.md` stays short enough to load in full. The reference files hold the good and bad
code examples and are read only when the matching area is in play.

## Installation

Copy or clone this directory into your personal skills folder:

```bash
git clone <this-repo> ~/.claude/skills/galaxy-code-standards
```

Or place it in a Galaxy checkout at `.claude/skills/galaxy-code-standards` to scope it to
that project.

To get the final review step as well, load the review plugin when starting Claude Code:

```bash
git clone https://github.com/jmchilton/gx-arch-review.git
claude --plugin-dir /path/to/gx-arch-review
```

## Usage

No explicit invocation is needed. The skill triggers whenever Claude writes, modifies, or
plans code in a Galaxy checkout. It can also be invoked directly with
`/galaxy-code-standards`.

## Keeping it current

The rules are derived from the command files in `gx-arch-review/commands/`. When that
repository changes, update the matching section of `SKILL.md` and the relevant reference
file. In particular, fill in the Models section once `gx-review-model.md` has content upstream.

## License

Same license as the Galaxy project.
