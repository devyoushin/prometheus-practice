# Prometheus Docs

Prometheus를 처음 보는 사람이 설치부터 운영까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `architecture-guide.md` | Prometheus 구조와 데이터 흐름 |
| 3 | `metrics-types-guide.md` | Counter, Gauge, Histogram, Summary 이해 |
| 4 | `promql-guide.md` | PromQL 기초와 실전 쿼리 |
| 5 | `service-monitor-guide.md`, `exporters-guide.md` | Kubernetes 연동과 exporter 구성 |
| 6 | `alerting-guide.md`, `recording-rules-guide.md` | 알림과 기록 규칙 |
| 7 | `storage-guide.md`, `remote-write-guide.md` | TSDB, 보존 정책, 원격 저장소 |
| 8 | `ha-guide.md`, `ha-lb-systemd-docker-guide.md`, `large-scale-operations-guide.md` | 고가용성, 일반 서버 LB, 대규모 운영 |
| 9 | `rules/README.md` | 문서와 운영 규칙 |
| 10 | `agents/README.md` | AI 작업 지침 |
| 11 | `templates/README.md` | 문서 템플릿 |
| 12 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/초기화 | `install.md`, `architecture-guide.md` |
| 메트릭/쿼리 | `metrics-types-guide.md`, `promql-guide.md`, `recording-rules-guide.md` |
| 연동 | `service-monitor-guide.md`, `exporters-guide.md`, `grafana-integration-guide.md`, `e2e-practice.md` |
| 알림/저장소 | `alerting-guide.md`, `storage-guide.md`, `remote-write-guide.md` |
| 운영 | `ha-guide.md`, `ha-lb-systemd-docker-guide.md`, `large-scale-operations-guide.md`, `troubleshooting-guide.md`, `security-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install.md`
2. `architecture-guide.md`
3. `metrics-types-guide.md`
4. `promql-guide.md`
5. `service-monitor-guide.md`
6. `alerting-guide.md`
7. `storage-guide.md`
8. `ha-guide.md`
9. `ha-lb-systemd-docker-guide.md`
10. `rules/README.md`
11. `agents/README.md`
12. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Helm values, manifest 예시, 실행 보조 자료
