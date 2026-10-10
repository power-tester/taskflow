# Demo Script

A step-by-step walkthrough for presenting TaskFlow's trunk-based development and feature toggle system to your team.

**Duration**: ~20 minutes  
**Audience**: Engineering team / stakeholders  
**Prerequisites**: App running locally (`npm run dev:local`) with MongoDB available

---

## 1. Introduction (2 min)

> "We're going to walk through how trunk-based development and feature toggles let us ship continuously without long-lived feature branches or risky big-bang releases."

Key points to set the scene:
- Single trunk (`main`) — all engineers commit here
- Feature flags replace feature branches for incomplete or risky work
- Four environments with different flag states: development (local) → ci → staging → production

---

## 2. Show the flag matrix (3 min)

Open `feature-flags/src/config/` in your editor. Show all four JSON files side by side.

> "This is the entire flag configuration. `flags.default.json` is the registry; the other files are per-environment overlays. In development everything is on — that's our full feature set. In ci, dark mode and AI suggestions are off because they're still in progress. In staging we have four flags. In production, only two."

Key file to show: `feature-flags/src/types.ts`

> "The `Provider` interface is the contract. Right now we use plain JSON files. If we want to move to LaunchDarkly, we implement this interface once and swap this single line in `index.ts`."

Show `feature-flags/src/index.ts`:
```ts
const provider: Provider = new ConfigFileProvider();
```

> "One line change. Nothing else in the codebase needs to know."

---

## 3. Live flag demo — local vs production behaviour (5 min)

**Step 1** — Start the app with all flags on (local):

```bash
# In packages/server/.env
NODE_ENV=development
```

Log in, show:
- Task priority field in the task form
- Labels on task cards
- Comments section on a task
- Navigation bar shows "Dashboard"
- Dark mode toggle visible
- Notification bell visible

**Step 2** — Simulate production (2 flags only):

```bash
# Change in packages/server/.env
NODE_ENV=production
```

Restart the server (`Ctrl+C` then `npm run dev:server` in Terminal 1) and refresh the client.

Show what disappeared:
- Priority field gone from task form
- Labels gone from task cards
- Comments tab gone
- Dashboard link gone from nav
- Dark mode toggle gone
- Notification bell gone

> "Same codebase, same deployment, same commit — just a different environment variable. Production users only see the two stable features."

**Step 3** — Show the API enforces it too:

```bash
curl http://localhost:4000/api/tasks/export/csv
# With NODE_ENV=production → 404
# With NODE_ENV=development → CSV download
```

> "The server returns 404 — the endpoint doesn't exist from the client's perspective. It's not 403 Forbidden, it's invisible."

---

## 4. Show how a feature branch works (5 min)

> "Let's say we're building AI suggestions. We commit to main behind a flag."

Walk through the flow:

1. Developer adds code behind `isEnabled('AI_SUGGESTIONS')` check
2. `flags.production.json` — leaves `AI_SUGGESTIONS` at its default, `enabled: false`
3. Code ships in every deploy to ci, staging, production — but the feature is invisible
4. When ready, add `"AI_SUGGESTIONS": { "enabled": true }` to `flags.production.json`, open a PR, merge
5. Feature appears in production with the next deploy

Show the React side:

```tsx
// TaskForm.tsx
const showAI = useFeatureFlag('AI_SUGGESTIONS');
{showAI && <AISuggestionPanel />}
```

Show the server side:

```ts
router.post('/suggest', featureGate('AI_SUGGESTIONS'), suggestHandler);
```

> "Even if someone calls the API directly, they get a 404. The flag enforces the boundary on both sides."

---

## 5. Show the CI/CD pipeline (3 min)

Open `.github/workflows/`:

- `pr.yml` — show the PR checks: lint, typecheck, test, build
- `deploy-ci.yml` — show it triggers on push to main and contains no deploy logic of its own: it calls the shared deploy workflow with `environment: ci`
- `build-image.yml` — also on push to main; publishes the server and client images to GHCR and refreshes the image dropdown in the two manual workflows
- `deploy-staging.yml` — show the `image_tag` dropdown of recent builds and the `resolve` job
- `deploy-production.yml` — show the `environment: production` line and the release job

> "Production has a GitHub Environment with a required reviewer. No code reaches production without a human approving the deployment in GitHub Actions."

Open [power-tester/deploy-workflows](https://github.com/power-tester/deploy-workflows) and show `deploy.yml`:

> "All three deploy targets, and every other repository we deploy this way, use the same steps — deploy each service to Railway, health check it, write a summary, comment on and label the PR. It lives in one shared repository, and each product repo pins a major version. Deploy behaviour changes there; environments, services and tokens stay here."

Show the staging run's PR comment and the `deployed: staging` label:

> "Every deploy reports back to the PR that introduced the commit, so the trail from merge to environment is in one place."

---

## 6. Releasing a feature (2 min)

> "When we decide to release TASK_PRIORITY to production, it's a one-line JSON change — no feature branch merge, no cherry-pick, no code freeze."

```json
// Before — flags.production.json
{}

// After
{ "TASK_PRIORITY": { "enabled": true } }
```

> "Open a PR, CI runs, merge, auto-deploys to ci, we verify, promote to staging, promote to production with approval. The feature is live."

---

## 7. Rollback story (1 min)

> "What if something goes wrong? Two options:"

**Kill-switch** (flag issue):
```json
{ "TASK_PRIORITY": { "enabled": false } }
```
Merge, promote — feature off in minutes without a code deployment.

**Redeploy the previous image** (bad build):
Re-run the deploy workflow and pick the previous `sha-` tag from the dropdown. Nothing is rebuilt; the image that was running before is deployed again.

**Code revert** (actual bug):
```bash
git revert <sha>
```
PR → CI → merge → promote. Standard git flow, no scary rollback commands.

---

## Q&A prompts

- *"How do we handle flags for A/B testing?"* — The `Provider` interface can return per-user values; swap to a provider that supports targeting rules.
- *"What stops a developer from committing with a flag left on in production JSON?"* — Code review + CI. The PR diff will show the JSON change clearly.
- *"How do we clean up old flags?"* — Delete from `types.ts` (TypeScript will error everywhere it's used), then remove all usages, then remove from JSON files.
