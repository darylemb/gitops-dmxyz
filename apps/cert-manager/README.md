# cert-manager in the dmxyz cluster

This directory bootstraps cert-manager (the Jetstack operator that issues
TLS certificates from Let's Encrypt, with the DNS-01 challenge solved via
Cloudflare).

## Layout

```
apps/cert-manager/
├── README.md              # this file
└── cluster-issuer.yaml    # the ClusterIssuer resource (applied by root-app)
```

The `cert-manager` chart itself is applied by the Application in
`apps/cert-manager.yaml` (sibling of this directory).

## One-time setup

After `apps/cert-manager.yaml` syncs and the pods come up, create the
Cloudflare API token secret **imperatively** (it's gitignored because it
contains a real token):

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token="$CLOUDFLARE_API_TOKEN"
```

The token must have scope `Zone:DNS:Edit` on `darylm.xyz` (same as the
Terraform token in the `darylemb/cloudflare` repo).

Then apply the ClusterIssuer:

```bash
kubectl apply -f apps/cert-manager/cluster-issuer.yaml
```

After ~10s, `kubectl get clusterissuer` should show `letsencrypt-prod`
with `READY = True`.

## Issuing a cert for an Ingress

Add the cert-manager annotation to the Ingress:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - grafana.darylm.xyz
      secretName: grafana-darylm-xyz-tls
  rules:
    - host: grafana.darylm.xyz
      ...
```

cert-manager will:
1. Create a `Certificate` resource (or one is created automatically from the
   Ingress annotation)
2. Solve the DNS-01 challenge by creating a `_acme-challenge` TXT record
   via the Cloudflare API
3. Save the issued cert in the `grafana-darylm-xyz-tls` Secret
4. Renew automatically 30 days before expiry

## Verifying

```bash
# Issuer is ready
kubectl get clusterissuer letsencrypt-prod

# Cert was issued for a hostname
kubectl get certificate -A

# The actual cert contents
kubectl get secret grafana-darylm-xyz-tls -n monitoring -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Order / challenge state
kubectl get challenge -A
kubectl get order -A
```

## Rotating the API token

1. Generate a new token in the Cloudflare dashboard with the same scope
2. Update the Secret imperatively:
   ```bash
   kubectl create secret generic cloudflare-api-token \
     --namespace cert-manager \
     --from-literal=api-token="$NEW_TOKEN" \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
3. The ClusterIssuer will pick up the new token on its next reconciliation
   (no need to restart cert-manager).

## Notes

- We use **DNS-01** (not HTTP-01) because the OCI Load Balancer does TCP
  passthrough to Traefik on :443, and Let's Encrypt HTTP-01 challenges need
  the LB to forward :80 traffic — which it doesn't.
- The Cloudflare Free plan supports `Zone:DNS:Edit` API tokens at no cost.
- We deliberately don't include the Secret in this repo (it would leak the
  token). Use External Secrets Operator later to pull it from Cloudflare
  itself (TODO in the README).
