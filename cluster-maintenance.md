# Cluster Maintenance

## 노드 다운 시 Kubernetes 동작

- 노드가 다운되면 해당 노드의 Pod에 접근 불가
- **5분(pod eviction timeout) 이내** 복구 시 → Pod 그대로 재시작
- **5분 초과** 시 → Pod를 dead로 간주, ReplicaSet Pod는 다른 노드에 재생성, 단독 Pod는 삭제

> pod eviction timeout은 controller-manager의 기본값 5분. 내부적으로 NoExecute taint를 노드에 적용해 Pod를 evict하는 방식으로 동작한다.

---

## 명령어 3가지

| 명령어 | 동작 | 기존 Pod |
|---|---|---|
| `drain` | 노드를 unschedulable로 표시 + 기존 Pod 이동 | 다른 노드로 재생성 |
| `cordon` | 노드를 unschedulable로 표시만 | 그대로 유지 |
| `uncordon` | unschedulable 해제 | 자동으로 돌아오지 않음 |

---

## 사용법

```bash
# 노드 비우기 (유지보수 전)
kubectl drain node01 --ignore-daemonsets

# 노드를 스케줄 불가 상태로만 표시 (Pod 이동 없음)
kubectl cordon node01

# 유지보수 완료 후 스케줄 재개
kubectl uncordon node01
```

> `drain` 후 노드가 복구되면 unschedulable 상태로 돌아온다 — 반드시 `uncordon` 실행 필요.  
> `uncordon` 해도 이미 다른 노드로 이동한 Pod는 자동으로 돌아오지 않는다.

---

## 클러스터 버전 업그레이드

### 컴포넌트 버전 호환 규칙

kube-apiserver 버전을 X라 할 때:

| 컴포넌트 | 허용 버전 |
|---|---|
| kube-apiserver | X |
| controller-manager, kube-scheduler | X-1 이하 |
| kubelet, kube-proxy | X-2 이하 |
| kubectl | X-1 ~ X+1 |

> 어떤 컴포넌트도 kube-apiserver보다 높은 버전일 수 없다.

### 업그레이드 원칙

- **한 번에 한 minor 버전씩** 업그레이드 (1.10 → 1.11 → 1.12)
- Kubernetes는 최근 **3개 minor 버전**만 공식 지원
- 업그레이드 순서: **마스터 노드 먼저 → 워커 노드**

### 워커 노드 업그레이드 전략

| 전략 | 특징 |
|---|---|
| 전체 동시 업그레이드 | 다운타임 발생 |
| 하나씩 순차 업그레이드 | 무중단, 시간 소요 |
| 새 노드 추가 후 구 노드 제거 | 클라우드 환경에서 권장 |

### kubeadm 업그레이드 절차 (1.28 → 1.29 예시)

> 참고: [kubeadm upgrade](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) / [upgrading linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)

업그레이드 순서: **Master Node → Worker Node 01 → Worker Node 02 → Worker Node 03**

---

#### [사전 작업] 패키지 저장소 업데이트 & 버전 확인

각 노드를 업그레이드하기 **직전**, 해당 노드에서 실행한다. (마스터 먼저, 이후 각 워커 노드 차례마다 반복)

```bash
# 저장소 파일의 버전 번호를 업그레이드 대상으로 수정 (예: v1.28 → v1.29)
vim /etc/apt/sources.list.d/kubernetes.list
# deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /

sudo apt-get update

# 설치 가능한 버전 확인
apt-cache madison kubeadm
# 출력 예: 1.29.3-1.1  ← 이 버전을 사용
```

---

#### [마스터 노드] 컨트롤 플레인 업그레이드

```bash
# 1. kubeadm 업그레이드
sudo apt-get install -y --allow-change-held-packages kubeadm=1.29.3-1.1
kubeadm version  # 버전 확인

# 2. 업그레이드 계획 확인 (버전 명시)
sudo kubeadm upgrade plan v1.29.3

# 3. 컨트롤 플레인 컴포넌트 업그레이드
sudo kubeadm upgrade apply v1.29.3

# 4. 마스터 노드 drain
kubectl drain controlplane --ignore-daemonsets

# 5. kubelet, kubectl 업그레이드
sudo apt-get install -y --allow-change-held-packages kubelet=1.29.3-1.1 kubectl=1.29.3-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. uncordon
kubectl uncordon controlplane

# 확인
kubectl get nodes
```

---

#### [워커 노드] node01 업그레이드

```bash
# node01에서 실행: kubeadm 업그레이드
sudo apt-get install -y --allow-change-held-packages kubeadm=1.29.3-1.1
sudo kubeadm upgrade node

# 마스터에서 실행: node01 drain
kubectl drain node01 --ignore-daemonsets

# node01에서 실행: kubelet 업그레이드
sudo apt-get install -y --allow-change-held-packages kubelet=1.29.3-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 마스터에서 실행: uncordon
kubectl uncordon node01
```

#### [워커 노드] node02 업그레이드

```bash
# node02에서 실행: kubeadm 업그레이드
sudo apt-get install -y --allow-change-held-packages kubeadm=1.29.3-1.1
sudo kubeadm upgrade node

# 마스터에서 실행: node02 drain
kubectl drain node02 --ignore-daemonsets

# node02에서 실행: kubelet 업그레이드
sudo apt-get install -y --allow-change-held-packages kubelet=1.29.3-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 마스터에서 실행: uncordon
kubectl uncordon node02
```

#### [워커 노드] node03 업그레이드

```bash
# node03에서 실행: kubeadm 업그레이드
sudo apt-get install -y --allow-change-held-packages kubeadm=1.29.3-1.1
sudo kubeadm upgrade node

# 마스터에서 실행: node03 drain
kubectl drain node03 --ignore-daemonsets

# node03에서 실행: kubelet 업그레이드
sudo apt-get install -y --allow-change-held-packages kubelet=1.29.3-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 마스터에서 실행: uncordon
kubectl uncordon node03
```

---

> `kubectl get nodes`의 버전은 API 서버가 아닌 **각 노드의 kubelet 버전**을 표시한다.  
> kubeadm은 kubelet을 자동 업그레이드하지 않으므로 반드시 수동으로 업그레이드해야 한다.  
> 마스터는 `kubeadm upgrade apply`, 워커는 `kubeadm upgrade node`로 명령어가 다르다.
