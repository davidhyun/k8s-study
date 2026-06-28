# Node Selector & Node Affinity

## 공통 목적

Pod를 **특정 노드에만 배치**하도록 제한.  
노드에 라벨을 붙이고, Pod spec에서 그 라벨을 참조한다.

---

## Node Selector — 단순 방식

### 1단계: 노드에 라벨 붙이기

```bash
kubectl label nodes node1 size=large
```

### 2단계: Pod spec에 nodeSelector 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processing
spec:
  containers:
    - name: data-processor
      image: data-processor
  nodeSelector:
    size: large
```

### 한계

| 요구사항 | nodeSelector로 가능? |
|---|---|
| `size=large` 노드에 배치 | 가능 |
| `large` 또는 `medium` 노드에 배치 (OR) | **불가** |
| `small`이 아닌 노드에 배치 (NOT) | **불가** |

→ 복잡한 조건이 필요하면 **Node Affinity** 사용

---

## Node Affinity — 고급 방식

### YAML 구조

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: size
              operator: In
              values:
                - large
```

### operator 종류

| operator | 의미 | values 필요? |
|---|---|---|
| `In` | 지정한 값 중 하나와 일치 | O |
| `NotIn` | 지정한 값이 아닌 노드 | O |
| `Exists` | 해당 key 라벨이 존재하는 노드 | X |

```yaml
# OR 조건: large 또는 medium
- key: size
  operator: In
  values:
    - large
    - medium

# NOT 조건: small이 아닌 노드
- key: size
  operator: NotIn
  values:
    - small

# 라벨 존재 여부만 확인 (값 무관)
- key: size
  operator: Exists
```

### Affinity 타입

```
                    스케줄링 시점          실행 중 변경 시
                  (Pod 최초 배치)        (노드 라벨 변경 등)
                  ─────────────────────────────────────────
required...ignored   매칭 필수            무시 (계속 실행)
preferred...ignored  최선 시도            무시 (계속 실행)
```

| 타입 | 스케줄링 | 실행 중 |
|---|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | 조건 필수 — 없으면 Pending | 라벨 변경 무시 |
| `preferredDuringSchedulingIgnoredDuringExecution` | 최선 시도 — 없으면 아무 노드 | 라벨 변경 무시 |

> **선택 기준**: 배치가 필수(critical) → `required` / 배치보다 실행이 중요 → `preferred`

---

## 비교 정리

| | Node Selector | Node Affinity |
|---|---|---|
| 표현력 | 단순 key=value | In / NotIn / Exists 등 |
| OR 조건 | 불가 | 가능 |
| NOT 조건 | 불가 | 가능 |
| 노드 미존재 시 | Pending | 타입에 따라 다름 |
| YAML 복잡도 | 낮음 | 높음 |

---

## Taints/Tolerations + Node Affinity 조합

각 기법을 단독으로 쓰면 **완전한 격리**가 되지 않는다.

```
시나리오: blue/red/green 노드에 각각 blue/red/green Pod만 배치하고,
         다른 팀의 Pod와 완전히 분리하고 싶다.
```

### Taints/Tolerations만 사용할 경우

```
blue 노드 → blue taint → blue toleration Pod만 수락
      ✓ 다른 팀 Pod가 우리 노드에 오는 것은 막힘
      ✗ 우리 Pod(red)가 taint 없는 다른 팀 노드에 배치될 수 있음
```

### Node Affinity만 사용할 경우

```
blue Pod → blue 노드 affinity → blue 노드에 배치됨
      ✓ 우리 Pod가 다른 팀 노드에 가는 것은 막힘
      ✗ 다른 팀 Pod가 우리 노드에 배치될 수 있음
```

### 두 가지 조합 → 완전한 격리

```
Taints/Tolerations  : 다른 팀 Pod가 우리 노드에 오지 못하게 차단
Node Affinity       : 우리 Pod가 다른 팀 노드로 가지 못하게 고정

→ 두 방향 모두 차단 = 완전한 노드 전용 할당
```

| 기법 | 막는 방향 |
|---|---|
| Taints & Tolerations | 외부 Pod → 우리 노드 진입 차단 |
| Node Affinity | 우리 Pod → 외부 노드 이탈 차단 |
