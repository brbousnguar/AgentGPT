# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

AgentGPT is a web app for deploying autonomous AI agents in-browser. Users set a goal; the agent breaks it into tasks, executes them with tools (web search, code, images, Wikipedia), and iterates. The repo is a monorepo with three services:

- `next/` — Next.js 13 frontend (TypeScript, T3 stack)
- `platform/` — FastAPI backend (Python 3.11+)
- `cli/` — Setup CLI (Node.js)
- `db/` — MySQL database config

## Commands

### Frontend (`next/`)

```bash
npm install          # install deps (also runs prisma generate)
npm run dev          # start dev server at http://localhost:3000
npm run build        # production build
npm run lint         # ESLint with auto-fix
npm test             # Jest tests
npm test -- --testPathPattern=<file>  # run a single test file
npx prisma migrate dev    # apply DB migrations
npx prisma generate       # regenerate Prisma client after schema changes
```

### Backend (`platform/`)

```bash
poetry install                          # install deps
poetry run python -m reworkd_platform   # start FastAPI at http://localhost:8000
poetry run pytest                       # run all tests
poetry run pytest tests/path/test.py    # run a single test
poetry run black .                      # format
poetry run isort .                      # sort imports
poetry run mypy .                       # type check
```

### Full stack

```bash
./setup.sh              # automated setup (copies .env, starts Docker)
docker-compose up       # start all services (frontend, platform, MySQL)
```

## Architecture

### Request flow

The frontend `AutonomousAgent` class (`next/src/services/agent/autonomous-agent.ts`) drives the agent loop. It maintains a work queue of `AgentWork` subclasses (`start-task-work`, `analyze-task-work`, `execute-task-work`, `create-task-work`, `summarize-work`). Each work item calls the FastAPI backend via `AgentApi` (`next/src/services/agent/agent-api.ts`), which posts to `NEXT_PUBLIC_BACKEND_URL` (default `http://localhost:8000`).

The backend exposes agent endpoints under `/api/agent/` (`platform/reworkd_platform/web/api/agent/views.py`):
- `POST /start` — generate initial task list for a goal
- `POST /analyze` — choose which tool to use for a task
- `POST /execute` — run the task (streaming response)
- `POST /create` — generate next tasks from results
- `POST /summarize` — summarize results (streaming)
- `POST /chat` — free-form chat (streaming)

### Frontend state management

Zustand stores in `next/src/stores/`: `agentStore` (agent instance + lifecycle), `messageStore` (chat messages), `taskStore`, `modelSettingsStore`, `configStore`. Stores are composed from slices and some are persisted to localStorage.

### tRPC layer

The Next.js server exposes a tRPC API (`next/src/server/api/`) with a single `agentRouter` that handles persistence: creating agents, saving task logs, listing/fetching/deleting agents. This is separate from the FastAPI backend — tRPC handles DB persistence while FastAPI handles AI execution.

### Backend agent service

`AgentService` (`platform/reworkd_platform/web/api/agent/agent_service/agent_service.py`) is a Protocol with two implementations:
- `OpenAIAgentService` — real LLM calls via Langchain + OpenAI
- `MockAgentService` — fake responses for development (enabled via `REWORKD_PLATFORM_FF_MOCK_MODE_ENABLED=true`)

Tools (`platform/reworkd_platform/web/api/agent/tools/`): `search` (Serper API), `reason`, `code`, `image` (Replicate), `conclude`, `wikipedia_search`, `sidsearch`.

### Environment variables

All backend env vars are prefixed with `REWORKD_PLATFORM_` (see `platform/reworkd_platform/constants.py`). Frontend env vars are validated at build time via Zod in `next/src/env/schema.mjs`. Copy `.env.example` to `next/.env` to get started — the frontend `.env` is shared with the backend via Docker Compose.

Key vars:
- `REWORKD_PLATFORM_OPENAI_API_KEY` / `OPENAI_API_KEY` — required for AI functionality
- `DATABASE_URL` — MySQL connection string for Prisma (frontend)
- `NEXT_PUBLIC_BACKEND_URL` — FastAPI URL the browser calls directly
- `REWORKD_PLATFORM_FF_MOCK_MODE_ENABLED` — bypass OpenAI calls in dev

### Database

MySQL is used by both services. Prisma (`next/prisma/schema.prisma`) manages the frontend schema (User, Agent, AgentTask, Run, etc.). SQLAlchemy with aiomysql is used by the FastAPI backend. Both point to the same MySQL instance.
