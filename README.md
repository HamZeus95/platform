# platform

GitOps repository for a two-cluster Kubernetes homelab. Everything in both
clusters is defined here — nothing is applied by hand.

```
┌─ laptop (WSL2) ──────────┐        ┌─ PC: Proxmox ─────────────────────┐
│  k3d cluster  "dev"      │        │  3 × Talos VMs → cluster "prod"   │
│  ArgoCD                  │        │  ArgoCD                           │
└──────────┬───────────────┘        └──────────┬────────────────────────┘
           │        both watch this repo       │
           └──────────────┬────────────────────┘
                  github.com/HamZeus95/platform
```

| | dev | prod |
|---|---|---|
| Runtime | k3d (k3s in Docker) | Talos v1.13.9, Kubernetes v1.36.3 |
| Nodes | 1 server + 1 agent | 1 control plane + 2 workers |
| Root app | `bootstrap/root-dev.yaml` | `bootstrap/root-prod.yaml` |
| Apps directory | `apps/dev/` | `apps/prod/` |
| Purpose | fast feedback, disposable | the real thing |

## How it works

ArgoCD's **app-of-apps** pattern. One `root` Application per cluster points at
that cluster's `apps/` directory. Every file in there is itself an
`Application`, so adding a component means adding one YAML file — the root
picks it up and deploys it. There is no install step.

```
bootstrap/root-prod.yaml   →   apps/prod/*.yaml   →   manifests/*  or  a Helm chart
     (applied once)              (Applications)          (the actual workloads)
```

Two sources are used, on purpose:

- **`manifests/`** — YAML in this repo, for things we author or want pinned and
  diffable (the app, Keycloak, upstream installers we vendored).
- **Helm chart references** — for large upstream charts (kube-prometheus-stack,
  Loki, Vault, Crossplane). The values live inline in the `Application`, so a
  chart's configuration is still reviewed in a pull request.

### Why per-cluster app directories

A single shared `apps/` directory does not survive two clusters. dev already
has a k3s-owned `local-path` StorageClass, so installing a storage provisioner
cluster-wide meant both ArgoCDs fighting k3s for ownership of the same object
forever. Splitting into `apps/dev` and `apps/prod` lets a component exist on
one cluster only — storage, MetalLB, ingress-nginx and the whole observability
stack are prod-only.

### Where changes actually come from

Application code lives in its own repositories. CI there builds an image and
writes the new tag **into this repo**, which ArgoCD then rolls out:

```
auth-app repo  →  GitHub Actions  →  ghcr.io image
                                  └→ commit here: newTag: <sha>
                                                  └→ ArgoCD deploys it
```

The tag lives in each overlay's kustomize `images:` block, which is the single
source of truth — the Deployment and the seed Job resolve from the same value
and cannot drift onto different builds.

## Layout

```
bootstrap/
  root-dev.yaml, root-prod.yaml   apply once per cluster to start GitOps
  argocd/                         ArgoCD's own Helm values (it cannot manage itself)
apps/
  dev/     Applications for the dev cluster
  prod/    Applications for the prod cluster
manifests/
  auth-app/base + overlays/       kustomize: shared base, per-env overlays
  keycloak/ backstage/            authored workloads
  metallb/ ingress-nginx/ ...     vendored upstream installers
  secrets/dev + secrets/prod      SealedSecrets, encrypted, safe in git
  crossplane-config/              XRD + Composition (the XDatabase API)
  devns-operator/                 CRD + RBAC + manager for the Go operator
  observability/                  Grafana dashboard + Prometheus alert rules
docs/
  AUDIT.md                        what is verified, and the known limitations
  PLATFORM.md                     URLs, credentials, day-to-day operations
  runbooks/                       one per alert, linked from the alert itself
  examples/                       sample XDatabase claim
```

## What runs on prod

| Area | Components |
|---|---|
| GitOps | ArgoCD (app-of-apps, self-healing, prune) |
| Storage | local-path-provisioner — Talos ships none |
| Networking | MetalLB (`192.168.1.50-59`), ingress-nginx, cert-manager |
| Remote access | Tailscale operator — services get MagicDNS names and real certs |
| Secrets | Sealed Secrets (in git) + Vault and External Secrets Operator |
| Identity | Keycloak — SSO for ArgoCD and Grafana |
| Observability | Prometheus, Alertmanager, Grafana, Loki, Tempo, Alloy |
| Platform APIs | Crossplane (`XDatabase`), `devns-operator` (`DeveloperNamespace`) |
| Portal | Backstage — catalog and a self-service template |
| Workload | `auth-app` + its Postgres |

## Adding a component

1. Put its manifests in `manifests/<name>/`, or note the chart and values.
2. Add `apps/prod/<name>.yaml` (an ArgoCD `Application`).
3. Commit and push. The root app finds it. Nothing else to run.

Secrets never go in plaintext — seal them first:

```bash
kubectl create secret generic my-secret -n my-ns \
  --from-literal=key=value --dry-run=client -o yaml \
  | kubeseal --controller-namespace kube-system \
      --controller-name sealed-secrets-controller --format yaml \
  > manifests/secrets/prod/my-secret.sealed.yaml
```

The ciphertext is only decryptable by that cluster's controller, which is why
`manifests/secrets/` is split per cluster and why this repo can be public.

## Self-service APIs

A developer does not write a Deployment for a database, or four objects for a
namespace:

```yaml
# a Postgres, via Crossplane
apiVersion: platform.homelab.io/v1alpha1
kind: XDatabase
metadata: { name: my-app }
spec: { size: small, storageGB: 5, databaseName: myapp, namespace: default }
```

```yaml
# a namespace with quota, limits and RBAC, via the Go operator
apiVersion: platform.homelab.io/v1alpha1
kind: DeveloperNamespace
metadata: { name: payments }
spec: { team: payments, size: medium, oidcGroup: payments-devs, role: edit }
```

## Things that will bite you

Recorded because each one cost real debugging time.

- **Talos enforces PodSecurity `baseline` cluster-wide.** Anything needing
  hostPath, hostNetwork or `IPC_LOCK` needs a privileged namespace label.
  That is why Vault runs with `disable_mlock`.
- **local-path provisions `hostPath` PVs, and Kubernetes does not apply
  `fsGroup` to hostPath volumes.** Charts that mount a PVC with a `subPath` and
  run as non-root need a chown init container — Prometheus and Loki both do.
  Mounting the volume *root* is fine, because local-path creates it `0777`.
- **RWO volume + node-local storage + `RollingUpdate` = two pods on one data
  directory.** Every stateful workload here uses `strategy: Recreate`.
- **Root syncs before its children.** When the change is inside an
  `Application` (Helm values, for instance), sync `root` first — syncing the
  child just re-pulls an unchanged chart.
- **Talos binds control-plane metrics to `127.0.0.1`**, so the
  kube-scheduler / kube-controller-manager ServiceMonitors are disabled rather
  than left permanently DOWN.

## Documentation

- **[docs/PLATFORM.md](docs/PLATFORM.md)** — every URL, who to log in as, where
  the passwords are kept, and the common operations.
- **[docs/AUDIT.md](docs/AUDIT.md)** — what has actually been verified, with
  evidence, plus an explicit list of limitations (node-local storage, single
  control plane, no backups).
- **[docs/FIXES.md](docs/FIXES.md)** — every bug hit while building this, with
  the real cause and the fix. Several are not specific to this homelab.
- **[docs/runbooks/](docs/runbooks/)** — one per alert, linked from the alert's
  own `runbook_url`.

## Related repositories

| Repo | What it is |
|---|---|
| [auth-app](https://github.com/HamZeus95/auth-app) | React + Hono + Postgres service, OpenTelemetry-instrumented |
| [devns-operator](https://github.com/HamZeus95/devns-operator) | Kubebuilder operator for `DeveloperNamespace` |
| [backstage](https://github.com/HamZeus95/backstage) | Developer portal: catalog and software templates |
