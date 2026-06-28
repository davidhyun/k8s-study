# Priority Classes

## 역할

Pod 간 **우선순위**를 정의해 고우선순위 워크로드가 항상 먼저 스케줄되도록 보장한다.  
리소스가 부족하면 스케줄러가 낮은 우선순위 Pod를 종료하고 높은 우선순위 Pod를 배치할 수 있다.

> 네임스페이스에 종속되지 않는 클러스터 전역 오브젝트다.

---

## 우선순위 범위

| 범위 | 용도 |
|---|---|
| `-2,147,483,648` ~ `1,000,000,000` | 일반 애플리케이션 워크로드 |
| `~2,000,000,000` | 시스템 내부 (컨트롤 플레인 컴포넌트 전용) |

숫자가 클수록 높은 우선순위. 기본 제공 클래스:

```bash
kubectl get priorityclass
# system-cluster-critical  ~2,000,000,000
# system-node-critical      ~2,000,001,000
```

---

## YAML 정의

### PriorityClass 생성

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false   # true로 설정하면 priorityClassName 미지정 Pod의 기본값
description: "Critical application priority"
```

> `globalDefault: true`는 클러스터 전체에서 **단 하나의 PriorityClass**에만 설정 가능.

### Pod에 적용

```yaml
spec:
  priorityClassName: high-priority
  containers:
    - name: app
      image: app
```

> `priorityClassName` 미지정 시 기본 우선순위는 `0`.

---

## Preemption Policy (선점 정책)

리소스가 부족할 때 높은 우선순위 Pod가 **기존 Pod를 종료**할지 여부를 결정한다.

| preemptionPolicy | 동작 |
|---|---|
| `PreemptLowerPriority` (기본값) | 낮은 우선순위 Pod를 종료하고 자리를 차지 |
| `Never` | 기존 Pod를 종료하지 않고 대기 (단, 대기 중인 Pod끼리는 우선순위 순서 유지) |

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-priority
value: 500000
preemptionPolicy: Never   # 선점하지 않고 대기
```
