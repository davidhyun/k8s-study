# Helm

## 개요

Kubernetes용 **패키지 매니저**.  
여러 YAML 파일로 구성된 애플리케이션을 하나의 패키지(Chart)로 묶어 설치/업그레이드/삭제를 단일 명령으로 처리한다.

---

## 왜 Helm인가?

일반적인 앱(예: WordPress)은 여러 Kubernetes 오브젝트로 구성된다.

```
WordPress 앱
├── Deployment  (웹 서버, DB 서버)
├── PersistentVolume
├── PersistentVolumeClaim
├── Service
├── Secret      (비밀번호 등)
└── Job         (백업 등)
```

Helm 없이 관리하면:
- 오브젝트마다 별도 YAML 파일 → `kubectl apply` 반복
- 설정 변경 시 여러 파일을 일일이 수정
- 업그레이드/롤백 시 변경 대상 파악이 어려움
- 삭제 시 관련 오브젝트를 하나씩 기억해서 제거

---

## Helm의 핵심 개념

| 개념 | 설명 |
|---|---|
| **Chart** | 앱을 구성하는 YAML 템플릿 묶음 (패키지) |
| **values.yaml** | 사용자 커스텀 설정을 한 곳에서 관리 |
| **Release** | Chart를 클러스터에 설치한 인스턴스 |
| **Revision** | Release의 변경 스냅샷 — 업그레이드/롤백 단위 |
| **Repository** | Chart를 저장하고 배포하는 저장소 |

---

## Chart

앱을 구성하는 YAML 템플릿 파일 묶음. 실제 값은 `values.yaml`에서 주입한다.

### 디렉토리 구조

```
my-chart/
├── Chart.yaml        # Chart 메타데이터 (이름, 버전, 의존성 등)
├── values.yaml       # 기본 설정값
├── templates/        # YAML 템플릿 파일
│   ├── deployment.yaml
│   └── service.yaml
├── charts/           # 의존 Chart 디렉토리
├── LICENSE
└── README.md
```

### 템플릿 예시

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}   # values.yaml의 값 참조
  containers:
    - image: {{ .Values.image.name }}
```

```yaml
# values.yaml
replicaCount: 2
image:
  name: nginx:1.21
```

> 대부분 Chart를 직접 만들지 않고 Repository에서 다운로드해 사용한다.  
> 커스터마이징은 `values.yaml`만 수정하면 된다.

### Chart.yaml

Chart 자체에 대한 메타데이터를 정의한다.

```yaml
apiVersion: v2              # v1: Helm 2용, v2: Helm 3+용
name: wordpress
description: A Helm chart for WordPress
type: application           # application(기본) 또는 library
version: 15.2.3             # Chart 자체 버전
appVersion: "6.2.0"         # 배포되는 앱(WordPress)의 버전 — 참고용

dependencies:
  - name: mariadb           # 의존 Chart (별도 관리 불필요)
    version: 11.x.x
    repository: https://charts.bitnami.com/bitnami

keywords:
  - wordpress
  - cms

maintainers:
  - name: Bitnami
    email: containers@bitnami.com

home: https://wordpress.org
```

| 필드 | 설명 |
|---|---|
| `apiVersion` | `v2` = Helm 3+용 Chart |
| `version` | Chart 버전 (Chart 변경 시 업데이트) |
| `appVersion` | 배포되는 앱의 버전 (참고용, 기능에 영향 없음) |
| `type: application` | 일반 앱 Chart (기본값) |
| `type: library` | 다른 Chart 빌드를 돕는 유틸리티 Chart |
| `dependencies` | 이 Chart가 의존하는 다른 Chart 목록 |

---

## Release와 Revision

```
helm install my-site bitnami/wordpress
  → Release: my-site (Chart: bitnami/wordpress)
    └── Revision 1 (최초 설치)

helm upgrade my-site bitnami/wordpress
    └── Revision 2 (업그레이드)

helm rollback my-site 1
    └── Revision 3 (Revision 1 상태로 복구 — 새 Revision으로 생성)
```

> rollback은 Revision 1로 되돌아가는 게 아니라, Revision 1과 동일한 설정으로 **새 Revision(3)을 생성**한다.

같은 Chart로 여러 Release를 생성할 수 있다.

```bash
helm install my-site bitnami/wordpress        # 운영용
helm install my-dev-site bitnami/wordpress    # 개발용
```

두 Release는 독립적으로 관리되며 설정도 다르게 지정할 수 있다.

### 특정 버전 설치

```bash
helm install nginx-release nginx/nginx --version 1.19.2
```

### Revision 이력 확인

```bash
helm list                        # 현재 Release 목록 및 Revision 번호 확인
helm history nginx-release       # Revision별 상세 이력 (Chart 버전, 앱 버전, 액션 등)
```

```
REVISION  STATUS     CHART          APP VERSION  DESCRIPTION
1         superseded nginx-1.19.2   1.19.2       Install complete
2         superseded nginx-1.21.4   1.21.4       Upgrade complete
3         deployed   nginx-1.19.2   1.19.2       Rollback to 1
```

### Rollback 주의사항

- Helm rollback은 Kubernetes 오브젝트(매니페스트)만 복구한다.
- PersistentVolume의 실제 데이터는 복구되지 않는다.
- DB 데이터 백업/복구는 Chart Hook을 사용해야 한다.

---

## 메타데이터 저장 방식

Helm은 Release 정보(사용한 Chart, Revision 상태 등)를 **Kubernetes Secret**으로 클러스터에 저장한다.

```bash
kubectl get secret -n <namespace> | grep helm
# sh.helm.release.v1.my-site.v1
# sh.helm.release.v1.my-site.v2
```

- 팀원 누구나 동일한 클러스터에서 Helm 상태를 공유 가능
- 로컬 파일에 저장하지 않아 분실 위험 없음

---

## Repository

Chart를 저장/배포하는 저장소. **Artifact Hub(artifacthub.io)** 에서 검색 가능.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami  # 저장소 추가
helm repo list                                             # 저장소 목록
helm repo update                                           # 로컬 캐시 갱신
helm search repo wordpress    # 로컬 저장소에서 검색
helm search hub wordpress     # Artifact Hub 전체 검색
```

> 공식 개발사가 배포한 Chart는 **Verified Publisher** 배지가 표시된다. 가능하면 이를 우선 사용한다.

---

## 기본 명령어 흐름

```bash
# 1. 저장소 추가 및 업데이트
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. 설치 — 수백 개의 오브젝트를 한 번에 생성
helm install my-release bitnami/wordpress

# 3. 설치된 Release 목록 확인
helm list

# 4. 업그레이드 — 변경된 오브젝트만 자동 업데이트
helm upgrade my-release bitnami/wordpress

# 5. 롤백 — 이전 revision으로 롤백
helm rollback my-release 1

# 6. 삭제 — 관련 오브젝트 전체 삭제
helm uninstall my-release
```

---

## 설치 시 값 커스터마이징

### 방법 1: --set 옵션 (간단한 값 변경)

```bash
helm install my-release bitnami/wordpress \
  --set wordpressBlogName="Helm Tutorials" \
  --set wordpressEmail="john@example.com"
```

### 방법 2: 커스텀 values 파일 (값이 많을 때)

```yaml
# customvalues.yaml
wordpressBlogName: "Helm Tutorials"
wordpressEmail: "john@example.com"
```

```bash
helm install my-release bitnami/wordpress --values customvalues.yaml
```

### 방법 3: Chart를 로컬에 받아서 직접 수정

```bash
# Chart 다운로드 및 압축 해제
helm pull bitnami/wordpress --untar

# values.yaml 직접 편집
vi wordpress/values.yaml

# 로컬 디렉토리에서 설치
helm install my-release ./wordpress
```

> `--set`과 `--values`는 기본 `values.yaml`을 오버라이드한다.  
> 값이 많거나 반복 사용한다면 커스텀 파일로 관리하는 것이 낫다.

---

## Umbrella Chart (다중 앱 한 번에 배포)

여러 서브 앱을 하나의 Chart로 묶어 `helm install` 한 번으로 전체 배포한다.

```
my-app/
├── Chart.yaml        ← dependencies 선언
├── values.yaml       ← 서브 Chart 설정 통합 관리
└── charts/           ← 서브 Chart들이 다운로드되는 위치
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: wordpress
    version: 15.x.x
    repository: https://charts.bitnami.com/bitnami
  - name: mariadb
    version: 11.x.x
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: 17.x.x
    repository: https://charts.bitnami.com/bitnami
```

각 서브 Chart의 값은 상위 `values.yaml`에서 Chart 이름으로 구분해 설정한다.

```yaml
# values.yaml
wordpress:
  wordpressBlogName: "My Site"

mariadb:
  auth:
    rootPassword: "secret"

redis:
  auth:
    enabled: false
```

```bash
# 의존 Chart 다운로드 (charts/ 디렉토리에 저장)
helm dependency update

# 전체 한 번에 배포
helm install my-app ./my-app
```

---

## 실무 Umbrella Chart 구조 (멀티팀 환경)

여러 팀이 독립적으로 개발하는 서비스를 하나의 플랫폼으로 배포할 때 적합하다.

```
platform-chart/          ← 인프라팀 관리 (Umbrella)
├── Chart.yaml
├── values-dev.yaml      ← 환경별 값 분리
├── values-prod.yaml
└── charts/
    ├── frontend/
    ├── backend/
    ├── gis/
    ├── toc/
    ├── forecast/
    └── db/
```

각 팀은 자신의 서브 Chart만 수정하고, 인프라팀이 Umbrella Chart로 전체 릴리즈를 관리한다.

```bash
# 개발 환경 배포
helm install platform ./platform-chart -f values-dev.yaml

# 운영 환경 배포
helm install platform ./platform-chart -f values-prod.yaml
```

### 서브 Chart 참조 방식

| 방식 | 적합한 경우 |
|---|---|
| 로컬 경로 (`file://../frontend`) | 모노레포, 소규모 팀 |
| 별도 Repository (Harbor, ChartMuseum 등) | 서비스별 독립 배포가 필요한 대규모 팀 |

---

## Helm vs 직접 관리 비교

| 작업 | 직접 관리 | Helm |
|---|---|---|
| 설치 | `kubectl apply` 반복 | `helm install` 1회 |
| 설정 변경 | 여러 YAML 파일 수정 | `values.yaml` 1곳 수정 |
| 업그레이드 | 변경 대상 파악 후 수동 수정 | `helm upgrade` 1회 |
| 롤백 | 이전 상태 파악 후 수동 복구 | `helm rollback` 1회 |
| 삭제 | 오브젝트 하나씩 삭제 | `helm uninstall` 1회 |
