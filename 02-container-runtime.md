# Container Runtime & CLI 도구

## 역사적 배경

```
초기: Docker만 존재 → Kubernetes가 Docker 전용으로 설계 (강하게 결합)
       ↓
다른 런타임(rocket 등) 등장 → Kubernetes가 CRI(Container Runtime Interface) 도입
       ↓
Docker는 CRI 이전에 만들어져 CRI 미지원 → Dockershim으로 임시 지원
       ↓
v1.24: Dockershim 완전 제거 → Docker 런타임 지원 종료
```

> Docker로 빌드된 이미지는 OCI Image Spec을 따르므로 containerd에서 계속 동작한다.

---

## OCI (Open Container Initiative)

Kubernetes가 다양한 런타임을 지원하기 위해 도입한 표준

| 스펙 | 정의 |
|---|---|
| **Image Spec** | 컨테이너 이미지를 빌드하는 방법의 표준 |
| **Runtime Spec** | 컨테이너 런타임을 개발하는 방법의 표준 |

OCI 표준을 준수하면 누구든 CRI를 통해 Kubernetes와 연동 가능하다.

---

## Container Runtime 비교

| 런타임 | CRI 지원 | 특징 |
|---|---|---|
| **Docker** | X (Dockershim 필요) | Kubernetes v1.24부터 지원 제거 |
| **containerd** | O | Docker에서 분리된 독립 프로젝트, CNCF 졸업 |
| **CRI-O** | O | OCI 표준 기반 경량 런타임 |

---

## CLI 도구 비교

| 도구 | 제작 주체 | 대상 런타임 | 용도 |
|---|---|---|---|
| **ctr** | containerd 커뮤니티 | containerd 전용 | 디버깅 전용 (기능 제한적) |
| **nerdctl** | containerd 커뮤니티 | containerd 전용 | 일반 목적 (Docker CLI와 거의 동일) |
| **crictl** | Kubernetes 커뮤니티 | 모든 CRI 호환 런타임 | 디버깅 전용 |

### ctr
- containerd 설치 시 함께 제공
- 기능이 매우 제한적, 프로덕션 환경 사용 비권장

```bash
ctr images pull docker.io/library/redis:latest
ctr run docker.io/library/redis:latest redis
```

### nerdctl (권장)
- Docker CLI와 거의 동일한 사용법 → `docker` → `nerdctl` 로 대체
- containerd의 신규 기능 선제 지원 (암호화 이미지, lazy pulling, P2P 배포, 이미지 서명 등)

```bash
nerdctl run nginx
nerdctl run -p 8080:80 nginx
```

### crictl
- Kubernetes 관점의 디버깅 도구
- 컨테이너 생성 가능하지만 **kubelet이 감지 후 삭제**하므로 생성 목적으로 사용 금지
- 기존 `docker` 명령을 `crictl`로 대체하여 트러블슈팅에 활용

```bash
crictl images            # 이미지 목록
crictl ps                # 컨테이너 목록
crictl exec -it <id> sh  # 컨테이너 접속
crictl logs <id>         # 로그 확인
crictl pods              # 파드 목록 (Docker에는 없는 기능)
```

#### 런타임 엔드포인트 설정

crictl은 기본적으로 아래 순서로 소켓에 연결 시도한다:

```
dockershim → containerd → cri-o → cri-dockerd
```

특정 런타임을 명시하려면:

```bash
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
# 또는
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
```

---

## 요약

```
┌─────────┬──────────────┬──────────────────────────────────────┐
│  도구    │    용도       │              비고                    │
├─────────┼──────────────┼──────────────────────────────────────┤
│ ctr     │ 디버깅        │ 거의 사용 안 함                       │
│ nerdctl │ 일반 목적     │ Docker 대체 CLI, 가장 많이 사용        │
│ crictl  │ 디버깅        │ 트러블슈팅 시 docker 명령 대체로 사용  │
└─────────┴──────────────┴──────────────────────────────────────┘
```

> **실습 환경**: 기존 Docker → containerd 전환. 트러블슈팅 시 `docker` 대신 `crictl` 사용.
