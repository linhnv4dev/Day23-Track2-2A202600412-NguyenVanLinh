# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** NGUYỄN VĂN LĨNH
**MHV:** 2A202600412
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/linhnv4dev/Day23-Track2-2A202600412-NguyenVanLinh

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.1)
Compose v2:    OK  (5.1.3)
RAM available: 23.31 GB (OK)
Ports free:    OK
Report written: 00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When     | What                | Evidence                             |
| -------- | ------------------- | ------------------------------------ |
| _T0_     | killed `day23-app`  | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired | screenshot `slack-firing.png`        |
| _T1_     | restored app        | —                                    |
| _T1+60s_ | alert resolved      | screenshot `slack-resolved.png`      |

### One thing surprised me about Prometheus / Grafana

PromQL's instant-vector vs range-vector distinction caught me off guard. Writing `rate(inference_requests_total[5m])` for the RPS panel failed silently when I accidentally used `rate(inference_requests_total)` without the range selector — the panel showed "No Data" with no error message. This taught me that Prometheus's error model is intentionally quiet: missing data is a signal, not a crash, and dashboards must be designed to fail-soft just like the queries they run.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{
    "model": "llama3-mock",
    "input_tokens": 4,
    "output_tokens": 54,
    "quality": 0.783,
    "duration_seconds": 0.1898,
    "trace_id": "affafc06d310287bf408e1a258796441",
    "event": "prediction served",
    "level": "info",
    "timestamp": "2026-05-11T06:25:47.234647Z"
}
```

Trace ID: `affafc06d310287bf408e1a258796441`

### Tail-sampling math

In this lab, under `make load` (10 concurrent users for 60s), the app produces approximately 2-3 traces/sec. The tail-sampling policy keeps:

- 100% of error traces (status_code == ERROR)
- 100% of slow traces (duration > 2s)
- 1% of healthy traces (probabilistic, rate=0.01)

Under steady-state with 2 traces/sec for 60s (120 traces), 1 forced error was generated. At a nominal 0.5% error rate, ~1 trace is kept by the error rule. The remaining ~119 healthy traces are sampled at 1%, keeping ~1-2 traces. Total kept: ~2-3 out of 120, or approximately 2% of total trace volume. This confirms the policy is cost-effective: error traces are never lost, while the sampling buffer stays well within the 10K-span memory limit.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
    "prompt_length": {
        "psi": 3.461,
        "kl": 1.7982,
        "ks_stat": 0.702,
        "ks_pvalue": 0.0,
        "drift": "yes"
    },
    "embedding_norm": {
        "psi": 0.0187,
        "kl": 0.0324,
        "ks_stat": 0.052,
        "ks_pvalue": 0.133853,
        "drift": "no"
    },
    "response_length": {
        "psi": 0.0162,
        "kl": 0.0178,
        "ks_stat": 0.056,
        "ks_pvalue": 0.086899,
        "drift": "no"
    },
    "response_quality": {
        "psi": 8.8486,
        "kl": 13.5011,
        "ks_stat": 0.941,
        "ks_pvalue": 0.0,
        "drift": "yes"
    }
}
```

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

- **prompt_length**: **PSI** — This is a continuous feature that directly impacts operational behavior (token cost, latency). PSI is interpretable by business stakeholders (PSI > 0.2 = "drift"), has well-known thresholds from credit risk modeling, and is symmetric between reference and current distributions. It's also the industry standard for ML monitoring.
- **embedding_norm**: **KS** — For features expected to be stable (no drift in this case), the Kolmogorov-Smirnov test provides a p-value that lets us set a statistical significance threshold. Since embedding_norm should remain stationary (same model, same normalization), KS gives us a false-positive rate we can control rather than a magnitude we must interpret.
- **response_length**: **PSI** — Similar to prompt_length, this is a continuous operational metric. However, since response_length distributions tend to be heavy-tailed (log-normal in our simulator), PSI with fixed bins can be sensitive to tail outliers. In production I'd complement PSI with a quantile-based drift check on P50/P95/P99.
- **response_quality**: **PSI** — This is a bounded [0,1] metric where distribution shape matters more than location. PSI captures the full distribution shift well here. The beta(8,2) → beta(2,6) shift is a fundamental quality degradation that PSI correctly flags at 8.85. KL divergence (13.50) also confirms this, but PSI's established threshold of 0.2 makes it more actionable for alerting.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Day 18 (Spark Application Active) was the hardest to expose because the Spark metrics endpoint (`spark_application_active`) requires the Spark UI to be running and exposes metrics in a non-Prometheus-native format (JMX/Graphite sink). Without the actual Spark cluster running, the stub path is a simple gauge, but in production, wiring Spark Structured Streaming metrics through the Prometheus JMX exporter involves configuring `spark.metrics.conf` with the `PrometheusServlet` sink, which adds deployment complexity beyond the other services. Day 19 (Qdrant) and Day 20 (llama.cpp) were straightforward since both expose `/metrics` natively or via a lightweight stub.

---

## 6. The single change that mattered most

The single change that mattered most was adding the `inference_quality_score` gauge — the "4th pillar" metric beyond RED/USE — and wiring it into the SLO burn-rate alert. Without it, the stack tells you that requests are fast and numerous (RED: Rate, Errors, Duration). With it, the stack tells you that the model is _degrading in ways that latency cannot detect_.

This connects directly to deck §5 (SLO + Burn-Rate) and §7 (Drift). A model can return 200 OK at P50=50ms while its output quality drops from 0.82 to 0.45 — that's a silent failure no HTTP-level metric catches. By making `inference_quality_score` a first-class metric family with its own label (`model`), the SLO dashboard immediately surfaces when model quality crosses the 0.7 threshold, and the multi-window burn-rate alert fires if quality stays below threshold for more than 2 minutes in a 1-hour window. This transforms the observability stack from "is the service up?" to "is the AI any good?" — which is the entire point of LLM-native observability covered in the deck.
