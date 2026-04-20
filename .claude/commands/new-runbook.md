새 Prometheus 운영 런북을 생성합니다.

**사용법**: `/new-runbook <작업명>`  **예시**: `/new-runbook 알람 임계치 조정`

작업 유형: 알람 설정 변경, 스토리지 확장, HA 구성, Remote Write 설정, 업그레이드

런북 포함 내용:
- 사전 체크리스트 (현재 알람 상태, 스토리지 용량)
- 단계별 kubectl/helm 명령어
- promtool 검증
- 롤백 절차
- 모니터링 포인트
