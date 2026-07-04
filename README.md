# otelcol — zero-Docker local OpenTelemetry viewer stack (Windows)

A self-contained, no-admin way to view OpenTelemetry traces + logs locally on a
Windows box **without Docker** — useful on machines where Docker Desktop's Linux
engine can't run (e.g. Azure AMD GPU VMs with no nested virtualization). Every
component is a user-local process launched by a scheduled task.

Any app that exports OTLP to `http://localhost:18890` (the OTel SDK default HTTP
endpoint) is fully traced with **no env var**: the Collector owns that port and
fans every trace out to both dashboards.

```
your app  --OTLP :18890/:18889-->  Collector  --+--> Aspire  :4319/:4320  (traces + logs)
                                                 +--> Jaeger  :4318/:4317  (traces only)
```

| Component | UI | OTLP in | Shows |
|-----------|----|---------|-------|
| Collector (`start-otelcol.ps1`) | — | :18890 HTTP / :18889 gRPC | fan-out only |
| Aspire (`start-aspire.ps1`) | http://localhost:18888 | :4319 HTTP / :4320 gRPC | traces **+ logs** (preferred) |
| Jaeger (`start-jaeger.ps1`) | http://localhost:16686 | :4318 HTTP / :4317 gRPC | traces only (fallback) |

## Setup

```powershell
# 1. download dotnet runtime + Aspire + Jaeger + Collector into ./bin (gitignored)
powershell -ExecutionPolicy Bypass -File .\bootstrap.ps1

# 2. register the three auto-start tasks (start at logon; -AtStartup for boot-as-SYSTEM)
powershell -ExecutionPolicy Bypass -File .\register-startup-tasks.ps1
```

`bootstrap.ps1` takes optional `-AspireVersion` / `-JaegerVersion` /
`-OtelcolVersion` / `-DotnetChannel` overrides and validates `config.yaml`
against the freshly downloaded collector before finishing.

## Contents

- `bootstrap.ps1` — download all binaries into `./bin/`
- `register-startup-tasks.ps1` — (re)register the `OTel-*` scheduled tasks
- `start-otelcol.ps1` / `start-aspire.ps1` / `start-jaeger.ps1` — launchers (self-locating via `$PSScriptRoot`)
- `config.yaml` — Collector fan-out config
- `bin/` — downloaded binaries + logs (gitignored)

The launchers are location-independent (`$PSScriptRoot`), so the whole folder can
be moved or cloned anywhere.

## Design notes

- **Aspire** is the preferred viewer (unified logs + traces). Its launcher sets
  `-WorkingDirectory` to the exe dir (else Blazor `wwwroot` assets 404 → blank
  page) and `Dashboard__Frontend__AuthMode=Unsecured` (the
  `DOTNET_DASHBOARD_UNSECURED_ALLOW_ANONYMOUS` shorthand does **not** unsecure the
  browser frontend in 13.x).
- **Jaeger** is traces-only; it 404s on `/v1/logs`, so `config.yaml` routes logs
  to Aspire only.
- Collector exporters use `compression: none` — Aspire's OTLP/HTTP receiver can't
  decode gzip (returns 500 invalid-wire-type).
- All three viewers are in-memory; restarting a component clears its telemetry.
