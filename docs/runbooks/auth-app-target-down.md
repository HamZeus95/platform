# Runbook: AuthAppTargetDown

**Severity:** critical · **Fires when:** `up{job="auth-app"} == 0` for 2m

Prometheus cannot scrape auth-app's metrics endpoint on `:9464`.

## First: is this user-visible?

The alert says *metrics are missing*, not *the app is down*. Check the
user-facing path before treating it as an outage:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://auth-app.tailf46ba0.ts.net/health
```

`200` means users are fine and this is a telemetry problem — lower the urgency.

## Triage

```bash
kubectl --context prod get pods -n default -l app=auth-app
kubectl --context prod describe pod -n default -l app=auth-app | tail -30
kubectl --context prod logs -n default -l app=auth-app --tail=50
```

## Likely causes

| Symptom | Cause | Action |
|---|---|---|
| Pod `Pending` | Node pressure, or the RWO PVC is bound to a down node | `kubectl --context prod get pvc -n default`; local-path is node-local, so the pod only schedules on the node holding its volume |
| Pod `CrashLoopBackOff` | Bad image or DB unreachable | Check logs; see the database section below |
| Pod `Running` but target down | `METRICS_PORT` unset, or the metrics port missing from the Service | `kubectl --context prod get svc auth-app -n default -o yaml` — port `metrics` must exist |
| All pods fine, still down | ServiceMonitor not matching | `kubectl --context prod get servicemonitor auth-app -n default -o yaml` — its selector must match the Service's `app: auth-app` label |

## Database

auth-app cannot serve without Postgres:

```bash
kubectl --context prod get pods -n default -l app=db
kubectl --context prod logs -n default -l app=db --tail=30
```

`db` uses `strategy: Recreate` with an RWO volume — during a restart there is a
short window with no database. That is expected, not a fault.

## Escalate / rollback

Roll back to the previous image by reverting the `newTag` commit in the platform
repo; ArgoCD reconciles automatically. Never edit the Deployment by hand —
`selfHeal` will revert it.

## Related

[[auth-app-high-error-rate]] · [[persistent-volume-almost-full]]
