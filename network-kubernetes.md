# Network - Kubernetes

## 노드 기본 요구사항

- 각 노드는 네트워크에 연결된 인터페이스 최소 1개 필요
- 각 인터페이스에 IP 주소 설정 필요
- 노드마다 **고유한 hostname** 및 **고유한 MAC 주소** 필요

> VM을 클론해서 구성하는 경우 MAC 주소 중복에 주의.

---

## 필수 오픈 포트

### 마스터 노드 (Control Plane)

| 프로토콜 | 방향 | 포트 | 컴포넌트 | 접근 주체 |
|---|---|---|---|---|
| TCP | Inbound | **6443** | kube-apiserver | 전체 |
| TCP | Inbound | **2379-2380** | etcd | kube-apiserver, etcd (peer 통신) |
| TCP | Inbound | **10250** | kubelet API | Self, 컨트롤 플레인 |
| TCP | Inbound | **10259** | kube-scheduler | Self |
| TCP | Inbound | **10257** | kube-controller-manager | Self |

### 워커 노드

| 프로토콜 | 방향 | 포트 | 컴포넌트 | 접근 주체 |
|---|---|---|---|---|
| TCP | Inbound | **10250** | kubelet API | Self, 컨트롤 플레인 |
| TCP | Inbound | **10256** | kube-proxy | Self, 로드밸런서 |
| TCP/UDP | Inbound | **30000-32767** | NodePort Services | 전체 |

> etcd는 외부 클러스터로 분리하거나 커스텀 포트로 변경 가능.  
> kube-apiserver 포트를 443으로 변경하거나, 로드밸런서가 443을 받아 6443으로 포워딩하는 방식도 일반적이다.

---

## 멀티 마스터 구성 시 추가 고려사항

- 마스터 노드 간 **2380 포트** 추가 오픈 필요 (etcd peer 통신)
- 위 포트 목록을 모든 마스터 노드에 동일하게 적용

---

## 방화벽/보안 그룹 설정

환경에 따라 아래 위치에서 포트 설정 필요:

| 환경 | 설정 위치 |
|---|---|
| 온프레미스 | iptables 또는 FirewallD |
| AWS | Security Group |
| GCP | Firewall Rules |
| Azure | Network Security Group |

> 클러스터 통신 문제 발생 시 포트 오픈 여부를 가장 먼저 확인한다.
