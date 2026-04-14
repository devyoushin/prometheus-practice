# Grafana 연동 가이드

## 개요

kube-prometheus-stack에는 Grafana가 포함되어 Prometheus 데이터소스가 자동으로 설정됩니다.

---

## Grafana 접근

```bash
# Port-forward
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring

# 기본 계정
# admin / prom-operator

# 비밀번호 확인
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

---

## 데이터소스 설정

### 자동 설정 확인

kube-prometheus-stack 설치 시 Prometheus 데이터소스가 자동 설정됩니다:

```bash
# 데이터소스 목록
curl -s http://admin:prom-operator@localhost:3000/api/datasources | jq '.[].name'
```

### 수동 추가 (values.yaml)

```yaml
# helm/values.yaml
grafana:
  additionalDataSources:
    # Mimir 데이터소스 추가
    - name: Mimir
      type: prometheus
      url: http://mimir-nginx.mimir.svc.cluster.local/prometheus
      access: proxy
      httpHeaderName1: X-Scope-OrgID
      httpHeaderValue1: "default"
      isDefault: false
      jsonData:
        timeInterval: "60s"
        prometheusType: Mimir

    # Thanos 데이터소스 추가
    - name: Thanos
      type: prometheus
      url: http://thanos-query.monitoring.svc:9090
      access: proxy
      isDefault: false
```

---

## 기본 내장 대시보드

kube-prometheus-stack은 다수의 사전 구성 대시보드를 제공합니다:

| 대시보드 | 내용 |
|---------|------|
| Kubernetes / Cluster | 클러스터 전체 리소스 |
| Kubernetes / Nodes | 노드별 CPU/메모리/디스크 |
| Kubernetes / Pods | 파드별 리소스 |
| Kubernetes / Namespaces | 네임스페이스별 리소스 |
| Kubernetes / API Server | k8s API 성능 |
| Node Exporter / Full | 상세 OS 메트릭 |
| Prometheus / Overview | Prometheus 자체 모니터링 |
| AlertManager | 알림 현황 |

---

## 커스텀 대시보드 (ConfigMap)

### values.yaml로 대시보드 프로비저닝

```yaml
grafana:
  dashboards:
    default:
      my-app-dashboard:
        json: |
          {
            "title": "My Application",
            "panels": [...]
          }
```

### ConfigMap으로 대시보드 추가

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"      # 이 레이블로 Grafana가 자동 감지
data:
  my-app-dashboard.json: |
    {
      "title": "My Application Dashboard",
      "uid": "my-app",
      "schemaVersion": 36,
      "version": 1,
      "panels": [
        {
          "title": "Request Rate",
          "type": "timeseries",
          "targets": [
            {
              "expr": "rate(http_requests_total[5m])",
              "legendFormat": "{{ method }} {{ status }}"
            }
          ]
        },
        {
          "title": "Error Rate",
          "type": "gauge",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))",
              "legendFormat": "Error Rate"
            }
          ]
        }
      ]
    }
```

---

## 알림 채널 설정 (Grafana Alerting)

### values.yaml에서 Slack 알림 채널 설정

```yaml
grafana:
  grafana.ini:
    unified_alerting:
      enabled: true
  alerting:
    contactpoints.yaml:
      apiVersion: 1
      contactPoints:
        - orgId: 1
          name: slack-notifications
          receivers:
            - uid: slack-uid-001
              type: slack
              settings:
                url: https://hooks.slack.com/services/xxx
                channel: "#grafana-alerts"
                title: "{{ template \"default.title\" . }}"
                text: "{{ template \"default.message\" . }}"
```

---

## 유용한 패널 쿼리 모음

### Kubernetes Overview 패널

```promql
# 클러스터 CPU 사용률
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
/ sum(kube_node_status_allocatable{resource="cpu"}) * 100

# 클러스터 메모리 사용률
sum(container_memory_working_set_bytes{container!=""})
/ sum(kube_node_status_allocatable{resource="memory"}) * 100

# 실행 중인 파드 수
count(kube_pod_status_phase{phase="Running"})

# 노드 수
count(kube_node_info)
```

### 애플리케이션 SLO 패널

```promql
# 가용성 (99.9% SLO)
(1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  / sum(rate(http_requests_total[30d]))
)) * 100

# 응답 시간 P99 (SLO: 1초 이내)
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

---

## Grafana 설정 (values.yaml)

```yaml
grafana:
  enabled: true
  adminPassword: "your-secure-password"

  persistence:
    enabled: true
    size: 10Gi
    storageClassName: gp3

  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - grafana.example.com
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.example.com

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  grafana.ini:
    server:
      root_url: https://grafana.example.com
    auth.anonymous:
      enabled: false
    security:
      allow_embedding: true
```
