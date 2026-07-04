# Auto Scaling

## 스케일링 2가지 방향

| 방향 | 의미 |
|---|---|
| **수직(Vertical)** | 기존 서버/Pod의 리소스(CPU/Memory)를 늘림 |
| **수평(Horizontal)** | 서버/Pod의 수를 늘림 |

---

## Kubernetes에서의 스케일링 대상

|  | 수평 | 수직 |
|---|---|---|
| **클러스터 인프라** | 노드 추가/제거 | 노드 리소스 증가 (비권장) |
| **워크로드 (Pod)** | Pod 수 증가/감소 | Pod 리소스 limits/requests 조정 |

> 클러스터 수직 스케일링은 서버 다운 필요 → 실무에서 거의 안 씀. 더 큰 노드를 새로 추가하고 기존 노드를 제거하는 방식으로 대체.

---

## 수동 vs 자동

| 대상 | 수동 | 자동 |
|---|---|---|
| 클러스터 인프라 (수평) | `kubeadm join`으로 노드 추가 | **Cluster Autoscaler** |
| 워크로드 (수평) | `kubectl scale` (Deployment/StatefulSet/ReplicaSet/RC 모두 지원) | **HPA** (Horizontal Pod Autoscaler) |
| 워크로드 (수직) | `kubectl edit`으로 requests/limits 수정 | **VPA** (Vertical Pod Autoscaler) |

---

## 자동 스케일러 3종 요약

| 스케일러 | 대상 | 동작 |
|---|---|---|
| **Cluster Autoscaler** | 클러스터 노드 | 부하에 따라 노드 자동 추가/제거 |
| **HPA** | Pod 수 | CPU/Memory 기준으로 Pod 수 자동 조정 |
| **VPA** | Pod 리소스 | Pod의 requests/limits 자동 조정 |

---

## HPA vs VPA 비교

| 항목 | HPA | VPA |
|---|---|---|
| 스케일링 방식 | Pod 수 증가/감소 | 기존 Pod의 CPU/Memory 증가/감소 |
| Pod 동작 | 기존 Pod 유지 | 새 값 적용 위해 **Pod 재시작** |
| 트래픽 스파이크 대응 | ✅ 즉각 Pod 추가 | ❌ 재시작 필요해 대응 어려움 |
| 비용 최적화 | ✅ 불필요한 idle Pod 감소 | ✅ 과도한 오버프로비저닝 방지 |
| 적합한 워크로드 | 웹서버, API, 메시지 큐, 마이크로서비스 (Stateless) | DB, JVM앱, AI/ML 워크로드 (Stateful) |

---

## HPA (Horizontal Pod Autoscaler)

CPU/Memory 또는 커스텀 메트릭 기준으로 Pod 수를 자동 조정한다.  
**v1.23부터 기본 내장**, Metrics Server가 전제 조건이다.

### 생성 방법 1: Imperative

```bash
kubectl autoscale deployment myapp --cpu-percent=50 --min=1 --max=10
```

### 생성 방법 2: Declarative (YAML)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### 조회 및 삭제

```bash
kubectl get hpa               # 현재 상태 (현재 사용량 / 임계값, 현재 replica 수)
kubectl delete hpa myapp-hpa
```

### 동작 원리

```
HPA → Metrics Server 주기적 폴링
    → CPU 사용량 > 50% → replica 증가
    → CPU 사용량 낮아짐 → replica 감소 (min 이하로 내려가지 않음)
```

> Pod의 `resources.limits.cpu`를 기준으로 사용률(%)을 계산하므로 limits 설정이 필수다.

---

## VPA (Vertical Pod Autoscaler)

HPA와 달리 **기본 내장이 아니므로 별도 설치** 필요.

```bash
# GitHub에서 VPA 설치
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
```

설치 후 kube-system에 3개 컴포넌트가 배포된다:

| 컴포넌트 | 역할 |
|---|---|
| **Recommender** | 메트릭 수집 → 최적 CPU/Memory 값 추천 |
| **Updater** | 리소스가 범위를 벗어난 Pod 감지 → 필요 시 evict(종료) |
| **Admission Controller** | Pod 재생성 시 개입 → 추천값으로 spec 자동 수정 |

### YAML 정의

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
```

### updateMode 4가지

| 모드 | 동작 |
|---|---|
| `Off` | 추천만 (Recommender만 동작, 변경 없음) |
| `Initial` | Pod 생성 시에만 적용 (Updater 동작 안 함) |
| `Recreate` | 범위 초과 시 Pod 종료 후 새 값으로 재생성 |
| `Auto` | 현재는 Recreate와 동일, 향후 In-Place 업데이트 지원 예정 |

### 추천값 확인

```bash
kubectl describe vpa myapp-vpa
```
