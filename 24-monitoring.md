# Monitoring

## 모니터링 대상

- **노드**: 노드 수, 상태, CPU/Memory/Network/Disk 사용률
- **Pod**: Pod 수, CPU/Memory 사용량

---

## 모니터링 솔루션

Kubernetes에 내장된 풀 기능 모니터링 솔루션은 없다.

| 솔루션 | 특징 |
|---|---|
| **Metrics Server** | 경량, 인메모리, 기본 메트릭만 제공 |
| Prometheus | 오픈소스, 히스토리 저장, 가장 많이 사용 |
| Elastic Stack | 로그 + 메트릭 통합 |
| Datadog / Dynatrace | 상용 SaaS |

> Heapster는 deprecated → Metrics Server로 대체됨

---

## Metrics Server

### 동작 원리

```
각 노드의 kubelet (cAdvisor 포함)
    │  Pod/노드 성능 메트릭 수집
    ▼
Metrics Server (인메모리 집계)
    │
    ▼
kubectl top 명령어로 조회
```

- **cAdvisor**: kubelet 내부 컴포넌트 — Pod 성능 메트릭을 수집해 API로 노출
- **인메모리 저장** → 재시작 시 데이터 사라짐, 과거 데이터 조회 불가

### 설치

```bash
# minikube
minikube addons enable metrics-server

# 그 외 (GitHub에서 최신 manifest를 받아 Metrics Server 설치)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 조회 명령어

```bash
kubectl top node   # 노드별 CPU/Memory 사용량
kubectl top pod    # Pod별 CPU/Memory 사용량
```

---

## Prometheus + Grafana (kube-prometheus-stack)

Metrics Server는 `kubectl top`과 HPA용 경량 컴포넌트고, 대시보드·알림·장기 데이터 저장이 필요하면 Prometheus + Grafana를 별도로 설치한다.

**kube-prometheus-stack** Helm 차트로 Prometheus + Grafana + AlertManager를 한 번에 설치할 수 있다.

```bash
# Helm repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# monitoring 네임스페이스에 설치
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

```bash
# Grafana 접속 (포트포워딩)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# 기본 계정: admin / prom-operator
```

> Metrics Server와 독립적으로 동작하지만, HPA를 함께 쓰려면 Metrics Server도 설치해야 한다.

---

## 외부 Prometheus + Grafana 연동

회사 클러스터 외부에 Prometheus/Grafana가 이미 운영 중이라면 두 가지 방식으로 수집할 수 있다.

### 방식 1: 외부 Prometheus가 클러스터를 직접 스크래핑 (Pull)

```
외부 Prometheus → (네트워크 접근) → kube-state-metrics / node-exporter
```

클러스터 내부에 Exporter만 설치하고, 외부 Prometheus가 주기적으로 가져간다.

```bash
# kube-state-metrics (Kubernetes 오브젝트 상태 메트릭)
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace monitoring --create-namespace

# node-exporter (노드 CPU/Memory/Disk 메트릭, DaemonSet으로 배포)
helm install node-exporter prometheus-community/prometheus-node-exporter \
  --namespace monitoring
```

**전제 조건 — 반드시 확인 필요:**
- 외부 Prometheus → 클러스터 내부 서비스 포트에 네트워크 접근 가능해야 함 (VPN, 사설망, 방화벽 허용 등)
- kube-state-metrics는 NodePort 또는 LoadBalancer로 외부에 노출 필요
- node-exporter는 각 노드의 9100 포트가 외부에서 접근 가능해야 함
- 네트워크 접근이 안 되면 이 방식은 불가

### 방식 2: 클러스터 내부에서 외부로 Push (Remote Write) ← 권장

```
클러스터 내 Prometheus Agent → remote_write → 외부 Prometheus
```

클러스터 내부에 경량 Prometheus Agent를 두고, 메트릭을 외부로 밀어낸다.  
인바운드 포트를 열 필요 없이 **아웃바운드 연결만** 필요해서 보안상 유리하다.

```yaml
# Prometheus Agent 설정 예시
remoteWrite:
  - url: "http://외부-prometheus-주소:9090/api/v1/write"
```

```bash
# Prometheus Agent 모드로 설치 (풀 기능 Prometheus 불필요)
helm install prometheus-agent prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set prometheusSpec.enableRemoteWriteReceiver=false \
  --set prometheusSpec.mode=agent
```

**전제 조건:**
- 클러스터 → 외부 Prometheus 주소로 아웃바운드 연결 가능해야 함
- 외부 Prometheus에서 `--enable-feature=remote-write-receiver` 옵션 활성화 필요

### 방식 선택 기준

| | Pull (직접 스크래핑) | Push (Remote Write) |
|---|---|---|
| 네트워크 방향 | 인바운드 필요 | 아웃바운드만 필요 |
| 보안 | 포트 노출 필요 | 상대적으로 유리 |
| 구성 복잡도 | 낮음 | 중간 |
| 권장 상황 | 동일 사설망 내 | 방화벽/보안 제약이 있을 때 |
