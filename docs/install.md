# Prometheus 설치 안내

Prometheus 설치 방식별 문서를 분리해 둔 입구다.

## 설치 방식

| 방식 | 문서 | 설명 |
|------|------|------|
| Helm | `install-helm.md` | EKS/kube-prometheus-stack 기준 설치 |
| systemd | `install-systemd.md` | 단일 VM/베어메탈에서 Prometheus, Alertmanager, node_exporter를 서비스로 관리 |
| Docker Compose | `install-docker-compose.md` | 로컬 개발이나 빠른 검증용 스택 |

## 읽는 순서

1. `install-helm.md`
2. `install-systemd.md`
3. `install-docker-compose.md`

## 공통 기준

- 운영 환경은 Helm 기준으로 관리한다.
- systemd와 Docker Compose는 비교 학습과 단일 호스트 실습용으로 둔다.
- 각 방식별 값과 경로는 해당 문서를 따른다.
