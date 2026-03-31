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
