# Recording Rules 가이드

## 개요

Recording Rules는 자주 사용되거나 계산 비용이 높은 PromQL 표현식을 미리 계산해 새로운 메트릭으로 저장합니다.

**효과**:
- 대시보드 로딩 속도 향상
- 복잡한 집계 쿼리 반복 실행 방지
- 장기 보존용 집계 메트릭 생성

---

## 네이밍 컨벤션

```
level:metric:operations
```

- **level**: 집계 수준 (`job`, `instance`, `cluster`, `namespace` 등)
- **metric**: 원본 메트릭 이름
- **operations**: 적용된 함수 (`rate`, `sum`, `avg` 등)

예시:
```
job:http_requests_total:rate5m
namespace:container_memory_usage_bytes:sum
cluster:node_cpu_seconds_total:avg_rate5m
```

---

## PrometheusRule로 Recording Rules 정의

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: recording-rules
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: http.recording_rules
      interval: 30s
      rules:
        # 서비스별 초당 요청 수
        - record: job:http_requests_total:rate5m
          expr: |
            sum by (job) (
              rate(http_requests_total[5m])
            )

        # 서비스별 에러율
        - record: job:http_requests_errors:rate5m
          expr: |
            sum by (job) (
              rate(http_requests_total{status=~"5.."}[5m])
            )
            / sum by (job) (
              rate(http_requests_total[5m])
            )

        # 서비스별 P99 응답 시간
        - record: job:http_request_duration_seconds:p99_rate5m
          expr: |
            histogram_quantile(0.99,
              sum by (job, le) (
                rate(http_request_duration_seconds_bucket[5m])
              )
            )
```

---

## 실용적인 Recording Rules 모음

### CPU 관련

```yaml
  - name: node.cpu.recording_rules
    rules:
      # 인스턴스별 CPU 사용률
      - record: instance:node_cpu_utilisation:rate5m
        expr: |
          1 - avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      # 클러스터 전체 CPU 사용률
      - record: cluster:node_cpu_utilisation:avg_rate5m
        expr: |
          avg(instance:node_cpu_utilisation:rate5m)
```

### 메모리 관련

```yaml
  - name: node.memory.recording_rules
    rules:
      # 인스턴스별 메모리 사용률
      - record: instance:node_memory_utilisation:ratio
        expr: |
          1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

      # 클러스터 전체 메모리 사용률
      - record: cluster:node_memory_utilisation:avg_ratio
        expr: |
          avg(instance:node_memory_utilisation:ratio)
```

### Kubernetes 관련

```yaml
  - name: kubernetes.recording_rules
    rules:
      # 네임스페이스별 파드 CPU 사용량
      - record: namespace:container_cpu_usage_seconds:sum_rate5m
        expr: |
          sum by (namespace) (
            rate(container_cpu_usage_seconds_total{container!=""}[5m])
          )

      # 네임스페이스별 파드 메모리 사용량
      - record: namespace:container_memory_usage_bytes:sum
        expr: |
          sum by (namespace) (
            container_memory_usage_bytes{container!=""}
          )

      # Deployment 가용성 (available / desired)
      - record: deployment:kube_deployment_status_replicas:availability
        expr: |
          kube_deployment_status_replicas_available
          / kube_deployment_spec_replicas
```

### 장기 보존용 (다운샘플링)

```yaml
  - name: aggregation.hourly
    interval: 5m
    rules:
      # 1시간 단위 요청 수 (장기 분석용)
      - record: job:http_requests_total:increase1h
        expr: |
          sum by (job, status) (
            increase(http_requests_total[1h])
          )
```

---

## Recording Rules 활용

### 알림 규칙에서 활용

Recording Rules를 알림 규칙에서 재사용:

```yaml
  - name: alerts.using_recording_rules
    rules:
      - alert: HighErrorRate
        # 미리 계산된 메트릭 사용 (빠름!)
        expr: job:http_requests_errors:rate5m > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.job }}"

      - alert: HighCPU
        expr: instance:node_cpu_utilisation:rate5m > 0.80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
```

### Grafana 대시보드에서 활용

```promql
# 느린 원본 쿼리 대신
histogram_quantile(0.99, sum by (job, le) (rate(http_request_duration_seconds_bucket[5m])))

# Recording Rule 사용
job:http_request_duration_seconds:p99_rate5m
```

---

## 확인

```bash
# Recording Rules 상태 확인 (Prometheus UI)
# http://localhost:9090/rules

# PrometheusRule 목록
kubectl get prometheusrule -n monitoring

# 생성된 메트릭 확인
# http://localhost:9090/graph
# query: job:http_requests_total:rate5m
```

---

## 주의사항

1. **Recording Rules는 새로운 메트릭**: 저장 공간 사용
2. **집계로 인한 정보 손실**: 레이블이 제거되면 세부 쿼리 불가
3. **평가 주기**: 너무 짧으면 부하 증가, 기본 `evaluation_interval` 활용 권장
4. **Cardinality**: Recording Rule의 결과 시계열 수를 최소화할 것
