# Labels, Selectors, Annotations

## Labels & Selectors

- **Labels**: 오브젝트에 부착하는 속성 (key-value)
- **Selectors**: 라벨을 기준으로 오브젝트를 **필터링**하는 조건

```
Label    : 각 객체에 "app=app1", "function=frontend" 같은 태그를 붙임
Selector : "app=app1" 조건으로 해당 객체들만 골라냄
```

> 수백~수천 개의 오브젝트가 생긴 클러스터에서 종류/애플리케이션/기능별로 그룹화·검색하기 위해 사용한다.

---

## Pod에 Label 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: app1
    function: frontend
spec:
  containers:
    - name: nginx
      image: nginx
```

```bash
# 라벨로 Pod 필터링
kubectl get pods --selector app=app1

# 특정 라벨을 가진 모든 종류의 오브젝트 조회 (Pod, ReplicaSet, Deployment, Service 등)
kubectl get all --selector env=prod
```

---

## Kubernetes 내부에서의 활용 (ReplicaSet 예시)

ReplicaSet은 `selector`로 자신이 관리할 Pod를 찾는다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  labels:            # ① ReplicaSet 자신의 라벨 (상위 오브젝트가 RS를 찾을 때 사용)
    app: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1      # ③ 이 라벨과 일치하는 Pod를 관리 대상으로 선택
  template:
    metadata:
      labels:
        app: app1    # ② Pod에 실제로 붙는 라벨
    spec:
      containers:
        - name: nginx
          image: nginx
```

> **초보자가 흔히 헷갈리는 부분**: 최상위 `metadata.labels`(① ReplicaSet 자체 라벨)와 `template.metadata.labels`(② Pod 라벨)는 서로 다른 객체에 붙는 라벨이다.  
> `spec.selector`(③)는 **②와 일치**해야 ReplicaSet이 Pod를 인식한다.

- 라벨이 여러 종류 있는 경우, 동일한 라벨을 가진 다른 용도의 Pod와 혼동되지 않도록 **selector에 여러 라벨을 동시에 지정** 가능
- Service도 동일한 방식으로 `selector`를 사용해 Pod를 찾는다

---

## Annotations

라벨/셀렉터처럼 그룹화·필터링 목적이 아니라, **정보 기록용**으로 사용한다.

```yaml
metadata:
  annotations:
    buildVersion: "1.2.3"
    contact: "devops@example.com"
```

예: 툴 이름/버전/빌드 정보, 담당자 연락처 등 통합·관리 목적의 메타데이터
