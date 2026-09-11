# The platform, end to end

What this is, how the pieces fit together, every URL, and how to get into each
one.

> **No passwords in this file.** This repository is public. Every credential
> lives in `~/homelab-backups/` on the laptop (mode `0600`) and the command to
> read each one is given below. There is also a local
> `~/homelab-backups/CREDENTIALS.md` that lists the actual values in one place —
> it is deliberately outside git.

## The short version

Two Kubernetes clusters, both driven entirely from the
[platform](https://github.com/HamZeus95/platform) repo by ArgoCD. Application
code lives in its own repos; CI builds an image and commits the new tag into
the platform repo, and ArgoCD rolls it out. Nothing is applied by hand.

```
                        github.com/HamZeus95/platform
                                     │
              ┌──────────────────────┴──────────────────────┐
              │                                             │
   ┌──────────▼───────────┐                    ┌────────────▼──────────────┐
   │ dev  (laptop, WSL2)  │                    │ prod  (PC → Proxmox)      │
   │ k3d: 1 server+1 agent│                    │ 3 × Talos VMs             │
   │ ArgoCD, 4 apps       │                    │ ArgoCD, 23 apps           │
   └──────────────────────┘                    └───────────────────────────┘
                                                            │
                                         Tailscale MagicDNS │ real TLS certs
                                                            ▼
                                          reachable from any tailnet device
```

| | dev | prod |
|---|---|---|
| Runtime | k3d (k3s in Docker) | Talos v1.13.9 / Kubernetes v1.36.3 |
| Nodes | 1 + 1 | `.118` control plane, `.116` + `.169` workers |
| kubectl context | `k3d-dev` | `prod` |
| Purpose | fast, disposable | the real thing |

## URLs and how to log in

All five are on the tailnet, so they work from any device signed into Tailscale
and from nowhere else. Certificates are real (Tailscale-issued), so no browser
warnings.

### Grafana — dashboards, logs, traces

**<https://grafana.tailf46ba0.ts.net>**

Two ways in:

| Method | User | Password |
|---|---|---|
| **Keycloak SSO** (preferred) — "Sign in with Keycloak" | `hamza` | `cat ~/homelab-backups/keycloak-user-hamza-password.txt` |
| Local admin (break-glass) | `admin` | `cat ~/homelab-backups/grafana-admin-password.txt` |

The local login form is **left enabled on purpose**: it is the way in if
Keycloak is down, which is exactly when you need the dashboards. SSO maps the
Keycloak group `platform-admins` to Grafana **Admin**; anyone else who
authenticates gets **Viewer**.

The dashboard to open first:
**<https://grafana.tailf46ba0.ts.net/d/auth-app-red/auth-app-red>** — request
rate, error rate and latency for `auth-app`, plus a logs panel where clicking a
`trace_id` jumps straight to that trace in Tempo.

### Keycloak — identity provider

**<https://keycloak.tailf46ba0.ts.net>**

| Purpose | User | Password |
|---|---|---|
| Admin console (`master` realm) | `admin` | `cat ~/homelab-backups/keycloak-admin-password.txt` |
| Platform login (`homelab` realm) | `hamza` | `cat ~/homelab-backups/keycloak-user-hamza-password.txt` |

- Admin console: **<https://keycloak.tailf46ba0.ts.net/admin>**
- Realm used by everything: **`homelab`** (not `master` — `master` is only for
  administering Keycloak itself)
- OIDC clients: `grafana`, `argocd`
- Groups: `platform-admins` (→ admin everywhere), `platform-viewers` (→ read-only)

Discovery document, useful when debugging SSO:

```bash
curl -s https://keycloak.tailf46ba0.ts.net/realms/homelab/.well-known/openid-configuration | jq .issuer
```

### ArgoCD — GitOps control plane

**<https://argocd.tailf46ba0.ts.net>**

| Method | User | Password |
|---|---|---|
| **Keycloak SSO** — "LOG IN VIA KEYCLOAK" | `hamza` | same Keycloak password as above |
| Local admin | `admin` | see command below |

```bash
kubectl --context prod -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

`platform-admins` → `role:admin`; everyone else defaults to `role:readonly`.

### Backstage — developer portal

**<https://backstage.tailf46ba0.ts.net>**

**No login required** — it uses the guest provider. This is the one component
not behind Keycloak; wiring SSO needs an extra backend module. It is
tailnet-only, so it is not publicly exposed, but it is not SSO either.

### auth-app — the demo application

**<https://auth-app.tailf46ba0.ts.net>** (also `https://auth.lab` on the LAN
via MetalLB `192.168.1.50`)

| User | Password |
|---|---|
| `alice@example.com` | `password123` |
| `bob@example.com` | `password456` |

Seeded by an ArgoCD PostSync hook, so a rebuilt cluster is demo-able with no
manual steps. **These are demo credentials on an app any tailnet device can
reach** — fine for a homelab, not fine if this were exposed further.

### Other credentials

| What | Where |
|---|---|
| Vault root token + unseal key | `~/homelab-backups/vault-init.json` |
| Sealed-secrets private keys (both clusters) | `~/homelab-backups/sealed-secrets-key.*.yaml` |
| Talos config (cannot be regenerated) | `~/talos/talosconfig` |

**Vault needs unsealing after every restart** — there is no cloud KMS to
auto-unseal against:

```bash
KEY=$(python3 -c "import json;print(json.load(open('$HOME/homelab-backups/vault-init.json'))['unseal_keys_b64'][0])")
kubectl --context prod exec -n vault vault-0 -- vault operator unseal "$KEY"
```

## What runs where

| Area | Components | Cluster |
|---|---|---|
| GitOps | ArgoCD (app-of-apps, self-heal, prune) | both |
| Storage | local-path-provisioner | prod |
| Networking | MetalLB `.50-.59`, ingress-nginx, cert-manager | prod |
| Remote access | Tailscale operator (MagicDNS + certs) | prod |
| Secrets in git | Sealed Secrets | both |
| Secrets at runtime | Vault + External Secrets Operator | prod |
| Identity | Keycloak (+ its Postgres) | prod |
| Metrics | Prometheus, Alertmanager, node-exporter, kube-state-metrics | prod |
| Logs | Loki + Alloy | prod |
| Traces | Tempo | prod |
| Dashboards | Grafana | prod |
| Infra API | Crossplane → `XDatabase` | prod |
| Namespace API | devns-operator → `DeveloperNamespace` | prod |
| Portal | Backstage (+ its Postgres) | prod |
| Workload | auth-app (+ its Postgres) | both |

## The two self-service APIs

These are the point of a platform: a developer describes *what* they want, not
*how* to build it.

**A database** — Crossplane composes a PVC, Deployment, Service and a
credentials Secret containing a ready-to-use DSN:

```yaml
apiVersion: platform.homelab.io/v1alpha1
kind: XDatabase
metadata: { name: my-app }
spec: { size: small, storageGB: 5, databaseName: myapp, namespace: default }
```

```bash
kubectl --context prod get xdatabase
kubectl --context prod get secret my-app-db -o jsonpath='{.data.dsn}' | base64 -d
```

**A team namespace** — the Go operator creates the namespace, a quota, default
container limits, and RBAC bound to a Keycloak group:

```yaml
apiVersion: platform.homelab.io/v1alpha1
kind: DeveloperNamespace
metadata: { name: payments }
spec: { team: payments, size: medium, oidcGroup: payments-devs, role: edit }
```

```bash
kubectl --context prod get devns
```

## How a code change reaches production

```
1. push to  auth-app  main
2. GitHub Actions builds ghcr.io/hamzeus95/auth-app:<sha>
3. the same job commits  newTag: <sha>  into  platform/manifests/auth-app/overlays/*/kustomization.yaml
4. ArgoCD notices the commit and syncs
5. new pods roll out; the PostSync hook re-seeds demo users
```

Nothing in that path involves a human running `kubectl`. The image tag lives in
the overlays' kustomize `images:` block as a single source of truth, so the
Deployment and the seed Job can never end up on different builds.

## Day-to-day operations

### Look at things

```bash
kubectl --context prod get applications -n argocd     # everything ArgoCD manages
kubectl --context prod get pvc -A                     # all persistent volumes
kubectl --context prod get devns                      # team namespaces
kubectl --context prod get xdatabase                  # provisioned databases
```

### Deploy a change

Commit to the platform repo and push. To make ArgoCD look immediately instead
of waiting for its poll:

```bash
kubectl --context prod -n argocd annotate app <name> argocd.argoproj.io/refresh=hard --overwrite
kubectl --context prod -n argocd patch app <name> --type merge -p '{"operation":{"sync":{}}}'
```

**Ordering matters.** If the change is *inside* an `Application` (Helm values,
for instance), sync **`root` first** — syncing the child only re-pulls an
unchanged chart. This is the single most common way to conclude "my change
didn't apply".

### Add a secret

Never commit plaintext. Seal it first — the ciphertext is only decryptable by
that cluster's controller, which is why this repo can be public:

```bash
kubectl create secret generic my-secret -n my-ns \
  --from-literal=key=value --dry-run=client -o yaml \
  | kubeseal --controller-namespace kube-system \
      --controller-name sealed-secrets-controller --format yaml \
  > manifests/secrets/prod/my-secret.sealed.yaml
```

Watch for trailing newlines: `--from-file` preserves bytes exactly, so a value
captured with `echo` or `print()` ends up one byte too long. That is what
produced an `unauthorized_client` error from Keycloak once.

### Start or stop the prod cluster

Shut down gracefully — workers first, control plane last:

```bash
TALOSCONFIG=~/talos/talosconfig talosctl -n 192.168.1.116,192.168.1.169 -e 192.168.1.118 shutdown
TALOSCONFIG=~/talos/talosconfig talosctl -n 192.168.1.118 -e 192.168.1.118 shutdown
```

Then power the PC off. On the way back up: wait for the VMs, confirm
`kubectl --context prod get nodes`, then **unseal Vault** (see above). Data
survives reboots — that was verified with a real power cycle.

### When an alert fires

Every alert carries a `runbook_url` pointing at
[docs/runbooks/](runbooks/). Start there; they list the actual triage commands.

## Known limitations

Stated plainly, because they matter more than the feature list.

1. **Storage is node-local with no replication.** `local-path` provisions
   hostPath volumes. If the node holding a volume dies, that data is gone and
   the pod cannot reschedule elsewhere. Longhorn would fix this.
2. **Single control plane, no API-server VIP.** The kubeconfig is pinned to one
   node's IP. Not highly available.
3. **Mimir is not deployed.** This is Prometheus + Loki + Tempo + Grafana —
   calling it "the LGTM stack" overstates it.
4. **No backups.** No etcd snapshots, no Velero, no automated `pg_dump`. And
   `~/talos/talosconfig` cannot be regenerated — lose it and the cluster can
   never be administered again.
5. **Nodes use DHCP** with no reservations, which is why the API-server address
   is unstable.
6. **Vault needs a manual unseal after every restart.**
7. **No image scanning in CI** — no Trivy, no SBOM, no signing.
8. **Backstage uses guest auth**, not Keycloak.
9. **auth-app has no rate limiting or account lockout.**
10. **Control-plane metrics are not collected** — Talos binds
    kube-scheduler and kube-controller-manager metrics to `127.0.0.1`, so those
    scrape targets are disabled rather than left permanently down.

[docs/AUDIT.md](AUDIT.md) has the full list with the evidence behind each claim.
