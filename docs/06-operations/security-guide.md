# 보안 가이드

## Prometheus 보안 고려사항

Prometheus는 메트릭을 통해 시스템 내부 정보가 노출될 수 있으므로 보안 설정이 중요합니다.

---

## 1. 접근 제어 (Ingress + 인증)

### Ingress + Basic Auth

```yaml
# Prometheus Ingress
grafana:
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: basic-auth
      nginx.ingress.kubernetes.io/auth-realm: "Prometheus"
    hosts:
      - prometheus.internal.example.com
    tls:
      - secretName: prometheus-tls
        hosts:
          - prometheus.internal.example.com
```

```bash
# Basic Auth 시크릿 생성
htpasswd -c auth admin
kubectl create secret generic basic-auth \
  --from-file=auth \
  -n monitoring
```

---

## 2. RBAC 설정

### Prometheus ServiceAccount 권한

kube-prometheus-stack은 ClusterRole을 자동으로 생성합니다:

```bash
# Prometheus가 사용하는 ClusterRole 확인
kubectl get clusterrole kube-prometheus-stack-prometheus -o yaml
```

### 최소 권한 원칙

```yaml
# 특정 네임스페이스만 모니터링
prometheus:
  prometheusSpec:
    serviceMonitorNamespaceSelector:
      matchLabels:
        monitoring: enabled    # 이 레이블이 있는 네임스페이스만 허용

    podMonitorNamespaceSelector:
      matchLabels:
        monitoring: enabled
```

```bash
# 모니터링 허용 네임스페이스에 레이블 추가
kubectl label namespace my-app monitoring=enabled
```

---

## 3. 네트워크 정책 (NetworkPolicy)

```yaml
# Prometheus → 타겟 접근만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus-network-policy
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Grafana에서만 접근 허용
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: grafana
      ports:
        - port: 9090
    # AlertManager에서 접근 허용
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: alertmanager
  egress:
    # Kubernetes API 접근
    - ports:
        - port: 6443
    # 메트릭 수집 (모든 파드)
    - ports:
        - port: 9090
        - port: 9091
        - port: 9100
        - port: 8080
    # AlertManager 접근
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: alertmanager
      ports:
        - port: 9093
```

---

## 4. 시크릿 관리

### AlertManager Webhook Secret

```bash
# Slack Webhook URL을 Secret으로 관리
kubectl create secret generic alertmanager-slack \
  --from-literal=webhook-url='https://hooks.slack.com/services/xxx' \
  -n monitoring
```

```yaml
alertmanager:
  alertmanagerSpec:
    secrets:
      - alertmanager-slack    # AlertManager Pod에 마운트
```

### Remote Write 인증 Secret

```bash
kubectl create secret generic remote-write-auth \
  --from-literal=username='your-username' \
  --from-literal=password='your-password' \
  -n monitoring
```

---

## 5. TLS 설정

### Prometheus 자체 TLS

```yaml
prometheus:
  prometheusSpec:
    web:
      tlsConfig:
        cert:
          secret:
            name: prometheus-tls
            key: tls.crt
        keySecret:
          name: prometheus-tls
          key: tls.key
      httpConfig:
        http2: true
```

### Scrape 대상 TLS 검증

```yaml
# ServiceMonitor에서 TLS 설정
spec:
  endpoints:
    - port: metrics
      scheme: https
      tlsConfig:
        ca:
          configMap:
            name: ca-bundle
            key: ca.crt
        # 또는 인증서 검증 스킵 (테스트 환경만)
        insecureSkipVerify: false
```

---

## 6. Grafana 보안

```yaml
grafana:
  grafana.ini:
    security:
      admin_user: admin
      # 시크릿으로 관리 권장
      # admin_password: ${GF_SECURITY_ADMIN_PASSWORD}
      secret_key: "your-secret-key"
      disable_gravatar: true
      cookie_secure: true
      cookie_samesite: strict

    auth:
      disable_login_form: false
      disable_signout_menu: false

    auth.anonymous:
      enabled: false

    # LDAP/OAuth 연동
    auth.ldap:
      enabled: false

  # 환경 변수로 비밀번호 주입
  envFromSecret: grafana-secrets
```

---

## 7. PodSecurityContext

```yaml
prometheus:
  prometheusSpec:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 2000
      seccompProfile:
        type: RuntimeDefault
```

---

## 보안 체크리스트

- [ ] Prometheus/Grafana UI에 인증 적용 (Internal 환경도)
- [ ] HTTPS 적용 (Ingress TLS 또는 Prometheus 자체 TLS)
- [ ] NetworkPolicy로 불필요한 접근 차단
- [ ] Alertmanager 시크릿 (Slack webhook 등) Secret 리소스로 관리
- [ ] Admin API 비활성화 (enableAdminAPI: false, 기본값)
- [ ] Grafana anonymous access 비활성화
- [ ] Remote write 자격증명 Secret으로 관리
- [ ] ServiceMonitor namespaceSelector로 수집 범위 제한
