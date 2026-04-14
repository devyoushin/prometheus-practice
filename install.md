# Prometheus 설치 가이드 (kube-prometheus-stack)

## 사전 준비

### kubectl 컨텍스트 확인
```bash
kubectl config current-context
kubectl get nodes
```

### Helm 저장소 추가
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 네임스페이스 생성
```bash
kubectl create namespace monitoring
```

---

## 기본 설치 (개발용)

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values helm/values.yaml \
  --version 65.x.x
```

### 설치 확인
```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

예상 파드:
```
kube-prometheus-stack-prometheus-0
kube-prometheus-stack-alertmanager-0
kube-prometheus-stack-grafana-xxxx
kube-prometheus-stack-kube-state-metrics-xxxx
kube-prometheus-stack-prometheus-node-exporter-xxxx  (각 노드마다)
kube-prometheus-stack-operator-xxxx
```

---

## Grafana 접근

### Port-forward로 로컬 접근
```bash
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# http://localhost:3000
# 기본 계정: admin / prom-operator
```

### 비밀번호 확인
```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

---

## Prometheus UI 접근

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# http://localhost:9090
```

---

## AlertManager UI 접근

```bash
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# http://localhost:9093
```

---

## 업그레이드

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values helm/values.yaml
```

---

## 삭제

```bash
helm uninstall kube-prometheus-stack -n monitoring

# CRD도 함께 삭제하려면
kubectl delete crd \
  alertmanagerconfigs.monitoring.coreos.com \
  alertmanagers.monitoring.coreos.com \
  podmonitors.monitoring.coreos.com \
  probes.monitoring.coreos.com \
  prometheuses.monitoring.coreos.com \
  prometheusrules.monitoring.coreos.com \
  servicemonitors.monitoring.coreos.com \
  thanosrulers.monitoring.coreos.com
```

---

## 주요 CRD 목록

```bash
kubectl get crd | grep monitoring.coreos.com
```

| CRD | 용도 |
|-----|------|
| `servicemonitors` | 서비스 기반 메트릭 수집 설정 |
| `podmonitors` | 파드 기반 메트릭 수집 설정 |
| `prometheusrules` | 알림/Recording 규칙 |
| `alertmanagerconfigs` | AlertManager 설정 |
| `probes` | Blackbox Exporter 프로브 |
