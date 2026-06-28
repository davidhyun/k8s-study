# Multiple Schedulers

## 요약

커스텀 스케줄러를 만들고 `schedulerName`으로 Pod에 지정할 수 있다.

---

## 역할

Kubernetes는 커스텀 스케줄러를 추가로 배포할 수 있다.  
특정 Pod만 커스텀 스케줄러를 사용하도록 지정하고, 나머지는 기본 스케줄러가 처리한다.

---

## 스케줄러 이름 설정

여러 스케줄러가 공존하려면 각각 **고유한 이름**이 필요하다.  
이름은 스케줄러 설정 파일에 지정한다.

```yaml
# kube-scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-custom-scheduler
```

> 기본 스케줄러 이름은 `default-scheduler`.

---

## 커스텀 스케줄러 배포 (Pod/Deployment)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
    - name: kube-scheduler
      image: k8s.gcr.io/kube-scheduler:v1.x.x
      command:
        - kube-scheduler
        - --config=/etc/kubernetes/my-scheduler-config.yaml
```

### leaderElect 옵션

HA 구성(마스터 노드 여러 대)에서 같은 스케줄러가 여러 복사본으로 실행될 때,  
**하나만 활성화**되도록 리더를 선출하는 옵션.

```yaml
# my-scheduler-config.yaml
leaderElection:
  leaderElect: false   # 단일 마스터면 false, HA면 true
```

---

## Pod에 스케줄러 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  schedulerName: my-custom-scheduler   # 커스텀 스케줄러 지정
  containers:
    - name: app
      image: app
```

> 지정한 스케줄러가 올바르게 구성되지 않으면 Pod는 **Pending** 상태로 남는다.

---

## 스케줄링 4단계와 플러그인

Pod가 스케줄되는 내부 흐름과 각 단계에서 동작하는 플러그인.

```
1. Scheduling Queue  → 우선순위 순으로 정렬 (PrioritySort 플러그인)
2. Filter            → 조건 불만족 노드 제거 (NodeResourcesFit, NodeName, NodeUnschedulable 등)
3. Score             → 남은 노드에 점수 부여 (NodeResourcesFit, ImageLocality 등)
4. Binding           → 최고 점수 노드에 Pod 바인딩 (DefaultBinder)
```

각 단계에는 **extension point**가 있어 커스텀 플러그인을 꽂을 수 있다.  
(preFilter → filter → postFilter → preScore → score → reserve → permit → preBind → bind → postBind)

---

## Scheduler Profiles (v1.18+)

별도 바이너리로 여러 스케줄러를 띄우면 **race condition** 위험이 있다.  
v1.18부터는 하나의 스케줄러 바이너리 안에 여러 프로필을 정의할 수 있다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: my-scheduler
    plugins:
      score:
        disabled:
          - name: "*"   # 기본 score 플러그인 전부 비활성화
  - schedulerName: my-scheduler-2
    plugins:
      filter:
        disabled:
          - name: TaintToleration   # 특정 플러그인만 비활성화
```

> 각 프로필이 별도 스케줄러처럼 동작하면서 단일 프로세스로 관리 → race condition 없음.

---

## 확인 명령어

```bash
# 어떤 스케줄러가 Pod를 스케줄했는지 확인
kubectl get events -o wide
# REASON: Scheduled, SOURCE: my-custom-scheduler 확인

# 스케줄러 Pod 로그 확인
kubectl logs my-custom-scheduler -n kube-system
```
