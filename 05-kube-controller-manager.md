# kube-controller-manager

## 역할

클러스터의 **다양한 컨트롤러를 하나의 프로세스로 묶어 실행**하는 컴포넌트.  
각 컨트롤러는 특정 리소스의 상태를 지속적으로 감시하고 **desired state**로 유지한다.

---

## 컨트롤러란?

> 클러스터 구성 요소의 상태를 **지속적으로 모니터링**하고, 문제 발생 시 **자동으로 조치**하는 프로세스

---

## 주요 컨트롤러

### Node Controller

노드의 상태를 모니터링하고 Pod를 재배치한다.

```
heartbeat 체크 주기   : 5초
unreachable 판정 대기  : 40초
복구 대기 시간         : 5분
복구 실패 시           → Pod 삭제 후 정상 노드에 재배치
```

### Replication Controller

ReplicaSet 내 Pod 수를 항상 desired 상태로 유지한다.  
Pod가 죽으면 새 Pod를 자동 생성한다.

### 그 외

Deployment, Service, Namespace, PersistentVolume 등 모든 Kubernetes 리소스의 동작 로직이 각각의 컨트롤러로 구현되어 있다.

---

## 설정 확인 방법

| 배포 방식 | 확인 방법 |
|---|---|
| **kubeadm** | `/etc/kubernetes/manifests/kube-controller-manager.yaml` |
| **직접 구성** | `/etc/systemd/system/kube-controller-manager.service` |
| **공통** | `ps -aux \| grep kube-controller-manager` |

---

## 주요 옵션

```bash
--node-monitor-period=5s         # 노드 상태 체크 주기
--node-monitor-grace-period=40s  # unreachable 판정 대기 시간
--pod-eviction-timeout=5m        # 노드 복구 대기 후 Pod 퇴출 시간
--controllers=*                  # 활성화할 컨트롤러 지정 (기본: 전체)
```

> 특정 컨트롤러가 동작하지 않으면 `--controllers` 옵션 확인
