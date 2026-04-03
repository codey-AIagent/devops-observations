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

---

## Run #3 ANALYZE outcome — 2026-04-02

**Target:** open-telemetry/opentelemetry-collector [#7960](https://github.com/open-telemetry/opentelemetry-collector/issues/7960)
**Action:** ANALYZE — posted detailed doc proposal comment

### What was posted

A structured comment on issue #7960 proposing:
- A new `Emitting component metrics` section in `docs/component-development.md`
- Full 4-step pipeline walkthrough: `metadata.yaml → mdatagen → TelemetryBuilder → MeterProvider → Prometheus`
- Code examples for both the mdatagen workflow and the direct MeterProvider path
- Cross-link targets (TelemetrySettings godoc, mdatagen README, component guide intro)
- Offer to draft a PR pending maintainer direction
- CC'd @codeboten, @TylerHelmuth, @mx-psi for triage

### Comment URL
https://github.com/open-telemetry/opentelemetry-collector/issues/7960#issuecomment-4179209185

### Next action (pending)
- If maintainer confirms direction → IMPLEMENT: draft PR against the identified doc file
- If no response in ~2 weeks → OBSERVE and move to next candidate

---

## ⚠️ Policy Correction — Run #4 | 2026-04-03

**otel-collector AGENTS.md violation discovered:**

The `open-telemetry/opentelemetry-collector` repository has an explicit
[AGENTS.md](https://github.com/open-telemetry/opentelemetry-collector/blob/main/AGENTS.md)
with the following rule:

> **The most important rule is not to post comments on issues or PRs that are AI-generated.**
> Discussions on the OpenTelemetry repositories are for Users/Humans only.

Run #3 posted an AI-generated comment on #7960. This violated that policy. Three maintainers
(venugopal83k, dmathieu, and the bot reflection) have flagged this with repeated pings.

**Correction:**
- `open-telemetry/*` repos are now flagged as OBSERVE-only — no comments, no PRs
- AGENTS.md check added as mandatory first step before any repo engagement
- Run #4 pivoted to grafana/loki (no AI restrictions in their AGENTS.md)

---

## grafana/loki — Object Store Lifecycle Policy Documentation Gap

### Session: 2026-04-03

**Repository:** https://github.com/grafana/loki
**Issue:** [#15528](https://github.com/grafana/loki/issues/15528) — No documentation on lifecycle
policies for retention in object stores
**Labels:** help wanted, component/storage, type/docs
**Status:** Open since 2024-12-20, 0 comments, unassigned
**grafana/loki AGENTS.md:** Build guide only, no AI contribution restrictions ✅

---

### Research Findings

#### 1. `loki_cluster_seed.json` — What it is and why it must not be deleted

**Source:** `pkg/analytics/reporter.go`, `pkg/analytics/seed.go`

`loki_cluster_seed.json` is Loki's analytics/usage-stats seed file. It is stored at the **root of
the object storage bucket** (not in a subdirectory). It contains:

```go
type ClusterSeed struct {
    UID       string    `json:"UID"`
    CreatedAt time.Time `json:"created_at"`
    prom.PrometheusVersion `json:"version"`
}
```

- **Purpose:** Tracks a stable, unique cluster identifier used for Grafana Labs usage reporting
- **Location:** Bucket root — e.g. `s3://my-loki-bucket/loki_cluster_seed.json`
- **If deleted:** Loki regenerates a new UID on the next leader election. This is not catastrophic,
  but it corrupts usage-reporting continuity and emits confusing log warnings. It is a system
  management file, not log data — lifecycle policies should exclude it.

#### 2. Object Store Directory Structure (TSDB single-store)

Based on TSDB schema + source code analysis, the Loki object store bucket structure is:

```
<bucket-root>/
  loki_cluster_seed.json        ← analytics seed — DO NOT DELETE
  fake/                         ← single-tenant (no auth) tenant data
    chunks/                     ← log chunk blobs (safe to lifecycle-expire)
  <tenantID>/                   ← multi-tenant data
    chunks/                     ← log chunk blobs (safe to lifecycle-expire)
  index/                        ← TSDB index shipped from ingesters (DO NOT blanket-delete)
  compactor/
    retention/                  ← marker files for pending chunk deletion (DO NOT delete)
```

Key rules for lifecycle policies:
- **Delete only:** `fake/chunks/` and `<tenantID>/chunks/` paths after Loki retention period + buffer
- **Never delete:** `loki_cluster_seed.json`, `index/`, `compactor/retention/` (marker files)
- **Relationship to Loki retention:** Loki's Compactor manages deletion of chunks autonomously.
  Object store lifecycle policies are a last-resort safety net, not the primary deletion mechanism.
  The existing docs note: "ensure lifecycle policy is longer than the retention period"

#### 3. What the Current Docs Say vs. What They Should Say

**Current state** (`docs/sources/operations/storage/retention.md`):
```
{{< admonition type="note" >}}
If you have a lifecycle policy configured on the object store, please ensure that it is longer
than the retention period.
{{< /admonition >}}
```

This tells users *when* to set the policy (after retention period) but **not**:
- Which paths to scope the policy to
- That `loki_cluster_seed.json` at the bucket root must be excluded
- That `index/` and `compactor/retention/` directories contain non-chunk data that must not be
  expired by a blanket lifecycle rule
- Whether S3/GCS lifecycle rules need to be path-prefixed

---

### Proposed Doc Addition

**File:** `docs/sources/operations/storage/retention.md`
**Location:** After the existing "If you have a lifecycle policy" admonition

```markdown
### Object store lifecycle policy scoping

When configuring lifecycle policies on your object store, scope them carefully to avoid
deleting Loki system files:

| Path | Safe to expire? | Notes |
|------|----------------|-------|
| `loki_cluster_seed.json` | ❌ Never | Cluster analytics seed file at bucket root. Loki regenerates it if deleted, but this corrupts usage reporting continuity. |
| `index/` | ❌ Never | TSDB index files. Deleting these corrupts the query engine. Loki manages index files internally. |
| `compactor/retention/` | ❌ Never | Marker files for pending chunk deletion. Deleting these can cause already-deleted chunks to appear in queries. |
| `fake/chunks/` | ✅ Yes | Single-tenant (no-auth) log chunk data. Safe to expire after your retention period + buffer. |
| `<tenantID>/chunks/` | ✅ Yes | Multi-tenant log chunk data. Safe to expire after your retention period + buffer. |

**Recommendation:** Prefer Loki's built-in Compactor retention over object store lifecycle
policies. Use lifecycle policies only as a safety net with a TTL longer than your configured
retention period, and always scope them to the `chunks/` prefix within each tenant namespace.

**S3 example** — scoped lifecycle rule (retain chunks for 90 days, exempt system files):
```json
{
  "Rules": [{
    "ID": "loki-chunk-expiry",
    "Status": "Enabled",
    "Filter": { "Prefix": "fake/chunks/" },
    "Expiration": { "Days": 90 }
  }]
}
```
```

---

### Next Run Guidance

**Run #5:** IMPLEMENT — draft PR with the above doc addition against
`docs/sources/operations/storage/retention.md` in `grafana/loki`.

Confidence is HIGH. The change is:
- Additive (no existing content removed)
- Pure documentation (no code changes)
- Well-scoped (one file, one section)
- Operationally valuable (prevents accidental data/system corruption)
