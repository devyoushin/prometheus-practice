# Prometheus Docs

Prometheus를 처음 보는 사람이 설치부터 운영까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install/install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `install/upgrade/` | Prometheus 업그레이드 |
| 3 | `architecture-guide.md` | Prometheus 구조와 데이터 흐름 |
| 4 | `metrics-types-guide.md` | Counter, Gauge, Histogram, Summary 이해 |
| 5 | `promql-guide.md` | PromQL 기초와 실전 쿼리 |
| 6 | `service-monitor-guide.md`, `exporters-guide.md` | Kubernetes 연동과 exporter 구성 |
| 7 | `alerting-guide.md`, `recording-rules-guide.md` | 알림과 기록 규칙 |
| 8 | `storage-guide.md`, `remote-write-guide.md` | TSDB, 보존 정책, 원격 저장소 |
| 9 | `ha-guide.md`, `ha-lb-systemd-docker-guide.md`, `large-scale-operations-guide.md` | 고가용성, 일반 서버 LB, 대규모 운영 |
| 10 | `rules/README.md` | 문서와 운영 규칙 |
| 11 | `agents/README.md` | AI 작업 지침 |
| 12 | `templates/README.md` | 문서 템플릿 |
| 13 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/초기화 | `install/install.md`, `install/upgrade/`, `architecture-guide.md` |
| 메트릭/쿼리 | `metrics-types-guide.md`, `promql-guide.md`, `recording-rules-guide.md` |
| 연동 | `service-monitor-guide.md`, `exporters-guide.md`, `grafana-integration-guide.md`, `e2e-practice.md` |
| 알림/저장소 | `alerting-guide.md`, `storage-guide.md`, `remote-write-guide.md` |
| 운영 | `ha-guide.md`, `ha-lb-systemd-docker-guide.md`, `large-scale-operations-guide.md`, `troubleshooting-guide.md`, `security-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install/install.md`
2. `install/upgrade/`
3. `architecture-guide.md`
4. `metrics-types-guide.md`
5. `promql-guide.md`
6. `service-monitor-guide.md`
7. `alerting-guide.md`
8. `storage-guide.md`
9. `ha-guide.md`
10. `ha-lb-systemd-docker-guide.md`
11. `rules/README.md`
12. `agents/README.md`
13. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Helm values, manifest 예시, 실행 보조 자료
