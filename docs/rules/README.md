# 문서와 운영 규칙

이 디렉터리는 Prometheus 문서와 운영 절차를 쓸 때 지켜야 할 기준을 모은다.

## 문서 목록

| 문서 | 용도 |
|------|------|
| `doc-writing.md` | 문서 작성 규칙 |
| `monitoring.md` | 모니터링 문서 기준 |
| `prometheus-conventions.md` | Prometheus 전용 컨벤션 |
| `security-checklist.md` | 보안 점검 기준 |

## 어떻게 쓰나

1. 새 문서를 만들 때 `doc-writing.md`를 먼저 본다.
2. 메트릭/알림 규칙은 `prometheus-conventions.md`를 따른다.
3. 알림, 로그, 접근 권한이 걸리면 `security-checklist.md`를 확인한다.
4. 모니터링 문서의 표현 방식은 `monitoring.md`를 기준으로 맞춘다.

## 관련 문서

- `../agents/README.md`
- `../templates/README.md`
- `../install.md`
