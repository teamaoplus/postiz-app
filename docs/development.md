# Development setup

How to run Postiz locally from this repository. For production self-hosting with the prebuilt image, use `docker-compose.yaml` and the [hosted docs](https://docs.postiz.com).

## Repository layout

| Path | What it is |
| --- | --- |
| `apps/backend` | NestJS API (port 3000) |
| `apps/orchestrator` | NestJS Temporal worker: background workflows and activities (publishing, refresh, clipping, ...) |
| `apps/frontend` | Web app (port 4200) |
| `apps/extension` | Browser extension for cookie-based providers |
| `apps/commands` | One-off CLI commands |
| `libraries/nestjs-libraries` | Shared server code: Prisma schema and repositories, services, DTOs, providers, AI tools |
| `libraries/helpers`, `libraries/react-shared-libraries` | Shared utilities and React components |

## Prerequisites

- Node.js `>=22.12.0 <23` (see `engines` in `package.json`)
- pnpm `10.6.1`. Use only pnpm.
- Docker, for Postgres, Redis and Temporal

## 1. Start the infrastructure

```bash
pnpm run dev:docker
```

This runs `docker-compose.dev.yaml`, which starts:

| Service | Port |
| --- | --- |
| Postgres 17 | 5432 |
| Redis | 6379 |
| Temporal | 7233 |
| Temporal UI | 8080 |
| pgAdmin, RedisInsight | see `docker-compose.dev.yaml` |

## 2. Configure the environment

```bash
cp .env.example .env
```

All apps read the single root `.env` (each app's scripts run `dotenv -e ../../.env`). The minimum to boot:

| Variable | Value for local dev |
| --- | --- |
| `DATABASE_URL` | `postgresql://postiz-user:postiz-password@localhost:5432/postiz-db-local` |
| `REDIS_URL` | `redis://localhost:6379` |
| `JWT_SECRET` | any long random string |
| `FRONTEND_URL` | `http://localhost:4200` |
| `NEXT_PUBLIC_BACKEND_URL` | `http://localhost:3000` |
| `BACKEND_INTERNAL_URL` | `http://localhost:3000` |
| `STORAGE_PROVIDER` | `local` (plus `UPLOAD_DIRECTORY` and `NEXT_PUBLIC_UPLOAD_STATIC_DIRECTORY`), or `cloudflare` with the `CLOUDFLARE_*` variables |

Optional variables:

- `RESEND_API_KEY`: when set, new users must activate their account by email. When unset, they're activated automatically.
- `DISABLE_REGISTRATION`: blocks new sign-ups.
- Provider credentials (`X_API_KEY`, `LINKEDIN_CLIENT_ID`, ...): each provider only works once its keys are set.
- `OPENAI_API_KEY`: AI features (the agent, post generation, clipping).
- `API_LIMIT`: public API post-creation limit per organization per hour.
- Stripe keys: leave them unset to run without billing. Every account then gets full access.
- Clipping: see [Video clipping](./clipping.md#enabling-clipping-self-hosted).

`.env.example` documents every variable.

## 3. Install and prepare the database

```bash
pnpm install          # also runs prisma generate (postinstall)
pnpm run prisma-db-push
```

The Prisma schema is at `libraries/nestjs-libraries/src/database/prisma/schema.prisma`. Re-run `prisma-db-push` after changing it.

## 4. Run the apps

```bash
pnpm run dev          # extension, orchestrator, backend and frontend in parallel
```

Or run them one at a time:

```bash
pnpm run dev:backend
pnpm run dev:orchestrator
pnpm run dev:frontend
```

`pnpm run dev-backend` starts only the backend and frontend. Use it when you don't need background jobs. Nothing gets published without the orchestrator.

Open http://localhost:4200 and register. The first user creates the first organization.

### Useful extras

- `pnpm run dev:stripe` forwards Stripe webhooks to `localhost:3000/stripe` while running dev. It needs the Stripe CLI.
- Temporal UI at http://localhost:8080 shows running workflows, which helps with debugging publishing and clipping.
- The backend serves Swagger/OpenAPI docs at http://localhost:3000/docs.

## Build, lint and test

```bash
pnpm run build        # frontend, backend, orchestrator
pnpm test             # jest
```

Run linting from the repository root only.

## Conventions

Read `CLAUDE.md` and `CONTRIBUTING.md` before opening a PR. In short:

- Backend changes go through DTO → Controller → (Manager →) Service → Repository. Most logic lives in `libraries/nestjs-libraries`.
- Use Prisma only. No raw SQL.
- Keep provider-specific logic inside the provider, behind the provider interface.
- Never edit a Temporal workflow or activity signature that is already on `main`. Add a new versioned workflow or activity instead.
- Frontend data fetching uses SWR through `useFetch`, with one hook per SWR call.
- PR descriptions follow `.github/PULL_REQUEST_TEMPLATE.md` and include a `# QA` section.
