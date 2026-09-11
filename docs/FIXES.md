# What was broken, and what fixed it

Every bug found while building this platform, with the symptom, the actual
cause, and the fix. Ordered by the phase it came up in.

Several of these are not specific to this homelab — the hostPath `fsGroup`
one, the `RollingUpdate` on RWO storage one, and the owner-reference one will
happen to anyone doing the same things.

## The five most instructive

If you only read part of this, read these.

| # | Symptom | Real cause |
|---|---|---|
| [1](#1-the-database-was-wiped-on-every-reboot) | Database empty after every reboot | No StorageClass at all; Postgres ran on the container's writable layer |
| [13](#13-prometheus-permission-denied-on-its-own-volume) | `permission denied` on a fresh PVC | Kubernetes does not apply `fsGroup` to **hostPath** volumes |
| [23](#23-grafana-database-is-locked) | `SQLITE_BUSY`, stuck rollout | RWO + node-local storage + `RollingUpdate` = two pods, one file |
| [40](#40-a-liveness-probe-caused-a-permanent-outage) | Portal never started again | Liveness killed it mid-migration and left a **lock** behind |
| [44](#44-self-healing-silently-did-not-work) | Deleted quota never came back | `Owns()` maps events **only** through owner references |

---

## Phase 0 — Storage

### 1. The database was wiped on every reboot

**Symptom.** The prod database was empty after every power cycle. It had
happened five times.

**Cause.** Talos ships **no storage provisioner**, so `kubectl get sc` returned
nothing and the Postgres Deployment had no `volumes` or `volumeMounts` at all.
`PGDATA` lived on the container's writable layer, which is destroyed with the
container. Every boot, `initdb` ran into an empty directory.

**Fix.** Vendored `local-path-provisioner`, prod-only, and gave Postgres a real
PVC. Proven with a genuine power cycle: a row written beforehand survived.

### 2. Two Postgres processes on one data directory

**Cause.** A Deployment with an RWO volume and the default `RollingUpdate`
strategy. Because `local-path` is node-local and binds with
`WaitForFirstConsumer`, the scheduler puts the *new* pod on the *same node*,
where it can also mount the volume — so both write to one `PGDATA`.

**Fix.** `strategy: Recreate` on every stateful workload. This trap recurred
twice more (Grafana, Keycloak's database).

### 3–4. Talos rejected the provisioner

`local-path` defaults to `/opt/local-path-provisioner`, but Talos's root
filesystem is read-only apart from `/var`. And its helper pods bind-mount a
hostPath, which Talos's cluster-wide PodSecurity `baseline` forbids.

**Fix.** Path moved to `/var/local-path-provisioner`, plus a
`pod-security.kubernetes.io/enforce: privileged` label on that namespace.

## Phase 1 — Secrets

### 5–7. The sealed-secrets controller quietly gave up

**Symptom.** After replacing an imperatively-created Secret with a
`SealedSecret`, the Secret was never created. The status stayed stale.

**Cause.** Two layers. The controller refuses to adopt a Secret it does not
own — reasonable. But after deleting the old one it *still* did nothing,
because it had already logged `Error updating, giving up` (five retries, then
dropped from the workqueue), and a nudge via annotation was rejected with
`update suppressed, no changes in spec` — it only reacts to **spec** changes.

**Fix.** Restart the controller to force a resync. Worth knowing: a
sealed-secrets controller that has given up will not recover on its own.

## Phase 2 — Ingress, TLS, CI

### 8. ingress-nginx was permanently OutOfSync

**Cause.** Its two admission Jobs ship `ttlSecondsAfterFinished: 0`, so they
delete themselves the instant they succeed. ArgoCD compared git (job present)
to the cluster (job gone), recreated them, and never converged.

**Fix.** Run them as ArgoCD `Sync` hooks, which excludes them from the
comparison.

### 9. cert-manager could not be applied at all

**Cause.** Its CRDs are larger than the 262144-byte cap on the
`last-applied-configuration` annotation that client-side apply writes.

**Fix.** `ServerSideApply=true`.

### 10. Production was configured with `localhost`

`CLIENT_URL: http://localhost:3000` was baked into the shared manifests, so
both clusters got it.

**Fix.** Kustomize base + per-environment overlays.

### 11. A silent CI failure that would have stopped all deployments

**Cause.** CI rewrote the image tag with `sed` on a fixed path, and ended with
`git commit ... || echo "no changes"`. When the manifests moved to `base/`, a
no-op `sed` became **indistinguishable from "already up to date"** — deployments
would have silently stopped forever.

**Fix.** Verify the file exists, compare checksums before and after, and fail
loudly if nothing matched.

### 12. The Tailscale operator crash-looped

```
requested tags [tag:k8s-operator] are invalid or not permitted (400)
```

The chart tags the operator's own node `tag:k8s-operator`, but the tailnet ACL
and OAuth client only authorized `tag:k8s`.

**Fix.** Point the operator at the tag that exists, rather than widening the
credential's scope.

## Phase 3 — Observability

### 13. Prometheus: permission denied on its own volume

**Symptom.**

```
open /prometheus/queries.active: permission denied
```

on a freshly-provisioned, empty PVC — despite `fsGroup: 2000` being set.

**Cause.** This is the most reusable lesson here. `local-path` provisions
**hostPath** PVs, and **Kubernetes does not apply `fsGroup` ownership to
hostPath volumes**. The volume stayed `root:root`, and the kubelet-created
`prometheus-db` **subPath** directory was `root:root 0755` — unwritable by
uid 1000. Grafana worked only because its chart already ships an
`init-chown-data` container.

**Fix.** The same init container for Prometheus, and pre-emptively for Loki.

**Tempo needed none** — it mounts the volume *root*, which local-path creates
`0777`, with no subPath. That confirms the subPath directory was the actual
culprit, and explains why Postgres never hit it.

### 15. Control-plane scrape targets would never work

Talos binds kube-scheduler and kube-controller-manager metrics to `127.0.0.1`.

**Fix.** Disable those ServiceMonitors rather than leave permanently-DOWN
targets, which is why the target count reads a clean 22/22. Enabling them needs
a machine-config patch setting `bind-address=0.0.0.0`.

### 16. Syncing the child app did nothing (a process bug)

Repeatedly syncing the `monitoring` app never picked up new Helm values. Its
synced "revision" is `90.0.0` — the **chart version**, not a git SHA. The values
live in the `Application` object, which the **root** app owns.

**Rule.** When the change is inside an `Application`, sync **root first**.

### 17. Every metric series was labelled `service_name=""`

**Cause.** `@hono/otel` stamps its *own* `service_name` metric attribute and
defaults it to an empty string rather than reading the provider's resource.
The dashboard could not have filtered by service.

**Fix.** Pass `serviceName`/`serviceVersion` to the middleware explicitly.
Caught in local testing before it shipped.

### 18. A healthy service showed "No data" for its error rate

**Cause.** With no 5xx series in range, the numerator is empty — and in PromQL
empty divided by anything is empty. The error-rate tile rendered **"No data"**,
which is indistinguishable from broken monitoring.

**Fix.** `or vector(0)` plus an explicit `noValue`, so healthy reads `0%`.

### 19. The Deployment and the seed Job could drift onto different builds

**Fix.** Move the image tag into each overlay's kustomize `images:`
transformer, so one value drives every container using that image. CI now
rewrites `newTag` instead of a raw `image:` line.

### 21. Login returned 401 with a perfectly good database

**Cause.** Not a broken migration — `migrate()` had correctly created `users`
and `refresh_tokens` on boot. The table was simply **empty**, because the
Phase 0 volume migration produced a fresh `PGDATA` and nothing ever inserted a
user. The 401 was the application behaving correctly.

**Fix.** Seed as an ArgoCD **PostSync hook**, so a rebuilt cluster is demo-able
with no manual steps.

## Phase 4 — Vault, Keycloak, SSO

### 22. Keycloak rejected every login with `invalid_scope`

Requesting `scope=groups` asks for a **client scope** of that name, which did
not exist. The groups claim comes from a protocol **mapper** on the client's
dedicated scope, which is applied on every request regardless of scopes.

**Fix.** Drop `groups` from the requested scopes.

### 23. Grafana: `database is locked`

**Symptom.** `database is locked (5) (SQLITE_BUSY)` and a rollout that never
finished, with two Grafana pods Running.

**Cause.** Grafana keeps state in SQLite on an RWO PVC. Node-local storage put
both the old and new pod on the same node, where **both** mounted the volume —
two Grafanas writing one `grafana.db`. The same trap as #2.

**Fix.** `strategy: Recreate`.

### 24. …and applying that fix failed

```
spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy type is 'Recreate'
```

Server-side apply **merges** rather than removes, so the stale `rollingUpdate`
block survived and blocked the change.

**Fix.** Clear the field once; GitOps converges after.

### 25. SSO failed at the token exchange

```
dial tcp: lookup keycloak.tailf46ba0.ts.net on 10.96.0.10:53: no such host
```

**Cause.** The classic OIDC-in-Kubernetes split. The **browser** must use the
public MagicDNS name, but the token exchange happens **server-side inside the
cluster**, where CoreDNS has no idea what a tailnet name is.

**Fix.** `auth_url` stays public; `token_url` and `api_url` use in-cluster
Service DNS.

ArgoCD could not use the same trick — it validates that the issuer it fetches
matches the one configured. Solved instead by giving Keycloak a `homelab-ca`
certificate covering its public hostname, serving HTTPS in-cluster, and adding
a host alias.

### 26. `unauthorized_client: Invalid client credentials`

The client secret had been captured with a `print()`, which appends a newline,
and `--from-file` preserves bytes exactly. The cluster held **33 bytes where
Keycloak expected 32**.

### 27–29. Keycloak's own probes

- Without `KC_PROXY_HEADERS`, OIDC discovery advertises `http://` URLs and every
  redirect breaks. (Pre-empted.)
- Configuring a TLS certificate also switches the **management port to HTTPS**,
  so plain-HTTP probes fail and the pod never goes Ready.
- The default `timeoutSeconds: 1` is not enough for a TLS handshake against a
  JVM: `context deadline exceeded while awaiting headers`.

### 30–32. ArgoCD configuration traps

- `hostAliases` is a **`global.`** value in that chart, not `server.`.
- A secret is referenced as **`$secret-name:key`**. The bare `$key` form only
  looks inside `argocd-secret`.
- `helm upgrade --reuse-values` against a newer chart than the installed
  release fails with a nil-pointer template error. Pin the chart version.

## Phase 5 — Crossplane

### 33–35. The claim produced nothing

- `cannot apply cluster scoped composed resource ... for a namespaced composite
  resource` — Crossplane v2 only lets a namespaced composite compose namespaced
  resources, and provider-kubernetes ships `Object` as cluster-scoped only.
  **Fix:** cluster-scoped XR with the namespace as a spec field.
- An XRD's **`scope` is immutable**, so that change needs a delete and
  recreate.
- After recreating it, Crossplane **does not re-register the controller** for
  the new CRD. The XR sits with **no status at all**, which looks exactly like
  a broken Composition. **Fix:** restart the Crossplane deployment.

### 36–37. Two problems avoided by design

- The database password is derived deterministically from the composite's UID.
  `randAlphaNum` would regenerate on every reconcile, rewriting the Secret and
  breaking the database it had just provisioned.
- Provider RBAC binds to the service-account **group**, not the provider's SA
  name — that name carries a hash that changes on upgrade, and a stale binding
  surfaces as a `Forbidden` that looks like a Composition bug.

## Phase 6 — Backstage

### 38. The image had an empty catalog

Upstream's Dockerfile copies only `examples/`. The config pointed at
`./catalog` and `./templates`, which were not in the image.

**Fix.** `COPY` them explicitly.

### 39. `node: bad option: --config`

The image's `CMD` already passes both config files, and the node base image's
entrypoint re-execs its arguments — so overriding `args` handed `--config`
straight to node.

### 40. A liveness probe caused a permanent outage

**Symptom.** `Liveness probe failed ... connection refused`, looping, while the
logs showed plugins still initializing. Then, on every later start:

```
Plugin 'catalog' startup failed; caused by MigrationLocked: Migration table is already locked
```

**Cause.** Backstage runs database migrations for every plugin on first boot,
which took longer than the liveness probe's 90-second grace. The kubelet killed
it mid-migration — and each killed pod left `knex_migrations_lock` **held**, so
the damage outlived the probe that caused it.

**Fix.** A **`startupProbe`**, so liveness does not begin until startup
succeeds, and clear the stale lock:

```sql
UPDATE knex_migrations_lock SET is_locked = 0;   -- per backstage_plugin_* database
```

### 42. `entities: 0` while the database held 15

**A diagnostic trap, not a bug.** Unauthenticated requests to
`/api/catalog/entities/by-query` return an **empty list rather than a 401**, so
a missing credential looks identical to failed ingestion. The catalog had been
working the whole time.

`/api/catalog/locations` *does* return `401 Missing credentials` — that is the
quickest way to tell the two apart. When in doubt, query the database:

```sql
SELECT count(*) FROM final_entities;
```

## Phase 7 — The Go operator

### 44. Self-healing silently did not work

**Symptom.** Deleting a `ResourceQuota` by hand in a live cluster — it never
came back. Meanwhile the test suite was green.

**Cause.** Only the Namespace carried an owner reference. controller-runtime's
`Owns()` maps a child event back to its parent **purely** through the owner
reference, so with none on the quota, limit range and role binding, **no event
ever reached the controller**.

**Fix.** `SetControllerReference` on every child.

### 45. …and the test that should have caught it

The existing spec called `Reconcile` directly. That proves the logic recreates
a deleted child, but **not that anything would ever notice the deletion**. A
spec asserting the owner references themselves is what actually covers it.

Worth generalising: testing a reconciler by calling it directly never exercises
the event plumbing.

### 46. An assertion that passed against anything

`ResourceList.Cpu()` looks up the key `cpu`, while a ResourceQuota is keyed
`requests.cpu`. The helper silently returned zero, so the assertion would have
passed against any quota at all. Index the key explicitly.

---

## Patterns worth carrying forward

1. **Node-local storage changes the rules.** Anything RWO needs
   `strategy: Recreate`, because both pods land on the same node and both can
   mount. `fsGroup` does not apply to hostPath, so non-root containers writing
   to a `subPath` need a chown init container.
2. **A probe can create damage that outlives it.** A liveness probe that fires
   during a migration leaves a lock behind. Use `startupProbe` for anything
   with a slow first boot.
3. **Green tests are not green behaviour.** Two bugs here passed their tests:
   the owner references (test called the function directly) and the quota
   assertion (helper read the wrong key).
4. **"No data" and "zero" must be distinguishable** — on a dashboard tile and in
   an API response alike. Both cost real debugging time here.
5. **Silent success is worse than failure.** The CI `|| echo "no changes"` would
   have stopped every deployment without a single red build.
6. **In OIDC, browser-facing and server-facing URLs are different things.** The
   browser needs a public name; the server needs one it can resolve.
