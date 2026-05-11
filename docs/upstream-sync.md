# Upstream Sync Procedure

This document describes how to sync `avbdev/plane` with new releases from the upstream `makeplane/plane` repository.

## Overview

`avbdev/plane` is a **minimal patch fork** of `makeplane/plane`. The fork applies exactly two categories of patches:

1. **Non-root user directives** in all production Dockerfiles (security hardening for Kubernetes `runAsNonRoot: true`)
2. **django-prometheus** integration for Prometheus metrics scraping in the homelab cluster

All other customisation (Kubernetes manifests, CI/CD, secrets, ingress) lives in the separate `avbdev/plane-k8s` deployment repository. The smaller the fork's divergence, the cheaper every upstream sync is.

## Patched Files

These are the only files that differ from upstream. Check each one after every sync:

| File | Patch description | Upstream conflict risk |
|------|------------------|----------------------|
| `apps/api/Dockerfile.api` | `addgroup/adduser plane` + `USER 1000` before EXPOSE | **Low** — only risks conflict if upstream adds their own USER or removes the file |
| `apps/live/Dockerfile.live` | `chown -R node:node /app` + `USER node` in runner stage | **Low** |
| `apps/space/Dockerfile.space` | `chown -R node:node /app` + `USER node` in runner stage | **Low** |
| `apps/web/Dockerfile.web` | `USER nginx` in production stage | **Low** |
| `apps/admin/Dockerfile.admin` | `USER nginx` in production stage | **Low** |
| `apps/api/requirements/base.txt` | `django-prometheus==2.3.1` appended | **Very low** — new line, rarely conflicts |
| `apps/api/plane/settings/common.py` | `django_prometheus` in `INSTALLED_APPS`, `PrometheusBeforeMiddleware` first, `PrometheusAfterMiddleware` last in `MIDDLEWARE` | **Low** — only risks conflict if upstream reorders MIDDLEWARE heavily |
| `apps/api/plane/urls.py` | `path("", include("django_prometheus.urls"))` added before web URLs | **Low** |

## Trigger

The weekly [Upstream Release Check](.github/workflows/upstream-check.yml) workflow runs every Monday at 09:00 UTC. When a new upstream release is detected, it opens a GitHub Issue titled:

```
⬆️ Upstream release available: vX.Y.Z
```

That issue is your prompt to run the sync procedure below.

## Sync Procedure

### Step 1 — Review the upstream diff (10 min)

Before touching any code, read the upstream changelog and check if any of our patched files were modified:

```bash
# Replace v1.3.0 with the previous tag and v1.4.0 with the new upstream tag
open https://github.com/makeplane/plane/compare/v1.3.0...v1.4.0
```

**Focus on:** `apps/api/Dockerfile.api`, `apps/*/Dockerfile.*`, `apps/api/requirements/`, `apps/api/plane/settings/common.py`, `apps/api/plane/urls.py`

If none of those files were touched upstream → the sync will be zero-conflict. Proceed.

### Step 2 — Add the upstream remote (first time only)

```bash
git remote add upstream https://github.com/makeplane/plane.git
git remote -v  # Verify: origin = avbdev/plane, upstream = makeplane/plane
```

### Step 3 — Fetch upstream tags

```bash
git fetch upstream --tags
```

### Step 4 — Create a sync branch from main

```bash
# Replace v1.4.0 with the actual upstream tag
git checkout -b upstream-sync/v1.4.0 main
```

### Step 5 — Merge the upstream tag

```bash
git merge upstream/v1.4.0 --no-ff -m "chore: merge upstream v1.4.0"
```

**If there are no conflicts:** skip to Step 6.

**If there are conflicts** (only possible on the 8 patched files listed above):

```bash
git status  # See which files conflicted
```

**Resolution rules:**

| Scenario | Action |
|----------|--------|
| Upstream updated `FROM python:3.12.10` → `FROM python:3.13.0` in Dockerfile | **Keep both** — their FROM change + our USER directive. Accept their version of the FROM line. |
| Upstream added their own `USER` directive | **Use theirs if it's non-root.** Remove ours. We won. |
| Upstream added a new package to `requirements/base.txt` | **Keep both** — append their new package and keep our `django-prometheus` line. |
| Upstream restructured `INSTALLED_APPS` | **Keep structure** — re-add `django_prometheus` at the end of the list. |
| Upstream restructured `MIDDLEWARE` | **Keep structure** — re-add `PrometheusBeforeMiddleware` as first entry and `PrometheusAfterMiddleware` as last entry. |
| Upstream restructured `urls.py` | **Keep structure** — re-add `path("", include("django_prometheus.urls"))` before the web URL include. |

**Rule of thumb:** Our security patches ALWAYS win. But if upstream fixed the security issue themselves (e.g., added their own USER directive), remove our version and use theirs.

### Step 6 — Verify patches are still present

```bash
echo "=== Checking API Dockerfile ==="
grep -n "USER 1000\|addgroup\|adduser plane" apps/api/Dockerfile.api

echo "=== Checking live Dockerfile ==="
grep -n "USER node\|chown.*node" apps/live/Dockerfile.live

echo "=== Checking space Dockerfile ==="
grep -n "USER node\|chown.*node" apps/space/Dockerfile.space

echo "=== Checking web Dockerfile ==="
grep -n "USER nginx" apps/web/Dockerfile.web

echo "=== Checking admin Dockerfile ==="
grep -n "USER nginx" apps/admin/Dockerfile.admin

echo "=== Checking requirements ==="
grep "django-prometheus" apps/api/requirements/base.txt

echo "=== Checking settings ==="
grep "django_prometheus\|PrometheusBeforeMiddleware\|PrometheusAfterMiddleware" \
  apps/api/plane/settings/common.py

echo "=== Checking urls.py ==="
grep "django_prometheus" apps/api/plane/urls.py
```

All checks must return at least one match. If any come back empty, re-apply the relevant patch manually.

### Step 7 — Open a PR

```bash
git push origin upstream-sync/v1.4.0

gh pr create \
  --title "chore: sync upstream v1.4.0" \
  --body "Merges makeplane/plane v1.4.0 into avbdev/plane.

## Patches verified present
- [x] Non-root USER in Dockerfile.api
- [x] Non-root USER in Dockerfile.live / Dockerfile.space
- [x] USER nginx in Dockerfile.web / Dockerfile.admin
- [x] django-prometheus in requirements/base.txt
- [x] django_prometheus in INSTALLED_APPS
- [x] Prometheus middleware in MIDDLEWARE
- [x] django_prometheus.urls in urls.py

## Changelog
https://github.com/makeplane/plane/releases/tag/v1.4.0" \
  --base main
```

### Step 8 — CI validates and auto-promotes to staging

After the PR merges to `main`, the CI pipeline ([build-and-push.yml](.github/workflows/build-and-push.yml)) automatically:

1. Runs tests
2. Builds all 5 images with tag `main-{sha}`
3. Scans with Trivy
4. Updates `avbdev/plane-k8s` staging overlay
5. ArgoCD syncs staging

Validate at `https://staging-plane.tools.avb.dev` before tagging a production release.

## If Upstream Fixes One of Our Patches

The ideal outcome: upstream ships non-root USER directives themselves.

When this happens during a sync:
1. Remove our version of the patch (don't keep duplicate USER directives)
2. Update this document — remove that row from the patched files table
3. Comment in the merge commit: `avbdev patch removed: upstream now handles [description]`

Over time, the goal is for this patched files table to become empty.

## Escalation

If an upstream sync introduces a breaking change that conflicts with our deployment:

1. **Do not merge to main.** Leave the sync branch open.
2. Open a GitHub Issue describing the conflict.
3. Tag `@avbdev` for review before proceeding.

If upstream changes their database schema format or Django version in a major breaking way, escalate to a full review before merging — that's no longer a routine sync.
