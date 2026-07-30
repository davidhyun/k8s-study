# Secrets

## 역할

비밀번호, API 키 등 민감한 정보를 저장하는 오브젝트.  
ConfigMap과 구조는 동일하지만 값이 **Base64 인코딩** 형태로 저장된다.

> 호스트명·유저명은 ConfigMap, 비밀번호·키는 Secret으로 분리하는 것이 원칙이다.

---

## 생성 방법

### 방법 1: Imperative (명령어)

**사용자가 직접 Base64 인코딩할 필요 없다.**

```bash
# 직접 key-value 지정
kubectl create secret generic app-secret \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PASSWORD=supersecret

# secret.properties
DB_HOST=mysql
DB_USER=root
DB_PASSWORD=mypassword

# 파일에서 읽기
kubectl create secret generic app-secret --from-file=secret.properties
```

### 방법 2: Declarative (YAML)

값을 반드시 **Base64 인코딩**해서 작성해야 한다.

```bash
# 인코딩
echo -n 'mysql' | base64        # bXlzcWw=
echo -n 'supersecret' | base64  # c3VwZXJzZWNyZXQ=

# 디코딩 (확인 시)
echo -n 'bXlzcWw=' | base64 --decode
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: bXlzcWw=
  DB_PASSWORD: c3VwZXJzZWNyZXQ=
```

---

## 조회

```bash
kubectl get secrets
kubectl describe secret <secret-name>      # 값은 숨김
kubectl get secret <secret-name> -o yaml   # 인코딩된 값 확인

# 특정 key 디코딩해서 보기
kubectl get secret <secret-name> -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode

# 전체 key-value 한 번에 디코딩 (go-template)
kubectl get secret <secret-name> -o go-template='{{range $k,$v := .data}}{{$k}}: {{$v | base64decode}}{{"\n"}}{{end}}'

# 전체 key-value 한 번에 디코딩 (jq 설치된 경우)
kubectl get secret <secret-name> -o json | jq '.data | map_values(@base64d)'
```

---

## Pod에 주입하는 방법

### 방법 1: envFrom — Secret 전체를 환경변수로 주입

```yaml
spec:
  containers:
    - name: app
      image: app
      envFrom:
        - secretRef:
            name: app-secret
```

### 방법 2: env — 특정 key만 환경변수로 주입

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD
```

### 방법 3: Volume — 파일로 마운트

Secret의 각 key가 파일로 생성되고, 값이 파일 내용이 된다.

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

> Secret이 3개의 key를 가지면 → 마운트 경로에 파일 3개 생성

---

## Encryption at Rest (etcd 저장 데이터 암호화)

Secret 오브젝트는 Base64 인코딩일 뿐 암호화가 아니다.  
etcd에 접근할 수 있다면 누구든 평문으로 읽을 수 있다.  
**Encryption at Rest**를 활성화하면 etcd에 저장될 때 실제로 암호화된다.

### 1. 암호화 활성화 여부 확인

```bash
ps aux | grep kube-apiserver | grep encryption-provider-config
# 결과 없으면 미활성화
```

### 2. 암호화 설정 파일 생성

```bash
# 32바이트 랜덤 키 생성
head -c 32 /dev/urandom | base64
```

```yaml
# /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:                  # 첫 번째 provider가 암호화에 사용됨
          keys:
            - name: key1
              secret: <base64-32byte-key>
      - identity: {}             # 미암호화 (하위 호환용, 맨 뒤에 위치)
```

> providers 순서가 중요 — **첫 번째 provider로 암호화**되고, 복호화는 순서대로 시도한다.  
> `identity`가 첫 번째면 암호화 안 됨.

### 3. kube-apiserver에 옵션 추가

`/etc/kubernetes/manifests/kube-apiserver.yaml` 수정:

```yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml  # 추가
      volumeMounts:
        - name: enc
          mountPath: /etc/kubernetes/enc
          readOnly: true
  volumes:
    - name: enc
      hostPath:
        path: /etc/kubernetes/enc
        type: DirectoryOrCreate
```

### 4. 기존 Secret 재암호화

활성화 이후 **새로 생성되는 Secret만 암호화**된다. 기존 Secret은 아래 명령으로 강제 재암호화:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```
