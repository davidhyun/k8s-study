# Troubleshooting

## 1. Application Failures

### 기본 접근법

문제를 분석하기 전에 애플리케이션 구성을 도식화한다.  
각 오브젝트와 연결 지점을 순서대로 확인하며 원인을 찾는다.

```
사용자
  → Web Service
    → Web Pod
      → DB Service
        → DB Pod
```

### 확인 순서

#### 1. 웹 서비스 접근 여부

```bash
curl http://<node-ip>:<node-port>
```

#### 2. Service → Pod 연결 확인

Service가 Pod를 endpoint로 발견했는지 확인한다.

```bash
kubectl describe service web-service
# Endpoints 항목이 비어 있으면 selector 불일치
```

Service의 `selector`와 Pod의 `labels`가 일치하는지 비교한다.

#### 3. Pod 상태 확인

```bash
kubectl get pod web-pod
kubectl describe pod web-pod    # Events 확인
```

- `STATUS`: Running이 아니면 문제 있음
- `RESTARTS`: 횟수가 높으면 반복 크래시 중

#### 4. Pod 로그 확인

```bash
kubectl logs web-pod            # 현재 로그
kubectl logs web-pod -f         # 실시간 스트리밍 (재현 대기)
kubectl logs web-pod --previous # 이전 컨테이너 로그 (크래시 후 재시작된 경우)
```

> Pod가 재시작된 경우 현재 로그에는 이전 실패 원인이 없다.  
> `--previous`로 이전 컨테이너 로그를 확인한다.

#### 5. DB Service 및 DB Pod 확인

위 1~4 과정을 DB Service와 DB Pod에도 동일하게 반복한다.

```bash
kubectl describe service db-service
kubectl get pod db-pod
kubectl logs db-pod
```

---

## 2. Control Plane Failures

### 확인 순서

#### 1. 노드 상태 확인

```bash
kubectl get nodes
```

#### 2. Pod 상태 확인

```bash
kubectl get pods -A
```

#### 3. 컨트롤 플레인 컴포넌트 상태 확인

**kubeadm으로 구성한 경우** — 컨트롤 플레인이 Pod로 실행됨

```bash
kubectl get pods -n kube-system
```

**직접 서비스로 구성한 경우** — 마스터 노드에서 서비스 상태 확인

```bash
# 마스터 노드
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler

# 워커 노드
systemctl status kubelet
systemctl status kube-proxy
```

#### 4. 로그 확인

**kubeadm (Pod로 실행)**

```bash
kubectl logs kube-apiserver-master -n kube-system
kubectl logs kube-controller-manager-master -n kube-system
kubectl logs kube-scheduler-master -n kube-system
```

**직접 서비스로 구성**

```bash
journalctl -u kube-apiserver
journalctl -u kube-controller-manager
journalctl -u kube-scheduler
```

---

## 3. Worker Node Failures

### 확인 순서

#### 1. 노드 상태 확인

```bash
kubectl get nodes
kubectl describe node <node-name>
```

#### 2. Node Conditions 해석

`describe node`의 Conditions 항목으로 원인을 파악한다.

| Condition | True일 때 의미 |
|---|---|
| `OutOfDisk` | 디스크 공간 부족 |
| `MemoryPressure` | 메모리 부족 |
| `DiskPressure` | 디스크 용량 낮음 |
| `PIDPressure` | 프로세스 수 과다 |
| `Ready` | 노드 정상 |

노드가 마스터와 통신이 끊기면 모든 상태가 `Unknown`으로 바뀐다.  
`LastHeartbeatTime`으로 노드가 언제 응답을 멈췄는지 확인한다.

#### 3. 노드 리소스 확인

노드에 직접 접속해 CPU / 메모리 / 디스크 여유 공간을 확인한다.

```bash
df -h      # 디스크
free -m    # 메모리
top        # CPU / 프로세스
```

#### 4. kubelet 상태 및 로그 확인

```bash
systemctl status kubelet
journalctl -u kubelet
```

#### 5. kubelet 인증서 확인

```bash
openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text
```

- 만료 여부 (`Not After`)
- 올바른 그룹 소속 여부 (`system:nodes`)
- 올바른 CA 발급 여부

---

## 4. Network Troubleshooting

네트워크 문제를 진단하기 전에 클러스터 내 핵심 네트워킹 구성 요소를 파악한다.

```
Kubernetes Cluster
├── Control Plane
│   └── CoreDNS          — 클러스터 DNS (서비스 이름 → IP 변환)
└── Worker Node
    ├── kube-proxy        — iptables 규칙 관리 (Service → Pod 라우팅)
    ├── kubelet           — Pod 생명주기 관리
    ├── Pod A / Pod B     — 실제 워크로드
    └── CNI Plugin        — Pod 네트워크 설정 및 IP 할당
```

| 구성 요소 | 역할 |
|---|---|
| **Pods** | 앱 실행 단위 (App building blocks) |
| **Services** | Pod에 대한 안정적인 접근점 (Stable access points) |
| **CoreDNS** | 클러스터 내부 DNS (Cluster phonebook) |
| **CNI** | Pod 네트워크 설정 및 IP 할당 (Network setup & IPs) |
| **kube-proxy** | Service → Pod iptables 규칙 관리 (Rule management) |
