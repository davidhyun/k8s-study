# Network Basics

## 네트워크 구성 요소

### Switch (스위치)

같은 네트워크 내 호스트 간 통신을 담당한다.

```
[Host A] ──┐
           ├── [Switch] (192.168.1.0/24)
[Host B] ──┘
```

### Router (라우터)

서로 다른 네트워크를 연결한다. 각 네트워크에 IP를 하나씩 가진다.

```
[192.168.1.0/24] ── [Router] ── [192.168.2.0/24]
                     1.1 / 2.1
```

### Gateway (게이트웨이)

외부 네트워크로 나가는 문. 시스템이 모르는 네트워크로 가는 패킷을 어디로 보낼지 알려준다.

---

## 주요 명령어

```bash
# 네트워크 인터페이스 목록 확인
ip link

# IP 주소 확인
ip addr

# IP 주소 설정 (재시작 시 초기화)
ip addr add 192.168.1.10/24 dev eth0

# 라우팅 테이블 확인
route
ip route

# 라우팅 추가
ip route add 192.168.2.0/24 via 192.168.1.1

# 기본 게이트웨이 설정 (모든 외부 트래픽)
ip route add default via 192.168.1.1
# 또는
ip route add 0.0.0.0/0 via 192.168.1.1
```

> `ip addr`, `ip route add` 등의 변경은 **재시작 시 초기화**된다.  
> 영구 적용하려면 `/etc/network/interfaces` 파일에 설정해야 한다.

---

## 라우팅 테이블 이해

```
Destination     Gateway         Genmask         Iface
192.168.1.0     0.0.0.0         255.255.255.0   eth0   ← 직접 연결 (게이트웨이 불필요)
192.168.2.0     192.168.1.1     255.255.255.0   eth0   ← 라우터 경유
0.0.0.0         192.168.1.1     0.0.0.0         eth0   ← 기본 게이트웨이 (나머지 모두)
```

- `0.0.0.0` (Gateway 필드): 게이트웨이 불필요, 직접 연결된 네트워크
- `0.0.0.0/0` (Destination 필드): `default`와 동일, 모든 목적지

---

## Linux 호스트를 라우터로 사용

호스트 B가 두 네트워크를 연결하는 라우터 역할:

```
[A: 1.5] ── eth0:[B: 1.6 / 2.6]:eth1 ── [C: 2.5]
  192.168.1.0/24                192.168.2.0/24
```

### 각 호스트 라우팅 설정

```bash
# Host A: 2.0 네트워크는 B(1.6)를 통해
ip route add 192.168.2.0/24 via 192.168.1.6

# Host C: 1.0 네트워크는 B(2.6)를 통해
ip route add 192.168.1.0/24 via 192.168.2.6
```

### IP 포워딩 활성화 (Host B)

Linux는 기본적으로 인터페이스 간 패킷 전달을 차단한다.

```bash
# 현재 상태 확인 (0=비활성, 1=활성)
cat /proc/sys/net/ipv4/ip_forward

# 임시 활성화 (재시작 시 초기화)
echo 1 > /proc/sys/net/ipv4/ip_forward

# 영구 활성화
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
```

---

## DNS (Domain Name System)

### /etc/hosts — 로컬 이름 해석

```bash
# /etc/hosts
192.168.1.11    db
```

- 호스트 이름 → IP 매핑을 로컬에서 직접 지정
- 실제 호스트명과 달라도 무관 (host A는 파일 내용을 그대로 신뢰)
- 소규모 환경에서만 적합 — 서버가 많아지면 관리 불가

### /etc/resolv.conf — DNS 서버 지정

```bash
# /etc/resolv.conf
nameserver 192.168.1.100    # 내부 DNS 서버
nameserver 8.8.8.8          # 외부 DNS 서버 (Google)
search mycompany.com        # 검색 도메인 (web → web.mycompany.com으로 자동 확장)
```

### 이름 해석 우선순위

기본 순서: **로컬 hosts 파일 → DNS 서버**

```bash
# /etc/nsswitch.conf
hosts: files dns    # files = /etc/hosts, dns = nameserver
```

순서 변경 시 이 파일을 수정한다.

---

## DNS 레코드 타입

| 타입 | 설명 | 예시 |
|---|---|---|
| **A** | 호스트명 → IPv4 | `web → 192.168.1.1` |
| **AAAA** | 호스트명 → IPv6 | `web → ::1` |
| **CNAME** | 이름 → 이름 (별칭) | `eat → food.com` |

---

## 도메인 구조

```
        .                  ← root
        com                ← 최상위 도메인 (TLD)
        google             ← 도메인
        maps.google.com    ← 서브도메인
        mail.google.com
```

---

## DNS 계층 해석 과정

내부 DNS 서버가 모르는 도메인은 외부 DNS 서버들을 순차적으로 질의해 해석한다.

```
클라이언트
  → 내부 DNS 서버 (모름)
    → Root DNS (. )
      → TLD DNS (.com)
        → Google DNS (google.com)
          → IP 반환
```

### DNS 포워딩 & 캐싱

```
내부 DNS 서버가 모르는 도메인
  → 외부 공용 DNS(8.8.8.8)로 포워딩
  → 응답 결과를 일정 시간(TTL) 동안 캐시
  → 같은 요청은 캐시에서 즉시 응답
```

- 매번 외부 질의 없이 캐시로 빠르게 응답
- TTL(Time To Live): 보통 수 초 ~ 수 분

> `/etc/resolv.conf`에 `nameserver 8.8.8.8`을 추가하거나, 내부 DNS 서버 자체에 포워딩을 설정하면 외부 도메인도 해석 가능하다.

---

## DNS 조회 도구

```bash
# DNS 서버에서 IP 조회 (/etc/hosts 무시)
nslookup web.mycompany.com

# 더 상세한 DNS 조회
dig web.mycompany.com
```

> `nslookup`, `dig` 모두 `/etc/hosts`를 참조하지 않는다. DNS 서버에만 질의한다.
