# GitHub App Service Token — Decision Document

## Context

The `build-image.yml` workflow includes an `update-deploy-options` job that, after
every successful image build, patches the `image_tag` dropdown options in
`deploy-staging.yml` and `deploy-production.yml` and commits the updated files
directly to `main`.

Committing to files under `.github/workflows/**` requires a token with the
`workflows` scope. GitHub's built-in `GITHUB_TOKEN` explicitly **cannot** be
granted this scope from within a workflow — it is a hard platform restriction
regardless of what is in the `permissions:` block.

This document records the decisions made and the reasoning behind each one.

---

## Decision 1 — Use a GitHub App installation token (not a PAT or machine user)

### Options considered

#### Option A — Personal Access Token (PAT) belonging to a human user ❌

A fine-grained PAT with `Contents: read/write` + `Workflows: read/write` can
push workflow files. The PAT owner's account must also be added to the branch
ruleset bypass list so the direct push to `main` is allowed.

**Why we rejected it:**

| Concern | Detail |
|---|---|
| Tied to a person | If the engineer who owns the PAT leaves the organisation, the token is revoked and the pipeline silently breaks |
| Long-lived credential | Fine-grained PATs can be set to never expire; a leaked token provides persistent write access to workflows |
| Org seat consumption | The bypass list entry is a named user; on paid plans that user consumes an org seat |
| Audit trail | Commits appear as the human's account, not a service identity — hard to distinguish automated from manual changes |

---

#### Option B — Dedicated machine/service user account ❌

A separate GitHub account (e.g. `taskflow-ci-bot`) generates PATs like a real
user and is added to the bypass list.

**Why we rejected it:**

| Concern | Detail |
|---|---|
| Still a user account | Consumes an org seat on paid plans |
| Long-lived PAT | Same rotation/leak risk as Option A |
| GitHub ToS | GitHub's Terms of Service discourage "bot" personal accounts; GitHub Apps are the supported mechanism |

---

#### Option C — GitHub App installation token ✅ **Chosen**

A GitHub App is a first-class service identity on GitHub. It is not a user
account, it does not consume an org seat, and it generates **short-lived
installation tokens** (valid for 1 hour) at runtime using a private key.

**Why we chose it:**

| Property | Detail |
|---|---|
| Service identity | The app has its own bot identity (`your-app-name[bot]`) — fully decoupled from any human account |
| Short-lived tokens | Tokens are minted per-run and expire in 1 hour — a leaked token has a tiny blast radius |
| Minimal permissions | The app installation is granted only `Contents: read/write` and `Workflows: read/write` — nothing else |
| No seat consumption | GitHub Apps do not count as org members |
| Clean audit trail | Commits appear as `your-app-name[bot]` — immediately identifiable as automated |
| Bypass list entry | The bypass list entry is the App, not a user — survives team changes |

---

## Decision 2 — Create the App at organisation level (not repository level)

### Options considered

#### Option A — Repo-level GitHub App ❌

The App is created under a personal account or scoped to a single repository.

**Why we rejected it:**

| Concern | Detail |
|---|---|
| Does not scale | A separate App must be created and maintained for every repository that needs the same capability |
| Fragmented ownership | App settings, private keys, and bypass list entries are scattered across individual repos |
| Harder to audit | No central view of which repos have automation write access to workflows |
| Rotation cost | Rotating the private key requires updating secrets in every repo individually |

---

#### Option B — Organisation-level GitHub App ✅ **Chosen**

The App is created under the GitHub organisation
(`github.com/organizations/<org>/settings/apps`).

**Why we chose it:**

| Property | Detail |
|---|---|
| Single app for all repos | Install once at org level; choose which repos it has access to — no duplication |
| Centralised management | Org admins own the App, its private key, and its installation — independent of any individual repo or engineer |
| Selective repo access | Installation scope can be set to **Selected repositories** — each repo must be explicitly granted access, enforcing least-privilege |
| Single key rotation point | Rotating the private key means updating one org-level secret; all repos pick it up automatically |
| Consistent audit trail | All automated commits across every repo in the org share the same `[bot]` identity — easy to filter and audit |

---

## Decision 3 — Store credentials as organisation-level secrets (not repo-level)

### Options considered

#### Option A — Repository secrets ❌

`WORKFLOW_APP_ID` and `WORKFLOW_APP_PRIVATE_KEY` stored under each repo's own
Settings → Secrets → Actions.

**Why we rejected it:**

| Concern | Detail |
|---|---|
| Duplication | Every repo that uses the App must store the same two values |
| Rotation cost | Key rotation requires visiting every repo and updating the secret manually |
| Inconsistency risk | Repos may fall out of sync if a rotation is missed |

---

#### Option B — Organisation secrets scoped to selected repositories ✅ **Chosen**

`WORKFLOW_APP_ID` stored as an org-level variable, `WORKFLOW_APP_PRIVATE_KEY`
stored as an org-level secret, both scoped to the repositories that need them
(Settings → Secrets and variables → Actions at the org level → **Selected repositories**).

**Why we chose it:**

| Property | Detail |
|---|---|
| Single source of truth | One copy of the App ID and private key for the entire org |
| Instant rotation | Update the org secret once; every workflow that references it gets the new value on the next run |
| Controlled access | Org admins choose which repos can read the secret — repos are opted in explicitly |
| Consistent workflow files | Every repo's `build-image.yml` references the same `vars.WORKFLOW_APP_ID` / `secrets.WORKFLOW_APP_PRIVATE_KEY` names — no per-repo variation |

---

## How It Works in the Workflow

```yaml
- name: Generate GitHub App token
  id: app-token
  uses: actions/create-github-app-token@v1
  with:
    app-id: ${{ vars.WORKFLOW_APP_ID }}
    private-key: ${{ secrets.WORKFLOW_APP_PRIVATE_KEY }}

- name: Checkout main
  uses: actions/checkout@v4
  with:
    ref: main
    token: ${{ steps.app-token.outputs.token }}
```

`actions/create-github-app-token` calls the GitHub API at runtime to exchange
the app's private key for a short-lived installation token scoped to the current
repository. That token is then used by `actions/checkout` and by the subsequent
`git push`.

---

## Setup Instructions

### 1 — Create the GitHub App at org level

1. Go to `github.com/organizations/<your-org>/settings/Developer Setting/GitHub Apps`
2. Click **New GitHub App**
3. Fill in:
   - **App name**: e.g. `<org>-workflow-updater`
   - **Homepage URL**: your org URL, e.g. `https://github.com/<your-org>` — since the App is owned by and serves the whole org, not a single repo
   - **Webhook**: uncheck "Active" (not needed)
4. Under **Repository permissions**, set:
   - **Contents** → `Read and write`
   - **Workflows** → `Read and write`
5. Set **Where can this GitHub App be installed?** → `Only on this account`
6. Click **Create GitHub App**

### 2 — Generate a private key

On the App's settings page → **Generate a private key** → download the `.pem` file.

### 3 — Install the App on selected repositories

App settings page → **Install App** → select your organisation → under
**Repository access** choose **Only select repositories** → add each repo that
needs it → click **Install**.

### 4 — Add the App to each repo's branch ruleset bypass list

For each repository: **Settings** → **Rules** → **Rulesets** → `main` →
**Bypass list** → **Add bypass** → search for your App by name → **Add**.

### 5 — Store credentials as org-level secrets

Navigate to your org → **Settings** → **Secrets and variables** → **Actions**.

| Type | Name | Value | Repository access |
|---|---|---|---|
| Variable | `WORKFLOW_APP_ID` | The App's numeric ID (shown on the App settings page) | Selected repositories |
| Secret | `WORKFLOW_APP_PRIVATE_KEY` | Full contents of the downloaded `.pem` file | Selected repositories |

For each, click **Selected repositories** and add every repo that will use the App.

### 6 — Note the App's bot username (for git config)

The committer identity used in the workflow is:

```bash
git config user.name  "<org>-workflow-updater[bot]"
git config user.email "<org>-workflow-updater[bot]@users.noreply.github.com"
```

The email format `<app-slug>[bot]@users.noreply.github.com` is the canonical
no-reply address GitHub assigns to App bot users. Replace `<org>-workflow-updater`
with your App's slug (visible in the App's settings URL).

---

## Loop-Avoidance

Pushes made with a GitHub App token **can** re-trigger `push:` workflows.
The infinite build loop is prevented by including `[skip ci]` in the commit
message:

```bash
git commit -m "chore: refresh deploy image_tag options [skip ci]"
```

GitHub Actions skips all workflow runs triggered by pushes whose HEAD commit
message contains `[skip ci]`. The same marker works on GitLab CI (`[skip ci]`,
`[ci skip]`, `[no ci]`).
