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
