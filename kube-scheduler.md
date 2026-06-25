# kube-scheduler

## 역할

**어떤 Pod를 어떤 노드에 배치할지 결정**하는 컴포넌트.  
단, 실제로 Pod를 노드에 올리는 것은 **kubelet**의 역할이다.

---

## 스케줄링 2단계

```
Pod 배치 요청
     │
     ▼
1단계: Filter (필터링)
     리소스(CPU/Memory)가 부족한 노드 제외
     │
     ▼
2단계: Score (순위 결정)
     남은 노드에 0~10점 우선순위 부여
     → 배치 후 여유 리소스가 가장 많은 노드가 높은 점수
     │
     ▼
최고 점수 노드에 배치 결정
```

---

## 스케줄링 기준 (심화)

기본 리소스 외에도 아래 기준으로 배치를 제어할 수 있다.

- Resource Requirements & Limits
- Taints & Tolerations
- Node Selectors / Affinity Rules
- 커스텀 스케줄러 작성 가능

---

## 수동 스케줄링 (스케줄러 없을 때)

### 동작 원리

```
Pod 생성 시 nodeName 필드는 기본적으로 미설정 상태
      │
      ▼
스케줄러가 nodeName 없는 Pod를 탐색
      │
      ▼
스케줄링 알고리즘으로 적합한 노드 결정
      │
      ▼
Binding 객체 생성 → Pod의 nodeName을 해당 노드로 설정
```

> 스케줄러가 없으면 Pod는 계속 **Pending** 상태로 남는다.

### 방법 1: Pod 생성 시 nodeName 직접 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx
      image: nginx
  nodeName: node02   # 생성 시에만 지정 가능
```

### 방법 2: 이미 생성된 Pod에 Binding 객체로 할당

`nodeName`은 생성 후 직접 수정 불가 → Binding 객체를 만들어 `pods/binding` API에 POST 요청 (스케줄러의 동작을 그대로 모방)

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: myapp-pod
target:
  apiVersion: v1
  kind: Node
  name: node02
```

```bash
# YAML을 JSON으로 변환 후 POST 요청 필요
curl --header "Content-Type:application/json" --request POST \
  --data '{...}' \
  http://<api-server>/api/v1/namespaces/default/pods/myapp-pod/binding
```

---

## 설정 확인 방법

| 배포 방식 | 확인 방법 |
|---|---|
| **kubeadm** | `/etc/kubernetes/manifests/kube-scheduler.yaml` |
| **공통** | `ps -aux \| grep kube-scheduler` |
