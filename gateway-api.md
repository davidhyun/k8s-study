# Gateway API

## 등장 배경

Ingress의 한계를 극복하기 위해 등장한 차세대 Kubernetes 네트워킹 API.

### Ingress의 한계

| 한계 | 설명 |
|---|---|
| **멀티테넌시 부족** | 단일 리소스를 여러 팀이 공유 → 충돌 발생 가능 |
| **제한적인 라우팅** | HTTP(host/path matching)만 지원 — TCP, UDP, gRPC 미지원 |
| **컨트롤러 종속 설정** | 고급 기능은 nginx/traefik 전용 annotation으로만 설정 가능 |
| **Kubernetes 비인식** | annotation 값을 K8s가 검증하지 않음 — 오류 감지 불가 |

---

## Gateway API 개요

공식 Kubernetes 프로젝트로, L4/L7 라우팅을 표준화한 차세대 API.  
annotation 없이 spec으로 모든 설정을 표현하며, 컨트롤러에 관계없이 동일하게 동작한다.

---

## 3개의 오브젝트와 담당 역할

```
인프라 제공자   → GatewayClass  (어떤 로드밸런서를 쓸지 정의)
클러스터 운영자  → Gateway       (GatewayClass의 인스턴스, 실제 진입점)
앱 개발자      → HTTPRoute 등  (라우팅 규칙 정의)
```

---

## 설치 (NGINX Gateway Fabric)

### 방법 1: kubectl apply (실습/온프레미스 환경)

```bash
# 1. Gateway API CRD 설치
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.5.1" | kubectl apply -f -

# 2. NGINX Gateway Fabric CRD 설치
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml

# 3. NGINX Gateway Fabric 배포 (NodePort 방식)
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/nodeport/deploy.yaml

# 4. 배포 확인
kubectl get pods -n nginx-gateway

# 5. 서비스 확인
kubectl get svc -n nginx-gateway nginx-gateway -o yaml

# 6. NodePort 번호 지정 (HTTP: 30080, HTTPS: 30081)
kubectl patch svc nginx-gateway -n nginx-gateway --type='json' -p='[
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30080},
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30081}
]'
```

### 방법 2: Helm (클라우드/프로덕션 환경)

```bash
# Gateway API CRD 설치 (experimental — TCP/UDP 등 포함)
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/experimental?ref=v1.6.2" | kubectl apply -f -

# NGINX Gateway Controller 설치
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
```

---

## GatewayClass

어떤 컨트롤러가 Gateway를 관리할지 정의한다.  
`controllerName`은 설치한 컨트롤러 이름과 일치해야 한다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-controller
```

---

## Gateway

트래픽 진입점과 리스너(포트/프로토콜)를 정의한다.

### HTTP

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

### HTTPS (TLS 종료)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway-tls
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: tls-secret
      allowedRoutes:
        namespaces:
          from: All
```

### TCP / UDP

```yaml
# TCP (예: MySQL 3306)
listeners:
  - name: tcp
    protocol: TCP
    port: 3306
    allowedRoutes:
      namespaces:
        from: All

# UDP (예: DNS 53)
listeners:
  - name: udp
    protocol: UDP
    port: 53
    allowedRoutes:
      namespaces:
        from: All
```

---

## HTTPRoute

라우팅 규칙을 정의한다. `parentRefs`로 Gateway에 연결한다.

### 기본 경로 라우팅

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: basic-route
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
      namespace: nginx-gateway   # Gateway가 다른 네임스페이스에 있을 때 명시
      sectionName: http          # 연결할 리스너 이름 (Gateway의 listeners[].name)
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /app
      backendRefs:
        - name: my-app
          port: 80
```

```bash
# 확인
kubectl get httproute <name>
kubectl describe httproute <name>
```

### HTTP → HTTPS 리다이렉트

```yaml
rules:
  - filters:
      - type: RequestRedirect
        requestRedirect:
          scheme: https
```

### 경로 재작성 (rewrite)

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /old
    filters:
      - type: URLRewrite
        urlRewrite:
          path:
            replacePrefixMatch: /new
    backendRefs:
      - name: my-app
        port: 80
```

`/old/foo` → `/new/foo` 로 변환 후 백엔드 전달.

### 헤더 수정

```yaml
rules:
  - filters:
      - type: RequestHeaderModifier
        requestHeaderModifier:
          add:
            - name: x-env
              value: staging
    backendRefs:
      - name: my-app
        port: 80
```

### 트래픽 분할 (카나리 배포)

```yaml
rules:
  - backendRefs:
      - name: v1-service
        port: 80
        weight: 80
      - name: v2-service
        port: 80
        weight: 20
```

### 요청 미러링

실제 응답은 `my-app`이 처리하고, `mirror-service`에 복사본을 전송한다.  
새 서비스 테스트나 트래픽 분석에 사용.

```yaml
rules:
  - filters:
      - type: RequestMirror
        requestMirror:
          backendRef:
            name: mirror-service
            port: 80
    backendRefs:
      - name: my-app
        port: 80
```

### gRPC 라우팅

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grpc-route
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - matches:
        - method:
            service: my.grpc.Service
            method: GetData
      backendRefs:
        - name: grpc-service
          port: 50051
```

---

## 지원 라우트 종류

| 종류 | 설명 |
|---|---|
| `HTTPRoute` | L7 HTTP/HTTPS 라우팅 |
| `TLSRoute` | TLS 라우팅 |
| `TCPRoute` | L4 TCP 라우팅 |
| `UDPRoute` | L4 UDP 라우팅 |
| `GRPCRoute` | gRPC 라우팅 |

---

## Ingress vs Gateway API 비교

### 트래픽 분할 (카나리)

**Ingress (nginx 전용 annotation):**
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "20"
```

**Gateway API (컨트롤러 무관):**
```yaml
backendRefs:
  - name: app-v1
    port: 80
    weight: 80
  - name: app-v2
    port: 80
    weight: 20
```

### CORS 설정

**Ingress (nginx 전용 annotation):**
```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
```

**Gateway API (spec으로 표현):**
```yaml
filters:
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      add:
        - name: Access-Control-Allow-Origin
          value: "*"
```

---

## 지원 컨트롤러 (GA)

AWS, Azure Application Gateway, Contour, Envoy Gateway, GKE, HAProxy, Istio, Nginx Gateway Fabric, Traefik 등.

---

## Ingress vs Gateway API 요약

| 항목 | Ingress | Gateway API |
|---|---|---|
| 멀티테넌시 | 단일 리소스, 팀 간 충돌 가능 | 역할별 오브젝트 분리 |
| 라우팅 지원 | HTTP만 | HTTP, TCP, UDP, gRPC 등 |
| 고급 설정 | 컨트롤러 전용 annotation | 표준 spec으로 표현 |
| 이식성 | 컨트롤러 종속 | 컨트롤러 무관 |
| K8s 검증 | annotation 검증 불가 | spec 검증 가능 |
