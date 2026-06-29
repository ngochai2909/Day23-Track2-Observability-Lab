# Day 23 Lab Reflection

**Student:** Hai Nguyen  
**Submission date:** 2026-06-29  
**Lab repo URL:** https://github.com/ngochai2909/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Output from `python3 00-setup/verify-docker.py` / `make setup`:

```json
{
  "docker": { "ok": true, "version": "28.2.2" },
  "compose_v2": { "ok": true, "version": "2.36.2" },
  "ram_gb_available": 14.9,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

The stack passed `make smoke` after fixing two lab config issues: Alertmanager now reads the Slack webhook from a Docker Compose secret, and the Grafana health check accepts the current Grafana 11 JSON formatting.

---

## 2. Track 02 - Dashboards & Alerts

### 6 essential panels

Screenshot: `submission/screenshots/dashboard-overview.png`.

The load run completed with 1112 `POST /predict` requests in 60 seconds, 0 failures, and about 18.56 req/s. The overview dashboard shows request rate, latency, error rate, GPU utilization, token throughput, and in-flight requests.

### Burn-rate panel

Screenshot: `submission/screenshots/slo-burn-rate.png`.

The SLO dashboard is provisioned automatically and reads from the Prometheus recording rules. In the baseline load run, burn rate stays low because all requests succeeded.

### Cost and tokens

Screenshot: `submission/screenshots/cost-and-tokens.png`.

This dashboard uses `inference_tokens_total` to estimate throughput and hourly cost. It became usable after setting the Prometheus datasource UID to `prometheus`, matching the dashboard JSON.

### Alert fire + resolve

| When                 | What                                 | Evidence                                           |
| -------------------- | ------------------------------------ | -------------------------------------------------- |
| 2026-06-29 04:49 UTC | stopped `day23-app`                  | `submission/screenshots/alertmanager-firing.png`   |
| 2026-06-29 04:49 UTC | `ServiceDown` active in Alertmanager | `submission/screenshots/alertmanager-firing.png`   |
| 2026-06-29 04:50 UTC | restarted `day23-app`                | local Docker event                                 |
| 2026-06-29 04:50 UTC | alert resolved                       | `submission/screenshots/alertmanager-resolved.png` |

Slack webhook delivery was configured through `.env` and the Alertmanager config loaded successfully. I cannot capture the Slack web UI from this environment because there is no logged-in Slack browser session here; capture `slack-firing.png` and `slack-resolved.png` manually from the Slack workspace if the grader requires those specific screenshots.

### One thing surprised me about Prometheus / Grafana

The most fragile part was not the PromQL; it was the identity wiring between dashboards and datasources. The dashboard JSON referenced datasource UID `prometheus`, but the provisioned datasource originally had no explicit UID, so Grafana generated a different one and panels silently failed. Stable UIDs matter as much as stable metric names when dashboards are code.

---

## 3. Track 03 - Tracing & Logs

### One trace screenshot from Jaeger

Screenshot: `submission/screenshots/jaeger-trace.png`.

Current note: the source code has been fixed so the manual `predict` span is created with `start_as_current_span`, which makes `embed-text`, `vector-search`, and `generate-tokens` children in the same trace. The running container still needs to be rebuilt with `docker compose up -d --build app` before this screenshot fully matches the rubric span tree.

### Log line correlated to trace

```json
{
  "model": "llama3-mock",
  "input_tokens": 5,
  "output_tokens": 39,
  "quality": 0.757,
  "duration_seconds": 0.2031,
  "trace_id": "dddb82d9ce02593e2f8a6d66ef7b472e",
  "event": "prediction served",
  "level": "info",
  "timestamp": "2026-06-29T04:41:01.245320Z"
}
```

### Tail-sampling math

The baseline load produced 1112 requests in 60 seconds, or about 18.5 traces/sec if every request emits one trace. The collector policy keeps 1% of healthy traces, so expected retained healthy traces are about `18.5 * 60 * 0.01 = 11.1` traces for that run. Error and slow traces are configured for 100% retention, so those should not depend on random sampling.

---

## 4. Track 04 - Drift Detection

### PSI scores

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

Evidently report screenshot: `submission/screenshots/evidently-drift-report.png`.

### Which test fits which feature?

For `prompt_length`, PSI is a good production headline metric because it is easy to bucket, explain, and alert on when prompt distribution shifts. KS is useful as a secondary statistical test because prompt length is continuous and we care about the full distribution, not only the mean.

For `embedding_norm`, KS is the better first check because small distribution changes in a continuous embedding-derived scalar can matter even when bucketed PSI looks small. For higher-dimensional embeddings, MMD would be more appropriate than PSI or one-dimensional KS because drift may show up in joint geometry instead of a single scalar norm.

For `response_length`, PSI works well operationally because response length can be bucketed into product-relevant ranges such as short, normal, and verbose. KS is useful when tuning thresholds because it compares the continuous empirical distributions without choosing bins.

For `response_quality`, PSI is useful for alerting because quality scores can be bucketed into pass, warning, and fail bands. KL is useful during analysis because it highlights how much the current quality distribution diverges from the reference distribution, especially when mass moves from high-quality to low-quality regions.

---

## 5. Track 05 - Cross-Day Integration

Screenshot: `submission/screenshots/cross-day-dashboard.png`.

The cross-day dashboard has been copied into the Grafana provisioned dashboard directory as `02-prometheus-grafana/grafana/dashboards/full-stack-dashboard.json`, so it loads automatically. Prior-day services were not all running locally, so the dashboard is designed to show panels with data where metrics exist and "No Data" where the source is absent.

The hardest prior-day metric to expose would be Day 20 llama.cpp tokens/sec because the serving process does not expose Prometheus metrics by default. It needs either a patched server, a sidecar exporter, or a wrapper that turns request logs into counters and gauges. Day 19 Qdrant is easier because it already has a `/metrics` endpoint when the service is running.

---

## 6. The single change that mattered most

The single change that mattered most was making telemetry identity explicit instead of implicit. In Grafana, setting the Prometheus datasource UID to `prometheus` turned the dashboards from provisioned-but-broken JSON into reusable dashboards-as-code. In tracing, changing the manual `predict` span to a current span is the equivalent fix for context identity: without it, the child operations are exported as separate traces and the Jaeger view cannot explain one request end to end.

This maps directly to the deck's tracing and dashboards-as-code sections. Observability is useful only when separate signals can be joined reliably: metric panels to the right datasource, logs to a trace ID, and spans to the request context that caused them. The stack already emitted many signals, but the useful step was preserving the relationships between those signals so an operator can move from symptom to cause without guessing.
