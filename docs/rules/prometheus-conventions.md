# Prometheus 컨벤션 규칙

## 메트릭 네이밍
- `{namespace}_{subsystem}_{name}_{unit}` 형식 준수
- 단위 접미사 필수: `_seconds`, `_bytes`, `_total`, `_ratio`
- Counter는 `_total` 접미사
- Histogram/Summary는 `_bucket`, `_sum`, `_count` 자동 생성
- 부정형 이름 금지 (예: `not_ready` → `ready` + `== 0`)

## 레이블 규칙
- 레이블명: snake_case
- 고카디널리티 레이블 금지 (user_id, request_id, IP)
- 표준 레이블 우선: `job`, `instance`, `namespace`, `pod`
- 레이블 수 최소화 (scrape 타겟당 10개 이하 권장)

## Alerting Rule 규칙
```yaml
groups:
  - name: {service}.rules
    rules:
      - alert: AlertName          # PascalCase
        expr: |
          # PromQL 표현식
        for: 5m                   # 최소 5분 (flapping 방지)
        labels:
          severity: critical      # critical / warning / info
          team: platform
        annotations:
          summary: "{{ $labels.instance }}: 요약"
          description: "상세 설명 {{ $value }}"
          runbook_url: "https://..."
```

## Recording Rule 규칙
- 이름 형식: `level:metric:operation`
- 예: `job:http_requests_total:rate5m`
- 복잡한 쿼리는 recording rule로 사전 계산

## scrape_config 규칙
- `scrape_interval`: 기본 30s (고빈도 타겟은 15s)
- `scrape_timeout`: scrape_interval의 80% 이하
- `metrics_path`: 기본 `/metrics` (변경 시 명시)
- 레이블 재지정은 `relabel_configs`로 처리
