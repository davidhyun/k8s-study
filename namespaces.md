# Namespaces

## 역할

클러스터 내 리소스를 **논리적으로 격리**하는 단위. 같은 클러스터를 dev/prod처럼 분리해서 사용할 수 있다.

```
같은 집(클러스터) 안에서도 다른 집(namespace) 사람과는
풀네임(이름.네임스페이스)으로 불러야 한다.
```

---

## 기본 제공 Namespace

| Namespace | 용도 |
|---|---|
| **default** | 별도 지정 없을 때 기본적으로 사용되는 네임스페이스 |
| **kube-system** | DNS, 네트워킹 등 Kubernetes 내부 시스템용 리소스 (사용자 격리) |
| **kube-public** | 모든 사용자에게 공개되어야 하는 리소스 |

---

## 네임스페이스 간 통신

같은 namespace 내에서는 서비스 이름만으로 접근 가능하지만, 다른 namespace의 서비스에 접근하려면 DNS 풀네임이 필요하다.

```
<서비스이름>.<네임스페이스>.svc.cluster.local
```

```
예: db-service.dev.svc.cluster.local
```

| 부분 | 의미 |
|---|---|
| `db-service` | 서비스 이름 |
| `dev` | 네임스페이스 |
| `svc` | 서비스 서브도메인 |
| `cluster.local` | 클러스터 기본 도메인 |

---

## 명령어

```bash
# 특정 네임스페이스의 Pod 조회
kubectl get pods --namespace=kube-system

# 모든 네임스페이스의 Pod 조회
kubectl get pods --all-namespaces

# 네임스페이스 생성
kubectl create -f namespace-dev.yml
kubectl create namespace dev

# 현재 컨텍스트의 기본 네임스페이스를 dev로 전환
kubectl config set-context --current --namespace=dev
```

### 네임스페이스 생성 (YAML)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### Pod를 특정 네임스페이스에 고정 (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev   # 항상 dev에 생성되도록 고정
```

---

## ResourceQuota

네임스페이스별로 리소스 사용량을 제한할 수 있다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 10Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```
