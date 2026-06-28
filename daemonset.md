# DaemonSet

## 역할

클러스터의 **모든 노드에 Pod를 1개씩** 자동으로 배포하는 오브젝트.

```
노드 추가 → 해당 노드에 Pod 자동 생성
노드 제거 → 해당 노드의 Pod 자동 삭제
```

ReplicaSet과 구조는 비슷하지만, replicas 수가 아닌 **노드 수**에 맞춰 배포된다.

---

## 주요 사용 사례

| 사용 사례 | 설명 |
|---|---|
| 모니터링 에이전트 | 모든 노드에서 메트릭 수집 |
| 로그 수집기 | 모든 노드의 로그를 중앙으로 전송 |
| kube-proxy | 각 노드에서 Service 트래픽 포워딩 |
| 네트워크 플러그인 | Calico 등 CNI 에이전트 배포 |

---

## YAML 정의

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: monitoring-agent
```

> ReplicaSet과 구조가 거의 동일 — `kind: DaemonSet`이고 `replicas` 필드가 없다는 점만 다르다.

---

## 명령어

```bash
kubectl create -f daemonset.yaml
kubectl get daemonsets
kubectl describe daemonset monitoring-daemon
```

---

## 동작 원리

- **v1.12 이전**: Pod의 `nodeName` 필드를 직접 지정해 스케줄러를 우회
- **v1.12 이후**: 기본 스케줄러 + **Node Affinity** 규칙을 사용해 각 노드에 배치
