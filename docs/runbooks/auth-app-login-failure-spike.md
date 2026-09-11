# Runbook: AuthAppLoginFailureSpike

**Severity:** warning · **Fires when:** > 30 failed logins/min for 1m

## Why this is separate from the error rate

A `401` means the app worked correctly and rejected bad credentials. Folding it
into the error rate would burn error budget on *correct* behaviour and hide real
faults. A sustained burst, though, is a credential-stuffing signal.

## Is it an attack or a broken client?

```promql
sum(rate(http_server_request_duration_count{service_name="auth-app",http_route="/auth/login",http_response_status_code="401"}[5m])) by (http_route)
```

Then read the logs — the request logger records every attempt:

```
{app="auth-app"} | json | path="/auth/login" | status=401
```

- **Many distinct emails** → credential stuffing
- **One email repeatedly** → a user locked out, or a client stuck in a retry loop
- **Started right after a deploy** → suspect the client bundle, not an attacker

## Response

This app has **no rate limiting and no account lockout** — worth knowing before
you need it. Immediate options:

1. The app is tailnet-only (`auth-app.tailf46ba0.ts.net`) plus LAN
   (`auth.lab`). It is not on the public internet, so the source is someone with
   tailnet access or a device on the LAN.
2. Identify the device in the Tailscale admin console and revoke its key.
3. For a compromised account, rotate that user's credentials.

## Follow-up

Real fixes, in order of value: per-IP rate limiting on `/auth/login`,
progressive backoff on repeated failures per account, and an audit log of
authentication events.

## Related

[[auth-app-high-error-rate]]
