# Backup Cluster

## 백업 대상 3가지

| 대상 | 설명 |
|---|---|
| **리소스 설정** | Deployment, Service, ConfigMap 등 오브젝트 정의 |
| **etcd** | 클러스터 전체 상태 저장소 |
| **Persistent Volume** | 애플리케이션 데이터 |

---

## 방법 1: 리소스 설정 백업 (kube-apiserver 쿼리)

선언형(declarative)으로 관리하면 YAML 파일 자체가 백업이다.  
하지만 명령형(imperative)으로 생성된 오브젝트는 파일이 없으므로 API 서버를 직접 조회해 백업해야 한다.

```bash
# 전체 네임스페이스의 모든 리소스를 YAML로 추출
kubectl get all --all-namespaces -o yaml > all-resources.yaml
```

> Velero(구 Ark)와 같은 도구를 사용하면 더 체계적으로 백업할 수 있다.

---

## 방법 2: etcd 백업

etcd에는 클러스터 전체 상태(노드, 오브젝트 등)가 저장되어 있어 가장 완전한 백업이 된다.

> etcd 버전 확인: `etcdctl version`

### etcdctl — 스냅샷 백업 (live etcd 대상)

```bash
etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### etcdutl — 파일 기반 백업 (etcd 중지 상태에서도 가능)

```bash
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup
```

### 스냅샷 상태 확인

```bash
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

---

## etcd 복구 절차

### Step 1. kube-apiserver 중지

```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30
```

### Step 2. 스냅샷 복구

```bash
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir /var/lib/etcd-from-backup
```

### Step 3. etcd 설정 변경 (volumes hostPath 수정)

```bash
vi /etc/kubernetes/manifests/etcd.yaml
```

volumes 섹션에서 etcd-data의 hostPath를 변경:

```yaml
# 변경 전
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data

# 변경 후
  - hostPath:
      path: /var/lib/etcd-from-backup
      type: DirectoryOrCreate
    name: etcd-data
```

> `--data-dir` 플래그는 etcd.yaml의 command 인자(컨테이너 내부 경로)와 volumes hostPath(호스트 실제 경로) 두 곳에 존재한다.  
> 복구 시 변경하는 것은 **volumes hostPath만** 이다 — command의 `--data-dir`은 컨테이너 내부 마운트 경로이므로 그대로 둔다.  
> 파일 저장 시 kubelet이 감지해 etcd Pod 자동 재시작.

### Step 4. kube-apiserver 재시작

```bash
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
sleep 60
```

### Step 5. 나머지 컨트롤 플레인 컴포넌트 재시작

```bash
# kube-controller-manager
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
sleep 20
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/

# kube-scheduler
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
sleep 20
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

# kubelet
systemctl restart kubelet
```

### Step 6. 재시작 모니터링

```bash
watch crictl ps
# 모든 컴포넌트 STATUS = Running 확인 (약 2~3분 소요)
```

### Step 7. 복구 확인

```bash
kubectl get deployments,services --all-namespaces
kubectl get pods --all-namespaces
kubectl get nodes
```

> `restore` 실행 시 etcd는 새로운 클러스터를 초기화한다 — 기존 클러스터에 실수로 합류하는 것을 방지하기 위함.

### etcdutl backup 복구

백업 디렉토리 내용을 `/var/lib/etcd`로 복사 후 etcd 재시작.

---

## etcdctl vs etcdutl

| 항목 | etcdctl | etcdutl |
|---|---|---|
| 스냅샷 생성 | `snapshot save` (live etcd 대상) | - |
| 스냅샷 복구 | - | `snapshot restore` |
| 파일 백업 | - | `backup` (etcd 중지 상태 가능) |
| 스냅샷 상태 확인 | `snapshot status` | - |

---

## 두 방법 비교

| 항목 | 리소스 설정 백업 | etcd 백업 |
|---|---|---|
| 방식 | kube-apiserver 쿼리 | etcd 스냅샷 |
| 접근 권한 | kubectl만 있으면 됨 | etcd 직접 접근 필요 |
| 관리형 클러스터 (EKS 등) | ✅ 사용 가능 | ❌ etcd 접근 불가한 경우 많음 |
| 백업 범위 | 오브젝트 정의만 | 클러스터 전체 상태 |
