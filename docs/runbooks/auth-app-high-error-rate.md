# Runbook: AuthAppHighErrorRate

**Severity:** critical · **Fires when:** 5xx share of requests > 5% for 5m

Also covers **AuthAppHighLatency** (p95 > 1s for 10m) — same triage path.

## What this alert deliberately excludes

Only **5xx** counts. A `401` from `/auth/login` is a rejected credential, not a
service fault, and is tracked by `AuthAppLoginFailureSpike` instead. If you are
here because of a 401 spike, you want [[auth-app-login-failure-spike]].

## Find the failing route

Dashboard: **auth-app – RED** (`/d/auth-app-red`), panel *Responses by status
code*, then *p95 latency by route* to see whether errors and slowness share a
route.

```promql
sum(rate(http_server_request_duration_count{service_name="auth-app",http_response_status_code=~"5.."}[5m])) by (http_route)
```

## Get from a failing request to its cause

This is what the trace correlation is for:

1. On the dashboard's **logs** panel, filter to errors: `{app="auth-app"} | json | level="error"`
2. Click the `trace_id` on a failing line → opens the trace in Tempo
3. The span shows the route, status and duration for that exact request

## Most common causes

**Database unreachable.** Overwhelmingly the top cause — every authenticated
route touches Postgres.

```bash
kubectl --context prod get pods -n default -l app=db
kubectl --context prod logs -n default -l app=db --tail=50
```

**Volume full.** Postgres fails writes when its PVC fills → see
[[persistent-volume-almost-full]].

**Bad deploy.** Correlate the onset with the last `deploy auth-app@<sha>` commit
in the platform repo:

```bash
git -C ~/projects/platform log --oneline -10 -- manifests/auth-app
```

Roll back by reverting that commit. ArgoCD reconciles on its own.

## Related

[[auth-app-target-down]] · [[auth-app-login-failure-spike]]
