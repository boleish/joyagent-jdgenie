# JoyAgent Genie Project Context

## Product and Architecture

JoyAgent Genie is a multi-agent application for research, analysis, report generation, and data-agent workflows. It is implemented as four cooperating services:

```text
React UI (:3000)
  -> Spring Boot orchestration and data backend (:8080, including SSE)
       -> Python tool service (:1601)
       -> Python MCP client service (:8188)
       -> configured LLM, search, MCP, JDBC, Qdrant, and Elasticsearch services
```

- The UI starts conversations and renders streamed plans, tool activity, results, files, and reports.
- The backend owns agent orchestration, LLM settings, prompts, tool dispatch, SSE responses, and Data Agent APIs.
- The tool service owns code interpretation, file processing, deep search, report generation, analysis, and local tool persistence.
- The MCP client adapts remote MCP servers to the backend's `/v1/tool/list` and `/v1/tool/call` HTTP contract.
- Data Agent code spans backend SQL/schema/vector retrieval services and Python analysis/table-RAG tools. Changes there often cross service boundaries.

## Repository Layout

- `ui/`: React 19, TypeScript, Vite, Ant Design, Tailwind CSS, ECharts, and SSE client code.
- `genie-backend/`: Java 17, Spring Boot 3.2, Maven, agent orchestration, Data Agent services, JDBC, Qdrant, Elasticsearch, and SSE controllers.
- `genie-tool/`: Python 3.11+ FastAPI service and tool implementations, managed with `uv`.
- `genie-client/`: Python 3.10-3.13 FastAPI MCP client, managed with `uv`.
- `docs/`: architecture, product screenshots, and MRAG documentation.
- `README.md`, `README_EN.md`, `README_DataAgent.md`, `README_mrag.md`, and `Deploy.md`: user and deployment guidance.
- `Dockerfile`: multi-stage build that packages all four services.
- `Genie_start.sh`: local dependency/configuration check, build, initialization, and service launcher.
- `start_genie.sh`: container entrypoint; its runtime directory names are prepared by the Docker image and differ from source directory names.
- `check_dep_port.sh`: checks Java, Maven, Node, pnpm, and the standard local ports.

## Important Code Boundaries

### Backend

- `controller/`: HTTP and SSE entrypoints (`GenieController`, `DataAgentController`).
- `service/` and `service/impl/`: orchestration and Data Agent application behavior.
- `agent/agent/`: planning, execution, ReAct, and summary agent lifecycles.
- `agent/tool/`: Java-side tool adapters and MCP integration.
- `agent/llm/` and `agent/prompt/`: model configuration, token handling, and prompts.
- `handler/`: response handling for agent modes.
- `data/`: SQL parsing, JDBC providers/catalogs, query models, and dialects.
- `config/`: application and Data Agent configuration.

### Tool Service

- `genie_tool/api/`: `/v1` FastAPI routes and request handling.
- `genie_tool/tool/`: code, file, search, report, analysis, and table-RAG tools.
- `genie_tool/model/`: tool and API models.
- `genie_tool/db/`: SQLModel/SQLite persistence.
- `genie_tool/prompt/`: Python-side prompts.
- `genie_tool/util/`: middleware and shared utilities.

### UI and MCP Client

- `ui/src/pages/`, `components/`, and `layout/`: screens and presentation components.
- `ui/src/services/` and `utils/`: backend requests and SSE parsing.
- `ui/src/types/`: shared frontend types; keep streamed event types synchronized with backend responses.
- `ui/src/router/`: route configuration.
- `genie-client/server.py`: MCP HTTP endpoints; `genie-client/app/` contains SSE client, header forwarding, configuration, and logging.

## Runtime and Configuration

- UI: `http://localhost:3000`; `ui/.env` currently sets `SERVICE_BASE_URL=http://127.0.0.1:8080`.
- Backend: `http://localhost:8080`; health/SSE routes include `/web/health` and `/web/api/v1/gpt/queryAgentStreamIncr`.
- Tool service: `http://localhost:1601`; routes are mounted below `/v1`.
- MCP client: `http://localhost:8188`; health is `/health` and API routes are below `/v1`.
- The backend requires `genie-backend/src/main/resources/application.yml`. It is intentionally gitignored and is not present in a fresh checkout; it must remain local.
- `genie-tool/.env_template` is the checked-in Python environment contract. Copy it to the ignored `genie-tool/.env` and replace placeholders locally.
- `ui/.env` is tracked. Do not put secrets in Vite variables because frontend values are shipped to browsers.
- The tool service creates ignored/local state such as `.venv/`, `logs/`, `file_db_dir/`, and `autobots.db`.
- Backend build output and logs live under `genie-backend/target/` and `genie-backend/genie-backend_startup.log`.

## Implemented Interfaces

- Main backend agent endpoints are rooted at `/`, including `/AutoAgent` and streaming `/web/...` endpoints.
- Data Agent endpoints are rooted at `/data`, including model metadata, retrieval, chat, NL2SQL, and preview operations.
- The UI uses `SERVICE_BASE_URL` both for direct API requests and SSE requests; Vite proxies `/web` in development.
- Tool service APIs use a `/v1` router and return tool/file/analysis results consumed by the backend.
- MCP client endpoints are `GET /health`, `POST /v1/serv/pong`, `POST /v1/tool/list`, and `POST /v1/tool/call`.

## Tests and Current Quality Baseline

- The backend has one tracked Spring Boot test, `GenieTest`, which can contact a configured MCP server. It is not a hermetic unit test.
- No tracked automated tests currently exist for `ui/`, `genie-tool/`, or `genie-client/`.
- UI lint and production build are available. Both Python projects have locked `uv` environments but no configured lint or test scripts.
- `genie-backend/build.sh` runs `mvn clean package -DskipTests` and rewrites `aliyun-settings.xml`; passing that build does not prove tests pass.
- Runtime checks depend on local secrets and external services. Do not claim end-to-end verification when only compilation or process startup was checked.

## Sources of Truth

Resolve conflicts in this order:

1. The user's current request and any explicitly named specification define the requested scope.
2. Current source, manifests, lockfiles, and request/response models define implemented behavior.
3. Local configuration defines the developer's current runtime only; never publish its secrets.
4. The module READMEs and `Deploy.md` describe supported workflows but may contain older examples.
5. Screenshots and diagrams in `docs/` are explanatory references, not API contracts.

If a requested change conflicts with an existing cross-service contract or the expected behavior is unclear, identify the conflict and ask before silently choosing a new contract.

