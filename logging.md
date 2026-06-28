# Logging

## 명령어

```bash
# Pod 로그 조회
kubectl logs <pod-name>

# 실시간 스트리밍
kubectl logs -f <pod-name>

# Multi-container Pod에서 특정 컨테이너 로그 조회
kubectl logs <pod-name> -c <container-name>
```

> Multi-container Pod에서 컨테이너 이름을 지정하지 않으면 오류 발생 — `-c`로 명시해야 한다.

---

## Docker와 비교

| | Docker | Kubernetes |
|---|---|---|
| 로그 조회 | `docker logs <container-id>` | `kubectl logs <pod-name>` |
| 실시간 스트리밍 | `docker logs -f` | `kubectl logs -f` |

---

## 실무 로그 수집

`kubectl logs`는 단일 Pod 조회용이며, 프로덕션에서는 중앙 집중식 로그 수집 솔루션을 사용한다.

| 솔루션 | 특징 |
|---|---|
| **EFK** (Elasticsearch + Fluentd + Kibana) | 가장 널리 쓰이는 오픈소스 스택 |
| **Loki + Grafana** | 경량, Grafana와 통합 용이 |
| Datadog / Dynatrace | 상용 SaaS |
