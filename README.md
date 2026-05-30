# prometheus-practice

EKS + kube-prometheus-stack 기준으로 Prometheus 설치, 메트릭 수집, PromQL, 알림, 저장소, 고가용성, 대규모 운영을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/install.md`
- 전체 흐름: 설치 -> 아키텍처 -> 메트릭/PromQL -> Kubernetes 연동 -> 알림/저장소 -> 운영/보안
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
prometheus-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/     # Claude 에이전트 프롬프트
│   ├── rules/      # 작성/운영 규칙
│   ├── templates/  # 재사용 템플릿
│   └── *.md        # 주제별 학습 문서
└── ops/
    ├── README.md
    └── config/     # Helm values와 Kubernetes 매니페스트 예시
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
