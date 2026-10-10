# Trunk-Based Development

TaskFlow is built on trunk-based development (TBD): all engineers commit to a single long-lived branch (`main`). Feature toggles replace long-lived feature branches.

---

## Core principles

| Principle | How it's applied in TaskFlow |
| --------- | ----------------------------- |
| One trunk | All work merges to `main`; no `develop` or `release` branches |
| Short-lived branches | Feature branches live ≤ 2 days; PR → review → merge |
| Feature toggles as the gate | Incomplete features ship behind a flag that's off in production |
| CI on every commit | Lint + type-check + test + build runs on every PR |
| Continuous deployment | Push to `main` auto-deploys to ci; staging/prod are manual promotions |

---

## Branch strategy

```
main (trunk)
  │
  ├── feat/csv-export          ← short-lived (flag: CSV_EXPORT=false in prod)
  ├── feat/ai-suggestions      ← short-lived (flag: AI_SUGGESTIONS=false in prod)
  └── fix/task-sort-order      ← bugfix, no flag needed
```

Rules:
- Branch from `main`, target `main`
- Delete the branch after merge
- Never merge `main` back into a long-lived branch (because there are none)
- If a feature is risky, wrap it in a flag; if it's safe (a bug fix, a refactor), ship it directly

---

## Pipeline overview

The deploy pipeline itself (Railway deploy per service, health checks, step summary, PR comment and label) is the shared reusable workflow in [power-tester/deploy-workflows](https://github.com/power-tester/deploy-workflows), pinned to `@v1`. The workflows in this repository are thin callers that supply the environment, the services and the token.

```
PR opened
  └── pr.yml
        ├── Lint (ESLint)
        ├── Type-check (tsc --noEmit)
        ├── Test (Vitest)
        └── Build (feature-flags → server → client)

Push to main (PR merged)
  ├── deploy-ci.yml
  │     └── deploy-workflows/deploy.yml@v1 (environment: ci)
  │           ├── railway up — server and client, from source, in parallel
  │           ├── Health-check SERVER_URL/health and CLIENT_URL/
  │           ├── GitHub Step Summary
  │           └── Comment on + label the merged PR ("deployed: ci")
  └── build-image.yml
        ├── Build server + client images → ghcr.io, tagged sha-<short> and latest
        └── Refresh the image_tag dropdown in deploy-staging.yml / deploy-production.yml

Manual trigger (staging)
  └── deploy-staging.yml
        ├── Inputs: image_tag (dropdown of recent builds, or "(build from source)") + sha
        ├── resolve-deploy-inputs@v1 — normalize the inputs to a clean tag or SHA
        └── deploy-workflows/deploy.yml@v1 (environment: staging)

Manual trigger + approval (production)
  └── deploy-production.yml
        ├── Same inputs as staging, plus release_notes
        ├── resolve-deploy-inputs@v1
        ├── deploy-workflows/deploy.yml@v1 (environment: production) ← required reviewer gate
        └── Create GitHub Release (v<date>-<short-sha>)
```

See [RAILWAY_DEPLOYMENT.md](RAILWAY_DEPLOYMENT.md) for the Railway setup and the source-vs-image deploy strategies.

---

## Promotion flow

```
Feature complete
    ↓
Open PR → CI passes → Code review → Merge to main
    ↓
Auto-deploy to ci (NODE_ENV=ci, 7 flags on) + images published to GHCR
    ↓
QA on ci — note the image tag (sha-<short>) from the PR comment or build summary
    ↓
Manual: deploy-staging.yml (pick the image tag) → staging (NODE_ENV=staging, 4 flags on)
    ↓
Stakeholder sign-off
    ↓
Manual + approval: deploy-production.yml (same image tag) → production (NODE_ENV=production, 2 flags on)
    ↓
Turn on flag in flags.production.json → merge → auto-deploy to ci → promote to prod
```

Promoting the same pre-built image through staging and production means the bytes that were tested are the bytes that ship. "(build from source)" with a SHA is the fallback when no image exists for the commit.

---

## Releasing a feature

When a feature is fully validated in staging, releasing it to production is just a JSON change. The per-environment files are overlays on `flags.default.json`, so they only list the flags that differ from the default:

```json
// feature-flags/src/config/flags.production.json
{}
```

Change to:

```json
{
  "TASK_PRIORITY": { "enabled": true }
}
```

Merge the PR → ci auto-deploys → promote to staging → approve production. No code changes, no feature branch merges.

---

## Revert strategy

### Revert a bad deploy (code bug)
```bash
git revert <bad-commit-sha>
# Opens a PR, CI runs, merge → auto-deploys to ci → promote
```

Or re-run `deploy-staging.yml` / `deploy-production.yml` with the previous image tag: the dropdown keeps the last ten builds, so rolling back is a redeploy, not a rebuild.

### Disable a feature instantly (flag kill-switch)
```json
// flags.production.json — set to false
{ "TASK_PRIORITY": { "enabled": false } }
```
Merge and promote. The feature disappears without a code rollback.

---

## Environment variable reference

| Variable   | ci                | staging             | production           |
| ---------- | ----------------- | ------------------- | -------------------- |
| `NODE_ENV` | `ci`              | `staging`           | `production`         |
| `PORT`     | `4000`            | `4000`              | `4000`               |
| Flags on   | 7 of 9            | 4 of 9              | 2 of 9               |

Locally, `NODE_ENV=development` turns all 9 flags on. In Railway these values come from the dotenv-vault entry that each environment's `DOTENV_KEY` unlocks; see [RAILWAY_DEPLOYMENT.md](RAILWAY_DEPLOYMENT.md).

---

## GitHub Environments setup

1. Go to **Settings → Environments** in your repo
2. Create three environments: `ci`, `staging`, `production`
3. Add a **Required reviewer** to `production`
4. Per environment, add the secret `RAILWAY_TOKEN` (a Railway project token scoped to that Railway environment)
5. Per environment, add the variables `SERVER_URL` and `CLIENT_URL` (the Railway public domains; used for the health checks and the PR comment link)

Application secrets (`MONGODB_URI`, `JWT_SECRET`, …) are not GitHub secrets: they live encrypted in `.env.vault` and are unlocked at runtime by the `DOTENV_KEY` set in Railway.

The `deploy-production.yml` workflow passes `environment: production` to the shared deploy workflow. Only the deploy job declares the environment, so the required reviewer gates the Railway deploy itself, not the summary or PR comment jobs around it.
