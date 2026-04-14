# ServiceMonitor / PodMonitor 가이드

## 개요

Prometheus Operator를 사용하면 `ServiceMonitor`와 `PodMonitor` CRD로 수집 대상을 선언적으로 관리할 수 있습니다.

```
ServiceMonitor (CRD)
       │
       ▼
Prometheus Operator
       │  configures
       ▼
Prometheus Server
       │  scrapes
       ▼
  Service → Pod /metrics
```

---

## ServiceMonitor

Service를 통해 파드의 메트릭을 수집.

### 기본 구조

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring          # Prometheus가 있는 네임스페이스
  labels:
    release: kube-prometheus-stack  # Prometheus selector와 일치해야 함
spec:
  selector:
    matchLabels:
      app: my-app                # 이 레이블을 가진 Service를 찾음
  namespaceSelector:
    matchNames:
      - default                  # 대상 Service가 있는 네임스페이스
  endpoints:
    - port: metrics              # Service의 port 이름
      interval: 30s              # 수집 주기
      path: /metrics             # 메트릭 경로 (기본값)
```

### 예제 앱 + ServiceMonitor

```yaml
# 1. 애플리케이션 Service
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app                  # ServiceMonitor selector와 일치
spec:
  selector:
    app: my-app
  ports:
    - name: metrics              # ServiceMonitor endpoint port와 일치
      port: 9090
      targetPort: 9090
    - name: http
      port: 8080
---
# 2. ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    any: true                    # 모든 네임스페이스의 my-app Service 탐색
  endpoints:
    - port: metrics
      interval: 15s
```

---

### 고급 ServiceMonitor 설정

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-advanced
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - production
      - staging
  endpoints:
    - port: metrics
      interval: 30s
      scrapeTimeout: 10s         # 타임아웃 설정
      path: /metrics
      scheme: https              # HTTPS 수집
      tlsConfig:
        insecureSkipVerify: true
      metricRelabelings:         # 수집 후 메트릭 필터링
        - sourceLabels: [__name__]
          regex: 'go_.*'
          action: drop           # go_ 메트릭 제외
      relabelings:               # 수집 전 레이블 조작
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
      honorLabels: false         # 대상의 레이블을 서버 레이블로 덮어쓸지 여부
      honorTimestamps: true
```

---

## PodMonitor

Service 없이 파드에서 직접 메트릭 수집.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pods
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    any: true
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### ServiceMonitor vs PodMonitor 선택 기준

| 상황 | 권장 |
|------|------|
| 표준 배포 (Service가 있는 경우) | ServiceMonitor |
| Service 없이 파드만 있는 경우 | PodMonitor |
| DaemonSet (node-exporter 등) | PodMonitor |
| 멀티 포트 서비스 | ServiceMonitor |

---

## Prometheus가 ServiceMonitor를 인식하는 조건

### Prometheus CR의 serviceMonitorSelector 확인

```bash
kubectl get prometheus -n monitoring -o yaml | grep -A5 serviceMonitorSelector
```

보통 kube-prometheus-stack은:
```yaml
serviceMonitorSelector:
  matchLabels:
    release: kube-prometheus-stack
```

따라서 ServiceMonitor에 반드시 이 레이블 필요:
```yaml
metadata:
  labels:
    release: kube-prometheus-stack
```

---

## Relabeling 활용

### 자주 쓰는 relabeling 패턴

```yaml
relabelings:
  # 파드 이름을 레이블로 추가
  - sourceLabels: [__meta_kubernetes_pod_name]
    targetLabel: pod

  # 네임스페이스 추가
  - sourceLabels: [__meta_kubernetes_namespace]
    targetLabel: namespace

  # 특정 어노테이션을 레이블로 변환
  - sourceLabels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    targetLabel: __metrics_path__
    regex: (.+)

  # 포트 어노테이션 사용
  - sourceLabels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    targetLabel: __address__
```

### metricRelabelings — 수집된 메트릭 가공

```yaml
metricRelabelings:
  # 불필요한 메트릭 제거 (스토리지 절약)
  - sourceLabels: [__name__]
    regex: 'go_gc_.*|go_memstats_.*'
    action: drop

  # 레이블 값 변환
  - sourceLabels: [instance]
    regex: '(.+):.*'
    replacement: '$1'
    targetLabel: host

  # 레이블 추가
  - targetLabel: environment
    replacement: production
```

---

## 확인 및 디버깅

### 수집 대상 확인

```bash
# Prometheus UI에서 확인
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# http://localhost:9090/targets
```

### 수집 실패 원인 확인

```bash
# Prometheus 로그
kubectl logs -l app.kubernetes.io/name=prometheus -n monitoring -c prometheus

# ServiceMonitor 목록
kubectl get servicemonitor -A

# 특정 ServiceMonitor 상세
kubectl describe servicemonitor my-app -n monitoring
```

### 수집 대상이 안 보일 때 체크리스트

1. ServiceMonitor의 `release` 레이블이 Prometheus CR의 `serviceMonitorSelector`와 일치하는가?
2. ServiceMonitor의 `selector.matchLabels`가 실제 Service의 레이블과 일치하는가?
3. ServiceMonitor의 `endpoints.port`가 Service의 `port.name`과 일치하는가?
4. `namespaceSelector`가 Service의 네임스페이스를 포함하는가?
5. RBAC: Prometheus가 대상 네임스페이스의 Service를 읽을 권한이 있는가?
