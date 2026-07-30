# Persistent Volume & Persistent Volume Claim

## 개념

Pod는 일시적(transient)이라 삭제되면 내부 데이터도 함께 사라진다.  
데이터를 유지하려면 Pod에 **Volume을 연결**한다.

```
Pod 삭제 → 컨테이너 데이터 삭제
Pod 삭제 → Volume 데이터 유지  ✅
```

---

## YAML 정의

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-pod
spec:
  containers:
    - name: alpine
      image: alpine
      command: ["/bin/sh", "-c"]
      args: ["shuf -i 1-100 -n 1 >> /opt/number.out"]
      volumeMounts:
        - mountPath: /opt          # 컨테이너 내부 경로
          name: data-volume
  volumes:
    - name: data-volume
      hostPath:
        path: /data                # 호스트 디렉토리
        type: Directory
```

---

## 스토리지 옵션

### hostPath

호스트 노드의 디렉토리를 그대로 사용.

```yaml
volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

> **단일 노드에서만 권장** — 멀티 노드 클러스터에서는 각 노드의 `/data`가 서로 달라 데이터 불일치 발생.

### 외부 스토리지 (멀티 노드 권장)

| 유형 | 예시 |
|---|---|
| 퍼블릭 클라우드 | AWS EBS, Azure Disk/File, GCP Persistent Disk |
| 네트워크 스토리지 | NFS, CephFS, GlusterFS |
| 기타 | Fibre Channel, ScaleIO, Flocker |

AWS EBS 예시:

```yaml
volumes:
  - name: data-volume
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```

---

> 볼륨을 Pod마다 개별 정의하면 관리가 복잡해진다. → **Persistent Volume(PV) / Persistent Volume Claim(PVC)** 으로 중앙 관리.

---

## Persistent Volume (PV)

관리자가 클러스터 단위로 스토리지 풀을 미리 생성해두면, 사용자는 PVC를 통해 필요한 만큼 가져다 쓴다.

```
관리자: PV 생성 (스토리지 풀)
사용자: PVC 생성 → PV에서 필요한 용량 할당
Pod:    PVC를 마운트해서 사용
```

### YAML 정의

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain  # Retain / Delete / Recycle
  hostPath: # ⚠️ 테스트/학습 전용 — 프로덕션 사용 금지
    path: /tmp/data
```

### accessModes

| 값 | 의미 |
|---|---|
| `ReadOnlyMany` | 여러 노드에서 읽기 전용 마운트 |
| `ReadWriteOnce` | 단일 노드에서 읽기/쓰기 마운트 |
| `ReadWriteMany` | 여러 노드에서 읽기/쓰기 마운트 |

### kubectl 명령어

```bash
kubectl create -f pv.yaml
kubectl get persistentvolume
```

### 외부 스토리지 사용 (프로덕션)

```yaml
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

> **`hostPath`는 프로덕션에서 절대 사용하지 않는다.**  
> 멀티 노드 환경에서 Pod가 다른 노드에 스케줄되면 데이터가 없는 노드의 디렉토리를 바라보게 되어 데이터 불일치가 발생한다.  
> 프로덕션에서는 반드시 AWS EBS, GCP Persistent Disk, NFS 등 외부 스토리지를 사용한다.

---

## Persistent Volume Claim (PVC)

사용자가 PV 풀에서 스토리지를 요청하는 오브젝트. PV와 PVC는 **1:1로 바인딩**된다.

### 바인딩 규칙

| 기준 | 설명 |
|---|---|
| **accessModes** | PVC와 PV의 accessModes가 일치해야 함 |
| **storage 용량** | PV 용량 ≥ PVC 요청 용량 |
| **storageClassName** | 둘 다 같은 StorageClass를 지정하거나 둘 다 없어야 함 |
| **레이블/셀렉터** | PVC에 selector 지정 시 해당 레이블을 가진 PV에만 바인딩 |

- 조건을 만족하는 PV 중 **용량이 가장 작은 것**을 우선 선택
- 딱 맞는 PV가 없으면 더 큰 PV에 바인딩 (남은 용량은 다른 클레임 사용 불가)
- 조건을 만족하는 PV가 없으면 PVC는 **Pending** 상태 유지 → 새 PV 추가 시 자동 바인딩

### YAML 정의

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### kubectl 명령어

```bash
kubectl create -f pvc.yaml
kubectl get persistentvolumeclaim
kubectl delete persistentvolumeclaim my-claim
```

### Pod에서 PVC 사용

```yaml
spec:
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-claim
  containers:
    - name: app
      image: app
      volumeMounts:
        - mountPath: /var/data
          name: my-storage
```

---

## PVC 삭제 시 PV 처리 정책 (persistentVolumeReclaimPolicy)

| 정책 | 동작 |
|---|---|
| `Retain` (기본값) | PV 유지, 관리자가 수동 삭제 — 다른 클레임에서 재사용 불가 |
| `Delete` | PVC 삭제 즉시 PV도 함께 삭제 |
| `Recycle` | 데이터 삭제 후 재사용 — **deprecated**, 사용 비권장 |

> 현재 권장 방식은 **StorageClass + CSI 드라이버를 사용한 동적 프로비저닝**이다.

---

## Static vs Dynamic Provisioning

| 방식 | 설명 |
|---|---|
| **Static Provisioning** | 관리자가 PV/PVC 직접 생성 |
| **Dynamic Provisioning** | PVC만 생성하면 StorageClass가 PV 자동 생성 |

---

## StorageClass (Dynamic Provisioning)

StorageClass를 사용하면 PVC 생성 시 PV와 스토리지를 자동으로 프로비저닝한다.  
**PV를 직접 만들 필요 없다.**

```
PVC 생성 → StorageClass(nfs-provisioner)가 PV 자동 생성 → PVC와 바인딩
```

> **PVC 생성 시 `storageClassName`만 지정하면 된다.**  
> PV 생성, 스토리지 할당, PVC 바인딩까지 StorageClass가 자동으로 처리한다.

### 주요 Provisioner

| 환경 | Provisioner |
|---|---|
| NFS | `nfs-client` (nfs-provisioner 설치 필요) |
| 로컬 (테스트) | `rancher.io/local-path` |
| GCP | `kubernetes.io/gce-pd` |
| AWS | `kubernetes.io/aws-ebs` |
| Azure | `kubernetes.io/azure-disk` |

---

### NFS Provisioner (온프레미스 권장)

NFS 서버의 공유폴더를 스토리지 풀로 사용한다. 별도 nfs-provisioner 설치가 필요하다.

```
NFS 서버 공유폴더
└── PVC-A 디렉토리
└── PVC-B 디렉토리
└── archived/          ← PVC 삭제 시 자동 백업
```

#### StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs-client          # nfs-provisioner 설치 필요
reclaimPolicy: Delete            # 기본값 Delete → Retain으로 변경 가능
parameters:
  archivedOnDelete: "true"       # PVC 삭제 시 데이터 삭제 대신 archived 폴더로 백업
```

#### PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteMany                # NFS는 다중 노드 동시 마운트 가능
  storageClassName: nfs-client
  resources:
    requests:
      storage: 500Mi
```

> NFS는 `ReadWriteMany` 지원 — 여러 Pod가 동시에 같은 볼륨을 읽고 쓸 수 있어 온프레미스 멀티 노드 환경에 적합하다.

#### 데이터 저장 위치 및 동작

| 상황 | 동작 |
|---|---|
| PVC 생성 | NFS 공유폴더 하위에 디렉토리 자동 생성 |
| PVC 삭제 (`archivedOnDelete: true`) | 데이터 삭제 대신 `archived` 폴더로 이동 |
| PVC 삭제 (`archivedOnDelete: false`) | 데이터 완전 삭제 |

#### kubectl 명령어

```bash
kubectl get sc                        # StorageClass 목록
kubectl describe sc nfs-client        # 상세 확인 (archivedOnDelete 등)
```
