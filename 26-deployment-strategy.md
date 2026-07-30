# Deployment Strategy (Rolling Update & Rollback)

## 배포 전략

| 전략 | 동작 | 특징 |
|---|---|---|
| **RollingUpdate** (기본값) | 구버전 Pod를 하나씩 내리고 신버전을 하나씩 올림 | 무중단 배포 |
| **Recreate** | 구버전 전체 삭제 후 신버전 일괄 생성 | 배포 중 다운타임 발생 |

---

## Rollout & 버전 관리

Deployment를 생성하거나 업데이트할 때마다 새 **revision**이 기록된다.

```bash
kubectl rollout status deployment myapp-deployment    # 롤아웃 진행 상태
kubectl rollout history deployment myapp-deployment   # 개정 이력 확인
```

---

## 업데이트 방법

```bash
# 방법 1: YAML 수정 후 apply (권장 — 파일과 클러스터 상태 일치)
kubectl apply -f deployment-definition.yaml

# 방법 2: 명령어로 이미지 직접 변경 (파일과 불일치 주의)
kubectl set image deployment myapp-deployment nginx=nginx:1.18
```

### 업데이트 내부 동작 (RollingUpdate)

```
구 ReplicaSet (5개) → Pod 1개 감소
신 ReplicaSet (0개) → Pod 1개 증가
        ... 반복 ...
구 ReplicaSet (0개)
신 ReplicaSet (5개)
```

---

## Rollback

```bash
# 이전 revision으로 되돌리기
kubectl rollout undo deployment myapp-deployment
```

롤백 후 구 ReplicaSet이 다시 활성화되고 신 ReplicaSet은 0으로 축소된다.

---

## 명령어 요약

```bash
kubectl rollout status deployment myapp-deployment    # 롤아웃 상태
kubectl rollout history deployment myapp-deployment   # 이력 확인
kubectl rollout undo deployment myapp-deployment      # 롤백
kubectl set image deployment myapp-deployment nginx=nginx:1.18  # 이미지 변경
```
