Prometheus 가이드 문서 또는 YAML을 검토합니다.

**사용법**: `/review-doc <파일 경로>`  **예시**: `/review-doc alerting-guide.md`

검토 기준:
- PrometheusRule: `for` 기간 적절성, `labels` 라우팅 일치 여부
- Recording Rule: 네이밍 규칙 (`level:metric:operations`)
- ServiceMonitor: selector 정확성, scrape interval
- PromQL: 카디널리티 영향, 성능 고려
- 문서: 예시 쿼리 동작 가능 여부, 트러블슈팅 포함 여부
