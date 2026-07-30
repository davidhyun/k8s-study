# Network - DNS in Kubernetes

## 개요

Kubernetes는 클러스터 구성 시 내장 DNS 서버를 자동으로 배포한다.  
Pod/Service 간 이름 기반 통신을 위해 사용하며, 구현체는 **CoreDNS**다.

---

## Service DNS 레코드

Service가 생성되면 DNS 서버는 자동으로 레코드를 생성한다.

### 같은 네임스페이스

```
web-service
```

### 다른 네임스페이스

```
web-service.apps
```

### 전체 FQDN (Fully Qualified Domain Name)

```
<service-name>.<namespace>.svc.cluster.local
```

예시:
```
web-service.apps.svc.cluster.local
```

---

## DNS 이름 구조

```
web-service  .  apps  .  svc  .  cluster.local
     │            │        │           │
  Service명   Namespace  서비스    루트 도메인
                        서브도메인  (기본값)
```

| 구성 요소 | 설명 |
|---|---|
| `<service-name>` | Service 이름 |
| `<namespace>` | Service가 속한 네임스페이스 |
| `svc` | 서비스 전용 서브도메인 |
| `cluster.local` | 클러스터 루트 도메인 (기본값) |

---

## Pod DNS 레코드

Pod 레코드는 기본적으로 생성되지 않는다. **명시적으로 활성화** 해야 한다.

활성화 시 Pod IP의 `.`을 `-`로 변환해 hostname을 생성한다.

```
IP: 10.244.2.5  →  hostname: 10-244-2-5
```

### Pod FQDN

```
<ip-with-dashes>.<namespace>.pod.cluster.local
```

예시:
```
10-244-2-5.default.pod.cluster.local
```

---

## 네임스페이스 간 접근 요약

| 상황 | 사용 가능한 이름 |
|---|---|
| 같은 네임스페이스 | `web-service` |
| 다른 네임스페이스 | `web-service.apps` |
| 항상 사용 가능 (FQDN) | `web-service.apps.svc.cluster.local` |

> 같은 네임스페이스 내에서는 Service 이름만으로 접근 가능.  
> 네임스페이스가 다르면 반드시 네임스페이스를 포함해야 한다.

---

## CoreDNS 구현 방식

### 배포 구조

```
kube-system 네임스페이스
└── Deployment: coredns
    └── ReplicaSet (2개 Pod — 고가용성)
        └── Pod: coredns
            └── 실행: coredns 바이너리
```

```bash
kubectl get pods -n kube-system | grep coredns
kubectl get deployment -n kube-system coredns
```

### Corefile 설정 (ConfigMap)

CoreDNS 설정은 ConfigMap으로 관리된다. 수정 시 `kubectl edit`으로 변경 가능.

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure    # Pod DNS 레코드 활성화 옵션 (기본 비활성)
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf   # 외부 도메인은 노드 DNS로 포워딩
    cache 30
    loop
    reload
    loadbalance
}
```

| 플러그인 | 역할 |
|---|---|
| `kubernetes` | Kubernetes 클러스터 DNS 처리, 루트 도메인(`cluster.local`) 설정 |
| `pods insecure` | Pod IP 기반 DNS 레코드 활성화 |
| `forward` | 알 수 없는 도메인을 노드 DNS 서버로 포워딩 |
| `cache` | DNS 응답 캐싱 |

### kube-dns Service

CoreDNS Pod는 `kube-dns` Service를 통해 클러스터에 노출된다.

```bash
kubectl get svc kube-dns -n kube-system
# ClusterIP: 10.96.0.10 (기본값)
```

---

## Pod의 DNS 설정 자동화

Pod 생성 시 **kubelet**이 `/etc/resolv.conf`를 자동으로 설정한다.

```bash
# kubelet 설정에서 DNS 서버 IP 확인
cat /var/lib/kubelet/config.yaml | grep -i dns
```

Pod 내부의 `/etc/resolv.conf`:

```
nameserver 10.96.0.10          # kube-dns Service IP
search default.svc.cluster.local svc.cluster.local cluster.local
```

### search 도메인 덕분에 가능한 단축 이름

`search` 항목이 설정되어 있어 짧은 이름으로도 Service에 접근할 수 있다.

| 입력한 이름 | 실제 조회 순서 |
|---|---|
| `web-service` | `web-service.default.svc.cluster.local` |
| `web-service.default` | `web-service.default.svc.cluster.local` |
| `web-service.default.svc` | `web-service.default.svc.cluster.local` |

> `search` 도메인은 **Service에만 적용**된다.  
> Pod에 접근할 때는 반드시 FQDN 전체를 입력해야 한다.

```bash
# Service는 단축 이름으로 조회 가능
nslookup web-service
# → web-service.default.svc.cluster.local

# Pod는 FQDN 필요
nslookup 10-244-2-5.default.pod.cluster.local
```
