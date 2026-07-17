# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Python/FastAPI backend service for the RPA Guidance PoC ("evals runtime"), built from DEFRA's CDP python backend template and deployed to the Core Delivery Platform (CDP). It runs on port 8285. The sibling orchestrator repo two levels up (`../../`) wires this service together with the other PoC services for local development; this repo also runs standalone via its own `compose.yml`.

## Commands

```bash
uv sync                      # install deps (dev group included by default)
uv run task format           # ruff format + ruff check --fix
uv run task lint             # ruff format --check + ruff check (no fixes)
uv run task typecheck        # mypy over app/ and tests/
uv run task test             # lint + typecheck + pytest with coverage reports
uv run pytest                # tests only
uv run pytest tests/health/test_health_router.py::TestHealthProbe::test_health  # one test
pre-commit install           # git hooks (ruff + hygiene checks)
```

Running locally:

```bash
docker compose up --build                    # dependencies ONLY (floci, mongodb, redis)
docker compose --profile service up --build  # dependencies + this service
uv run --env-file .env ai-uc-rpa-guidance-evals-runtime  # service on host, deps in docker
```

The app service sits behind the `service` compose profile, so plain `docker compose up` deliberately starts only the dependencies (the README doesn't mention this). Copy `.env.example` to `.env` first — `AppConfig` has required fields and refuses to start without them. `compose.debug.yaml` is an overlay that runs the app under debugpy (port 5679, waits for client).

## Gotchas

- **Two pytest configs exist and `pytest.ini` wins** (pytest warns it ignores `pyproject.toml`'s `[tool.pytest.ini_options]`). Practical consequence: pytest-asyncio runs in `strict` mode, not the `auto` mode declared in pyproject, and coverage is always on via `addopts`.
- `tests/conftest.py` seeds required environment variables with `os.environ.setdefault` *before* any app import. `AppConfig` is a validate-on-construct singleton (`app.config.get_config()`), so tests that need different config must mock `get_config`, not mutate env vars after import.
- `boto3` is not a declared dependency — it arrives transitively via `pymongo[aws]`, yet `app/common/s3.py` imports it directly. Declare it in `pyproject.toml` if AWS SDK usage grows.
- Docker images and the `defradigital/python*` base image versions are pinned; ruff's pre-commit rev must match the ruff version in `uv.lock` (comment in `.pre-commit-config.yaml`).
- mypy runs with `disallow_untyped_defs` on `app/` (tests are exempt via override).

## Current state (as of the template-application refactor)

The working tree is mid-refactor and two modules that the rest of the repo already references do not exist yet — test collection fails until they are written:

- `app/entrypoints/fastapi.py` — must expose an `app` FastAPI instance (see `tests/health/test_health_router.py`) and a `main()` function (the `[project.scripts]` entry and the Dockerfile `CMD` both invoke the `ai-uc-rpa-guidance-evals-runtime` console script). It replaces the deleted `app/main.py`.
- `app/infra/bedrock/llm.py` — must expose a module-level `claude_sonnet` with a `settings` dict, built from `get_config().bedrock.claude_sonnet` including optional guardrail config (see `tests/infra/bedrock/test_llm.py` for the expected shape).

`app/publishing/router.py` also imports a nonexistent `config` name from `app.config` (should be `get_config`).

## Architecture

- **Config** (`app/config.py`): pydantic-settings `AppConfig` read from environment/`.env`, cached in a module-level singleton behind `get_config()`. `CLAUDE_SONNET_MODEL_CONFIG` is a packed string — `model_id,inference_profile[,guardrail_id:guardrail_version]` — parsed by a field validator into `BedrockModelConfig`; a computed `bedrock` field exposes it as structured config. LLM access is AWS Bedrock via inference-profile ARNs with optional guardrails.
- **Routers**: one package per feature (`app/health/`, `app/publishing/`), each exporting an `APIRouter` that the entrypoint mounts. `GET /health` is required by CDP — do not remove it.
- **Shared plumbing** (`app/common/`): lazily-initialised global singletons exposed as FastAPI dependencies — `mongo.py` (async PyMongo client, custom CA from truststore env vars), `s3.py` (boto3 client pointed at `FLOCI_ENDPOINT_URL` locally), `http_client.py`, `tracing.py` (propagates the `x-cdp-request-id` header), `metrics.py` (aws-embedded-metrics, gated by `ENABLE_METRICS`), `tls.py` (loads `TRUSTSTORE_*` env vars as CA certs, mirroring CDP's node convention).
- **Local AWS** is `floci` (LocalStack-style emulator) on port 4566 with dummy credentials; `compose/floci/start.d/10-setup-resources.sh` creates the app's S3 bucket at container start. Add new local buckets/queues there. `compose/mongo/10-init.js` seeds MongoDB (currently a placeholder).
- **Tests** (`tests/`) mirror the `app/` package layout. HTTP-level tests use Starlette's `TestClient` against the app; there is no running-dependency requirement for the unit suite.
- **Docker**: multi-stage `Dockerfile` with `development` and `production` targets on CDP's `defradigital/python*` base images. `LOG_CONFIG` selects the logging setup (`logging-dev.json` in development, `logging.json` — ECS-formatted — in production). In CDP, config and secrets come from CDP conventions, not `.env`.
