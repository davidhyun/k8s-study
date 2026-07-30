# Deployment

## 역할

ReplicaSet보다 상위 개념으로, 프로덕션 환경에서 필요한 배포 관련 기능을 모두 제공한다.

| 기능 | 설명 |
|---|---|
| **Rolling Update** | Pod를 한 번에 하나씩 순차 업데이트 (무중단 배포) |
| **Rollback** | 문제 발생 시 이전 버전으로 되돌리기 |
| **Pause / Resume** | 여러 변경사항을 묶어서 한 번에 적용 |

---

## 오브젝트 계층 구조

```
Deployment
  └── ReplicaSet (자동 생성)
        └── Pod × N (자동 생성)
```

---

## YAML 정의 파일

ReplicaSet과 거의 동일하며 `kind`만 다르다.

```yaml
apiVersion: apps/v1
kind: Deployment        # ReplicaSet → Deployment로 변경
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx
```

---

## 주요 명령어

```bash
kubectl create -f deployment-definition.yaml   # 생성
kubectl get deployments                        # Deployment 목록
kubectl get replicaset                         # 자동 생성된 ReplicaSet 확인
kubectl get pods                               # 자동 생성된 Pod 확인
kubectl get all                                # 모든 오브젝트 한번에 확인
```
