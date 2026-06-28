# Static Pods

## 왜 알아야 하나?

`kubectl get pods -n kube-system`으로 보이는 `kube-apiserver`, `etcd` 등의 컨트롤 플레인 Pod들이 바로 static pod다.
장애 발생 시 어디서 설정을 찾아야 하는지, 왜 `kubectl delete`로 삭제가 안 되는지 이해하려면 static pod 개념이 필수다.

---

## 개념

kube-apiserver 없이 **kubelet이 단독으로 생성·관리하는 Pod**.  
지정된 디렉토리에 Pod 정의 파일을 두면 kubelet이 자동으로 생성하고, 크래시 시 재시작한다.

```
manifest 디렉토리에 YAML 추가 → kubelet이 Pod 생성
파일 수정 → Pod 재생성
파일 삭제 → Pod 자동 삭제
```

> Pod만 생성 가능. ReplicaSet·Deployment·Service 등은 불가 (kubelet은 Pod 단위만 이해함)

---

## manifest 디렉토리 설정

kubelet 실행 옵션 또는 config 파일로 지정한다.

```bash
# 방법 1: kubelet 서비스 옵션 직접 지정
--pod-manifest-path=/etc/kubernetes/manifests

# 방법 2: config 파일 경로 지정 (kubeadm 방식)
--config=kubeconfig.yaml
# kubeconfig.yaml 내부:
# staticPodPath: /etc/kubernetes/manifests
```

### 경로 확인 순서

```
1. kubelet 서비스 파일에서 --pod-manifest-path 확인
2. 없으면 --config 옵션으로 지정된 파일 확인
3. 해당 파일 내 staticPodPath 값 확인
```

---

## 클러스터에 참여한 경우

kubelet이 static pod를 생성하면 kube-apiserver에 **읽기 전용 mirror 객체**도 함께 생성된다.

```
kubectl get pods 로 조회 가능 → 단, 편집/삭제 불가
삭제하려면 manifest 디렉토리의 파일을 직접 제거해야 함
```

> Pod 이름에 노드 이름이 자동으로 붙는다. 예: `kube-apiserver-controlplane`

---

## 주요 사용 사례: 컨트롤 플레인 컴포넌트 배포

kubeadm이 이 방식으로 컨트롤 플레인을 구성한다.

```
/etc/kubernetes/manifests/
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
├── kube-scheduler.yaml
└── etcd.yaml
```

- kubelet이 컨트롤 플레인 컴포넌트를 Pod로 띄우고 장애 시 자동 재시작
- 바이너리 설치·서비스 설정 불필요

> `kubectl get pods -n kube-system`으로 조회되는 컨트롤 플레인 Pod들이 바로 static pod다.

---

## Static Pod 식별 방법

1. Pod 이름 끝에 노드 이름이 붙어 있음 → `kube-apiserver-controlplane`, `etcd-controlplane`
2. `ownerReferences` 확인으로 확실히 구분 가능

```bash
kubectl get pod <name> -n kube-system -o yaml | grep -A3 ownerReferences
```

```
# Static Pod: kind가 Node
ownerReferences:
  - kind: Node
    name: controlplane

# 일반 Pod: kind가 ReplicaSet 등
ownerReferences:
  - kind: ReplicaSet
```

---

## 다른 노드의 Static Pod 삭제

`kubectl delete pod`로 삭제해도 kubelet이 즉시 재생성한다.  
반드시 **해당 노드에 SSH 접속 후 manifest 파일을 직접 삭제**해야 한다.

```bash
# 1. 해당 노드로 SSH 접속
ssh node01

# 2. 그 노드의 staticPodPath 확인 (노드마다 다를 수 있음)
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# 3. 해당 경로에서 파일 삭제
rm /path/to/manifests/<pod-name>.yaml
```

> `staticPodPath`가 노드마다 다를 수 있으므로 삭제 전 반드시 경로를 먼저 확인할 것.

---

## Static Pods vs DaemonSet

| | Static Pods | DaemonSet |
|---|---|---|
| 생성 주체 | kubelet 단독 | DaemonSet 컨트롤러 (kube-apiserver 경유) |
| kube-scheduler 관여 | 없음 | 없음 |
| 주요 용도 | 컨트롤 플레인 컴포넌트 배포 | 모니터링·로그 에이전트 등 |
| kubectl로 삭제 | 불가 (파일 직접 삭제) | 가능 |
