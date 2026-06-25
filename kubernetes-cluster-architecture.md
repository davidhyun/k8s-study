# Kubernetes 클러스터 아키텍처

## Kubernetes의 목적

애플리케이션을 **컨테이너 형태**로 자동화된 방식으로 호스팅하여, 필요한 수의 인스턴스를 쉽게 배포하고 서비스 간 통신을 가능하게 한다.

---

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────┐
│                   Master Node                       │
│  etcd | kube-scheduler | controller-manager         │
│              kube-apiserver                         │
└──────────────────────┬──────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Worker Node  │ │ Worker Node  │ │ Worker Node  │
│ kubelet      │ │ kubelet      │ │ kubelet      │
│ kube-proxy   │ │ kube-proxy   │ │ kube-proxy   │
│ [containers] │ │ [containers] │ │ [containers] │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## 구성 요소

### Master Node (Control Plane)

클러스터 전체를 관리·모니터링하는 노드

| 구성 요소 | 역할 |
|---|---|
| **etcd** | 클러스터의 모든 상태 정보를 저장하는 Key-Value 데이터베이스 |
| **kube-scheduler** | 컨테이너를 어느 노드에 배치할지 결정 (노드 용량, 정책, 제약 조건 고려) |
| **Controller Manager** | 클러스터 상태를 감시하고 원하는 상태로 유지 |
| **kube-apiserver** | 클러스터의 중앙 관리 허브. 모든 구성 요소 간 통신의 진입점 |

#### Controller Manager 세부 컨트롤러

- **Node Controller** — 노드의 상태 모니터링, 신규 노드 온보딩, 장애 노드 처리
- **Replication Controller** — 지정된 수의 컨테이너(Pod)가 항상 실행되도록 보장

---

### Worker Node

실제 컨테이너(애플리케이션)가 실행되는 노드

| 구성 요소 | 역할 |
|---|---|
| **Container Runtime Engine** | 컨테이너를 실행하는 엔진 (Docker, containerd, rkt 등) |
| **kubelet** | kube-apiserver의 지시를 받아 컨테이너를 배포·삭제하고 상태를 보고 |
| **kube-proxy** | 노드 간 네트워크 통신 규칙을 관리하여 서비스 간 연결을 지원 |

---

## 통신 흐름

```
외부 사용자 / kubectl
        │
        ▼
  kube-apiserver          ← 모든 통신의 중심
   ├── etcd               ← 상태 저장/조회
   ├── kube-scheduler     ← 배치 결정
   ├── controller-manager ← 상태 유지
   └── kubelet (각 노드)  ← 실제 컨테이너 실행
```
