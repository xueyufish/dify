# AGENTS.md

## Monorepo Structure

```
api/          Python Flask backend (Domain-Driven Design)
web/          Next.js 16 frontend (React, TypeScript)
e2e/          Cucumber + Playwright end-to-end tests
packages/     Shared workspace packages
  contracts/    OpenAPI-generated API client types
  dify-ui/      Overlay primitives and design tokens
  dev-proxy/    Local dev proxy
sdks/         Client SDKs (Node.js, PHP)
docker/       Docker Compose deployment
```

All commands below run from the repo root unless noted.

## Backend (api/)

- **Python 3.12** (exact), managed with **uv** (`uv run --project api <cmd>`)
- Detailed coding standards, architecture rules, and SQLAlchemy patterns: **`api/AGENTS.md`**

### Commands

```bash
make format                       # ruff format
make lint                         # ruff format + fix + import lint + dotenv-lint
make type-check                   # basedpyright + pyrefly + mypy
make type-check-core              # basedpyright + mypy (no pyrefly)
make test                         # all unit tests (parallel via pytest-xdist)
make test TARGET_TESTS=./api/tests/unit_tests/core/tools/tool_engine.py  # single file
```

- Integration tests are **CI-only**; do not attempt to run them locally.
- `pytest.ini` injects mock env vars — no real API keys needed for unit tests.

## Frontend (web/)

- **Node 22**, **pnpm 11**, **Next.js 16** with vinext/vite-plus
- Detailed workflow, overlay rules, query patterns: **`web/AGENTS.md`**

### Commands

```bash
pnpm install                      # install all workspace deps from root
pnpm dev                          # dev server (vinext + proxy)
pnpm type-check                   # tsgo (TypeScript native preview)
pnpm lint:fix                     # ESLint auto-fix across workspace
pnpm lint:quiet                   # ESLint, quiet mode
cd web && pnpm test               # vitest (happy-dom)
cd web && pnpm test:coverage      # vitest with v8 coverage
cd web && pnpm test:watch         # vitest watch mode
```

- Frontend type-check uses `tsgo` (not `tsc`). Run from repo root: `pnpm type-check`.
- Frontend lint uses `vp run lint` (vite-plus runner). Run from repo root.
- Dead code detection: `cd web && pnpm knip`.

## E2E (e2e/)

- **Cucumber + Playwright**. Full lifecycle docs: **`e2e/AGENTS.md`**
- Requires Docker for middleware stack (PostgreSQL, Redis, Weaviate, etc.)

```bash
pnpm -C e2e check                 # format + lint + type-check for e2e
pnpm -C e2e e2e                   # run against already-running stack
pnpm -C e2e e2e:full              # full reset + fresh install + all scenarios
```

## Verification Order

Before submitting any change:

1. **Backend**: `make lint` → `make type-check` → `make test`
2. **Frontend**: `pnpm lint:fix` → `pnpm type-check` → `cd web && pnpm test`

## Key Conventions

- **Backend config**: use `configs.dify_config` — never read env vars directly.
- **Backend logging**: `logger = logging.getLogger(__name__)` — never `print`.
- **Backend async work**: Celery with Redis broker; tasks live in `api/tasks/`.
- **Frontend i18n**: all user-facing strings must go through `web/i18n/en-US/`.
- **Frontend overlays**: use only `@langgenius/dify-ui/*` primitives (see `web/AGENTS.md`).
- **Frontend queries/mutations**: follow `frontend-query-mutation` skill patterns.
- **Tenant awareness**: `tenant_id` must flow through every backend layer touching shared resources.
- **No comments** in code unless explicitly requested.

## CI Gates (`.github/workflows/`)

- `style.yml`: ruff + import lint + type-check + dotenv-lint (backend); ESLint + tsslint + type-check + knip (frontend)
- `api-tests.yml`: backend unit tests
- `web-tests.yml`: frontend vitest
- `web-e2e.yml`: E2E suite

## Workspace Packages (pnpm-workspace.yaml)

All dependencies use `catalog:` protocol — versions are pinned in `pnpm-workspace.yaml`, not in individual `package.json` files. Do not add version numbers directly to workspace package deps.
