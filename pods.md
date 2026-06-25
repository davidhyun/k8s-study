# Pods

## Pod란?

**컨테이너를 감싸는 Kubernetes의 최소 배포 단위**.  
Kubernetes는 컨테이너를 직접 워커 노드에 배포하지 않고, 반드시 Pod로 감싸서 배포한다.

---

## Pod와 컨테이너의 관계

```
일반적: Pod 1개 = 컨테이너 1개 (1:1 관계)
예외:   Pod 1개 = 메인 컨테이너 + helper 컨테이너 (Multi-container Pod)
```

### 스케일링 방법

| 목적 | 방법 |
|---|---|
| 스케일 업 | 기존 Pod에 컨테이너 추가 X → **새 Pod 생성** |
| 스케일 다운 | **기존 Pod 삭제** |
| 노드 용량 초과 시 | 새 노드를 클러스터에 추가 후 Pod 배포 |

---

## Multi-container Pod

같은 Pod 내 컨테이너는 아래를 자동으로 공유한다.

- **네트워크**: `localhost`로 서로 통신 가능
- **스토리지**: 동일한 볼륨 접근 가능
- **생명주기**: 함께 생성되고, 함께 종료됨

> 주로 메인 앱 + 보조 helper 컨테이너 조합에서 사용. 일반적인 케이스는 아님.

---

## YAML 정의 파일

### 4가지 필수 최상위 필드

```yaml
apiVersion: v1          # 사용할 Kubernetes API 버전
kind: Pod               # 생성할 오브젝트 타입
metadata:               # 오브젝트 메타데이터 (딕셔너리)
  name: myapp-pod
  labels:
    app: myapp          # 임의의 key-value 가능
spec:                   # 오브젝트 상세 스펙 (딕셔너리)
  containers:
    - name: nginx-container
      image: nginx      # Docker Hub에서 pull
```

### 필드별 설명

| 필드 | 타입 | 설명 |
|---|---|---|
| `apiVersion` | String | 오브젝트 종류에 따라 다름 (Pod → `v1`) |
| `kind` | String | `Pod`, `ReplicaSet`, `Deployment`, `Service` 등 |
| `metadata` | Dictionary | `name`, `labels`만 허용 (임의 필드 추가 불가) |
| `metadata.labels` | Dictionary | 임의 key-value 자유롭게 추가 가능 |
| `spec.containers` | List | 여러 컨테이너 지원 → `-`(dash)로 항목 구분 |

### apiVersion 참고

| Kind | apiVersion |
|---|---|
| Pod | `v1` |
| ReplicaSet | `apps/v1` |
| Deployment | `apps/v1` |
| Service | `v1` |

### YAML 들여쓰기 주의사항

```yaml
metadata:
  name: myapp-pod   # name과 labels는 같은 depth (형제)
  labels:
    app: myapp      # labels 하위 항목은 한 단계 더 들여쓰기
```

- 형제 항목은 **같은 칸수**로 정렬
- 자식 항목은 부모보다 **더 많은 칸수**로 들여쓰기
- 칸수 자체보다 **일관성**이 중요

---

## 명령어

```bash
# Pod 생성 (명령형)
kubectl run nginx --image=nginx

# Pod 생성 (YAML)
kubectl create -f pod-definition.yaml

# Pod 목록 조회
kubectl get pods

# Pod 상세 정보 (생성 시간, 라벨, 컨테이너, 이벤트 등)
kubectl describe pod myapp-pod
```

> 이미지는 Docker Hub 등 컨테이너 레지스트리에서 pull한다.  
> 생성 직후 `ContainerCreating` → `Running` 상태로 전환된다.
