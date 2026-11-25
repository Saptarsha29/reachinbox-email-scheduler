# Project Handoff — ReachInbox Email Scheduler

Maintainer: **Saptarsha Saha** (`saptarshasaha6@gmail.com`)  
Repository: `https://github.com/Saptarsha29/reachinbox-email-scheduler`  
Last updated: `2026-11-25`

---

## 1. Overview

Production-grade **email job scheduler and React dashboard** built with TypeScript, Express, BullMQ + Redis (AOF), PostgreSQL (Prisma 6), Elasticsearch 8, and React 18 + Vite + Tailwind v4 in an npm workspaces monorepo.

- **Workspace root:** `e:\new\reachinbox-email-scheduler-main\reachinbox-email-scheduler-main` *(run all `npm` commands from this inner directory where `package.json` lives)*
- `apps/api` — Express REST API, BullMQ workers (`email-send`, `notifications`, `search-index`), Vitest integration/unit suites
- `apps/web` — Vite + React dashboard
- `packages/shared` — shared Zod schemas, types, and pure logic (spintax, spam check, warm-up, health), compiled with `tsc` to `dist/`

## 2. Project Rules

1. **Git authorship:**
   ```bash
   git config user.name "Saptarsha Saha"
   git config user.email "saptarshasaha6@gmail.com"
   ```
2. **Hard constraints:**
   - **No cron anywhere** — enforced by `npm run check:no-cron` and CI.
   - **No duplicate sends** — enforced across BullMQ `jobId`, conditional DB claim, `Idempotency-Key`, and deterministic `Message-ID`.
3. **Collaborators (if submitting privately):** Grant access to `Mitrajit` and `Yadav036`.

## 3. Quick Start & Commands

**Prerequisites:** Node 20+ and Docker Desktop (running).

```bash
npm install          # installs deps, builds @ri/shared, generates Prisma client
npm run setup        # creates .env, starts PG/Redis/ES, runs migrations, seeds 3 Ethereal senders, builds ES index
npm run dev          # starts API (:4000), worker, and web (:5173)
npm run dev:demo     # same stack with 60s windows, 2s min delay, 120s warm-up "days"
```

### Quality & Verification Gates

```bash
npm run check:no-cron
npm run typecheck
npm run lint
npm test             # requires Postgres (:5434), Redis (:6380), Elasticsearch (:9200), and .env
npm run build
npm audit
```

### Useful Scripts

- **Queue dashboard:** `http://localhost:4000/admin/queues`
- **Demo load / restart / verify / reset:** `npm run demo -w @ri/api -- load | restart | verify <campaignId> | reset`
- **Local dev session token (no Google OAuth needed):** `npm run dev:session -w @ri/api`
- **Infrastructure containers:** `npm run infra:up` / `npm run infra:down`

## 4. Architecture Summary

- **Scheduling:** Each email is persisted in Postgres and enqueued as a BullMQ **delayed job** (`jobId = emailId`, `delay = scheduledAt - now`). No cron or polling timers.
- **Persistence:** Redis runs with AOF (`appendonly yes`); Postgres is the source of truth; a one-shot boot reconciler (`apps/api/src/recovery/reconciler.ts`) repairs any missing jobs at worker startup.
- **Idempotency:** Four layers — `jobId = emailId`, conditional `UPDATE ... WHERE status IN ('SCHEDULED', 'RATE_LIMITED')`, request `Idempotency-Key`, and deterministic `Message-ID`. Interrupted in-flight sends are marked `FAILED` (at-most-once) rather than risk duplicates.
- **Rate limiting:** Atomic Redis Lua scripts reserve per-sender send slots (minimum `2000 ms` gap) and check global, per-sender, per-campaign, and warm-up daily caps all-or-none. Over-limit emails move to `RATE_LIMITED` and reschedule into the next window in FIFO order.
- **Search & Live UI:** Every status change re-indexes from Postgres to Elasticsearch (`npm run reindex -w @ri/api` rebuilds). Live dashboard updates stream via Redis pub/sub → Server-Sent Events (`/api/events`).
- **Slack & Ask Inbox:** Per-user encrypted Slack OAuth v2 notifications (`AES-256-GCM`); rule-based offline assistant with confirmation tokens audited in `AssistantAction`.

## 5. Remaining Owner Setup (Live Credentials & Demo)

| Item | Action |
|---|---|
| **Google OAuth** | Create Web OAuth credentials in Google Cloud Console (`http://localhost:4000/api/auth/google/callback`) and set `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` in `.env` |
| **Live Slack OAuth** | Expose `:4000` over HTTPS (e.g. `cloudflared tunnel --url http://localhost:4000`), configure Slack app scopes (`incoming-webhook`, `chat:write`), and set `SLACK_*` in `.env` |
| **Demo Recording** | Follow [`docs/DEMO_SCRIPT.md`](DEMO_SCRIPT.md) in `npm run dev:demo` mode and link the recording in `README.md` |

## 6. Key Files

| Purpose | Path |
|---|---|
| Setup, architecture & API docs | [`README.md`](../README.md) |
| Requirement review pack | [`docs/REVIEW.md`](REVIEW.md) |
| Architecture & decision log | [`GODFATHER.md`](../GODFATHER.md) |
| Measured benchmarks & crash drills | [`docs/VERIFICATION.md`](VERIFICATION.md) |
| Demo video script | [`docs/DEMO_SCRIPT.md`](DEMO_SCRIPT.md) |
| Worker pipeline | [`apps/api/src/queues/emailProcessor.ts`](../apps/api/src/queues/emailProcessor.ts) |
| Atomic Redis rate limiter | [`apps/api/src/throttle/rateLimiter.ts`](../apps/api/src/throttle/rateLimiter.ts) |
| Boot reconciler | [`apps/api/src/recovery/reconciler.ts`](../apps/api/src/recovery/reconciler.ts) |
| Shared schemas & pure logic | [`packages/shared/src/`](../packages/shared/src/) |
