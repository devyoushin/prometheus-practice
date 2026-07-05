# 메트릭 타입 가이드

## 4가지 기본 타입

### 1. Counter (카운터)

**특징**: 단조 증가하는 값. 재시작 시 0으로 리셋.

**용도**: 요청 수, 에러 수, 처리된 바이트 수

```
# HELP http_requests_total HTTP 요청 총 수
# TYPE http_requests_total counter
http_requests_total{method="GET", status="200"} 1234
http_requests_total{method="POST", status="500"} 5
```

**PromQL 활용**:
```promql
# 5분간 초당 요청 수
rate(http_requests_total[5m])

# 5분간 에러율
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m])
```

> **주의**: Counter는 항상 `rate()` 또는 `increase()`와 함께 사용. 절대값은 의미없음.

---

### 2. Gauge (게이지)

**특징**: 증가/감소 모두 가능한 현재값.

**용도**: 현재 메모리 사용량, 동시 연결 수, 큐 크기, 온도

```
# HELP node_memory_MemAvailable_bytes 가용 메모리 (bytes)
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 2147483648

# HELP http_active_requests 현재 활성 요청 수
# TYPE http_active_requests gauge
http_active_requests 42
```

**PromQL 활용**:
```promql
# 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 최근 1시간 최대 활성 요청 수
max_over_time(http_active_requests[1h])
```

---

### 3. Histogram (히스토그램)

**특징**: 관측값을 버킷으로 분류하여 분포 측정. 자동으로 3가지 시계열 생성:
- `_bucket{le="0.1"}` — 0.1초 이하 요청 수 (누적)
- `_sum` — 관측값의 합계
- `_count` — 관측 횟수

**용도**: 응답 시간, 요청 크기 분포

```
# HELP http_request_duration_seconds HTTP 요청 응답 시간
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 100
http_request_duration_seconds_bucket{le="0.01"}  150
http_request_duration_seconds_bucket{le="0.025"} 200
http_request_duration_seconds_bucket{le="0.05"}  250
http_request_duration_seconds_bucket{le="0.1"}   300
http_request_duration_seconds_bucket{le="0.25"}  350
http_request_duration_seconds_bucket{le="0.5"}   380
http_request_duration_seconds_bucket{le="1"}     390
http_request_duration_seconds_bucket{le="+Inf"}  400
http_request_duration_seconds_sum 45.3
http_request_duration_seconds_count 400
```

**PromQL 활용**:
```promql
# 90th percentile 응답 시간
histogram_quantile(0.90, rate(http_request_duration_seconds_bucket[5m]))

# 99th percentile 응답 시간
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 평균 응답 시간
rate(http_request_duration_seconds_sum[5m])
/ rate(http_request_duration_seconds_count[5m])
```

---

### 4. Summary (서머리)

**특징**: 클라이언트 측에서 분위수(quantile)를 직접 계산. Histogram과 비슷하지만 서버 측 집계 불가.

**용도**: 정확한 분위수가 필요할 때 (단일 인스턴스)

```
# HELP go_gc_duration_seconds GC 중단 시간
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.000034
go_gc_duration_seconds{quantile="0.25"} 0.000056
go_gc_duration_seconds{quantile="0.5"} 0.000089
go_gc_duration_seconds{quantile="0.75"} 0.000132
go_gc_duration_seconds{quantile="1"} 0.000456
go_gc_duration_seconds_sum 1.234
go_gc_duration_seconds_count 5678
```

---

## 타입 선택 가이드

| 상황 | 타입 |
|------|------|
| HTTP 요청 수, 에러 수 | Counter |
| 현재 파드 수, 메모리 사용량 | Gauge |
| 응답 시간 분포, 요청 크기 | Histogram |
| 단일 인스턴스의 정확한 분위수 | Summary |

### Histogram vs Summary

| 항목 | Histogram | Summary |
|------|-----------|---------|
| 분위수 계산 위치 | 서버(PromQL) | 클라이언트 |
| 여러 인스턴스 집계 | 가능 | 불가능 |
| 정확도 | 버킷 설정에 따라 근사값 | 정확값 |
| 메모리 사용 | 버킷 수에 따라 증가 | 분위수 수에 따라 증가 |
| **권장 사용 | 대부분의 경우** | 단일 인스턴스 + 정확도 필요 |

---

## 메트릭 네이밍 컨벤션

```
# 패턴: <namespace>_<subsystem>_<name>_<unit>
http_server_request_duration_seconds
http_server_requests_total
node_memory_MemAvailable_bytes
process_cpu_seconds_total
```

**규칙**:
- 소문자와 언더스코어만 사용
- 단위를 이름에 포함: `_seconds`, `_bytes`, `_total`
- Counter는 `_total`로 끝내기 (Prometheus 2.x 권장)
- 복수형 사용 금지: `request_total` ✓, `requests_total` ✗

---

## 레이블(Labels)

레이블로 메트릭을 다차원으로 분류:

```
http_requests_total{
  method="GET",
  path="/api/v1/users",
  status="200",
  instance="pod-abc123"
}
```

**레이블 주의사항**:
- 카디널리티가 높은 값(UUID, 사용자 ID)은 레이블로 사용 금지
- 레이블 조합마다 별도의 시계열 생성 → 메모리 증가
- 10개 이상의 레이블 값 조합 권장하지 않음

```promql
# 레이블로 필터링
http_requests_total{method="GET", status="200"}

# 정규식 매칭
http_requests_total{status=~"5.."}          # 5xx 에러
http_requests_total{method!="GET"}           # GET 제외
http_requests_total{path!~"/health.*"}       # 헬스체크 제외
```
