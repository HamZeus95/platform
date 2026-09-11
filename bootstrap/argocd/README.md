# ArgoCD bootstrap values

ArgoCD is the one component ArgoCD does not manage, because something has to
exist before GitOps can start. It is installed with Helm and these values are
kept here so the install is reproducible rather than living only in a shell
history:

```bash
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace \
  --version 10.4.3 -f bootstrap/argocd/values.yaml
```

Two things in here are non-obvious:

- **`server.insecure: true`** — TLS terminates at the Tailscale proxy. Without
  this, argocd-server also redirects HTTP to HTTPS and the browser bounces
  between the two forever.
- **`global.hostAliases`** — ArgoCD resolves the OIDC issuer *server-side*, and
  CoreDNS cannot resolve a tailnet name (`keycloak.tailf46ba0.ts.net`). The
  alias points that exact name at the in-cluster Keycloak Service, which serves
  a homelab-ca certificate covering it, so the issuer ArgoCD validates is
  identical to the one the browser sees. Note this key is `global.hostAliases`,
  not `server.hostAliases`.

The OIDC client secret is NOT in this file. It lives in the `argocd-keycloak-oidc`
Secret and is referenced as `$argocd-keycloak-oidc:oidc.keycloak.clientSecret`
— the `$secret:key` form, because the bare `$key` form only looks inside
`argocd-secret`.
