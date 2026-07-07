# Ingress

## Ingress란?

클러스터 외부 트래픽을 내부 Service로 라우팅하는 **L7(HTTP/HTTPS) 로드밸런서**.  
URL 경로/호스트명 기반 라우팅과 SSL 종료를 Kubernetes 오브젝트로 관리할 수 있다.

```
외부 사용자
  → Ingress Controller (단일 진입점)
    → /store  → store-service
    → /watch  → video-service
    → SSL 종료, URL 라우팅, 인증 처리
```

> Ingress 없이 서비스마다 LoadBalancer를 만들면 클라우드 비용이 증가하고 설정이 분산된다.

---

## Ingress Controller vs Ingress Resource

| 구분 | 설명 |
|---|---|
| **Ingress Controller** | 실제 트래픽을 처리하는 구현체 (nginx, GCE, HAProxy 등) — 직접 배포 필요 |
| **Ingress Resource** | 라우팅 규칙을 정의하는 Kubernetes 오브젝트 (YAML 파일) |

> Kubernetes는 Ingress Controller를 기본으로 제공하지 않는다. 반드시 직접 설치해야 한다.

---

## Ingress Controller 배포 (nginx)

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 80
            - containerPort: 443
```

### ConfigMap (nginx 설정 분리)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
```

### Service (외부 노출)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: ingress-nginx
spec:
  type: NodePort
  selector:
    app: nginx-ingress
  ports:
    - port: 80
      targetPort: 80
    - port: 443
      targetPort: 443
```

### ServiceAccount (클러스터 감시 권한)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
```

> Ingress Controller가 클러스터의 Ingress Resource 변경을 감지하려면 적절한 RBAC 권한이 필요하다.

---

## Ingress Resource

### 1. 단일 백엔드 (모든 트래픽 → 하나의 서비스)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-simple
spec:
  defaultBackend:
    service:
      name: store-service
      port:
        number: 80
```

### 2. URL 경로 기반 라우팅 (단일 도메인)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
spec:
  rules:
    - http:
        paths:
          - path: /store
            pathType: Prefix
            backend:
              service:
                name: store-service
                port:
                  number: 80
          - path: /watch
            pathType: Prefix
            backend:
              service:
                name: video-service
                port:
                  number: 80
```

### 3. 호스트명 기반 라우팅 (멀티 도메인)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-host
spec:
  rules:
    - host: mystore.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: store-service
                port:
                  number: 80
    - host: watch.mystore.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: video-service
                port:
                  number: 80
```

---

## 라우팅 방식 비교

| 방식 | rules 수 | paths 수 | 사용 시점 |
|---|---|---|---|
| URL 경로 기반 | 1 | 여러 개 | 도메인 하나, 경로로 분기 |
| 호스트명 기반 | 여러 개 | 각 1개 이상 | 도메인 여러 개로 분기 |

---

## Annotations & rewrite-target

### 문제 상황

Ingress가 `/watch` 경로를 백엔드에 그대로 전달하면, 앱이 `/watch`를 모르기 때문에 404가 발생한다.

```
# rewrite-target 없을 때
사용자: /watch 요청
  → 백엔드 앱: /watch 수신 → 404 (앱은 / 만 처리)

# rewrite-target 적용 후
사용자: /watch 요청
  → Ingress가 / 로 변환 후 전달 → 정상 응답
```

### 단순 치환 (`/path` → `/`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /pay
            pathType: Prefix
            backend:
              service:
                name: pay-service
                port:
                  number: 8282
```

`/pay` → `/` 로 치환해서 백엔드에 전달.

### 정규식 치환 (`/something/abc` → `/abc`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-rewrite-regex
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: rewrite.bar.com
      http:
        paths:
          - path: /something(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

| 요청 경로 | 백엔드 전달 경로 |
|---|---|
| `/something` | `/` |
| `/something/abc` | `/abc` |
| `/something/a/b` | `/a/b` |

> `$2`는 정규식의 두 번째 캡처 그룹 `(.*)` 을 참조한다.

---

## TLS / SSL 설정

- 실무에서는 **cert-manager**를 사용해 인증서를 자동 발급/갱신한다.

- annotation의 `cluster-issuer`를 감지해 cert-manager가 자동으로 인증서를 발급하고 `mystore-tls` Secret에 저장한다. 
- 서브도메인은 DNS에 각각 Ingress Controller IP로 등록해야 한다.

```
cert-manager (클러스터 내 설치)
  → Let's Encrypt에서 인증서 자동 발급
  → 만료 전 자동 갱신
  → Secret으로 저장 → Ingress에서 참조
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-tls
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - wear.mystore.com
        - watch.mystore.com
      secretName: mystore-tls    # 인증서가 저장될 Secret 이름
  rules:
    - host: wear.mystore.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80
    - host: watch.mystore.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: video-service
                port:
                  number: 80
```

---

## Default Backend

규칙에 매칭되지 않는 요청을 처리하는 서비스.  
별도로 배포하지 않으면 기본 404 응답을 반환한다.

```yaml
spec:
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
```

---

## kubectl 명령어

```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl create -f ingress.yaml
kubectl delete ingress <name>
```

### Imperative 방식 (1.20+)

```bash
# 형식
kubectl create ingress <name> --rule="host/path=service:port"

# 예시: 호스트 + 경로 기반
kubectl create ingress ingress-test --rule="wear.mystore.com/wear*=wear-service:80"

# 예시: 경로만
kubectl create ingress ingress-test --rule="/store=store-service:80"
```
