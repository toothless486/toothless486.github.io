---
layout: post
title: "[로드밸런서 삽질기 #3] Round Robin 동작 확인 + 성능 측정"
date: 2026-02-25 02:00:00 +0900
categories: [인프라]
tags: [Nginx, Docker, LoadBalancer, RoundRobin, 성능측정, ApacheBench]
---

> nginx.conf와 compose.yaml을 모두 고쳤다. 이제 실제로 Round Robin이 동작하는지 확인하고, 서버가 몇 대냐에 따라 성능이 어떻게 달라지는지 측정해봤다.

> 결과물: https://github.com/toothless486/system-design-lab/pull/2

---

## 드디어 실행되는 구성

[#1](/posts/load-balancer-1-nginx-conf/)과 [#2](/posts/load-balancer-2-docker-compose/)의 삽질을 모두 해결한 최종 구성:

```
simple-round-robin/
├── load-balancer/
│   └── nginx.conf       # upstream + server + proxy_pass
├── web-server/
│   ├── server.js        # Express + os.hostname() 반환
│   ├── package.json     # express 의존성
│   └── Dockerfile       # node:18-alpine 기반
└── compose.yaml         # Nginx 1대 + web-server 3대
```

---

## 1. Round Robin 동작 확인

### 실행

```bash
docker compose up -d
```

```
Container simple-round-robin-web-server-1-1    Started
Container simple-round-robin-web-server-2-1    Started
Container simple-round-robin-web-server-3-1    Started
Container simple-round-robin-load-balancer-1   Started
```

### curl로 확인

```bash
curl http://localhost:8080
# {"server":"5705054b2563"}

curl http://localhost:8080
# {"server":"aba04c2cb1ac"}

curl http://localhost:8080
# {"server":"aa527b3649ab"}

curl http://localhost:8080
# {"server":"5705054b2563"}  ← 다시 1번 서버
```

3개의 서버가 순서대로 돌아가며 응답한다. Round Robin 동작 확인 완료.

각 hostname은 Docker가 컨테이너에 부여한 ID이고, `os.hostname()`으로 반환하는 값이다.

### 주의: `&&`로 연속 호출하면 같은 서버만 응답한다

```bash
# 이렇게 하면 같은 서버만 응답함
curl -s http://localhost:8080 && curl -s http://localhost:8080 && curl -s http://localhost:8080
# {"server":"5705054b2563"}
# {"server":"5705054b2563"}
# {"server":"5705054b2563"}
```

`&&`로 연결하면 연결이 재사용될 수 있어서 Nginx가 같은 upstream으로 보낸다. **개별로 호출해야 Round Robin을 확인할 수 있다.**

---

## 2. 성능 측정

### 환경

- OS: Windows 11 Pro / Docker Desktop
- 측정 도구: Apache Bench (`jordi/ab` Docker 이미지)
- 측정 방법: Docker 내부 네트워크에서 직접 호출

```bash
# Before: 서버 1대 직접 호출
docker run --rm --network simple-round-robin_default \
  jordi/ab -n 1000 -c 100 http://web-server-1:3000/

# After: Nginx 로드밸런서 경유 (3대 분산)
docker run --rm --network simple-round-robin_default \
  jordi/ab -n 1000 -c 100 http://load-balancer:80/
```

### 결과 1: 1000 req / 동시 100

| | 서버 1대 (Before) | 로드밸런서 3대 (After) | 변화 |
|---|---|---|---|
| **처리량** | 3,025 req/s | 4,111 req/s | **+36%** |
| **응답시간 (mean)** | 33ms | 24ms | **-27%** |
| **응답시간 (p50)** | 29ms | 8ms | **-72%** |
| **응답시간 (max)** | 68ms | 93ms | +37% |
| **에러율** | 0% | 0% | - |

### 결과 2: 5000 req / 동시 500 (고부하)

| | 서버 1대 (Before) | 로드밸런서 3대 (After) | 변화 |
|---|---|---|---|
| **처리량** | 4,650 req/s | 5,574 req/s | **+20%** |
| **응답시간 (mean)** | 107ms | 89ms | **-17%** |
| **응답시간 (p50)** | 96ms | 96ms | 동일 |
| **응답시간 (max)** | 229ms | 205ms | **-10%** |
| **에러율** | 0% | 0% | - |

---

## 3. 왜 3배가 아닌가

현재 server.js는 `os.hostname()`만 반환하는 초경량 작업이다:

```js
app.get('/', (req, res) => {
    res.json({ server: os.hostname() });
});
```

- CPU를 거의 사용하지 않음
- 단일 서버만으로도 3,000+ req/s 처리 가능
- **서버가 병목이 아니라 네트워크가 병목**인 상태

### CPU 부하를 추가하면 달라진다

fibonacci(30)을 계산하는 CPU 부하를 추가해서 측정:

| | 서버 1대 | 로드밸런서 3대 | 변화 |
|---|---|---|---|
| **처리량** | 147 req/s | 332 req/s | **+126% (2.3배)** |
| **응답시간 (mean)** | 677ms | 300ms | **-56%** |

**CPU 부하가 걸리면 로드밸런서의 효과가 극적으로 나타난다.**

```
서버 부하가 낮을 때:  로드밸런서 효과 미미 (+20~36%)
서버 부하가 높을 때:  로드밸런서 효과 극적 (+126%, 2.3배)
```

**로드밸런서는 "서버가 병목일 때" 빛나는 기술이다.** 서버가 충분히 빠르면 Nginx 프록시 오버헤드만 늘어날 수 있다.

---

## 새로 알게 된 개념

### Round Robin 알고리즘

Round Robin은 **요청을 서버 목록 순서대로 하나씩 돌아가며 분배**하는 방식이다.

```
요청 1 → web-server-1
요청 2 → web-server-2
요청 3 → web-server-3
요청 4 → web-server-1  (다시 처음으로)
요청 5 → web-server-2
...
```

가장 단순한 로드밸런싱 알고리즘이다. 모든 서버의 처리 능력이 같다고 가정하고, 단순히 순서대로 나눈다. Nginx의 기본 알고리즘이기도 하다.

`upstream` 블록에 알고리즘을 따로 지정하지 않으면 Round Robin이 적용된다.

### Apache Bench 플래그 읽기

```bash
jordi/ab -n 1000 -c 100 http://load-balancer:80/
#            ↑       ↑
#            │       └─ concurrency: 동시에 보내는 요청 수
#            └─ requests: 총 요청 수
```

- `-n 1000`: 총 1000번 요청을 보낸다
- `-c 100`: 그 중 100개를 동시에 보낸다 (동시 접속자 수 시뮬레이션)

결과 지표:

| 지표 | 의미 |
|------|------|
| **Requests/sec (처리량)** | 초당 처리한 요청 수. 클수록 좋음. |
| **Time per request - mean (평균 응답시간)** | 요청 1개당 평균 걸린 시간 |
| **Time per request - p50** | 전체 요청 중 50%가 이 시간 이내에 완료됨 (중앙값) |
| **Time per request - max** | 가장 오래 걸린 요청의 시간 |

처리량이 높아도 max 응답시간이 크면, 일부 사용자는 오래 기다린다. 두 지표를 함께 봐야 한다.

---

## 3줄 요약

1. **Round Robin은 curl을 개별 호출해야 확인 가능 — `&&` 연속 호출은 연결 재사용으로 같은 서버만 응답**
2. **초경량 서버에서는 로드밸런서 효과가 +20~36% 수준 — 서버 부하가 병목이 아니면 극적 차이 없음**
3. **CPU 부하가 걸리면 2.3배 처리량 향상 — 로드밸런서는 "서버가 병목일 때" 진가를 발휘한다**
