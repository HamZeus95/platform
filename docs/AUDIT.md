# Platform audit

**Date:** 2026-09-11 · **Cluster:** `prod` — 3× Talos v1.13.9, Kubernetes v1.36.3
· **Repos:** [platform](https://github.com/HamZeus95/platform),
[auth-app](https://github.com/HamZeus95/auth-app),
[devns-operator](https://github.com/HamZeus95/devns-operator),
[backstage](https://github.com/HamZeus95/backstage)

Every claim below was checked against the running cluster. Where something is
partial or unverified it says so — an audit that only lists successes is not
an audit.

## Verdict per claim

| Claim | State | Evidence |
|---|---|---|
| GitOps (ArgoCD app-of-apps, two clusters) | **Done** | 23 Applications, per-cluster roots, `Synced/Healthy` |
| Persistent storage, dynamic provisioning | **Done** | 9 PVCs bound; survived a real power cycle |
| GitOps-managed encrypted secrets | **Done** | Sealed Secrets; 7 sealed secrets in git, no plaintext |
| Ingress, automated TLS, MetalLB, Kustomize overlays | **Done** | `auth.lab` 200 `verify=0`; MetalLB 192.168.1.50 |
| Tailscale mesh + MagicDNS service exposure | **Done** | 4 services on the tailnet with real certs |
| Prometheus / Loki / Tempo observability, dashboards, alerting | **Done** | 22/22 targets up, 149 rules, alert fired end-to-end |
| OpenTelemetry instrumentation | **Done** | Traces in Tempo, RED metrics scraped, log↔trace linked |
| Vault + External Secrets Operator | **Done** | Vault→ESO→Secret rotation proven |
| Keycloak SSO | **Done** | Grafana and ArgoCD logins driven end-to-end |
| Crossplane (XRD + Composition) | **Done** | Claim produced a live, connectable Postgres |
| Custom Kubernetes operator in Go | **Done** | Deployed, 9 envtest specs, self-healing proven |
| Backstage developer portal | **Done** | 15 catalog entities + golden-path template, tailnet-reachable |
| Mimir / full "LGTM" | **Not done** | Deliberately — see Honest limitations |

---

## Phase 0 — Persistent storage

Prod had **no StorageClass at all**; Postgres ran on the container's writable
layer and `initdb` re-ran empty on every boot. The database had already been
wiped five times.

- `local-path-provisioner` v0.0.37, prod-only (dev has a k3s-owned
  `local-path`, which ArgoCD would have fought for ownership of forever)
- Two Talos-specific patches: provisioner path off read-only `/opt` → `/var`,
  and a `pod-security: privileged` namespace label, because Talos enforces
  `baseline` cluster-wide and the hostPath helper pods are forbidden under it
- `strategy: Recreate` on Postgres — with RWO + `WaitForFirstConsumer` a
  rolling update schedules the new pod onto the *same* node while the old one
  still holds PGDATA, giving two postgres processes on one data directory

**Proof.** A row written before a full graceful power cycle survived it. Node
`Ready` timestamps clustered at 13:39–13:40, the old db pod's container was
gone, and a **brand-new** container started at 13:40:45 — the exact situation
that previously wiped the database. Both rows came back.

## Phase 1 — Secrets in Git

`auth-app-secrets` had been created with an imperative `kubectl create` and
existed nowhere in git, so ArgoCD reported Healthy while the only copy of the
token-signing keys lived in cluster state.

Sealed Secrets was chosen over SOPS because ArgoCD decrypts SealedSecrets
natively via the in-cluster controller; SOPS needs a repo-server plugin.
Secrets are sealed **per cluster** — each controller holds its own keypair, so
the prod ciphertext cannot be decrypted by dev.

**Two real bugs.** The controller refuses to adopt an unmanaged Secret. And
after deleting the imperative one it *still* would not reconcile: the log
showed `Error updating, giving up` (5 retries, then dropped from the
workqueue), and an annotation nudge was rejected with `update suppressed, no
changes in spec`. A controller restart was required.

**Proof.** The recreated Secret is byte-identical to the original (so issued
tokens still validate) and owned by `SealedSecret/auth-app-secrets`. Sealing
keys backed up to `~/homelab-backups/` at 0600, with `.gitignore` guarding
against committing them.

## Phase 2 — Ingress, TLS, Kustomize

- **MetalLB** pool `192.168.1.50-59`, inside the static range below the
  router's DHCP pool. Pool CRs at sync-wave 1 so the webhook is serving first.
- **ingress-nginx** took `.50`. Its two admission Jobs ship
  `ttlSecondsAfterFinished: 0`, so they delete themselves on success and
  ArgoCD recreated them forever — permanently OutOfSync. Fixed by running
  them as Sync hooks.
- **cert-manager** needs `ServerSideApply=true`; its CRDs exceed the
  262144-byte `last-applied-configuration` cap.
- **Kustomize base + dev/prod overlays.** `CLIENT_URL: http://localhost:3000`
  is gone from production.

**Proof.** `curl https://auth.lab` → **200**, `ssl_verify_result=0` verified
against the CA, cert issued by `homelab-ca`, HTTP 308→HTTPS.

Tailscale MagicDNS exposes `auth-app`, `grafana`, `keycloak`, `argocd` and
`backstage` with publicly-trusted certs. The operator crash-looped until its
own tag was corrected — the chart defaults to `tag:k8s-operator`, but the ACL
and OAuth client only authorize `tag:k8s`.

## Phase 3 — Observability

Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, Loki,
Tempo and Alloy, all on local-path PVCs.

**22/22 scrape targets up, 0 down. 149 alert rules loaded.**

Talos binds `kube-controller-manager` and `kube-scheduler` metrics to
`127.0.0.1`, so those ServiceMonitors are disabled rather than left
permanently DOWN — which is why the target count is clean. Enabling them needs
a machine-config patch setting `bind-address=0.0.0.0`.

**The most reusable bug.** Prometheus crash-looped with
`open /prometheus/queries.active: permission denied`. Root cause:
local-path provisions **hostPath** PVs, and Kubernetes does not apply `fsGroup`
ownership to hostPath volumes. Despite `fsGroup: 2000` the volume stayed
`root:root` and the kubelet-created `prometheus-db` **subPath** was root-owned
`0755` — unwritable by uid 1000. Grafana was fine only because its chart ships
an `init-chown-data` container.

Loki needed the same fix. **Tempo did not** — it mounts the volume root with no
subPath, and local-path creates that root `0777`. That confirms the subPath
subdirectory was the actual cause, and explains why Postgres never hit it.

### Instrumentation

Instrumented at the Hono layer, not with `@opentelemetry/sdk-node`: the
auto-instrumentations patch modules via `require-in-the-middle`, which is not
dependable under Bun. Telemetry is opt-in — with `OTEL_EXPORTER_OTLP_ENDPOINT`
unset the app runs unchanged, which is what the dev overlay wants.

A bug caught in local testing: `@hono/otel` stamps its own `service_name`
metric attribute and defaults it to an empty string rather than reading the
provider's resource. Every series would have been labelled `service_name=""`.

**Proof.** Counts matched generated traffic exactly (`/health` 20, `/hello` 12,
`/auth/login` 401×5, 200×3). A `trace_id` pulled from a **Loki log line**
resolved in **Tempo** to the matching `POST /auth/login` span with
`status=401`.

### Dashboard and alerting

Dashboard `auth-app-red` (9 panels) at
`https://grafana.tailf46ba0.ts.net/d/auth-app-red/auth-app-red`.

The error rate counts **5xx only**. A 401 from `/auth/login` means the app
correctly rejected a credential; folding those in would burn error budget on
correct behaviour and mask real faults. Failed logins get their own alert,
because a *burst* is a credential-stuffing signal.

A flaw found by checking every query returns data: the error-rate tile came
back with **0 series** — with no 5xx in range the numerator is empty, and
empty/anything is empty in PromQL, so a healthy service rendered "No data",
indistinguishable from broken monitoring. Fixed with `or vector(0)`.

**Proof.** `AuthAppLoginFailureSpike` went `inactive → pending → firing` as
the rate climbed past 30/min, reached **Alertmanager** with its description
templated (`451 failed logins per minute`) and runbook URL attached, then
returned to `inactive`. All four runbook URLs return **200**.

## Phase 4 — Vault, ESO, Keycloak

Vault runs **standalone with file storage on a PVC**, not dev mode — dev mode
holds everything in memory behind a fixed root token and loses every secret on
restart. The cost is a manual unseal after each restart. `disable_mlock` is
required because Talos's `baseline` PSA forbids `IPC_LOCK`.

ESO authenticates with its own Kubernetes ServiceAccount token, which Vault
validates against the cluster API before issuing a short-lived token carrying a
read-only policy bound to that one service account in that one namespace. No
long-lived Vault credential exists in the cluster.

**Proof.** A value written in Vault appeared as a Kubernetes Secret; changing it
in Vault updated the Secret (`ROTATED at 16:02:48Z`).

Keycloak 26.4.7 with its own Postgres. `KC_PROXY_HEADERS` is essential — without
it OIDC discovery advertises `http://` and every redirect breaks.

**Four bugs, each worth being able to explain:**

1. `invalid_scope` — requesting `scope=groups` asks for a *client scope* of
   that name; the groups claim comes from a protocol **mapper** on the client's
   dedicated scope, applied regardless of scopes.
2. `database is locked (SQLITE_BUSY)` — Grafana keeps state in SQLite on an RWO
   PVC, and because local-path is node-local both old and new pods land on the
   same node and both mount it. `strategy: Recreate`. Applying that then failed
   with `spec.strategy.rollingUpdate: Forbidden`, because server-side apply
   merges rather than removes.
3. `lookup keycloak.tailf46ba0.ts.net ... no such host` — the **browser** needs
   the public MagicDNS name, but the token exchange runs **server-side inside
   the cluster**, where CoreDNS cannot resolve a tailnet name. Grafana's
   `auth_url` stays public; `token_url`/`api_url` use cluster DNS.
4. `unauthorized_client` — the client secret had been captured with a `print()`
   that appends a newline, and `--from-file` preserves bytes exactly, so the
   cluster held 33 bytes where Keycloak expects 32.

ArgoCD could not use Grafana's split-URL trick, because it validates that the
issuer it fetches matches the one configured. Solved by giving Keycloak a
`homelab-ca` certificate covering its public hostname, serving HTTPS
in-cluster, and a `global.hostAliases` entry (not `server.hostAliases`).

**Proof.** Grafana: `login: benali.hamza01@gmail.com`,
`authLabels: ['Generic OAuth']`, `role=Admin` from `platform-admins`.
ArgoCD: `loggedIn: true`, issuer correct,
`groups: ["platform-admins", ...]`, RBAC `platform-admins → role:admin`.

Grafana's local admin login stays enabled on purpose — the break-glass path if
Keycloak is down, which is exactly when you need the dashboards.

## Phase 5 — Crossplane

Crossplane 2.4.0, `provider-kubernetes`, and a function pipeline
(`go-templating` + `auto-ready`) — v2 removed the legacy `resources` array, so
every Composition is a pipeline now.

`XDatabase` is a developer-facing API: *"I want a database"*, not four
Kubernetes objects. `size` is a t-shirt size, so the platform can retune what
"small" means without teams editing YAML.

Two decisions worth defending:

- **The password is derived deterministically from the composite's UID**, not
  `randAlphaNum`. A random value would be regenerated on every reconcile,
  rewriting the Secret and breaking the database it had just provisioned.
- **Provider RBAC binds to the service-account *group***, not the provider's SA
  name — that name carries a hash that changes on upgrade, and a stale binding
  surfaces as a `Forbidden` that looks like a Composition bug.

The XR had to become **cluster-scoped**: Crossplane v2 only lets a namespaced
composite compose namespaced resources, and provider-kubernetes v0.18 ships
`Object` as cluster-scoped only.

**Proof.** A claim produced a PVC, Deployment, Service and credentials Secret;
connecting with only the generated DSN reached **PostgreSQL 16.15** on database
`demoapp` and wrote a row.

*Gotcha recorded:* changing an XRD's scope needs a delete/recreate (it is
immutable), and Crossplane does **not** re-register the controller for the new
CRD — the XR sits with no status at all, which looks like a broken Composition.
Restart the Crossplane deployment.

## Phase 7 — Go operator

`DeveloperNamespace` → namespace + ResourceQuota + LimitRange + RoleBinding
bound to a Keycloak group. Cluster-scoped, so the namespace it creates can
own-reference it and be garbage-collected with it.

The LimitRange is not decoration: a quota without default container requests
admits pods that declare none and counts them as zero, which makes the quota
close to meaningless.

The controller refuses to adopt a namespace it did not create, so a
`DeveloperNamespace` naming `kube-system` cannot take it over, and reports the
refusal as `Ready=False`/`NamespaceConflict` rather than only logging it.

**The best bug of the whole exercise.** Self-healing silently did not work.
Only the Namespace carried an owner reference, and controller-runtime's
`Owns()` maps a child event back to its parent *purely* through that reference
— so deleting a ResourceQuota by hand was never noticed. The existing spec
passed because it called `Reconcile` directly, which proves the logic recreates
a deleted child but **not** that anything would notice the deletion. A spec
asserting the owner references themselves now covers it.

Two test bugs of mine also exposed a real subtlety: `ResourceList.Cpu()` looks
up the key `cpu`, while a ResourceQuota is keyed `requests.cpu`, so the helper
silently returned zero and the assertion would have passed against anything.

**Proof.** 9 envtest specs pass. In the live cluster a `medium` request
produced `requests.cpu: 4 / requests.memory: 8Gi / pods: 40` and a RoleBinding
with `role=edit subject=Group/platform-admins`. Deleting the quota by hand had
it **recreated within seconds with no manual trigger**, with the reconcile
visible in the operator log.

RBAC is deliberately narrow — namespaces, quotas, limit ranges and rolebindings
only. It cannot create Pods or read Secrets. The `bind` verb on
`view`/`edit`/`admin` is required: granting a role you do not hold is rejected
as privilege escalation.

## Phase 6 — Backstage

Scaffolded, built through CI, deployed with its own Postgres, and reachable at
`https://backstage.tailf46ba0.ts.net`. The catalog holds real platform entities
and there is a golden-path software template that renders manifests plus a
Crossplane `XDatabase` claim, opens a PR against the platform repo, and
registers the component.

Bugs found and fixed:

- The Dockerfile copies only `examples/`, so `catalog/` and `templates/` had to
  be added — otherwise the image starts with an empty catalog while the config
  points at directories that are not there. (Verified present in the image.)
- `node: bad option: --config` — the image CMD already passes both config
  files, and the node base image's entrypoint re-execs its arguments, so
  overriding `args` handed `--config` straight to node.
- Liveness killed it mid-migration. Backstage runs DB migrations for every
  plugin on first boot and needed longer than the 90s grace. A `startupProbe`
  is the right tool; liveness does not begin until startup succeeds.
- Those killed pods left `knex_migrations_lock` **held**, after which the
  catalog plugin failed with `MigrationLocked: Migration table is already
  locked`. Clearing the lock per plugin database is the fix.

**Proof.** 15 entities ingested, confirmed against the catalog database:
`auth-app`, `devns-operator`, `platform-gitops`, the `auth-api` API, the
`xdatabase` Resource, the `homelab` Domain, the `platform`/`auth` Systems, the
`platform-admins` Group, and `template:default/platform-service`.

One trap worth recording: the API returned `entities: 0` while the database
held 15. Unauthenticated requests to `/api/catalog/entities/by-query` return an
**empty list rather than a 401**, so "zero entities" looked like failed
ingestion when it was actually an auth artifact. `/api/catalog/locations` does
return `401 Missing credentials`, which is what gave it away.

**Also outstanding:** Backstage uses the **guest** auth provider. Keycloak OIDC
needs `auth-backend-module-oidc-provider` wired into the backend. The portal is
tailnet-only, so it is not publicly reachable — but SSO is not yet claimable
for Backstage specifically.

---

## Honest limitations

These matter more than the successes for interview purposes.

1. **`local-path` is node-local with no replication.** If the node holding a
   volume dies, that data is gone and the pod cannot reschedule.
   "Persistent storage with dynamic provisioning" is accurate; *replicated* or
   *HA storage* is not. Longhorn would earn the stronger claim.
2. **Single control plane.** Three nodes, one control plane, no API-server VIP,
   so the kubeconfig is pinned to one node's IP. Not HA.
3. **Mimir is not deployed.** This is Prometheus + Loki + Tempo + Grafana.
   Calling it "the LGTM stack" overstates it.
4. **Nodes use DHCP** (`.116/.118/.169`) with no reservations, which is why the
   API-server address is unstable.
5. **No etcd snapshots, no Velero, no automated Postgres backups.** The
   `talosconfig` under `~/talos/` cannot be regenerated — lose it and the
   cluster cannot be administered again.
6. **Vault needs a manual unseal after every restart.** No auto-unseal without
   a cloud KMS.
7. **No image scanning in CI.** No Trivy, no SBOM, no signing. "build + scan +
   push" is not yet true.
8. **Demo credentials.** `alice@example.com / password123` is seeded into prod
   by a PostSync hook, on an app any tailnet device can reach.
9. **`auth-app` has no rate limiting or account lockout**, which the login-spike
   runbook states plainly.
10. **Kyverno policy-as-code was not done** (Phase 4 optional item).
11. **Traces carry `service.version: dev`** — `APP_VERSION` is not wired to the
    image tag, deliberately, to keep one source of truth for the tag.
12. **Control-plane metrics are not collected** (Talos binds them to localhost).

## Reproducing the checks

```bash
kubectl --context prod get applications -n argocd
kubectl --context prod get pvc -A
kubectl --context prod get devns
kubectl --context prod get xdatabase
curl -s https://auth-app.tailf46ba0.ts.net/health
curl -s https://keycloak.tailf46ba0.ts.net/realms/homelab/.well-known/openid-configuration | jq .issuer
```

Grafana admin password, Keycloak admin password, Vault unseal key and the
sealed-secrets keys are in `~/homelab-backups/` (0600) — never in git.
