# Services

## 역할

Pod 간, 또는 외부 사용자와 Pod 간의 **통신을 연결**하는 오브젝트.  
마이크로서비스 간 느슨한 결합(loose coupling)을 가능하게 한다.

---

## Service 타입 3가지

| 타입 | 설명 |
|---|---|
| **NodePort** | 노드의 특정 포트를 Pod 포트에 연결 → 외부에서 접근 가능 |
| **ClusterIP** | 클러스터 내부에서만 사용 가능한 가상 IP 생성 → 서비스 간 통신 |
| **LoadBalancer** | 클라우드 환경에서 외부 로드밸런서 프로비저닝 |

---

## NodePort 상세

### 포트 3가지

```
외부 사용자
    │
    ▼ NodePort (30000~32767)
  Node
    │
    ▼ Port (Service 자체 포트)
  Service
    │
    ▼ TargetPort (Pod 포트)
   Pod
```

| 포트 | 설명 | 필수 여부 |
|---|---|---|
| `targetPort` | Pod에서 실제 앱이 listening하는 포트 | 선택 (미입력 시 port와 동일) |
| `port` | Service 자체 포트 | **필수** |
| `nodePort` | 외부에서 접근하는 노드 포트 (30000~32767) | 선택 (미입력 시 자동 할당) |

### YAML 정의 파일

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80   # Pod 포트
      port: 80         # Service 포트
      nodePort: 30008  # 외부 접근 포트
  selector:            # 연결할 Pod 선택 (Pod의 labels와 매핑)
    app: myapp
```

---

## ClusterIP 상세

여러 계층(front-end, back-end, Redis, DB 등)으로 구성된 애플리케이션에서, Pod IP는 변동되고 어떤 Pod로 연결할지도 알 수 없기 때문에 **계층 간 통신용 단일 인터페이스**가 필요하다.

```
Front-end Pods → [Service: backend]  → Back-end Pods
Back-end Pods  → [Service: redis]    → Redis Pods
```

- 각 Service는 클러스터 내부에서만 유효한 **고유 IP + 이름**을 가짐
- 다른 Pod는 **Service 이름** 또는 **ClusterIP**로 접근
- 요청은 매칭되는 Pod 중 하나로 **랜덤 포워딩**
- `type`을 생략하면 **ClusterIP가 기본값**

### YAML 정의 파일

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend       # 다른 Pod는 이 이름으로 접근
spec:
  type: ClusterIP      # 생략 가능 (기본값)
  ports:
    - targetPort: 80   # Pod 포트
      port: 80         # Service 포트
  selector:             # 연결할 Pod 선택
    app: myapp
```

---

## LoadBalancer 상세

NodePort만으로는 사용자에게 `노드IP:포트` 같은 URL을 안내해야 하는 문제가 있다. 사용자에게는 단일 URL(예: `example-voting-app.com`)이 필요하다.

### 해결 방법

| 방법 | 설명 |
|---|---|
| 직접 구성 | 별도 VM에 HAProxy, nginx 등으로 로드밸런서 구축·운영 (관리 부담 큼) |
| **LoadBalancer 타입** | 클라우드(GCP/AWS/Azure)의 **네이티브 로드밸런서**를 Kubernetes가 자동 연동 |

```yaml
spec:
  type: LoadBalancer   # NodePort 대신 지정
```

> **지원되지 않는 환경**(VirtualBox 등 온프레미스)에서는 `LoadBalancer`를 지정해도 **NodePort와 동일하게 동작**한다 (외부 로드밸런서 미생성).

---

## 다중 Pod / 다중 노드 환경

- **Pod 여러 개**: `selector`에 매칭되는 Pod 전체를 자동으로 엔드포인트로 등록 → **랜덤 알고리즘**으로 로드밸런싱
- **노드 여러 개**: 별도 설정 없이 모든 노드에 동일한 NodePort로 자동 확장

> Pod 추가/삭제 시 Service가 **자동으로 업데이트**된다.

---

## 주요 명령어

```bash
kubectl create -f service-definition.yaml  # 생성
kubectl get services                        # 목록 조회 (ClusterIP, 포트 확인)

# NodePort로 접근 테스트
curl http://192.168.1.2:30008
```
