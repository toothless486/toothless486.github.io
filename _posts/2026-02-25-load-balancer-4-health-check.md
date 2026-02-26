---
layout: post
title: "[로드밸런서 삽질기 #4] Health Check: Docker HC vs Nginx Passive HC"
date: 2026-02-25 03:00:00 +0900
categories: [인프라]
tags: [Nginx, Docker, DockerCompose, HealthCheck, LoadBalancer, Failover]
---

> 웹 서버 1대가 죽으면 Nginx는 어떻게 반응할까? Docker HC와 Nginx Passive HC의 역할이 어떻게 다른지 정리했다.

> 결과물: https://github.com/toothless486/system-design-lab/pull/3

---

## 해결할 문제

[#3](/posts/load-balancer-3-round-robin/)에서 Round Robin 동작을 검증했다. 다음 단계는 **Health Check**다.

지금 구성에서 웹 서버 1대가 죽으면?

```
[Nginx]  →  web-server-1  (죽음)  ← 요청 → 502 Bad Gateway
         →  web-server-2  (살아있음)
         →  web-server-3  (살아있음)
```

Nginx가 죽은 서버를 감지하고 자동으로 제외해야 한다. 이것이 **Health Check**가 필요한 이유다.

---

## TL;DR

```
오해: "Nginx 오픈소스 버전엔 Health Check 기능이 없다"
사실: Active HC(주기적 능동 확인)가 없을 뿐,
      Passive HC(요청 실패 감지)는 이미 있다.
      Docker HC + Nginx Passive HC + restart 정책을 조합하면 완성된다.
```

---

## 1. Nginx 오픈소스 버전의 Health Check 한계

| 구분 | 설명 | Nginx 오픈소스 | Nginx Plus |
|---|---|---|---|
| **Passive HC** | 실제 요청이 실패할 때 감지 | ✅ 지원 | ✅ 지원 |
| **Active HC** | 주기적으로 능동적으로 체크 | ❌ 미지원 | ✅ 지원 |

기존 nginx.conf에 이미 Passive HC가 있었다:

```nginx
upstream backend {
  server web-server-1:3000 max_fails=3 fail_timeout=30s;
  server web-server-2:3000 max_fails=3 fail_timeout=30s;
  server web-server-3:3000 max_fails=3 fail_timeout=30s;
}
```

Health Check를 구현한다고 `max_fails`를 지워버리면 안 된다. **Docker HC와 Nginx Passive HC는 서로 다른 역할이다.**

---

## 2. Docker HC vs Nginx Passive HC — 역할이 완전히 다르다

```
[Docker HC]                    [Nginx Passive HC]
"컨테이너가 살아있나?"           "요청이 성공하나?"
    ↓                               ↓
Docker 데몬이 관리               Nginx가 관리
컨테이너 상태 레이블만 붙임        트래픽 라우팅 직접 제어
```

### Docker Healthcheck

Docker 데몬이 주기적으로 컨테이너 내부에서 명령을 실행한다.

```yaml
healthcheck:
  test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
  interval: 10s      # 10초마다 체크
  timeout: 10s       # 10초 안에 응답 없으면 실패
  retries: 3         # 3번 연속 실패 → unhealthy
  start_period: 40s  # 시작 후 40초는 실패해도 카운트 안 함
  start_interval: 5s # start_period 동안은 5초마다 체크
```

결과는 `docker ps`에서 확인 가능한 상태 레이블이다:

```
CONTAINER ID   STATUS
abc123         Up 2 minutes (healthy)
def456         Up 1 minute  (unhealthy)
```

**중요:** Docker HC가 `unhealthy`로 표시해도 컨테이너는 계속 실행 중이고, Nginx는 이 레이블을 읽지 않는다. **Nginx는 Docker를 모른다.**

### Nginx Passive Healthcheck

Nginx가 **실제 트래픽을 보내면서** 결과를 보고 판단한다.

```nginx
server web-server-1:3000 max_fails=3 fail_timeout=10s;
```

```
클라이언트 요청 → Nginx → web-server-1로 프록시
                              ↓
                         응답 실패? → 실패 카운트 +1
                         3번 실패?  → 10초 동안 제외
                         10초 후    → 자동으로 복귀 시도
```

요청이 실제로 실패해야만 감지한다는 게 핵심 한계다.

### 왜 둘 다 필요한가

```
상황: web-server-1이 시작 중 (아직 포트 안 열림)

Docker HC만 있을 때:
  Nginx → web-server-1 요청 → 연결 실패 → 에러 응답
  (Docker가 unhealthy 표시해도 Nginx는 계속 시도)

max_fails만 있을 때:
  실제 요청이 3번 실패해야 제외 → 그 3번은 클라이언트가 에러 봄

둘 다 있을 때:
  Docker HC → unhealthy 감지 (시작 순서 제어에 활용)
  Nginx max_fails → 실패 요청 감지 후 라우팅 제외
  → 빠르게 감지, 최소한의 에러
```

---

## 3. `condition: service_healthy` — 시작 순서 보장

### depends_on 기본 동작의 문제

```yaml
depends_on:
  - web-server-1
# 의미: "web-server-1 컨테이너가 시작되면 nginx도 시작"
# 문제: 컨테이너가 시작됐다 ≠ 서버가 요청 받을 준비됨
```

컨테이너가 시작된 직후에는 Node.js가 아직 초기화 중일 수 있다. 이 타이밍에 Nginx가 시작되면 첫 요청들이 실패한다.

### condition: service_healthy 적용

```yaml
depends_on:
  web-server-1:
    condition: service_healthy
  web-server-2:
    condition: service_healthy
  web-server-3:
    condition: service_healthy
```

```
[기본 depends_on]
t=0s  web-server-1 컨테이너 시작
t=0s  nginx 즉시 시작 → 아직 서버 안 뜬 web-server-1로 요청 → 에러

[condition: service_healthy]
t=0s   web-server-1 컨테이너 시작
t=5s   start_interval마다 /health 체크 시작
t=15s  /health 200 응답 → healthy
t=15s  nginx 시작 → 이미 준비된 서버로 요청 → 정상
```

---

## 4. `restart: unless-stopped` — 자동 복구

| 정책 | 설명 |
|---|---|
| `no` | 재시작 안 함 (기본값) |
| `always` | 항상 재시작 |
| `on-failure` | 비정상 종료(exit code ≠ 0)일 때만 재시작 |
| `unless-stopped` | 수동으로 멈춘 경우만 제외하고 항상 재시작 |

### `always`와의 핵심 차이

```
시나리오: docker stop web-server-1 후 PC 재부팅

always:
  PC 켜짐 → Docker 데몬 시작 → web-server-1 자동 시작 ← 내가 멈췄는데?

unless-stopped:
  PC 켜짐 → Docker 데몬 시작 → web-server-1 시작 안 함 ← 내 의도 존중
```

"내가 멈춘 상태"를 Docker가 기억한다.

크래시가 반복되면 Docker는 재시작 간격을 늘린다 (1s → 2s → 4s → ...).

---

## 5. 전체 Failover 흐름

```
[서버 장애 발생]
       ↓
Nginx passive HC 감지 (max_fails=3)
       ↓
해당 서버로 트래픽 차단 (fail_timeout 동안)
       ↓                        ↓
Docker HC → unhealthy      restart: unless-stopped
                                   ↓
                           컨테이너 자동 재시작
                                   ↓
                           /health 200 응답 → healthy
                                   ↓
                           Nginx fail_timeout 만료
                                   ↓
                           트래픽 자동 복귀
```

| 컴포넌트 | 역할 | 없으면? |
|---|---|---|
| Docker HC | 상태 감지 + 시작 순서 보장 | depends_on condition 못 씀 |
| Nginx max_fails | 트래픽 라우팅 제어 | 죽은 서버로 계속 요청 |
| `restart: unless-stopped` | 자동 복구 | 죽으면 영구 다운 |

---

## 6. 최종 구현

### server.js — `/health` 엔드포인트 추가

```javascript
app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
});
```

### nginx.conf

```nginx
upstream backend {
  server web-server-1:3000 max_fails=3 fail_timeout=10s;
  server web-server-2:3000 max_fails=3 fail_timeout=10s;
  server web-server-3:3000 max_fails=3 fail_timeout=10s;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

### compose.yaml

```yaml
services:
  load-balancer:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./load-balancer/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      web-server-1:
        condition: service_healthy
      web-server-2:
        condition: service_healthy
      web-server-3:
        condition: service_healthy

  web-server-1:
    build: ./web-server
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 10s
      retries: 3
      start_period: 40s
      start_interval: 5s

  web-server-2:
    build: ./web-server
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 10s
      retries: 3
      start_period: 40s
      start_interval: 5s

  web-server-3:
    build: ./web-server
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 10s
      retries: 3
      start_period: 40s
      start_interval: 5s
```

---

## 새로 알게 된 개념

### exit code — 프로세스가 어떻게 끝났는지를 나타내는 숫자

프로세스가 종료될 때 숫자 하나를 반환한다. 이걸 **exit code**라고 한다.

| exit code | 의미 |
|-----------|------|
| `0` | 정상 종료 |
| `1` | 일반적인 에러 |
| `127` | 명령어를 찾을 수 없음 (command not found) |
| `137` | SIGKILL로 강제 종료됨 (128 + 9) |

`on-failure` restart 정책은 exit code가 `0`이 아닐 때만 재시작한다. 그래서 정상 종료(`exit 0`)로 끝나는 서버에는 적합하지 않다.

```yaml
restart: on-failure     # exit code != 0 일 때만 재시작
restart: unless-stopped # exit code 관계없이 재시작 (수동 정지 제외)
```

프로세스가 어떻게 종료됐는지 확인하려면:

```bash
docker inspect <container> --format '{{.State.ExitCode}}'
```

---

## 3줄 요약

1. **Docker HC와 Nginx Passive HC는 역할이 다르다 — Docker는 상태 레이블, Nginx는 트래픽 제어. 둘 다 있어야 한다.**
2. **`condition: service_healthy`는 시작 순서를 보장한다 — 컨테이너가 시작됐다 ≠ 서버가 준비됐다.**
3. **`restart: unless-stopped`가 없으면 죽은 서버는 영구 다운 — 세 컴포넌트의 역할 분담이 완전한 HC를 만든다.**

---

> *"Nginx Plus 없어도 됩니다라고 생각했는데, 알고 보니 Docker HC, Nginx Passive HC, restart 정책이 각자 역할을 나눠서 Nginx Plus의 Active HC를 대체하고 있었다. 도구 하나로 다 해결하려는 게 오히려 오해였다."*
