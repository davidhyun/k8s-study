# Network - Docker

## 네트워크 옵션 3가지

| 옵션 | 설명 |
|---|---|
| **none** | 네트워크 없음 — 외부/다른 컨테이너와 완전 격리 |
| **host** | 호스트 네트워크 공유 — 포트 매핑 불필요, 동일 포트 중복 불가 |
| **bridge** | 내부 가상 네트워크 생성 — 기본값, 가장 일반적 |

```bash
docker run --network=none nginx
docker run --network=host nginx
docker run nginx   # 기본값: bridge
```

---

## Bridge 네트워크 동작 방식

Docker 설치 시 `bridge` 네트워크(호스트에서는 `docker0` 인터페이스)가 자동 생성된다.

```bash
docker network ls          # bridge 네트워크 확인
ip link                    # docker0 인터페이스 확인
ip addr show docker0       # docker0 IP 확인 (172.17.0.1)
```

### 컨테이너 생성 시 내부 동작

```
컨테이너 생성
  → Docker가 네트워크 네임스페이스 생성
  → veth pair 생성 (한쪽은 컨테이너, 한쪽은 docker0 브리지에 연결)
  → 컨테이너에 IP 할당 (172.17.0.x)
```

```bash
# 네임스페이스 목록 확인
ip netns

# 컨테이너의 네임스페이스 확인
docker inspect <container>

# 호스트 측 인터페이스 확인
ip link

# 컨테이너 측 인터페이스 확인
ip -n <namespace> link
ip -n <namespace> addr
```

> veth pair는 번호로 식별 — 9↔10, 7↔8 처럼 홀짝 쌍으로 연결됨

---

## 포트 매핑

컨테이너는 내부 네트워크에 있어 외부에서 직접 접근 불가.  
외부 접근을 허용하려면 포트 매핑을 설정한다.

```bash
# 호스트 8080 → 컨테이너 80으로 매핑
docker run -p 8080:80 nginx
```

### 내부 동작 (iptables NAT)

Docker는 iptables PREROUTING 규칙으로 포트 포워딩을 구현한다.

```bash
# Docker가 자동 생성하는 iptables 규칙 확인
iptables -t nat -L -n

# 수동으로 동일한 규칙 추가하는 경우
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.3:80
```

---

## 전체 흐름 요약

```
외부 요청 (host:8080)
  → iptables PREROUTING (NAT)
    → docker0 브리지
      → veth pair
        → 컨테이너 네임스페이스 (172.17.0.x:80)
```
