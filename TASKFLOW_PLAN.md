# TaskFlow — Trunk-Based Development + Feature Toggles Demo

> **Historical document.** This is the original build plan, kept as written. The
> implementation has since moved on in two ways: hosting is Railway, not Render
> (see [docs/RAILWAY_DEPLOYMENT.md](docs/RAILWAY_DEPLOYMENT.md)), and the deploy
> pipeline is the shared reusable workflow in
> [power-tester/deploy-workflows](https://github.com/power-tester/deploy-workflows)
> rather than a local `_deploy.yml`. The `ci.yml` and `deploy-dev.yml` named below
> are `pr.yml` and `deploy-ci.yml` today. For the current pipeline see
> [docs/TRUNK_BASED_DEV.md](docs/TRUNK_BASED_DEV.md).

**Stack**: React + TypeScript (Vite), Express + TypeScript, MongoDB Atlas / Docker  
**Deployment**: Render (PaaS) — 3 separate services: `taskflow-dev`, `taskflow-staging`, `taskflow-prod`  
**Auth**: JWT (enables per-user flag demo potential)  
**Feature flags**: File-based JSON config provider behind a single abstraction module — designed to be swapped for LaunchDarkly/Unleash by changing 1–2 files  
**Branching**: No long-lived branches — everyone commits to `main`; flags protect WIP features

---

## The Demo Story — Feature Flag Matrix

| Flag | local | dev | staging | prod | Story |
|---|---|---|---|---|---|
| `TASK_LABELS` | ✓ | ✓ | ✓ | ✓ | Fully released |
| `NOTIFICATIONS` | ✓ | ✓ | ✓ | ✓ | Fully released |
| `TASK_PRIORITY` | ✓ | ✓ | ✓ | ✗ | Staging QA, not in prod yet |
| `TASK_COMMENTS` | ✓ | ✓ | ✓ | ✗ | Staging QA |
| `CSV_EXPORT` | ✓ | ✓ | ✗ | ✗ | Dev testing only |
| `DASHBOARD_ANALYTICS` | ✓ | ✓ | ✗ | ✗ | Dev testing only |
| `COLLABORATION` | ✓ | ✓ | ✗ | ✗ | Dev only |
| `DARK_MODE` | ✓ | ✗ | ✗ | ✗ | Experimental, local prototype |
| `AI_SUGGESTIONS` | ✓ | ✗ | ✗ | ✗ | Experimental, local only |

This matrix creates a clear visual demo: same codebase, same `main` branch — different flags per environment.

---

## Feature Flag Architecture (the key design)

The entire app talks to a single `feature-flags` package. All components and routes call
`isEnabled('FLAG_NAME')` — nothing else. Swapping providers means:

1. Create `launchdarkly-provider.ts` implementing the same `Provider` interface
2. Update `feature-flags/src/index.ts` to import it
3. **Zero changes to application code**

```
feature-flags/
  src/
    index.ts            ← Public API (isEnabled, getAllFlags) — this is all the app sees
    types.ts            ← FlagName union type + Provider interface
    config-provider.ts  ← Current impl: reads flags.{NODE_ENV}.json
    config/
      flags.local.json
      flags.dev.json
      flags.staging.json
      flags.production.json
```

The server exposes `GET /api/flags` so the React client knows which flags are active without
bundling environment configs in the frontend build.

---

## Monorepo Structure

```
/
├── packages/
│   ├── client/                    (React + Vite + TypeScript)
│   │   └── src/
│   │       ├── contexts/
│   │       │   └── FeatureFlagContext.tsx
│   │       ├── hooks/
│   │       │   └── useFeatureFlag.ts
│   │       ├── components/        (feature-gated UI components)
│   │       └── pages/
│   └── server/                    (Express + TypeScript + Mongoose)
│       └── src/
│           ├── middleware/
│           │   ├── auth.ts
│           │   └── featureGate.ts
│           ├── models/
│           ├── routes/
│           └── app.ts
├── feature-flags/                 (shared package, used by server)
│   └── src/
│       ├── index.ts
│       ├── types.ts
│       ├── config-provider.ts
│       └── config/
│           ├── flags.local.json
│           ├── flags.dev.json
│           ├── flags.staging.json
│           └── flags.production.json
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── _deploy.yml            (reusable)
│       ├── deploy-dev.yml
│       ├── deploy-staging.yml
│       └── deploy-production.yml
├── docker-compose.yml
├── package.json                   (npm workspaces root)
├── tsconfig.base.json
└── README.md
```

---

## GitHub Actions Pipeline Design

```
Short-lived branch → PR → CI gate (lint, type-check, test, build)
                        ↓
                   Merge to main
                        ↓
               Auto-deploy to Dev (Render)
                        ↓
         Manual trigger: promote to Staging (SHA input)
                        ↓
    Manual trigger + GitHub Environment approval: promote to Prod
                        ↓
              Auto-creates GitHub Release tag
```

| Workflow | Trigger | Target | Notes |
|---|---|---|---|
| `ci.yml` | Pull request to `main` | — | Lint, type-check, test, build |
| `deploy-dev.yml` | Push to `main` | Dev | Auto-fires on every merge |
| `deploy-staging.yml` | `workflow_dispatch` (SHA input) | Staging | Posts comment on related PR |
| `deploy-production.yml` | `workflow_dispatch` + env approval | Production | Creates GitHub Release tag |

`_deploy.yml` is the reusable core shared by all three deploy workflows:
**install → build → call Render deploy hook → health-check**

---

## Implementation Phases

### Phase 1 — Monorepo Skeleton *(blocking, do first)*
1. Root `package.json` with npm workspaces (`packages/*`, `feature-flags`)
2. Shared `tsconfig.base.json`, ESLint + Prettier config
3. `docker-compose.yml` — MongoDB + server + client

### Phase 2 — Feature Flag System *(blocks Phases 3 & 4)*
4. `feature-flags/src/types.ts` — `FlagName` union type, `Provider` interface
5. `feature-flags/src/config-provider.ts` — reads JSON by `NODE_ENV`
6. `feature-flags/src/index.ts` — public API that delegates to provider
7. Four JSON config files with the flag matrix above

### Phase 3 — Server *(depends on Phase 2)*
8. Express app scaffold: CORS, Helmet, JSON body parser, error handler
9. MongoDB/Mongoose connection; models: `User`, `Task`, `Label`, `Comment`
10. JWT auth routes (`/api/auth/register`, `/api/auth/login`) + auth middleware
11. `GET /api/flags` route
12. Tasks CRUD, feature-gated routes via `featureGate(flagName)` Express middleware:
    - `POST /api/tasks/:id/comments` → gated by `TASK_COMMENTS`
    - `GET /api/tasks/export/csv` → gated by `CSV_EXPORT`
    - `GET /api/dashboard` → gated by `DASHBOARD_ANALYTICS`
    - `POST /api/collaboration/share` → gated by `COLLABORATION`

### Phase 4 — Client *(depends on Phase 2, can run parallel with Phase 3)*
13. Vite React-TS scaffold inside `packages/client`
14. `FeatureFlagContext` loaded at app startup via `GET /api/flags`
15. `useFeatureFlag(flag)` hook — `const isOn = useFeatureFlag('DARK_MODE')`
16. React Router: `/login`, `/register`, `/tasks`, `/dashboard`
17. Auth context + protected route wrapper
18. Feature-gated UI — each component guarded by its flag:
    - `LabelSelector` → `TASK_LABELS`
    - `PriorityBadge` → `TASK_PRIORITY`
    - `CommentSection` → `TASK_COMMENTS`
    - `ExportButton` → `CSV_EXPORT`
    - `DashboardPage` → `DASHBOARD_ANALYTICS` (entire route gated)
    - `CollaborationPanel` → `COLLABORATION`
    - `AiSuggestionButton` → `AI_SUGGESTIONS`
    - `DarkModeToggle` → `DARK_MODE`
    - `NotificationBell` → `NOTIFICATIONS`
19. Navbar that shows/hides links based on active flags

### Phase 5 — CI/CD Workflows *(parallel with Phases 3 & 4)*
20. `ci.yml` — PR gate (lint + type-check + test + build)
21. `_deploy.yml` — reusable deploy core
22. `deploy-dev.yml` — push to main trigger
23. `deploy-staging.yml` — manual dispatch with SHA input
24. `deploy-production.yml` — manual dispatch + GitHub Environment approval + release tag

### Phase 6 — Docs
25. `README.md` — local setup, environment variables, npm scripts
26. `docs/TRUNK_BASED_DEV.md` — workflow guide with flag lifecycle diagram
27. `docs/FEATURE_FLAGS.md` — how to add a flag, how to swap to a paid provider
28. `docs/DEMO_SCRIPT.md` — step-by-step walkthrough for engineering demo

---

## Key Files Summary

| File | Purpose |
|---|---|
| `feature-flags/src/index.ts` | **Provider swap seam** — only file the app imports |
| `feature-flags/src/config-provider.ts` | File-based implementation (replace this to migrate) |
| `feature-flags/src/types.ts` | `FlagName` union + `Provider` interface |
| `feature-flags/src/config/flags.*.json` | Per-environment flag values |
| `packages/server/src/middleware/featureGate.ts` | Express middleware for gating routes |
| `packages/server/src/routes/flags.ts` | `GET /api/flags` endpoint |
| `packages/client/src/contexts/FeatureFlagContext.tsx` | React context loaded at startup |
| `packages/client/src/hooks/useFeatureFlag.ts` | `useFeatureFlag('FLAG_NAME')` hook |
| `.github/workflows/_deploy.yml` | Reusable deploy logic (DRY) |

---

## Provider Swap Contract — How to Migrate to a Paid Provider

To switch to LaunchDarkly (or Unleash, Split.io, etc.) in the future:

1. Create `feature-flags/src/launchdarkly-provider.ts` implementing the `Provider` interface
2. Update `feature-flags/src/index.ts` to import the new provider instead of `config-provider`
3. Archive the JSON config files
4. **Zero changes to application code** — no server routes, no React components, no hooks

This is the **entire migration surface area**: 2 files.

---

## Verification Checklist

- [ ] `docker-compose up` → all 3 services start locally
- [ ] Register + login → JWT stored in localStorage
- [ ] All 9 feature-gated components visible on `localhost`
- [ ] Set `NODE_ENV=dev` → `DARK_MODE` + `AI_SUGGESTIONS` not visible; others correct
- [ ] Set `NODE_ENV=staging` → only staging-approved flags shown
- [ ] Set `NODE_ENV=production` → only fully-released flags visible
- [ ] CI workflow blocks a PR that has a lint error
- [ ] Merge to `main` → `deploy-dev.yml` fires automatically
- [ ] Staging deploy requires manual trigger + SHA input
- [ ] Prod deploy shows GitHub Environment approval gate before proceeding
- [ ] Production deploy creates a GitHub Release tag

---

## Further Considerations

1. **Render secrets** — Create 3 Render services (dev/staging/prod) and add deploy hook URLs as GitHub secrets: `RENDER_DEV_HOOK`, `RENDER_STAGING_HOOK`, `RENDER_PROD_HOOK`. Workflows will use placeholders until these are configured.
2. **MongoDB** — Local: Docker container. Cloud environments: MongoDB Atlas free tier with the connection URI stored as a GitHub secret per environment.
3. **Per-user flags (future)** — Because auth is included, flags can optionally be extended to support per-user overrides (e.g., beta testers) without changing the `Provider` interface — just add a `isEnabledForUser(flag, userId)` method.
