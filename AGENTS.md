## Project Overview

Open TutorAI CE is a standalone educational AI platform. The backend is a FastAPI application and the frontend is a SvelteKit app in `ui/`. The product supports chat-based tutoring, learner supports, classrooms, files/uploads, RAG/knowledge bases, model/provider management, realtime Socket.IO collaboration flows, audio/image media features, admin settings, users/groups, and governance/evaluation workflows.

This repository is the Community Edition foundation. OpenWebUI and Hermes may be useful as read-only design references, but Open TutorAI keeps its own domain names, route ownership, and runtime code. Never import from `open_webui`, copy external reference package names into runtime modules, or add new legacy OpenWebUI paths unless a compatibility exception is explicit.

## Stack

- Backend language/runtime: Python >=3.11,<3.13.
- Backend framework: FastAPI 0.115.7, Pydantic 2.10.6, Uvicorn 0.30.6.
- Backend persistence: sync SQLAlchemy 2.0.32, default SQLite, optional external DB via `DATABASE_URL`.
- Frontend: SvelteKit 2, Svelte 4.2, Vite 5, TypeScript 5.5.
- Package managers: pip/requirements for Python, npm for frontend, Rye metadata exists in `pyproject.toml` but local instructions use pip.
- Node engines: >=18.13.0 and <=22.x.x; CI uses Node 22.
- Realtime: `python-socketio` ASGI mounted at `/realtime`; legacy `/ws/socket.io` is intentionally rejected.
- AI/RAG stack includes OpenAI/Anthropic/Google clients, LangChain, Chroma, Milvus, Qdrant, OpenSearch, sentence-transformers, ColBERT, PyODide, and media/document parsers.
- Docker and Kubernetes assets live under `devops/`; Compose is the default full-stack dev/deploy path.

## Commands

- Install backend deps: `pip install -r requirements.txt`
- Install CI/minimal backend deps: `pip install -r requirements-ci.txt`
- Install frontend deps: `cd ui && npm install`
- Backend dev server: `uvicorn main:app --reload --port 8080` or `./devops/scripts/dev.sh`
- Backend console entrypoint after install: `open-tutorai`
- Frontend dev server: `cd ui && npm run dev`
- Frontend dev on fixed port: `cd ui && npm run dev:5050`
- Docker dev stack: `make install` or `docker compose -f devops/docker/docker-compose.yaml up --build`
- Backend tests: `pytest -q --tb=short`
- Single backend test file: `pytest tests/test_chats.py -q --tb=short`
- Contract test: `pytest tests/test_contract_coverage.py -q --tb=short`
- Backend format check: `black . --check --exclude ".venv/|/venv/|ui/"`
- Backend format: `black . --exclude ".venv/|/venv/|ui/"`
- Frontend tests: `cd ui && npm run test:frontend`
- Single frontend test pattern: `cd ui && npm run test -- -t "<name>"`
- Frontend typecheck: `cd ui && npm run check`
- Frontend lint/types: `cd ui && npm run lint`
- Frontend format: `cd ui && npm run format`
- Frontend i18n parse: `cd ui && npm run i18n:parse`
- Frontend build: `cd ui && npm run build`
- Full local check: `make check`

## Runtime Configuration

Settings are centralized in `config/settings.py` and loaded from `.env` via `python-dotenv` before settings are instantiated.

- `DEBUG`: enables development behavior and allows the dev JWT secret fallback. Production requires `SECRET_KEY`.
- `GLOBAL_LOG_LEVEL`: controls Uvicorn log level when running `main.py`.
- `SECRET_KEY`: JWT signing key; required when `DEBUG=false`.
- `DATABASE_URL`: defaults to `sqlite:///./var/tutorai.db`; non-SQLite URLs use SQLAlchemy defaults.
- `CORS_ALLOW_ORIGIN`: comma-separated origins; `*` is handled through CORS regex.
- `UPLOAD_DIR`: defaults to `./var/uploads`.
- `MAX_UPLOAD_SIZE_MB`: integer upload size limit, default `100`.
- `VECTOR_DB_PATH`: defaults to `./var/vector_db`.
- `EMBEDDING_MODEL`: defaults to `sentence-transformers/all-MiniLM-L6-v2`.
- `AUDIO_TTS_ENGINE`, `AUDIO_STT_ENGINE`, `IMAGES_ENGINE`: select media engines.
- `TUTORAI_BUILD_HASH`: optional build metadata printed on startup.
- `FRONTEND_BUILD_DIR`: defaults to `./ui/build` and is served by FastAPI when present.

Never commit `.env`, real secrets, API keys, uploaded user data, generated DB files, or vector stores.

## Architecture

- `main.py` imports `create_app()` from `gateway/http/app.py`, exposes `app`, and provides the `open-tutorai` console entrypoint.
- `gateway/http/app.py` owns app creation, lifespan startup, CORS, router registration, Socket.IO mounting, legacy realtime rejection, and static SPA serving.
- `gateway/http/api_routes.py` registers top-level `/api/*` bootstrap/compatibility routes.
- `gateway/http/routers/*.py` are HTTP adapters only: parse requests, depend on auth/services, translate domain exceptions, and shape responses.
- `gateway/http/dependencies.py` owns FastAPI dependencies such as current-user auth and service/session injection helpers.
- `gateway/realtime/socket.py` owns Socket.IO ASGI behavior and realtime auth/session state.
- Domain packages own business logic in `service.py` and persistence access in `repository.py`.
- `data/database.py` owns `Base`, SQLAlchemy engine/session setup, SQLite directory creation, `get_db()`, `init_database()`, and `close_database()`.
- `data/models/` contains ORM models; all models must be imported/registered through `data/models/__init__.py`.
- `data/repositories/base.py` provides generic sync CRUD using the injected SQLAlchemy session.
- Frontend API clients live in `ui/src/lib/apis/<domain>/index.ts`; routes live under `ui/src/routes/`; reusable UI lives under `ui/src/lib/components/`.
- `tests/test_contract_coverage.py` scans frontend `fetch()` calls and asserts that matching backend OpenAPI paths exist.

## Project Structure

- `accounts/`: auth, users, roles, permissions, first-user-admin behavior.
- `learning/`: learners, teachers, classrooms, courses, sessions/chats, personalized learner supports.
- `ai/`: LLM schemas/service/transports, providers, model catalog, retrieval/RAG, media, memory, tools.
- `content/`: user files, uploads, extracted content, and learning resources.
- `governance/`: human-in-the-loop evaluation and self-regulation feedback.
- `system/`: runtime configuration, app bootstrap, and app-info services.
- `gateway/`: HTTP and realtime transport layer.
- `data/`: database engine, ORM models, and repository base classes.
- `common/`: shared exceptions, logging, and cross-cutting helpers.
- `config/`: settings and constants.
- `ui/`: SvelteKit app, API clients, components, routes, stores, i18n, static assets, workers, and Cypress files.
- `tests/`: pytest suite for app API, auth, chats, configs, contracts, files, health, knowledge, media, models, providers, realtime, retrieval, users.
- `docs/`: project docs.
- `devops/`: Dockerfiles, Compose overlays, Kubernetes/Helm assets, and scripts.
- `.github/workflows/`: active CI and release workflows plus disabled historical/optional workflows.

## Domain Map

- `accounts/auth`: signin/signout/signup/session behavior and JWT auth contracts.
- `accounts/users`: user CRUD and admin/user ownership rules.
- `learning/sessions`: chat session persistence, sharing, tags, archive/pin/folder behavior.
- `learning/supports`: personalized tutoring supports for learners.
- `ai/providers`: OpenAI-compatible, Ollama, provider config, proxying, and profiles.
- `ai/llm`: LLM schemas, service orchestration, and transport abstractions.
- `ai/retrieval/knowledge`: knowledge base persistence and RAG-facing services.
- `ai/media`: audio and image generation/integration services.
- `content/files`: uploads, extracted content, file metadata, and ownership.
- `governance/self_regulation`: response feedback/evaluation/self-regulation data.
- `system/configs`: runtime configuration store exposed through API routes.
- `system/app`: application info/bootstrap behavior.

## Backend Coding Conventions

- Keep the boundary pattern strict: repository for data access, service for business rules, router for HTTP concerns.
- Routers should not run ORM queries, enforce domain ownership rules, or contain multi-step business workflows.
- Services should enforce authorization/ownership and orchestrate repositories.
- Repositories should not parse requests, read current users, shape HTTP responses, or decide permissions.
- Use sync SQLAlchemy sessions from `data.database.get_db()`. Do not introduce async ORM patterns.
- Use existing exception types from `common.exceptions` where possible, and convert to HTTP errors at the transport boundary.
- Register new ORM models in `data/models/__init__.py` or `Base.metadata.create_all()` will miss them.
- Register new routers in `gateway/http/app.py`; mount API routes before the SPA catch-all.
- Prefer `/api/v1/*` namespaces for new product APIs.
- Preserve explicit compatibility exceptions such as `/api/chat/completions`; do not add new legacy paths casually.
- Keep auth through JWT helpers and `get_current_user` in `gateway/http/dependencies.py`.
- Preserve first-user admin behavior when touching auth/signup.

## Frontend Coding Conventions

- Keep fetch clients in `ui/src/lib/apis/<domain>/index.ts`, not scattered across components.
- Use existing API constants from `ui/src/lib/constants.ts`; make backend prefix changes deliberately.
- Keep TypeScript request/response shapes aligned with backend route bodies and path/query params.
- Preserve i18n when changing user-visible text; CI runs `npm run i18n:parse` and checks for clean diffs.
- Prefer existing reusable components under `ui/src/lib/components/common`, `chat`, `admin`, `workspace`, and `student` before adding new UI primitives.
- When editing Svelte components, read `.agents/skills/svelte-core-bestpractices/SKILL.md` first.
- Pyodide/Kokoro workers and large browser-side models are sensitive to build size and runtime loading; avoid eager imports in UI entrypoints.

## Adding Or Changing A Domain

1. Choose the owning boundary before adding files: `accounts`, `learning`, `ai`, `content`, `governance`, or `system`.
2. Read the closest existing domain with the same shape and copy its layering, naming, and test style.
3. Add or update `<boundary>/<domain>/repository.py`, `service.py`, and `__init__.py` as needed.
4. Add or update `data/models/<domain>.py` and register the model in `data/models/__init__.py`.
5. Add or update `gateway/http/routers/<public_namespace>.py` and register it in `gateway/http/app.py` under the right prefix, usually `/api/v1`.
6. Add service dependency helpers in the router or shared dependencies, matching local patterns.
7. Add or update `ui/src/lib/apis/<domain>/index.ts` only after the backend route contract is clear.
8. Add focused tests in `tests/test_<domain>.py` covering success, auth/ownership, missing resource, and validation/error cases.
9. Run `pytest tests/test_contract_coverage.py -q --tb=short` if a UI API client or backend route path changed.
10. Remove entries from `_SCANNED_PATH_EXCLUSIONS` when previously deferred UI paths become implemented.

## UI/API Contract Workflow

- Every UI `fetch()` in `ui/src/lib/apis/**/*.ts` should map to a real FastAPI path in OpenAPI.
- `tests/test_contract_coverage.py` maps frontend base constants like `TUTOR_API_BASE_URL`, `RETRIEVAL_API_BASE_URL`, `AUDIO_API_BASE_URL`, and `IMAGES_API_BASE_URL` to backend prefixes.
- Prefer implementing the backend route or repointing the UI client over adding exclusions.
- If a path is intentionally absent, add a narrow `_SCANNED_PATH_EXCLUSIONS` entry with a comment explaining the product gap.
- Keep HTTP method, path params, query params, and JSON body names aligned between TypeScript and FastAPI.
- Watch for template param normalization: frontend `${chatId}` becomes FastAPI `{chat_id}` in the scanner.
- `FORBIDDEN_PATTERNS` protects against legacy paths and runtime `open_webui` references; do not weaken it without a clear compatibility reason.

## Testing And Debugging Workflow

- Reproduce failures with the smallest command first, then read the failing assertion or stack trace before editing.
- For API bugs, trace UI client -> router -> service -> repository/model.
- For auth bugs, check `gateway/http/dependencies.py`, JWT settings, and ownership checks in services.
- For persistence bugs, check model registration, `Base.metadata.create_all()`, and repository commit/refresh behavior.
- Backend behavior changes need focused pytest coverage for auth, ownership, success, and error paths.
- Frontend behavior changes use Vitest unless the existing pattern requires Cypress or browser-level coverage.
- Realtime bugs should inspect `gateway/realtime/socket.py` and client Socket.IO path configuration together.
- Do not silence exceptions, skip tests, weaken assertions, or broaden contract exclusions as a substitute for fixing behavior.
- State checks not run and why.

## Code Review Workflow

- Start from `git status` and the relevant diff.
- Review risk first: auth/authorization, persistence, API compatibility, contract coverage, error handling, and missing tests.
- For backend diffs, check layer boundaries, model registration, router registration, and service-owned ownership checks.
- For frontend diffs, check API paths, i18n, loading/error states, and whether existing components/utilities were reused.
- Findings should lead, ordered by severity, with exact file/line references.
- Do not report style-only nits unless they hide a real defect or maintainability risk.

## Skills

Reusable skills live in `.agents/skills/`. To use a skill, read its `SKILL.md` and follow the instructions inside.

Available skills:

- `fastapi` - Use when changing FastAPI routes, dependencies, or Pydantic models.
- `svelte-core-bestpractices` - Use when editing or reviewing Svelte components or SvelteKit modules.

To invoke: read `.agents/skills/<skill-name>/SKILL.md` and follow its instructions.

## CI/CD

- Backend CI: `.github/workflows/ci-backend.yaml` runs on Python, requirements, pyproject, and workflow changes for PRs/pushes to `main` and `dev`.
- Backend CI jobs: setup Python 3.11, install Black 24.8.0, run `black . --check --exclude ".venv/|/venv/|ui/"`, install `requirements-ci.txt`, run `pytest -q --tb=short`.
- Frontend CI: `.github/workflows/ci-frontend.yaml` runs on `ui/**` and workflow changes for PRs/pushes to `main` and `dev`.
- Frontend CI jobs: setup Node 22, `npm ci`, `npm run format -- --check`, `npm run i18n:parse`, `git diff --exit-code`, `npm run build`, and `npm run test:frontend`.
- Release CI: `.github/workflows/build-release.yml` runs on `v*` tags and builds GitHub releases from changelog content.
- Disabled workflow files exist for historical or optional checks; do not assume they run in CI.
- CI uses `requirements-ci.txt` for backend speed and stability, not the full `requirements.txt`.

## DevOps

- `Makefile` wraps Docker Compose with `install`, `start`, `startAndBuild`, `stop`, `update`, `lint`, `test`, and `check`.
- Main Compose file: `devops/docker/docker-compose.yaml`.
- Compose overlays include GPU, AMD GPU, API, data, Playwright, and A1111 test variants.
- Dockerfiles: `devops/docker/Dockerfile.backend` and `devops/docker/Dockerfile.frontend`.
- Local backend helper: `devops/scripts/dev.sh` loads `.env` and runs Uvicorn with reload.
- Docker helpers in `devops/scripts/` cover single-container runs, Compose runs, Ollama-in-Docker, and Ollama model updates.
- Kubernetes/Helm assets live under `devops/kubernetes/helm/`; do not change deployment defaults without checking Docker and local-dev assumptions.
- FastAPI serves the built frontend from `FRONTEND_BUILD_DIR` only if that directory exists; API routes must be registered before the SPA catch-all.

## Permissions

### Allowed without prompting

Read files, inspect git status/diffs, run format checks, run linters on touched files, and run focused unit tests.

### Ask before doing

Install packages, download browsers or models, run Docker Compose services, change schemas or persisted data, edit `.env`, commit, push, or open a PR.

### Never

Commit secrets or `.env` files, force-push `main` or `dev`, import runtime code from OpenWebUI/Hermes, hard-delete user records without explicit approval, or remove security/auth checks to satisfy tests.

## PR Standards

- Target `dev` unless maintainers say otherwise.
- Use PR title prefixes from `.github/pull_request_template.md`: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `ci`, `build`, `style`, `i18n`, `BREAKING CHANGE`, or `WIP`.
- Keep PRs focused by domain/workflow and include a changelog-style summary in the PR body.
- Before PR handoff, run the narrowest relevant checks and state any checks not run.
- If architecture, tooling, conventions, or domain rules changed, update `AGENTS.md`.
- If a new repeatable workflow genuinely needs more detail than belongs here, add a skill under `.agents/skills/` and import it in `CLAUDE.md`.
