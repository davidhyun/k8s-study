# Taints & Tolerations

## 핵심 개념

> **노드를 보호하는 스프레이 + 그 스프레이에 면역인 곤충**

```
Taint       : 노드에 설정 → "이 노드는 특정 조건을 만족하지 않는 Pod를 거부한다"
Toleration  : Pod에 설정 → "나는 이 taint를 견딜 수 있다"
```

- Taint는 **노드**에, Toleration은 **Pod**에 설정한다
- 보안/침입 방지 개념이 **아니다** — 단지 어떤 Pod가 어떤 노드에 스케줄될 수 있는지 제한하는 것

---

## 동작 원리

```
기본 상태: Pod는 어떤 taint도 견디지 못함 (toleration 없음)
      │
node1에 taint(blue) 설정
      │
      ▼
toleration 없는 Pod → node1에서 튕겨나감 (다른 노드로 배치)
toleration(blue) 있는 Pod → node1에 배치 허용
```

> **주의**: taint는 "특정 Pod만 허용"하는 것이지 "특정 Pod를 반드시 그 노드로 보낸다"는 의미가 아니다.  
> toleration이 있는 Pod도 다른(taint 없는) 노드에 배치될 수 있다.  
> 특정 노드에 강제로 배치하려면 **Node Affinity**가 필요하다.

---

## Taint 설정

```bash
kubectl taint nodes node1 app=blue:NoSchedule
```

### 문법 2가지

```
key=value:effect   # app=blue:NoSchedule
key:effect          # value 없이도 가능 (value는 빈 문자열로 취급)
```

> 예: master(control-plane) 노드의 기본 taint는 `node-role.kubernetes.io/control-plane:NoSchedule`처럼 value가 없는 형태다.

### Taint 제거

명령 끝에 `-`(마이너스)를 붙이면 제거된다. key, value(있다면), effect가 **정확히 일치**해야 한다.

```bash
kubectl taint nodes node1 app=blue:NoSchedule-

# value 없는 taint 제거
kubectl taint nodes controlplane node-role.kubernetes.io/control-plane:NoSchedule-
```

```bash
# 현재 설정된 taint 확인
kubectl describe node node1 | grep Taints
```

### Taint Effect 3종

| Effect | 동작 |
|---|---|
| `NoSchedule` | 새 Pod를 스케줄링하지 않음 |
| `PreferNoSchedule` | 가급적 배치하지 않으려 함 (보장 아님) |
| `NoExecute` | 새 Pod 스케줄링 금지 + **기존에 실행 중이던 Pod도 추방(evict)** |

---

## Toleration 설정 (Pod YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx
      image: nginx
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
```

> 쌍따옴표는 필수가 아니다. YAML은 일반 영숫자 문자열을 따옴표 없이도 문자열로 인식한다.  
> 값이 `true`/`false`/`null`/숫자처럼 다른 타입으로 잘못 해석될 위험이 있을 때만 따옴표로 명시하면 된다.

---

## NoExecute 예시

```
node1에 NoExecute taint 적용
      │
      ▼
toleration 없는 기존 Pod → 즉시 evict(추방, kill)
toleration 있는 기존 Pod → 계속 실행 유지
```

---

## Master Node의 기본 Taint

클러스터 생성 시 master node에는 **자동으로 taint가 설정**되어 일반 워크로드 Pod가 배치되지 않는다.

```bash
kubectl describe node kube-master | grep Taint
```

> 마스터 노드에 애플리케이션 워크로드를 배치하지 않는 것이 best practice.
