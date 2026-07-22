# Coldcluster CI/CD

CI/CD pipeline for [BluelightNG/coldcluster-folawej-](https://github.com/BluelightNG/coldcluster-folawej-), running on a separate GitHub account to avoid the org's Actions billing block.

## Architecture

```
BluelightNG/coldcluster-folawej-  (source repo)
        |
        |  repository_dispatch / workflow_dispatch
        v
FeezyHendrix/coldcluster-cicd     (this repo)
   ├── ci.yml      → checkout source → test/build all 7 apps in parallel
   └── deploy.yml  → SSH to VPS → git pull → docker compose build + up → health check
```

## Apps tested

| Job | App | What it checks |
|-----|-----|----------------|
| gravas-test | Go backend | `go vet` + `go build` + `go test -race` |
| sentry-build | Admin panel (React) | `tsc --noEmit` + `vite build` |
| mobile-build | Customer app (Ionic React) | `npm run build` |
| terminal-typecheck | Coin kiosk (Electron) | `tsc --noEmit` |
| watcher-typecheck | Staff kiosk (Electron) | `npm run typecheck` |
| watchtower-build | Audit dashboard (React) | `pnpm build` |
| compose-validate | Docker Compose | `docker compose config` |

## Required secrets

Set these in **this repo's** Settings → Secrets and variables → Actions:

| Secret | Description |
|--------|-------------|
| `SOURCE_REPO_TOKEN` | GitHub PAT with `repo` scope on `BluelightNG/coldcluster-folawej-` (for checkout) |
| `VPS_SSH_KEY` | ed25519 private key for the deploy user on the VPS |
| `VPS_KNOWN_HOSTS` | `ssh-keyscan 161.97.127.74` output |
| `VPS_HOST` | `161.97.127.74` |
| `VPS_USER` | `folawejv2` |

## How to trigger

### Manual (workflow_dispatch)

```bash
# Run CI on a specific branch
gh workflow run ci.yml --repo FeezyHendrix/coldcluster-cicd -f ref=master

# Deploy a specific ref to production
gh workflow run deploy.yml --repo FeezyHendrix/coldcluster-cicd -f ref=prod -f environment=production

# Deploy to staging
gh workflow run deploy.yml --repo FeezyHendrix/coldcluster-cicd -f ref=master -f environment=staging
```

### Automatic (from the main repo)

Add a webhook workflow to `BluelightNG/coldcluster-folawej-/.github/workflows/trigger-cicd.yml`:

```yaml
name: Trigger CI/CD

on:
  push:
    branches: [master, prod]
  pull_request:
    branches: [master, prod]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.CICD_DISPATCH_TOKEN }}
          repository: FeezyHendrix/coldcluster-cicd
          event-type: ci-trigger
          client-payload: >
            {"ref": "${{ github.head_ref || github.ref_name }}",
             "sha": "${{ github.sha }}"}

  deploy:
    if: github.ref == 'refs/heads/prod' && github.event_name == 'push'
    needs: trigger
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.CICD_DISPATCH_TOKEN }}
          repository: FeezyHendrix/coldcluster-cicd
          event-type: deploy-trigger
          client-payload: >
            {"ref": "prod", "sha": "${{ github.sha }}", "environment": "production"}
```

For this, add a `CICD_DISPATCH_TOKEN` secret to the **main repo** — a PAT from `FeezyHendrix` with `repo` scope on this CI/CD repo.

## VPS deploy flow

1. SSH into `folawejv2@161.97.127.74`
2. `cd /opt/coldcluster && git fetch && git checkout <ref> && git reset --hard origin/<ref>`
3. `bash deploy/scripts/deploy.sh` (docker compose build + up)
4. Health checks: `api.folawej.com/health`, `admin.folawej.com`, `app.folawej.com`

## Rollback

The deploy step logs `ROLLBACK_REF=<sha>` before pulling new code. To roll back:

```bash
ssh folawejv2@161.97.127.74
cd /opt/coldcluster
git checkout <ROLLBACK_REF>
bash deploy/scripts/deploy.sh
```
