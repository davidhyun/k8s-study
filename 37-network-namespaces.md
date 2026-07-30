# Network Namespaces

## 개념

컨테이너가 호스트 및 다른 컨테이너로부터 네트워크를 격리하기 위해 사용하는 Linux 기능.

```
호스트 (집 전체)
├── 네임스페이스 A (방 1) ← 자신의 인터페이스/라우팅/ARP 테이블만 보임
├── 네임스페이스 B (방 2)
└── 호스트는 모든 네임스페이스를 볼 수 있음
```

- 컨테이너 내부에서 `ps`를 실행하면 PID 1만 보임
- 호스트에서 실행하면 컨테이너 프로세스가 다른 PID로 보임

---

## 기본 명령어

```bash
# 네임스페이스 생성
ip netns add red
ip netns add blue

# 네임스페이스 목록
ip netns

# 네임스페이스 내부에서 명령 실행
ip netns exec red ip link
ip -n red link          # 동일한 명령 (ip 명령에서만 사용 가능)

# 네임스페이스 내부 ARP/라우팅 테이블 확인
ip netns exec red arp
ip netns exec red route
```

---

## 네임스페이스 간 연결 (veth pair)

두 네임스페이스를 가상 케이블(veth pair)로 직접 연결한다.

```
[red ns] veth-red ──────── veth-blue [blue ns]
```

```bash
# veth pair 생성
ip link add veth-red type veth peer name veth-blue

# 각 인터페이스를 네임스페이스에 연결
ip link set veth-red netns red
ip link set veth-blue netns blue

# IP 주소 할당
ip -n red addr add 192.168.15.1/24 dev veth-red
ip -n blue addr add 192.168.15.2/24 dev veth-blue

# 인터페이스 활성화
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

---

## 여러 네임스페이스 연결 (Linux Bridge)

네임스페이스가 많을 경우 가상 스위치(Bridge)로 연결한다.

```
[red ns] ──── veth-red-br ──┐
[blue ns] ─── veth-blue-br ─┤── [v-net-0 bridge] ── 호스트
[ns3] ──────────────────────┘
```

```bash
# 브리지 생성 및 활성화
ip link add v-net-0 type bridge
ip link set dev v-net-0 up

# 기존 veth pair 삭제 (한쪽만 삭제해도 pair 전체 삭제)
ip link delete veth-red

# 네임스페이스 ↔ 브리지 연결용 veth pair 생성
ip link add veth-red type veth peer name veth-red-br
ip link add veth-blue type veth peer name veth-blue-br

# 각 end를 네임스페이스와 브리지에 연결
ip link set veth-red netns red
ip link set veth-red-br master v-net-0
ip link set veth-blue netns blue
ip link set veth-blue-br master v-net-0

# IP 할당 및 활성화
ip -n red addr add 192.168.15.1/24 dev veth-red
ip -n blue addr add 192.168.15.2/24 dev veth-blue
ip -n red link set veth-red up
ip -n blue link set veth-blue up
ip link set veth-red-br up
ip link set veth-blue-br up
```

### 호스트 ↔ 네임스페이스 연결

브리지 인터페이스에 IP를 부여하면 호스트에서 네임스페이스로 접근 가능하다.

```bash
ip addr add 192.168.15.5/24 dev v-net-0
ping 192.168.15.1   # 호스트 → red ns 접근 가능
```

---

## 외부 네트워크 연결

네임스페이스는 기본적으로 외부와 격리되어 있다.  
외부와 통신하려면 라우팅 + NAT 설정이 필요하다.

### 1. 라우팅 추가 (네임스페이스 → LAN)

```bash
# blue 네임스페이스에서 외부 LAN 트래픽을 호스트(브리지)로 라우팅
ip -n blue route add 192.168.1.0/24 via 192.168.15.5

# 인터넷까지 접근하려면 default gateway 추가
ip -n blue route add default via 192.168.15.5
```

### 2. NAT 설정 (호스트에서)

외부에서 응답이 돌아올 수 있도록 호스트 IP로 주소를 변환한다.

```bash
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```

### 3. 외부 → 네임스페이스 포트 포워딩

외부에서 네임스페이스 내부 서비스에 접근하려면 포트 포워딩을 설정한다.

```bash
# 호스트 포트 80 → blue ns(192.168.15.2) 포트 80으로 포워딩
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.15.2:80
```

---

## 전체 흐름 요약

```
네임스페이스 내부
  → 브리지(v-net-0) 경유
    → 호스트 eth0 (NAT)
      → 외부 LAN / 인터넷
```

> 외부에서 네임스페이스로의 접근은 포트 포워딩 또는 라우팅 추가로 가능하다.

---

## 트러블슈팅

### ping이 안 될 때 확인 사항

**1. IP 설정 시 서브넷 마스크 누락**

```bash
# ❌ 잘못된 설정
ip -n red addr add 192.168.15.1 dev veth-red

# ✅ 올바른 설정 (/24 필수)
ip -n red addr add 192.168.15.1/24 dev veth-red
```

**2. iptables/FirewallD가 트래픽 차단**

```bash
# 허용 규칙 추가
iptables -A FORWARD -i v-net-0 -j ACCEPT

# 또는 학습 환경에서만 iptables 비활성화
iptables -F
```

> 프로덕션 환경에서 iptables를 비활성화하면 안 된다. 학습 환경에서만 사용할 것.
