# Prometheus Ops

Prometheus 운영 보조 자료를 두는 공간입니다.

| 폴더 | 내용 |
|------|------|
| `config/helm/` | kube-prometheus-stack Helm values |
| `config/manifests/` | ServiceMonitor, PodMonitor, PrometheusRule, 샘플 앱 매니페스트 |

운영 개념과 절차 문서는 `docs/`에 두고, 실제 적용 가능한 매니페스트와 설정 예시는 `ops/`에 둡니다. 설치 방식은 `docs/01-installation/install.md`에서 Helm, systemd, Docker Compose로 나눠 설명합니다.
