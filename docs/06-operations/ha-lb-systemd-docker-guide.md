# Prometheus 일반 서버 HA와 LB 구성

컨테이너 오케스트레이션 없이 VM이나 베어메탈에서 Prometheus를 운영할 때의 이중화와 LB 구성 방식이다. Docker Compose와 systemd 모두 프로세스 실행 방식만 다를 뿐, HA 설계 원칙은 같다.

## 기본 구조

```text
Grafana
  |
Thanos Query / Mimir / Prometheus LB
  |
+--------------+--------------+
|                             |
Prometheus-1                  Prometheus-2
|                             |
same scrape targets           same scrape targets
|                             |
+--------------+--------------+
               |
        Alertmanager Cluster
```

## Prometheus HA 특성

Prometheus는 로컬 TSDB에 데이터를 저장한다. 따라서 여러 Prometheus 인스턴스를 띄워도 하나의 공유 클러스터처럼 동작하지 않는다.

일반적인 HA 방식은 **active-active scrape**이다.

```text
Prometheus-1 -> node_exporter/app exporter
Prometheus-2 -> node_exporter/app exporter
```

두 Prometheus가 같은 target을 각각 수집한다. 한쪽이 죽어도 다른 쪽에는 동일 target의 시계열이 남는다.

## Prometheus 앞단 LB

Prometheus 앞단 LB는 수집 경로가 아니라 UI/API 접근이나 Grafana datasource 접근에 사용한다.

```text
사용자/Grafana
  |
Prometheus LB
  |
+-------------+-------------+
|                           |
Prometheus-1                Prometheus-2
```

주의할 점은 Prometheus-1과 Prometheus-2의 로컬 데이터가 완전히 동일하다고 보장할 수 없다는 것이다. scrape 타이밍, 재시작, 네트워크 지연 때문에 쿼리 결과가 미묘하게 달라질 수 있다.

그래서 Prometheus LB는 보통 round-robin보다 **active/passive**가 안전하다.

```haproxy
frontend prometheus_front
    bind *:9090
    mode http
    default_backend prometheus_back

backend prometheus_back
    mode http
    option httpchk GET /-/ready
    server prom1 10.0.1.21:9090 check
    server prom2 10.0.1.22:9090 check backup
```

## 운영 권장 구조

운영 환경에서는 Grafana가 Prometheus LB를 직접 보는 것보다 Thanos Query나 Mimir를 datasource로 보는 구성이 더 안정적이다.

```text
Grafana
  |
Thanos Query / Mimir
  |
+-------------+-------------+
|                           |
Prometheus-1                Prometheus-2
```

이 구성에서는 Prometheus replica 간 중복 데이터를 Thanos나 Mimir 계층에서 처리한다.

## Alertmanager HA

Prometheus가 active-active로 동작하면 같은 alert가 여러 Prometheus에서 동시에 발생할 수 있다. Alertmanager 클러스터는 이 중복 알림을 dedup 처리한다.

```text
Prometheus-1 -> Alertmanager-1,2,3
Prometheus-2 -> Alertmanager-1,2,3

Alertmanager-1 <-> Alertmanager-2 <-> Alertmanager-3
```

Prometheus 설정 예시:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager-1:9093
            - alertmanager-2:9093
            - alertmanager-3:9093
```

Alertmanager 실행 예시:

```bash
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-1:9094 \
  --cluster.peer=alertmanager-2:9094 \
  --cluster.peer=alertmanager-3:9094
```

Alertmanager UI/API 앞단에는 round-robin LB를 둘 수 있다.

```haproxy
frontend alertmanager_front
    bind *:9093
    mode http
    default_backend alertmanager_back

backend alertmanager_back
    mode http
    balance roundrobin
    option httpchk GET /-/ready
    server am1 10.0.1.31:9093 check
    server am2 10.0.1.32:9093 check
    server am3 10.0.1.33:9093 check
```

## Docker로 배포할 때

Docker Compose를 사용해도 HA는 서버 단위로 나누어야 한다. 한 서버 안에 Prometheus 컨테이너를 여러 개 띄우는 것은 서버 장애를 막지 못한다.

```text
Server-1:
  prometheus
  alertmanager

Server-2:
  prometheus
  alertmanager

Server-3:
  alertmanager
  haproxy
```

각 Prometheus 컨테이너는 동일한 scrape config와 rule 파일을 사용하고, `external_labels`로 replica와 site 정보를 구분한다.

```yaml
global:
  external_labels:
    cluster: production
    replica: prom-1
```

## systemd로 배포할 때

systemd는 각 서버의 프로세스를 관리한다. HA 자체를 제공하지는 않는다.

```text
prometheus.service
alertmanager.service
node_exporter.service
haproxy.service
```

역할 분리는 다음처럼 가져간다.

```text
systemd: 서비스 시작, 재시작, 로그 관리
LB: health check와 트래픽 전환
Prometheus: active-active scrape
Alertmanager: alert dedup과 알림 라우팅
Thanos/Mimir: 장기 저장과 통합 쿼리
```

## 핵심 요약

```text
Prometheus HA
= Prometheus 2대 이상 active-active scrape

Prometheus 앞단 LB
= UI/API 접근용, active/passive 권장

Alertmanager HA
= 3대 이상 클러스터 구성, 중복 알림 제거

운영형 쿼리 HA
= Thanos Query 또는 Mimir 권장

Docker/systemd
= 실행 방식 차이일 뿐, HA는 서버 분산과 LB health check로 구성
```
