# etcd

## etcd란?

**분산형 신뢰 가능한 Key-Value 저장소** — 단순하고, 안전하고, 빠르다.

Kubernetes 클러스터의 모든 상태 정보를 저장하는 데이터베이스 역할을 한다.

---

## 저장소 유형 비교

| 유형 | 스키마 | 복잡한 쿼리 | 유연성 | 적합한 데이터 |
|---|---|---|---|---|
| **관계형 DB** (SQL) | 필수 | 가능 | 낮음 | 정형 데이터 |
| **Document Store** (MongoDB 등) | 불필요 | 제한적 | 높음 | 반정형 데이터 |
| **Key-Value Store** (etcd) | 불필요 | 미지원 | 매우 높음 | 단순 빠른 조회 |

---

## Key-Value Store 구조

```
key           →  value
─────────────────────────────
name          →  John
age           →  45
salary        →  5000
user:john-doe →  { "location": "New York", "grade": "A" }
```

하나의 키-값 변경이 다른 데이터에 영향을 주지 않는다.

---

## 설치 및 실행

```bash
# 바이너리 다운로드 후 실행
./etcd
# 기본 포트 2379에서 서비스 시작
```

---

## 기본 CLI 명령어 (etcdctl)

```bash
# 데이터 저장
etcdctl put key1 value1

# 데이터 조회
etcdctl get key1

# 데이터 삭제
etcdctl del key1

# 버전 확인
etcdctl version
```

---

## API 버전 변화 (v2 vs v3)

| 작업 | v2 명령어 | v3 명령어 |
|---|---|---|
| 저장 | `etcdctl set` | `etcdctl put` |
| 조회 | `etcdctl get` | `etcdctl get` |
| 삭제 | `etcdctl rm` | `etcdctl del` |
| 트랜잭션 | 미지원 | 지원 |

> 오래된 블로그/문서에서 `set`, `rm` 명령을 보면 v2 기준이다. `etcdctl version`으로 API 버전 확인 가능.

---

## etcd in Kubernetes

### 저장 데이터

`kubectl get`으로 조회되는 모든 정보는 etcd에서 읽어온다. etcd에 저장 완료되어야 변경이 적용된 것으로 간주한다.

```
/registry
├── nodes
├── pods
├── replicasets
├── deployments
├── configmaps
├── secrets
├── roles
└── rolebindings
```

### 포트 구분

| 포트 | 용도 |
|---|---|
| **2379** | 클라이언트 통신 — kube-apiserver, kubectl 등 외부에서 데이터 읽기/쓰기 |
| **2380** | 피어 통신 — etcd 인스턴스 간 데이터 동기화 (Raft 프로토콜) |

### 배포 방식

| 방식 | etcd 배포 방법 |
|---|---|
| **직접 구성 (scratch)** | 바이너리 다운로드 → 직접 서비스 등록, `--advertise-client-urls`로 주소(포트 2379) 설정 |
| **kubeadm** | `kube-system` 네임스페이스에 Pod로 자동 배포 |

```bash
# kubeadm 환경에서 저장된 키 목록 조회
kubectl exec etcd-master -n kube-system -- etcdctl get / --prefix --keys-only
```

### 고가용성(HA) 환경

마스터 노드가 여러 개인 경우, 각 노드의 etcd 인스턴스가 서로를 인식해야 한다.

```bash
# etcd 서비스 설정에서 클러스터 멤버 지정
--initial-cluster peer-1=https://...:2380,peer-2=https://...:2380
```

---

## 릴리스 히스토리

| 시기 | 버전 | 주요 변경 |
|---|---|---|
| 2013.08 | v0.1 | 최초 릴리스 |
| 2015.02 | v2.0 | Raft 알고리즘 재설계, 1000+ writes/sec |
| 2017.01 | v3.0 | 성능 최적화, API 변경, 트랜잭션 지원 |
| 2018.11 | — | CNCF 인큐베이터 프로젝트 편입 |
| 2020.11 | — | CNCF 졸업(Graduated) |
| 2021.06 | v3.5 | 최신 안정 버전 |
