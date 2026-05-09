# 알림 설정 가이드

## 알림 흐름

```
PrometheusRule (CRD)
       │
       ▼
Rule Engine (15s마다 평가)
       │
       ├── INACTIVE: 조건 미충족
       ├── PENDING: 조건 충족, for 기간 대기
       └── FIRING: for 기간 경과 → AlertManager 전송
                           │
                    AlertManager
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
             Slack      PagerDuty    Email
```

---

## PrometheusRule 작성

### 기본 구조

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack  # Prometheus가 인식하기 위해 필수
spec:
  groups:
    - name: my-app.rules            # 규칙 그룹 이름
      interval: 30s                 # 평가 주기 (기본: global evaluation_interval)
      rules:
        - alert: HighErrorRate      # 알림 이름
          expr: |                   # PromQL 표현식
            rate(http_requests_total{status=~"5.."}[5m])
            / rate(http_requests_total[5m]) > 0.05
          for: 5m                   # PENDING 유지 시간
          labels:
            severity: warning       # 알림 레이블 (라우팅에 사용)
            team: backend
          annotations:
            summary: "High error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"
            runbook_url: "https://wiki/runbooks/high-error-rate"
```

---

## 주요 알림 규칙 예제

### 1. 파드 상태 알림

```yaml
groups:
  - name: kubernetes-pods
    rules:
      # 파드 크래시루프
      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
          description: "Container {{ $labels.container }} restarted {{ $value }} times in 15 minutes"

      # 파드 준비되지 않음
      - alert: PodNotReady
        expr: |
          sum by (namespace, pod) (
            max by (namespace, pod, condition) (
              kube_pod_status_conditions{condition="Ready", status="true"}
            ) == 0
          ) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"

      # OOMKilled
      - alert: ContainerOOMKilled
        expr: |
          kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Container OOMKilled: {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }}"
```

### 2. 노드 알림

```yaml
  - name: kubernetes-nodes
    rules:
      # 노드 NotReady
      - alert: NodeNotReady
        expr: kube_node_status_condition{condition="Ready", status="true"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.node }} is NotReady"

      # CPU 사용률 높음
      - alert: NodeHighCPU
        expr: |
          100 - (avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          ) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}%"

      # 메모리 사용률 높음
      - alert: NodeHighMemory
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"

      # 디스크 사용률 높음
      - alert: NodeDiskSpaceLow
        expr: |
          (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
          / node_filesystem_size_bytes) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} is {{ $value | humanize }}% full"

      # 4시간 내 디스크 꽉 참 예측
      - alert: NodeDiskWillFillIn4Hours
        expr: |
          predict_linear(
            node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[1h], 4 * 3600
          ) < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk will fill in 4 hours on {{ $labels.instance }}"
```

### 3. 애플리케이션 알림

```yaml
  - name: application
    rules:
      # 높은 에러율
      - alert: HighErrorRate
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (service) (rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate: {{ $value | humanizePercentage }}"

      # 높은 응답 시간
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (service, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.service }}"
          description: "P99 latency: {{ $value | humanizeDuration }}"

      # 서비스 다운
      - alert: ServiceDown
        expr: up{job="my-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Instance {{ $labels.instance }} has been down for more than 1 minute"
```

---

## AlertManager 설정

### 기본 설정 구조

```yaml
# kube-prometheus-stack Helm values에 포함
alertmanager:
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/...'
      resolve_timeout: 5m

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s        # 첫 알림 전 대기 (같은 그룹 수집)
      group_interval: 5m     # 같은 그룹 추가 알림 간격
      repeat_interval: 12h   # 동일 알림 재발송 간격
      receiver: 'default'
      routes:
        - match:
            severity: critical
          receiver: 'pagerduty-critical'
        - match:
            severity: warning
          receiver: 'slack-warning'
        - match:
            alertname: Watchdog
          receiver: 'null'

    receivers:
      - name: 'null'

      - name: 'default'
        slack_configs:
          - channel: '#alerts'
            send_resolved: true

      - name: 'slack-warning'
        slack_configs:
          - channel: '#alerts-warning'
            title: '{{ template "slack.default.title" . }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
            send_resolved: true

      - name: 'pagerduty-critical'
        pagerduty_configs:
          - service_key: '<pagerduty-key>'
            send_resolved: true

    inhibit_rules:
      # critical이 firing 중이면 같은 서비스의 warning 억제
      - source_match:
          severity: 'critical'
        target_match:
          severity: 'warning'
        equal: ['alertname', 'cluster', 'service']
```

---

## Slack 알림 템플릿

```yaml
templates:
  - |
    {{ define "slack.custom.title" }}
    [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
    {{ end }}

    {{ define "slack.custom.text" }}
    {{ range .Alerts }}
    *Alert:* {{ .Annotations.summary }}
    *Severity:* {{ .Labels.severity }}
    *Description:* {{ .Annotations.description }}
    {{ if .Annotations.runbook_url }}*Runbook:* {{ .Annotations.runbook_url }}{{ end }}
    {{ end }}
    {{ end }}
```

---

## AlertManager 확인

```bash
# AlertManager UI
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# http://localhost:9093

# 현재 활성 알림 확인
curl http://localhost:9093/api/v2/alerts | jq .

# 사일런스 목록
curl http://localhost:9093/api/v2/silences | jq .
```

---

## 알림 상태 확인

```bash
# Prometheus에서 알림 규칙 상태 확인
# http://localhost:9090/alerts

# PrometheusRule 목록
kubectl get prometheusrule -A

# 특정 규칙 확인
kubectl get prometheusrule my-app-alerts -n monitoring -o yaml
```

---

## 알림 설계 원칙

1. **증상 기반 알림**: 원인이 아닌 사용자 영향(증상)에 집중
   - ✓ "에러율이 5% 이상"
   - ✗ "데이터베이스 연결 수가 100 이상"

2. **적절한 `for` 기간**: 일시적 스파이크 방지
   - Critical: 1-5분
   - Warning: 5-15분

3. **Runbook 링크 포함**: 알림 발생 시 즉시 조치 가능하도록

4. **알림 피로 방지**: 너무 많은 알림은 무시됨
   - `inhibit_rules`로 중복 알림 억제
   - `group_by`로 관련 알림 묶기
