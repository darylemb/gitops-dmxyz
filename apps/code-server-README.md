# code-server (code.darylm.xyz)

Self-hosted VS Code in the browser. Single-user instance, gated by Cloudflare Access at the edge.

## Architecture

```
Browser (code.darylm.xyz)
    ↓ HTTPS
Cloudflare Access (Google OAuth, 24h session)
    ↓ CF Tunnel (or DNS A record → OCI LB → Traefik)
Traefik (cert-manager TLS, letsencrypt-prod)
    ↓
code-server pod (1 replica, 200m-1.5CPU, 512Mi-3Gi RAM)
    ↓ bind-mounts
/home/opc           (RW)
/etc/rancher/k3s    (RO)
/var/lib/rancher/k3s/storage  (RO)
/home/opc/gitops-dmxyz  (RW, default workspace)
```

## Files

| File | Purpose |
|---|---|
| `code-server.yaml` | Argo CD Application (chart-based, helm values inline) |
| `code-server-pvc.yaml` | PVC for `/config` (5Gi, local-path) |
| `code-server-configmap.yaml` | Default settings.json + code-server-config.yaml |
| `code-server-ingress.yaml` | Traefik Ingress + upload-size Middleware |
| `code-server-secret.yaml` | **TEMPLATE ONLY** — manual Secret creation |

## Initial setup (one-time, manual)

The admin password Secret is NOT applied via Git. Create it once with:

```bash
openssl rand -hex 32 | kubectl create secret generic code-server-admin \
  --namespace=code-server --from-file=password=/dev/stdin
```

To rotate: regenerate the Secret and restart the pod (or wait for Argo selfHeal).

## Cloudflare setup (terraform)

The `darylemb/cloudflare` repo creates:

1. **DNS A record**: `code.darylm.xyz` → OCI Load Balancer IP
2. **Access Application**: name=`code-server`, domain=`code.darylm.xyz`
3. **Access Policy**: Allow `darylemb@gmail.com` (one-time login per 24h)

After `terraform apply`, login at `https://code.darylm.xyz` — Cloudflare will prompt for Google auth, then forward to code-server (which will show a password prompt — use the one from the Secret, OR disable the in-app password since CF Access already auth'd you).

## What you get in the browser

- Full VS Code UI (dark theme, JetBrains Mono, format-on-save)
- File explorer showing your `~` (host `/home/opc`)
- Terminal in the browser (real bash, runs as `opc`)
- Extensions auto-install (persisted in PVC at `/config/extensions`)
- Git integration with the local SSH key (mounted at `~/.ssh` from `/home/opc/.ssh`)
- Workspace: `/workspace/gitops-dmxyz` (the `gitops-dmxyz` repo) opens by default

## Why local-path storage?

Same as `gitea`, `chroma`, `prometheus`, `loki` — keeps PVCs on the big
`/mnt/ampere-data` disk (115GB free) via the bind-mount config we did earlier.

## Security notes

- **Cloudflare Access is the primary auth** — code-server's in-app password is
  defense-in-depth only. If you trust CF Access, you can set the chart's
  `serviceAccount` and `auth: none` to skip the in-app prompt.
- **Filesystem access is scoped** — only `/home/opc` (RW), k3s configs (RO),
  and the gitops repo (RW). No access to `/etc`, `/var`, or other namespaces.
- **The container runs as `opc` (uid 1000)** — same as your SSH user, so file
  ownership stays consistent between code-server and shell sessions.

## Upgrades

To bump code-server, edit `targetRevision` in `code-server.yaml` (and the
`image.tag`) and open a PR. Argo will sync within ~1 minute.

## Troubleshooting

- **Cert not issuing**: check `kubectl describe certificate code-darylm-xyz-tls -n code-server`
- **Traefik 413**: ensure `code-server-uploadsize` Middleware is applied (see ingress annotations)
- **Files not showing**: verify hostPath mounts in `code-server.yaml` — paths must exist on the node
- **Access denied at CF**: re-check the Access policy in CF dashboard — your email must match exactly