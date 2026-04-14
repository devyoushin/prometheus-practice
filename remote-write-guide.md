# Remote Write 가이드

## 개요

Prometheus의 로컬 TSDB는 보존 기간이 제한적(기본 15일)입니다. `remote_write`를 사용해 Grafana Mimir, Thanos, Cortex 등의 장기 저장소로 메트릭을 전송할 수 있습니다.

```
Prometheus TSDB (15일)
       │
       │ remote_write
       ▼
Grafana Mimir (수년간 보존)
       │
       ▼
S3 / GCS (장기 저장)
```

---

## Mimir로 Remote Write

### 기본 설정

```yaml
# helm/values.yaml
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: http://mimir-nginx.mimir.svc.cluster.local/api/v1/push
        headers:
          X-Scope-OrgID: "default"           # Mimir 테넌트 ID
        queueConfig:
          capacity: 10000
          maxShards: 50
          maxSamplesPerSend: 5000
          batchSendDeadline: 5s
```

### 다중 테넌트 Remote Write

```yaml
prometheusSpec:
  remoteWrite:
    # 프로덕션 테넌트
    - url: http://mimir-nginx.mimir.svc/api/v1/push
      headers:
        X-Scope-OrgID: "production"
      writeRelabelConfigs:
        - sourceLabels: [namespace]
          regex: "production"
          action: keep              # production 네임스페이스만 전송

    # 스테이징 테넌트
    - url: http://mimir-nginx.mimir.svc/api/v1/push
      headers:
        X-Scope-OrgID: "staging"
      writeRelabelConfigs:
        - sourceLabels: [namespace]
          regex: "staging"
          action: keep
```

---

## Thanos Sidecar로 장기 보존

### 아키텍처

```
Prometheus + Thanos Sidecar
       │  upload every 2h
       ▼
     S3 Bucket
       │
Thanos Store Gateway (오래된 블록)
Thanos Querier (통합 쿼리)
```

### kube-prometheus-stack + Thanos 설정

```yaml
prometheus:
  prometheusSpec:
    thanos:
      image: quay.io/thanos/thanos:v0.35.0
      objectStorageConfig:
        key: objstore.yml
        name: thanos-objstore-config

# Thanos 오브젝트 스토리지 설정 (Secret)
# kubectl create secret generic thanos-objstore-config \
#   --from-file=objstore.yml=./objstore.yml -n monitoring
```

```yaml
# objstore.yml
type: S3
config:
  bucket: my-thanos-bucket
  endpoint: s3.ap-northeast-2.amazonaws.com
  region: ap-northeast-2
  sse_config:
    type: SSE-S3
```

---

## Remote Write 큐 튜닝

### 설정 옵션

```yaml
remoteWrite:
  - url: http://mimir/api/v1/push
    queueConfig:
      # 큐 버퍼 용량
      capacity: 10000          # 기본: 2500

      # 병렬 샤드 (동시 전송 연결 수)
      maxShards: 50             # 기본: 200
      minShards: 1

      # 배치당 최대 샘플 수
      maxSamplesPerSend: 5000   # 기본: 500

      # 배치 전송 대기 시간
      batchSendDeadline: 5s

      # 재시도 설정
      minBackoff: 30ms
      maxBackoff: 5s
      maxRetries: 3

    writeRelabelConfigs:
      # 불필요한 메트릭 제외 (네트워크 절약)
      - sourceLabels: [__name__]
        regex: 'go_gc_.*'
        action: drop
```

### 성능 지표 모니터링

```promql
# Remote Write 지연
prometheus_remote_storage_queue_highest_sent_timestamp_seconds
- prometheus_remote_storage_queue_lowest_sent_timestamp_seconds

# 전송 실패
rate(prometheus_remote_storage_failed_samples_total[5m])

# 전송 대기 중인 샘플 수
prometheus_remote_storage_pending_samples

# 샤드 수
prometheus_remote_storage_shards

# 전송 성공률
rate(prometheus_remote_storage_succeeded_samples_total[5m])
/ (rate(prometheus_remote_storage_succeeded_samples_total[5m])
   + rate(prometheus_remote_storage_failed_samples_total[5m]))
```

---

## Remote Write 인증

### Basic Auth

```yaml
remoteWrite:
  - url: https://mimir.example.com/api/v1/push
    basicAuth:
      username:
        name: remote-write-secret
        key: username
      password:
        name: remote-write-secret
        key: password
```

### Bearer Token

```yaml
remoteWrite:
  - url: https://grafana-cloud-mimir/api/prom/push
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    # 또는
    bearerToken: "your-token-here"
```

### TLS 설정

```yaml
remoteWrite:
  - url: https://mimir.internal/api/v1/push
    tlsConfig:
      ca:
        secret:
          name: mimir-tls
          key: ca.crt
      cert:
        secret:
          name: mimir-tls
          key: tls.crt
      keySecret:
        name: mimir-tls
        key: tls.key
```

---

## Grafana Cloud Remote Write

```yaml
remoteWrite:
  - url: https://prometheus-prod-01-eu-west-0.grafana.net/api/prom/push
    basicAuth:
      username:
        name: grafana-cloud-secret
        key: username           # Grafana Cloud instance ID
      password:
        name: grafana-cloud-secret
        key: password           # Grafana Cloud API Key
```

---

## Remote Write 확인

```bash
# Prometheus UI에서 확인
# http://localhost:9090/status → Runtime & Build Information → Remote Storage

# 로그에서 에러 확인
kubectl logs -l app.kubernetes.io/name=prometheus -n monitoring -c prometheus \
  | grep "remote_write"

# 메트릭으로 확인
# http://localhost:9090/graph?g0.expr=prometheus_remote_storage_pending_samples
```

---

## WAL Replay와 내구성

Remote Write는 WAL(Write-Ahead Log)를 통해 내구성을 보장합니다.

```
scrape → WAL 저장 → remote_write 전송 → WAL에서 제거
                ↑
            재시작 시 복구 (최대 2시간)
```

Prometheus 재시작 시 WAL에 남은 미전송 데이터를 자동으로 재전송합니다.
