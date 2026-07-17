# JoyAgent Genie Development Guide

## Before Editing

1. Read `AGENTS.md`, `.agents/context.md`, and `.agents/constraints.md`.
2. Run `git status --short` and preserve unrelated changes and local artifacts.
3. Trace the request through all affected services. A UI streaming change may also require backend event and Python tool changes.
4. Inspect the nearest existing implementation and follow its naming, package, response, and logging conventions.
5. Define a focused verification command before changing non-trivial behavior.

## Change Discipline

- Make the smallest coherent change that satisfies the request.
- Keep service ownership intact. Do not move behavior between Java and Python merely for convenience.
- Prefer existing models, helpers, handlers, and components over new abstractions.
- Add an abstraction only for a required boundary or demonstrated repeated behavior.
- Preserve public routes, JSON fields, SSE event ordering/types, and environment names unless the task explicitly changes the contract.
- When a contract must change, update every producer and consumer in the same change, plus the relevant README or environment template.
- Do not reformat or refactor unrelated files.

## Backend Development

- Use Java 17 and the existing Spring Boot/Maven structure.
- Keep HTTP concerns in controllers, orchestration in services/handlers, agent lifecycle in `agent/agent`, and persistence/query concerns in `data`, mappers, or their owning service.
- Put request and response shapes in the existing `model/req`, `model/response`, `model/dto`, or Data Agent DTO packages as appropriate.
- Validate external input at controller/service boundaries. Keep LLM and MCP failures distinct from validation and internal failures.
- Preserve streaming endpoints' `text/event-stream` behavior and the UI's expected event schema.
- Use existing JDBC providers/dialects and parameterized paths. Keep database-specific behavior inside the dialect/catalog boundary.
- Keep prompts and model configuration out of controllers and avoid duplicating configured model names or URLs in code.

Useful commands from `genie-backend/`:

```bash
mvn -s aliyun-settings.xml -DskipTests package
mvn -s aliyun-settings.xml test
mvn -s aliyun-settings.xml -Dtest=GenieTest test
sh build.sh
sh start.sh
tail -f genie-backend_startup.log
```

`GenieTest` loads the Spring context and may call an external MCP server from local configuration. Run it only when the needed local configuration and dependency are available. `sh build.sh` skips tests.

## Tool Service Development

- Use the locked `uv` environment; do not install project dependencies globally.
- Keep route registration in `genie_tool/api`, schemas in `genie_tool/model`, implementations in `genie_tool/tool`, persistence in `genie_tool/db`, and shared mechanics in `genie_tool/util`.
- Keep tool interfaces serializable and compatible with the Java adapters that call them.
- Treat generated files, fetched web content, model-produced code, SQL, and paths as untrusted. Validate at the tool boundary.
- Avoid importing optional/heavy tool modules at application import time when a local import preserves startup isolation.

Useful commands from `genie-tool/`:

```bash
uv sync
cp .env_template .env          # first setup only; then edit the ignored file locally
uv run python -m genie_tool.db.db_engine
uv run python server.py
uv run python -m compileall genie_tool server.py
```

Database initialization creates local SQLite state. Back it up first if a task could alter valuable local data.

## MCP Client Development

- Keep the FastAPI transport layer in `server.py` and SSE/header/logging behavior in `app/`.
- Preserve the current response envelope (`code`, `message`, `data`) unless all callers are updated together.
- Preserve request headers only when required; never log credentials, cookies, or authorization values.
- Validate or constrain remote `server_url` usage when changing this boundary; it is a network access surface.

Useful commands from `genie-client/`:

```bash
uv sync
uv run python server.py
uv run python -m compileall app server.py
curl http://localhost:8188/health
```

## Frontend Development

- Use pnpm and preserve `ui/pnpm-lock.yaml` when dependencies change.
- Keep pages/layout at the route level, reusable rendering in `components`, data access in `services`/`utils`, and shared TypeScript shapes in `types`.
- Reuse the existing Ant Design, Tailwind, icons, and component patterns before adding another UI library.
- Preserve incremental SSE parsing, cancellation, error, and completion handling when editing chat flows.
- Do not expose secrets through `SERVICE_BASE_URL`, other Vite definitions, browser storage, logs, or rendered error messages.
- Follow the checked-in ESLint configuration and strict TypeScript settings.

Useful commands from `ui/`:

```bash
pnpm install
pnpm dev
pnpm lint
pnpm build
pnpm preview
```

There is no frontend test script. For behavior changes, combine lint/build with a focused browser check and state exactly what was exercised.

## Running the System

- Prerequisites checked by the repository: Java 17+, Maven, Node 18+, pnpm 7+, Python/`uv`, and free ports 3000, 8080, 1601, and 8188.
- Run `sh check_dep_port.sh` from the repository root for a read-only dependency/port check.
- Prepare the ignored backend `application.yml` and tool `.env` before startup; never fill placeholders with committed credentials.
- `sh Genie_start.sh` builds/initializes and starts the local services. Read it before use because initialization can create or change local state.
- For focused development, start only the affected services with their module commands.
- Validate health with an HTTP request and logs. A process existing or a port listening is not proof that dependencies or agent workflows work.

## Verification Strategy

- Documentation-only change: verify links, paths, commands, and statements against tracked files.
- UI-only change: `pnpm lint`, `pnpm build`, then a focused browser interaction if feasible.
- Java change: compile/package plus the smallest relevant Maven test; note external configuration gaps.
- Python change: compile the affected module and run a focused route/import/runtime check; add tests when introducing logic that can be isolated.
- API/SSE change: verify the producer and every consumer, including failure and stream-completion paths.
- Data Agent change: verify SQL/schema handling, empty/error cases, datasource-specific behavior, and Java/Python result compatibility.
- Tool or file change: test allowed paths/types/sizes and rejection behavior, not only the happy path.
- Cross-service change: start with module checks, then run a focused end-to-end request if configured external dependencies are available.

Always report the exact commands run, their outcomes, and any checks skipped because local configuration or external services were unavailable.

