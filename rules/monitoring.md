# 모니터링 규칙 — prometheus-practice

## Prometheus 자체 모니터링

### 핵심 메트릭

```promql
# 수집 타겟 중 다운된 수
count(up == 0)

# 스크레이핑 실패율
rate(scrape_samples_post_metric_relabeling[5m]) / rate(scrape_samples_scraped[5m])

# WAL 재생 시간 (재시작 시)
prometheus_tsdb_head_wal_replay_duration_seconds

# 활성 시계열 수
prometheus_tsdb_head_series

# 쿼리 지연 (p99)
histogram_quantile(0.99, rate(prometheus_engine_query_duration_seconds_bucket[5m]))
```

### 알림 규칙 (권장)

| 알림 | 조건 | 심각도 |
|------|------|--------|
| PrometheusTargetDown | `up == 0` for 5m | warning |
| PrometheusConfigReloadFailed | `prometheus_config_last_reload_successful == 0` | critical |
| PrometheusStorageFull | `predict_linear(prometheus_tsdb_storage_blocks_bytes[6h], 24*3600) > disk_total * 0.9` | warning |
| PrometheusRuleEvaluationFailed | `rate(prometheus_rule_evaluation_failures_total[5m]) > 0` | warning |

## SLO 정의 (Prometheus 서비스)

| SLI | 목표 | 측정 방법 |
|-----|------|---------|
| 가용성 | 99.9% | `up{job="prometheus"}` |
| 스크레이핑 성공률 | 95% 이상 | `up` 타겟 비율 |
| 쿼리 p99 지연 | < 30초 | `prometheus_engine_query_duration_seconds` |

## 대시보드 구성 (Grafana)
- **Overview**: 타겟 수, 업타임, 시계열 수
- **Scraping**: 타겟별 스크레이핑 성공/실패
- **Rules**: 알림 규칙 평가 시간
- **TSDB**: 스토리지 사용량, 청크 수
