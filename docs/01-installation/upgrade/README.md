# Prometheus 업그레이드 가이드

Prometheus 스택 업그레이드는 Helm release, Prometheus Operator CRD, Alertmanager, Grafana, exporter가 함께 바뀔 수 있습니다. 업그레이드 전 chart 릴리즈 노트와 CRD 변경 사항을 확인합니다.

## 1. 사전 점검

```bash
export NAMESPACE="monitoring"
export RELEASE="kube-prometheus-stack"
export CHART_VERSION="65.x.x"
export VALUES_FILE="../../../ops/config/helm/values.yaml"

helm status ${RELEASE} -n ${NAMESPACE}
helm history ${RELEASE} -n ${NAMESPACE}
helm get values ${RELEASE} -n ${NAMESPACE} > values-before-upgrade.yaml
kubectl get pods,svc -n ${NAMESPACE}
kubectl get prometheus,alertmanager,servicemonitor,podmonitor,prometheusrule -A
```

## 2. Helm 업그레이드

```bash
helm repo update prometheus-community
helm upgrade ${RELEASE} prometheus-community/kube-prometheus-stack \
  --namespace ${NAMESPACE} \
  --values ${VALUES_FILE} \
  --version ${CHART_VERSION} \
  --timeout 10m \
  --wait
```

CRD 변경이 포함된 버전은 chart 문서에 따라 CRD를 먼저 적용해야 할 수 있습니다.

## 3. 확인

```bash
kubectl get pods -n ${NAMESPACE}
kubectl rollout status deployment/${RELEASE}-operator -n ${NAMESPACE}
kubectl get prometheus,alertmanager -n ${NAMESPACE}
kubectl get prometheusrule,servicemonitor,podmonitor -A
```

Prometheus UI, Alertmanager UI, Grafana datasource 연결, 주요 alert rule 평가 상태를 확인합니다.

## 4. 롤백

```bash
helm history ${RELEASE} -n ${NAMESPACE}
helm rollback ${RELEASE} <REVISION> -n ${NAMESPACE} --wait
```

CRD schema가 이미 변경된 경우 Helm rollback만으로 완전히 되돌아가지 않을 수 있습니다. major 업그레이드는 테스트 클러스터에서 먼저 검증합니다.

## 5. systemd / Docker Compose

systemd 설치는 Prometheus, Alertmanager, node_exporter 바이너리를 한 노드씩 교체하고 서비스를 재시작합니다. Docker Compose 설치는 image tag를 변경한 뒤 `docker compose pull && docker compose up -d`로 반영합니다. 두 방식 모두 TSDB 데이터 디렉터리와 설정 파일을 먼저 백업합니다.

