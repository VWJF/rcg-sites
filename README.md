# rcg-sites

CI/CD for Hugo sites that publish to **GitHub Pages** and/or run as a **Docker image on Nomad** (including optional **Decap CMS**).

This repository is being extracted from [`coastal-science/coastal-science.github.io`](https://github.com/coastal-science/coastal-science.github.io). The filtered import (workflows, Nomad/nginx/Docker) is on [`refactor-from-cs`](https://github.com/VWJF/rcg-sites/tree/refactor-from-cs). `main` holds product docs and will later hold released tags.

---

## What it does

You get two deploy paths. Turn each on or off with repository variables `ENABLE_GITHUB_PAGES` and `ENABLE_NOMAD`.

### 1. GitHub Pages

Builds the Hugo site and deploys it with GitHub Pages.

- **Default branch (`main`):** every push rebuilds and publishes the site for that commit (the live Pages site tracks the latest commit on `main`).
- **Pull requests:** each new commit on a PR also builds and deploys a Pages preview for that PR (so reviewers can open the site without merging). Closing the PR stops further Pages publishes for that draft.

### 2. Docker + Nomad

Builds a Docker image of the site, pushes it to the container registry, and runs it as a Nomad job.

- **Default branch (`main`):** production (or “latest”) Nomad deployment for that commit.
- **Pull requests:** a **preview** Nomad job per PR (unique URL / job id). When you push again to the same PR, the new preview is deployed and **older preview jobs for that PR are stopped**. When the PR is closed without merging—or the head branch is deleted—those preview jobs are **cleaned up**. After a merge to `main`, leftover preview jobs for that PR are cleaned up as well.
- Preview deploy status is reported back onto the PR commit when possible.

### 3. Decap CMS (optional)

Nomad ships with a **Decap CMS** job template (`nomad/decap.template.nomad`) plus nginx/Decap-oriented config under `nginx/`. You can select the Decap template (or let branch-based “auto” logic choose) via `manual-deploy` / workflow inputs (`nomad_job_template`, `use_decap`). That path is for editorial CMS previews/environments that need Decap’s Git Gateway–style flow, not only a static site.

### Lifecycle at a glance

| Event | GitHub Pages | Nomad |
| --- | --- | --- |
| Push to `main` | Build + publish latest site | Build image + deploy production job; clean up merged PR previews |
| PR opened / new commits | Build + publish PR preview | Build image + deploy PR preview; replace prior previews for that PR |
| PR closed (not merged) | No new publish | Stop/purge that PR’s preview jobs |
| PR head branch deleted | — | Backup cleanup (if the close run was skipped) |

Both paths are independent: you can run Pages only, Nomad only, or both.

---

## What’s in the repo

| Area | Purpose |
| --- | --- |
| `.github/workflows/` | Entrypoints and reusable build/deploy/cleanup workflows |
| `.github/actions/cleanup-pr-nomad-job/` | Stop/purge Nomad preview jobs by id/prefix |
| `nomad/template.nomad` | Standard site Nomad job |
| `nomad/decap.template.nomad` | Nomad job for Decap CMS |
| `nginx/` | Reverse-proxy / Decap templates used by the container |
| `Dockerfile*`, `docker-compose*`, `entrypoint.sh` | Image build and local/runtime helpers |

### Workflow entrypoints (see `refactor-from-cs`)

| Workflow | Role |
| --- | --- |
| `pages-gh.yml` | Everyday automation: `push` to `main` + pull requests (Pages and/or Nomad) |
| `manual-deploy.yml` | Run from the Actions tab (pick Nomad template, Decap override, enables) |
| `pages-custom.yml` | Same pipelines on a dedicated self-hosted runner, with disk/image cleanup afterward |
| `cleanup-pr-nomad-on-delete.yml` | Cleans Nomad previews when a PR branch is deleted |
| `pages-sfu.yml` | Legacy SFU-style workflow (reference; prefer `pages-custom` for self-hosted) |

Reusable workflows called by those entrypoints: `deploy-github-pages.yml`, `deploy-nomad.yml`.

> Until version tags are published from `main`, use `refactor-from-cs` (or an in-tree copy in your site repo) as the source of the workflow files.

## Stages (mental model)

GitHub Actions has no GitLab-style `stages:`. Order comes from **`needs:`**. The intended shape is:

**build → test (future) → deploy → stop**

Today: build+deploy for Pages and Nomad; **stop** means PR/branch preview cleanup (and optional self-hosted runner cleanup).

## Quick start (consumer site)

1. Keep Hugo content and themes in your site repository.
2. Create the GitHub **Environments** and variables/secrets below (names must match).
3. Set repository variables `ENABLE_GITHUB_PAGES` and/or `ENABLE_NOMAD` to `true`.
4. Call the entrypoint (illustrative once tagged releases exist):

```yaml
jobs:
  site:
    uses: VWJF/rcg-sites/.github/workflows/pages-gh.yml@<tag-or-sha>
    secrets: inherit
```

---

## Gates (`ENABLE_GITHUB_PAGES` / `ENABLE_NOMAD`)

Job-level `if:` reads **`vars.ENABLE_*`**. Define them as **repository** (or org) Actions variables—not only on an environment.

Environment variables are available only after a job selects `environment:`, so they cannot gate whether the job starts. Coastal-science sets both enables at **repo** scope.

Optional force-on: inputs `enable_github_pages` / `enable_nomad` on manual / `workflow_call` runs.

Optional: repo var `RUNS_ON` (e.g. `["self-hosted","rcg"]`) for the branch-delete cleanup workflow.

## Environments

**Settings → Environments** (exact names):

| Environment | Used by | Role |
| --- | --- | --- |
| `github-pages` | Pages deploy | Pages deployment + related vars |
| `docker-image` | Docker build, Nomad deploy & cleanup | Registry, Nomad, Decap/CMS vars and secrets |

Jobs that call a reusable workflow cannot set `environment:` themselves; the leaf jobs inside `deploy-*.yml` do.

## Repository variables

**Settings → Secrets and variables → Actions → Variables**

| Name | Notes |
| --- | --- |
| `ENABLE_GITHUB_PAGES` | `true` / `false` — Pages path |
| `ENABLE_NOMAD` | `true` / `false` — Nomad path + delete cleanup |
| `HUGO_VERSION` | e.g. `0.170.0` |
| `NOMAD_VERSION` | e.g. `2.0.3` (workflows have a fallback) |
| `__SERVICE__` | Service/job name prefix (e.g. `orca`) |
| `RUNS_ON` | Optional runner label or JSON array for delete cleanup |

## Environment variables

### `github-pages`

| Name | Purpose |
| --- | --- |
| `HUGO_FORMS` | Form endpoint for Hugo (if used) |

### `docker-image`

| Name | Purpose |
| --- | --- |
| `__NAMESPACE__` | Nomad namespace |
| `__DATACENTERS__` | Nomad datacenters list as a **JSON array string**. Square brackets and escaped quotes are required, e.g. `[\"researcher-fir-cloud\"]` (stored/displayed that way in GitHub Actions variables). |
| `__DATACENTER_CLOUD__` | DNS segment for preview hostnames |
| `__JOB_DRIVER__` | e.g. `docker` |
| `__REGISTRY__` | e.g. `ghcr.io` |
| `__IMAGE_FORCE_PULL__` | Optional pull policy |
| `HUGO_FORMS` | Form endpoint in the image build |
| `HUGO_SERVICES_GOOGLEANALYTICS_ID` | Analytics (if used) |
| `OAUTH_GITHUB_CLIENT_ID` | Decap / GitHub OAuth app id |
| `ORIGINS` | Allowed origins (comma-separated) |
| `CMS_BACKEND_DEBUG` | Decap/CMS debug flag |

## Environment secrets

### `docker-image`

| Name | Purpose |
| --- | --- |
| `NOMAD_SERVER_ADDRESS` | Nomad API URL |
| `NOMAD_SERVER_TOKEN` | Nomad ACL token |
| `NOMAD_REGISTRY_PASSWORD` | Credential Nomad uses to **pull** the image |
| `__RESEARCHER__` | Researcher / DNS label in Nomad URLs |
| `__DOMAIN_NAME__` | Production hostname on `main` |
| `OAUTH_GITHUB_CLIENT_SECRET` | Decap / GitHub OAuth secret |

`GITHUB_TOKEN` is provided by Actions for pushing to GHCR; you do not create it.

### `github-pages`

No extra secrets in the coastal-science reference setup.

Prefer putting Nomad/OAuth secrets on **`docker-image`**, not as repo-level secrets, so only jobs that use that environment can read them.

## Permissions

Workflows typically need `contents`, `pages`, `id-token` (Pages), `packages` (GHCR), `statuses` (Nomad preview commit status), and `pull-requests` (find PR on branch delete).

## Self-hosted runners

- Pass `runs_on` as a JSON array string, e.g. `["self-hosted","rcg"]`.
- Use **`pages-custom.yml`** on a **dedicated** runner when you must free Docker images and workspace disk after each run.
- Do not run both `pages-gh` and `pages-custom` on the same push/PR triggers (duplicate deploys).

## Related

- Source site: [coastal-science/coastal-science.github.io](https://github.com/coastal-science/coastal-science.github.io)
- Packaging reference: [VWJF/mirroring](https://github.com/VWJF/mirroring) (`dockerize`)
