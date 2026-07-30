# Network - CNI (Container Network Interface)

## 등장 배경

Docker, Rocket, Kubernetes 등 모든 컨테이너 런타임이 네트워크 문제를 비슷한 방식으로 해결하고 있었다.  
중복 개발을 없애기 위해 **네트워킹 표준**을 만든 것이 CNI다.

---

## CNI란?

컨테이너 런타임 환경에서 네트워킹 문제를 해결하는 **프로그램(플러그인) 개발 표준**.

```
컨테이너 런타임 (Kubernetes, Rocket 등)
        ↓ 컨테이너 생성/삭제 시 호출
    CNI 플러그인 (bridge, flannel, calico 등)
        ↓
    네트워크 설정 (IP 할당, 라우팅 등)
```

---

## CNI 책임 분리

### 컨테이너 런타임 책임

- 컨테이너마다 네트워크 네임스페이스 생성
- 컨테이너가 연결할 네트워크 식별
- 컨테이너 생성 시 플러그인의 `add` 명령 호출
- 컨테이너 삭제 시 플러그인의 `del` 명령 호출
- JSON 파일로 네트워크 플러그인 설정

### CNI 플러그인 책임

- `add` / `del` / `check` 명령 지원
- 컨테이너 ID, 네트워크 네임스페이스를 파라미터로 수신
- Pod에 IP 주소 할당
- 컨테이너 간 통신에 필요한 라우팅 설정
- 결과를 지정된 형식으로 반환

---

## CNI 기본 내장 플러그인

| 종류 | 플러그인 |
|---|---|
| 네트워크 | bridge, vlan, ipvlan, macvlan |
| IPAM (IP 관리) | host-local, dhcp |
| Windows | win-bridge, win-overlay |

## 서드파티 플러그인

Weave, Flannel, Cilium, Calico, VMware NSX, Infoblox 등

---

## Docker와 CNI

Docker는 CNI가 아닌 자체 표준인 **CNM(Container Network Model)** 을 사용한다.  
따라서 CNI 플러그인을 Docker에 직접 지정할 수 없다.

### Kubernetes의 해결 방법

```
1. Docker 컨테이너를 --network=none으로 생성 (네트워크 없이)
2. Kubernetes가 직접 CNI 플러그인을 호출해 네트워크 설정
```

> CNI를 구현한 런타임(Kubernetes, Rocket 등)은 어떤 CNI 플러그인과도 함께 사용할 수 있다.

---

## 주요 CNI 플러그인 비교

| CNI | NetworkPolicy | 기반 기술 | 특징 |
|---|---|---|---|
| **Flannel** | ❌ | VXLAN 오버레이 | 설정 단순, 가벼움 — 학습/소규모 환경 |
| **Calico** | ✅ | BGP 라우팅 | 오버레이 없이 동작 가능, 대규모 온프레미스 |
| **Cilium** | ✅ | eBPF | 고성능, L7 정책 지원, 최근 가장 주목받는 CNI |
| **Weave** | ✅ | 오버레이 | 멀티클러스터 지원, 암호화 내장, 상대적으로 무거움 |

### Flannel

- k3s 기본 CNI
- **NetworkPolicy 미지원** — 세밀한 네트워크 제어 불가
- 학습/소규모/단순한 환경에 적합

#### Flannel 삭제

```bash
kubectl delete daemonset -n kube-flannel kube-flannel-ds
kubectl delete cm kube-flannel-cfg -n kube-flannel
rm /etc/cni/net.d/10-flannel.conflist
```

### Calico

- 프로덕션 온프레미스에서 가장 널리 사용
- BGP 기반 라우팅으로 오버레이 없이 동작 → 성능 우수
- NetworkPolicy를 가장 강력하게 지원
- Flannel보다 설정이 복잡

#### Calico 설치 방법

```bash
# 1. Calico CRD 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/operator-crds.yaml

# 2. Calico Operator 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/tigera-operator.yaml

# 3. custom-resources.yaml 다운로드
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/custom-resources.yaml -O

# 4. Pod CIDR 수정 (클러스터 환경에 맞게)
vi custom-resources.yaml

# 5. 적용
kubectl apply -f custom-resources.yaml

# 6. Pod 상태 확인
watch kubectl get pods -A
```

`custom-resources.yaml` 수정 예시:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 172.17.0.0/16       # 클러스터 Pod CIDR에 맞게 변경
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

> `cidr`는 kubeadm 초기화 시 설정한 `--pod-network-cidr` 값과 일치해야 한다.

### Cilium

- eBPF 기반으로 커널 레벨에서 동작 → 고성능
- L7(HTTP, gRPC) 수준의 네트워크 정책 지원
- 최신 클라우드 네이티브 환경에서 빠르게 채택 중
- 상대적으로 러닝커브 높음

### Weave

- 멀티클러스터 환경 지원
- 트래픽 암호화 내장
- 다른 CNI에 비해 상대적으로 무거움

---

> **선택 기준 요약**  
> 학습/소규모 → Flannel  
> 온프레미스 프로덕션 → Calico  
> 고성능/최신 클라우드 환경 → Cilium

---

## Kubernetes에서 CNI 설정

CNI 플러그인을 호출하는 주체는 **컨테이너 런타임(containerd, cri-o)** 이다.

### 주요 경로

| 경로 | 설명 |
|---|---|
| `/opt/cni/bin/` | CNI 플러그인 바이너리 (bridge, flannel, dhcp 등) |
| `/etc/cni/net.d/` | CNI 설정 파일 — 런타임이 이 디렉토리를 읽어 플러그인 결정 |

> 설정 파일이 여러 개면 **알파벳 순서로 첫 번째 파일**을 사용한다.

### 설정 파일 예시 (bridge)

```json
{
  "name": "mynet",
  "type": "bridge",
  "isGateway": true,
  "ipMasquerade": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16"
  }
}
```

| 필드 | 설명 |
|---|---|
| `type` | 사용할 CNI 플러그인 이름 |
| `isGateway` | 브리지 인터페이스에 IP 부여 (게이트웨이 역할) |
| `ipMasquerade` | NAT IP 마스커레이딩 활성화 |
| `ipam.type` | IP 관리 방식 (`host-local`: 로컬 관리 / `dhcp`: 외부 DHCP) |
| `ipam.subnet` | Pod에 할당할 IP 대역 |

### 트러블슈팅 시 확인 포인트

```bash
ls /opt/cni/bin/          # 플러그인 바이너리 존재 여부
ls /etc/cni/net.d/        # 설정 파일 존재 여부
cat /etc/cni/net.d/*.conf # 설정 내용 확인
```
