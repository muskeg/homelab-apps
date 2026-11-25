# Authentik

Private identity provider for the homelab, served at
`https://auth.k8s.muskegg.com`. Argo CD installs the pinned `2026.8.6`
Authentik Helm chart, bundled PostgreSQL, External Secrets, and persistent
file/media storage.

## Required 1Password item

Create an **`authentik`** item in the `muskeg-platform` vault before syncing:

| Field | Value |
| --- | --- |
| `secret-key` | Long random Authentik signing key; generate with `openssl rand -hex 60` |
| `postgres-password` | Strong PostgreSQL password |
| `bootstrap-email` | Initial `akadmin` email address |
| `bootstrap-password` | Initial `akadmin` password |

Keep `secret-key` stable after the first deployment. Rotating it invalidates
sessions and encrypted application data. Back up both 1Password and the database
PVC before migrations or version upgrades.

## DNS and deployment

The existing wildcard `*.k8s.muskegg.com` certificate and Traefik configuration
serve the ingress. Ensure LAN DNS resolves `auth.k8s.muskegg.com` before opening
the initialization flow.

The private root Application discovers `apps/authentik.yaml` and reconciles the
chart plus `workloads/authentik`. PostgreSQL and media use protected
`local-path` PVCs. The Helm chart's PostgreSQL dependency is adequate for this
single-node homelab; for a future HA installation, migrate to an externally
managed PostgreSQL service first.

## First login

1. Wait for both the `authentik-server` and `authentik-worker` pods to be ready.
2. Open `https://auth.k8s.muskegg.com`.
3. Sign in as `akadmin` using the bootstrap email and password from 1Password.
4. Configure MFA for the administrator account.

## Stonks OIDC provider

In Authentik, create a confidential OAuth2/OpenID Connect provider for Stonks:

- Redirect URI: `https://stonks.k8s.muskegg.com/auth/callback`
- Post-logout redirect URI: `https://stonks.k8s.muskegg.com/`
- Scopes: `openid`, `profile`, `email`
- Signing key: the existing Authentik self-signed certificate
- Application slug: `stonks`

The resulting discovery issuer is:

```text
https://auth.k8s.muskegg.com/application/o/stonks/
```

Copy the generated client ID and client secret into the Stonks 1Password item.
The Stonks ExternalSecret maps them into the `stonks-oidc` Secret; restart the
Stonks deployment after that Secret becomes ready.

Restrict the Authentik application to the intended user or group. Creating the
provider alone does not grant access to every Authentik user.

## Verification

```sh
kubectl -n argocd get application authentik
kubectl -n authentik get pods,pvc,externalsecret
kubectl -n authentik rollout status deployment/authentik-server --timeout=10m
kubectl -n authentik rollout status deployment/authentik-worker --timeout=10m
curl -fsS https://auth.k8s.muskegg.com/-/health/live/
```

Test a fresh browser session, normal SSO, logout, session expiry, and an
unauthenticated `/api/portfolio` request before exposing the service outside the
LAN.

## Backups and upgrades

Back up both 1Password secrets and the Authentik PostgreSQL/media PVCs. Stop
Argo CD auto-sync before database maintenance or rollback, and never restore the
database without the matching `secret-key`. Review chart and Authentik release
notes before changing the pinned chart version.
