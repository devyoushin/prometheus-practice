# Prometheus 설치 가이드 (Helm)

EKS 환경에서 `kube-prometheus-stack` Helm Chart로 Prometheus 스택을 설치한다.

## 사전 준비

```bash
kubectl config current-context
kubectl get nodes
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
```

## 설치

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values ../ops/config/helm/values.yaml \
  --version 65.x.x
```

## 설치 확인

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

예상 파드:

```text
kube-prometheus-stack-prometheus-0
kube-prometheus-stack-alertmanager-0
kube-prometheus-stack-grafana-xxxx
kube-prometheus-stack-kube-state-metrics-xxxx
kube-prometheus-stack-prometheus-node-exporter-xxxx
kube-prometheus-stack-operator-xxxx
```

## UI 접근

```bash
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
```

## 업그레이드

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values ../ops/config/helm/values.yaml
```

## 삭제

```bash
helm uninstall kube-prometheus-stack -n monitoring
```

CRD를 함께 정리해야 하면 아래를 참고한다.

```bash
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

## 자주 보는 CRD

| CRD | 용도 |
|-----|------|
| `servicemonitors` | 서비스 기반 수집 |
| `podmonitors` | 파드 기반 수집 |
| `prometheusrules` | 알림/Recording 규칙 |
| `alertmanagerconfigs` | Alertmanager 설정 |
| `probes` | Blackbox 프로브 |

