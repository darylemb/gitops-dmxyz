# apps/

This directory is mounted by the `root-app` Argo CD Application (see
`bootstrap/root-app.yaml`). Every `.yaml` file here is applied to the
cluster. Two patterns coexist:

- **Application manifests** (`*-app.yaml`, `*.yaml` with
  `kind: Application`): declare another Argo CD Application that pulls a
  chart or repo and reconciles its manifests.
- **Direct resources** (everything else — `Ingress`, `Service`, etc.):
  applied verbatim to the cluster, with Argo's automated sync keeping
  them reconciled.

If you delete a file, `root-app` has `prune: true` and will delete the
corresponding resources from the cluster on the next sync. The user trusts
this flow for destructive Git changes.

## Layout

| File | Pattern | Purpose |
|---|---|---|
| `chroma-server.yaml` | Application | Pulls manifests from `darylemb/chroma-server` repo, deploys chroma-server. |
| `monitoring-stack.yaml` | Application | Installs `kube-prometheus-stack` Helm chart. |
| `loki-stack.yaml` | Application | Installs `loki-stack` Helm chart (logs). |
| `default-backend.yaml` | Direct resources | Traefik's default 404 backend. |
| `grafana-ingress.yaml` | Direct resource | Ingress for `grafana.darylm.xyz` with cert-manager-issued TLS. |
| `argo-ingress.yaml` | Direct resource | Ingress for `argo.darylm.xyz` with cert-manager-issued TLS. |
| `cert-manager.yaml` | Application | Installs `cert-manager` Helm chart. |
| `cert-manager/` | Direct resources | The `ClusterIssuer` for Let's Encrypt DNS-01 via Cloudflare. |

## Adding a new service

1. Add the A record + Access app in the `darylemb/cloudflare` repo's
   `services.tfvars`.
2. Add an Ingress here (or a Service of type `ClusterIP` if you want to
   route it via the cloudflared tunnel once that's running).
3. PR → Argo CD syncs → service is live.

## TLS / cert-manager

The `cert-manager` Application installs the operator. The
`letsencrypt-prod` `ClusterIssuer` (in `apps/cert-manager/cluster-issuer.yaml`)
issues certs via Let's Encrypt DNS-01 (Cloudflare is the DNS provider).
See `apps/cert-manager/README.md` for the one-time setup of the API
token Secret.
