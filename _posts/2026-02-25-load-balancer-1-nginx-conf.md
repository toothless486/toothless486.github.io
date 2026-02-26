---
layout: post
title: "[로드밸런서 삽질기 #1] Nginx 설정 3가지 실수"
date: 2026-02-25 00:00:00 +0900
categories: [인프라]
tags: [Nginx, Docker, LoadBalancer, RoundRobin, 삽질기록]
---

> Nginx 1대 + Node.js 3대 Round Robin 실습에서 nginx.conf를 처음 작성할 때 빠지기 쉬운 함정들을 기록합니다.

> 결과물: https://github.com/toothless486/system-design-lab/pull/2

---

## 실습 배경

시스템디자인 90일 학습 프로젝트 **Week 1: 로드밸런서** 실습이다.

목표: **Nginx 1대 + Node.js 웹 서버 3대**를 Docker로 띄워서 Round Robin 방식으로 트래픽을 분산한다.

```
[사용자]
    ↓
[Nginx :8080]
    ↓  ↓  ↓
[web-server-1:3000]
[web-server-2:3000]
[web-server-3:3000]
```

파일 구성:

```
simple-round-robin/
├── load-balancer/
│   └── nginx.conf
├── web-server/
│   ├── server.js
│   ├── package.json
│   └── Dockerfile
└── compose.yaml
```

---

## TL;DR

nginx.conf에 `upstream` 블록만 덜렁 작성하고 "이러면 되겠지?" 했다가 **3가지 치명적 실수**를 발견했다.

```nginx
# 내가 처음 작성한 설정 (동작 안 함)
upstream backend {
   server web-server-1:3000;
   server web-server-2:3001;
   server web-server-3:3002;
}
```

결론부터 말하면 — **이 설정은 실행은 되지만 아무 일도 하지 않는다.**

---

## 실수 1: `server` 블록이 없다

### 문제

```nginx
upstream backend {
   server web-server-1:3000;
   server web-server-2:3001;
   server web-server-3:3002;
}
# 끝. 이게 전부였다.
```

Nginx를 띄우면 에러 없이 시작된다. 하지만 `curl http://localhost:8080`을 치면 **Nginx 기본 Welcome 페이지**만 나온다.

### 원인

`upstream`은 **백엔드 서버 목록을 정의**하는 블록일 뿐이다. 실제로 사용하는 `server` 블록 + `proxy_pass`가 없으면 Nginx는 그냥 무시한다.

- `upstream` = 전화번호부에 번호를 적어놓은 것
- `server` + `proxy_pass` = 실제로 전화를 거는 행위

### 해결

```nginx
upstream backend {
    server web-server-1:3000;
    server web-server-2:3000;
    server web-server-3:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

> **교훈:** Nginx 설정은 "정의"와 "사용"이 분리되어 있다. `upstream`으로 정의만 하면 안 되고, `server > location > proxy_pass`로 **사용**해야 한다.

---

## 실수 2: 포트를 3000, 3001, 3002로 나눴다

### 문제

```nginx
upstream backend {
   server web-server-1:3000;
   server web-server-2:3001;   # 왜 3001?
   server web-server-3:3002;   # 왜 3002?
}
```

로컬에서 Node.js 서버를 3개 띄울 때의 습관이 나왔다. 로컬에서는 같은 포트를 쓸 수 없으니 나누는 게 맞다. 하지만 **Docker에서는 다르다.**

### 원인

Docker 컨테이너는 각각 **독립된 네트워크 네임스페이스**를 가진다.

```
[로컬 환경]                    [Docker 환경]
:3000 서버1                   web-server-1:3000  OK
:3001 서버2                   web-server-2:3000  OK (독립!)
:3002 서버3                   web-server-3:3000  OK (독립!)
(포트 겹치면 충돌)              (컨테이너마다 분리됨)
```

### 해결

```nginx
upstream backend {
    server web-server-1:3000;   # 모두 3000
    server web-server-2:3000;
    server web-server-3:3000;
}
```

> **교훈:** Docker 컨테이너 = 각각 독립된 미니 서버. 로컬 개발 습관을 Docker에 그대로 적용하면 안 된다.

---

## 실수 3: `proxy_pass http://backend`를 URL 경로로 착각했다

nginx.conf를 수정하다가 이 한 줄에서 순간 혼란이 왔다:

```nginx
proxy_pass http://backend;
```

"`http://backend`가 실제 URL인가? `localhost:8080/backend`로 요청해야 하는 건가?"

### 원인

`proxy_pass`의 `http://backend`에서 `backend`는 두 가지로 해석될 수 있다:

| 해석 | 예시 |
|------|------|
| 실제 호스트 주소 | `proxy_pass http://google.com;` |
| **upstream 블록 이름** | `proxy_pass http://backend;` |

Nginx는 `backend`라는 이름이 `upstream` 블록에 정의되어 있으면, 그 그룹의 서버 목록을 사용한다. `backend`는 예약어가 아니라 내가 붙인 이름이다.

### 요청 흐름

```
curl localhost:8080/
        ↓
   Nginx (listen 80)
        ↓  location / 에 매칭
   proxy_pass http://backend
        ↓  "backend" upstream 찾기
   web-server-1:3000  ← 1번째 요청
   web-server-2:3000  ← 2번째 요청
   web-server-3:3000  ← 3번째 요청 (Round Robin)
```

사용자는 `localhost:8080/`로만 접근하면 된다. `localhost:8080/backend`가 아니다.

> **교훈:** `proxy_pass http://backend`의 `backend`는 URL 경로가 아니라 `upstream` 블록의 이름이다.

---

## 보너스: load-balancer/Dockerfile도 불완전했다

### before

```dockerfile
# load-balancer/Dockerfile — FROM이 없음
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

`FROM` 지시어가 없어서 Docker 빌드 자체가 불가능하다. 그런데 이 Dockerfile 자체가 **불필요하다.** compose.yaml에서 볼륨 마운트로 대체할 수 있다:

### after

```yaml
load-balancer:
  image: nginx:latest
  volumes:
    - ./load-balancer/nginx.conf:/etc/nginx/conf.d/default.conf
```

설정 파일 하나 바꾸려고 Dockerfile을 만들 필요 없다. **volumes 마운트면 충분하다.**

---

## 새로 알게 된 개념

### volumes (bind mount)

compose.yaml에서 `volumes`는 **호스트의 파일을 컨테이너 안에 연결**하는 방법이다.

```yaml
volumes:
  - ./load-balancer/nginx.conf:/etc/nginx/conf.d/default.conf
  # 호스트 경로            :  컨테이너 경로
```

"복사"가 아니라 "연결"이다. 호스트에서 파일을 수정하면 컨테이너 안에서도 즉시 반영된다. 이 덕분에 Nginx 설정을 바꿀 때 이미지를 다시 빌드할 필요가 없다.

```
[호스트]                         [컨테이너]
./load-balancer/nginx.conf  ←→  /etc/nginx/conf.d/default.conf
(수정하면 즉시 반영)
```

volumes 없이 설정 파일을 쓰려면 Dockerfile을 따로 만들어야 하는데, 설정 파일 하나를 위해 빌드 과정 전체를 추가하는 건 과한 일이다.

### Nginx 설정 블록 구조

Nginx 설정은 **정의 → 사용** 두 단계로 이루어진다:

```
upstream backend { ... }    ← 1단계: 백엔드 서버 그룹 정의
        ↓
server {                    ← 2단계: 이 서버로 들어오는 요청 처리
    listen 80;              ← 어떤 포트에서 들을지
    location / {            ← 어떤 경로의 요청에 대해
        proxy_pass http://backend;  ← 1단계에서 정의한 그룹으로 전달
    }
}
```

- `upstream`: 서버 그룹 이름과 멤버 목록 등록
- `server`: 들어오는 요청을 처리하는 가상 호스트 단위
- `location`: URL 경로 패턴별로 다른 처리를 지정
- `proxy_pass`: 요청을 다른 서버(또는 upstream 그룹)로 전달

`upstream`만 있고 `server`가 없으면 — 전화번호부에 번호는 적혀있는데 전화를 안 건 상태다.

---

## 3줄 요약

1. **Nginx `upstream`은 정의일 뿐 — `server` + `proxy_pass`로 사용해야 동작한다**
2. **Docker 컨테이너는 독립 네트워크 — 포트를 나눌 필요 없이 모두 같은 포트 사용 가능**
3. **`proxy_pass http://backend`의 `backend`는 URL 경로가 아니라 upstream 블록 이름이다**
