# Homelab applications

Private Argo CD application registrations for the homelab cluster.

The public `muskeg-platform` repository manages cluster and platform components. This repository intentionally owns private application names, namespaces, and the Argo CD pointers to workload repositories.

## One-time setup

Create an `argocd-github` item in the `muskeg-platform` 1Password vault with:

- `github-username`: a custom text field containing a GitHub username with access to the private application repositories
- `github-token`: a custom concealed field containing a fine-grained PAT with read-only **Contents** permission for `homelab-apps` and each workload repository Argo must read

Use these exact custom field names. Do not use the item's built-in `username` field; External Secrets can otherwise find multiple fields with that label.

Then bootstrap this repository from a local checkout:

```bash
kubectl apply -f bootstrap/repository-credentials.yaml
kubectl wait --for=condition=Ready externalsecret/argocd-github \
  -n argocd --timeout=120s
kubectl get secret argocd-github -n argocd
kubectl apply -f bootstrap/root.yaml
```

The repository-credential Secret uses the `repo-creds` form, so Argo CD can authenticate to all `https://github.com/muskeg/...` repositories without listing each private repository publicly.

After bootstrap, commit Application manifests under `apps/`. The `homelab-apps` root Application is automated and self-healing, and Argo CD will create and reconcile them.

## Adding an application

Add a file such as:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/muskeg/example.git
    targetRevision: main
    path: deploy/kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: example
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

The workload repository owns its Kubernetes manifests and image-promotion workflow; this repository only registers it with Argo CD.
