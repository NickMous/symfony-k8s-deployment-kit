# symfony-k8s-deployment-kit

Shared Kubernetes manifests, GitHub Actions workflows, and the nginx image
that powers Symfony deployments. Projects consume this repo via a Git ref;
updates here flow into projects on their next ref bump.

## Layout

```
base/                       # kustomize: always included
  deployment.yaml           # app Deployment (nginx + php-fpm), 4 init containers
  service.yaml              # app Service
  configmap.yaml            # app-config (APP_ENV, TZ, hostnames)
  nginx-default.conf        # nginx server block (X-Accel-Redirect, gzip, brotli)
  kustomization.yaml

components/                 # kustomize: opt-in per project
  postgres/                 # Postgres StatefulSet + Service
  redis/                    # Redis Deployment + Service + PVC
  messenger/                # Symfony Messenger worker
  storage/                  # shared ReadWriteMany PVC

docker/
  nginx/Dockerfile          # the shared nginx + ngx_brotli image

.github/workflows/
  build-nginx.yaml          # builds + publishes the shared nginx image (runs here)
  deploy-production.yaml    # REUSABLE — projects call via `uses:`
  deploy-staging.yaml       # REUSABLE — projects call via `uses:`
  cleanup-staging.yaml      # REUSABLE — projects call via `uses:`

examples/
  overlay-production/       # consume the template for production
  overlay-staging/          # consume for staging, with per-branch generator
  workflows/                # caller stubs — copy into your project's .github/workflows/
    deploy-production.yaml
    deploy-staging.yaml
    cleanup-staging.yaml
```

## Naming convention

Generic resource names — `app`, `app-postgres`, `app-redis`, `app-messenger`,
`app-storage`, `app-config`, `app-secret`, `app-postgres-secret`,
`app-branch-secret`. The project's overlay sets `namePrefix:` (e.g.
`zcdb-`) and kustomize rewrites every in-graph reference.

**Always include the trailing dash in `namePrefix`** — kustomize uses it as
a literal string with no separator logic. `namePrefix: zcdb` produces
`zcdbapp`; `namePrefix: zcdb-` produces `zcdb-app`.

Image placeholders: `app-image` (the project's own Symfony build) and
`app-nginx-image` (point this at the shared nginx image from this repo).

## Shared nginx image

The nginx Dockerfile in `docker/nginx/` is built by `.github/workflows/build-nginx.yaml`
and published as a multi-arch image to this repo's GHCR package. All Symfony
projects consuming this template point `app-nginx-image` at the same image:

```yaml
images:
  - name: app-nginx-image
    newName: ghcr.io/nickmous/symfony-k8s-deployment-kit
    newTag: nginx-latest         # or nginx-sha-<commit> for reproducibility
```

Projects no longer need to build or maintain their own nginx image. Changes
to nginx config (`base/nginx-default.conf`) flow in via the kustomize ref bump;
changes to the image itself (e.g. an nginx upgrade) flow in via tag bump.

## Consuming the template

Once this repo is published, an overlay looks like:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zcdb-production
namePrefix: zcdb-

resources:
  - github.com/nickmous/symfony-k8s-deployment-kit//base?ref=v1.0.0

components:
  - github.com/nickmous/symfony-k8s-deployment-kit//components/postgres?ref=v1.0.0
  - github.com/nickmous/symfony-k8s-deployment-kit//components/redis?ref=v1.0.0
  - github.com/nickmous/symfony-k8s-deployment-kit//components/messenger?ref=v1.0.0
  - github.com/nickmous/symfony-k8s-deployment-kit//components/storage?ref=v1.0.0
```

Each project pins its own `?ref=`, so a change here doesn't force every
project to upgrade at once. See `examples/overlay-production/` for a full
overlay you can build with:

```bash
kubectl kustomize examples/overlay-production
```

## Required overlay pieces

The base/components don't know the project's prefix, so the overlay must:

### 1. Set `namePrefix` with trailing dash

```yaml
namePrefix: zcdb-
```

### 2. Wire hostnames via `replacements`

Kustomize's transformer order applies `namePrefix` *after* `replacements`
declared in base/components, so hostname wiring must live in the overlay
where it runs after the prefix is applied:

```yaml
replacements:
  - source: { kind: Service, name: app, fieldPath: metadata.name }
    targets:
      - select: { kind: ConfigMap, name: app-config }
        fieldPaths: [data.APP_SERVICE_HOST]
  - source: { kind: Service, name: app-postgres, fieldPath: metadata.name }
    targets:
      - select: { kind: ConfigMap, name: app-config }
        fieldPaths: [data.POSTGRES_HOST, data.DATABASE_HOST]
        options: { create: true }
  - source: { kind: Service, name: app-redis, fieldPath: metadata.name }
    targets:
      - select: { kind: ConfigMap, name: app-config }
        fieldPaths: [data.REDIS_HOST]
        options: { create: true }
```

Skip entries for components you didn't enable.

### 3. Supply secrets

The base deployment references three secrets by their generic names:

- `app-secret` — always required (APP_SECRET, DATABASE_URL, …)
- `app-postgres-secret` — required when the postgres component is enabled
  (`username` + `password` keys)
- `app-branch-secret` — optional, used by the per-branch staging generator

Whether you bring these in via SOPS, External Secrets, or another tool, they
must be present in `resources:` so `namePrefix` rewrites the deployment's
references.

### 4. Set per-project literals

```yaml
configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - TZ=Europe/Amsterdam
      - DEFAULT_URI=https://example.com
      - POSTGRES_DB=myapp        # required if postgres component is enabled
```

### 5. Supply ingress / certificate / middleware

Project-specific (domain names, TLS issuer, basic-auth allowlists). The
template ships placeholders in the example overlays — copy and adapt.

## Init container ordering

The app Deployment's init containers run in this order:

1. `wait-for-postgres` — no-op when `POSTGRES_HOST` config key is absent
2. `copy-public-files` — copies the app image's `/public` into a shared volume
3. `run-migrations` — `doctrine:migrations:migrate`
4. `cache-warmup` — `cache:clear`, `cache:warmup`, `importmap:install`

**If you strategic-merge any of these, list all four names in your patch**
even if you only modify one. Without the pin, kustomize moves the patched
item to position 0 and breaks startup. See
`examples/overlay-production/patches/cache-warmup-extras.yaml`.

## Per-branch staging

The deploy-staging workflow generates a per-branch kustomization in the
fleet-infra repo by running `envsubst` on
`examples/overlay-staging/generator/kustomization.yaml.tmpl` (template
placeholders: `${BRANCH_SLUG}`, `${COMMIT_SHA}`) and `sed`-rewriting domain
hosts in `ingress.yaml` / `certificate.yaml` from `staging.DOMAIN` to
`{slug}.staging.DOMAIN`.

Two ways the generated kustomization can find the base + components:

- **Mode A (public template repo):** generator's `resources:` references
  `github.com/nickmous/symfony-k8s-deployment-kit//base?ref=…`. Flux/kustomize fetches it
  directly. Simpler.
- **Mode B (private template):** workflow `cp -r`'s `k8s/base/` into the
  fleet repo so kustomize finds it locally. Use this when the template is
  private and kustomize can't reach it across repos.

The bundled `generator/kustomization.yaml.tmpl` shows both modes with one
commented out.

## Project workflows

The deploy logic lives here as **reusable workflows** — projects invoke them
via `uses:` just like a third-party action. No copy-paste; bumping the `@ref`
upgrades the deploy logic across all projects.

Caller stubs in `examples/workflows/` show what each project needs:

```yaml
# .github/workflows/deploy-production.yaml in the project repo
name: Deploy to Production
on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      version:
        required: true

jobs:
  deploy:
    # Reusable workflows can only RESTRICT caller permissions — never
    # expand them. Declare the scopes the reusable workflow needs.
    permissions:
      contents: read
      packages: write
    uses: nickmous/symfony-k8s-deployment-kit/.github/workflows/deploy-production.yaml@v1.0.0
    with:
      version: ${{ github.event.inputs.version || github.ref_name }}
      prod-url: https://example.com
    secrets: inherit
```

**Two gotchas to know:**

1. **Permissions cascade**: a reusable workflow inherits the caller's
   `GITHUB_TOKEN` permissions and can only further restrict them. If the
   caller doesn't grant `packages: write`, the reusable workflow can't
   either — the run fails with `startup_failure` and no log output. Set
   the needed permissions at the caller job level (see stubs).
2. **`inputs` context**: only exists for `workflow_dispatch` /
   `workflow_call` triggers. Referencing `${{ inputs.x }}` on a `push` or
   `release` trigger fails workflow startup. Use
   `${{ github.event.inputs.x || '' }}` instead — it's always defined.

### Reusable workflow inputs

`deploy-production.yaml`:

| Input | Required | Description |
| --- | --- | --- |
| `version` | yes | Image tag to build (e.g. `v1.0.0`) |
| `prod-url` | no | Primary production URL (shown in environment + summary) |
| `sentry-org`, `sentry-project`, `sentry-url` | no | Leave blank to skip Sentry release tracking |

`deploy-staging.yaml`:

| Input | Required | Description |
| --- | --- | --- |
| `app-name` | yes | Used in fleet repo path `apps/<app-name>/` |
| `fleet-repo` | yes | `OWNER/repo` of the Flux fleet repo |
| `staging-domain-base` | yes | Base domain for branch URLs (`<slug>.<staging-domain-base>`) |
| `staging-domains` | yes | Comma-separated `staging.X` hosts to rewrite per branch |
| `branch` | no | Override the ref the workflow ran on |
| `sentry-*` | no | As above |

`cleanup-staging.yaml`:

| Input | Required | Description |
| --- | --- | --- |
| `app-name` | yes | As above |
| `fleet-repo` | yes | As above |
| `branch-name` | yes | The branch / environment name to remove |

### Required secrets in the project repo

| Secret | Required for | Notes |
| --- | --- | --- |
| `GITHUB_TOKEN` | all | Provided automatically |
| `FLEET_REPO_TOKEN` | staging + cleanup | Fine-grained PAT with `Contents: write` on the fleet repo |
| `SENTRY_AUTH_TOKEN` | optional | Only if `sentry-org` is set |
| `FLUX_WEBHOOK_URL` | optional | Activated via `vars.FLUX_WEBHOOK_ENABLED=true` |

`secrets: inherit` in the caller passes them all through; no need to list
each explicitly.

## Versioning

Tag releases (`v1.0.0`, `v1.1.0`, …). Projects pin via `?ref=vX.Y.Z`.
Breaking changes (renames, structural shifts) get a major bump; additive
changes get a minor bump.
