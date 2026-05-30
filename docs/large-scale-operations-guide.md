# 대규모 Prometheus 운영 가이드

이 문서는 단일 클러스터 실습을 넘어, 여러 팀/서비스/클러스터가 Prometheus를 함께 사용하는 환경에서 필요한 운영 기준을 정리합니다.

목표는 "Prometheus를 많이 띄우는 방법"이 아니라 **수집량, 카디널리티, 쿼리 비용, 알림 품질, 장애 복구를 통제 가능한 상태로 유지하는 것**입니다.

---

## 운영 원칙

### 1. Prometheus는 실시간 로컬 수집 계층으로 본다

Prometheus 로컬 TSDB는 빠른 실시간 쿼리와 알림 평가에 강합니다. 하지만 장기 보존, 멀티 클러스터 통합 조회, 대규모 사용자 쿼리까지 한 인스턴스에 맡기면 병목이 빨리 옵니다.

**권장 역할 분리**:

| 계층 | 역할 | 예시 |
|------|------|------|
| Edge Prometheus | 클러스터/샤드별 scrape, 짧은 보존, 알림 평가 | kube-prometheus-stack |
| Long-term Storage | 장기 보존, 글로벌 조회, 다운샘플링 | Mimir, Thanos, Cortex |
| Query Frontend | 대시보드/분석 쿼리 캐싱 및 제한 | Mimir query-frontend, Thanos Query Frontend |
| Grafana | 사용자 대시보드 | recording rule 우선 사용 |

### 2. 확장은 "수집 대상 분리"가 먼저다

처음부터 거대한 중앙 Prometheus 하나를 만들지 않습니다. 대규모 환경에서는 클러스터, 리전, 테넌트, 워크로드 성격별로 수집 책임을 나눕니다.

```
cluster-a Prometheus ── remote_write ─┐
cluster-b Prometheus ── remote_write ─┼──► Mimir/Thanos ──► Grafana
cluster-c Prometheus ── remote_write ─┘
```

**분리 기준**:
- 클러스터별: EKS 클러스터마다 Prometheus 배포
- 환경별: production/staging/dev 분리
- 테넌트별: 팀 또는 서비스 그룹별 external label 부여
- 수집 특성별: 인프라 메트릭, 애플리케이션 메트릭, blackbox probe 분리

---

## 용량 계획

### 핵심 지표

대규모 운영에서 먼저 봐야 할 지표는 CPU보다 **활성 시계열 수와 샘플 입력률**입니다.

```promql
# 현재 활성 시계열 수
prometheus_tsdb_head_series

# 초당 수집 샘플 수
rate(prometheus_tsdb_head_samples_appended_total[5m])

# scrape 후 실제 저장되는 샘플 수
sum(rate(scrape_samples_post_metric_relabeling[5m]))

# 메모리 사용량
process_resident_memory_bytes{job="prometheus"}

# 쿼리 p99 지연
histogram_quantile(0.99, rate(prometheus_engine_query_duration_seconds_bucket[5m]))
```

### 대략적인 규모 판단

| 규모 | 활성 시계열 | 권장 운영 방식 |
|------|------------|----------------|
| 소규모 | 10만 이하 | 단일 Prometheus + PVC |
| 중간 규모 | 10만~100만 | HA 2대 + remote_write + recording rule |
| 대규모 | 100만 이상 | 샤딩, 타겟 분리, 장기 저장소 필수 |
| 초대규모 | 수백만 이상 | 팀/클러스터 단위 Prometheus + 중앙 Mimir/Thanos |

정확한 한계는 하드웨어, scrape interval, 레이블 수, 쿼리 패턴에 따라 달라집니다. 숫자는 절대 기준이 아니라 설계 시작점으로 봅니다.

### scrape interval 정책

모든 타겟을 15초로 긁으면 수집량이 빠르게 증가합니다. 서비스 중요도별로 기본값을 나눕니다.

| 대상 | 권장 scrape interval | 이유 |
|------|----------------------|------|
| 핵심 애플리케이션 | 15s~30s | 빠른 장애 탐지 |
| Kubernetes control plane | 30s | 상태 변화 추적 |
| node-exporter | 30s~60s | 노드 지표는 초단위 정밀도가 덜 중요 |
| kube-state-metrics | 30s~60s | API 오브젝트 상태 중심 |
| blackbox probe | 30s~60s | 외부 의존성 확인 |
| 비용성/분석성 메트릭 | 60s 이상 | 장기 추세 목적 |

---

## 카디널리티 관리

### 카디널리티 폭발이 가장 흔한 장애 원인

Prometheus에서 시계열 수는 다음 조합으로 증가합니다.

```
metric_name x label_value 조합 x target 수
```

다음 레이블은 운영 환경에서 금지하거나 별도 승인을 받습니다.

| 금지/주의 레이블 | 이유 |
|-----------------|------|
| `user_id` | 사용자 수만큼 시계열 증가 |
| `request_id`, `trace_id` | 요청마다 새 시계열 생성 |
| `session_id` | 수명이 짧고 값이 많음 |
| `path` 원문 | `/users/1`, `/users/2`처럼 무한 증가 |
| `ip`, `email` | 값 범위가 크고 개인정보 위험 |
| `pod_uid` | 재시작마다 값 변경 |

**좋은 예**:

```text
http_requests_total{method="GET", route="/users/:id", status="200"}
```

**나쁜 예**:

```text
http_requests_total{method="GET", path="/users/123456", user_id="123456"}
```

### 카디널리티 점검 쿼리

```promql
# 메트릭별 시계열 수 상위 20개
topk(20, count by (__name__) ({__name__=~".+"}))

# job별 시계열 수
topk(20, count by (job) ({__name__=~".+"}))

# namespace별 시계열 수
topk(20, count by (namespace) ({__name__=~".+"}))

# 메트릭 + 레이블 조합이 과도한 후보 찾기
topk(20, count by (__name__, job) ({__name__=~".+"}))
```

### metricRelabelings로 수집 전 제거

저장 후 삭제보다 **수집 전에 drop**하는 것이 안전합니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  endpoints:
    - port: metrics
      interval: 30s
      metricRelabelings:
        # 요청 단위 ID가 붙은 메트릭 제거
        - sourceLabels: [request_id]
          regex: ".+"
          action: drop

        # 불필요한 런타임 메트릭 제거
        - sourceLabels: [__name__]
          regex: "go_memstats_.*"
          action: drop
```

---

## 샤딩과 HA

### HA와 샤딩은 목적이 다르다

| 방식 | 목적 | 결과 |
|------|------|------|
| `replicas: 2` | 고가용성 | 같은 타겟을 2번 수집 |
| `shards: N` | 수집 부하 분산 | 타겟을 N개 Prometheus로 나눠 수집 |
| `replicas: 2`, `shards: 3` | HA + 확장 | 총 6개 Prometheus Pod 생성 |

Prometheus Operator에서는 `replicas * shards` 만큼 Pod가 생성됩니다.

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    shards: 3
    replicaExternalLabelName: "__replica__"
    externalLabels:
      cluster: "eks-production"
      region: "ap-northeast-2"
```

### 샤딩을 고려할 때

다음 조건 중 여러 개가 동시에 보이면 샤딩을 검토합니다.

- `prometheus_tsdb_head_series`가 계속 증가한다
- scrape duration이 scrape interval에 가까워진다
- rule evaluation 시간이 길어져 알림 평가가 지연된다
- remote_write pending samples가 계속 쌓인다
- Grafana 대시보드가 자주 timeout 난다
- Prometheus 재시작 후 WAL replay가 오래 걸린다

### 샤딩 전 먼저 할 일

1. 불필요한 메트릭 drop
2. scrape interval 재조정
3. 고비용 대시보드 쿼리를 recording rule로 전환
4. 네임스페이스/팀별 수집 범위 분리
5. Prometheus 리소스 request/limit과 PVC IOPS 확인

---

## Remote Write 운영

### remote_write는 백업이 아니라 전송 파이프라인이다

remote_write 대상이 느려지면 Prometheus 내부 큐가 밀립니다. 큐가 계속 쌓이면 메모리 사용량이 증가하고 WAL 읽기가 지연됩니다.

```promql
# 전송 대기 샘플
prometheus_remote_storage_samples_pending

# 실패 샘플
rate(prometheus_remote_storage_failed_samples_total[5m])

# 재시도 샘플
rate(prometheus_remote_storage_retried_samples_total[5m])

# remote write 샤드 수
prometheus_remote_storage_shards

# 가장 오래된 pending 샘플의 지연 시간
time() - prometheus_remote_storage_queue_lowest_sent_timestamp_seconds
```

### 큐 튜닝 기준

```yaml
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: http://mimir-nginx.mimir.svc/api/v1/push
        headers:
          X-Scope-OrgID: "production"
        queueConfig:
          capacity: 10000
          minShards: 4
          maxShards: 50
          maxSamplesPerSend: 5000
          batchSendDeadline: 5s
          minBackoff: 30ms
          maxBackoff: 5s
```

튜닝 순서:

1. receiver(Mimir/Thanos Receive 등)의 ingest 한계와 에러율 확인
2. Prometheus의 CPU, 메모리, 네트워크 사용량 확인
3. `samples_pending`이 지속 증가하면 `maxShards`와 `maxSamplesPerSend` 조정
4. 큐를 키울 때는 메모리도 같이 증설
5. 모든 메트릭을 보내지 말고 `writeRelabelConfigs`로 장기 보존이 필요한 것만 선별

```yaml
writeRelabelConfigs:
  # 장기 분석에 불필요한 메트릭 제외
  - sourceLabels: [__name__]
    regex: "go_.*|process_.*"
    action: drop

  # 개발 네임스페이스 제외
  - sourceLabels: [namespace]
    regex: "dev-.*"
    action: drop
```

---

## Query와 Recording Rule 운영

### 사용자 쿼리는 플랫폼 안정성에 영향을 준다

Grafana 대시보드 하나가 너무 넓은 범위의 raw metric을 조회하면 Prometheus CPU와 메모리를 크게 사용합니다.

**대시보드 원칙**:
- 기본 범위는 1h~6h로 제한
- 장기 조회는 Mimir/Thanos datasource 사용
- 반복 패널에서 고카디널리티 label 사용 금지
- `rate()`는 너무 짧은 range vector로 사용하지 않기
- 상위 화면은 recording rule 기반 집계 메트릭 사용

### 비용 높은 쿼리 예시

```promql
# 나쁨: 모든 pod/container 원본 시계열을 긴 범위로 조회
sum by (pod, container) (
  rate(container_cpu_usage_seconds_total[5m])
)
```

```promql
# 좋음: namespace 단위로 사전 집계한 recording rule 사용
namespace:container_cpu_usage_seconds:sum_rate5m
```

### Rule group 운영 기준

| 항목 | 권장 |
|------|------|
| rule group 크기 | 서비스/도메인별로 분리 |
| evaluation interval | 기본 30s~1m, 비용 큰 집계는 2m~5m |
| 알림 `for` | 최소 5m 이상으로 flapping 완화 |
| recording rule | 대시보드와 알림이 공유하는 집계부터 생성 |
| 변경 검증 | `promtool check rules` 필수 |

```bash
promtool check rules ../ops/config/manifests/prometheusrule.yaml
```

---

## 알림 운영

### 대규모 환경의 목표는 "더 많은 알림"이 아니다

운영자가 실제로 행동할 수 있는 알림만 남깁니다. 단순 증상 알림이 많으면 장애 시 AlertManager가 노이즈를 증폭합니다.

**알림 설계 기준**:
- 사용자 영향이 있는가?
- 담당 팀이 명확한가?
- runbook이 있는가?
- 자동 복구가 실패했을 때만 울리는가?
- `severity`, `team`, `service`, `cluster`, `namespace` 레이블이 있는가?

### AlertManager 라우팅 예시

```yaml
route:
  group_by: ["alertname", "cluster", "service"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: platform-default
  routes:
    - matchers:
        - severity="critical"
      receiver: pagerduty
      continue: true
    - matchers:
        - team="payments"
      receiver: payments-slack
```

### 억제 규칙 예시

노드 장애가 발생했을 때 그 노드 위의 PodDown 알림을 모두 보내면 노이즈가 커집니다.

```yaml
inhibit_rules:
  - source_matchers:
      - alertname="NodeDown"
    target_matchers:
      - alertname="PodDown"
    equal: ["cluster", "node"]
```

---

## 멀티 클러스터와 테넌시

### external labels는 필수다

멀티 클러스터에서는 모든 Prometheus에 식별 가능한 external label을 붙입니다.

```yaml
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: "eks-prod-a"
      region: "ap-northeast-2"
      environment: "production"
```

없으면 중앙 저장소에서 같은 서비스 이름, 같은 namespace가 충돌합니다.

### 테넌트 분리

Mimir 같은 멀티 테넌트 저장소를 사용할 때는 팀/환경 단위로 tenant를 분리합니다.

```yaml
remoteWrite:
  - url: http://mimir-nginx.mimir.svc/api/v1/push
    headers:
      X-Scope-OrgID: "platform-production"
```

운영 기준:
- production과 non-production은 tenant 분리
- 팀별 비용/쿼리량을 추적하려면 tenant 또는 label로 분리
- 공용 Grafana datasource는 조회 범위와 권한을 제한
- 중앙 저장소에도 retention 정책을 tenant별로 다르게 적용

---

## Prometheus 자체 SLO

모니터링 시스템도 서비스로 보고 SLO를 둡니다.

| SLI | 예시 목표 | 측정 |
|-----|----------|------|
| scrape 성공률 | 99% 이상 | `avg(up)` |
| rule 평가 성공률 | 99.9% 이상 | rule failure rate |
| remote_write 지연 | 5분 이하 | pending timestamp lag |
| 쿼리 지연 | p99 30초 이하 | query duration histogram |
| 데이터 신선도 | 2분 이하 | latest sample lag |

### 권장 알림

```yaml
groups:
  - name: prometheus-platform.rules
    rules:
      - alert: PrometheusRemoteWriteLagging
        expr: |
          time() - prometheus_remote_storage_queue_lowest_sent_timestamp_seconds > 300
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Prometheus remote_write is lagging"
          description: "Remote write queue is more than 5 minutes behind."

      - alert: PrometheusHighCardinalityGrowth
        expr: |
          increase(prometheus_tsdb_head_series[1h]) > 100000
        for: 30m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Prometheus active series increased rapidly"
          description: "Investigate recent deployments and high-cardinality labels."

      - alert: PrometheusRuleEvaluationSlow
        expr: |
          prometheus_rule_group_last_duration_seconds
          > prometheus_rule_group_interval_seconds * 0.8
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Prometheus rule group evaluation is slow"
```

---

## 변경 관리

### 배포 전 체크리스트

- [ ] 새 메트릭에 고카디널리티 레이블이 없는가?
- [ ] `ServiceMonitor` selector가 너무 넓지 않은가?
- [ ] scrape interval이 서비스 중요도에 맞는가?
- [ ] 새 recording/alerting rule을 `promtool`로 검증했는가?
- [ ] Grafana 대시보드가 raw metric 장기 조회를 하지 않는가?
- [ ] remote_write 대상 tenant와 external label이 올바른가?
- [ ] 알림에 `team`, `service`, `severity`, `runbook_url`이 있는가?

### 업그레이드 절차

1. Prometheus Operator, kube-prometheus-stack 차이점 확인
2. staging 클러스터에 먼저 배포
3. `/-/ready`, `/targets`, `/rules`, `/status` 확인
4. config reload 실패 알림 확인
5. remote_write pending samples 확인
6. Grafana 핵심 대시보드 로딩 확인
7. production은 한 클러스터 또는 한 샤드씩 점진 배포

---

## 장애 대응 Runbook

### 1. Prometheus 메모리 급증

확인:

```promql
process_resident_memory_bytes{job="prometheus"}
prometheus_tsdb_head_series
topk(20, count by (__name__) ({__name__=~".+"}))
```

조치:

1. 최근 배포된 서비스의 메트릭 변경 확인
2. 고카디널리티 메트릭을 `metricRelabelings`로 drop
3. scrape interval 증가
4. 리소스 limit에 걸린 경우 메모리 증설
5. 재시작은 마지막 수단으로 사용

### 2. remote_write 지연

확인:

```promql
prometheus_remote_storage_samples_pending
rate(prometheus_remote_storage_failed_samples_total[5m])
time() - prometheus_remote_storage_queue_lowest_sent_timestamp_seconds
```

조치:

1. remote storage receiver 상태 확인
2. 네트워크 에러 또는 429/5xx 로그 확인
3. 불필요한 메트릭 write drop
4. `maxShards`, `maxSamplesPerSend`, `capacity` 조정
5. Prometheus 메모리 여유 확인

### 3. 대시보드 timeout

확인:

```promql
histogram_quantile(0.99, rate(prometheus_engine_query_duration_seconds_bucket[5m]))
prometheus_engine_queries
prometheus_engine_queries_concurrent_max
```

조치:

1. Grafana 패널 쿼리 범위 축소
2. 반복 패널 label 확인
3. recording rule 추가
4. 장기 조회는 Mimir/Thanos로 이동
5. query timeout과 max samples 제한 검토

---

## 참고 링크

- Prometheus Remote write tuning: https://prometheus.io/docs/practices/remote_write/
- Prometheus Storage operational aspects: https://prometheus.io/docs/prometheus/latest/storage/
- Prometheus Federation: https://prometheus.io/docs/prometheus/latest/federation/
- Prometheus Metric and label naming: https://prometheus.io/docs/practices/naming/
- Prometheus Instrumentation best practices: https://prometheus.io/docs/practices/instrumentation/
- Prometheus Operator API reference: https://prometheus-operator.dev/docs/api-reference/api/
