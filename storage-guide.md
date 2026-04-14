# 스토리지 가이드

## Prometheus 로컬 스토리지 (TSDB)

### 저장 구조

```
/prometheus/data/
├── 01BKGV7JBM69T2G1BGBGM6KB12/   ← 2시간짜리 블록
│   ├── chunks/
│   │   └── 000001               ← 실제 데이터
│   ├── index                    ← 레이블 인덱스
│   ├── meta.json                ← 블록 메타데이터
│   └── tombstones               ← 삭제 마커
├── 01BKGTZQ1SYQJTR4PB43C8PD98/
├── chunks_head/                 ← 현재 활성 청크 (메모리)
└── wal/                         ← Write-Ahead Log
    ├── 00000001
    └── 00000002
```

### 블록 생명주기

```
메트릭 수집
    │
    ▼
WAL (Write-Ahead Log)  ←── 재시작 시 복구용
    │  2시간마다
    ▼
in-memory block
    │  flush
    ▼
2h 블록 (디스크)
    │  compaction
    ▼
12h 블록
    │  compaction
    ▼
24h 블록
    │  compaction
    ▼
48h+ 블록 (최대 10% of retention)
    │  retention 초과 시
    ▼
삭제
```

---

## EKS에서 EBS PVC 설정

### StorageClass 확인

```bash
kubectl get storageclass
# gp2 또는 gp3 사용
```

### Helm values에서 PVC 설정

```yaml
# helm/values.yaml
prometheus:
  prometheusSpec:
    retention: 15d               # 보존 기간
    retentionSize: "50GB"        # 크기 기반 보존 (retention과 중 먼저 도달한 것 적용)
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3  # EKS gp3 StorageClass
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi      # 충분한 여유 공간 확보
```

### PVC 상태 확인

```bash
kubectl get pvc -n monitoring
kubectl describe pvc prometheus-kube-prometheus-stack-prometheus-db-0 -n monitoring
```

---

## 보존 정책 (Retention)

### 시간 기반 보존

```yaml
prometheusSpec:
  retention: 15d    # 기본값
  # 단위: d(일), h(시간), m(분)
```

### 크기 기반 보존

```yaml
prometheusSpec:
  retentionSize: "50GB"    # 이 크기 초과 시 오래된 블록부터 삭제
```

### 권장 설정

| 환경 | 보존 기간 | PVC 크기 | 이유 |
|------|----------|---------|------|
| 개발 | 3d | 10Gi | 비용 절감 |
| 스테이징 | 7d | 20Gi | 디버깅용 |
| 프로덕션 | 15d | 100Gi | 인시던트 분석 + 여유분 |
| 장기 분석 | Mimir/Thanos로 remote_write | — | TSDB 한계 극복 |

---

## 스토리지 용량 계산

### 대략적인 계산 방법

```
하루 데이터 크기 ≈ (시계열 수) × (샘플 수/일) × (샘플당 크기)

- 샘플 수/일 = 86400 / scrape_interval(15s) = 5760
- 샘플당 크기 ≈ 1-2 bytes (TSDB 압축 후)

예) 100,000 시계열, 15초 interval, 15일 보존:
100,000 × 5,760 × 1.5bytes × 15days ≈ 12.96 GB
(WAL, 인덱스, 여유분 포함 × 2배 = ~26 GB)
```

### 실제 사용량 확인

```promql
# 현재 시계열 수
prometheus_tsdb_head_series

# WAL 크기 (bytes)
prometheus_tsdb_wal_storage_size_bytes

# 블록 수
prometheus_tsdb_blocks_loaded

# Prometheus 메모리 사용량
process_resident_memory_bytes{job="prometheus"}
```

---

## 스냅샷 및 백업

### 스냅샷 생성 (TSDB Admin API 활성화 필요)

```yaml
# values.yaml
prometheusSpec:
  enableAdminAPI: true
```

```bash
# 스냅샷 생성
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# 스냅샷 위치
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 \
  -- ls /prometheus/snapshots/
```

### PVC 기반 백업 (AWS EBS Snapshot)

```bash
# PVC 이름 확인
kubectl get pvc -n monitoring

# AWS CLI로 EBS 스냅샷
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxx \
  --description "Prometheus backup $(date +%Y%m%d)"
```

---

## 문제 해결

### 디스크 부족

```bash
# 사용량 확인
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 \
  -- df -h /prometheus

# 강제 WAL 압축 (TSDB compaction)
curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

### 시계열 폭발 (Cardinality 문제)

```promql
# 카디널리티가 높은 메트릭 확인
topk(10, count by (__name__) ({__name__=~".+"}))

# 레이블 수가 많은 메트릭
topk(10, count by (__name__) (count by (__name__, job) ({__name__=~".+"})))
```

```bash
# 삭제 요청 (Admin API)
curl -X POST \
  -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=bad_metric_name'
```

### PVC 확장

```bash
# StorageClass가 allowVolumeExpansion: true인 경우
kubectl patch pvc prometheus-kube-prometheus-stack-prometheus-db-0 \
  -n monitoring \
  --type merge \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```
