# Admission Controllers

## 역할

인증(Authentication) → 인가(Authorization) 이후, **API 요청이 실제 처리되기 전** 마지막으로 검사·변경하는 단계.

```
kubectl 요청
    │
    ▼
Authentication (인증 — 누구인가?)
    │
    ▼
Authorization (인가 — 권한이 있는가? RBAC)
    │
    ▼
Admission Controllers ← 여기서 추가 검사/변경
    │
    ▼
etcd 저장 & 리소스 생성
```

RBAC으로는 할 수 없는 것들을 처리한다.

| RBAC으로 가능 | Admission Controller가 필요한 것 |
|---|---|
| 특정 사용자의 Pod 생성 허용/거부 | 특정 레지스트리 이미지만 허용 |
| 네임스페이스별 접근 제어 | `latest` 태그 사용 금지 |
| | root 사용자 컨테이너 거부 |
| | metadata에 라벨 강제 추가 |

---

## 주요 내장 Admission Controller

| 컨트롤러 | 동작 |
|---|---|
| `AlwaysPullImages` | Pod 생성 시 항상 이미지를 새로 pull |
| `DefaultStorageClass` | PVC에 storage class 미지정 시 기본값 자동 추가 |
| `EventRateLimit` | API 서버 요청 수 제한 |
| `NamespaceLifecycle` | 존재하지 않는 네임스페이스 요청 거부, default/kube-system/kube-public 삭제 방지 |

> `NamespaceExists`, `NamespaceAutoProvision`은 deprecated → `NamespaceLifecycle`로 통합

---

## Validating vs Mutating

| 종류 | 동작 | 예시 |
|---|---|---|
| **Validating** | 요청을 검사해 허용/거부 | NamespaceLifecycle (없는 namespace 거부) |
| **Mutating** | 요청 내용을 변경 | DefaultStorageClass (storage class 자동 추가) |

> **실행 순서**: Mutating → Validating  
> Mutating이 먼저 실행돼야 그 변경 결과를 Validating이 검사할 수 있다.

---

## 커스텀 Webhook (MutatingWebhook / ValidatingWebhook)

내장 컨트롤러로 부족할 때 **직접 서버를 만들어** 연결할 수 있다.

```
API 요청 → 내장 AC 통과 → Webhook 호출 → 커스텀 서버가 허용/거부 응답
```

### 설정 흐름

1. **Webhook 서버 배포** — 요청을 받아 허용/거부를 판단하는 서버 (Deployment + Service)
2. **Webhook 설정 오브젝트 생성** — 어떤 요청에 대해 어느 서버를 호출할지 정의

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration   # 또는 MutatingWebhookConfiguration
metadata:
  name: my-webhook
webhooks:
  - name: example.com
    clientConfig:
      service:               # 클러스터 내부 서비스로 배포한 경우
        namespace: default
        name: webhook-service
      caBundle: <base64-cert>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
```

> 통신은 반드시 TLS — 서버에 인증서 설정 후 `caBundle`에 CA 인증서를 base64로 제공해야 한다.  
> 시험에서 Webhook 서버 코드를 직접 작성하는 문제는 출제되지 않는다.

---

## 활성화 / 비활성화

```bash
# 현재 활성화된 admission controller 확인 (kubeadm 환경)
kubectl exec -n kube-system kube-apiserver-controlplane -- \
  kube-apiserver -h | grep enable-admission-plugins
```

kube-apiserver manifest 파일(`/etc/kubernetes/manifests/kube-apiserver.yaml`)에서 플래그 수정:

```yaml
- --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
- --disable-admission-plugins=DefaultStorageClass
```
