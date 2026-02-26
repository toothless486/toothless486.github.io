---
layout: post
title: "[로드밸런서 삽질기 #5] Failover 테스트: docker kill이 Windows에서 안 되는 이유"
date: 2026-02-25 04:00:00 +0900
categories: [인프라]
tags: [Docker, Nginx, LoadBalancer, Failover, HealthCheck, Windows, 삽질기록]
---

> Health Check를 구현했다. 이제 실제로 서버를 죽여서 제대로 동작하는지 확인할 차례다. 예상대로 되지 않았다.

> 결과물: https://github.com/toothless486/system-design-lab/pull/3

---

## 검증 시나리오

[#4](/posts/load-balancer-4-health-check/)에서 Health Check를 구현했다:
- Docker Healthcheck (`wget /health`)
- Nginx Passive HC (`max_fails=3`)
- `restart: unless-stopped`

검증 계획:

```
1. 터미널 A: 연속 요청 루프 실행
2. 터미널 B: 서버 1대 강제 종료
3. 확인: 에러 없이 나머지 2대가 응답하는가?
4. 확인: 종료된 서버가 자동으로 재시작되는가?
```

두 가지 삽질이 있었다.

---

## TL;DR

```
삽질 1: node:18-alpine에 curl이 없어서 healthcheck 자체가 실패
삽질 2: docker kill로 재시작 테스트했는데 unless-stopped가 동작 안 함
해결  : curl → wget, docker kill → docker exec kill 1
```

---

## 실패 1: `curl` 없음 — Nginx가 아예 시작 안 됨

### 증상

```bash
docker compose up -d
# dependency failed to start: container web-server-2 is unhealthy
```

3개 web-server가 모두 `unhealthy` → `condition: service_healthy` 조건 실패 → Nginx 시작 거부.

### 원인 추적

```bash
docker exec web-server-1 curl --version
# Exit code 127
# executable file not found in $PATH
```

**exit code 127 = 명령어를 찾을 수 없음.**

healthcheck 명령어는 컨테이너 안에서 실행되는데, `node:18-alpine`은 Alpine Linux 기반 경량 이미지라 `curl`을 기본 포함하지 않는다.

```
healthcheck 실행 흐름:
  Docker 데몬 → 컨테이너 안에서 curl 실행
                     ↓
               curl 없음 → exit 127 → 실패
                     ↓
               3번 반복 → unhealthy
                     ↓
  condition: service_healthy → Nginx 시작 거부
```

### `node:18` vs `node:18-alpine` 차이

| 이미지 | 기반 OS | 용량 | curl |
|---|---|---|---|
| `node:18` | Debian | ~950MB | ✅ 포함 |
| `node:18-alpine` | Alpine Linux | ~170MB | ❌ 미포함 |

Alpine은 크기를 최소화한 대신 기본 도구가 없다.

### 해결: `curl` → `wget` (Alpine 기본 포함)

```yaml
# 수정 전
test: ["CMD", "curl", "-f", "http://localhost:3000/health"]

# 수정 후
test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
```

> **교훈:** 이미지 이름 뒤에 `-alpine`이 붙으면 curl, bash 등 기본 도구가 없다고 가정하라. healthcheck 명령어는 컨테이너 내부에서 실행된다 — 호스트 환경과 무관하다.

---

## Failover 테스트 (성공)

수정 후 정상 기동 확인:

```bash
docker compose up -d
docker ps
# web-server-1   Up (healthy)
# web-server-2   Up (healthy)
# web-server-3   Up (healthy)
# load-balancer  Up
```

**터미널 A — 연속 요청 루프:**

```bash
for i in $(seq 1 30); do curl -s http://localhost:8080; echo ""; sleep 0.5; done
```

**터미널 B — 서버 강제 종료:**

```bash
docker kill simple-round-robin-web-server-1-1
```

**kill 직후 터미널 A 결과:**

```
{"server":"d44828d4f8cf"}   ← web-server-1
{"server":"d1fac7244b59"}   ← web-server-2
{"server":"564c4a9fb8dd"}   ← web-server-3
{"server":"d44828d4f8cf"}   ← web-server-1  ← 여기서 kill
{"server":"d1fac7244b59"}   ← 1번 사라짐, 에러 없음
{"server":"564c4a9fb8dd"}
{"server":"d1fac7244b59"}   ← 2, 3만 응답
{"server":"564c4a9fb8dd"}
...
```

**에러 없이 2개 서버가 계속 응답 → Nginx Passive HC(max_fails) 정상 작동 확인.**

---

## 실패 2: `docker kill` 후 `unless-stopped` 재시작 안 됨

### 증상

```bash
docker ps -a
# web-server-1   Exited (137) 5 minutes ago
```

5분이 지나도 재시작 없음.

```bash
docker inspect web-server-1 --format '{{json .HostConfig.RestartPolicy}}'
# {"Name":"unless-stopped","MaximumRetryCount":0}  ← 정책은 설정됨

docker inspect web-server-1 --format '{{json .RestartCount}}'
# 0  ← 한 번도 재시작 시도 안 함
```

`RestartCount = 0` — Docker 엔진이 재시작을 **시도조차 하지 않았다.**

### 원인: Docker Desktop/Windows에서 `docker kill`은 "의도적 종료"로 인식

Docker는 내부적으로 `HasBeenManuallyStopped` 플래그로 재시작 여부를 결정한다:

```
docker stop → SIGTERM → SIGKILL → HasBeenManuallyStopped = true  → 재시작 X
docker kill → SIGKILL             → HasBeenManuallyStopped = ???
```

Linux Docker 엔진에서는 `docker kill`이 플래그를 세우지 않아 재시작이 **되어야** 한다. 그러나 **Docker Desktop for Windows (WSL2 기반)** 에서는 `docker kill`도 의도적 종료로 처리한다. `RestartCount = 0`이 이를 증명한다.

### 해결: 컨테이너 내부에서 프로세스 종료

```bash
# 컨테이너 안의 PID 1(npm start)에 SIGTERM 전송
docker exec simple-round-robin-web-server-1-1 kill 1
```

Docker 입장에서:

```
컨테이너 외부에서 kill/stop → HasBeenManuallyStopped = true  → 재시작 X
컨테이너 내부 프로세스 종료  → HasBeenManuallyStopped = false → 재시작 O ✅
```

### `kill 1` 실행 후 로그

```bash
docker logs simple-round-robin-web-server-1-1
```

```
npm error signal SIGTERM        ← 1번째 종료 (kill 1)
> node server.js                ← restart: unless-stopped → 재시작
npm error signal SIGTERM        ← 2번째 종료
> node server.js                ← 재시작
npm error signal SIGTERM        ← 3번째 종료
> node server.js                ← 재시작 → 현재 running (healthy)
```

`restart: unless-stopped` 정상 작동 확인.

---

## 세 가지 종료 방식 최종 비교

| 방법 | 신호 | Docker 인식 | unless-stopped | 실제 운영 유사도 |
|---|---|---|---|---|
| `docker stop` | SIGTERM → SIGKILL | 의도적 종료 | 재시작 X | 배포/점검 시 |
| `docker kill` | SIGKILL | 의도적 종료 (Desktop/Win) | 재시작 X | - |
| `docker exec kill 1` | SIGTERM (내부) | 프로세스 크래시 | 재시작 O ✅ | OOM, panic |
| OOM, 앱 crash | - | 프로세스 크래시 | 재시작 O ✅ | 실제 장애 |

---

## 새로 알게 된 개념

### PID 1 — 컨테이너의 첫 번째 프로세스

컨테이너가 시작될 때 Dockerfile의 `CMD`로 실행된 프로세스가 **PID 1**이 된다.

```dockerfile
CMD ["npm", "start"]   # 이 프로세스가 PID 1
```

PID 1은 컨테이너에서 특별한 위치를 가진다:

- **PID 1이 종료되면 컨테이너 전체가 종료된다.** 다른 프로세스는 어떻게 됐든 상관없이.
- Docker의 restart 정책은 PID 1의 종료를 감지해서 재시작을 결정한다.

`docker exec kill 1`은 컨테이너 안에서 PID 1에 신호를 보내는 명령이다. Docker 외부에서 `docker kill`을 쓰는 것과는 Docker 엔진이 인식하는 방식이 다르다.

### SIGTERM vs SIGKILL

Unix/Linux에서 프로세스를 종료할 때 "신호(signal)"를 보낸다.

| 신호 | 번호 | 의미 | 프로세스 무시 가능? |
|------|------|------|---------------------|
| `SIGTERM` | 15 | "종료 요청" — 정상적으로 마무리하고 끝내라 | ✅ 무시 가능 |
| `SIGKILL` | 9 | "즉시 강제 종료" — 커널이 직접 죽임 | ❌ 무시 불가 |

```
docker stop → SIGTERM 보내고 10초 대기 → 안 죽으면 SIGKILL
docker kill → 즉시 SIGKILL (기본값)
```

`SIGTERM`을 받으면 프로세스는 현재 처리 중인 요청을 마무리하고, 파일을 닫고, 정상적으로 종료할 수 있다. `SIGKILL`은 그런 기회 없이 바로 강제 종료한다.

`docker exec kill 1`은 기본적으로 `SIGTERM`을 보낸다. exit code 관점에서 "예상치 못한 종료"이므로, Docker 엔진이 크래시로 인식해 restart 정책이 발동한다.

---

## 3줄 요약

1. **`-alpine` 이미지에는 `curl`이 없다 — healthcheck 명령어는 컨테이너 안에서 실행되므로 `wget`으로 대체**
2. **`docker kill`은 Docker Desktop/Windows에서 의도적 종료로 인식 — `unless-stopped` 재시작 안 됨**
3. **`docker exec kill 1`이 실제 장애를 시뮬레이션하는 올바른 방법 — 내부 프로세스 종료는 크래시로 인식**
