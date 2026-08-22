# TaskFlow

[![PR Validation](https://github.com/PramodKumarYadav/taskflow/actions/workflows/pr.yml/badge.svg)](https://github.com/PramodKumarYadav/taskflow/actions/workflows/pr.yml)
[![Deploy → ci](https://github.com/PramodKumarYadav/taskflow/actions/workflows/deploy-ci.yml/badge.svg)](https://github.com/PramodKumarYadav/taskflow/actions/workflows/deploy-ci.yml)
[![Deploy → Staging](https://github.com/PramodKumarYadav/taskflow/actions/workflows/deploy-staging.yml/badge.svg)](https://github.com/PramodKumarYadav/taskflow/actions/workflows/deploy-staging.yml)
[![Deploy → Production](https://github.com/PramodKumarYadav/taskflow/actions/workflows/deploy-production.yml/badge.svg)](https://github.com/PramodKumarYadav/taskflow/actions/workflows/deploy-production.yml)

<a href="https://taskflow-client-ci.up.railway.app/" target="_blank" rel="noopener noreferrer"><img src="https://img.shields.io/badge/%F0%9F%9A%80_CI-open-6366f1?style=flat-square" alt="CI environment"></a>
<a href="https://taskflow-client-staging.up.railway.app/" target="_blank" rel="noopener noreferrer"><img src="https://img.shields.io/badge/%F0%9F%9A%80_Staging-open-6366f1?style=flat-square" alt="Staging environment"></a>
<a href="https://taskflow-client-prod.up.railway.app/" target="_blank" rel="noopener noreferrer"><img src="https://img.shields.io/badge/%F0%9F%9A%80_Production-open-6366f1?style=flat-square" alt="Production environment"></a>

A full-stack task management app built to demo **trunk-based development with feature toggles**.

**Stack**: React + TypeScript (Vite) · Express + TypeScript · MongoDB · npm workspaces monorepo

## Background & Requirements

This project was built to demonstrate trunk-based development and feature toggles to an engineering team. The original requirements were:

- **Trunk-based development showcase** — demonstrate how teams ship continuously to a single `main` branch without long-lived feature branches
- **Feature toggles as the gate** — incomplete or risky features ship behind a flag that's off in production; turning a feature on is a JSON change, not a code deployment
- **File-based flag config, no paid tooling** — flags are plain JSON files per environment; the system is built around a `Provider` interface so migrating to LaunchDarkly, Unleash, or any other tool requires changing one file
- **Four environments** — `development` (all 9 flags on) → `ci` (7 flags) → `staging` (4 flags) → `production` (2 flags), each with progressively fewer experimental features
- **Real-world CI/CD pipelines** — GitHub Actions with automated dev deploys on merge, manual staging promotion by SHA, and a production gate requiring environment approval before deploy
- **React + Express + MongoDB** — a realistic task management app with enough modules (tasks, comments, labels, collaboration, dashboard analytics) to meaningfully demo multiple flags at once
- **JWT authentication** — register/login with token-based auth protecting all task routes
- **Run with or without Docker** — `docker-compose up --build` for a one-command start; `npm run dev:local` for a local run without Docker

## Quick Start

### Prerequisites

- Node.js 20+
- MongoDB — either [MongoDB Community](https://www.mongodb.com/try/download/community) installed locally, or a free [MongoDB Atlas](https://www.mongodb.com/atlas) cluster

---

### Option A — With Docker (easiest, no local MongoDB needed)

**Extra prerequisite**: Docker & Docker Compose

```bash
docker-compose up --build
```

| Service    | URL                   |
| ---------- | --------------------- |
| Client     | http://localhost:5173 |
| Server API | http://localhost:4000 |
| MongoDB    | localhost:27017       |

---

### Option B — Without Docker

**1. Install dependencies**

```bash
npm install
```

**2. Configure environment variables**

```bash
cp packages/client/.env.example packages/client/.env
# Usually no edits needed; defaults point to localhost:4000

# Server secrets are managed via dotenv-vault.
# Obtain the DOTENV_KEY for development from the team and set it in packages/server/.env:
# DOTENV_KEY=dotenv://:key_xxxx@dotenv.org/vault/.env.vault?environment=development
```

**3. Start MongoDB**

```bash
# macOS (Homebrew)
brew services start mongodb-community
```

If you prefer not to install MongoDB locally, use a free [Atlas](https://www.mongodb.com/atlas) cluster and set `MONGODB_URI` in `packages/server/.env` to your Atlas connection string.

**4. Run everything at once**

```bash
npm run dev:local
```

This builds the feature-flags package, then starts the Express server (port 4000) and the Vite dev server (port 5173) side-by-side with colour-coded output.

**Or run them separately in two terminals:**

```bash
# Terminal 1
npm run dev:server

# Terminal 2
npm run dev:client
```

| Service    | URL                   |
| ---------- | --------------------- |
| Client     | http://localhost:5173 |
| Server API | http://localhost:4000 |

By default, the server runs with `NODE_ENV=development` — __all 9 feature flags are enabled__ in this mode. (Run `dev:ci` to test with the ci flag set.)

---

## Project Structure

```text
/
├── packages/
│   ├── client/          React + Vite + TypeScript
│   └── server/          Express + TypeScript + Mongoose
├── feature-flags/       @taskflow/feature-flags — shared flag package
├── docs/                Guides: feature flags, trunk-based dev, demo script
├── .github/workflows/   CI + CD pipelines (GitHub Actions)
└── docker-compose.yml
```

---

## Environment Variables

### Server (`packages/server/.env`)

Server secrets are managed via [dotenv-vault](https://www.dotenv.org/). Set the `DOTENV_KEY` for your environment in `packages/server/.env` — dotenv-vault will inject all other values at startup:

| Variable        | Description                          | Example / Default                    |
| --------------- | ------------------------------------ | ------------------------------------ |
| `NODE_ENV`      | Controls which flag config is loaded | `development`                        |
| `PORT`          | HTTP port                            | `4000`                               |
| `MONGODB_URI`   | MongoDB connection string            | `mongodb://localhost:27017/taskflow` |
| `JWT_SECRET`    | Long random string for signing JWTs  | _(generate — see below)_             |
| `CLIENT_ORIGIN` | CORS allowed origin                  | `http://localhost:5173`              |

Generate a JWT secret:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### Client (`packages/client/.env`)

Copy `packages/client/.env.example` to `packages/client/.env`:

| Variable       | Description          | Default                 |
| -------------- | -------------------- | ----------------------- |
| `VITE_API_URL` | Backend API base URL | `http://localhost:4000` |

---

## Feature Flags

Flags are file-based JSON configs in `feature-flags/src/config/`. The active config is chosen by `NODE_ENV`.
See [docs/FEATURE_FLAGS.md](docs/FEATURE_FLAGS.md) for the full guide, including how to swap to LaunchDarkly in one file change.

| Flag                  | development | ci | staging | production |
| --------------------- | :---------: | :-: | :-----: | :--------: |
| `TASK_LABELS`         |  ✓   |  ✓  |    ✓    |  ✓  |
| `NOTIFICATIONS`       |  ✓   |  ✓  |    ✓    |  ✓  |
| `TASK_PRIORITY`       |  ✓   |  ✓  |    ✓    |  ✗  |
| `TASK_COMMENTS`       |  ✓   |  ✓  |    ✓    |  ✗  |
| `CSV_EXPORT`          |  ✓   |  ✓  |    ✗    |  ✗  |
| `DASHBOARD_ANALYTICS` |  ✓   |  ✓  |    ✗    |  ✗  |
| `COLLABORATION`       |  ✓   |  ✓  |    ✗    |  ✗  |
| `DARK_MODE`           |  ✓   |  ✗  |    ✗    |  ✗  |
| `AI_SUGGESTIONS`      |  ✓   |  ✗  |    ✗    |  ✗  |

### Use Cases & Implementation

Feature toggles can guard changes at every layer of the stack. The table below shows a distinct category of change, the flag that demonstrates it in this codebase, and the exact files involved.

| Use Case | Category | Flag | Key files |
| --- | --- | --- | --- |
| A completely new page is not ready for users yet — hide the route, the nav link, and the server endpoint behind a single flag | **New page** | `DASHBOARD_ANALYTICS` | [App.tsx](packages/client/src/App.tsx) (`<FlagGate>` wraps the `/dashboard` route) · [Navbar.tsx](packages/client/src/components/Navbar.tsx) (nav link) · [DashboardPage.tsx](packages/client/src/pages/DashboardPage.tsx) · [dashboard.ts](packages/server/src/routes/dashboard.ts) (`featureGate` middleware) |
| A new section is added inside a page that already ships to production — only the section is toggled, not the whole page | **New UI section within existing page** | `TASK_COMMENTS` | [TaskCard.tsx](packages/client/src/components/TaskCard.tsx) (`useFeatureFlag` hides comment section) · [CommentSection.tsx](packages/client/src/components/CommentSection.tsx) |
| A new input is added to an existing form, backed by a schema field that is deployed but not yet surfaced | **New field on existing form & model** | `TASK_PRIORITY` | [TaskForm.tsx](packages/client/src/components/TaskForm.tsx) (priority selector) · [TaskCard.tsx](packages/client/src/components/TaskCard.tsx) (priority badge) · [Task.ts](packages/server/src/models/Task.ts) (`priority` enum field in schema) |
| A persistent widget (button, toggle, bell) is added to the global app shell visible on every page | **New global UI element** | `DARK_MODE` · `NOTIFICATIONS` | [Navbar.tsx](packages/client/src/components/Navbar.tsx) (`useFeatureFlag` for both) · [DarkModeToggle.tsx](packages/client/src/components/DarkModeToggle.tsx) · [NotificationBell.tsx](packages/client/src/components/NotificationBell.tsx) |
| A new server endpoint is added but the feature is not ready — gate it at the API layer so even a direct HTTP call is rejected | **New backend API route** | `CSV_EXPORT` | [tasks.ts](packages/server/src/routes/tasks.ts) (`GET /api/tasks/export/csv` guarded by `featureGate`) · [featureGate.ts](packages/server/src/middleware/featureGate.ts) (the middleware) · [TasksPage.tsx](packages/client/src/pages/TasksPage.tsx) (export button hidden on client) |
| A new Mongoose model and its collection are introduced — documents should only be written while the flag is on | **New database collection** | `TASK_COMMENTS` | [Comment.ts](packages/server/src/models/Comment.ts) (new schema/collection) · [tasks.ts](packages/server/src/routes/tasks.ts) (comment routes guarded by `featureGate`) |
| A feature touches every layer — a new schema field, new server routes, and new UI components all ship together | **Full-stack cross-cutting feature** | `COLLABORATION` | [Task.ts](packages/server/src/models/Task.ts) (`sharedWith[]` field) · [collaboration.ts](packages/server/src/routes/collaboration.ts) (routes + `featureGate`) · [CollaborationPanel.tsx](packages/client/src/components/CollaborationPanel.tsx) · [TaskCard.tsx](packages/client/src/components/TaskCard.tsx) (share button) |
| A speculative or experimental feature that may be cut entirely — ships in the codebase without risk because it is off everywhere except development | **Experimental / AI feature** | `AI_SUGGESTIONS` | [TaskForm.tsx](packages/client/src/components/TaskForm.tsx) (`useFeatureFlag` gates the "Suggest" button and the AI logic) |
| A new field is added to an existing document — the flag can be safely toggled off in production at any time without data loss or errors because the field is optional with a default | **Backward-compatible schema change (additive)** | `TASK_PRIORITY` · `COLLABORATION` | [Task.ts](packages/server/src/models/Task.ts) — `priority` has `default: 'medium'` so pre-existing documents read back safely; `sharedWith` is an optional array that defaults to `[]`. No migration script needed. Toggling the flag off hides the UI and stops writes, but existing field data is untouched and safe to re-enable later. |

> **Pattern note — additive vs. breaking schema changes:**
> All schema changes in this codebase follow the **additive-only** rule: new fields are always optional with a safe default. This is what makes the flag safely toggleable in both directions. If you need to add a `required` field, or change an existing field's type, you must **migrate first**: deploy a migration that backfills all existing documents with a safe value, confirm it has run on production, and only then turn the flag on. The flag gates the application code; the migration must make the data safe independently.

The shared flag contract (type definitions, `Provider` interface) lives in [feature-flags/src/types.ts](feature-flags/src/types.ts). Default values and metadata for all flags are in [feature-flags/src/config/flags.default.json](feature-flags/src/config/flags.default.json), with per-environment overrides in the sibling `flags.<env>.json` files.

To simulate a different environment locally, edit `NODE_ENV` in `packages/server/.env`:

```bash
NODE_ENV=staging     # 4 flags on  (simulates staging before next release)
NODE_ENV=production  # 2 flags on  (simulates what users currently see)
```

---

## CI/CD Pipelines

See [docs/TRUNK_BASED_DEV.md](docs/TRUNK_BASED_DEV.md) for the full trunk-based development guide.

| Workflow                | Trigger                       | What it does                                   |
| ----------------------- | ----------------------------- | ---------------------------------------------- |
| `pr.yml`                | Pull request to `main`        | Lint, type-check, test, build                  |
| `deploy-ci.yml`         | Push to `main`                | Auto-deploys to ci environment                 |
| `deploy-staging.yml`    | Manual (commit SHA input)     | Promotes a specific commit to staging          |
| `deploy-production.yml` | Manual + environment approval | Promotes to prod and creates a GitHub Release  |

---

## GitHub Secrets & Variables Required

Set per environment in **Settings → Environments → `<env>` → Secrets**:

| Secret | Description |
| --- | --- |
| `RAILWAY_TOKEN` | Railway project token — Settings → Tokens in Railway dashboard (scope to the environment) |

> __`DOTENV_KEY` is set in Railway per environment (ci/staging/production), not in GitHub.__ In the Railway dashboard for your server service and each environment → Variables, add a `DOTENV_KEY` with the value from `npx dotenv-vault@latest keys <env>`. When the server process starts, dotenv-vault uses this key to decrypt `.env.vault` and load `MONGODB_URI`, `JWT_SECRET`, and all other secrets into the process environment.

Set per environment in **Settings → Environments → `<env>` → Variables**:

| Variable | Example |
| --- | --- |
| `CLIENT_URL` | `https://taskflow-ci-client.up.railway.app` |
| `SERVER_URL` | `https://taskflow-ci-server.up.railway.app` |

Repeat for each of the three environments: `ci`, `staging`, `production`.

> All other secrets (`MONGODB_URI`, `JWT_SECRET`, etc.) are encrypted inside `.env.vault` and injected at runtime by dotenv-vault — no need to add them to GitHub. The only secret GitHub needs is `RAILWAY_TOKEN` to trigger deploys.

---

## Scripts Reference

| Script               | What it runs                                             |
| -------------------- | -------------------------------------------------------- |
| `npm run dev:local`  | Build flags package, then start server + client together |
| `npm run dev:server` | Build flags package, then start server only              |
| `npm run dev:client` | Start Vite dev server only                               |
| `npm run dev`        | `docker-compose up --build`                              |
| `npm run build`      | Production build of all packages                         |
| `npm run lint`       | ESLint across all packages                               |
| `npm run typecheck`  | `tsc --noEmit` across all packages                       |
| `npm run test`       | Run all tests (Vitest)                                   |

---

## Docs

- [Feature Flags guide](docs/FEATURE_FLAGS.md) — flag matrix, provider interface, migration to paid tools
- [Trunk-Based Development guide](docs/TRUNK_BASED_DEV.md) — branch strategy, pipeline walkthrough
- [Demo Script](docs/DEMO_SCRIPT.md) — step-by-step walkthrough for presenting to your team

## Troubleshooting

### Port already in use

If you see `EADDRINUSE` errors on startup, a previous `ts-node-dev` or Vite process is still running.
`ts-node-dev` spawns two processes (watcher + server child), so killing by port alone may miss the parent.
Kill all processes on both ports, then re-run:

```bash
# Kill by port — catches the child process holding the socket
lsof -ti :4000 | xargs kill -9 2>/dev/null; lsof -ti :5173 | xargs kill -9 2>/dev/null

# If port 4000 is still in use, kill the entire ts-node-dev process tree too
pkill -9 -f ts-node-dev 2>/dev/null; true
```

Then re-run your dev command.