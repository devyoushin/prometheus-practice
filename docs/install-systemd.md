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

## 설치 절차

1. 바이너리를 `/usr/local/bin` 또는 `/opt/<component>`에 둔다.
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

