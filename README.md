# Nodebase

A browser-based visual AI workflow builder. Compose multi-step LLM pipelines on a React Flow canvas, execute them reliably via Inngest's durable job engine, and manage encrypted AI credentials — all in a single Next.js 15 application.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Tech Stack](#tech-stack)
4. [Prerequisites](#prerequisites)
5. [Quick Start — Docker](#quick-start--docker)
6. [Quick Start — Local Dev](#quick-start--local-dev)
7. [Environment Variables](#environment-variables)
8. [Database](#database)
9. [External Services Setup](#external-services-setup)
10. [Development Scripts](#development-scripts)
11. [Production Deployment](#production-deployment)
12. [Project Structure](#project-structure)

---

## Overview

Nodebase lets users build directed-acyclic-graph (DAG) pipelines visually and execute them against real LLM providers. A workflow consists of **trigger nodes** (what starts execution), **AI nodes** (what does the work), and **action nodes** (what happens with the output).

**Supported node types**

| Category | Nodes |
|---|---|
| Triggers | Manual, HTTP Request, Google Form, Stripe Webhook |
| AI Models | Anthropic Claude, OpenAI GPT, Google Gemini |
| Actions | Discord, Slack |

Key design decisions:

- **Inngest** handles all async execution — retries, fan-out, and realtime status updates are free without managing queues.
- **Better Auth** owns the session layer so the app is not coupled to a specific cloud provider's auth.
- **Cryptr (AES-256-GCM)** encrypts API credentials at rest in Postgres — the encryption key never leaves the server.
- **Polar** manages subscription billing without building a checkout flow.

---

## Architecture

```
                        ┌───────────────────────────────────────────────────┐
                        │                    Browser                         │
                        │  React 19 · XYFlow (canvas) · Jotai · tRPC hooks  │
                        └────────────────────┬──────────────────────────────┘
                                             │ HTTPS  (tRPC / RSC)
                        ┌────────────────────▼──────────────────────────────┐
                        │              Next.js 15  (App Router)              │
                        │                                                    │
                        │  /api/trpc/[trpc]   — tRPC procedure router        │
                        │  /api/auth/[...all] — Better Auth handler          │
                        │  /api/inngest       — Inngest event receiver        │
                        │  /api/webhooks/*    — HTTP / Stripe / Google Forms  │
                        └───────┬───────────────────────┬────────────────────┘
                                │                       │
               ┌────────────────▼───┐       ┌──────────▼──────────────┐
               │   PostgreSQL 16    │       │   Inngest Cloud / Dev   │
               │   (via Prisma 6)   │       │   executeWorkflow fn     │
               │                   │       │   topological sort       │
               │  user             │       │   per-node executor      │
               │  session          │       └─────────────────────────┘
               │  account          │
               │  workflow         │       ┌─────────────────────────┐
               │  node             │       │   External Providers     │
               │  connection       │       │   Anthropic / OpenAI /   │
               │  credential       │       │   Gemini / Discord /     │
               │  execution        │       │   Slack / Stripe / etc.  │
               └───────────────────┘       └─────────────────────────┘
```

**Execution flow:**

1. User saves a workflow (nodes + connections persisted to Postgres via tRPC).
2. A trigger fires — manual button, incoming HTTP POST, form submission, or Stripe event.
3. The trigger handler sends a `workflows/execute.workflow` event to Inngest.
4. `executeWorkflow` (in `src/inngest/functions.ts`) fetches the graph, topologically sorts nodes, and runs each executor in order, threading output context between steps.
5. Execution status (`RUNNING → SUCCESS | FAILED`) is written back to Postgres in real-time.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15.5 (App Router, Turbopack) |
| UI | React 19, Tailwind CSS 4, shadcn/ui (Radix), XYFlow |
| API | tRPC 11 + TanStack Query 5 |
| Auth | Better Auth 1.3 (GitHub & Google OAuth, email/password) |
| ORM | Prisma 6 |
| Database | PostgreSQL 16 |
| Async Jobs | Inngest 3 |
| AI SDK | Vercel AI SDK 5 |
| AI Providers | Anthropic, OpenAI, Google Gemini |
| Billing | Polar |
| Encryption | Cryptr (AES-256-GCM) |
| State | Jotai (editor atoms), nuqs (URL state) |
| Monitoring | Sentry |
| Code Quality | Biome 2 |

---

## Prerequisites

| Tool | Min version | Notes |
|---|---|---|
| Node.js | 20.x | LTS; check with `node -v` |
| npm | 10.x | Bundled with Node 20 |
| Docker | 24.x | For the containerised path |
| Docker Compose | v2.x | `docker compose version` |

---

## Quick Start — Docker

This path runs the full stack (Postgres + app + auto-migration) with a single command.

```bash
# 1. Clone
git clone <repo-url> nodebase && cd nodebase

# 2. Configure environment
cp .env.example .env
#    Edit .env — at minimum set:
#      POSTGRES_PASSWORD, BETTER_AUTH_SECRET, ENCRYPTION_KEY

# 3. Build & start
docker compose up --build

# 4. Open  http://localhost:3000
```

Postgres data is persisted in the `postgres_data` Docker volume. The container runs `prisma migrate deploy` before starting Next.js, so the schema is always current.

**Tear down (keeps data)**

```bash
docker compose down
```

**Tear down (destroy data)**

```bash
docker compose down -v
```

---

## Quick Start — Local Dev

Use this path for active development — hot reload, Prisma Studio, Inngest dev UI.

```bash
# 1. Install dependencies
npm install

# 2. Configure environment
cp .env.example .env.local
#    Set DATABASE_URL to your local Postgres instance

# 3. Generate Prisma client & push schema
npx prisma generate
npx prisma migrate dev

# 4. Start everything (Next.js + Inngest dev server + ngrok)
npm run dev:all

# — or start services individually:
npm run dev          # Next.js on :3000
npm run inngest:dev  # Inngest dev UI on :8288
npm run ngrok:dev    # ngrok tunnel (requires NGROK_URL in .env.local)
```

> **Inngest dev server**: Inngest must be running locally for workflow execution to work in development. The dev UI at `http://localhost:8288` lets you inspect, replay, and debug function runs.

---

## Environment Variables

Copy `.env.example` to `.env` (Docker) or `.env.local` (local dev) and fill in the required values.

### Required

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string. Docker Compose constructs this automatically; set it manually for local dev. |
| `BETTER_AUTH_SECRET` | 32+ character random secret for signing sessions. Generate: `openssl rand -base64 32` |
| `BETTER_AUTH_URL` | Public base URL of the app, no trailing slash (e.g. `https://app.example.com`). |
| `ENCRYPTION_KEY` | Hex string used to AES-encrypt stored AI credentials. **Changing this after first use renders all saved credentials unreadable.** Generate: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `POSTGRES_PASSWORD` | Password for the `db` Compose service. Not needed for local dev. |

### OAuth Providers

Required if you want GitHub / Google login. Email + password auth works without these.

| Variable | Where to get it |
|---|---|
| `GITHUB_CLIENT_ID` | github.com → Settings → Developer settings → OAuth Apps |
| `GITHUB_CLIENT_SECRET` | Same as above |
| `GOOGLE_CLIENT_ID` | console.cloud.google.com → APIs & Services → Credentials |
| `GOOGLE_CLIENT_SECRET` | Same as above |

Callback URL pattern: `{BETTER_AUTH_URL}/api/auth/callback/{provider}`

### Inngest

| Variable | Description |
|---|---|
| `INNGEST_EVENT_KEY` | Production only — from app.inngest.com. Leave blank for local dev (auto-discovered). |
| `INNGEST_SIGNING_KEY` | Production only — used to verify webhook payloads from Inngest Cloud. |

### Polar (Billing)

| Variable | Description |
|---|---|
| `POLAR_ACCESS_TOKEN` | Personal access token from polar.sh / sandbox.polar.sh |
| `POLAR_SUCCESS_URL` | Redirect URL after successful checkout (e.g. `https://app.example.com/workflows`) |

### Optional

| Variable | Description |
|---|---|
| `SENTRY_DSN` | Ingest URL for error reporting |
| `SENTRY_AUTH_TOKEN` | Build-time token for uploading source maps to Sentry |
| `NGROK_URL` | Static ngrok domain for webhook testing in local dev |
| `POSTGRES_USER` | Postgres username (default: `nodebase`) |
| `POSTGRES_DB` | Postgres database name (default: `nodebase`) |
| `DB_PORT` | Host port for the db container (default: `5432`) |
| `APP_PORT` | Host port for the app container (default: `3000`) |

---

## Database

The app uses Prisma migrations. All migration files live in `prisma/migrations/`.

```bash
# Apply all pending migrations (safe for production)
npx prisma migrate deploy

# Create a new migration during development
npx prisma migrate dev --name <descriptive-name>

# Open Prisma Studio (browser-based DB browser)
npx prisma studio

# Re-generate the Prisma client after schema changes
npx prisma generate
```

### Schema overview

```
User ──< Session
     ──< Account       (OAuth connections)
     ──< Credential    (encrypted API keys: OPENAI | ANTHROPIC | GEMINI)
     ──< Workflow ──< Node ──< Connection
                  └─< Execution
```

Credentials are encrypted with `ENCRYPTION_KEY` before insertion and decrypted only at execution time inside server-side Inngest functions — they are never sent to the browser.

---

## External Services Setup

### Inngest

Nodebase uses Inngest for durable, retryable workflow execution.

- **Development**: Run `npm run inngest:dev`. The SDK auto-registers `src/inngest/functions.ts` against `http://localhost:3000/api/inngest`.
- **Production**: Create an app at [app.inngest.com](https://app.inngest.com), copy `INNGEST_EVENT_KEY` and `INNGEST_SIGNING_KEY` into your environment, and point the Inngest dashboard at your `/api/inngest` endpoint.

### Polar

Used for subscription billing. The `pro` product ID is hardcoded in `src/lib/auth.ts`. If you fork this project, update that product ID to your own.

- Sandbox: [sandbox.polar.sh](https://sandbox.polar.sh)
- Production: [polar.sh](https://polar.sh)

The app currently targets the Polar sandbox (`server: "sandbox"` in `src/lib/polar.ts`). Switch to `"production"` before going live.

### GitHub & Google OAuth

Set the **Authorization callback URL** in each provider's OAuth app settings:

```
https://<your-domain>/api/auth/callback/github
https://<your-domain>/api/auth/callback/google
```

For local dev using ngrok, use:

```
https://<your-ngrok-domain>/api/auth/callback/github
https://<your-ngrok-domain>/api/auth/callback/google
```

---

## Development Scripts

| Script | What it does |
|---|---|
| `npm run dev` | Next.js dev server on `:3000` with Turbopack |
| `npm run dev:all` | Parallel: Next.js + Inngest dev server + ngrok tunnel |
| `npm run inngest:dev` | Inngest dev server + UI on `:8288` |
| `npm run ngrok:dev` | ngrok HTTP tunnel to `:3000` using `NGROK_URL` from env |
| `npm run build` | Production build with Turbopack |
| `npm start` | Start production server (run `build` first) |
| `npm run lint` | Biome static analysis |
| `npm run format` | Biome auto-format (writes files) |

---

## Production Deployment

### Docker Compose (self-hosted)

See [Quick Start — Docker](#quick-start--docker). For a production host:

1. Point your DNS A record at the server.
2. Terminate TLS at a reverse proxy (nginx, Caddy, Traefik) and proxy to port 3000.
3. Set `BETTER_AUTH_URL` to your public HTTPS domain.
4. Rotate `BETTER_AUTH_SECRET` and `ENCRYPTION_KEY` before go-live — these cannot be changed after users have signed up or saved credentials without a coordinated migration.
5. Configure `INNGEST_EVENT_KEY` and `INNGEST_SIGNING_KEY` from Inngest Cloud.
6. Switch `src/lib/polar.ts` from `server: "sandbox"` to `server: "production"`.

**Recommended Caddy snippet (automatic TLS + HTTP/2):**

```
app.example.com {
    reverse_proxy app:3000
}
```

### Vercel

The project ships with Sentry's Vercel integration pre-configured. Remove `output: "standalone"` from `next.config.ts` before deploying to Vercel — the standalone output is only needed for self-hosted Docker.

Set all environment variables listed above in the Vercel dashboard under **Settings → Environment Variables**.

### Building the image independently

```bash
docker build -t nodebase:latest .

docker run --rm \
  --env-file .env \
  -e DATABASE_URL="postgresql://user:pass@host:5432/db" \
  -p 3000:3000 \
  nodebase:latest
```

---

## Project Structure

```
nodebase/
├── prisma/
│   ├── schema.prisma           # Data model + generator config
│   └── migrations/             # Timestamped migration files
│
├── public/
│   └── logos/                  # Node-type SVG logos
│
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── (auth)/             # Login & signup pages
│   │   ├── (dashboard)/
│   │   │   ├── (editor)/       # Workflow canvas editor
│   │   │   └── (rest)/         # Credentials, executions, workflows list
│   │   └── api/
│   │       ├── auth/           # Better Auth catch-all
│   │       ├── inngest/        # Inngest event receiver
│   │       ├── trpc/           # tRPC handler
│   │       └── webhooks/       # HTTP / Stripe / Google Forms
│   │
│   ├── components/
│   │   ├── ui/                 # shadcn/ui primitives
│   │   └── react-flow/         # XYFlow custom nodes & edges
│   │
│   ├── config/
│   │   ├── constants.ts        # Pagination defaults
│   │   └── node-components.ts  # Node type → component mapping
│   │
│   ├── features/               # Vertical feature slices
│   │   ├── auth/
│   │   ├── credentials/
│   │   ├── editor/             # Jotai atoms + editor components
│   │   ├── executions/         # Execution history + node executors
│   │   ├── triggers/
│   │   ├── workflows/
│   │   └── subscriptions/
│   │
│   ├── inngest/
│   │   ├── client.ts           # Inngest instance
│   │   ├── functions.ts        # executeWorkflow — topological executor
│   │   ├── utils.ts            # Toposort helpers
│   │   └── channels/           # One executor per node type
│   │
│   ├── lib/
│   │   ├── auth.ts             # Better Auth config
│   │   ├── db.ts               # Prisma singleton
│   │   ├── encryption.ts       # encrypt / decrypt (Cryptr)
│   │   └── polar.ts            # Polar client
│   │
│   └── trpc/
│       ├── init.ts             # Context + protectedProcedure middleware
│       ├── client.tsx          # TRPCReactProvider
│       ├── server.tsx          # Server caller + prefetch helpers
│       └── routers/            # Router tree (_app.ts + sub-routers)
│
├── .env.example                # All supported environment variables
├── .dockerignore
├── docker-compose.yml          # Postgres + app (production)
├── docker-entrypoint.sh        # migrate → start
├── Dockerfile                  # Multi-stage production image
├── biome.json                  # Linting & formatting config
├── components.json             # shadcn/ui config
├── mprocs.yaml                 # Dev: run next + inngest + ngrok in parallel
├── next.config.ts
├── prisma/schema.prisma
└── tsconfig.json
```


