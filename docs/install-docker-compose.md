# Prometheus Docker Compose 설치

로컬 개발이나 빠른 검증용으로 Prometheus 스택을 `docker compose`로 올릴 때 사용하는 문서다.

## 대상

- `prometheus`
- `alertmanager`
- `grafana`

## 준비물

- `compose.yaml`
- Prometheus 설정 파일
- Alertmanager 설정 파일
- Grafana provisioning 폴더

## 예시 흐름

1. `compose.yaml`에 이미지, 포트, 볼륨을 정의한다.
2. 설정 파일을 bind mount 한다.
3. `docker compose up -d`로 스택을 띄운다.
4. `docker compose ps`, `docker compose logs -f`로 상태를 본다.

## 확인 명령

```bash
docker compose ps
docker compose logs -f prometheus
docker compose logs -f alertmanager
curl http://localhost:9090/-/ready
curl http://localhost:9093/-/ready
```

## 운영 포인트

- 개발용은 단일 호스트 기준으로 유지한다.
- 볼륨은 데이터 보존이 필요한 서비스만 사용한다.
- 운영 환경은 Helm 또는 systemd 문서를 따른다.
