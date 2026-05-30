# 보안 체크리스트 — prometheus-practice

## Prometheus 보안 설정

### 인증/인가
- [ ] Prometheus UI에 기본 인증(Basic Auth) 또는 OAuth 적용
- [ ] Alertmanager API 접근 제한
- [ ] /metrics 엔드포인트 내부망 제한 (외부 노출 금지)
- [ ] TLS 설정 (scrape_configs의 tls_config)

### 자격증명 관리
- [ ] Kubernetes Secret으로 스크레이핑 자격증명 관리
- [ ] bearer_token 파일 권한 600 이하
- [ ] 자격증명 하드코딩 금지 (설정 파일에 직접 입력 금지)

### 네트워크 보안
- [ ] NetworkPolicy로 Prometheus → 타겟 포트만 허용
- [ ] Prometheus 포트(9090) 외부 노출 최소화
- [ ] Alertmanager 포트(9093) 접근 제한

### 데이터 보안
- [ ] 민감 레이블(token, password 등) metric_relabel_configs로 제거
- [ ] 로그에 자격증명 출력 여부 확인
- [ ] 스냅샷 백업 파일 접근 제한

## 정기 보안 점검 (월별)
- [ ] 불필요한 scrape 타겟 제거
- [ ] 사용하지 않는 알림 규칙 정리
- [ ] 자격증명 순환(Rotation) 수행
- [ ] Prometheus 버전 보안 업데이트 확인
