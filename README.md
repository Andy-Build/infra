# Image Updater install

One-time setup. Run from the jumpbox.

## 1. Create a GitHub fine-grained PAT

Go to https://github.com/settings/personal-access-tokens

Click "Generate new token":

- **Token name**: `argocd-image-updater (cluster: andybuild-prod)`
- **Expiration**: 1 year (calendar a rotation reminder)
- **Resource owner**: Andy-Build
- **Repository access**: Only select repositories -> `Andy-Build/infra`
- **Permissions** -> Repository permissions:
  - **Contents**: Read and write
  - **Metadata**: Read-only (auto-selected)
  - **Pull requests**: No access (we write direct to main)
  - Everything else: No access

Click "Generate token" and copy the value once. Store it in 1Password.

## 2. Install the PAT as a K8s secret

```bash
GITHUB_PAT='github_pat_...'

kubectl -n argocd apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: git-creds
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd-image-updater
stringData:
  # Image Updater reads username + password (PAT used as password)
  username: argocd-image-updater
  password: ${GITHUB_PAT}
EOF
```

## 3. Configure the GitOps repo write-back target

The ArgoCD Application manifest for the image-updater app references this
secret implicitly via annotations. The annotations on each *consuming app*
(e.g., hamcoach-prod) tell Image Updater which credential to use:

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/write-back-target: kustomization
    # The credential reference. Defaults to the secret named 'git-creds'
    # in the argocd namespace, so this is usually omitted.
```

## 4. Apply the Image Updater Application

This is just `git push` to the infra repo with the new files:

- `apps/image-updater.yaml`
- `platform/image-updater/values.yaml`

ArgoCD's root app-of-apps will pick it up and reconcile. Watch:

```bash
kubectl -n argocd get application image-updater -w
# Want: SYNC=Synced, HEALTH=Healthy

kubectl -n argocd logs deployment/argocd-image-updater --tail=30
# Want: "Starting image update cycle" with no auth errors
```

## 5. Verify it can write

After Image Updater starts, check it's authenticating to GitHub correctly
the next time it polls (every 2 minutes by default):

```bash
kubectl -n argocd logs deployment/argocd-image-updater | grep -iE 'auth|git|push' | tail -20
```

You should see successful API calls to github.com. Failed auth shows up as
`401` or `permission denied` errors.

## 6. (Later) Configure your first app to use it

When you deploy `hamcoach-web`, add Image Updater annotations to its
ArgoCD Application -- see the spec's "ArgoCD Image Updater" section.

The first time the app is configured, Image Updater will:
1. Read the current image tag from the manifest
2. Compare to the latest matching tag in DOCR
3. If newer, write a commit to the infra repo bumping the tag
4. ArgoCD auto-syncs and rolls out the new image

## Troubleshooting

**Image Updater starts but never bumps tags.** Most common cause: tag regex
mismatch. The app's `allow-tags: "regexp:^g[0-9a-f]{7,}$"` requires tags
that start with `g` followed by 7+ hex chars (matches `git rev-parse
--short HEAD`). If your CI pushes `1.2.3` or `abc123def`, the regex won't
match. Adjust the regex in the app's annotations.

**Auth failures to GitHub.** PAT expired or scoped wrong. Check the secret:
`kubectl -n argocd get secret git-creds -o jsonpath='{.data.password}' | base64 -d`.
If empty or stale, recreate it (step 2).

**Auth failures to DOCR.** The `digitalocean-registry` pull secret in the
argocd namespace is what Image Updater uses. It's auto-rotated by the doctl
integration, but if you've manually messed with it: run
`doctl registry kubernetes-manifest | kubectl apply -f -` to recreate.

**Bumps committed but ArgoCD doesn't sync.** The Application needs
`syncPolicy.automated` enabled (it should be, by default). Without auto-sync,
Image Updater commits but nothing applies them. Check:
```bash
kubectl -n argocd get application <app-name> -o jsonpath='{.spec.syncPolicy.automated}'
# Want: {"prune":true,"selfHeal":true}
```
