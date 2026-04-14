# End-to-End 실습: 설치부터 알림까지

## 실습 목표

1. kube-prometheus-stack 설치
2. 예제 애플리케이션 배포 및 메트릭 수집
3. PromQL로 쿼리 작성
4. 알림 규칙 설정 및 테스트
5. Grafana 대시보드 생성

---

## Step 1: kube-prometheus-stack 설치

```bash
# Helm 저장소 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 네임스페이스 생성
kubectl create namespace monitoring

# 개발용 values로 설치
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values helm/values.yaml

# 설치 확인 (모든 파드가 Running 상태 확인)
kubectl get pods -n monitoring -w
```

**예상 소요 시간**: 2-3분

---

## Step 2: 예제 애플리케이션 배포

```bash
# 예제 앱 + ServiceMonitor 배포
kubectl apply -f manifests/example-app.yaml
kubectl apply -f manifests/servicemonitor.yaml

# 배포 확인
kubectl get pods -n default
kubectl get svc -n default
kubectl get servicemonitor -n monitoring
```

### 메트릭 노출 확인

```bash
# 예제 앱의 /metrics 확인
kubectl port-forward svc/example-app 8080:8080
curl http://localhost:8080/metrics | head -30
```

---

## Step 3: Prometheus에서 수집 확인

```bash
# Prometheus UI 접근
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
```

1. `http://localhost:9090/targets` 접속
2. `example-app` 타겟이 `UP` 상태 확인
3. `http://localhost:9090/graph` 에서 쿼리 테스트:

```promql
# 예제 앱의 HTTP 요청 수
http_requests_total{job="example-app"}

# 초당 요청 수
rate(http_requests_total{job="example-app"}[5m])

# 에러율
rate(http_requests_total{job="example-app", status=~"5.."}[5m])
/ rate(http_requests_total{job="example-app"}[5m])
```

---

## Step 4: 부하 발생 및 메트릭 변화 관찰

```bash
# 부하 발생 (다른 터미널에서)
kubectl run load-generator --image=busybox --rm -it --restart=Never -- \
  sh -c "while true; do wget -qO- http://example-app.default:8080/; sleep 0.1; done"
```

Prometheus UI에서 실시간 변화 확인:
```promql
rate(http_requests_total{job="example-app"}[1m])
```

---

## Step 5: 알림 규칙 배포 및 테스트

```bash
# 알림 규칙 배포
kubectl apply -f manifests/prometheusrule.yaml

# 규칙 로드 확인 (Prometheus UI)
# http://localhost:9090/alerts
```

### 알림 트리거 테스트

에러 발생 엔드포인트에 요청:
```bash
# 5xx 에러 유발
for i in $(seq 1 100); do
  curl http://localhost:8080/error 2>/dev/null
done
```

Prometheus UI에서:
1. `http://localhost:9090/alerts` → `HighErrorRate` 알림 상태 확인
2. `INACTIVE` → `PENDING` → `FIRING` 전환 관찰 (for: 5m 대기)

AlertManager에서:
```bash
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# http://localhost:9093 에서 알림 확인
```

---

## Step 6: Recording Rules 적용

```bash
# Recording Rules 배포
kubectl apply -f manifests/recording-rules.yaml
```

기다린 후 생성된 메트릭 확인:
```promql
# Recording Rule로 생성된 메트릭
job:http_requests_total:rate5m{job="example-app"}
```

---

## Step 7: Grafana 대시보드 생성

```bash
# Grafana 접근
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# admin / prom-operator
```

### 수동으로 대시보드 생성

1. **Dashboards → New → New Dashboard**
2. **Add visualization**
3. Data source: **Prometheus**

패널 1 - 요청 수:
```promql
rate(http_requests_total{job="example-app"}[5m])
```
- Panel type: `Time series`
- Title: `Request Rate`

패널 2 - 에러율:
```promql
rate(http_requests_total{job="example-app", status=~"5.."}[5m])
/ rate(http_requests_total{job="example-app"}[5m])
```
- Panel type: `Gauge`
- Unit: `Percent (0.0-1.0)`
- Title: `Error Rate`

패널 3 - P99 응답 시간:
```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="example-app"}[5m]))
```
- Panel type: `Time series`
- Unit: `seconds (s)`
- Title: `P99 Latency`

4. **Save dashboard** → Name: `Example App`

---

## Step 8: kube-prometheus-stack 기본 대시보드 탐색

Grafana에서 다음 대시보드들을 탐색:

- **Kubernetes / Compute Resources / Cluster**: 클러스터 전체 리소스
- **Kubernetes / Compute Resources / Node (Pods)**: 노드별 파드
- **Node Exporter / Full**: 상세 OS 메트릭
- **Prometheus / Overview**: Prometheus 자체 상태

---

## Step 9: Remote Write 설정 (옵션 - Mimir 있는 경우)

```bash
# values 업데이트
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values helm/values.yaml \
  --set prometheus.prometheusSpec.remoteWrite[0].url=http://mimir-nginx.mimir.svc/api/v1/push \
  --set 'prometheus.prometheusSpec.remoteWrite[0].headers.X-Scope-OrgID=default'

# Remote Write 상태 확인
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# http://localhost:9090/graph
# query: prometheus_remote_storage_pending_samples
```

---

## 실습 완료 체크리스트

- [ ] kube-prometheus-stack 모든 파드 Running
- [ ] 예제 앱 타겟이 Prometheus에서 UP 상태
- [ ] PromQL로 메트릭 쿼리 성공
- [ ] 부하 발생 시 메트릭 변화 확인
- [ ] 알림 규칙 PENDING/FIRING 전환 확인
- [ ] AlertManager에서 알림 수신 확인
- [ ] Recording Rules 메트릭 생성 확인
- [ ] Grafana 대시보드 생성 및 확인
- [ ] 기본 내장 대시보드 탐색

---

## 리소스 정리

```bash
kubectl delete -f manifests/
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring
```
