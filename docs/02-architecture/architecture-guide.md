# Prometheus 아키텍처 가이드

## 전체 구조

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                     │
│                                                           │
│  ┌──────────────┐    scrape    ┌─────────────────────┐  │
│  │ node-exporter│◄─────────────│                     │  │
│  └──────────────┘              │   Prometheus Server  │  │
│  ┌──────────────┐    scrape    │                     │  │
│  │kube-state-   │◄─────────────│  - Retrieval        │  │
│  │metrics       │              │  - TSDB Storage      │  │
│  └──────────────┘              │  - HTTP API          │  │
│  ┌──────────────┐    scrape    │  - Rule Engine       │  │
│  │ App Metrics  │◄─────────────│                     │  │
│  │ /metrics     │              └────────┬────────────┘  │
│  └──────────────┘                       │                │
│                                         │ alerts         │
│  ┌──────────────┐                       ▼                │
│  │  Pushgateway │───push───►   ┌─────────────────────┐  │
│  └──────────────┘              │   AlertManager       │  │
│                                │  - Routing           │  │
│                                │  - Grouping          │  │
│                                │  - Silencing         │  │
│                                └────────┬────────────┘  │
│                                         │                │
└─────────────────────────────────────────┼────────────────┘
                                          │ notify
                         ┌────────────────┼────────────────┐
                         ▼                ▼                ▼
                      Slack            PagerDuty         Email
```

---

## 핵심 컴포넌트

### 1. Prometheus Server

메트릭 수집, 저장, 쿼리를 담당하는 핵심 컴포넌트.

**주요 기능**:
- **Retrieval**: 대상(target)으로부터 주기적으로 메트릭을 pull
- **TSDB (Time Series Database)**: 로컬 시계열 데이터베이스에 저장
- **HTTP API**: PromQL 쿼리 및 외부 연동 API 제공
- **Rule Engine**: Recording Rules, Alert Rules 평가

**설정 파일 (prometheus.yaml)**:
```yaml
global:
  scrape_interval: 15s        # 메트릭 수집 주기
  evaluation_interval: 15s    # 규칙 평가 주기

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

---

### 2. Prometheus Operator

kube-prometheus-stack에 포함된 컨트롤러. CRD를 통해 Prometheus 설정을 선언적으로 관리.

**역할**:
- `ServiceMonitor` → prometheus.yaml scrape_config 자동 생성
- `PrometheusRule` → 알림/Recording 규칙 자동 로드
- `AlertmanagerConfig` → AlertManager 설정 관리

```
ServiceMonitor (CRD)
       │
       ▼
Prometheus Operator
       │  generates
       ▼
prometheus.yaml scrape_configs
       │
       ▼
Prometheus Server (실제 수집)
```

---

### 3. AlertManager

Prometheus에서 발송된 알림을 처리하는 컴포넌트.

**처리 흐름**:
```
Prometheus ──firing alert──► AlertManager
                                │
                    ┌───────────┴───────────┐
                    │        처리 단계        │
                    ├─ Grouping: 유사 알림 묶기 │
                    ├─ Routing: 수신자 결정    │
                    ├─ Inhibition: 상위 알림 시 하위 억제 │
                    └─ Silencing: 유지보수 중 알림 억제 │
                                │
                         최종 통보 발송
```

---

### 4. node-exporter

각 노드의 OS/하드웨어 메트릭을 수집. DaemonSet으로 배포.

**수집 메트릭 예시**:
- `node_cpu_seconds_total` — CPU 사용률
- `node_memory_MemAvailable_bytes` — 가용 메모리
- `node_disk_read_bytes_total` — 디스크 읽기
- `node_network_receive_bytes_total` — 네트워크 수신

---

### 5. kube-state-metrics

Kubernetes API 오브젝트의 상태를 메트릭으로 변환.

**수집 메트릭 예시**:
- `kube_pod_status_phase` — 파드 상태 (Running, Pending 등)
- `kube_deployment_status_replicas_available` — 가용 레플리카 수
- `kube_node_status_condition` — 노드 조건 (Ready, DiskPressure 등)
- `kube_persistentvolumeclaim_status_phase` — PVC 상태

---

### 6. Pushgateway

Pull 방식이 불가능한 배치 잡, Lambda 등이 메트릭을 push하는 중간 저장소.

```bash
# 메트릭 push 예시
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/my-batch-job
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 42.5
EOF
```

> **주의**: Pushgateway는 상태가 누적되므로 잡 완료 후 명시적으로 삭제 필요.

---

## 데이터 흐름

### Pull 기반 수집
```
1. Prometheus → Service Discovery (k8s API)
2. Service Discovery → 수집 대상 목록 반환
3. Prometheus → /metrics 엔드포인트 scrape
4. 메트릭 파싱 및 TSDB 저장
5. Rule Engine → 알림 조건 평가
6. 알림 발생 시 → AlertManager 전송
```

### Kubernetes Service Discovery
Prometheus Operator 사용 시 `ServiceMonitor`가 자동으로 서비스를 발견:
```yaml
# ServiceMonitor가 이 레이블을 가진 Service를 자동 발견
selector:
  matchLabels:
    app: my-app
```

---

## 저장소 구조 (TSDB)

```
/prometheus/data/
├── chunks_head/          # 현재 활성 청크 (메모리 + 디스크)
├── wal/                  # Write-Ahead Log (2시간치)
├── 01BKGV7JBM69T2G1BGBGM6KB12/  # 블록 (2시간 단위)
│   ├── chunks/
│   ├── index
│   └── meta.json
└── 01BKGTZQ1SYQJTR4PB43C8PD98/  # 오래된 블록
```

- **청크**: 2시간 단위로 압축된 블록 생성
- **WAL**: 재시작 시 데이터 복구용
- **기본 보존 기간**: 15일
