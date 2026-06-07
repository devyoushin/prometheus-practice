# Prometheus systemd 설치

단일 VM이나 베어메탈에서 Prometheus를 돌릴 때 사용하는 방식이다. Prometheus server, Alertmanager, node_exporter를 각각 systemd 서비스로 관리한다.

## 대상

- `prometheus.service`
- `alertmanager.service`
- `node_exporter.service`

## 준비물

- 바이너리 tarball 또는 배포 패키지
- 전용 사용자와 그룹
- 설정 디렉터리: `/etc/prometheus`, `/etc/alertmanager`
- 데이터 디렉터리: `/var/lib/prometheus`, `/var/lib/alertmanager`

## RPM 설치

Prometheus 계열은 배포판이나 사내 저장소에 따라 RPM 이름이 다를 수 있다. 공식 운영 환경에서는 사내 RPM 저장소나 검증된 패키지 저장소를 기준으로 고정 버전을 설치한다.

### 원격 RPM 파일을 직접 설치하는 경우

```bash
sudo dnf install -y ./prometheus-<VERSION>-1.x86_64.rpm
sudo dnf install -y ./alertmanager-<VERSION>-1.x86_64.rpm
sudo dnf install -y ./node_exporter-<VERSION>-1.x86_64.rpm
```

### 내부 저장소를 사용하는 경우

```bash
sudo dnf install -y prometheus alertmanager node_exporter
rpm -qa | grep -E 'prometheus|alertmanager|node_exporter'
```

RPM으로 설치한 경우에도 설정 파일과 데이터 디렉터리 위치가 패키지마다 다를 수 있으므로 아래를 먼저 확인한다.

```bash
rpm -ql prometheus | head
systemctl cat prometheus
```

## tarball 설치

공식 릴리스 tarball을 내려받아 바이너리를 배치하는 방식이다. 버전은 운영 기준에 맞게 고정한다.

```bash
PROM_VERSION="<VERSION>"
ARCH="linux-amd64"

curl -LO "https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.${ARCH}.tar.gz"
tar -xzf "prometheus-${PROM_VERSION}.${ARCH}.tar.gz"
sudo install -m 0755 "prometheus-${PROM_VERSION}.${ARCH}/prometheus" /usr/local/bin/prometheus
sudo install -m 0755 "prometheus-${PROM_VERSION}.${ARCH}/promtool" /usr/local/bin/promtool
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r "prometheus-${PROM_VERSION}.${ARCH}/consoles" /etc/prometheus/
sudo cp -r "prometheus-${PROM_VERSION}.${ARCH}/console_libraries" /etc/prometheus/
```

Alertmanager와 node_exporter도 같은 방식으로 버전을 고정해서 설치한다.

```bash
AM_VERSION="<VERSION>"
NODE_EXPORTER_VERSION="<VERSION>"

curl -LO "https://github.com/prometheus/alertmanager/releases/download/v${AM_VERSION}/alertmanager-${AM_VERSION}.linux-amd64.tar.gz"
curl -LO "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
```

## 설치 절차

1. RPM 또는 tarball 방식 중 하나로 바이너리를 설치한다.
2. 설정 파일과 rule 파일을 `/etc/<component>/`에 둔다.
3. systemd unit 파일을 `/etc/systemd/system/<service>.service`에 둔다.
4. `systemctl daemon-reload` 후 `enable --now`로 시작한다.
5. `journalctl -u <service> -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status prometheus
systemctl status alertmanager
systemctl status node_exporter
curl http://localhost:9090/-/ready
curl http://localhost:9093/-/ready
curl http://localhost:9100/metrics | head
```

## 운영 포인트

- rule 파일과 scrape config는 `ExecReload`로 재적용할 수 있게 만든다.
- 데이터 디렉터리는 삭제하지 말고 백업과 함께 다룬다.
- node_exporter는 서버별 공통 서비스로 다루고, Prometheus는 별도 구성으로 관리한다.
