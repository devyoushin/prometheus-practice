# Prometheus Practice

EKS 환경에서 Prometheus를 활용한 Kubernetes 모니터링 실습 가이드입니다.

## 학습 경로

### 1단계: 설치 및 기본 설정
- [install.md](install.md) — kube-prometheus-stack Helm 설치

### 2단계: 핵심 개념 이해
- [architecture-guide.md](architecture-guide.md) — Prometheus 아키텍처 및 컴포넌트
- [metrics-types-guide.md](metrics-types-guide.md) — 메트릭 타입 (Counter, Gauge, Histogram, Summary)

### 3단계: PromQL 활용
- [promql-guide.md](promql-guide.md) — PromQL 기초부터 고급까지

### 4단계: Kubernetes 연동
- [service-monitor-guide.md](service-monitor-guide.md) — ServiceMonitor / PodMonitor 설정
- [exporters-guide.md](exporters-guide.md) — node-exporter, kube-state-metrics, blackbox-exporter

### 5단계: 알림 및 규칙
- [alerting-guide.md](alerting-guide.md) — AlertManager와 알림 규칙 설정
- [recording-rules-guide.md](recording-rules-guide.md) — Recording Rules로 쿼리 최적화

### 6단계: 저장소 및 확장
- [storage-guide.md](storage-guide.md) — 로컬 스토리지, 보존 정책, EBS
- [remote-write-guide.md](remote-write-guide.md) — Mimir/Thanos로 장기 보존

### 7단계: 운영 및 고가용성
- [ha-guide.md](ha-guide.md) — Prometheus HA 구성 전략
- [grafana-integration-guide.md](grafana-integration-guide.md) — Grafana 연동 및 대시보드
- [security-guide.md](security-guide.md) — TLS, 인증, RBAC

### 8단계: 트러블슈팅
- [troubleshooting-guide.md](troubleshooting-guide.md) — 자주 발생하는 문제와 해결법

### 통합 실습
- [e2e-practice.md](e2e-practice.md) — 설치부터 알림까지 End-to-End 실습

## 예제 파일
- [helm/values.yaml](helm/values.yaml) — 개발용 Helm values
- [helm/values-prod.yaml](helm/values-prod.yaml) — 프로덕션용 Helm values
- [manifests/](manifests/) — Kubernetes 리소스 예제 모음

## 주요 컴포넌트

| 컴포넌트 | 역할 |
|---------|------|
| Prometheus Server | 메트릭 수집 및 저장, PromQL 처리 |
| AlertManager | 알림 라우팅, 그룹핑, 억제 |
| node-exporter | 노드(OS) 메트릭 수집 |
| kube-state-metrics | Kubernetes 오브젝트 상태 메트릭 |
| Grafana | 시각화 및 대시보드 |
| Pushgateway | 배치 잡 메트릭 수집 |
| blackbox-exporter | 외부 엔드포인트 프로브 |

## 환경 정보
- EKS + kube-prometheus-stack
- 네임스페이스: `monitoring`
- 저장소: EBS PVC (기본), Mimir (장기 보존)
