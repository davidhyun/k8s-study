# Network - Service

## Service란?

Pod에 직접 접근하지 않고 안정적인 엔드포인트를 제공하는 **가상 오브젝트**.

- 실제 프로세스/네임스페이스/인터페이스가 없음 — 메모리에만 존재
- 클러스터 전체에 걸쳐 동작 (특정 노드에 종속되지 않음)
- IP + Port 조합으로 트래픽을 Pod로 포워딩

---

## Service 종류

| 종류 | 접근 범위 | 설명 |
|---|---|---|
| **ClusterIP** | 클러스터 내부 | 기본값. 클러스터 내 Pod끼리 통신 |
| **NodePort** | 외부 | 모든 노드의 특정 포트를 통해 외부 접근 가능 |
| **LoadBalancer** | 외부 | 클라우드 로드밸런서 연동 |

---

## Service IP 할당

Service 생성 시 kube-apiserver의 `--service-cluster-ip-range` 옵션에서 IP를 할당한다.

```bash
# kube-apiserver 옵션 확인
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
# 예: --service-cluster-ip-range=10.96.0.0/12
```

> Pod CIDR과 Service CIDR은 **반드시 겹치지 않아야** 한다.
>
> 예) Pod CIDR: `10.244.0.0/16` / Service CIDR: `10.96.0.0/12`

---

## kube-proxy 동작 방식

Service가 생성/삭제될 때마다 kube-proxy가 각 노드에 **iptables 규칙**을 자동으로 추가/삭제한다.

```
Service 생성
  → kube-proxy가 kube-apiserver 감지
  → 모든 노드에 iptables NAT 규칙 추가
  → Service IP:Port → Pod IP:Port 로 트래픽 포워딩
```

### proxy 모드

| 모드 | 설명 |
|---|---|
| `iptables` | 기본값. iptables NAT 규칙으로 포워딩 |
| `ipvs` | L4 로드밸런싱, 대규모 클러스터에서 성능 우수 |
| `userspace` | 구식 방식, 거의 사용 안 함 |

```bash
# kube-proxy 모드 확인
kubectl logs -n kube-system <kube-proxy-pod>
```

---

## iptables 규칙 확인

```bash
# Service 이름으로 iptables 규칙 검색
iptables -t nat -L -n | grep <service-name>
```

예시 출력:
```
# 10.103.132.104:3306(Service IP) → 10.244.1.2:3306(Pod IP) 로 DNAT
DNAT  tcp  --  anywhere  10.103.132.104  tcp dpt:3306  to:10.244.1.2:3306
```

---

## 흐름 요약

```
Pod A → Service IP:Port
  → iptables NAT 규칙 (kube-proxy가 생성)
    → Pod B IP:Port
```

> NodePort의 경우 모든 노드의 해당 포트로 들어오는 트래픽도 동일한 방식으로 Pod로 포워딩된다.
