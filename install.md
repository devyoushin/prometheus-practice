### 1. 리포지토리 추가 및 업데이트
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. prometheus 설치 (Namespace: monitoring)
```bash
# Prometheus 서버, Alertmanager, Node Exporter 등 핵심 요소만 설치할 때 사용합니다.
helm install prometheus prometheus-community/prometheus --create-namespace --namespace monitoring
```

