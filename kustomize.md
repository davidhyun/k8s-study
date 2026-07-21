# Kustomize

## 등장 배경

환경별(dev/staging/prod)로 다른 설정이 필요할 때, YAML 파일을 각 환경 디렉토리에 복사하는 방식은 관리가 어렵다.

```
# 비효율적인 방식 — 파일 중복
dev/
  deployment.yaml    # replicas: 1
staging/
  deployment.yaml    # replicas: 2
prod/
  deployment.yaml    # replicas: 5
```

- 새 리소스 추가 시 모든 환경 디렉토리에 복사해야 함
- 공통 변경 시 모든 디렉토리를 수정해야 함
- 누락/불일치 발생 가능성 높음

---

## Kustomize란?

**Base + Overlay** 구조로 Kubernetes 설정을 환경별로 관리하는 도구.  
템플릿 언어 없이 순수 YAML만 사용하며, `kubectl`에 내장되어 있다.

```
Base (공통 설정)
  +
Overlay (환경별 변경사항)
  ↓
최종 Kubernetes 매니페스트
```

---

## 핵심 개념

| 개념 | 설명 |
|---|---|
| **Base** | 모든 환경에서 공통으로 사용하는 기본 설정 및 기본값 |
| **Overlay** | 환경별로 Base에서 변경할 값 또는 추가할 리소스 |

---

## 디렉토리 구조

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml     # replicas: 1 (기본값)
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── deployment-patch.yaml   # replicas: 2
    └── prod/
        ├── kustomization.yaml
        └── deployment-patch.yaml   # replicas: 5
```

---

## kustomization.yaml 작성법

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 관리할 리소스 목록
resources:
  - deployment.yaml
  - service.yaml

# 모든 리소스에 공통 레이블 추가 (선택)
commonLabels:
  app: nginx
  company: myorg
```

### overlays/staging/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# base 디렉토리 참조
resources:
  - ../../base

# base에서 변경할 내용 (patch)
patches:
  - path: deployment-patch.yaml
```

### overlays/staging/deployment-patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2    # base의 1에서 2로 변경
```

> `patches`는 base의 전체 파일을 복사하지 않고 변경할 필드만 작성한다.  
> `apiVersion`과 `kind`는 생략 가능하다 — Kustomize가 자동 인식하지만, 명시하는 것이 권장된다.

---

## 여러 디렉토리 관리

파일이 많아져 서브 디렉토리로 분리했을 때, 디렉토리마다 `kubectl apply`를 반복하지 않아도 된다.

### 단순 구조 (파일 직접 나열)

```
k8s/
├── kustomization.yaml
├── api/
│   ├── deployment.yaml
│   └── service.yaml
└── database/
    ├── deployment.yaml
    └── service.yaml
```

```yaml
# k8s/kustomization.yaml
resources:
  - api/deployment.yaml
  - api/service.yaml
  - database/deployment.yaml
  - database/service.yaml
```

### 권장 구조 (서브 디렉토리별 kustomization.yaml)

디렉토리가 많아지면 루트 파일이 복잡해진다. 각 서브 디렉토리에 `kustomization.yaml`을 두고 루트에서 디렉토리만 참조한다.

```
k8s/
├── kustomization.yaml
├── api/
│   ├── kustomization.yaml   ← api 리소스만 관리
│   ├── deployment.yaml
│   └── service.yaml
├── database/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── cache/
│   └── kustomization.yaml
└── kafka/
    └── kustomization.yaml
```

```yaml
# k8s/kustomization.yaml (루트) — 디렉토리만 참조
resources:
  - api/
  - database/
  - cache/
  - kafka/
```

```yaml
# k8s/database/kustomization.yaml — 해당 디렉토리 리소스만 관리
resources:
  - deployment.yaml
  - service.yaml
```

```bash
# 전체 한 번에 적용
kubectl apply -k k8s/
```

---

## 실무 구조 (멀티 디렉토리 + 환경 분리)

여러 서브 디렉토리 구조에 dev/prod 환경 분리를 결합한 패턴.

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── api/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── database/
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── api-patch.yaml       # replicas: 1
    └── prod/
        ├── kustomization.yaml
        └── api-patch.yaml       # replicas: 5
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base

patches:
  - path: api-patch.yaml
```

```bash
kubectl apply -k overlays/dev/    # 개발 배포
kubectl apply -k overlays/prod/   # 운영 배포
```

> base에 공통 설정을 두고, overlay에서 환경별로 replicas, 이미지 태그, 리소스 제한 등만 덮어쓴다.

overlay에서는 patch 외에 **base에 없는 새 리소스도 추가**할 수 있다.  
예를 들어 prod에만 Grafana를 배포하고 싶다면:

```
overlays/prod/
├── kustomization.yaml
├── api-patch.yaml
└── grafana-deployment.yaml    # prod에만 존재하는 리소스
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
  - grafana-deployment.yaml    # base에 없는 리소스를 여기서 추가

patches:
  - path: api-patch.yaml
```

---

## Components

특정 기능에 필요한 Kubernetes 설정을 **재사용 가능한 블록**으로 묶는 기능.

- **base**: 모든 overlay에 공통 적용
- **overlay 직접 정의**: 해당 overlay에만 적용
- **component**: **일부 overlay에만** 선택적으로 적용 — 복사/붙여넣기 없이 재사용

### 언제 쓰는가

예시: 앱을 dev / premium / self-hosted 세 가지로 배포할 때

| 기능 | dev | premium | self-hosted |
|---|---|---|---|
| 캐싱 (Redis) | X | O | O |
| 외부 DB (Postgres) | O | O | X |

캐싱 설정을 base에 두면 dev까지 포함되고, overlay마다 복사하면 중복된다.  
→ component로 분리하고 필요한 overlay에서만 import.

### 디렉토리 구조

```
k8s/
├── base/
│   ├── kustomization.yaml
│   └── api-deployment.yaml
├── components/
│   ├── caching/
│   │   ├── kustomization.yaml   ← kind: Component
│   │   └── redis-deployment.yaml
│   └── database/
│       ├── kustomization.yaml   ← kind: Component
│       ├── postgres-deployment.yaml
│       └── deployment-patch.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml   ← database component만 import
    ├── premium/
    │   └── kustomization.yaml   ← caching + database 둘 다 import
    └── self-hosted/
        └── kustomization.yaml   ← caching component만 import
```

### Component kustomization.yaml

일반 `kustomization.yaml`과 달리 `apiVersion`과 `kind`가 다르다.

```yaml
# components/database/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component                              # Kustomization이 아닌 Component

resources:
  - postgres-deployment.yaml

secretGenerator:
  - name: db-secret
    literals:
      - password=mysecret

patches:
  - path: deployment-patch.yaml    # base의 api-deployment에 환경변수 주입
```

```yaml
# components/database/deployment-patch.yaml (Strategic Merge)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
        - name: api
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
```

### Overlay에서 import

`components` 필드에 경로를 지정한다.

```yaml
# overlays/dev/kustomization.yaml
resources:
  - ../../base

components:
  - ../../components/database    # database component만 적용
```

```yaml
# overlays/premium/kustomization.yaml
resources:
  - ../../base

components:
  - ../../components/caching     # 두 component 모두 적용
  - ../../components/database
```

---

## Patches

Common Transformer가 **모든 리소스**에 일괄 적용된다면, Patch는 `kustomization.yaml`의 `patches` 필드를 사용해 **특정 리소스만** 정밀하게 변경한다.

두 가지 방식이 있으며, 어느 쪽이든 `patches` 아래에 작성한다.

| 방식 | 특징 |
|---|---|
| **JSON 6902** | `op` / `path` / `value`로 특정 필드를 명시적으로 변경 — 배열 조작에 유리 |
| **Strategic Merge** | 일반 Kubernetes YAML처럼 작성 후 병합 — 가독성 좋음 |

---

### JSON 6902

`op`(작업), `path`(경로), `value`(값) 세 가지를 지정한다.

| op | 설명 |
|---|---|
| `replace` | 기존 값을 새 값으로 교체 |
| `add` | 필드 또는 배열 항목 추가 |
| `remove` | 필드 또는 배열 항목 제거 (`value` 불필요) |

`path`는 YAML 트리를 `/`로 구분해 내려간다.

```
metadata:
  name: api                          →  /metadata/name
spec:
  replicas: 1                        →  /spec/replicas
  template:
    metadata:
      labels:
        component: api               →  /spec/template/metadata/labels/component
```

#### replace — 값 교체

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/template/metadata/labels/component
        value: web
```

#### add — 키 추가

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: add
        path: /spec/template/metadata/labels/org
        value: kodekloud
```

#### remove — 키 삭제 (`value` 불필요)

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: remove
        path: /spec/template/metadata/labels/org
```

`target`에는 `kind`, `name`, `namespace`, `version`, `labelSelector`, `annotationSelector` 등을 조합해 사용할 수 있다.

---

### Strategic Merge

일반 Kubernetes YAML 형식으로 작성해 기존 리소스와 병합한다.  
`metadata.name`으로 대상을 식별하고, **변경할 필드만** 작성한다.  
키를 삭제할 때는 값을 `null`로 지정한다.

```yaml
# kustomization.yaml
patches:
  - path: label-patch.yaml
```

```yaml
# label-patch.yaml — 키 교체/추가
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        component: web      # 교체: api → web
        org: kodekloud      # 추가
```

```yaml
# label-patch.yaml — 키 삭제
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: null           # null로 지정하면 삭제
```

---

### 리스트(배열) 조작

컨테이너 목록처럼 리스트 형태의 필드를 변경하는 방법이다.

#### JSON 6902 — 인덱스로 항목 지정

리스트의 첫 번째 항목은 인덱스 `0`, 두 번째는 `1`.  
맨 끝에 추가할 때는 인덱스 대신 `-`를 사용한다.

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      # 교체: 첫 번째 컨테이너의 이름과 이미지 변경
      - op: replace
        path: /spec/template/spec/containers/0/name
        value: haproxy
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: haproxy

      # 추가: 맨 끝에 새 컨테이너 추가 (-), 특정 위치는 인덱스 지정
      - op: add
        path: /spec/template/spec/containers/-
        value:
          name: haproxy
          image: haproxy

      # 삭제: 두 번째 컨테이너 제거 (value 불필요)
      - op: remove
        path: /spec/template/spec/containers/1
```

#### Strategic Merge — 이름으로 항목 지정

추가는 새 항목을 나열하면 자동 병합.  
삭제는 `$patch: delete` 디렉티브를 사용한다.

```yaml
# patch.yaml — 컨테이너 추가
spec:
  template:
    spec:
      containers:
        - name: haproxy     # 새 컨테이너 추가
          image: haproxy
```

```yaml
# patch.yaml — 컨테이너 삭제
spec:
  template:
    spec:
      containers:
        - name: database
          $patch: delete    # name으로 대상을 찾아 삭제
```

> JSON 6902는 인덱스 기반이라 배열 순서에 의존하고,  
> Strategic Merge는 이름 기반이라 순서에 무관하다 — 리스트 삭제에는 Strategic Merge가 더 안전하다.

---

## Common Transformations

`kustomization.yaml`에 선언하면 관리 중인 **모든 리소스에 일괄 적용**된다.

| 트랜스포머 | 설명 |
|---|---|
| `commonLabels` | 모든 리소스에 공통 레이블 추가 |
| `commonAnnotations` | 모든 리소스에 공통 어노테이션 추가 |
| `namespace` | 모든 리소스를 특정 네임스페이스에 배치 |
| `namePrefix` / `nameSuffix` | 모든 리소스 이름에 접두사/접미사 추가 |

```yaml
# kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  org: mycompany
  env: prod

commonAnnotations:
  maintainer: devops-team

namespace: production

namePrefix: prod-
nameSuffix: -v1
```

위 설정을 적용하면 `metadata.name: nginx-deployment`가 `prod-nginx-deployment-v1`로 변경되고, 레이블/어노테이션/네임스페이스가 모든 리소스에 자동으로 추가된다.

---

## Image Transformer

모든 리소스에서 특정 이미지를 찾아 이름 또는 태그를 변경한다.

```yaml
# kustomization.yaml
resources:
  - deployment.yaml

images:
  # 이미지 이름 변경
  - name: nginx          # 교체할 대상 이미지
    newName: haproxy     # 새 이미지 이름

  # 태그만 변경
  - name: nginx
    newTag: "2.4"

  # 이름과 태그 동시 변경
  - name: nginx
    newName: haproxy
    newTag: "2.4"        # 결과: haproxy:2.4
```

> `name`은 컨테이너 이름이 아닌 **이미지 이름**을 참조한다.  
> `newTag`는 반드시 **문자열**로 작성해야 한다 — `"4.2"` (O), `4.2` (X).  
> overlay에서 환경별로 이미지 태그(버전)를 다르게 지정할 때 유용하다.

---

## 사용법

### 방법 1: kubectl kustomize (미리보기, 권장)

```bash
# 최종 매니페스트 미리 확인 (클러스터에 적용 안 됨)
kubectl kustomize overlays/prod/
```

### 방법 2: kubectl apply -k (바로 적용, 권장)

```bash
kubectl apply -k overlays/dev/
kubectl apply -k overlays/staging/
kubectl apply -k overlays/prod/
```

### 리소스 삭제

```bash
kubectl delete -k overlays/prod/
```

---

## Helm + Kustomize 함께 사용하기

Helm과 Kustomize는 선택이 아닌 **조합**이 가능하다.  
Helm이 패키지 배포를 담당하고, Kustomize가 환경별 후처리를 담당한다.

```
Helm (패키지 관리)
  → 매니페스트 생성
    → Kustomize (환경별 커스터마이징)
      → kubectl apply
```

### 방법 1: helm template + Kustomize 파이프

```bash
helm template my-app bitnami/wordpress | kubectl kustomize - | kubectl apply -f -
```

### 방법 2: kustomization.yaml에서 Helm Chart 직접 호출

```yaml
# kustomization.yaml
helmCharts:
  - name: wordpress
    repo: https://charts.bitnami.com/bitnami
    version: 15.x.x
    releaseName: my-site
    valuesFile: values.yaml

patches:
  - path: prod-patch.yaml    # 환경별 추가 변경
```

### 역할 분담 기준

| 역할 | 도구 |
|---|---|
| 외부 패키지 설치 (nginx-ingress, cert-manager 등) | Helm |
| 자체 개발 앱의 환경별 설정 분리 | Kustomize |
| 외부 패키지의 환경별 세부 조정 | Helm values + Kustomize patch |

> Helm으로 패키지를 가져오고, Kustomize로 환경별 차이를 관리하는 조합이 실무에서 가장 유연하다.

### 디렉토리 구조 예시

```
k8s/
├── base/                          # 자체 앱 공통 설정
│   ├── kustomization.yaml
│   ├── api/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── frontend/
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml     # Helm Chart + base + dev patch
    │   ├── mariadb-values.yaml    # dev용 DB 설정
    │   ├── nginx-values.yaml      # dev용 ingress 설정
    │   └── dev-patch.yaml         # replicas: 1 등
    └── prod/
        ├── kustomization.yaml     # Helm Chart + base + prod patch
        ├── mariadb-values.yaml    # prod용 DB 설정
        ├── nginx-values.yaml      # prod용 ingress 설정
        └── prod-patch.yaml        # replicas: 5 등
```

```yaml
# overlays/prod/kustomization.yaml
helmCharts:
  - name: mariadb-operator
    repo: https://helm.mariadb.com/mariadb-operator
    version: 0.x.x
    valuesFile: mariadb-values.yaml
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.x.x
    valuesFile: nginx-values.yaml

resources:
  - ../../base                # 자체 앱 공통 설정 참조

patches:
  - path: prod-patch.yaml     # prod 환경별 조정
```

```bash
kubectl apply -k overlays/dev/    # 개발 환경 전체 배포
kubectl apply -k overlays/prod/   # 운영 환경 전체 배포
```

---

## Helm vs Kustomize

| 항목 | Helm | Kustomize |
|---|---|---|
| 방식 | 템플릿 언어 사용 | 순수 YAML |
| 학습 곡선 | 높음 (템플릿 문법 필요) | 낮음 |
| 가독성 | 복잡한 Chart는 읽기 어려움 | 항상 일반 YAML |
| 패키지 관리 | 가능 (Artifact Hub) | 불가 |
| 환경별 설정 | values.yaml 오버라이드 | Overlay |
| 설치 필요 | 별도 설치 | kubectl 내장 |

> 단순한 환경별 설정 분리는 Kustomize,  
> 패키지 배포/버전 관리가 필요하면 Helm이 적합하다.
