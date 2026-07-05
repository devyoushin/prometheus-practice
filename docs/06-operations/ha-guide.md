# Prometheus HA 가이드

## Prometheus HA의 특성

Prometheus는 **로컬 스토리지를 사용하기 때문에** 진정한 의미의 HA 클러스터가 아닙니다.

**HA 전략**:
1. **Active-Active 복제**: 동일 설정의 Prometheus 2개 운영 → Alertmanager에서 중복 처리
2. **Remote Storage 활용**: Mimir/Thanos로 데이터를 외부 저장 후 쿼리 통합

---

## Active-Active 구성

```
Prometheus-0  ─ scrape ─► 동일 타겟들
Prometheus-1  ─ scrape ─► 동일 타겟들
       │                        │
       └──── firing alert ───────┘
                    │
              AlertManager (중복 제거)
```

### Helm values 설정

```yaml
prometheus:
  prometheusSpec:
    replicas: 2                          # Active-Active
    replicaExternalLabelName: "__replica__"  # 레플리카 구분 레이블
    externalLabels:
      cluster: "my-eks-cluster"          # 클러스터 식별 레이블

    # 각 레플리카가 독립적인 PVC 사용
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

    # 파드 분산 배포 (다른 노드/AZ)
    podAntiAffinity: "hard"
    podAntiAffinityTopologyKey: "kubernetes.io/hostname"
```

### AlertManager에서 중복 알림 처리

Prometheus HA 사용 시 같은 알림이 2번 발송될 수 있음.
AlertManager의 클러스터링으로 자동 처리:

```yaml
alertmanager:
  alertmanagerSpec:
    replicas: 3                    # AlertManager도 HA
    clusterAdvertiseAddress: false

    # AlertManager 클러스터 피어 설정
    # (kube-prometheus-stack이 자동 처리)
```

---

## Remote Storage 기반 HA (권장)

### Thanos Sidecar + Query

```
Prometheus-0 + Thanos Sidecar ──► S3
Prometheus-1 + Thanos Sidecar ──► S3
                                    │
                          Thanos Store Gateway
                                    │
                          Thanos Querier ◄── Grafana
                          (중복 제거 처리)
```

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    replicaExternalLabelName: "__replica__"
    thanos:
      image: quay.io/thanos/thanos:v0.35.0
      objectStorageConfig:
        key: objstore.yml
        name: thanos-objstore-config
```

### Mimir remote_write

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    replicaExternalLabelName: "__replica__"
    remoteWrite:
      - url: http://mimir-nginx.mimir.svc/api/v1/push
        headers:
          X-Scope-OrgID: "production"
        # Mimir가 레플리카 중복 제거 처리
```

---

## 멀티 클러스터 모니터링

### 클러스터별 external labels

```yaml
# 클러스터 1
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: "eks-production"
      region: "ap-northeast-2"
    remoteWrite:
      - url: http://mimir-central/api/v1/push
        headers:
          X-Scope-OrgID: "production"

# 클러스터 2
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: "eks-staging"
      region: "us-east-1"
    remoteWrite:
      - url: http://mimir-central/api/v1/push
        headers:
          X-Scope-OrgID: "staging"
```

### 중앙 집계 쿼리

Mimir/Thanos에서 클러스터 레이블로 필터링:

```promql
# 특정 클러스터의 에러율
rate(http_requests_total{cluster="eks-production", status=~"5.."}[5m])

# 모든 클러스터 합산
sum by (cluster) (rate(http_requests_total{status=~"5.."}[5m]))
```

---

## AZ 분산 배포

### PodAntiAffinity 설정

```yaml
prometheus:
  prometheusSpec:
    # 서로 다른 AZ에 배포
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: prometheus
```

---

## 운영 고려사항

### 메모리 설정

Prometheus는 메모리를 많이 사용합니다:

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2000m
```

### 쿼리 동시성 제한

```yaml
prometheus:
  prometheusSpec:
    query:
      maxConcurrency: 20      # 동시 쿼리 수 제한
      timeout: 2m             # 쿼리 타임아웃
      maxSamples: 50000000    # 최대 샘플 수
```

### 샤딩 (대규모 환경)

시계열이 매우 많은 경우 (100만+), Prometheus를 샤딩:

```yaml
prometheus:
  prometheusSpec:
    shards: 3                  # 3개의 Prometheus 샤드
    # 각 샤드는 타겟의 1/3씩 담당
```
