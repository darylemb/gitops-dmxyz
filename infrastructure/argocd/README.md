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

`argocd-repo-server-strategic-patch.yaml` adds an `initContainer` that:

1. Uses `debian:bookworm-slim` (glibc-based — same as the argocd-repo-server
   container, so git's libs are ABI-compatible)
2. Runs `apt-get install -y git ca-certificates`
3. Copies the `git` binary **and all its dynamic libraries** (resolved via
   `ldd`) into a shared `emptyDir` volume at `/git-bin`
4. The main `argocd-repo-server` container picks it up via:
   - `PATH=/git-bin/bin:...` (so the binary is found)
   - `LD_LIBRARY_PATH=/lib/aarch64-linux-gnu:/usr/lib/aarch64-linux-gnu:/lib:/usr/lib:/git-bin/lib`
     (puts the container's own glibc libs FIRST, so existing binaries like
     `gpg` keep working — only falls back to `/git-bin/lib` for git's own
     deps that aren't satisfied by the container's libs)

### Why not alpine/git or apk add git?

We tried both. The argocd thin image is **Debian/glibc-based** and most of
its binaries (e.g. `gpg`) link against glibc. Alpine's `git` uses musl libc
— when we copy it into the argocd container, the dynamic loader picks up
the musl libs from `/git-bin/lib` and `gpg` (glibc) crashes with
`libc.musl-aarch64.so.1: cannot open shared object file`. Switching to a
debian-sourced git keeps everything glibc-consistent.

The cluster also has **no outbound DNS to dl-cdn.alpinelinux.org** (verified
during testing), so `apk add` fails with `temporary error (try again later)`.
`apt-get` works because the cluster has outbound to `deb.debian.org`.

## Why this isn't a chart re-install

Argo CD itself was **installed imperatively** on the cluster (not via an
`apps/argocd.yaml` Application). That was a deliberate choice when Argo v2.13
was set up in 2025 — at the time we didn't have the root-app pattern fully
working. Migrating Argo to manage itself is a separate project (PR-sized).
This patch keeps the install topology stable.

## Apply the patch

```bash
cd ~/gitops-dmxyz
kubectl -n argocd patch deployment argocd-repo-server \
  --type=strategic --patch-file=infrastructure/argocd/argocd-repo-server-strategic-patch.yaml
```

(The `argocd-repo-server-patch.yaml` in this directory is the full Deployment
manifest for reference / disaster recovery — apply it with `kubectl replace --force`
only if you need to fully re-create the Deployment, e.g. if the cluster was
wiped. For incremental changes, use the strategic patch above.)

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

## ✅ Verified on 2026-07-06

Applied the patch to the live cluster. The `git executable not found` error
went away and Argo started cloning `gitops-dmxyz.git` correctly — the
`root-app` status changed from `Unknown` / `ComparisonError` to `OutOfSync`
(it now compares the live state with the desired state in git).

### Side note: how the apps get into the cluster

The `root-app` is configured as `path: apps/` with `targetRevision: HEAD` —
it works as a **directory sync source**, but it does NOT auto-discover
`Application` CRDs from subdirectories. The 7 apps currently in Argo
(`cert-manager`, `chroma-server`, `gitea`, `kube-prometheus-stack`,
`loki-stack`, `disk-usage-alerts`, and the root itself) were each applied
manually with `kubectl apply -f apps/<name>.yaml`. The root-app just keeps
them in sync once they exist.

**To deploy the new ARC apps**, after merging PRs #15-#18, you still need to
run once (one-shot bootstrap):
```bash
kubectl apply -f apps/arc-controller.yaml
kubectl apply -f apps/arc-runners-gitops.yaml
kubectl apply -f apps/arc-runners-cloudflare.yaml
```
After that, the root-app's `syncPolicy.automated.selfHeal: true` keeps them
in sync.

### Side note: the cross-node networking bug

While testing, the new repo-server pod was scheduled to `rocky-worker` (the
secondary node) and started timing out on its gRPC connection back to the
argocd-application-controller. This is the **pre-existing WireGuard Flannel
cross-node bug** documented in your memory. Workaround: pin the repo-server
to the Ampere control-plane node with:
```bash
kubectl -n argocd patch deployment argocd-repo-server --type=merge -p \
  '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"instance-20250822-0701"}}}}}'
```
The proper fix is to debug the WireGuard Flannel setup on the cluster — out
of scope for this PR.

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
