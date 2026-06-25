# Imperative vs Declarative

## 핵심 차이

```
Imperative : "어떻게(how)" 할지 단계별로 지시
Declarative: "무엇(what)"이 필요한지 선언, 시스템이 알아서 처리
```

> 택시 기사에게 길을 한 단계씩 안내(imperative) vs 목적지만 말하면 시스템이 경로를 찾음(declarative)

---

## Kubernetes에서의 비교

### Imperative — 명령형 커맨드

```bash
kubectl run nginx --image=nginx              # 생성
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80    # 서비스 생성
kubectl edit pod nginx                        # 수정
kubectl scale deployment nginx --replicas=3  # 스케일
kubectl set image deployment nginx nginx=nginx:1.18  # 이미지 변경
```

- 빠르고 간단 (YAML 작성 불필요)
- 복잡한 요구사항(멀티 컨테이너 등)에는 한계
- 실행 후 기록이 남지 않아 **추적 어려움**

### Imperative — 설정 파일 기반

```bash
kubectl create -f pod.yaml    # 생성
kubectl replace -f pod.yaml   # 수정 (객체가 이미 존재해야 함)
kubectl delete -f pod.yaml    # 삭제
```

- YAML은 남지만, **명령어 자체는 imperative** (create/replace를 상황에 맞게 직접 선택)
- 객체 존재 여부를 사용자가 직접 확인해야 함 (없으면 `replace` 실패, 있으면 `create` 실패)

### Declarative — `kubectl apply`

```bash
kubectl apply -f pod.yaml        # 파일 1개
kubectl apply -f ./manifests/    # 디렉토리 전체
```

- 객체가 없으면 생성, 있으면 변경분만 업데이트 → **에러 없이 항상 동작**
- Git 등 코드 저장소로 형상관리 가능 → 변경 리뷰/승인 프로세스 적용 가능

---

## `kubectl edit`의 함정

```
kubectl edit으로 직접 수정
   → 변경 사항이 로컬 YAML 파일에는 반영되지 않음
   → 이후 로컬 YAML로 apply/replace 하면 직전 수정이 덮어써져 사라짐
```

> 변경을 추적하려면: 로컬 YAML 수정 → `kubectl replace` 또는 `kubectl apply` 사용

---

## 명령어 비교 정리

| 구분 | 객체 없을 때 | 객체 있을 때 |
|---|---|---|
| `kubectl create -f` | 생성됨 | **에러** (already exists) |
| `kubectl replace -f` | **에러** (not found) | 업데이트됨 |
| `kubectl apply -f` | 생성됨 | 변경분만 업데이트 (에러 없음) |

---

## `kubectl apply` 내부 동작 원리

`apply`는 변경 사항을 결정할 때 **3가지 설정**을 비교한다.

```
1. Local 파일        — 로컬에 작성한 YAML
2. Live 객체 설정     — 클러스터(etcd)에 실제 저장된 현재 상태
3. Last Applied 설정  — 마지막으로 apply했던 시점의 설정 (JSON)
```

### 동작 흐름

```
객체 없음 → 생성
  - Live 객체 생성
  - Local YAML을 JSON으로 변환해 "last-applied-configuration" 어노테이션으로 Live 객체에 저장

객체 있음 → 비교 후 업데이트
  - Local ↔ Live ↔ Last Applied 세 값을 비교
  - 필드가 Local에 있으면 → Live에 반영
  - 필드가 Live에만 있고 Local·Last Applied에 없으면 → 그대로 유지
  - 필드가 Last Applied에는 있었는데 Local에서 사라졌으면 → 삭제된 것으로 판단해 Live에서도 제거
  - 변경 후 Last Applied는 항상 최신 상태로 갱신
```

> **last-applied-configuration이 필요한 이유**: 단순히 Local과 Live만 비교하면 "필드가 삭제됐다"는 것을 알 수 없다. Last Applied가 있어야 "이전엔 있었는데 지금은 없다 → 삭제 의도"를 판단할 수 있다.

### 저장 위치

`last-applied-configuration`은 별도 파일이 아니라, **Live 객체의 annotation**으로 클러스터 내부에 저장된다.

```bash
kubectl get pod myapp-pod -o yaml
# metadata.annotations.kubectl.kubernetes.io/last-applied-configuration 확인 가능
```

> `create`/`replace`(imperative)는 이 annotation을 저장하지 않는다.  
> **imperative와 declarative 방식을 섞어서 쓰면** last-applied-configuration이 꼬여 의도치 않은 필드 삭제/누락이 발생할 수 있다 — 한 객체에는 한 가지 방식만 일관되게 사용할 것.
