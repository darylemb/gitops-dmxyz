# apps/gitea

Self-hosted Gitea instance at `gitea.darylm.xyz`, deployed via the official
`gitea/gitea` Helm chart pinned to `12.6.0` (app `1.26.1`).

## Topology

- Runs on the **control plane** (OCI VM) — same node as monitoring / chroma.
- PVC: `local-path`, 10Gi, ReadWriteOnce. Single replica (RWO constraint).
- Database: SQLite (built-in via the chart). Switch to external Postgres
  later if needed.
- Auth: **Cloudflare Access** (Google OAuth via `darylemb@gmail.com`) gates
  the public hostname. Gitea's own user table stays empty of public users
  — the single admin (`daryl`) is for ops + recovery.

## What gets installed

| File | Pattern | Purpose |
|---|---|---|
| `../gitea.yaml` | Argo `Application` | Installs `gitea/gitea` Helm chart in the `gitea` namespace. |
| `ingress.yaml` | Direct resource | Traefik Ingress with cert-manager TLS for `gitea.darylm.xyz`. |
| `uploadsize-middleware.yaml` | Direct resource | Traefik middleware allowing up to 512 MiB request bodies (large `git push`). |

## DNS / Access

`darylemb/cloudflare` owns `services.tfvars` and creates the matching A
record + Cloudflare Access policy. Both PRs need to land before the URL is
resolvable. See PR there for the exact diff.

## Initial admin

The chart's `gitea.admin.username/password` are inlined as Helm values in
`gitea.yaml` for first-boot convenience. **This is a placeholder pattern.**

Once the first sync is healthy and you've logged in once, replace those
values with a reference to a Kubernetes Secret:

```yaml
gitea:
  admin:
    existingSecret: gitea-admin
```

Then put the password in a Secret:

```bash
kubectl create secret generic gitea-admin \
  --namespace gitea \
  --from-literal=password="$(openssl rand -hex 32)"
```

## Post-deploy checklist

1. Visit `https://gitea.darylm.xyz/` (you'll pass Cloudflare Access via
   Google OAuth).
2. Log in as `daryl` with the inlined password.
3. Change the admin password immediately (or migrate to `existingSecret`).
4. Create the `daryl/daryl` personal repo (or import the existing GitHub
   repos you want to mirror).
5. (Optional) Configure SSH access — the chart exposes port 22 on the
   `gitea-ssh` Service. To expose it, add an L4 OCI LB listener or
   route via Cloudflare Tunnel's TCP support.

## Notes for future operators

- **Storage**: `local-path` means the PVC is bound to the node where the
  pod first schedules. If Gitea moves nodes (e.g., after a control-plane
  rebuild) the data **does not follow it**. Move to a network storage
  class (Longhorn, Rook, OCI Block) before going multi-node.
- **Backups**: SQLite + the PVC are the entire dataset. Snapshot the PVC
  (or copy `/data/gitea/gitea.db` plus the repos directory). The chart's
  PVC lives at `/data` inside the pod.
- **Upgrades**: Bump `targetRevision` in `gitea.yaml` and `image.tag` if
  needed. The chart will roll the StatefulSet/Deployment. Watch for
  schema migrations on first boot — Gitea prints them to logs.
