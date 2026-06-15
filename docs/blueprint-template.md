# Day 13 Observability Lab Report

> **Instruction**: Fill in all sections below. This report is designed to be parsed by an automated grading assistant. Ensure all tags (e.g., `[GROUP_NAME]`) are preserved.

## 1. Team Metadata
- NAME: Lê Quang Miền
- REPO_URL: https://github.com/LeQuangMien/2A202600715-LeQuangMien-Day13

---

## 2. Group Performance (Auto-Verified)
- VALIDATE_LOGS_FINAL_SCORE: 100/100
- TOTAL_TRACES_COUNT: 175
- PII_LEAKS_FOUND: 0

---

## 3. Technical Evidence (Group)

### 3.1 Logging & Tracing
- EVIDENCE_CORRELATION_ID_SCREENSHOT: docs/evidence/correlation_id.png
- EVIDENCE_PII_REDACTION_SCREENSHOT: docs/evidence/pii_redaction.png
- EVIDENCE_TRACE_WATERFALL_SCREENSHOT: docs/evidence/langfuse_traces.png
- TRACE_WATERFALL_EXPLANATION: The `agent.run` span is the root observation. Inside it, `retrieve()` accounts for the majority of wall-clock time under `rag_slow` incident (5–13s), clearly visible as a wide span before the `FakeLLM.generate` call. Under normal conditions the entire span completes in ~150ms, confirming the bottleneck is exclusively in the RAG retrieval layer.

### 3.2 Dashboard & SLOs
- DASHBOARD_6_PANELS_SCREENSHOT: docs/evidence/dashboard.png
- SLO_TABLE:

| SLI | Target | Window | Current Value |
|---|---:|---|---:|
| Latency P95 | < 3000ms | 28d | 150ms |
| Error Rate | < 2% | 28d | 0% |
| Cost Budget | < $2.5/day | 1d | $0.0228 |
| Quality Score | ≥ 0.75 | 28d | 0.88 |

### 3.3 Alerts & Runbook
- ALERT_RULES_SCREENSHOT: docs/evidence/alert_rules.png
- SAMPLE_RUNBOOK_LINK: docs/alerts.md#1-high-latency-p95

---

## 4. Incident Response (Group)

### Incident 1: rag_slow
- SCENARIO_NAME: rag_slow
- SYMPTOMS_OBSERVED: End-to-end latency spiked from p95=151ms to p95=2651ms. Load test showed individual requests taking 5–13 seconds. No HTTP errors — all responses returned 200 OK. Quality score unchanged at 0.88.
- ROOT_CAUSE_PROVED_BY: Langfuse trace waterfall shows the `agent.run` span duration expanded while the sub-span for `retrieve()` held the majority of elapsed time. Log field `latency_ms` in `response_sent` events jumped from ~150 to ~5000–13000. Confirmed by injecting `rag_slow=True` toggle in `app/incidents.py` which adds artificial delay to `mock_rag.retrieve()`.
- FIX_ACTION: `python scripts/inject_incident.py --scenario rag_slow --disable`
- PREVENTIVE_MEASURE: Add p95 latency alert (`high_latency_p95` in `alert_rules.yaml`) with threshold 5000ms / 30m. Add RAG-specific span timing in Langfuse to detect retrieval slowness independently of LLM latency.

### Incident 2: tool_fail
- SCENARIO_NAME: tool_fail
- SYMPTOMS_OBSERVED: 100% of requests returned HTTP 500. Correlation ID missing from responses (`None`). `error_breakdown: {RuntimeError: 10}` appeared in `/metrics`. Latency was extremely low (~16–23ms) because the pipeline failed before reaching LLM.
- ROOT_CAUSE_PROVED_BY: Log entries with `"level": "error"` and `"error_type": "RuntimeError"` in `data/logs.jsonl`. Exception propagated from agent pipeline through the `except Exception` handler in `main.py` before `response_sent` could be logged, explaining missing correlation ID in client response.
- FIX_ACTION: `python scripts/inject_incident.py --scenario tool_fail --disable`
- PREVENTIVE_MEASURE: `high_error_rate` alert (P1, >5% for 5m). Add retry logic with exponential backoff in `agent.py` for transient tool failures before raising to HTTP 500.

### Incident 3: cost_spike
- SCENARIO_NAME: cost_spike
- SYMPTOMS_OBSERVED: Requests succeeded (200 OK) but `avg_cost_usd` increased 86% ($0.0021 → $0.0039) and `tokens_out_total` increased 182% (2,670 → 7,538). No latency increase, no errors — incident was silent without cost monitoring.
- ROOT_CAUSE_PROVED_BY: `/metrics` showed `total_cost_usd` jump from $0.0421 to $0.1161 after 10 requests. Langfuse trace `usage` field showed abnormally high `output` token counts per trace. `response_sent` log entries confirmed elevated `tokens_out` values.
- FIX_ACTION: `python scripts/inject_incident.py --scenario cost_spike --disable`
- PREVENTIVE_MEASURE: `cost_budget_spike` alert (P2, >2x baseline for 15m). Add per-request token budget cap in `mock_llm.py`. Track `tokens_out` as a dedicated metric with anomaly threshold.

---

## 5. Bonus Items (Optional)
- BONUS_COST_OPTIMIZATION: Not implemented
- BONUS_AUDIT_LOGS: Not implemented
- BONUS_CUSTOM_METRIC: Not implemented
