# prometheus-practice

EKS + kube-prometheus-stack 기준으로 Prometheus 설치, 메트릭 수집, PromQL, 알림, 저장소, 고가용성, 대규모 운영을 정리한 개인 학습 문서입니다.

## 어디서 시작할까

- 문서 지도: `docs/README.md`
- 첫 문서: `docs/install.md`
- 설치 방식: Helm / systemd / Docker Compose
- 운영 보조 자료: `ops/README.md`
- AI 작업 지침: `CLAUDE.md`

## 구조

| 경로 | 내용 |
|------|------|
| `docs/` | 설치, 아키텍처, 메트릭, PromQL, 알림, 저장소, 운영 문서 |
| `ops/` | Helm values와 Kubernetes 매니페스트 예시 |
| `CLAUDE.md` | 이 레포에서 Claude가 참고할 작업 지침 |

## 상세 구조

```text
prometheus-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/
│   ├── rules/
│   ├── templates/
│   └── *.md
└── ops/
    ├── README.md
    └── config/
```

## 학습 경로

| 단계 | 문서 |
|------|------|
| 설치 | `docs/install.md` |
| 핵심 개념 | `docs/architecture-guide.md`, `docs/metrics-types-guide.md` |
| 쿼리 | `docs/promql-guide.md`, `docs/recording-rules-guide.md` |
| Kubernetes 연동 | `docs/service-monitor-guide.md`, `docs/exporters-guide.md` |
| 알림/저장소 | `docs/alerting-guide.md`, `docs/storage-guide.md`, `docs/remote-write-guide.md` |
| 운영 | `docs/ha-guide.md`, `docs/large-scale-operations-guide.md`, `docs/troubleshooting-guide.md` |
| 통합 | `docs/grafana-integration-guide.md`, `docs/e2e-practice.md` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| 배포 | kube-prometheus-stack Helm Chart |
| Namespace | `monitoring` |
| Storage | EBS PVC |
| Remote storage | Grafana Mimir |
