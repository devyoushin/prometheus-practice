Prometheus 트러블슈팅 케이스를 추가합니다.

**사용법**: `/add-troubleshooting <증상 설명>`  **예시**: `/add-troubleshooting 타겟 scrape 실패`

형식:
```markdown
### <증상>
**원인**: <근본 원인>
**확인**:
\`\`\`bash
kubectl logs -n monitoring prometheus-<pod> -c prometheus
promtool check config /etc/prometheus/prometheus.yml
\`\`\`
**해결**: <해결 방법>
```
