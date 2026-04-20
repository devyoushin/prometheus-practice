# prometheus-practice — 프로젝트 가이드

## 프로젝트 설정
- 플랫폼: EKS (Amazon Elastic Kubernetes Service)
- 설치 방식: Helm (kube-prometheus-stack)
- Prometheus 버전: 2.x (LTS)
- 네임스페이스: monitoring
- 스토리지: EBS (PersistentVolumeClaim)
- 원격 저장소: Grafana Mimir (옵션)

---

## 디렉토리 구조

```
prometheus-practice/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── .claude/
│   ├── settings.json
│   └── commands/              # /new-doc, /new-runbook, /review-doc, /add-troubleshooting, /search-kb
├── agents/                    # doc-writer, alerting-designer, exporter-advisor, troubleshooter
├── templates/                 # service-doc, runbook, incident-report
├── rules/                     # doc-writing, prometheus-conventions, security-checklist, monitoring
├── helm/                      # Helm values 파일
├── manifests/                 # Kubernetes 매니페스트
└── *-guide.md                 # 주제별 가이드 문서
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 가이드 문서 생성 | `/new-doc custom-exporter` |
| `/new-runbook` | 새 런북 생성 | `/new-runbook PrometheusTargetDown 알림 대응` |
| `/review-doc` | 문서 검토 | `/review-doc promql-guide.md` |
| `/add-troubleshooting` | 트러블슈팅 케이스 추가 | `/add-troubleshooting 스크레이핑 타임아웃` |
| `/search-kb` | 지식베이스 검색 | `/search-kb PromQL rate vs irate` |

---

## 가이드 문서 목록

| 문서 | 주제 |
|------|------|
| `install.md` | kube-prometheus-stack 설치 (Helm) |
| `architecture-guide.md` | Prometheus 아키텍처 |
| `metrics-types-guide.md` | 메트릭 타입 (Counter, Gauge, Histogram, Summary) |
| `promql-guide.md` | PromQL 기초 및 실전 |
| `recording-rules-guide.md` | Recording Rule 최적화 |
| `alerting-guide.md` | 알림 규칙 및 Alertmanager |
| `service-monitor-guide.md` | ServiceMonitor/PodMonitor |
| `exporters-guide.md` | Exporter 종류 및 활용 |
| `storage-guide.md` | TSDB 스토리지 관리 |
| `remote-write-guide.md` | Remote Write 설정 |
| `ha-guide.md` | 고가용성 구성 |
| `security-guide.md` | 보안 설정 |
| `grafana-integration-guide.md` | Grafana 연동 |
| `troubleshooting-guide.md` | 트러블슈팅 |
| `e2e-practice.md` | 엔드투엔드 실습 |

---

## 핵심 명령어

```bash
# Prometheus 상태 확인
kubectl get pods -n monitoring -l app=prometheus

# 타겟 스크레이핑 상태
curl http://localhost:9090/api/v1/targets

# 알림 규칙 확인
kubectl get prometheusrule -A

# promtool로 규칙 검증
promtool check rules rules/*.yaml

# PromQL 쿼리 (CLI)
curl 'http://localhost:9090/api/v1/query?query=up'
```
