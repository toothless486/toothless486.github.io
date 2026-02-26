---
layout: post
title: "[로드밸런서 삽질기 #2] docker compose up 하면 터지는 이유"
date: 2026-02-25 01:00:00 +0900
categories: [인프라]
tags: [Docker, DockerCompose, LoadBalancer, 삽질기록]
---

> nginx.conf, server.js, package.json을 다 작성하고 "이제 되겠지?" 했는데 `docker compose up`이 3중으로 터졌다.

> 결과물: https://github.com/toothless486/system-design-lab/pull/2

---

## 여기까지 온 과정

[#1 — Nginx 설정 3가지 실수](/posts/load-balancer-1-nginx-conf/)에서 nginx.conf를 고쳤고, server.js, package.json도 작성을 완료했다.

파일 구성:

```
simple-round-robin/
├── load-balancer/
│   └── nginx.conf     ✅ 완성
├── web-server/
│   ├── server.js      ✅ 완성
│   ├── package.json   ✅ 완성
│   └── Dockerfile     ❓ 빈 파일
└── compose.yaml       ❓ 문제 있음
```

이제 `docker compose up` 한 번이면 끝이겠지 했는데.

---

## TL;DR

```
실패 1: Dockerfile이 비어있어서 빌드 자체가 안 됨
실패 2: 호스트 포트 3000이 3개 서비스에서 중복 → 충돌
실패 3: volumes 마운트가 불필요한 복잡성을 추가
```

---

## 실패 1: Dockerfile이 비어있다 (FATAL)

### 문제

```
$ docker compose up
=> ERROR [web-server-1] failed to solve: failed to read dockerfile
```

compose.yaml에서 `build: ./web-server`라고 썼는데, `web-server/Dockerfile`이 빈 파일이었다.

```yaml
# compose.yaml
web-server-1:
  build: ./web-server    # 이 경로의 Dockerfile을 찾아서 빌드하겠다는 뜻
```

```dockerfile
# web-server/Dockerfile
(텅 빈 파일)
```

### 필요한 Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
CMD ["npm", "start"]
```

각 줄이 하는 일:

```
FROM node:18-alpine  → Node.js가 설치된 경량 리눅스를 기반으로 시작
WORKDIR /app         → 컨테이너 안에서 /app 폴더를 작업 디렉토리로 설정
COPY package.json .  → package.json만 먼저 복사 (Docker 레이어 캐시 최적화)
RUN npm install      → dependencies(express) 설치
COPY . .             → 나머지 파일(server.js) 복사
CMD ["npm", "start"] → 컨테이너 시작 시 npm start 실행
```

**왜 `package.json`을 먼저 복사할까?**

```
[전부 한번에 복사할 경우]
server.js 한 줄 수정 → COPY . . 캐시 무효화 → npm install 다시 실행 (느림!)

[package.json 먼저 복사할 경우]
server.js 한 줄 수정 → COPY package.json . 캐시 유효 (안 바뀜)
                     → npm install 캐시 유효 (스킵!)
                     → COPY . . 만 다시 실행 (빠름!)
```

`package.json`이 안 바뀌면 `npm install`을 건너뛴다. 빌드 시간이 크게 줄어든다.

> **교훈:** compose.yaml에서 `build: ./path`를 쓸 때 해당 경로에 유효한 Dockerfile이 반드시 있어야 한다. 빈 파일은 "없는 것"과 같다.

---

## 실패 2: 호스트 포트 3000이 3개 서비스에서 중복 (FATAL)

### 문제

```yaml
web-server-1:
  ports:
    - "3000:3000"    # 호스트 3000

web-server-2:
  ports:
    - "3000:3000"    # 호스트 3000 → 충돌!

web-server-3:
  ports:
    - "3000:3000"    # 호스트 3000 → 충돌!
```

```
$ docker compose up
Error: Bind for 0.0.0.0:3000 failed: port is already allocated
```

### #1에서 배운 것과의 혼동

"Docker 컨테이너는 독립 네트워크이므로 **내부 포트는 모두 3000으로 같아도 된다**"

이건 맞다. **하지만 `ports`는 호스트 포트 매핑이다.**

```
ports: "호스트포트:컨테이너포트"

컨테이너 내부 포트 :3000  ← 컨테이너끼리 독립, 같아도 OK
호스트 포트 매핑 "3000:3000" ← 하나의 OS에서 공유, 중복 불가
```

### 진짜 해결: ports 자체를 제거

포트를 3000, 3001, 3002로 나누면 충돌은 해결되지만, **애초에 웹 서버를 외부에 노출할 필요가 없다.**

```
사용자 → localhost:8080 → Nginx → Docker 내부 네트워크 → web-server-1:3000
                                                      → web-server-2:3000
                                                      → web-server-3:3000
```

사용자는 Nginx(8080)만 접근하고, 웹 서버에 직접 접근할 일이 없다. **`ports` 자체가 불필요하다.**

```yaml
# 수정 전
web-server-1:
  build: ./web-server
  ports:
    - "3000:3000"        # 불필요. 제거.

# 수정 후
web-server-1:
  build: ./web-server
  # ports 없음
```

> **교훈:** "컨테이너 내부 포트가 같아도 된다"와 "호스트 포트 매핑이 같아도 된다"는 전혀 다른 이야기다. 로드밸런서 뒤의 백엔드 서버는 외부에 노출할 이유가 없다.

---

## 실패 3: volumes 마운트가 만드는 함정 (MINOR)

### 문제

```yaml
web-server-1:
  build: ./web-server
  volumes:
    - ./web-server/server.js:/app/server.js
    - ./web-server/package.json:/app/package.json
```

이렇게 하면 Dockerfile에서 `COPY`한 파일을 **컨테이너 실행 시점에 덮어씌운다.**

```
[빌드 시점] COPY package.json .  → /app/package.json
            RUN npm install      → /app/node_modules/ (설치됨)
            COPY . .             → /app/server.js

[실행 시점] volumes 마운트       → /app/server.js 덮어씌움
                                 → /app/package.json 덮어씌움
                                 → node_modules는? 호스트에 없으면 문제!
```

개발 중 핫리로드가 목적이면 유용하지만, 학습 프로젝트에서는 **Dockerfile의 COPY만으로 충분하다.**

---

## 최종 compose.yaml

```yaml
services:
  load-balancer:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./load-balancer/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - web-server-1
      - web-server-2
      - web-server-3

  web-server-1:
    build: ./web-server

  web-server-2:
    build: ./web-server

  web-server-3:
    build: ./web-server
```

### 수정 전후 비교

| 파일 | 수정 전 | 수정 후 |
|------|--------|--------|
| Dockerfile | 빈 파일 (FATAL) | 6줄 작성 |
| compose.yaml ports | 3개 서비스 모두 `3000:3000` (충돌) | 제거 |
| compose.yaml volumes | server.js, package.json 마운트 (불필요) | 제거 |

---

## 새로 알게 된 개념

### `image` vs `build` — compose.yaml에서 컨테이너를 만드는 두 가지 방법

```yaml
# image: 이미 만들어진 이미지를 그대로 사용
load-balancer:
  image: nginx:latest        # Docker Hub에서 nginx 공식 이미지 가져오기

# build: 소스 코드로부터 직접 이미지를 빌드
web-server-1:
  build: ./web-server        # 이 경로의 Dockerfile을 실행해서 이미지 생성
```

- `image`: 이미 완성된 이미지를 가져다 쓴다. 빠르고 간단.
- `build`: Dockerfile을 실행해서 이미지를 직접 만든다. 소스 코드가 포함된 커스텀 서버에 사용.

Nginx는 공식 이미지를 그대로 써도 되지만, Node.js 서버는 내 코드를 넣어야 하므로 `build`를 사용한다.

### `depends_on` — 서비스 시작 순서 제어

```yaml
load-balancer:
  depends_on:
    - web-server-1
    - web-server-2
    - web-server-3
```

`depends_on`은 **"이 서비스들이 시작된 다음에 나를 시작해라"** 라는 의미다.

Nginx는 백엔드 서버들이 없으면 프록시할 대상이 없으므로, 웹 서버 3대가 먼저 시작되어야 한다. `depends_on` 없이 동시에 시작하면 Nginx가 아직 없는 서버에 연결을 시도해 초기 요청이 실패할 수 있다.

단, 기본 `depends_on`은 "컨테이너가 시작됐다"는 것만 보장한다. "서버가 요청을 받을 준비됐다"까지 보장하려면 `condition: service_healthy`가 필요하다 — 이건 [#4](/posts/load-balancer-4-health-check/)에서 다룬다.

---

## 3줄 요약

1. **Dockerfile이 비어있으면 `build`가 실패한다 — compose.yaml에서 `build: ./path`를 쓸 때 Dockerfile은 필수**
2. **"컨테이너 내부 포트 같아도 됨" ≠ "호스트 포트 매핑 같아도 됨" — 로드밸런서 뒤의 서버는 ports 자체가 불필요**
3. **volumes와 Dockerfile COPY가 겹치면 혼란만 늘어난다 — 학습 단계에서는 COPY만으로 충분**
