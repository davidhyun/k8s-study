# kube-apiserver

## 역할

클러스터의 **중앙 관리 허브**. 모든 구성 요소는 kube-apiserver를 통해서만 통신하며, **etcd에 직접 접근할 수 있는 유일한 컴포넌트**다.

---

## 요청 처리 흐름

```
kubectl / REST API
        │
        ▼
  kube-apiserver
  1. 인증 (Authentication)
  2. 유효성 검증 (Validation)
  3. etcd 읽기/쓰기
        │
        ▼
     응답 반환
```

---

## Pod 생성 시 흐름

```
1. 사용자 요청 → kube-apiserver (인증 + 유효성 검증)
2. kube-apiserver → etcd에 Pod 객체 저장 (노드 미배정 상태)
3. scheduler → kube-apiserver 감지 → 적합한 노드 선택 → apiserver에 통보
4. kube-apiserver → etcd 업데이트 → kubelet에 전달
5. kubelet → 컨테이너 런타임에 이미지 배포 지시
6. kubelet → kube-apiserver에 상태 보고 → etcd 업데이트
```

---

## 핵심 책임

- 요청 **인증 및 유효성 검증**
- **etcd** 데이터 읽기/쓰기 (유일한 직접 접근 컴포넌트)
- scheduler, controller-manager, kubelet의 **통신 중계**

---

## 설정 확인 방법

| 배포 방식 | 확인 방법 |
|---|---|
| **kubeadm** | `kubectl get pod -n kube-system` → Pod 정의 파일: `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| **직접 구성** | `/etc/systemd/system/kube-apiserver.service` |
| **공통** | `ps -aux \| grep kube-apiserver` |

---

## 주요 옵션

```bash
--etcd-servers=https://127.0.0.1:2379   # etcd 연결 주소
--advertise-address=<master-ip>          # API 서버 주소
# 그 외 다수의 TLS 인증서 관련 옵션 존재
```
