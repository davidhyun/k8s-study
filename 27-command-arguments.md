# Commands & Arguments

## Docker에서의 개념

컨테이너는 **프로세스가 살아있는 동안만 실행**된다. 프로세스가 종료되면 컨테이너도 종료된다.

### CMD vs ENTRYPOINT

| Dockerfile 지시어 | 역할 | 런타임 파라미터 전달 시 |
|---|---|---|
| `CMD` | 실행할 명령어 전체 정의 | 전달값이 CMD 전체를 **대체** |
| `ENTRYPOINT` | 실행할 실행파일 정의 | 전달값이 ENTRYPOINT 뒤에 **추가** |

```dockerfile
# CMD만 사용
CMD ["sleep", "5"]
# docker run ubuntu-sleeper 10 → sleep 10 (CMD 전체 대체)

# ENTRYPOINT만 사용
ENTRYPOINT ["sleep"]
# docker run ubuntu-sleeper 10 → sleep 10 (파라미터 추가)
# docker run ubuntu-sleeper    → sleep (오류: 인자 없음)

# ENTRYPOINT + CMD 조합 (기본값 설정)
ENTRYPOINT ["sleep"]
CMD ["5"]
# docker run ubuntu-sleeper    → sleep 5 (CMD가 기본값)
# docker run ubuntu-sleeper 10 → sleep 10 (CMD 대체)
```

> JSON 배열 형식으로 작성해야 ENTRYPOINT와 CMD가 올바르게 조합된다.

### ENTRYPOINT 런타임 오버라이드

```bash
docker run --entrypoint sleep2.0 ubuntu-sleeper 10
# → sleep2.0 10
```

---

## Kubernetes Pod에서의 적용

| Docker | Kubernetes Pod spec |
|---|---|
| `ENTRYPOINT` | `command` |
| `CMD` | `args` |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
    - name: ubuntu
      image: ubuntu-sleeper
      command: ["sleep2.0"]   # ENTRYPOINT 오버라이드
      args: ["10"]            # CMD 오버라이드
```

> `command`와 `args` 모두 지정하지 않으면 이미지의 ENTRYPOINT/CMD가 그대로 사용된다.
