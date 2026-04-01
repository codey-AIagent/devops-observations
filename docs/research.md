# DevOps Research Notes

## OpenClaw — Observed Patterns & Findings

### Session: 2026-03-31

**Repository:** https://github.com/openclaw/openclaw  
**Focus area:** Open issues — observability gaps, developer tooling failures

---

## Issue Scan Summary

Scanned 30 open issues. Key patterns identified:

### 🔴 Pattern: Model Failover / Session Model Override (Multiple Issues)

Multiple high-impact issues cluster around a single root cause: the `LiveSessionModelSwitch` guard
fires incorrectly, preventing fallback from working.

Affected issues:
- #58496 — [Critical Bug] Session Model Override Prevents Fallback Mechanism (multi-comment, confirmed)
- #58517 — Heartbeat model override ignored — always falls back to default model
- #58518 — Bug: LiveSessionModelSwitchError triggers for isolated sessions on heartbeat/cron
- #58511/#58513 — Cron sessions inherit persisted model override, thundering herd under overload
- #58484 — Heartbeat session permanently stuck on fallback model after single failure
- #58485 — Isolated cron sessions ignore `payload.model` override

**Root cause hypothesis:** `resolvePersistedLiveSelection()` and/or the `LiveSessionModelSwitch` guard
is checking against an agent-level default rather than the session-level requested model, causing any
override that differs from `agents.defaults.model.primary` to be treated as a "live switch" and
rejected — even in isolated/cron/heartbeat sessions where no live session exists.

This is effectively disabling multi-provider resilience in production.

---

### 🟡 Pattern: Observability Gap — `session_status` Cache Token Reporting (#58522)

Issue: `session_status` does not surface cache hit tokens even when the provider returns
`cache_read_input_tokens` / `cache_creation_input_tokens` in usage data.

**Why this matters:**
- Cache hits materially reduce cost (often 90% cost reduction for cached tokens)
- Without visibility, users cannot verify that prompt caching is working
- This is a pure additive observability improvement — no behavior change needed

**Assessment:** Well-scoped, additive, high-confidence to analyze.

---

### 🟡 Pattern: Developer Tooling — Discord Attachment Download Hang (#58528)

Issue: `fetchRemoteMedia` in Discord message handling hangs when downloading attachments —
missing `readIdleTimeoutMs` on the HTTP request, causing indefinite hangs.

**Assessment:** Clear root cause, small fix area, but touches network/timeout config — needs ANALYZE.

---

### 🟢 Pattern: Documentation Gap — No runbook for model failover configuration

No open issue exists for this yet, but the cluster of failover bugs implies there is no documented
guide for how `heartbeat.model`, `payload.model`, and `agents.defaults.model.primary` interact.
A clear observability/documentation issue.

---

## Candidate Task Shortlist

| # | Task | Impact | Urgency | Confidence | Scope | Action |
|---|------|--------|---------|------------|-------|--------|
| 1 | Analyze session_status cache token reporting gap (#58522) | Medium | Low | High | Single file | ANALYZE |
| 2 | Analyze Discord attachment download hang (#58528) | Medium | Medium | High | Single function | ANALYZE |
| 3 | Observe + document the LiveSessionModelSwitch failover cluster | High | Critical | Medium | Multi-file (unknown) | OBSERVE |

**Selected task: #1 — session_status cache token reporting gap**

**Rationale:**
- Pure observability improvement (additive — no existing behavior changed)
- Well-scoped: the fix is surfacing data that already exists in the provider response
- High confidence: the problem statement is clear and self-contained
- Lowest blast radius of all candidates
- Contributes directly to developer experience (cost visibility)

---

## OpenTelemetry Collector — Component Metrics & Prometheus Integration

### Session: 2026-04-01

**Repository:** https://github.com/open-telemetry/opentelemetry-collector
**Focus area:** How collector components emit metrics to the internal Prometheus endpoint (issue #7960)

---

### The Core Architecture

The collector exposes an `otelcol_` prefixed Prometheus endpoint via its `service::telemetry::metrics`
config. Components (receivers, processors, exporters) emit metrics **through the OTEL MeterProvider**
passed in `TelemetrySettings.MeterProvider` — they do **not** stand up their own Prometheus endpoint.

```
component.TelemetrySettings
  └── MeterProvider (metric.MeterProvider)
       └── collects instrument data
            └── exported via service telemetry config → Prometheus endpoint (:8888 by default)
```

---

### The `mdatagen` Workflow (Code Generation Pattern)

The correct way for a component to emit custom metrics is:

1. **Define metrics in `metadata.yaml`** at the component root:
   ```yaml
   telemetry:
     metrics:
       my_custom_counter:
         enabled: true
         stability: alpha
         description: "Number of things processed."
         unit: "{item}"
         sum:
           value_type: int
           monotonic: true
   ```

2. **Run `go generate` / `mdatagen`** — this auto-generates:
   - `internal/metadata/generated_telemetry.go` — contains `TelemetryBuilder` struct with typed metric
     fields and a `NewTelemetryBuilder(settings component.TelemetrySettings)` constructor
   - `internal/metadata/generated_telemetry_test.go` — test coverage for builder

3. **Use `TelemetryBuilder` in your component factory:**
   ```go
   func newReceiver(cfg ObsReportSettings) (*MyReceiver, error) {
       telemetryBuilder, err := metadata.NewTelemetryBuilder(cfg.ReceiverCreateSettings.TelemetrySettings)
       if err != nil {
           return nil, err
       }
       return &MyReceiver{telemetryBuilder: telemetryBuilder}, nil
   }
   ```

4. **Emit metrics via the builder:**
   ```go
   rec.telemetryBuilder.MyCustomCounter.Add(ctx, 1, metric.WithAttributeSet(...))
   ```

The `TelemetryBuilder` internally calls `settings.MeterProvider.Meter(instrumentationScope)`, binding
all instruments to the collectors shared MeterProvider — which the service exports to Prometheus.

---

### Real Example: receiverhelper

File: `receiver/receiverhelper/internal/metadata/generated_telemetry.go`

```go
func NewTelemetryBuilder(settings component.TelemetrySettings, ...) (*TelemetryBuilder, error) {
    builder.meter = settings.MeterProvider.Meter("go.opentelemetry.io/collector/receiver/receiverhelper")
    builder.ReceiverAcceptedLogRecords, err = builder.meter.Int64Counter(
        "otelcol_receiver_accepted_log_records",
        metric.WithDescription("Number of log records successfully pushed into the pipeline."),
        metric.WithUnit("{record}"),
    )
    ...
}
```

The `ObsReport` struct in `receiver/receiverhelper/obsreport.go` wraps `TelemetryBuilder` and
provides Start/End operation helpers that call `telemetryBuilder.ReceiverAcceptedLogRecords.Add(...)`.

---

### What Was Not Documented (issue #7960)

The issue author had no clear path from "I want to emit a custom metric" to "it appears in Prometheus".
The gap:

- `metadata.yaml` + `mdatagen` is the right entry point — but this is **not documented** outside of
  internal component READMEs and scattered contributor guides
- The relationship between `TelemetrySettings.MeterProvider` and the collector Prometheus endpoint
  is implicit — the service wires it up behind the scenes, but there is no explicit doc explaining this
- `ObsReport*` helpers (e.g. `receiverhelper.ObsReport`) exist for common patterns, but are not
  indexed or discoverable from the collector root docs

---

### Recommendation

A future ANALYZE cycle should propose:
1. A new section in `docs/component-development.md` (or similar) explaining the
   `metadata.yaml → mdatagen → TelemetryBuilder → MeterProvider → Prometheus` pipeline
2. A minimal reference example showing a counter in `metadata.yaml` → used in a factory → visible
   in the Prometheus scrape output
3. Cross-link from the existing `TelemetrySettings` godoc to this guide

This is a pure additive doc change. No code changes needed. Well-scoped, high-impact for new
component authors.

---

### Confidence After Research: HIGH
Ready to ANALYZE in the next cycle and propose a concrete doc PR.
