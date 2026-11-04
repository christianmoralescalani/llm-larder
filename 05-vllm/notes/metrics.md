# Metrics & monitoring

**Summary.** vLLM is Prometheus-first: the server exposes a `/metrics` endpoint with `vllm:`-prefixed metrics, split into **server-level** (gauges/counters — engine state) and **request-level** (histograms — the SLOs an SRE tracks). The mental model: server-level metrics *explain* request-level values (high `kv_cache_usage_perc` → rising queue time → rising TTFT). There's also periodic stats logging to the info log for ad-hoc use, optional OpenTelemetry tracing, and an official Grafana reference dashboard.

## The endpoint

```bash
# scrape it
curl -s http://localhost:8000/metrics | grep '^vllm:' | head

# Prometheus scrape config
scrape_configs:
  - job_name: vllm
    static_configs:
      - targets: ["vllm:8000"]
```

- **`/metrics` is NOT behind `--api-key`** — the key only guards `/v1`, `/v2`, `/inference`. Don't expose the port publicly without a reverse proxy / network policy.
- Related flags: `--disable-log-stats` (stop the periodic info-log stats), `--enable-per-request-metrics` (server-side per-request attribution), `--kv-cache-metrics` (KV residency metrics), `--enable-mfu-metrics` (FLOPs utilization), `--otlp-traces-endpoint` (tracing).
- Per-request detail: start the server with `--enable-per-request-metrics` to get server-side queue/prefill/decode attribution (pairs with `vllm bench serve --save-detailed`).

## Server-level (gauges & counters)

| Metric | Type | Watch for |
|---|---|---|
| `vllm:num_requests_running` / `_waiting` / `_swapped` | Gauge | `_waiting` climbing = overloaded; `_swapped` climbing = KV pressure |
| `vllm:kv_cache_usage_perc` | Gauge | sustained >0.9 → preemptions/evictions → latency spikes |
| `vllm:prefix_cache_queries` / `vllm:prefix_cache_hits` | Counter | **hit rate = hits/queries**; low on a "prefix-heavy" workload = your prompts aren't sharing prefixes (or caching is off) |
| `vllm:prompt_tokens_total` / `vllm:generation_tokens_total` | Counter | throughput trend; rate() for tok/s |
| `vllm:request_success_total` | Counter | by finish reason (`stop`, `length`, …); `length` rising = users hitting max tokens |
| `vllm:admission_rejections_total` | Counter | requests rejected at the queue limit — you're past capacity |
| `vllm:cache_config_info` / `vllm:lora_requests_info` | Info | config/LoRA registry labels |

## Request-level (histograms — the SLOs)

| Metric | Meaning |
|---|---|
| `vllm:time_to_first_token_seconds` | **TTFT** — send → first token (prefill + queue) |
| `vllm:request_time_per_output_token_seconds` | **TPOT** per request — (E2EL − TTFT)/(tokens−1) |
| `vllm:inter_token_latency_seconds` | **ITL** — one sample per streamed output event (the gap between tokens) |
| `vllm:e2e_request_latency_seconds` | end-to-end, send → final token |
| `vllm:request_queue_time_seconds` | time in queue before scheduling |
| `vllm:request_prefill_time_seconds` | prefill phase |
| `vllm:request_decode_time_seconds` | decode phase |
| `vllm:request_prompt_tokens` / `vllm:request_generation_tokens` | per-request sizes (histograms) |
| `vllm:prompt_tokens` / `vllm:generation_tokens` | token-count histograms |
| `vllm:request_success` | finished requests by finish reason |

TTFT/TPOT/ITL/E2EL here match the `vllm bench serve` definitions (see `notes/benchmarking.md`) — so benchmark percentiles and production percentiles are directly comparable.

## Specialized

| Metric | What it tells you |
|---|---|
| `vllm:spec_decode_draft_acceptance_rate` | speculative decoding: fraction of draft tokens accepted (<0.5 = speculation is costing you) |
| `vllm:spec_decode_efficiency` | emitted tokens per draft token |
| `vllm:spec_decode_num_draft_tokens` / `_accepted_tokens` / `_emitted_tokens` | raw counters |
| `vllm:kv_block_lifetime_seconds` | how long KV blocks live |
| `vllm:kv_block_reuse_gap_seconds` | gap between a block's uses (prefix-cache reuse pattern) |
| `vllm:kv_block_idle_before_evict_seconds` | idle time before eviction (cache pressure) |

## Dashboards & alerting

- **Grafana reference dashboard** — vLLM ships a Prometheus+Grafana example (vLLM repo, `examples/dashboard/`); the dashboard's metric subset is the official "these matter" list.
- Suggested alerts (starting points):
  - `vllm:kv_cache_usage_perc > 0.95` sustained 5 min → KV pressure
  - `histogram_quantile(0.95, rate(vllm:time_to_first_token_seconds_bucket[5m])) > 2` → TTFT SLO breach
  - `rate(vllm:admission_rejections_total[5m]) > 0` → requests being dropped
  - `rate(vllm:prefix_cache_hits[5m]) / rate(vllm:prefix_cache_queries[5m]) < 0.3` on a prefix-heavy service → caching not working as expected
  - `vllm:num_requests_waiting > N` → queue building
- **Logging mode** — without Prometheus, the server logs a stats line (request counts, token throughput, KV usage, prefix-cache hit rate) to the info log every few seconds (default 5 s); `docker logs` is a poor man's monitor.
- **Tracing** — `--otlp-traces-endpoint http://collector:4317` sends OpenTelemetry traces (`--collect-detailed-traces all|model|worker` for detail).

## What to watch, in order

1. **TTFT p95** — user-perceived "is it stuck?" (queue + prefill).
2. **TPOT / ITL p95** — streaming smoothness (decode).
3. **`kv_cache_usage_perc`** — the root cause behind both when it's bad.
4. **`num_requests_waiting`** — capacity headroom.
5. **prefix-cache hit rate** — whether your optimization is actually firing.
6. **finish reasons** — `length` vs `stop` tells you if users are being cut off.

## Gotchas

1. **`/metrics` unauthenticated** — network-level protection is on you.
2. **ITL ≠ TPOT** — ITL is per streamed event (one sample can bundle multiple tokens, e.g. with speculative decoding); TPOT is per finished request. They diverge under speculation.
3. **Histograms need rate()** — `vllm:time_to_first_token_seconds_bucket` is cumulative; always `rate(...[5m])` + `histogram_quantile`.
4. **Labels** — request-level histograms carry `model_name`, `lora_name`, etc.; aggregate across replicas with `sum by (model_name)`.
5. **Multi-process mode** — with `--api-server-count > 1` or DP replicas, each process publishes; dedupe/label by instance before alerting.
6. **Stats log ≠ Prometheus** — the info-log stats are coarse (intervals, no percentiles); fine for a glance, not for SLOs.

## Sources

- [Metrics design doc](https://docs.vllm.ai/en/stable/design/metrics/)
- [Metrics setup](https://docs.vllm.ai/en/stable/examples/observability/metrics/)
- [Prometheus & Grafana example](https://docs.vllm.ai/en/stable/examples/observability/prometheus_grafana/)
- [Per-request metrics](https://docs.vllm.ai/en/stable/features/per_request_metrics/)

## My experiments

- (fill in: e.g. "Grafana dashboard on 7B server: p95 TTFT 120ms @ 8 concurrent, 900ms @ 32; kv_cache_usage 0.97 sustained at 32 → preemption spikes visible in ITL; prefix hit rate 0.74 with shared system prompt vs 0.02 with unique prompts")
