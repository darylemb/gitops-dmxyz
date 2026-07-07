# Argo CD: bootstrap fixes (one-shot)

## The bug

The image `quay.io/argoproj/argocd:v3.2.5` (the **thin** image) does **not** include
the `git` binary. When Argo CD needs to fetch manifests from a git repo, it exec's
`git` from the container's PATH. Since git is missing, the `root-app` and any new
Application added to `gitops-dmxyz` fails with:

```
Failed to load target state: failed to generate manifest for source 1 of 1:
rpc error: code = Unknown desc = failed to initialize repository resources:
rpc error: code = Internal desc = Failed to fetch default:
exec: "git": executable file not found in $PATH
```

Symptom: `kubectl get applications -A` shows everything as `Unknown` / `Healthy`,
but adding a new app silently fails because the repo-server can't clone.

This bug is **pre-existing** (logs from 2026-07-05 already show it) and was
**masked** because Argo only re-evaluates when the git tree changes — so apps
that were already synced before the image upgrade appeared "Healthy" with stale
state.

## The fix

`argocd-repo-server-patch.yaml` adds an `initContainer` that installs `git` via
`apk` (the base image is Alpine 3.22.1) and exposes it to the main container
through a shared `emptyDir` volume at `/git-bin` plus a `PATH` override.

## Why this isn't a chart re-install

Argo CD itself was **installed imperatively** on the cluster (not via an
`apps/argocd.yaml` Application). That was a deliberate choice when Argo v2.13
was set up in 2025 — at the time we didn't have the root-app pattern fully
working. Migrating Argo to manage itself is a separate project (PR-sized).
This patch keeps the install topology stable.

## Apply the patch

```bash
cd ~/gitops-dmxyz
kubectl apply -f infrastructure/argocd/argocd-repo-server-patch.yaml
# Or, for a strict patch that uses strategic merge:
# kubectl -n argocd patch deployment argocd-repo-server \
#   --type strategic --patch-file infrastructure/argocd/argocd-repo-server-patch.yaml
```

Then:

```bash
kubectl -n argocd rollout status deployment/argocd-repo-server --timeout=120s
kubectl -n argocd exec deploy/argocd-repo-server -c argocd-repo-server -- \
  /git-bin/git --version
# Expected: git version 2.x.x
```

The repo-server pod will restart once (because the patch changes
`spec.template.spec.containers[0].env` — the deployment is annotated with
`argocd.argoproj.io/managed-by: manual-bootstrap` so a future Argo sync won't
fight us).

## Verify the fix

Once the new pod is up, force a re-sync of the root-app:

```bash
argocd app sync root-app --grpc-web
# Or, if argocd CLI is unavailable:
kubectl -n argocd patch application root-app --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' \
  --type merge
```

Then `kubectl get applications -A` should show `Synced` / `Healthy` for all
existing apps, and the new ARC applications (`arc-controller`,
`arc-runner-set-gitops-dmxyz`, `arc-runner-set-cloudflare`) should appear as
`OutOfSync` (they will sync automatically thanks to `automated.syncPolicy`).

## Roll back

```bash
kubectl -n argocd rollout undo deployment/argocd-repo-server
```

This brings back the previous (broken) state. The deployment has revision
history enabled.

## What's not fixed (out of scope)

- **Helm CLI not in image** — `helm` and `kubectl` aren't installed either.
  Argo v3.2.5 delegates to its own internal renderer for Helm charts, so this
  doesn't affect us today. If we ever need a plugin that shells out to `helm`,
  we'd need to extend the initContainer pattern.
- **The same bug in `argocd-application-controller-0`** — the application
  controller (a StatefulSet) is also a thin image. It doesn't need `git`
  itself (the repo-server does the clone), so this isn't currently broken.
  But if we ever switch to `argocd-application-controller` running
  `argocd-application-controller` with local repo access, we'd need to
  patch it too. We document the pattern here for future use.
- **Migrating Argo to GitOps** — making Argo manage its own install. This is
  a bigger refactor and belongs in its own PR.

## Related

- PR #15: ARC controller chart
- PR #16: ARC PAT Secret template + README
- PR #17: AutoscalingRunnerSet for gitops-dmxyz
- PR #18: AutoscalingRunnerSet for cloudflare
- PR #6 (cloudflare repo): terraform workflow using `runs-on: [self-hosted, ...]`
