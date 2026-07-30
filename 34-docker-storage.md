# Docker Storage

## Docker 데이터 저장 경로

```
/var/lib/docker/
├── aufs/
├── containers/   ← 컨테이너 관련 파일
├── image/        ← 이미지 관련 파일
└── volumes/      ← 볼륨 데이터
```

---

## 레이어드 아키텍처

Docker는 이미지를 빌드할 때 Dockerfile의 각 명령어마다 레이어를 생성한다.  
각 레이어는 이전 레이어 대비 **변경분만** 저장한다.

```
Layer 5: ENTRYPOINT 설정        (소용량)
Layer 4: 소스코드 복사           (소용량)
Layer 3: Python 패키지 설치      (소용량)
Layer 2: APT 패키지 설치         (~300MB)
Layer 1: Ubuntu base image      (~120MB)
```

### 레이어 재사용 (캐시)

동일한 레이어는 캐시에서 재사용 → 빌드 속도 향상, 디스크 절약

```
App A: [Ubuntu] [APT] [Python] [소스A] [EntryA]
App B: [Ubuntu] [APT] [Python] [소스B] [EntryB]
              ↑ 처음 3개 레이어 캐시 재사용
```

---

## 이미지 레이어 vs 컨테이너 레이어

```
┌─────────────────────────┐
│  Container Layer (R/W)  │  ← 컨테이너 실행 중 생성되는 데이터
├─────────────────────────┤
│  Image Layer 5  (R/O)   │
│  Image Layer 4  (R/O)   │
│  Image Layer 3  (R/O)   │
│  Image Layer 2  (R/O)   │
│  Image Layer 1  (R/O)   │
└─────────────────────────┘
```

- **이미지 레이어**: 읽기 전용, 여러 컨테이너가 공유
- **컨테이너 레이어**: 읽기/쓰기, 컨테이너 종료 시 삭제

### Copy-on-Write

이미지 레이어의 파일을 수정하면 Docker가 자동으로 컨테이너 레이어에 복사 후 수정한다.  
이미지 원본은 변경되지 않는다.

---

## 볼륨 (데이터 영속성)

컨테이너가 삭제되면 컨테이너 레이어의 데이터도 함께 삭제된다.  
데이터를 유지하려면 볼륨을 사용한다.

### Volume Mount

Docker가 관리하는 볼륨 (`/var/lib/docker/volumes/`)

```bash
# 볼륨 생성
docker volume create data_volume

# 컨테이너에 마운트
docker run -v data_volume:/var/lib/mysql mysql

# 볼륨 미생성 시 자동 생성
docker run -v data_volume2:/var/lib/mysql mysql
```

### Bind Mount

호스트의 임의 경로를 마운트

```bash
docker run -v /data/mysql:/var/lib/mysql mysql
```

### --mount 옵션 (권장)

`-v`는 구식. `--mount`가 더 명시적이어서 권장된다.

```bash
# Bind Mount 예시
docker run \
  --mount type=bind,source=/data/mysql,target=/var/lib/mysql \
  mysql
```

| 파라미터 | 설명 |
|---|---|
| `type` | `volume` 또는 `bind` |
| `source` | 호스트 경로 또는 볼륨 이름 |
| `target` | 컨테이너 내부 경로 |

---

## Storage Driver

레이어드 아키텍처, 컨테이너 레이어 생성, Copy-on-Write 등을 담당한다.

| Storage Driver | 주요 OS |
|---|---|
| overlay2 | Ubuntu (기본값) |
| devicemapper | CentOS / Fedora |
| btrfs / zfs | 특수 환경 |

> Docker가 OS에 맞는 드라이버를 자동 선택한다.
