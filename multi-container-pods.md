# Multi-container Pods

## 개념

하나의 Pod 안에 여러 컨테이너를 함께 실행하는 방식.  
마이크로서비스로 분리하되, 두 서비스가 긴밀하게 함께 동작해야 할 때 사용한다.

- **생명주기 공유**: 함께 생성되고 함께 종료
- **네트워크 공유**: `localhost`로 서로 통신 가능
- **스토리지 공유**: 동일한 볼륨 접근 가능

```yaml
spec:
  containers:
    - name: web-app
      image: web-app
    - name: main-app      # containers는 배열 → 여러 컨테이너 정의 가능
      image: main-app
```

---

## 3가지 패턴 비교

| 패턴 | 시작 순서 | 종료 시점 | 용도 |
|---|---|---|---|
| **Co-located** | 순서 보장 없음 | 함께 종료 | 상호 의존하는 두 서비스 |
| **Init Container** | init → main 순 | init은 main 시작 전 종료 | main 시작 전 초기화 작업 |
| **Sidecar** | sidecar → main 순 | main과 함께 종료 | 로그 수집 등 보조 서비스 |

---

## 1. Co-located (기본 Multi-container)

순서 보장 없이 두 컨테이너가 함께 시작·종료된다.

```yaml
spec:
  containers:
    - name: web-app
      image: web-app
    - name: main-app
      image: main-app
```

---

## 2. Init Container

main 컨테이너 시작 전 초기화 작업을 수행하고 종료된다.  
여러 개 정의하면 **순서대로 순차 실행**되며, 각각 exit 0으로 성공해야 다음으로 넘어간다.

```yaml
spec:
  initContainers:
    - name: init-myservice
      image: busybox:1.31
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
    - name: init-mydb
      image: busybox:1.31
      command: ['sh', '-c', 'until nslookup mydb; do echo waiting; sleep 2; done']
  containers:
    - name: main-app
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```

```
init-myservice 실행 → 성공(exit 0) → 종료
init-mydb 실행 → 성공(exit 0) → 종료
main-app 시작 → 계속 실행
```

> init container가 실패하면 Pod 전체가 재시작되고 모든 init container가 처음부터 다시 실행된다.

### restartPolicy 동작 (컨테이너 단위 적용)

| 정책 | 동작 |
|---|---|
| `Always` (기본) | 종료 코드 관계없이 항상 재시작 |
| `OnFailure` | 비정상 종료(non-zero)일 때만 재시작 |
| `Never` | 재시작 안 함 |

> Pod 전체가 재시작되는 것이 아니라 **해당 컨테이너만** 재시작된다. Pod는 노드 장애 또는 컨트롤러에 의해서만 재생성된다.

---

## 3. Sidecar Container (Native, k8s v1.33+)

`initContainers`에 `restartPolicy: Always`를 설정하면 Kubernetes가 Sidecar로 인식한다.

```yaml
spec:
  initContainers:
    - name: sidecar-logger
      image: busybox:1.31
      restartPolicy: Always   # Sidecar로 동작
      command: ['sh', '-c', 'while true; do echo Sidecar running; sleep 10; done']
  containers:
    - name: main-app
      image: busybox:1.31
      command: ['sh', '-c', 'echo Main app starting; sleep 60']
```

```
sidecar-logger 시작 (readiness 대기)
    → main-app 시작
    → main-app 종료
    → sidecar-logger 종료
```

- sidecar는 main 컨테이너보다 먼저 시작, 나중에 종료
- Pod의 `restartPolicy`와 무관하게 **항상 재시작**
- **실사용 예**: Filebeat(sidecar)가 앱 시작 로그부터 종료 로그까지 Elasticsearch로 전송
