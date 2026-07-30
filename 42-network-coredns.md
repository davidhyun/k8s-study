# Network - CoreDNS

## 개념

CoreDNS는 Kubernetes 클러스터 내부 DNS 서버로 사용되는 오픈소스 DNS 솔루션이다.  
클러스터 내 Pod/Service 이름을 IP로 해석하는 역할을 담당한다.

---

## 설치 (바이너리)

```bash
# 바이너리 다운로드
curl -LO https://github.com/coredns/coredns/releases/download/v1.12.4/coredns_1.12.4_linux_amd64.tgz

# 압축 해제
tar -zxf coredns_1.12.4_linux_amd64.tgz

# 실행 (기본 포트 53)
./coredns
```

---

## 설정 (Corefile)

CoreDNS는 `Corefile`에서 설정을 읽는다.

```
.:53 {
    # /etc/hosts 파일을 참조해 이름 해석
    hosts /etc/hosts {
        reload 1m      # 1분마다 hosts 파일 재로드
        fallthrough    # hosts에 없으면 다음 플러그인으로 넘김
    }

    # 매칭되지 않는 쿼리는 외부로 포워딩
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }

    cache 30   # 30초 캐싱
    log        # 쿼리 로깅
    errors     # 에러 로깅
}
```

### 동작 흐름

```
DNS 쿼리 수신
  → /etc/hosts에서 이름 검색
  → 없으면 /etc/resolv.conf의 nameserver로 포워딩
  → 결과 30초 캐싱
```

---

## CoreDNS vs kube-proxy

| 구분 | CoreDNS | kube-proxy |
|---|---|---|
| 역할 | 이름 → IP 해석 (DNS) | IP/포트 트래픽 라우팅 |
| 처리 대상 | 도메인 이름 쿼리 | 네트워크 패킷 |
| 동작 시점 | Pod가 Service 이름으로 접근할 때 | 실제 패킷이 Service IP로 전달될 때 |
| 구현 방식 | DNS 서버 (포트 53) | iptables / IPVS 규칙 |

```
Pod → "my-service" 이름으로 요청
  → CoreDNS가 ClusterIP(10.96.x.x)로 해석
  → kube-proxy가 ClusterIP → 실제 Pod IP로 트래픽 전달
```

---

## Kubernetes에서의 CoreDNS

kubeadm으로 구성된 클러스터에서 CoreDNS는 `kube-system` 네임스페이스에 Deployment로 배포된다.

```bash
kubectl get pods -n kube-system | grep coredns
kubectl get configmap coredns -n kube-system   # Corefile 확인
```

> Kubernetes용 CoreDNS 플러그인에 대한 자세한 내용:  
> https://coredns.io/plugins/kubernetes/
