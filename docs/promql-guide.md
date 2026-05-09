# PromQL 가이드

## 기본 개념

### 데이터 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| Instant Vector | 특정 시점의 시계열 집합 | `http_requests_total` |
| Range Vector | 시간 범위의 시계열 집합 | `http_requests_total[5m]` |
| Scalar | 단순 숫자 | `1.5` |
| String | 문자열 (거의 미사용) | `"hello"` |

---

## 셀렉터 (Selectors)

### 레이블 매칭

```promql
# 정확히 일치
http_requests_total{job="api-server"}

# 정확히 불일치
http_requests_total{status!="200"}

# 정규식 일치
http_requests_total{status=~"5.."}

# 정규식 불일치
http_requests_total{path!~"/health.*"}

# 여러 조건 조합
http_requests_total{job="api-server", status=~"2.."}
```

### 범위 쿼리

```promql
# 최근 5분간의 데이터
http_requests_total[5m]

# 최근 1시간간의 데이터
node_cpu_seconds_total[1h]

# 시간 단위: ms(밀리초), s(초), m(분), h(시간), d(일), w(주), y(년)
```

---

## 함수

### rate() — 초당 변화율

Counter에 사용. 범위 내 초당 평균 변화율 계산.

```promql
# 최근 5분간 초당 HTTP 요청 수
rate(http_requests_total[5m])

# 최근 5분간 초당 에러 수
rate(http_requests_total{status=~"5.."}[5m])
```

> **범위 선택 팁**: scrape_interval의 4배 이상 설정 권장 (15s scrape → [1m] 이상)

---

### irate() — 순간 변화율

마지막 두 샘플 기반. 급격한 변화에 민감.

```promql
# 순간 CPU 사용률 (급격한 스파이크 감지)
irate(node_cpu_seconds_total[5m])
```

---

### increase() — 범위 내 증가량

Counter의 범위 내 절대 증가량.

```promql
# 최근 1시간 동안 총 요청 수
increase(http_requests_total[1h])

# 최근 24시간 에러 수
increase(http_requests_total{status=~"5.."}[24h])
```

---

### histogram_quantile() — 분위수 계산

Histogram 타입에서 분위수 계산.

```promql
# 95th percentile 응답 시간
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# 서비스별 99th percentile
histogram_quantile(0.99,
  sum by (service, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

---

### 집계 함수

```promql
# 합계
sum(http_requests_total)

# 평균
avg(node_cpu_seconds_total)

# 최대/최소
max(node_memory_MemAvailable_bytes)
min(node_memory_MemAvailable_bytes)

# 개수
count(up{job="kubernetes-pods"})

# 표준편차
stddev(http_request_duration_seconds_sum)

# by로 레이블 유지
sum by (pod) (container_memory_usage_bytes)

# without으로 레이블 제거
sum without (instance) (http_requests_total)
```

---

### 시간 함수

```promql
# 값의 변화량 (Gauge용)
delta(node_memory_MemAvailable_bytes[1h])

# 파생 함수 (선형 보간)
deriv(node_memory_MemAvailable_bytes[1h])

# 범위 내 최대값
max_over_time(process_resident_memory_bytes[1h])

# 범위 내 평균
avg_over_time(node_cpu_seconds_total[30m])

# 범위 내 마지막 값
last_over_time(up[5m])
```

---

## 실전 쿼리 예제

### CPU 사용률

```promql
# 노드별 CPU 사용률 (%)
100 - (avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
) * 100)

# 파드별 CPU 사용률
sum by (pod, namespace) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
)
```

### 메모리 사용률

```promql
# 노드 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 컨테이너 메모리 사용량 (MiB)
container_memory_usage_bytes{container!=""} / 1024 / 1024

# 메모리 사용률 vs 요청량
container_memory_usage_bytes{container!=""}
/ container_spec_memory_limit_bytes{container!=""}
```

### HTTP 에러율

```promql
# 서비스별 에러율 (5xx)
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/ sum by (service) (rate(http_requests_total[5m]))

# 1% 이상 에러율인 서비스
(
  sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
  / sum by (service) (rate(http_requests_total[5m]))
) > 0.01
```

### 파드 상태

```promql
# 실행 중이지 않은 파드
kube_pod_status_phase{phase!="Running", phase!="Succeeded"}

# OOMKilled된 컨테이너
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

# 재시작 횟수가 5회 이상인 파드
kube_pod_container_status_restarts_total > 5

# Deployment 가용 레플리카 부족
kube_deployment_status_replicas_available
< kube_deployment_spec_replicas
```

### 디스크 사용률

```promql
# 노드 디스크 사용률 (%)
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes)
* 100

# 4시간 후 디스크 가득 찰 예상
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0
```

---

## 연산자

### 산술 연산자
```promql
# bytes → MiB 변환
node_memory_MemAvailable_bytes / 1024 / 1024

# 사용률 계산
(total - available) / total * 100
```

### 비교 연산자
```promql
# 임계값 초과 시계열만 반환
http_requests_total > 1000

# bool 반환 (0 또는 1)
http_requests_total > bool 1000
```

### 벡터 매칭
```promql
# 같은 레이블을 가진 시계열끼리 연산
http_request_duration_seconds_sum
/ http_request_duration_seconds_count

# on으로 매칭 레이블 지정
metric_a * on (pod) metric_b

# group_left로 one-to-many 매칭
container_memory_usage_bytes
* on (pod) group_left (node)
kube_pod_info
```

---

## 주의사항

### Staleness (스탈니스)
메트릭이 5분간 수집되지 않으면 stale로 처리됨.

### Lookback Delta
기본값 5분. 인스턴트 쿼리 시 최근 5분 내 데이터 조회.

### rate() vs irate()
- `rate()`: 장기 트렌드, 알림 규칙에 적합
- `irate()`: 순간 스파이크 감지에 적합, 그래프 불안정

### Counter Reset 처리
`rate()`와 `increase()`는 Counter 리셋을 자동 처리함.
