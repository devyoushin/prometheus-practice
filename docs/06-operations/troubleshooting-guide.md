# 트러블슈팅 가이드

## 진단 명령어 모음

```bash
# 전체 상태 확인
kubectl get pods -n monitoring
kubectl get servicemonitor -A
kubectl get prometheusrule -A
kubectl get pvc -n monitoring

# Prometheus 로그
kubectl logs -l app.kubernetes.io/name=prometheus -n monitoring -c prometheus --tail=100

# Operator 로그
kubectl logs -l app.kubernetes.io/name=prometheus-operator -n monitoring --tail=100
```

---

## 문제 1: 메트릭 수집 안 됨 (Target Down)

### 증상
- `http://localhost:9090/targets` 에서 타겟이 `DOWN` 상태
- `up{job="..."} == 0`

### 원인 및 해결

**원인 1: ServiceMonitor 레이블 불일치**
```bash
# Prometheus CR의 serviceMonitorSelector 확인
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'

# ServiceMonitor 레이블 확인
kubectl get servicemonitor -A --show-labels

# 해결: ServiceMonitor에 올바른 레이블 추가
kubectl label servicemonitor my-app release=kube-prometheus-stack -n monitoring
```

**원인 2: Service selector 불일치**
```bash
# Service가 파드를 선택하는지 확인
kubectl get endpoints my-app-svc -n default

# Service selector와 Pod 레이블 비교
kubectl get svc my-app -n default -o jsonpath='{.spec.selector}'
kubectl get pods -n default --show-labels
```

**원인 3: 잘못된 port 이름**
```bash
# Service의 port 이름 확인
kubectl get svc my-app -n default -o jsonpath='{.spec.ports[*].name}'
# ServiceMonitor의 endpoints.port와 일치해야 함
```

**원인 4: 네트워크 정책**
```bash
# Prometheus에서 타겟으로 직접 curl 테스트
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 \
  -- wget -qO- http://my-app-svc.default:9090/metrics | head -20
```

**원인 5: RBAC 권한 부족**
```bash
# Prometheus ServiceAccount 확인
kubectl get clusterrolebinding | grep prometheus

# 특정 네임스페이스 접근 권한 확인
kubectl auth can-i get endpoints \
  --as=system:serviceaccount:monitoring:kube-prometheus-stack-prometheus \
  -n target-namespace
```

---

## 문제 2: 알림이 발송되지 않음

### 증상
- Prometheus UI에서 알림이 FIRING인데 Slack에 메시지 없음

### 원인 및 해결

**원인 1: AlertManager에 알림 전달 실패**
```bash
# AlertManager 연결 확인
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# http://localhost:9093 에서 알림 확인
```

**원인 2: PrometheusRule 레이블 불일치**
```bash
# Prometheus의 ruleSelector 확인
kubectl get prometheus -n monitoring \
  -o jsonpath='{.items[0].spec.ruleSelector}'

# PrometheusRule에 올바른 레이블 추가
kubectl label prometheusrule my-rules \
  release=kube-prometheus-stack -n monitoring
```

**원인 3: AlertManager 설정 오류**
```bash
# AlertManager 로그 확인
kubectl logs -l app.kubernetes.io/name=alertmanager -n monitoring

# AlertManager 설정 검증
kubectl get secret alertmanager-kube-prometheus-stack-alertmanager -n monitoring \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 --decode
```

**원인 4: Slack Webhook URL 오류**
```bash
# Webhook URL 직접 테스트
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"Test message"}' \
  https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

---

## 문제 3: Prometheus OOMKilled

### 증상
```bash
kubectl get pods -n monitoring
# prometheus-kube-prometheus-stack-prometheus-0   0/1   OOMKilled
```

### 원인 및 해결

**원인 1: 시계열 수 폭증**
```promql
# 시계열 수 확인
prometheus_tsdb_head_series

# 카디널리티 높은 메트릭
topk(10, count by (__name__) ({__name__=~".+"}))
```

```yaml
# 메모리 제한 증가
prometheus:
  prometheusSpec:
    resources:
      limits:
        memory: 8Gi
```

**원인 2: 과도한 쿼리 부하**
```yaml
prometheus:
  prometheusSpec:
    query:
      maxConcurrency: 10       # 동시 쿼리 수 제한
      maxSamples: 10000000     # 최대 샘플 수 제한
```

---

## 문제 4: 스토리지 부족

### 증상
```bash
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 \
  -- df -h /prometheus
# /prometheus    50G    49G    1G    98%
```

### 해결

```bash
# 1. 보존 기간 단축
helm upgrade kube-prometheus-stack ... --set prometheus.prometheusSpec.retention=7d

# 2. 불필요한 메트릭 제거 (metricRelabelings)
# ServiceMonitor에 drop 규칙 추가

# 3. PVC 확장 (gp3 지원 필요)
kubectl patch pvc prometheus-kube-prometheus-stack-prometheus-db-0 \
  -n monitoring --type merge \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

---

## 문제 5: Remote Write 지연/실패

### 증상
```promql
# 전송 대기 샘플 수 급증
prometheus_remote_storage_pending_samples > 10000

# 전송 실패
rate(prometheus_remote_storage_failed_samples_total[5m]) > 0
```

### 해결

```yaml
# 큐 설정 조정
remoteWrite:
  - url: http://mimir/api/v1/push
    queueConfig:
      maxShards: 100           # 병렬 연결 증가
      capacity: 50000          # 큐 버퍼 증가
      maxSamplesPerSend: 10000 # 배치 크기 증가
```

```bash
# 대상 서비스 상태 확인
kubectl get pods -n mimir
kubectl logs -n mimir -l app.kubernetes.io/name=mimir
```

---

## 문제 6: Grafana 대시보드 빈 화면

### 증상
- Grafana 패널에 "No data" 표시

### 원인 및 해결

**원인 1: 데이터소스 연결 실패**
```bash
# Grafana에서 Configuration → Data Sources → Test 실행
# Prometheus URL 확인
kubectl get svc -n monitoring | grep prometheus
```

**원인 2: 잘못된 시간 범위**
- Grafana 우측 상단 시간 범위를 "Last 1 hour"로 설정
- Prometheus retention 기간을 초과하지 않았는지 확인

**원인 3: 메트릭 이름 오류**
```bash
# Prometheus에서 직접 쿼리 확인
# http://localhost:9090/graph
# 메트릭 자동완성 활용
```

---

## 유용한 디버깅 쿼리

```promql
# Prometheus 자체 상태
up{job="prometheus"}
prometheus_ready

# 수집 대상 수
count(up)
count(up == 1)      # 정상 대상
count(up == 0)      # 비정상 대상

# 수집 오류
scrape_samples_scraped == 0
rate(prometheus_target_scrapes_exceeded_sample_limit_total[5m]) > 0
rate(prometheus_target_scrapes_exceeded_native_histogram_bucket_limit_total[5m]) > 0

# Rule 평가 오류
rate(prometheus_rule_evaluation_failures_total[5m]) > 0
prometheus_rule_evaluation_duration_seconds{quantile="0.99"} > 1

# Remote Write 상태
prometheus_remote_storage_queue_highest_sent_timestamp_seconds
prometheus_remote_storage_shards
```
