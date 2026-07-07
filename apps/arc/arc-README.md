# Actions Runner Controller (ARC) — setup

Self-hosted GitHub Actions runners on the Ampere k3s cluster, orchestrated by
[actions/actions-runner-controller](https://github.com/actions/actions-runner-controller).

## Components in this directory

| File | Kind | Purpose |
|---|---|---|
| `arc-secret.yaml` | Secret (template) | Holds the GitHub PAT used to register runners |
| `../arc-controller.yaml` | Argo Application | Installs the ARC operator (chart v0.23.7) in `arc-systems` |
| `../arc-runners-gitops.yaml` | AutoscalingRunnerSet | Runner pool for `darylemb/gitops-dmxyz` (max 5) |
| `../arc-runners-cloudflare.yaml` | AutoscalingRunnerSet | Runner pool for `darylemb/cloudflare` (max 2) |

## One-time setup (manual, requires you)

### 1. Generate a GitHub PAT

Go to **https://github.com/settings/tokens/new** and create a **classic** token (`ghp_...`) with:

- **Note**: `arc-runners-ampere`
- **Expiration**: 90 days (set a calendar reminder to rotate)
- **Scopes**: `repo` (full)

Save the token in Proton Pass under `infra/k8s/arc-github-pat`.

### 2. Create the Secret imperatively

```bash
# Generate the token (or copy from Proton Pass), pipe to kubectl.
# Using --from-literal=/dev/stdin keeps the token out of shell history
# and out of any log file.
echo -n "$YOUR_PAT_HERE" | kubectl create secret generic arc-github-secret \
  --namespace=arc-runners \
  --from-literal=github_token=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:
```bash
kubectl get secret arc-github-secret -n arc-runners
# NAME                TYPE     DATA   AGE
# arc-github-secret   Opaque   1      5s

kubectl get secret arc-github-secret -n arc-runners -o jsonpath='{.data.github_token}' | base64 -d | head -c 4
# Should print: ghp_
```

### 3. Wait for the listener to pick up the Secret

The `AutoscalingRunnerSet` resources reference this Secret by name. After they
land (PR3, PR4), the listener will connect to GitHub and the runner pods will
start coming online as workflows request them.

## How the pieces fit

```
┌─────────────────────────────────────────────────────────────┐
│ GitHub API (api.github.com)                                 │
│ - knows about runners registered by PAT                    │
│ - dispatches jobs to runners with matching labels          │
└──────────────┬──────────────────────────────────────────────┘
               │  (PAT auth from runner pods)
               ▼
┌─────────────────────────────────────────────────────────────┐
│ arc-systems (controller)                                    │
│   - ARC controller pod: reconciles AutoscalingRunnerSets    │
│   - For each scale set: 1 EphemeralRunner + 1 Listener      │
│     (listeners long-poll GitHub for jobs)                   │
└──────────────┬──────────────────────────────────────────────┘
               │  (manages)
               ▼
┌─────────────────────────────────────────────────────────────┐
│ arc-runners (scale sets)                                    │
│   - gitops-dmxyz: 0..5 ubuntu-22.04 pods (kubectl, kustomize)
│   - cloudflare:    0..2 ubuntu-22.04 pods (terraform)       │
└─────────────────────────────────────────────────────────────┘
```

## Using the runners in workflows

Once everything is up, reference the runners in any workflow with the label
`self-hosted` (default for ARC) plus the runner pool name (added via
`runnerScaleSetName` and the `runs-on:` label that the chart derives from
the `AutoscalingRunnerSet` name).

```yaml
# In a workflow under .github/workflows/
jobs:
  build:
    # Runs on any ARC-managed runner (both pools)
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: echo "hello from $(hostname)"

  deploy-gitops:
    # Runs only on the gitops-dmxyz scale set
    runs-on: [self-hosted, arc-runner-gitops-set]
    steps:
      - run: kubectl get pods
```

The exact labels ARC adds are visible at
**Repo → Settings → Actions → Runners** after the first scale set comes online.

## Troubleshooting

### "Runner not connecting"

```bash
# 1. Check the Secret is valid
kubectl get secret arc-github-secret -n arc-runners -o jsonpath='{.data.github_token}' | base64 -d | head -c 4
# Should print: ghp_

# 2. Check the listener pod is up
kubectl get pods -n arc-runners -l app.kubernetes.io/name=actions-runner-controller

# 3. Check the listener logs
kubectl logs -n arc-runners -l actions.github.com/scale-set-name=arc-runner-gitops-set -c listener

# 4. Check that the PAT has repo scope
curl -s -H "Authorization: token $YOUR_PAT" https://api.github.com/user | jq .login
```

### "Pod stuck in Pending"

Almost always a resource issue. Check:

```bash
kubectl describe pod -n arc-runners <pod-name>
# Look at "Events" — usually:
#   - Insufficient cpu/memory
#   - PersistentVolumeClaim not bound (shouldn't happen — we use ephemeral storage)
#   - FailedScheduling due to node taints
```

If CPU/memory, either:
- Lower the `template.spec.containers[].resources.requests` in the scale set
- Or raise the cluster's allocatable resources (more memory on Ampere)

### "Secret does not exist" when listener starts

You created the scale set (PR3/PR4) but haven't created the Secret yet (PR2
manual step). Either:
- Create the Secret following step 2 above, OR
- If you don't want the scale set to start runners yet, scale `minRunners: 0`
  and `maxRunners: 0` temporarily

## Rotating the PAT

```bash
echo -n "$NEW_PAT" | kubectl create secret generic arc-github-secret \
  --namespace=arc-runners \
  --from-file=github_token=/dev/stdin \
  --dry-run=client -o yaml | kubectl replace -f -

# Force the listener to pick up the new Secret
kubectl rollout restart deployment -n arc-runners -l app.kubernetes.io/name=actions-runner-controller
```

Listeners reconnect automatically; in-flight runners continue with the old
PAT until they finish (typically <5 min).

## Security notes

- The PAT is stored in etcd, which is **not encrypted at rest** on this
  k3s cluster (default). Mitigations:
  - Enable k3s `--secrets-encryption` flag (one-time setup, restart the server)
  - Or use a SOPS/Sealed Secrets workflow (more setup, more secure)
- The PAT has full `repo` scope. Consider:
  - A **fine-grained PAT** scoped to ONLY the two repos (and only the
    `actions: read` permission) — but ARC currently requires classic PATs.
  - A **GitHub App** with installation tokens (more complex, see ARC docs
    `authenticating-to-the-github-api.md`)
- The runner pods run as **non-root** by default in the chart, and have
  no host network access. Sandbox is enforced by Kubernetes.
- Workflows that run on the runners can **read** the PAT (if they
  `kubectl describe secret` somehow), but cannot **exfiltrate** it
  through a normal CI step without explicit user action.

## Reference

- Chart: `oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set` v0.14.2
- Chart: `oci://ghcr.io/actions/actions-runner-controller-charts/actions-runner-controller` v0.23.7
- Docs: https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller
- Source: https://github.com/actions/actions-runner-controller
