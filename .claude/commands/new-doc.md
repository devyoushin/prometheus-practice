새 Prometheus 가이드 문서를 생성합니다.

**사용법**: `/new-doc <주제명>`  **예시**: `/new-doc blackbox-exporter`

주제 분류: alerting, recording-rules, exporters, service-monitor, storage, ha, security, promql

`<주제명>-guide.md` 생성 시 포함 내용:
- CLAUDE.md 환경 설정 반영 (EKS, kube-prometheus-stack)
- Prometheus YAML 또는 PrometheusRule 예시
- PromQL 쿼리 예시
- kubectl/promtool 확인 명령어
- 트러블슈팅 섹션
