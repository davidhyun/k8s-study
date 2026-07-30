# ConfigMap

## 역할

Pod definition 파일에 흩어진 환경변수를 **중앙에서 관리**하는 오브젝트.  
Key-Value 형태로 데이터를 저장하고, Pod 생성 시 주입한다.

> 환경변수를 `env.value`로 Pod에 직접 하드코딩할 수도 있지만, Pod가 많아지면 관리가 어려워져 ConfigMap으로 분리한다.

---

## 생성 방법

### 방법 1: Imperative (명령어)

```bash
# 직접 key-value 지정
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production

# 파일에서 읽기
kubectl create configmap app-config --from-file=app_config.properties
```

### 방법 2: Declarative (YAML)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: production
```

```bash
kubectl create -f configmap.yaml
```

---

## 조회

```bash
kubectl get configmaps
kubectl describe configmap app-config
```

---

## Pod에 주입하는 방법

### 방법 1: envFrom — ConfigMap 전체를 환경변수로 주입

```yaml
spec:
  containers:
    - name: app
      image: app
      envFrom:
        - configMapRef:
            name: app-config
```

### 방법 2: env — 특정 key만 환경변수로 주입

```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR
```

### 방법 3: Volume — 파일로 마운트

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
```
