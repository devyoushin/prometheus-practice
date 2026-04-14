# Exporters 가이드

## 개요

Exporter는 메트릭을 직접 노출하지 않는 서비스(데이터베이스, OS, 하드웨어 등)의 메트릭을 Prometheus 포맷으로 변환하는 어댑터입니다.

```
MySQL ──(MySQL protocol)──► MySQL Exporter ──(/metrics)──► Prometheus
OS    ──(syscalls)────────► node-exporter  ──(/metrics)──► Prometheus
k8s API ──(REST API)──────► kube-state-metrics ──(/metrics)──► Prometheus
```

---

## 1. node-exporter

**용도**: OS/하드웨어 메트릭 (CPU, 메모리, 디스크, 네트워크)
**배포 방식**: DaemonSet (각 노드마다 1개)

### 설치 (kube-prometheus-stack 포함)
```yaml
# values.yaml
prometheus-node-exporter:
  enabled: true
  tolerations:
    - effect: NoSchedule
      operator: Exists
```

### 주요 메트릭

```promql
# CPU 사용률
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 메모리 사용률
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 디스크 읽기/쓰기 처리량
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# 네트워크 처리량 (Mbps)
rate(node_network_receive_bytes_total[5m]) * 8 / 1024 / 1024
rate(node_network_transmit_bytes_total[5m]) * 8 / 1024 / 1024

# 로드 애버리지
node_load1   # 1분 평균
node_load5   # 5분 평균
node_load15  # 15분 평균

# 파일 디스크립터
node_filefd_allocated / node_filefd_maximum
```

---

## 2. kube-state-metrics

**용도**: Kubernetes API 오브젝트 상태 (Deployment, Pod, Node 등)
**배포 방식**: 단일 Deployment

### 주요 메트릭

```promql
# 파드 상태 분포
count by (namespace, phase) (kube_pod_status_phase)

# Deployment 가용성
kube_deployment_status_replicas_available
/ kube_deployment_spec_replicas

# 파드 재시작 횟수
sum by (namespace, pod, container) (
  kube_pod_container_status_restarts_total
)

# HPA 현재 레플리카
kube_horizontalpodautoscaler_status_current_replicas
kube_horizontalpodautoscaler_spec_max_replicas

# PVC 상태
kube_persistentvolumeclaim_status_phase{phase!="Bound"}

# Job 실패
kube_job_failed > 0

# 노드 상태
kube_node_status_condition{condition="Ready", status="true"}
kube_node_status_condition{condition="DiskPressure", status="true"}
kube_node_status_condition{condition="MemoryPressure", status="true"}
```

---

## 3. blackbox-exporter

**용도**: 외부 엔드포인트 프로브 (HTTP, TCP, ICMP, DNS)
**배포 방식**: Deployment

### 설치

```yaml
# values.yaml
prometheus-blackbox-exporter:
  enabled: true
```

### Probe CRD로 HTTP 체크

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: external-urls
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  module: http_2xx              # blackbox 모듈
  prober:
    url: kube-prometheus-stack-prometheus-blackbox-exporter.monitoring:9115
  targets:
    staticConfig:
      static:
        - https://api.example.com/health
        - https://www.example.com
      labels:
        environment: production
  interval: 30s
  scrapeTimeout: 10s
```

### 직접 HTTP 프로브 (ServiceMonitor)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox-probe
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  jobLabel: blackbox
  selector:
    matchLabels:
      app.kubernetes.io/name: prometheus-blackbox-exporter
  namespaceSelector:
    matchNames:
      - monitoring
  endpoints:
    - port: http
      interval: 30s
      path: /probe
      params:
        module: [http_2xx]
        target:
          - https://api.example.com/health
      metricRelabelings:
        - sourceLabels: [__param_target]
          targetLabel: target
```

### 주요 메트릭

```promql
# HTTP 엔드포인트 가용성
probe_success{job="blackbox"}

# HTTP 응답 코드
probe_http_status_code

# 응답 시간
probe_duration_seconds

# SSL 인증서 만료까지 남은 시간
probe_ssl_earliest_cert_expiry - time()

# 30일 이내 만료되는 인증서 알림
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
```

---

## 4. 데이터베이스 Exporters

### PostgreSQL Exporter

```yaml
# Helm 설치
helm install postgres-exporter prometheus-community/prometheus-postgres-exporter \
  --namespace monitoring \
  --set config.datasource.host=postgres-svc \
  --set config.datasource.user=postgres \
  --set config.datasource.password=secret \
  --set config.datasource.database=mydb
```

```promql
# 연결 수
pg_stat_activity_count

# 데이터베이스 크기
pg_database_size_bytes

# 트랜잭션 처리량
rate(pg_stat_database_xact_commit[5m])
rate(pg_stat_database_xact_rollback[5m])

# 슬로우 쿼리
pg_stat_statements_mean_exec_time_seconds > 1
```

### Redis Exporter

```yaml
helm install redis-exporter prometheus-community/prometheus-redis-exporter \
  --namespace monitoring \
  --set redisAddress=redis://redis-svc:6379
```

```promql
# 메모리 사용량
redis_memory_used_bytes

# 연결 수
redis_connected_clients

# 초당 명령 처리량
rate(redis_commands_processed_total[5m])

# 캐시 히트율
rate(redis_keyspace_hits_total[5m])
/ (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
```

### MySQL Exporter

```yaml
helm install mysql-exporter prometheus-community/prometheus-mysql-exporter \
  --namespace monitoring \
  --set mysql.host=mysql-svc \
  --set mysql.user=exporter \
  --set mysql.pass=secret
```

```promql
# 초당 쿼리 수
rate(mysql_global_status_queries[5m])

# 연결 사용률
mysql_global_status_threads_connected / mysql_global_variables_max_connections

# 슬로우 쿼리
rate(mysql_global_status_slow_queries[5m])
```

---

## 5. 커스텀 메트릭 노출 (애플리케이션)

### Go 예제

```go
import "github.com/prometheus/client_golang/prometheus"

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "status"},
    )
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, requestDuration)
}
```

### Python 예제

```python
from prometheus_client import Counter, Histogram, start_http_server

requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'status']
)
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration'
)

# /metrics 엔드포인트 시작
start_http_server(8000)
```

---

## Exporter 모범 사례

1. **전용 ServiceMonitor 사용**: 각 exporter마다 ServiceMonitor 생성
2. **scrape_timeout 설정**: 느린 exporter의 타임아웃 처리
3. **메트릭 필터링**: 불필요한 메트릭은 `metricRelabelings`로 제거
4. **인증 설정**: 외부 접근이 가능한 exporter는 인증 필수
5. **리소스 제한**: exporter의 CPU/Memory limit 설정
