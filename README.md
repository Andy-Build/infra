# andybuild k8s-infra -- Phase 2: GitOps platform

This phase brings the cluster from "DOKS exists + Vault initialized" to
"GitOps-managed platform with admin SSO, secrets sync, ingress, and a
working sample app".

## What gets deployed

| Component | Purpose |
| --- | --- |
| Cloudflare Tunnel | Outbound-only edge -> cluster connection |
| Cloudflare Access (Google SSO) | Authentication in front of admin UIs |
| ArgoCD | GitOps controller, watching this repo |
| Traefik | Ingress controller (HTTP only, behind tunnel) |
| cert-manager | Self-signed CA for internal cluster TLS |
| External Secrets Operator | Bridges Vault -> K8s Secrets |
| metrics-server | HPA + `kubectl top` |
| cloudflared | Tunnel daemon, in-cluster Deployment |
| whoami sample | End-to-end test -- Cloudflare -> Tunnel -> Traefik -> pod -> Vault secret |

Architecture:

```
browser -> cloudflare edge (TLS + Access auth)
       -> cloudflare tunnel
       -> cloudflared daemon (in cluster)
       -> traefik (HTTP)
       -> app pod
                    v (when secrets needed)
              ExternalSecret CR
                    v
              ESO controller
                    v (TLS via vault-ca bundle)
              vault-internal.andy.build:8200
                    v
              vault droplet (VPC-only)
```

## Prerequisites

- Phase 1 applied successfully
- Vault initialized + unsealed; `vault status` shows `Sealed: false`
- Snapshot token present at `/root/.vault-snapshot-token` on Vault droplet
- Vault root token retrievable from 1Password (you'll use it briefly)
- `kubectl get nodes` works from the jumpbox
- This repo cloned at `~/infra` on the jumpbox

## Setup

### 1. Apply Phase 2 Terraform

The Phase 2 Terraform adds: Cloudflare Tunnel, DNS records, Access app +
policy, and writes the tunnel credentials to `.generated/tunnel-credentials.json`.

```bash
cd ~/infra/terraform

# Add to ~/.andybuild-infra.env:
#   export TF_VAR_tunnel_secret="$(openssl rand -base64 32)"
# (only generate ONCE; rotation requires recreating the tunnel)

# Edit terraform.tfvars and confirm Phase 2 values are uncommented:
#   argocd_hostname, whoami_hostname, gitops_repo_url,
#   cloudflare_zero_trust_team, access_allowed_emails

source ~/.andybuild-infra.env
tofu plan -out tfplan
tofu apply tfplan
```

Verify the tunnel credentials file was written:

```bash
ls -la .generated/tunnel-credentials.json   # 0600, ~250 bytes
```

### 2. Verify Cloudflare Access has Google SSO configured

Visit https://one.dash.cloudflare.com -> Settings -> Authentication -> Login methods.

You should see Google listed and "Test" should pass. If Access is brand new
and Google isn't there yet:

1. Settings -> Authentication -> "Add new"
2. Choose Google
3. Follow the OAuth flow, grant permissions
4. Save

The Access application created by Terraform automatically picks up all
configured login methods.

### 3. Run the bootstrap

```bash
cd ~/infra/bootstrap

# The Vault root token is needed temporarily for 02-vault-config.sh.
# Pull it from 1Password and export it just for this session:
export VAULT_TOKEN='hvs.xxx'

./00-bootstrap.sh
```

What this does in order:

1. **01-namespaces.sh** -- creates `argocd`, `cert-manager`, `traefik-system`,
   `external-secrets`, `cloudflared`, `whoami` namespaces with
   pod-security baseline labels
2. **02-vault-config.sh** -- SSHes to Vault to enable kv-v2, K8s auth
   method, ESO policy/role, sample secret. Installs the Vault CA bundle
   as a K8s secret in `external-secrets`.
3. **03-install-argocd.sh** -- Helm install ArgoCD (HTTP only, internal
   auth disabled, anonymous = admin)
4. **04-install-root-app.sh** -- installs cloudflared credentials secret
   and applies the root app-of-apps

### 4. Watch ArgoCD reconcile everything

```bash
kubectl -n argocd get applications -w
```

Expect this order roughly:
1. `cert-manager` becomes Synced + Healthy first
2. `traefik` next (depends on no one)
3. `external-secrets` next, then `cloudflared`
4. `metrics-server` and `whoami` last

Initial reconcile takes 3-5 minutes total.

### 5. Verify end-to-end

```bash
# Tunnel is connected? (Look for "Registered tunnel connection" in logs)
kubectl -n cloudflared logs -l app=cloudflared --tail=20

# whoami should respond via Cloudflare:
curl -sSI https://whoami.andy.build
# HTTP/2 200

curl -s https://whoami.andy.build
# Hostname: whoami-xxx ...
# Look for the env vars including GREETING and SERVED_BY (proves ESO works)
```

### 6. Log into ArgoCD

Open https://argocd.andy.build in a browser. Cloudflare Access intercepts,
shows the login screen, you click Google, sign in with `andy@andybuild.com`.
ArgoCD UI loads with admin rights (no separate Argo login needed).

### 7. Revoke the Vault root token

```bash
# Back on the Vault droplet:
# Generate a less-privileged admin token for future ops, then revoke root.
# (Or keep root token in 1Password and don't revoke - it's there if needed.)
unset VAULT_TOKEN
```

The recommended pattern: keep the root token in 1Password, only export it
when you need to do admin work (creating policies, enabling auth methods,
etc.), and unset it when done.

## Adding a real app

The pattern for a new app `myapp.andy.build`:

1. Create `k8s-infra/apps/myapp.yaml` (an Argo Application pointing at
   `k8s-infra/<myapp>/`)
2. Create the manifests under `k8s-infra/<myapp>/`:
   - Deployment, Service
   - Optional ExternalSecret pulling from `kv/myapp/*`
   - IngressRoute matching `Host(\`myapp.andy.build\`)`
3. Add the hostname to the Cloudflare Tunnel config in
   `terraform/tunnel.tf` (new `ingress_rule` block) and a CNAME record
4. (Optional) Add a Cloudflare Access app for it in `terraform/tunnel.tf`
   if it should be SSO-gated
5. `tofu apply`, `git push` -- ArgoCD picks it up automatically

## Troubleshooting

**Tunnel won't connect**: Check `kubectl -n cloudflared logs`. The most
common issue is the credentials secret not existing or being malformed.
Re-run `04-install-root-app.sh` to recreate it.

**ArgoCD UI 502/504**: Tunnel isn't routing yet, or Traefik IngressRoute
isn't ready. `kubectl -n argocd port-forward svc/argocd-server 8080:80`
and visit `localhost:8080` to bypass the tunnel for diagnosis.

**ESO can't auth to Vault**: Check `kubectl -n external-secrets logs`
for the controller pod. Common: CA bundle wrong, Vault role binding
namespace mismatch, or K8s auth not configured. Re-run
`02-vault-config.sh` (idempotent).

**Cloudflare Access "user not allowed"**: Email in `access_allowed_emails`
must match exactly the Google account you're signing in with. Update the
tfvars and `tofu apply`.
