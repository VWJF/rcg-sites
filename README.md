# rcg-sites

Shared **GitHub Actions** pipelines for RCG-style static sites: Hugo → GitHub Pages and/or Docker → Nomad, plus preview cleanup.

This repository is being extracted from [`coastal-science/coastal-science.github.io`](https://github.com/coastal-science/coastal-science.github.io). Active import work lives on [`refactor-from-cs`](https://github.com/VWJF/rcg-sites/tree/refactor-from-cs) (filtered paths only; original commit authors and dates preserved). `main` is the product entrypoint for docs and, later, released workflow tags.

## What you get

| Area | Purpose |
| --- | --- |
| `.github/workflows/` | Entrypoints and reusable deploy workflows |
| `.github/actions/cleanup-pr-nomad-job/` | Composite action: stop/purge Nomad preview jobs |
| `nomad/` | Job templates (`template.nomad`, `decap.template.nomad`) |
| `nginx/` | Site / Decap proxy templates |
| `Dockerfile*`, `docker-compose*`, `entrypoint.sh` | Container build/runtime |

### Workflow entrypoints (on `refactor-from-cs`)

| Workflow | When to use |
| --- | --- |
| `pages-gh.yml` | Default: `push` to `main` + `pull_request` (Pages + Nomad) |
| `manual-deploy.yml` | Actions UI: template / Decap / enable overrides |
| `pages-custom.yml` | Dedicated self-hosted runner (e.g. `["self-hosted","rcg"]`) + trailing disk cleanup |
| `cleanup-pr-nomad-on-delete.yml` | `delete` of a PR branch (backup when `pull_request/closed` is skipped) |
| `pages-sfu.yml` | Legacy SFU monolith (kept for reference; prefer `pages-custom`) |

Reusable pieces (called by the entrypoints): `deploy-github-pages.yml`, `deploy-nomad.yml`.

> **Note:** Until workflows are published from `main` with version tags, treat `refactor-from-cs` as the source of truth for file layout and required settings below.

## Stages (mental model)

GitHub Actions has no GitLab-style `stages:`. Ordering is **`needs:`**. The intended pipeline is:

**build → test (future) → deploy → stop**

Today: build+deploy for Pages and Nomad; stop = PR cleanup / branch-delete cleanup / optional self-hosted runner cleanup.

## Quick start (consumer repo)

1. Ensure the site repo has Hugo content (and themes) as today; this library supplies CI/CD + Nomad/nginx/Docker pieces.
2. Create the GitHub **Environments** and settings in the tables below (names must match).
3. Set **repository** variables `ENABLE_GITHUB_PAGES` and `ENABLE_NOMAD` (see why under [Gates](#gates-enable_github_pages--enable_nomad)).
4. Wire a thin caller (eventually):

```yaml
# .github/workflows/deploy.yml in your site repo (illustrative)
jobs:
  site:
    uses: VWJF/rcg-sites/.github/workflows/pages-gh.yml@<tag-or-sha>
    secrets: inherit
    # with:
    #   runs_on: '["self-hosted","rcg"]'   # optional
```

Until extraction is finished, you can keep using the workflows **in-tree** as coastal-science does, or call a commit on `refactor-from-cs`.

## Gates (`ENABLE_GITHUB_PAGES` / `ENABLE_NOMAD`)

Job-level `if:` conditions read **`vars.ENABLE_*`**. Those must be **repository** (or org) Actions variables — **not** only environment variables.

Environment-scoped vars are available only **after** a job’s `environment:` is applied, so they are invisible to job `if:`. Coastal-science sets both enables at **repo** scope (`true`).

Optional force-on overrides: workflow inputs `enable_github_pages` / `enable_nomad` (manual / `workflow_call`).

Optional runner override for branch-delete cleanup: repo var `RUNS_ON` (label or JSON array, e.g. `["self-hosted","rcg"]`).

## Environments

Create these environments under **Settings → Environments** (names are exact):

| Environment | Used by | Role |
| --- | --- | --- |
| `github-pages` | Pages deploy job | GitHub Pages deployment record + optional Pages-related vars |
| `docker-image` | Docker build, Nomad deploy, Nomad cleanup jobs | Registry/Nomad/CMS config (vars + secrets); may use deployment branch policies |

Coastal-science reference: `docker-image` has a custom deployment branch policy; `github-pages` has none.

Leaf jobs set `environment:` inside the reusable workflows. Caller jobs that use `uses:` **cannot** set `environment:` — that is expected.

## Repository variables

Set under **Settings → Secrets and variables → Actions → Variables**.

| Name | Example / notes | Required for |
| --- | --- | --- |
| `ENABLE_GITHUB_PAGES` | `true` / `false` | Job gate for Pages |
| `ENABLE_NOMAD` | `true` / `false` | Job gate for Nomad + delete cleanup |
| `HUGO_VERSION` | e.g. `0.170.0` | Hugo build |
| `NOMAD_VERSION` | e.g. `2.0.3` (fallback exists in workflows) | Nomad CLI / cleanup action |
| `__SERVICE__` | e.g. `orca` | Image/job naming |
| `RUNS_ON` | optional; `ubuntu-latest` or `["self-hosted","rcg"]` | `cleanup-pr-nomad-on-delete.yml` |

## Environment variables

### `github-pages`

| Name | Purpose |
| --- | --- |
| `HUGO_FORMS` | Form endpoint injected into Hugo (if used) |

Also used from repo or env where referenced: `HUGO_VERSION`, analytics-related vars if you build Pages with the same Hugo settings as Docker.

### `docker-image`

| Name | Purpose |
| --- | --- |
| `__NAMESPACE__` | Nomad namespace (e.g. `apps`) |
| `__DATACENTERS__` | Nomad datacenter list JSON string (e.g. `[\"ruthjoy-fir-cloud\"]`) |
| `__DATACENTER_CLOUD__` | DNS segment for preview URLs (e.g. `fir-cloud`) |
| `__JOB_DRIVER__` | Nomad driver (e.g. `docker`) |
| `__REGISTRY__` | Container registry host (e.g. `ghcr.io`) |
| `__IMAGE_FORCE_PULL__` | optional image pull policy for jobs |
| `HUGO_FORMS` | Form endpoint for image build |
| `HUGO_SERVICES_GOOGLEANALYTICS_ID` | GA id (if used) |
| `OAUTH_GITHUB_CLIENT_ID` | Decap/GitHub OAuth app client id |
| `ORIGINS` | Allowed CORS/origin host list (comma-separated) |
| `CMS_BACKEND_DEBUG` | Decap/CMS debug flag (e.g. `1`) |

Some workflows also reference `HUGO_BASEURL` / legacy `HUGO_GOOGLEANALYTICS` where still present — prefer the names your templates actually use.

## Environment secrets

### `docker-image`

| Name | Purpose |
| --- | --- |
| `NOMAD_SERVER_ADDRESS` | Nomad API URL |
| `NOMAD_SERVER_TOKEN` | Nomad ACL token |
| `NOMAD_REGISTRY_PASSWORD` | Password/token Nomad uses to pull from the registry (not the Actions `docker/login` password) |
| `__RESEARCHER__` | Researcher/DNS label used in Nomad URL templates |
| `__DOMAIN_NAME__` | Production web hostname (main branch) |
| `OAUTH_GITHUB_CLIENT_SECRET` | Decap/GitHub OAuth client secret |

`GITHUB_TOKEN` is provided by Actions for GHCR push login; you do not create it.

### `github-pages`

No secrets required in the coastal-science reference setup (Pages uses `actions/deploy-pages` + `GITHUB_TOKEN` permissions).

## Repository secrets

Coastal-science keeps Nomad/OAuth secrets on the **`docker-image` environment** (not repo-level). Prefer the same so only jobs that select that environment can read them.

## Permissions the workflows expect

Entrypoints request combinations of:

- `contents`, `pages`, `id-token` — GitHub Pages  
- `packages`, `statuses` — GHCR push + PR commit status for Nomad previews  
- `pull-requests` — branch-delete PR lookup  

Org/repo Actions settings must allow the GITHUB_TOKEN enough scope (and Pages must be enabled if you use that path).

## Self-hosted / custom runners

- Pass `runs_on` as a **JSON array string**, e.g. `["self-hosted","rcg"]`, into `pages-gh` / `pages-custom` / `manual-deploy`.
- Use **`pages-custom.yml`** when a **single dedicated** runner must reclaim Docker images and workspace disk after the run.
- Do not enable both `pages-gh` and `pages-custom` on the same push/PR triggers (double deploy).

## Related layout (this repo)

```text
.github/workflows/     # entrypoints + reusable workflows
.github/actions/       # composite actions (Nomad cleanup)
nomad/                 # Nomad job templates
nginx/                 # reverse-proxy / Decap templates
Dockerfile*            # site image
docker-compose*        # local compose
entrypoint.sh          # container entry
```

## Related projects

- Source site: [coastal-science/coastal-science.github.io](https://github.com/coastal-science/coastal-science.github.io)
- Packaging style reference: [VWJF/mirroring](https://github.com/VWJF/mirroring) (`dockerize`)
