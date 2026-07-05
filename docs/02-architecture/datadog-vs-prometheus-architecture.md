# Prometheus와 Datadog의 메트릭 수집/저장 구조 차이

## 한 줄 요약

Prometheus는 사용자가 운영하는 Prometheus 서버가 exporter나 애플리케이션의 `/metrics` 엔드포인트를 주기적으로 pull해서 로컬 TSDB에 저장하는 구조다. Datadog은 각 호스트나 클러스터에 설치된 Datadog Agent가 메트릭을 수집하거나 DogStatsD로 받은 메트릭을 집계한 뒤, HTTPS로 Datadog의 관리형 SaaS 백엔드에 전송하는 구조다.

즉, Prometheus의 중심은 "내가 운영하는 TSDB"이고 Datadog의 중심은 "Datadog이 운영하는 관측성 플랫폼"이다.

## Prometheus 구조

Prometheus의 기본 흐름은 다음과 같다.

```text
Application / Exporter
        |
        | HTTP /metrics
        v
Prometheus Server
        |
        | append samples
        v
Local TSDB
        |
        v
PromQL / Rules / Alertmanager / Grafana
```

Prometheus는 target을 주기적으로 scrape한다. 애플리케이션이 직접 Prometheus client library로 `/metrics`를 노출할 수도 있고, MySQL, Redis, Node, JVM 같은 외부 시스템은 exporter가 해당 시스템의 상태를 읽어 Prometheus 형식의 메트릭으로 변환해 노출한다.

저장은 Prometheus 서버의 로컬 TSDB가 담당한다. 공식 문서 기준으로 Prometheus는 로컬 디스크 기반 time series database를 포함하며, 수집된 샘플은 기본적으로 2시간 단위 block으로 묶인다. 현재 들어오는 샘플은 head block과 WAL(write-ahead log)을 통해 보호되고, 이후 background compaction으로 더 긴 block으로 병합된다.

운영자가 직접 신경 써야 하는 대표 항목은 다음과 같다.

- scrape target과 scrape interval
- metric label cardinality
- `--storage.tsdb.path`
- retention time 또는 retention size
- HA 구성
- 장기 저장이 필요할 때 Thanos, Cortex, Mimir, VictoriaMetrics 같은 remote storage 계층

Prometheus의 장점은 저장 구조와 쿼리 모델이 명확하고, 로컬에서 독립적으로 운영 가능하며, PromQL과 label 기반 모델을 세밀하게 제어할 수 있다는 점이다. 반대로 장기 보관, 글로벌 뷰, 멀티 클러스터 운영, 권한 관리, 대규모 cardinality 관리는 사용자가 직접 설계해야 한다.

## Datadog 구조

Datadog의 기본 흐름은 다음과 같다.

```text
Host / Container / Application / Cloud Integration
        |
        v
Datadog Agent
  - Collector: checks 실행 및 인프라 메트릭 수집
  - DogStatsD: 애플리케이션 custom metric 수신 및 집계
  - Forwarder: Datadog으로 payload 전송
        |
        | HTTPS
        v
Datadog SaaS Backend
        |
        v
Datadog UI / Dashboards / Monitors / Notebooks / APIs
```

Datadog Agent는 호스트에서 실행되는 소프트웨어다. Agent는 호스트와 서비스의 이벤트 및 메트릭을 수집해서 Datadog으로 보낸다. Agent 6/7 아키텍처에서 핵심 구성요소는 Collector와 Forwarder다. Collector는 checks를 실행해 메트릭을 수집하고, Forwarder는 payload를 Datadog으로 전송한다.

애플리케이션 custom metric은 보통 DogStatsD를 통해 Agent로 보낸다. DogStatsD는 Datadog Agent에 포함된 StatsD 호환 서버이며, UDP 8125 또는 Unix domain socket으로 메트릭을 받는다. DogStatsD는 같은 metric/tag 조합으로 들어온 여러 datapoint를 flush interval 동안 집계해 하나의 datapoint로 만든 뒤 Agent pipeline을 통해 Datadog으로 보낸다. 공식 문서 기준 DogStatsD의 기본 flush interval은 10초다.

Datadog에서 사용자가 직접 Prometheus 같은 TSDB 파일 구조를 운영하지는 않는다. Agent는 로컬에 영구 TSDB를 만드는 것이 아니라, 수집한 데이터를 Datadog의 ingestion endpoint로 전송한다. 네트워크 문제가 있을 때 Forwarder가 메모리 버퍼링을 하지만, 버퍼 한도에 도달하면 오래된 메트릭은 버려질 수 있다. 따라서 Datadog의 장기 저장, rollup, query, dashboard, monitor는 Datadog SaaS 백엔드가 담당하는 관리형 영역이다.

## 저장 관점에서의 핵심 차이

| 관점 | Prometheus | Datadog |
| --- | --- | --- |
| 수집 방향 | Prometheus가 target을 pull/scrape | Agent, API, DogStatsD, cloud integration 등이 Datadog으로 push/forward |
| 수집 주체 | Prometheus server | Datadog Agent 또는 Datadog integration/API |
| exporter 역할 | 외부 시스템을 Prometheus metric 형식으로 노출 | 비슷한 역할은 Agent integration/check가 담당. Prometheus exporter도 OpenMetrics/Prometheus integration으로 수집 가능 |
| 저장 위치 | 기본적으로 사용자가 운영하는 Prometheus 로컬 TSDB | Datadog 관리형 SaaS backend |
| 로컬 저장 | Prometheus 서버가 block, WAL, index, chunks를 디스크에 저장 | Agent는 영구 TSDB가 아니라 전송 전 버퍼링 중심 |
| 쿼리 | PromQL | Datadog metric query language/UI |
| 확장 방식 | federation, remote write, Thanos/Cortex/Mimir 등 조합 | Datadog 플랫폼에 전송하고 Datadog backend가 저장/집계/조회 처리 |
| cardinality 비용 | 주로 저장소/메모리/쿼리 성능 비용 | custom metric 수와 tag 조합이 과금 및 사용량에 직접 영향 |
| 운영 책임 | TSDB, retention, HA, scale-out을 직접 설계 | Agent 설치/설정과 tag/cardinality 관리 중심. backend 운영은 Datadog 책임 |

## Datadog은 "어떤 형태로 저장되는가?"

공식 문서만으로 확인 가능한 범위에서는 Datadog 내부 저장소의 물리 구조, 예를 들어 block layout이나 index file format 같은 구현 상세는 공개되어 있지 않다. 따라서 Prometheus처럼 "디스크에 2시간 block이 생기고 chunks/index/WAL이 이런 식으로 존재한다"라고 설명할 수 있는 공개 모델은 아니다.

대신 사용자 관점의 저장 모델은 다음처럼 이해하는 것이 정확하다.

```text
metric name + timestamp + value + tags + type + interval
```

Datadog custom metric은 metric name, value, timestamp, tags, metric type, interval 같은 속성을 갖는다. Datadog은 이 데이터를 backend에 저장하고, 조회 시에는 시간 집계(time aggregation, rollup)와 공간 집계(space aggregation, tag grouping)를 적용한다.

Datadog 문서에서는 Datadog이 많은 datapoint를 저장하기 때문에 그래프에 모든 점을 그대로 표시하지 않고, 시간 범위에 따라 datapoint를 time bucket으로 묶는 rollup을 적용한다고 설명한다. 또한 metric은 host, container, region 같은 tag 기준으로 여러 timeseries로 나뉘며, query에서 sum, min, max, avg 같은 aggregator를 사용해 결합할 수 있다.

따라서 Datadog의 저장 형태를 Prometheus 관점으로 비유하면 다음과 같다.

- Prometheus: `metric_name{label=value}` 조합이 하나의 time series가 되고, 그 샘플이 내가 운영하는 TSDB block에 저장된다.
- Datadog: `metric name + tag 조합`이 query와 집계의 기준이 되는 time series처럼 동작하며, 실제 저장과 rollup은 Datadog SaaS backend가 관리한다.

## Datadog API로 메트릭을 보낼 때의 payload 형태

Datadog에 메트릭을 보내는 방법은 크게 세 가지로 나눠서 보는 것이 좋다.

```text
1. 애플리케이션 -> DogStatsD -> Datadog Agent -> Datadog backend
2. 애플리케이션/스크립트 -> Datadog Metrics HTTP API -> Datadog backend
3. OpenTelemetry SDK/Collector -> OTLP -> Datadog Agent 또는 Datadog endpoint
```

사용자가 직접 HTTP API를 호출해서 custom metric을 보낼 때는 JSON payload를 사용한다. Datadog Metrics API 문서의 `Submit metrics` 예시는 `/api/v1/series` endpoint에 다음과 같은 형태의 body를 보낸다.

```json
{
  "series": [
    {
      "host": "test.example.com",
      "metric": "system.load.1",
      "points": [
        [
          1636629071,
          0.7
        ]
      ],
      "tags": [
        "environment:test"
      ],
      "type": "gauge"
    }
  ]
}
```

여기서 중요한 필드는 다음과 같다.

- `series`: 보낼 timeseries 목록
- `metric`: metric name
- `points`: `[timestamp, value]` 쌍의 목록
- `host`: metric이 속한 host
- `tags`: `key:value` 형태의 tag 목록
- `type`: `gauge`, `count`, `rate` 등 metric type
- `interval`: `rate`나 `count` 타입에서 의미 있는 interval

Datadog API v2도 같은 개념을 유지하지만 payload 모델 이름이 `MetricPayload`, `MetricSeries`, `MetricPoint`처럼 정리되어 있고, `points`가 객체 형태로 표현된다. TypeScript 예시 기준으로는 다음처럼 `timestamp`와 `value`를 가진 point를 `series`에 담아 보낸다.

```json
{
  "series": [
    {
      "metric": "system.load.1",
      "type": 0,
      "points": [
        {
          "timestamp": 1636629071,
          "value": 0.7
        }
      ],
      "resources": [
        {
          "name": "dummyhost",
          "type": "host"
        }
      ]
    }
  ]
}
```

HTTP header 관점에서는 일반적으로 `Content-Type: application/json` 또는 문서 예시의 `text/json`, `Accept: application/json`, `DD-API-KEY`가 사용된다. Metrics API는 payload 압축도 지원하며, 공식 문서에는 최대 payload 크기와 압축된 payload의 decompressed size 제한도 명시되어 있다.

즉, "내 서비스에서 Datadog API로 메트릭을 직접 던진다"는 말은 보통 `metric name`, `timestamp`, `value`, `tags`, `host/resources`, `type`을 담은 JSON timeseries payload를 Datadog metric intake endpoint로 POST한다는 뜻이다.

## Datadog Agent가 backend로 보낼 때의 payload 형태

Datadog Agent가 backend로 보내는 내부 payload는 일반적인 사용자가 직접 다루는 API JSON과는 다르다. Datadog은 `DataDog/agent-payload` GitHub repository에서 Agent와 Datadog backend 사이 통신에 쓰이는 payload format 설명을 공개하고 있다. 이 repository 설명에 따르면 Agent 6/7이 Datadog backend와 통신할 때 사용하는 Protocol Buffers IDL이 포함되어 있고, metrics payload는 `proto/metrics/agent_payload.proto`에 정의되어 있다.

정리하면 다음과 같다.

| 경로 | 공개된 payload 형태 | 사용자가 직접 다루는가 |
| --- | --- | --- |
| Metrics HTTP API | JSON payload | 예 |
| DogStatsD | StatsD/DogStatsD line protocol over UDP/UDS | 애플리케이션에서 사용 |
| Agent -> Datadog backend | Protocol Buffers 기반 Agent payload | 보통 직접 다루지 않음 |
| OTLP ingest | OpenTelemetry Protocol, gRPC 또는 HTTP | OTel 사용 시 다룸 |

따라서 Datadog에서 "API로 던지는 파일/type"을 질문할 때는 목적에 따라 답이 달라진다.

- 직접 custom metric을 Datadog API에 제출한다면 JSON이다.
- 애플리케이션이 Agent의 DogStatsD로 보낸다면 DogStatsD line protocol이다.
- OpenTelemetry를 쓴다면 OTLP이며, Datadog Agent는 OTLP traces/metrics/logs를 gRPC 또는 HTTP로 받을 수 있다.
- Datadog Agent가 Datadog backend로 전달하는 내부 metrics payload는 Protocol Buffers 정의를 따른다.

## Prometheus exporter와 Datadog Agent integration의 대응 관계

Prometheus에서 exporter는 "대상 시스템을 Prometheus가 scrape할 수 있는 HTTP metric endpoint로 바꾸는 어댑터"에 가깝다.

Datadog에서는 이 역할을 주로 Agent integration/check가 한다. 예를 들어 DB, Redis, NGINX, Kubernetes 같은 대상에 대해 Agent integration을 설정하면 Agent가 해당 시스템에서 메트릭을 읽고 Datadog 형식으로 전송한다. 애플리케이션 내부 custom metric은 DogStatsD client library나 Datadog API를 통해 보낼 수 있다.

둘의 대응 관계는 다음과 같이 볼 수 있다.

```text
Prometheus:
  exporter -> /metrics -> Prometheus scrape -> Prometheus TSDB

Datadog:
  Agent integration/check -> Agent forwarder -> Datadog backend
  Application -> DogStatsD -> Agent aggregation/forwarder -> Datadog backend
```

중요한 차이는 Prometheus exporter가 보통 "수동적으로 노출하고 Prometheus가 가져가는 대상"이라면, Datadog Agent는 "능동적으로 수집하고 Datadog으로 보내는 에이전트"라는 점이다.

## Kubernetes 환경에서의 차이

Kubernetes에서 Prometheus는 ServiceMonitor, PodMonitor, scrape config 등을 통해 scrape target을 찾고 `/metrics`를 pull한다. Prometheus Operator를 쓰면 이 설정을 Kubernetes 리소스로 관리할 수 있다.

Datadog은 보통 DaemonSet 형태의 Agent가 각 노드에서 실행된다. Agent는 컨테이너, 노드, kubelet, integration, DogStatsD, Autodiscovery 등을 통해 메트릭을 수집하고 Datadog으로 보낸다. Cluster Agent를 추가하면 cluster-level 메트릭, external metrics, admission controller 같은 기능을 중앙에서 처리할 수 있다.

Prometheus는 "클러스터 안에 Prometheus 서버와 TSDB가 있다"는 느낌이 강하고, Datadog은 "클러스터 안에는 Agent가 있고 저장/조회/대시보드는 Datadog 플랫폼에 있다"는 느낌이 강하다.

## 실무적으로 기억할 포인트

Prometheus를 쓸 때는 "무엇을 scrape할 것인가, label cardinality를 어떻게 제한할 것인가, retention과 장기 저장을 어떻게 할 것인가"가 핵심이다.

Datadog을 쓸 때는 "Agent가 무엇을 수집하게 할 것인가, 어떤 tag를 붙일 것인가, custom metric/tag 조합이 얼마나 늘어나는가, Datadog 비용과 cardinality를 어떻게 관리할 것인가"가 핵심이다.

Prometheus는 오픈소스 TSDB를 직접 운영하는 모델이고, Datadog은 수집 Agent와 관리형 SaaS 백엔드를 결합한 모델이다. 그래서 둘 다 metric name과 label/tag 기반의 time series를 다루지만, 운영자가 책임지는 저장 계층의 범위가 크게 다르다.

## 참고 자료

- Prometheus 공식 문서: Overview - https://prometheus.io/docs/introduction/overview/
- Prometheus 공식 문서: Exporters and integrations - https://prometheus.io/docs/instrumenting/exporters/
- Prometheus 공식 문서: Storage - https://prometheus.io/docs/prometheus/latest/storage/
- Datadog 공식 문서: Agent - https://docs.datadoghq.com/agent/
- Datadog 공식 문서: Agent Architecture - https://docs.datadoghq.com/agent/architecture/
- Datadog 공식 문서: DogStatsD - https://docs.datadoghq.com/extend/dogstatsd/
- Datadog 공식 문서: Metrics - https://docs.datadoghq.com/metrics/
- Datadog 공식 문서: Custom Metrics - https://docs.datadoghq.com/metrics/custom_metrics/
- Datadog 공식 API 문서: Metrics API - https://docs.datadoghq.com/api/latest/metrics/
- Datadog 공식 문서: OTLP Ingestion by the Datadog Agent - https://docs.datadoghq.com/opentelemetry/setup/otlp_ingest_in_the_agent/
- Datadog GitHub: agent-payload - https://github.com/DataDog/agent-payload
