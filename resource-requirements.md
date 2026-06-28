# Resource Requirements

## 개념

Pod가 노드에 배치될 때 필요한 **CPU/메모리 자원**을 명시하는 방법.  
스케줄러는 이 값을 기준으로 충분한 리소스가 있는 노드를 선택한다.

```
Pod 배치 요청 → 스케줄러가 requests 확인 → 리소스 충분한 노드 선택
                                          → 없으면 Pod는 Pending 상태
```

---

## Requests & Limits

| 구분 | 의미 |
|---|---|
| **requests** | 스케줄링 기준 — 이 만큼은 보장된다 |
| **limits** | 사용 상한선 — 이 이상은 못 쓴다 |

```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "512Mi"
    cpu: "1"
```

### CPU 단위

| 표기 | 의미 |
|---|---|
| `1` | 1 vCPU (AWS), 1 Core (GCP/Azure), 1 Hyperthread |
| `0.1` = `100m` | 0.1 vCPU (m = milli) |
| 최솟값 | `1m` |

### Memory 단위

| 표기 | 의미 |
|---|---|
| `256Mi` | 256 Mebibyte (1Mi = 1024KB) |
| `256M` | 256 Megabyte (1M = 1000KB) |
| `1Gi` / `1G` | Gibibyte / Gigabyte |

---

## Limits 초과 시 동작

| 자원 | 초과 시 동작 |
|---|---|
| CPU | **Throttle** — 사용량을 한도로 제한 (종료 안 됨) |
| Memory | **OOMKill** — 한도 초과 지속 시 Pod 종료 (`OOMKilled`) |

---

## Requests/Limits 조합별 비교

```
① 둘 다 없음
   → 한 Pod가 노드 전체 리소스 독점 가능 (위험)

② limits만 설정
   → Kubernetes가 requests = limits로 자동 설정
   → 각 Pod는 정확히 limits만큼 보장 & 초과 불가

③ 둘 다 설정
   → requests만큼 보장, limits까지 사용 가능
   → 여유 리소스가 있어도 limits 이상은 못 씀 (비효율 가능)

④ requests만 설정 (권장)
   → requests만큼 보장
   → 여유 리소스가 있으면 무제한 사용 가능
   → 다른 Pod가 requests를 요구하면 그것도 보장됨
```

> **CPU 권장**: requests만 설정 → 여유 사이클 낭비 없이 유연하게 사용  
> **Memory 주의**: limits 없이 requests만 설정하면, 메모리는 throttle이 불가능하므로 초과 시 OOMKill로만 회수 가능

---

## LimitRange — 네임스페이스 기본값

Pod에 requests/limits가 명시되지 않은 경우 **자동으로 적용**되는 기본값.  
네임스페이스 단위로 설정하며, **새로 생성되는 Pod에만 적용**된다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 500m
        memory: 256Mi
      max:
        cpu: "1"
        memory: "1Gi"
      min:
        cpu: 100m
        memory: 64Mi
```

---

## ResourceQuota — 네임스페이스 총량 제한

네임스페이스 내 **모든 Pod의 리소스 합산**에 상한선을 설정한다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```
