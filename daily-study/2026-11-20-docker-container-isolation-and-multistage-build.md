# [2026-11-20] Docker 컨테이너 격리 원리와 멀티스테이지 빌드 — namespace·cgroup부터 이미지 레이어 캐시, 컨테이너 인지 JVM 메모리 설정까지

> **한 줄 핵심 요약**: 컨테이너는 "가벼운 가상머신"이 아니라 **호스트 커널을 공유하는 격리된 프로세스**이고, 격리의 실체는 **namespace(보이는 것)** 와 **cgroup(쓸 수 있는 양)** 두 축입니다. 그래서 컨테이너 안의 JVM은 호스트 메모리를 보지 않고 **cgroup 한도**를 봐야 하는데, 이 인식이 틀어지면 `OOMKilled(137)`이 나고, 멀티스테이지 빌드는 **빌드 도구와 런타임을 분리해 이미지 크기와 공격면을 동시에 줄이는** 수단이며, 그 효과는 대부분 **레이어 순서와 캐시 설계**에서 결정됩니다.

---

## 1. 개념 및 동작 원리

### 1.1 컨테이너는 VM이 아니라 격리된 프로세스다

`docker run`을 하면 커널이 하는 일은 세 가지뿐입니다.

1. **clone(2)** 로 프로세스를 만들면서 **namespace 플래그**를 함께 넘겨 격리된 시야를 준다.
2. 그 프로세스를 **새 cgroup** 에 넣고 CPU·메모리·PIDs·I/O 상한을 건다.
3. `pivot_root`(또는 `chroot`)로 **이미지의 루트 파일시스템**을 루트로 삼는다.

즉 `docker ps`에 보이는 "컨테이너"는 **호스트에서 돌고 있는 평범한 프로세스**입니다. `ps -ef`로 호스트에서 보면 `containerd-shim`의 자식으로 보이고, `/proc/<pid>/cgroup`에 어느 cgroup에 속했는지 적혀 있습니다.

| 항목 | 가상머신 | 컨테이너 |
| --- | --- | --- |
| 격리 경계 | 하이퍼바이저 + **게스트 커널** | 커널이 제공하는 namespace/cgroup |
| 부팅 시간 | 수 초 ~ 수십 초 | 수십 ~ 수백 ms (프로세스 생성 시간) |
| 메모리 오버헤드 | 게스트 OS 전체(수백 MB) | 프로세스 자체 + 이미지 페이지 캐시 |
| 밀도 | 호스트당 수십 개 | 호스트당 수백 개 |
| 격리 강도 | 강함(커널 취약점이 게스트에 영향 없음) | 약함(커널 공유 → 커널 취약점이 곧 탈출 경로) |
| 이미지/배포 단위 | 디스크 이미지(GB) | 레이어 이미지(MB) |

- **격리 강도가 약하다는 점이 실무의 핵심 제약**입니다. 컨테이너 안 root는 호스트 root에 가까울 수 있어서, 비루트 실행·capability 최소화·rootless Docker 같은 보강이 필요합니다(2.6절).
- 반대로 "커널을 공유하므로 가볍다"가 컨테이너의 존재 이유입니다. JVM 애플리케이션은 게스트 OS를 따로 띄울 이유가 거의 없습니다.

### 1.2 namespace — "무엇이 보이는가"를 바꾼다

| namespace | 격리 대상 | 컨테이너에서의 체감 |
| --- | --- | --- |
| **mnt** (mount) | 마운트 테이블 | 컨테이너의 `/`가 이미지 루트, 호스트 `/`는 안 보임 |
| **pid** | 프로세스 ID 공간 | 컨테이너의 메인 프로세스가 **PID 1** |
| **net** (network) | NIC·라우팅 테이블·포트 | 자기 `eth0`과 포트 공간을 가짐 |
| **ipc** | SysV IPC, POSIX 메시지 큐 | 공유 메모리 세그먼트 분리 |
| **uts** | hostname, NIS domain | 컨테이너마다 다른 hostname |
| **user** | UID/GID 매핑 | rootless에서 컨테이너 root를 호스트 일반 사용자에 매핑 |
| **cgroup** | cgroup 트리 시야 | 자기 cgroup만 보임 (v1에서는 호스트 전체가 보이던 문제) |
| **time** | monotonic/boottime clock offset | Linux 5.6+, 실험적 |

```bash
# 컨테이너 안에서는 PID 1인 셸이, 호스트에서는 다른 PID를 가진다
docker run --rm --name ns-demo alpine sh -c 'echo "inside: $$"'
docker inspect -f '{{.State.Pid}}' ns-demo   # 호스트에서 본 PID
lsns -p "$(docker inspect -f '{{.State.Pid}}' ns-demo)"   # 어떤 namespace를 썼는지
```

- **PID 1이 되는 것이 곧 시그널 문제**입니다. PID 1은 커널이 기본 시그널 처리를 하지 않는 특별한 프로세스라, 셸이 PID 1이면 `SIGTERM`을 자식에게 전달하지 않고 좀비를 거두지도 않습니다. `docker stop`이 10초 타임아웃 후 `SIGKILL`로 끝나는 전형적인 원인입니다(2.5절).
- **net namespace가 포트 충돌을 만드는 지점**을 구분해야 합니다. 컨테이너 안 8080은 컨테이너마다 독립이지만, 호스트로 **publish**하는 순간(`-p 8080:8080`) 호스트 포트는 하나뿐입니다. 같은 호스트에 두 컨테이너를 띄우면 두 번째는 `bind: address already in use`로 죽습니다.

### 1.3 cgroup — "얼마나 쓸 수 있는가"를 제한한다

namespace가 시야를 가르고, 실제 자원 상한은 cgroup이 겁니다. Docker/Kubernetes는 `--memory`, `--cpus` 같은 옵션을 cgroup 파일에 그대로 씁니다.

| 자원 | cgroup v1 | cgroup v2 | Docker 옵션 |
| --- | --- | --- | --- |
| 메모리 상한 | `memory.limit_in_bytes` | `memory.max` | `--memory=512m` |
| 메모리 압박 임계 | `memory.soft_limit_in_bytes` | `memory.high` | `--memory-reservation` |
| CPU 할당량 | `cpu.cfs_quota_us` / `cpu.cfs_period_us` | `cpu.max` ("50000 100000") | `--cpus=0.5` |
| CPU 가중치(경합 시) | `cpu.shares` | `cpu.weight` | `--cpu-shares` |
| 프로세스 수 | `pids.max` | `pids.max` | `--pids-limit` |
| 디스크 I/O | `blkio.throttle.*` | `io.max` | `--device-read-bps` 등 |

```bash
# 컨테이너 안에서 자기 한도를 직접 확인한다 (cgroup v2 기준)
docker run --rm --memory=512m --cpus=1.5 eclipse-temurin:21-jre \
  sh -c 'cat /sys/fs/cgroup/memory.max; cat /sys/fs/cgroup/cpu.max'

# 536870912        → 512MiB
# 150000 100000    → 1.5 core
```

- **메모리 상한 초과는 "느려짐"이 아니라 "즉사"** 입니다. cgroup은 스왑 없이 한도를 넘긴 페이지를 회수하려다 실패하면 **OOM killer**를 불러 해당 cgroup의 프로세스를 `SIGKILL`합니다. 컨테이너 종료 코드는 `137`(128 + 9), `docker inspect`의 `.State.OOMKilled`가 `true`입니다.
- **CPU 상한은 죽이지 않고 조인다.** `--cpus=1`인데 스레드 200개가 돌면 스케줄러가 잘게 쪼개 실행하므로 **스레드 수만큼 느려지고 지연(latency)이 폭증**합니다. 이때 JVM이 코어 수를 잘못 알면 스레드 풀·GC 스레드 수가 컨테이너에 비해 과하게 잡힙니다(1.5절).
- **cgroup v1 → v2 전환**은 실무에서 자주 발목을 잡습니다. v2에서는 `memory.stat`의 항목 의미와 `docker stats`의 계산이 조금 달라지고, 일부 모니터링 에이전트가 v1 경로(`/sys/fs/cgroup/memory/...`)를 하드코딩해 있으면 지표가 비어 보입니다. Ubuntu 21.10+, RHEL 9+, 최신 Docker Desktop은 기본이 v2입니다.

### 1.4 이미지는 "레이어의 스택"이다 — OverlayFS와 copy-on-write

이미지 레이어는 읽기 전용이고, 컨테이너는 그 위에 **얇은 쓰기 레이어**를 하나 얹습니다. 최종 파일시스템은 OverlayFS로 합쳐 보여주고, 파일을 수정하면 해당 파일만 쓰기 레이어로 복사됩니다(copy-up).

```text
[읽기 전용] base image layer       (ubuntu/eclipse-temurin)
[읽기 전용] RUN apt-get ... layer
[읽기 전용] COPY application.jar layer
[읽기/쓰기] container layer        ← 컨테이너 삭제 시 사라짐 (docker commit 하지 않는 한)
────────────────────────────────────────
OverlayFS가 위에서 아래로 겹쳐 하나의 / 로 보여줌
```

- **컨테이너의 쓰기 레이어는 휘발성**입니다. DB 데이터를 컨테이너 안에 두면 `docker rm` 한 번에 사라지므로 **볼륨/바인드 마운트**를 씁니다. "컨테이너는 언제든 버려도 되는 존재(immutable infrastructure)"라는 원칙의 근거입니다.
- **레이어는 지울 수 없습니다.** 한 레이어에 파일을 만들고 다음 레이어에서 `rm`하면, 위 레이어에는 "삭제 표시(whiteout)"만 남고 아래 레이어의 바이트는 **이미지에 그대로 남습니다**. 그래서 `apt-get update`와 `install`, `rm -rf /var/lib/apt/lists/*`는 **반드시 한 `RUN`** 에 넣습니다.

```dockerfile
# ❌ 나쁜 예: 40MB가 이미지에 그대로 남는다
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ 좋은 예: 한 레이어 안에서 설치와 정리를 끝낸다
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

- **캐시 키는 "명령 문자열 + 부모 레이어"** 입니다. `RUN`은 명령 텍스트가 같으면 캐시가 맞고, `COPY`/`ADD`는 **복사 대상 파일의 내용 체크섬**까지 봅니다. 그래서 `COPY . .`을 위쪽에 두면 소스 한 줄만 바뀌어도 그 아래 모든 레이어(의존성 다운로드 포함)가 무효화됩니다.
- **캐시 미스는 연쇄됩니다.** 어떤 레이어가 무효가 되면 그 자식 레이어는 전부 다시 만들어야 합니다. 순서를 "변경 빈도가 낮은 것 → 높은 것"으로 정렬하는 것이 빌드 시간의 90%를 좌우합니다.

```text
나쁜 순서                          좋은 순서
COPY . .                           COPY build.gradle settings.gradle ./
RUN ./gradlew dependencies         RUN ./gradlew dependencies
RUN ./gradlew bootJar              COPY src src
                                   RUN ./gradlew bootJar
→ 소스 1줄 수정 = 의존성 재다운로드   → 소스 1줄 수정 = 컴파일만
```

### 1.5 멀티스테이지 빌드 — 빌드 도구를 런타임에 남기지 않는다

JDK, Gradle 캐시, 소스는 **빌드할 때만** 필요합니다. 멀티스테이지 빌드는 이들을 중간 스테이지에 남겨두고 **최종 스테이지에는 산출물만 복사**합니다.

```dockerfile
# ---------- Stage 1: 빌드 ----------
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /workspace
COPY gradlew settings.gradle build.gradle ./
COPY gradle gradle
RUN ./gradlew dependencies --no-daemon
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# ---------- Stage 2: 런타임 ----------
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY --from=builder /workspace/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| 구성 | 대략적인 크기 | 런타임에 남는 것 |
| --- | --- | --- |
| 단일 스테이지 `jdk` + Gradle + 소스 | 600MB ~ 1GB+ | 컴파일러, Gradle 캐시, 소스, 테스트 산출물 |
| 멀티스테이지 `jre` + fat jar | 180 ~ 250MB | JRE + jar |
| 멀티스테이지 `jre-alpine` | 90 ~ 140MB | musl 기반 JRE + jar |
| 멀티스테이지 + **레이어 분리** | 위와 동일(전송량 감소) | jar를 4개 레이어로 쪼개 **변경분만 push/pull** |
| `jlink`/`jpackage` 커스텀 런타임 | 60 ~ 120MB | 필요한 모듈만 포함 (구성 난이도 높음) |

- **크기보다 중요한 효과가 두 가지** 있습니다. ① **공격면 축소**: 컴파일러·Gradle·소스가 런타임 이미지에 없으면 그만큼 취약점 스캔 대상과 유출 위험이 줄어듭니다. ② **전송량 감소**: 이미지 pull은 배포 시간에 직결됩니다.
- **`--from`은 스테이지 이름으로 참조**합니다. `COPY --from=builder`는 이전 스테이지의 파일시스템만 가져오므로, 최종 이미지에는 builder의 레이어가 **한 바이트도 포함되지 않습니다**.
- **주의**: 멀티스테이지는 이미지 크기를 줄이지만 **레이어 수를 줄이지는 않습니다**. 최종 스테이지의 레이어만 남을 뿐입니다.

```dockerfile
# BuildKit의 캐시 마운트 — Gradle 캐시를 레이어가 아닌 빌드 캐시에 둔다
# syntax=docker/dockerfile:1.7
RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    ./gradlew bootJar --no-daemon -x test
```

- `--mount=type=cache`는 캐시 내용을 **이미지 레이어에 넣지 않습니다.** `RUN ./gradlew ...`로 받은 의존성을 레이어로 굽는 것보다 이미지가 작고, 재빌드도 빠릅니다. Docker 23+는 BuildKit이 기본이라 별도 설정 없이 동작합니다(BuildKit 미사용 시 무시됨).
- **CI 캐시 재사용**은 `--cache-from` + `BUILDKIT_INLINE_CACHE=1`(또는 `--cache-to type=registry`)로 처리합니다. 로컬 캐시는 CI 에이전트가 바뀌면 사라지므로, 레지스트리에 캐시를 두는 편이 안정적입니다(2.11절).

### 1.6 Spring Boot 레이어드 jar — 배포 때 바뀌는 부분만 다시 받는다

Spring Boot 2.3+의 `bootJar`는 jar를 **의존성/스냅샷 의존성/로더/애플리케이션** 4개 레이어로 나눠 담습니다. 애플리케이션 코드는 자주 바뀌지만 의존성은 거의 안 바뀌므로, 레이어를 나눠 두면 **재배포 시 마지막 레이어(수 MB)만 전송**됩니다.

```bash
# 익스트랙트 (Spring Boot 3.3+)
java -Djarmode=tools -jar app.jar extract --layers --destination extracted

# Spring Boot 3.2 이하
java -Djarmode=layertools -jar app.jar extract --destination extracted

# 결과
extracted/dependencies/          # 외부 라이브러리 (가장 큼, 거의 안 바뀜)
extracted/spring-boot-loader/    # 로더 클래스
extracted/snapshot-dependencies/ # SNAPSHOT 의존성 (자주 바뀜)
extracted/application/           # 우리 클래스·리소스 (매 배포마다 바뀜)
```

```dockerfile
# ---------- Stage 2: 레이어 추출 ----------
FROM eclipse-temurin:21-jdk-jammy AS extractor
WORKDIR /workspace
COPY --from=builder /workspace/build/libs/*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --destination extracted

# ---------- Stage 3: 런타임 ----------
FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app

# 의존성은 레이어로 분리 → 소스만 바뀐 배포에서 이 레이어는 캐시된다
COPY --from=extractor /workspace/extracted/dependencies/ ./
COPY --from=extractor /workspace/extracted/spring-boot-loader/ ./
COPY --from=extractor /workspace/extracted/snapshot-dependencies/ ./
COPY --from=extractor /workspace/extracted/application/ ./

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

- **`JarLauncher`의 패키지가 버전에 따라 다릅니다.** Spring Boot 3.2+는 `org.springframework.boot.loader.launch.JarLauncher`, 3.0~3.1은 `org.springframework.boot.loader.JarLauncher`입니다. `-jar app.jar`로 실행해도 되지만, 레이어를 풀어 놓았을 때는 `JarLauncher`로 클래스패스를 조립하는 방식이 표준입니다.
- `-x test`로 빌드 단계에서 테스트를 건너뛰는 대신, **CI의 별도 스테이지에서 테스트를 먼저 돌립니다.** 이미지 빌드 안에서 테스트를 돌리면 빌드 시간이 늘고 테스트 실패가 이미지 빌드 실패로 뭉개집니다.
- **Gradle `build.gradle` 레이어 설정**(필요 시 명시):

```groovy
// build.gradle
bootJar {
    layered {
        enabled = true          // 2.3+ 기본값 true. 명시해 두면 의도가 드러난다
    }
}
```

### 1.7 컨테이너 인지 JVM — 힙 크기는 호스트가 아니라 cgroup을 봐야 한다

JDK 10에서 **컨테이너 인식(Container Awareness)** 이 들어오면서(JDK-8146115, 8u191·11.0.1 백포트), JVM이 `memory.max`와 `cpu.max`를 읽어 힙·프로세서 수를 정합니다.

| 시기 | 동작 | 결과 |
| --- | --- | --- |
| JDK 8u191 이전 | 호스트 전체 메모리 기준으로 `-Xmx` = 1/4 | 512MB 컨테이너인데 힙 8GB → **즉시 OOMKilled** |
| JDK 10+ (`UseContainerSupport` 기본 true) | cgroup 한도의 25%(기본 `MaxRAMPercentage`)를 힙 상한으로 | 컨테이너 안에서 합리적 |
| 명시 설정(권장) | `-XX:MaxRAMPercentage=75` | 비힙 영역 여유를 남기고 힙을 최대화 |

```bash
# 실제로 JVM이 무엇을 봤는지 확인 (컨테이너 안에서)
docker run --rm --memory=512m eclipse-temurin:21-jre \
  java -XX:+PrintFlagsFinal -version 2>/dev/null \
  | grep -E 'MaxHeapSize|MaxRAMPercentage|ActiveProcessorCount'
# MaxHeapSize ≈ 128MB (512MB의 25%)
# ActiveProcessorCount = 호스트 코어 수 (--cpus 미지정 시)
```

- **비힙(non-heap) 영역을 반드시 남겨야 합니다.** 컨테이너 메모리에는 힙 외에 **Metaspace, 스레드 스택(스레드당 기본 1MB), JIT 코드 캐시, GC 보조 구조, Direct Buffer(NIO), JNI/glibc malloc, mmap 파일**이 함께 들어갑니다. 힙을 한도의 90~100%로 잡으면 애플리케이션은 멀쩡한데 컨테이너가 죽습니다.
- **`MaxRAMPercentage=75`는 경험적 상한**입니다. 스레드가 매우 많은 서비스(톰캣 200 + 비동기 풀)라면 60~70%, Direct Buffer를 크게 쓰는 서비스(netty, 이미지 처리)라면 더 낮춥니다.
- **`ExitOnOutOfMemoryError`** 를 켜면 `OutOfMemoryError` 발생 시 JVM이 즉시 종료되어 오케스트레이터가 재시작합니다. 힙 OOM은 살아 있는 척 하면서 응답만 실패하는 상태가 더 위험합니다.
- **CPU도 컨테이너 인식 대상**입니다. `--cpus=1.5`면 JVM은 `ActiveProcessorCount`를 2로 잡고(올림), 그 값으로 **GC 스레드 수, ForkJoinPool 병렬도, 톰캣 기본 스레드**를 정합니다. `--cpus`를 안 걸면 호스트 64코어를 그대로 봐서 **불필요하게 많은 GC 스레드**가 생기고, cgroup 쿼터에 갇혀 GC pause가 오히려 늘어납니다(11-18 노트의 G1/ZGC 튜닝과 직접 연결되는 지점).
- **명시적으로 고정**하는 편이 재현성이 좋습니다. `-XX:ActiveProcessorCount=4`는 cgroup 설정이 바뀌어도 애플리케이션 동작(풀 크기)을 고정합니다.

### 1.8 엔트리포인트의 형태가 시그널 처리와 종료를 결정한다

| 형태 | 예 | PID 1 | SIGTERM 전달 |
| --- | --- | --- | --- |
| exec form | `ENTRYPOINT ["java","-jar","app.jar"]` | java | ✅ java가 직접 받음 |
| shell form | `ENTRYPOINT java -jar app.jar` | `/bin/sh -c` | ❌ 셸이 자식에게 전달하지 않음 |
| `sh -c`로 감싼 exec form | `ENTRYPOINT ["sh","-c","java -jar app.jar"]` | sh | ❌ 같은 문제 |

- **셸 형태는 `docker stop`이 10초를 기다린 뒤 `SIGKILL`** 로 끝나는 원인입니다. 로그에 종료 훅이 안 찍히고 커넥션이 뚝 끊기면 이걸 먼저 의심합니다.
- 부득이 셸이 필요하면 **tini/dumb-init**을 PID 1로 둡니다: `ENTRYPOINT ["/usr/bin/tini","--","java","-jar","app.jar"]`. `docker run --init`도 같은 효과입니다.
- **Spring Boot 쪽에서도 graceful shutdown을 켜야** 합니다.

```yaml
# application.yml
server:
  shutdown: graceful          # SIGTERM → 새 요청 거부 → 진행 중 요청 완료 대기
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

- 순서를 기억합니다. **LB/Ingress에서 트래픽이 빠진 뒤 → SIGTERM → 유예 시간 동안 처리 완료 → 종료**. 쿠버네티스에서는 readiness probe를 실패시켜 엔드포인트에서 빠지는 시간이 필요하므로 `preStop: sleep 5`를 두고, `terminationGracePeriodSeconds`는 `spring.lifecycle.timeout-per-shutdown-phase`보다 **길게** 잡습니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 Spring Boot 실전 Dockerfile (전체)

```dockerfile
# syntax=docker/dockerfile:1.7
# build/docker/Dockerfile — 멀티스테이지 + 레이어 분리 + 비루트

# ============ 1) 의존성 레이어 ============
FROM eclipse-temurin:21-jdk-jammy AS deps
WORKDIR /workspace
COPY --chmod=0755 gradlew ./
COPY settings.gradle build.gradle ./
COPY gradle gradle
RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    ./gradlew dependencies --no-daemon

# ============ 2) 애플리케이션 빌드 ============
FROM deps AS builder
COPY src src
RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    ./gradlew bootJar --no-daemon -x test

# ============ 3) 레이어 추출 ============
FROM eclipse-temurin:21-jdk-jammy AS extractor
WORKDIR /workspace
COPY --from=builder /workspace/build/libs/*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --destination extracted

# ============ 4) 런타임 ============
FROM eclipse-temurin:21-jre-jammy AS runtime

RUN groupadd --system --gid 1001 app \
 && useradd --system --uid 1001 --gid app --create-home app

WORKDIR /app
COPY --from=extractor --chown=app:app /workspace/extracted/dependencies/ ./
COPY --from=extractor --chown=app:app /workspace/extracted/spring-boot-loader/ ./
COPY --from=extractor --chown=app:app /workspace/extracted/snapshot-dependencies/ ./
COPY --from=extractor --chown=app:app /workspace/extracted/application/ ./

ENV TZ=Asia/Seoul \
    JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70 \
                       -XX:InitialRAMPercentage=40 \
                       -XX:+ExitOnOutOfMemoryError \
                       -XX:+HeapDumpOnOutOfMemoryError \
                       -XX:HeapDumpPath=/tmp/heapdumps \
                       -Djava.security.egd=file:/dev/./urandom"

RUN mkdir -p /tmp/heapdumps && chown app:app /tmp/heapdumps
USER app
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

> `--mount` 플래그는 **`RUN` 바로 뒤, 실행 명령 앞**에만 올 수 있습니다. `RUN chmod +x gradlew && --mount=...` 처럼 명령 뒤에 붙이면 문법 오류입니다. 실행 권한은 `COPY --chmod=0755 gradlew ./`(BuildKit) 또는 별도 `RUN chmod +x gradlew`로 처리합니다.

- `--chown`을 매 `COPY`마다 붙이는 대신, 마지막에 `RUN chown -R app:app /app`을 하면 **이미지에 전체 파일이 한 번 더 복사되어 레이어가 커집니다.** 복사 시점에 소유자를 정하는 편이 낫습니다.
- `JAVA_TOOL_OPTIONS`는 이미지에 굽지 않고 `docker run -e` / 쿠버네티스 env로 주입하는 편이 유연합니다. 다만 기본값을 이미지에 넣어두면 아무 설정 없이도 안전한 상태가 됩니다. 주의: `JAVA_TOOL_OPTIONS`를 쓰면 JVM이 시작 시 `Picked up JAVA_TOOL_OPTIONS: ...` 를 **stderr에 찍습니다**(로그 노이즈).
- `-Djava.security.egd=file:/dev/./urandom`은 최신 JDK에서 거의 불필요합니다(`/dev/urandom` 블로킹 이슈는 Linux 5.6+ getrandom 개선으로 해소). 레거시 관성으로 남아 있는 경우가 많으니 근거를 확인하고 지웁니다.

### 2.2 `.dockerignore` — 빌드 컨텍스트를 줄이는 첫 단계

```text
# .dockerignore
.git
.gitignore
.idea
*.iml
build
out
target
node_modules
**/.DS_Store
*.log
.env
.env.*
docker-compose*.yml
docs
```

- `docker build .`는 **컨텍스트 전체를 데몬으로 전송**합니다. `.git`(수백 MB), `build/`(수백 MB)가 포함되면 빌드 시작 전에 이미 느려지고, `COPY . .`를 쓰는 순간 캐시도 매번 깨집니다.
- `.env`, 인증서, 키가 컨텍스트에 들어가면 `COPY . .`로 **이미지에 구워집니다.** 레이어는 지워도 남으므로 한 번 올라간 유출은 되돌릴 수 없습니다.
- `.dockerignore`는 `!` 부정 패턴으로 예외를 둘 수 있습니다(`build/**` 제외 후 `!build/libs/*.jar` 허용). 다만 패턴이 많아지면 의도가 흐려지므로 **화이트리스트형(`*` 후 필요한 것만 `!`)이 더 안전**합니다.

### 2.3 이미지 크기 줄이기 — 효과 순서

| 방법 | 절감 | 비용/주의 |
| --- | --- | --- |
| 멀티스테이지(빌드 도구 제거) | 수백 MB | 거의 공짜, **가장 먼저 할 일** |
| `jre` 베이스로 전환 | 100~200MB | JRE에 필요한 게 다 있는지 확인(jcmd, keytool 등) |
| `alpine`/`distroless` | 80~150MB | musl vs glibc 차이, 셸·패키지 없음 → 디버깅 난이도↑ |
| 레이어 정리(`apt` 캐시 삭제, 한 `RUN`) | 10~50MB | 순서 실수 시 역효과 |
| 레이어드 jar / 라이브러리 분리 | 크기는 동일, **전송량** 감소 | Dockerfile 복잡도↑ |
| `jlink` 커스텀 런타임 | 60MB+ | `jdeps`로 모듈 분석 필요, 리플렉션 사용 시 누락 위험 |
| `.dockerignore` 정비 | 컨텍스트/캐시 | — |

- **distroless는 장점과 대가가 명확**합니다. 셸·패키지 매니저가 없어 공격면이 최소지만, `docker exec sh`로 들어가 볼 수 없습니다. 대신 `kubectl debug`(ephemeral container)나 디버그 전용 이미지를 준비합니다.
- **alpine 전환 시 musl DNS 이슈**를 기억합니다. musl의 resolver는 `search` 도메인 처리·`ndots` 동작이 glibc와 달라, 쿠버네티스 클러스터에서 간헐적 DNS 실패가 나는 사례가 있습니다. JDK 21은 musl용 빌드(`eclipse-temurin:21-jre-alpine`)가 지원되지만, 폰트·로케일·시간대 데이터가 빠져 있어 `-Duser.timezone`만으로는 부족할 수 있습니다(tzdata 패키지 설치 필요).

### 2.4 `OOMKilled` vs `OutOfMemoryError` — 증상으로 원인을 가른다

| 증상 | 의미 | 확인 | 대응 |
| --- | --- | --- | --- |
| exit code **137**, `OOMKilled: true`, 힙 덤프 없음 | **커널 cgroup OOM** (프로세스 전체 메모리 초과) | `docker inspect -f '{{.State.OOMKilled}}'`, `dmesg \| grep -i oom` | `MaxRAMPercentage` 낮추기, 한도 상향, 비힙 사용 점검 |
| exit code **1**, 로그에 `java.lang.OutOfMemoryError: Java heap space` | **JVM 힙 OOM** | 힙 덤프 분석 | 누수 또는 힙 부족 |
| 로그에 `OutOfMemoryError: Metaspace` | 클래스 로딩 누수(동적 프록시·리로드) | `jcmd VM.metaspace` | `MaxMetaspaceSize`, 리로드 제거 |
| `OutOfMemoryError: Direct buffer memory` | `-XX:MaxDirectMemorySize` 초과 | NIO/Netty 버퍼 | 상한 조정, 버퍼 누수 확인 |
| `unable to create native thread` | 스레드 스택 + 힙 + 메타스페이스 합이 한도 초과 | `pids.max`, 스레드 수 | 스레드 풀 축소, `-Xss` 조정 |
| 137인데 OOMKilled false | `docker stop` 타임아웃 후 SIGKILL | 종료 로그 없음 | 2.5절 시그널/graceful 종료 |

- **137은 "누가 죽였는지"를 말해주지 않습니다.** OOM killer, `docker stop` 타임아웃, 쿠버네티스 eviction, liveness probe 실패가 모두 137로 끝납니다. `.State.OOMKilled`와 종료 직전 로그를 함께 봐야 구분됩니다.
- **힙 덤프는 반드시 컨테이너 밖으로 꺼내야** 합니다. 컨테이너가 죽으면 파일도 사라지므로, `/tmp`를 emptyDir/볼륨으로 마운트하거나 S3 업로드 훅을 둡니다. 덤프 파일 크기가 힙 크기만큼이라 **디스크/메모리를 추가로 먹는다**는 점도 주의합니다(덤프 중 OOM 재발).
- **`HeapDumpOnOutOfMemoryError`만 켜고 `ExitOnOutOfMemoryError`를 안 켜면** 덤프를 쓰는 동안 프로세스가 살아 있어 오케스트레이터가 "정상"으로 판단하고 트래픽을 계속 보냅니다. 둘을 함께 켜는 이유입니다.

### 2.5 종료 경로 점검 — "deploy 때마다 502가 3초씩 난다"

```text
[정상]
1. 오케스트레이터가 엔드포인트에서 제거 (readiness 실패 또는 preStop)
2. SIGTERM → Spring graceful shutdown 시작
3. 진행 중 요청 완료, 커넥션 풀 정리, Kafka 컨슈머 커밋
4. 프로세스 종료(exit 0)

[비정상]
2'. 셸이 PID 1이라 SIGTERM이 java에 전달되지 않음
3'. 10초(기본 stop timeout) 동안 아무 일 없음
4'. SIGKILL → exit 137, 진행 중 요청 유실
```

- 체크: `ENTRYPOINT`가 **exec form**인가, `server.shutdown=graceful`인가, `terminationGracePeriodSeconds`/`stop_grace_period`가 셧다운 타임아웃보다 **긴가**, `preStop` 유예가 있는가.
- **Kafka 컨슈머**가 있는 서비스는 종료 시 `consumer.wakeup()`과 오프셋 커밋이 끝나야 합니다(11-16 노트). 강제 종료되면 **재처리(중복 소비)** 가 발생하므로, 셧다운 훅에서 컨슈머를 먼저 멈추는 순서를 지킵니다.
- **DB 커넥션 풀**도 마찬가지입니다. HikariCP는 종료 시 커넥션을 닫고, 그 전에 진행 중 트랜잭션이 끝나야 합니다. 트랜잭션 타임아웃 > 셧다운 타임아웃이면 종료 중 `TransactionTimedOutException`이 쏟아집니다.
- **롤링 배포 중 502의 가장 흔한 원인**은 셧다운 자체가 아니라 "LB가 아직 이 인스턴스로 보내는데 서버는 이미 요청을 거부"하는 창입니다. readiness를 먼저 내리고, 유예 후 SIGTERM이 오는 순서를 오케스트레이터에 맡기되, 안 되면 `preStop sleep 5~10`으로 강제합니다.

### 2.6 보안 기본값 — root가 아닌 사용자, 최소 capability

```dockerfile
# 비루트 + 읽기 전용 루트 + capability 제거
USER app
```

```bash
docker run --rm \
  --read-only --tmpfs /tmp:size=64m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -p 8080:8080 my-app:1.0.0
```

- **`--read-only`** 는 쓰기 레이어를 막아 컨테이너 침해 후 파일 변조를 어렵게 합니다. 대신 애플리케이션이 쓰는 경로(`/tmp`, 로그 디렉터리, 업로드 경로)를 **tmpfs/볼륨으로 명시**해야 합니다. Spring Boot는 기본적으로 `/tmp`에 임시 파일을 만들므로 `--tmpfs /tmp`가 없으면 시작 실패합니다.
- **`no-new-privileges`** 는 setuid 바이너리를 통한 권한 상승을 차단합니다.
- **rootless Docker**는 데몬과 컨테이너를 일반 사용자 namespace에서 돌려 컨테이너 탈출의 피해를 사용자 권한으로 제한합니다. 포트 80/443 바인딩 등 일부 기능이 제약됩니다.
- **`HEALTHCHECK`는 쿠버네티스에서 무시됩니다.** 쿠버네티스는 이미지의 HEALTHCHECK 대신 자체 `livenessProbe`/`readinessProbe`를 씁니다. docker/compose 환경에서만 유효하다는 점을 기억합니다.

```yaml
# docker-compose.yml — healthcheck는 compose에서만 동작
services:
  app:
    build:
      context: .
      dockerfile: build/docker/Dockerfile
    ports: ["8080:8080"]
    environment:
      SPRING_PROFILES_ACTIVE: prod
      JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=70 -XX:+ExitOnOutOfMemoryError"
    deploy:
      resources:
        limits: { cpus: "2.0", memory: 1g }
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:8080/actuator/health/readiness || exit 1"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 40s      # JVM 워밍업 동안의 실패는 세지 않는다
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root
      MYSQL_DATABASE: app
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
      interval: 5s
      retries: 20

volumes:
  mysql-data:
secrets:
  mysql_root:
    file: ./secrets/mysql_root.txt
```

- **`start_period`를 반드시 둡니다.** JVM 애플리케이션은 기동에 20~60초가 걸릴 수 있고, 이 동안 healthcheck가 실패하면 `unhealthy`로 판정되어 재시작 루프에 빠집니다.
- **`depends_on`의 `service_healthy`는 Compose v2 기능**이고, 쿠버네티스에는 이 개념이 없습니다. 쿠버네티스에서는 **애플리케이션이 DB 연결 실패를 견디도록**(재시도, `spring.datasource.hikari.initialization-fail-timeout` 조정, readiness에서 DB 제외) 설계하는 것이 정석입니다. 기동 순서를 인프라에 의존하지 않습니다.
- **`curl`이 이미지에 없는 경우가 흔합니다.** `eclipse-temurin:jre-jammy` 같은 슬림 베이스에는 `curl`이 기본 포함되지 않습니다. healthcheck를 쓰려면 `apt-get install -y curl`(레이어 증가) 또는 **Java 기반 체크**(actuator를 호출하는 작은 클래스)를 씁니다. 가장 깔끔한 건 쿠버네티스 httpGet probe를 쓰고 이미지 HEALTHCHECK를 두지 않는 것입니다. `docker run --rm image which curl`로 실제 존재 여부를 먼저 확인합니다.

### 2.7 CPU 한도와 스레드 풀 — "코어 1개인데 풀은 200"

```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 50                 # --cpus=2 환경에서 200은 과하다
      min-spare: 10
    accept-count: 100
  shutdown: graceful
spring:
  datasource:
    hikari:
      maximum-pool-size: 10   # 인스턴스 수 × pool ≤ DB max_connections
      connection-timeout: 3000
```

- **톰캣 기본 `max-threads=200`** 은 컨테이너에 그대로 두면 안 됩니다. CPU 1~2코어에 스레드 200개면 컨텍스트 스위칭과 GC 경합으로 p99 지연이 급등합니다.
- **HikariCP 기본 10**도 인스턴스를 늘리면 곱해집니다. `인스턴스 5 × pool 10 = 50`이 MySQL `max_connections`(기본 151) 안에 들어와야 합니다. 오토스케일링까지 고려하면 **총량 상한**을 정하고 인스턴스당 풀을 역산합니다.
- **`ActiveProcessorCount`가 톰캣 스레드를 자동으로 줄여주지는 않습니다.** JVM은 병렬 GC·ForkJoinPool에만 영향을 주고, 톰캣 `max-threads`는 설정값 그대로입니다. 명시적으로 맞춰야 합니다.

### 2.8 비밀값 — `ARG`/`ENV`에 넣으면 이미지에 남는다

```dockerfile
# ❌ 레이어 히스토리에 그대로 남는다
ARG NEXUS_PASSWORD
RUN curl -u "deploy:${NEXUS_PASSWORD}" https://nexus.internal/...

# ✅ BuildKit secret 마운트 — 레이어에 기록되지 않는다
RUN --mount=type=secret,id=nexus_pw \
    NEXUS_PW="$(cat /run/secrets/nexus_pw)" && \
    curl -u "deploy:${NEXUS_PW}" https://nexus.internal/...
```

```bash
docker build --secret id=nexus_pw,src=./secrets/nexus_pw -t my-app:1.0.0 .
```

- **`ARG`는 `docker history`에 값이 남습니다.** `ENV`도 마찬가지이며 `docker inspect`로 그대로 보입니다. 프라이빗 레지스트리 토큰, GPG 키, SSH 키를 `ARG`/`ENV`로 넘기는 것은 유출입니다.
- **`--mount=type=secret`은 빌드 중에만** 파일을 붙이고 레이어에 남기지 않습니다. `~/.ssh`나 `settings.xml`을 복사하는 패턴도 secret + cache mount로 대체할 수 있습니다.
- **`COPY . .` 뒤에 `.env`를 지우는 것도 무효**입니다(레이어에 남음). `.dockerignore`로 애초에 컨텍스트에서 빼야 합니다.
- 런타임 비밀값(DB 비밀번호, JWT 시크릿)은 이미지가 아니라 **환경변수 주입 또는 파일 마운트(Vault, k8s Secret)** 로 넣고, `application.yml`에 기본값을 두지 않습니다. 환경변수도 `/proc/<pid>/environ`로 노출될 수 있으므로 파일 마운트가 더 안전합니다.

### 2.9 재현성 — 태그는 움직이고 digest는 움직이지 않는다

```dockerfile
# 움직이는 태그: 오늘과 내일이 다른 이미지
FROM eclipse-temurin:21-jre-jammy

# 고정 digest: 항상 같은 이미지
FROM eclipse-temurin:21-jre-jammy@sha256:9f2b8... 
```

- `latest`와 `21-jre-jammy`는 **같은 태그가 다른 digest를 가리킬 수 있습니다.** 어제 통과한 빌드가 오늘 깨지는 "환경 탓"의 대표 원인입니다. 프로덕션은 digest 고정, 개발은 태그 + 정기 갱신(Dependabot/Renovate)이 실무적 절충입니다.
- **`--pull`** 은 베이스 이미지 최신 digest를 강제로 받습니다. 보안 패치를 반영하려면 CI에서 주기적으로 `--pull --no-cache` 빌드를 돌립니다.
- **`apt-get install`은 버전을 고정하지 않으면 재현 불가**입니다. 필요한 경우 `curl=7.81.0-1ubuntu1.15`처럼 핀을 걸되, 보안 업데이트가 막히므로 "핀 + 정기 갱신" 정책을 함께 둡니다.

### 2.10 로그와 관측 — 파일이 아니라 stdout으로

- 컨테이너 안에서 파일로 로그를 쓰면 **`docker logs`에 안 보이고**, 컨테이너가 사라지면 로그도 사라집니다. 표준은 **stdout/stderr + 12-factor**이고, 수집은 Docker log driver(json-file) 또는 사이드카/Fluent Bit이 담당합니다.
- **json-file 드라이버는 기본에 로테이션이 없습니다.** `/var/lib/docker/containers/*/*-json.log`가 디스크를 채워 호스트가 죽는 사고가 흔합니다. `--log-opt max-size=50m --log-opt max-file=5`를 기본값으로 둡니다.
- **JVM 로그에 타임스탬프와 시간대를 고정**합니다. 컨테이너는 기본 UTC이므로, `TZ=Asia/Seoul`(또는 `-Duser.timezone=Asia/Seoul`)을 넣지 않으면 로그 시각이 9시간 어긋나 장애 분석이 꼬입니다. **DB 저장 시각은 UTC, 표시만 KST**로 통일하는 편이 안전합니다.
- **`docker stats`의 MEM USAGE에는 페이지 캐시가 포함**됩니다. 파일을 많이 읽는 컨테이너는 실제 힙이 작아도 수치가 커 보입니다. cgroup v2에서는 `memory.stat`의 `anon`(실제 힙/스택)과 `file`(캐시)을 나눠 봅니다.

### 2.11 CI(Jenkins)에서의 이미지 빌드 — 캐시와 권한

- **DinD(Docker-in-Docker) vs 소켓 마운트**: `-v /var/run/docker.sock`은 호스트 Docker 데몬을 빌려 쓰므로 편하지만 **컨테이너가 호스트 root 권한을 갖는 것과 같습니다**. 공유 Jenkins에서는 DinD나 rootless(Buildah/Kaniko)가 안전합니다.
- **캐시는 레지스트리에 둡니다**: `docker buildx build --cache-to type=registry,ref=registry/app:cache --cache-from type=registry,ref=registry/app:cache`. 에이전트가 매번 새로 뜨는 환경에서 로컬 캐시는 의미가 없습니다.
- **빌드와 푸시를 분리**합니다: PR 빌드는 `--load`로 로컬에만, main 브랜치는 `--push`. 태그는 `git SHA` + `semver` 두 개를 붙여 추적성과 롤백을 동시에 확보합니다.
- **빌드 실패 시 이미지가 남지 않게** `docker builder prune --filter until=168h` 같은 정리 정책을 둡니다. 빌드 캐시는 빠르게 수십 GB가 됩니다.

### 2.12 자주 하는 실수 체크리스트

- [ ] `COPY . .`를 의존성 설치보다 **위**에 두지 않았는가(캐시 전멸)
- [ ] `ENTRYPOINT`가 **exec form**인가(시그널)
- [ ] `--memory` 한도와 `MaxRAMPercentage`가 **함께** 설정됐는가
- [ ] 비힙 여유(25~30%)를 남겼는가
- [ ] `OOMKilled` 확인 없이 "메모리 늘리기"부터 하지 않았는가
- [ ] `.dockerignore`에 `.git`, `build/`, `.env`가 있는가
- [ ] 최종 스테이지에 **JDK·Gradle·소스**가 없는가(`docker run image ls /`로 확인)
- [ ] `HEALTHCHECK`/probe가 **`start_period`** 를 갖는가
- [ ] `server.shutdown=graceful` + `terminationGracePeriodSeconds` 정합
- [ ] 로그가 stdout으로 나가는가, 로테이션이 걸렸는가
- [ ] 비밀값이 `ARG`/`ENV`/이미지 레이어에 없는가(`docker history --no-trunc`)
- [ ] 베이스 이미지가 digest로 고정됐는가
- [ ] 비루트 사용자로 실행되는가
- [ ] 톰캣 스레드·Hikari 풀이 **CPU/인스턴스 수와 정합**인가
- [ ] DB 데이터가 **볼륨**에 있는가(쓰기 레이어 아님)

```bash
# 체크리스트를 명령으로 확인
docker image history --no-trunc my-app:1.0.0 | head -20
docker run --rm my-app:1.0.0 sh -c 'ls /; which javac gradle'   # 없어야 정상
docker inspect -f '{{.Config.User}} {{.Config.Entrypoint}}' my-app:1.0.0
docker run --rm --memory=256m --cpus=1 my-app:1.0.0   # 한도 안에서 기동하는지
docker stats --no-stream
```

### 2.13 검증 시나리오

| 시나리오 | 기대 | 확인 방법 |
| --- | --- | --- |
| 메모리 128m로 기동 | 즉시 OOMKilled 또는 힙 OOM 로그 | `docker inspect -f '{{.State.OOMKilled}}'` |
| `--cpus=1`에서 JVM 코어 수 | `ActiveProcessorCount` 1~2 | `-XX:+PrintFlagsFinal` |
| `docker stop` | exit 0, 종료 로그에 graceful shutdown | `docker events`, 로그 |
| 이미지 크기 | 멀티스테이지 전후 비교 | `docker image ls` |
| 재빌드 시간(소스만 수정) | 의존성 레이어 캐시 히트 | `docker build` 출력의 `CACHED` |
| 레이어드 jar 재배포 | dependencies 레이어 `CACHED` | 빌드 로그 |
| 비루트 | `id` 결과가 1001 | `docker run --rm image id` |
| 읽기 전용 루트 | 기동 성공(쓰기 경로는 tmpfs) | `--read-only --tmpfs /tmp` |
| 비밀값 미포함 | 히스토리에 평문 없음 | `docker history --no-trunc \| grep -i pass` |

---

## 3. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. 컨테이너와 가상머신의 차이를 설명해 보세요.**
  - **A.** "VM은 하이퍼바이저 위에 게스트 커널을 포함한 OS를 통째로 띄우고, 컨테이너는 호스트 커널을 공유한 채 namespace로 시야를 격리하고 cgroup으로 자원을 제한한 프로세스입니다. 그래서 기동이 ms 단위이고 밀도가 높지만, 커널을 공유하므로 커널 취약점이 곧 격리 탈출 경로가 됩니다."

- **Q. 컨테이너 격리는 무엇으로 구현되나요?**
  - **A.** "두 축입니다. namespace는 mount·PID·network·IPC·UTS·user·cgroup을 격리해 '무엇이 보이는지'를 바꾸고, cgroup은 CPU·메모리·PIDs·I/O 상한으로 '얼마나 쓸 수 있는지'를 제한합니다. 루트 파일시스템은 pivot_root로 이미지 레이어를 루트로 삼습니다."

- **Q. 컨테이너 안의 JVM 힙은 어떻게 잡아야 하나요?**
  - **A.** "JDK 10부터 컨테이너 인식이 기본이라 `-XX:MaxRAMPercentage`로 cgroup 한도 대비 비율을 잡습니다. 기본값 25%는 너무 보수적이고 100%는 비힙 때문에 죽으므로 60~75%가 실무적입니다. Metaspace, 스레드 스택, 코드 캐시, Direct Buffer를 함께 고려해야 합니다."

- **Q. `OOMKilled`와 `OutOfMemoryError`는 어떻게 다른가요?**
  - **A.** "OOMKilled는 커널 cgroup OOM으로 **프로세스 전체**가 SIGKILL당해 exit 137이 되고 힙 덤프가 없습니다. OutOfMemoryError는 JVM 내부에서 힙·메타스페이스 등 특정 영역이 부족해 발생하는 예외로, 프로세스는 로그를 남기고 종료합니다. 원인과 대응이 완전히 다릅니다."

- **Q. 멀티스테이지 빌드의 장점은?**
  - **A.** "빌드에만 필요한 JDK·Gradle·소스를 최종 이미지에서 제거합니다. 크기가 수백 MB 줄고, 컴파일러와 소스가 없으니 공격면과 유출 위험이 줄며, pull 시간이 짧아져 배포가 빨라집니다. `COPY --from=builder`로 산출물만 가져오면 중간 레이어는 최종 이미지에 포함되지 않습니다."

- **Q. 이미지 레이어 캐시는 어떤 규칙으로 무효화되나요?**
  - **A.** "캐시 키는 명령 문자열과 부모 레이어이고, `COPY`/`ADD`는 대상 파일 내용의 체크섬까지 봅니다. 한 레이어가 무효가 되면 자식은 전부 다시 빌드합니다. 그래서 변경 빈도가 낮은 것부터(`build.gradle` → 의존성 → 소스) 배치하고, `COPY . .`는 마지막에 둡니다."

- **Q. 이미지에서 파일을 지웠는데 크기가 줄지 않는 이유는?**
  - **A.** "레이어는 불변이고 삭제는 위 레이어에 whiteout만 남기기 때문입니다. 아래 레이어의 바이트는 그대로 이미지에 존재합니다. 그래서 설치와 정리를 한 `RUN`에 넣어야 하고, 그래도 안 되면 멀티스테이지로 최종 스테이지에서만 파일을 만들거나 squash가 필요합니다."

- **Q. Spring Boot 이미지를 레이어로 나누는 이유는?**
  - **A.** "jar 하나를 통째로 복사하면 코드 한 줄 바뀌어도 수십 MB를 다시 받습니다. `layertools`(3.3+는 `tools`)로 dependencies·loader·snapshot-dependencies·application으로 나누면, 매 배포에서 실제로 바뀌는 application 레이어만 전송됩니다. 배포 시간이 줄고 레지스트리 부하도 줄어듭니다."

- **Q. `docker stop`이 10초 걸리고 강제 종료되는 이유는?**
  - **A.** "PID 1이 셸이면 SIGTERM이 java로 전달되지 않습니다. `ENTRYPOINT`를 exec form으로 쓰고, Spring의 `server.shutdown=graceful`을 켜고, stop timeout을 셧다운 타임아웃보다 길게 잡아야 합니다. 쿠버네티스에서는 readiness 해제와 preStop 유예도 함께 봐야 롤링 배포 중 502가 사라집니다."

- **Q. 컨테이너 안에서 root로 실행하면 왜 위험한가요?**
  - **A.** "커널을 공유하므로 컨테이너 root는 호스트 root에 매우 가깝습니다. 커널 취약점이나 마운트 설정 실수로 탈출하면 호스트가 그대로 노출됩니다. 비루트 사용자, `--cap-drop ALL`, `no-new-privileges`, 가능하면 rootless Docker로 권한을 낮춥니다."

- **Q. `ARG`와 `ENV`에 비밀번호를 넣으면 왜 안 되나요?**
  - **A.** "`ARG`는 `docker history`에, `ENV`는 `docker inspect`에 그대로 남습니다. 레이어는 불변이므로 나중에 지워도 이미지에서 사라지지 않습니다. 빌드 시점 비밀은 BuildKit의 `--mount=type=secret`으로, 런타임 비밀은 파일 마운트나 시크릿 매니저로 넣습니다."

- **Q. 컨테이너 CPU를 1코어로 제한하면 애플리케이션은 어떻게 달라지나요?**
  - **A.** "JVM은 `cpu.max`를 읽어 ActiveProcessorCount를 정하고, 그 값으로 GC 스레드와 ForkJoinPool 병렬도가 정해집니다. 하지만 톰캣 `max-threads`는 설정값 그대로라 200이면 1코어에 스레드 200개가 경합합니다. 풀 크기를 CPU에 맞추고, 필요하면 `ActiveProcessorCount`로 고정합니다. 지연은 CPU 상한에서 계단식으로 튀는 특성이 있습니다."

- **Q. 쿠버네티스에서 이미지 HEALTHCHECK를 써도 되나요?**
  - **A.** "쿠버네티스는 이미지의 HEALTHCHECK를 무시합니다. liveness/readiness/startup probe를 따로 정의해야 합니다. HEALTHCHECK는 docker/compose에서만 유효하고, 두 환경을 다 지원하려면 이미지에는 두고 쿠버네티스에서는 probe로 대체합니다."

- **Q. `--read-only`로 컨테이너를 돌리려면 무엇을 준비해야 하나요?**
  - **A.** "애플리케이션이 쓰는 경로를 tmpfs나 볼륨으로 명시해야 합니다. Spring Boot는 `/tmp`를 쓰므로 `--tmpfs /tmp`가 없으면 시작에 실패합니다. 로그는 stdout으로 보내고, 업로드·캐시 디렉터리는 별도 볼륨으로 분리합니다."

- **Q. Dev/Prod 이미지를 같은 Dockerfile로 관리하는 방법은?**
  - **A.** "타깃 스테이지를 나눕니다. `AS development`(핫리로드 도구 포함)와 `AS runtime`(JRE만)을 두고, `docker build --target development` / `--target runtime`으로 선택합니다. 프로덕션 태그가 development 스테이지를 가리키지 않도록 CI에서 태그를 검증합니다."

---

## 참고

- Docker Docs — Storage drivers / overlay2: https://docs.docker.com/engine/storage/drivers/overlayfs-driver/
- Docker Docs — Multi-stage builds: https://docs.docker.com/build/building/multi-stage/
- Docker Docs — Build cache invalidation: https://docs.docker.com/build/cache/invalidation/
- Docker Docs — `run --mount` (cache, secret): https://docs.docker.com/reference/dockerfile/#run---mount
- Docker Docs — Resource constraints (`--memory`, `--cpus`): https://docs.docker.com/engine/containers/resource_constraints/
- Docker Docs — `HEALTHCHECK`: https://docs.docker.com/reference/dockerfile/#healthcheck
- Linux kernel — Control Group v2: https://docs.kernel.org/admin-guide/cgroup-v2.html
- Linux man-pages — namespaces(7): https://man7.org/linux/man-pages/man7/namespaces.7.html
- JDK-8146115 — Improve docker container detection: https://bugs.openjdk.org/browse/JDK-8146115
- Oracle — `-XX:MaxRAMPercentage` / container awareness: https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html
- Spring Boot — Layered JAR / container images: https://docs.spring.io/spring-boot/reference/packaging/container-images/efficient-images.html
- Spring Boot — Graceful shutdown: https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html
- BuildKit — cache backends: https://docs.docker.com/build/cache/backends/
