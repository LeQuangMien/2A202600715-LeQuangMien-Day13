# Alert Runbooks

## 1. high-latency-p95
**Condition:** p95 latency > 5000ms for 30 minutes  
**Severity:** P2  
**Steps:**
1. Check `/metrics` endpoint for current latency distribution
2. Check Langfuse traces — filter by high duration to find slow spans
3. If RAG slow: check `rag_slow` incident toggle via `inject_incident.py`
4. Escalate to on-call if not resolved in 30m

## 2. high-error-rate
**Condition:** error_rate > 5% for 5 minutes  
**Severity:** P1  
**Steps:**
1. Check `data/logs.jsonl` for `"level": "error"` entries
2. Look at `error_type` field — common: `TypeError`, `TimeoutError`
3. Check incident toggles: `curl localhost:8000/health`
4. Disable faulty incident: `python scripts/inject_incident.py --scenario tool_fail --disable`

## 3. cost-budget-spike
**Condition:** hourly cost > 2x baseline for 15 minutes  
**Severity:** P2  
**Steps:**
1. Check `/metrics` for `cost_usd_total`
2. Check Langfuse for traces with abnormally high token counts
3. Check `cost_spike` incident toggle
4. Notify finops-owner if confirmed

## 4. low-quality-score
**Condition:** avg quality_score < 0.6 for 15 minutes  
**Severity:** P2  
**Steps:**
1. Check Langfuse traces — filter by low quality_score metadata
2. Verify RAG docs are being retrieved (`doc_count` in trace metadata)
3. Check if `rag_slow` incident is causing empty doc returns

## 5. rag-retrieval-slow
**Condition:** RAG p95 latency > 2000ms for 10 minutes  
**Severity:** P3  
**Steps:**
1. Run `python scripts/inject_incident.py --scenario rag_slow` to confirm reproduction
2. Check Langfuse span duration for `retrieve()` calls
3. Disable incident: `python scripts/inject_incident.py --scenario rag_slow --disable`