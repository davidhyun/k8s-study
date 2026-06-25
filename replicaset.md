# ReplicaSet / Replication Controller

## 역할

- 지정한 수의 Pod가 **항상 실행 중**임을 보장 (고가용성)
- Pod 장애 시 **자동으로 새 Pod 생성**
- 여러 노드에 걸쳐 **로드 밸런싱 및 스케일링** 지원

> **Replication Controller** → 구버전 (deprecated)  
> **ReplicaSet** → 현재 권장 방식

---

## YAML 비교

### Replication Controller

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
spec:
  replicas: 3
  template:          # Pod 정의를 그대로 중첩 (apiVersion, kind 제외)
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx
```

### ReplicaSet

```yaml
apiVersion: apps/v1   # v1이 아닌 apps/v1 주의
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:           # ReplicaSet에서 필수 (selector)
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx
```

---

## selector가 필요한 이유

ReplicaSet은 **자신이 생성하지 않은 기존 Pod도 관리**할 수 있다.  
`selector.matchLabels`로 어떤 Pod를 관리할지 명시적으로 지정한다.

> Pod가 이미 3개 존재해도 `template`은 반드시 작성해야 한다.  
> 향후 Pod 장애 시 재생성에 사용되기 때문이다.

---

## 스케일링 방법

```bash
# 방법 1: 파일 수정 후 적용 (파일에 영구 반영)
# replicas 값을 6으로 수정 후
kubectl replace -f replicaset-definition.yaml

# 방법 2: 명령어로 즉시 변경 (파일에는 반영 안 됨)
kubectl scale --replicas=6 -f replicaset-definition.yaml
kubectl scale --replicas=6 replicaset myapp-rs
```

---

## 주요 명령어

```bash
kubectl create -f replicaset-definition.yaml   # 생성
kubectl get replicaset                         # 목록 조회
kubectl delete replicaset myapp-rs             # 삭제 (하위 Pod도 함께 삭제)
kubectl replace -f replicaset-definition.yaml  # 업데이트
kubectl scale --replicas=6 replicaset myapp-rs # 스케일 조정
```
