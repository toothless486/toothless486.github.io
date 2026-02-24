---
layout: post
title: "MQ 연결이 자꾸 끊긴다 - Lock과 Blocking I/O의 함정"
date: 2026-02-23 00:00:00 +0900
categories: [동시성]
tags: [C, STOMP, MQ, Mutex, BlockingIO, 멀티스레드, 디버깅]
---

> MessageQueue 서버와의 연결이 간헐적으로 끊어지는 문제를 추적하고 해결하기까지의 과정을 기록합니다.

---

## 증상

테스트 도구로 메시지를 전송하면, 수신 서버가 메시지를 **간헐적으로 못 받는** 현상이 발생했다.

로그를 확인하니 MQ(Message Queue) 서버와의 연결이 끊어졌다가 재연결되는 패턴이 반복되고 있었다. 항상 발생하는 것도 아니고, 가끔씩 발생하는 전형적인 **간헐적 장애**였다.

---

## 배경 지식

### STOMP 프로토콜

STOMP(Simple Text Oriented Messaging Protocol)는 MQ 서버와 통신하기 위한 텍스트 기반 프로토콜이다. HTTP처럼 Command, Headers, Body로 구성된 프레임을 주고받는다.

```
SEND                    ← Command
destination:/queue/a    ← Header
content-length:13       ← Header
                        ← 빈 줄 (헤더 끝)
Hello, World!           ← Body
\0\n                    ← 프레임 종료 (NULL + 개행)
```

클라이언트와 서버는 TCP 연결을 유지하면서 주기적으로 **Heartbeat**를 교환한다. 일정 시간 동안 Heartbeat가 오지 않으면 상대방이 죽은 것으로 판단하고 연결을 끊는다.

### 시스템의 스레드 구조

우리 시스템은 3개의 스레드가 하나의 MQ 연결(TCP 소켓)을 공유한다.

```
┌─────────────────────────────────────────────────────┐
│                    공유 자원                          │
│              [ TCP 소켓 연결 ]                        │
└──────────┬──────────┬──────────┬────────────────────┘
           │          │          │
     ┌─────┴───┐ ┌────┴────┐ ┌──┴──────────┐
     │ Read    │ │ Conn    │ │ Main        │
     │ Thread  │ │ Thread  │ │ Thread      │
     │         │ │         │ │             │
     │ 메시지  │ │ 연결관리│ │ 메시지 전송  │
     │ 수신    │ │ Heart-  │ │             │
     │         │ │ beat    │ │             │
     └─────────┘ └─────────┘ └─────────────┘
```

여러 스레드가 동시에 하나의 소켓에 접근하면 데이터가 꼬일 수 있다. 이를 방지하기 위해 **Lock**(잠금)을 사용해야 한다.

---

## 1단계: 원래 코드 분석 - Lock이 없었다

코드를 열어보니 놀라운 사실을 발견했다. Read Thread에서 Lock을 **주석 처리**해놓고 있었다.

```c
// read_thread (원래 코드)
while (running) {
    // ...
    //pthread_mutex_lock(&connection_mutex);     // ← 주석!
    rc = mq_read_frame(conn, &frame, pool);
    //pthread_mutex_unlock(&connection_mutex);   // ← 주석!
    // ...
}
```

왜 주석 처리했을까? 이유는 간단했다. Lock을 걸면 **연결이 더 자주 끊어졌기 때문**이다. 그래서 이전 개발자가 "일단 Lock을 빼자"고 판단한 것이다.

하지만 Lock 없이 운영하면 다른 문제가 생긴다.

### Race Condition 발생

Read Thread가 `mq_read_frame()`으로 메시지를 읽는 도중에, Connection Thread가 연결 끊김을 감지하고 `mq_disconnect()`를 호출하면 어떻게 될까?

```
시간 →

Read Thread:    [ mq_read_frame(connection) .............. ]  💥 CRASH!
                         ↑ connection 사용 중

Conn Thread:              [ mq_disconnect(connection) ]
                                    ↑ connection을 NULL로 만듦
```

Read Thread가 사용 중인 connection이 갑자기 NULL이 되면서 **Segmentation Fault**로 프로세스가 죽는다.

정리하면:
- **Lock 없음** → Race Condition → Crash
- **Lock 있음** → 연결이 끊김

둘 다 문제다. Lock을 걸었을 때 왜 연결이 끊기는지 파헤쳐 봐야 했다.

---

## 2단계: Lock을 걸면 왜 연결이 끊길까?

### mq_read_frame()의 실체

Lock을 걸었을 때 문제의 핵심은 `mq_read_frame()` 함수에 있었다. 이 함수가 어떻게 동작하는지 소스 코드를 열어보았다.

```c
// read_line() - 한 줄을 읽는 함수
while (1) {
    size_t length = 1;  // ← 1바이트씩 읽는다!
    rc = recv(conn->socket, tail->data + i, length, 0);
    // ...
    if (tail->data[i - 1] == '\n') break;  // 개행문자가 올 때까지 반복
}
```

```c
// mq_read_frame() - Body를 읽는 부분
for (int i = 0; i < content_length; i++) {
    size_t read_length = 1;  // ← 여기도 1바이트씩!
    rc = recv(conn->socket, f->body + i, read_length, 0);
    // ...
}
```

`mq_read_frame()` 함수는 소켓에서 데이터를 **1바이트씩** 읽고 있었다. MQ 프레임 하나를 완성하려면:

1. **Command 읽기**: `read_line()` → 1바이트씩 개행까지 반복
2. **Headers 읽기**: `read_line()` × 헤더 수 → 각각 1바이트씩
3. **Body 읽기**: 1바이트씩 × Body 길이

500바이트짜리 메시지를 읽으려면 `recv()`를 **수백 번** 호출해야 한다.

### 40초 타임아웃의 함정

여기에 소켓 타임아웃 설정까지 더해지면 문제가 심각해진다.

```c
#define SOCKET_TIMEOUT_SEC  40  // 소켓 타임아웃: 40초

struct timeval tv = { .tv_sec = SOCKET_TIMEOUT_SEC, .tv_usec = 0 };
setsockopt(conn->socket, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
```

Blocking I/O에서 `recv()`는 데이터가 올 때까지 **최대 40초**를 기다린다. 1바이트 읽기 한 번당 최대 40초가 걸릴 수 있다는 뜻이다.

### Lock 경합 시나리오

이 상황에서 Lock을 추가하면 다음과 같은 일이 벌어진다.

```
시간 →  (단위: 초)
0s                    10s                   20s           30s
│                      │                     │             │
├── Read Thread ───────────────────────────────────────────┤
│   Lock 획득!                                             │
│   mq_read_frame() 실행 중...                             │
│   ├─ read_line(): 1byte... 1byte... (timeout 대기)...   │
│   ├─ read_line(): 1byte... 1byte... (timeout 대기)...   │
│   └─ read_body(): 1byte × N회... (각각 최대 40초)       │
│                                                          │
│                                                Lock 해제 │
│                                                          │
├── Conn Thread ──────────────────────────────────────────┤
│        Lock 대기... Lock 대기... Lock 대기...            │
│        Heartbeat 못 보냄... Heartbeat 못 보냄...         │
│                                                          │
├── MQ 서버 ──────────────────────────────────────────────┤
│        "Heartbeat가 안 와? 연결 끊어야지"                 │
│                                                 연결 끊김!│
└──────────────────────────────────────────────────────────┘
```

Read Thread가 `mq_read_frame()` 안에서 Lock을 잡은 채 수십 초간 블로킹된다. 이 동안 Connection Thread는 Lock을 얻지 못해 Heartbeat를 보낼 수 없다. 클라이언트는 5초마다 Heartbeat를 보내야 하고, 10초 이상 보내지 못하면 서버가 연결을 끊는다. Lock 보유 시간이 40초이므로 타임아웃을 훌쩍 넘긴다.

**이것이 바로 "Lock을 걸면 연결이 끊기는" 이유였다.**

---

## 3단계: 삽질의 시작 - 타임아웃 줄이기

### 첫 시도: 타임아웃을 100ms로

> "타임아웃이 40초라 Lock을 오래 잡는 거잖아? 타임아웃을 줄이면 되지 않을까?"

소켓 타임아웃을 read 100ms, write 40초로 분리 설정했다.

```c
// OS 소켓의 타임아웃을 직접 설정
struct timeval read_tv;
read_tv.tv_sec = 0;
read_tv.tv_usec = 100000;  // 100ms
setsockopt(os_sock, SOL_SOCKET, SO_RCVTIMEO, &read_tv, sizeof(read_tv));

struct timeval write_tv;
write_tv.tv_sec = 40;
write_tv.tv_usec = 0;
setsockopt(os_sock, SOL_SOCKET, SO_SNDTIMEO, &write_tv, sizeof(write_tv));
```

**결과**: `mq_read_frame()`가 1바이트를 읽다가 100ms 안에 다음 데이터가 안 오면 타임아웃으로 실패한다. 1바이트씩 읽는 구조에서는 타임아웃을 줄이면 정상 메시지도 못 읽는다.

### 두 번째 시도: 타임아웃을 2초로

> "100ms가 너무 짧았나. 2초 정도면 충분하지 않을까?"

```c
// read timeout: 2s, write timeout: 2s
struct timeval read_tv = { .tv_sec = 2, .tv_usec = 0 };
setsockopt(os_sock, SOL_SOCKET, SO_RCVTIMEO, &read_tv, sizeof(read_tv));
```

**결과**: 타임아웃이 발생하면 `mq_read_frame()`가 에러를 리턴하고, 기존 코드는 모든 에러를 "연결 끊김"으로 처리해서 불필요한 재연결이 발생했다.

### 세 번째 시도: 타임아웃 에러 처리 추가

> "타임아웃은 정상 동작으로 처리해야겠다."

```c
// ETIMEDOUT은 정상 동작으로 처리
} else if (rc == ETIMEDOUT) {
    print_debug("Socket Read Timeout (normal)\n");
    // 재연결하지 않고 계속 읽기
}
```

타임아웃 에러를 무시하도록 처리했지만, 근본적인 문제는 해결되지 않았다. `mq_read_frame()`가 1바이트씩 읽는 구조에서는 **한 번의 `mq_read_frame()` 호출 안에서** 여러 번 타임아웃이 발생할 수 있었다. `mq_read_frame()` 내부에서 타임아웃이 발생하면 함수 자체가 에러를 리턴한다.

### 네 번째: 코드 정리하면서 발견한 중첩 Lock 문제

타임아웃 조정 작업을 하면서 코드를 정리하다 보니, 다른 문제도 발견했다. `disconnect()` 함수에서 Lock을 잡은 채로 오래 걸리는 작업을 수행하는 **중첩 Lock 패턴**이 있었다.

```c
// disconnect() - 중첩 Lock 코드 (수정 전)
pthread_mutex_lock(&state_mutex);  // Lock A 획득
if (conn_state == STATE_CONNECTED) {
    unsubscribe_all();           // 내부에서 Lock B 사용 (네트워크 I/O)
    send_disconnect_msg();       // 내부에서 Lock B 사용 (네트워크 I/O)

    pthread_mutex_lock(&connection_mutex);  // Lock B 획득 시도
    mq_disconnect(&conn);
    pthread_mutex_unlock(&connection_mutex);
}
conn_state = STATE_DISCONNECTED;
pthread_mutex_unlock(&state_mutex);  // Lock A 해제
```

Lock A(`state_mutex`)를 잡은 상태에서 `unsubscribe_all()`, `send_disconnect_msg()` 같은 네트워크 I/O 작업을 수행한다. 이 작업들은 내부에서 Lock B를 잡았다 풀었다 하면서 **수초 이상** 걸릴 수 있다. 그 동안 다른 스레드가 연결 상태를 확인하려고 Lock A를 잡으려 하면 오래 대기하게 된다.

당장 데드락이 발생하는 코드는 아니었지만, 중첩 Lock 패턴은 코드가 변경될 때 데드락 위험을 만들 수 있는 나쁜 습관이다. Lock 보유 시간을 최소화하도록 수정했다.

**수정**: Lock A를 잡아서 state를 복사한 후 **즉시 해제**하고, 이후 작업은 Lock A 없이 수행하도록 변경했다.

```c
// disconnect() - 수정 후
pthread_mutex_lock(&state_mutex);
target_state = TARGET_DISCONNECT;
int saved_state = conn_state;              // state 복사
conn_state = STATE_DISCONNECTED;
pthread_mutex_unlock(&state_mutex);        // 즉시 해제!

// Lock A 없이 cleanup 수행
if (saved_state == STATE_CONNECTED) {
    unsubscribe_all();
    send_disconnect_msg();
    // ...
}
```

같은 패턴이 `conn_thread()`에도 있어서 함께 수정했다.

---

## 4단계: 근본적 해결 - 읽기 방식 자체를 바꿔야 한다

타임아웃을 아무리 조정해도, `mq_read_frame()`가 1바이트씩 읽는 구조가 바뀌지 않으면 문제는 해결되지 않는다.

> 핵심 질문: **Lock을 잡고 있는 시간을 최소화할 수는 없을까?**

### 발상의 전환: 읽기와 파싱을 분리

기존 `mq_read_frame()`는 **읽기와 파싱을 동시에** 수행한다. 소켓에서 1바이트 읽고, 프레임 구조를 분석하고, 또 1바이트 읽고... 이 모든 과정이 Lock 안에서 일어난다.

이걸 두 단계로 분리하면 어떨까?

```
[기존 방식 - mq_read_frame()]
Lock 획득 ──── 1byte 읽기 → 파싱 → 1byte 읽기 → 파싱 → ... ──── Lock 해제
              ├──────────── 수초 ~ 수십초 ────────────────────┤

[새 방식 - 버퍼링]
Lock 획득 ──── 청크 단위로 한 번에 읽기 ──── Lock 해제 ──── 파싱 (Lock 없이)
              ├──── 최대 500ms ────────┤
```

1. Lock을 잡고 소켓에서 데이터를 **청크 단위(1KB)**로 한 번에 읽어 **버퍼에 저장**
2. Lock을 **즉시 해제**
3. 버퍼에 쌓인 데이터를 **Lock 없이** 파싱하여 완성된 프레임 추출

### 구현

#### 버퍼 구조

```c
// 읽기 버퍼
static char *read_buffer = NULL;
static size_t read_buffer_size = 0;      // 할당된 크기
static size_t read_buffer_used = 0;      // 사용 중인 크기
#define CHUNK_SIZE 1024                  // 한 번에 읽는 크기
```

#### read_and_process() - 핵심 읽기 함수

```c
int read_and_process(void) {
    // 1. 버퍼 공간 확보 (buffer mutex)
    pthread_mutex_lock(&buffer_mutex);
    if (read_buffer == NULL) {
        read_buffer = malloc(CHUNK_SIZE * 2);
        // ...
    }
    pthread_mutex_unlock(&buffer_mutex);

    // 2. 소켓에서 읽기 (connection mutex - 최대 500ms)
    char temp_buffer[CHUNK_SIZE];
    size_t recv_len = CHUNK_SIZE;

    pthread_mutex_lock(&connection_mutex);   // Lock!
    rc = recv(socket, temp_buffer, recv_len, 0);
    pthread_mutex_unlock(&connection_mutex); // 즉시 Unlock!
    // ↑ 여기서 Lock 보유 시간은 최대 500ms (소켓 타임아웃)

    // 3. 버퍼에 추가 (buffer mutex)
    pthread_mutex_lock(&buffer_mutex);
    memcpy(read_buffer + read_buffer_used, temp_buffer, recv_len);
    read_buffer_used += recv_len;
    pthread_mutex_unlock(&buffer_mutex);

    // 4. 버퍼에서 완성된 프레임 파싱 (connection Lock 없이!)
    while (1) {
        pthread_mutex_lock(&buffer_mutex);
        int result = process_frame();
        pthread_mutex_unlock(&buffer_mutex);
        if (result == 0) break;  // 더 이상 완성된 프레임 없음
    }
}
```

#### parse_frame() - 버퍼 기반 MQ 프레임 파서

기존 `mq_read_frame()`는 소켓에서 직접 읽으면서 파싱했지만, 새 함수는 **메모리 버퍼에서** 파싱한다.

```c
static parse_result_t parse_frame(mq_frame **frame_out) {
    if (read_buffer_used == 0)
        return PARSE_RESULT_INCOMPLETE;  // 데이터 부족

    // 1. Command 파싱 (첫 번째 \n까지)
    while (pos < read_buffer_used && read_buffer[pos] != '\n') pos++;
    if (pos >= read_buffer_used)
        return PARSE_RESULT_INCOMPLETE;  // 아직 Command가 완성되지 않음

    // Heartbeat 체크 (빈 줄 = \n만 있으면 heartbeat)
    if (pos == 0) {
        memmove(read_buffer, read_buffer + 1, read_buffer_used - 1);
        read_buffer_used--;
        return PARSE_RESULT_HEARTBEAT;
    }

    // 2. Headers 파싱 (\n\n이 올 때까지)
    // ...

    // 3. Body 파싱 (content-length 또는 \0\n까지)
    // ...

    // 4. 파싱 완료 - 사용한 데이터를 버퍼에서 제거
    size_t remaining = read_buffer_used - pos;
    memmove(read_buffer, read_buffer + pos, remaining);
    read_buffer_used = remaining;

    *frame_out = frame;
    return PARSE_RESULT_COMPLETE;
}
```

핵심은 `PARSE_RESULT_INCOMPLETE`이다. 버퍼에 데이터가 부족하면 파싱을 중단하고, 다음 소켓 읽기에서 데이터가 더 들어오면 이어서 파싱한다. 소켓에서 직접 블로킹하며 기다리지 않는다.

### 소켓 타임아웃: 500ms

```c
struct timeval tv = { .tv_sec = 0, .tv_usec = 500000 };  // 500ms
setsockopt(socket, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
```

40초에서 500ms로 줄였다. Lock을 잡고 있는 최대 시간이 **40초 → 500ms**로 단축된다.

---

## 전후 비교

```
[기존 방식]
mq_read_frame() 1회 호출:
  Lock ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Unlock
  │   read_line (1byte×N, 각 최대 40s)  │
  │   read_line (1byte×N, 각 최대 40s)  │
  │   read_body (1byte×N, 각 최대 40s)  │
  └──────── 수초 ~ 수십초 ──────────────┘

[버퍼링 방식]
read_and_process() 1회 호출:
  Lock ━━━━ Unlock    (connection lock)
  │ recv()  │
  └ ~500ms ─┘
              Lock ━━━ Unlock  (buffer lock, μs 단위)
              │ parse │
              └───────┘
```

| 항목 | 기존 | 개선 후 |
|------|------|---------|
| 소켓 타임아웃 | 40초 | 500ms |
| 1회 읽기 단위 | 1바이트 | 1KB 청크 |
| Lock 보유 시간 | 수초~수십초 | 최대 500ms |
| 프레임 파싱 | Lock 내에서 | Lock 없이 |
| Heartbeat 지연 | 빈번 | 최소화 (최대 500ms) |

---

## 리팩토링

버퍼링 구현 후, 코드 품질을 높이기 위해 추가 작업을 진행했다.

- **함수 추출**: `read_thread`에 몰려있던 로직을 `read_and_process()`, `process_frame()` 등으로 분리
- **상태 관리 함수 추출**: `create_connection()`, `set_connection_state()` 분리
- **read_buffer thread-safety**: `buffer_mutex` 추가, `clear_buffer()` 함수 추출
- **Heartbeat 처리 개선**: `PARSE_RESULT_ERROR`로 heartbeat를 처리하던 것을 별도 `PARSE_RESULT_HEARTBEAT` enum으로 분리
- **단위 테스트 추가**: `read_and_process()`를 `TESTABLE`로 변경하여 테스트 가능하도록 개선

---

## 교훈

### Blocking I/O + Lock = 위험

Blocking I/O와 Lock을 함께 사용할 때는 반드시 **Lock 보유 시간**을 고려해야 한다. I/O 대기 시간 동안 Lock을 보유하면 다른 스레드가 모두 멈출 수 있다.

### Lock 안에서는 빠른 작업만

Lock을 잡고 있는 동안에는 **즉시 완료되는 작업**만 수행해야 한다. 네트워크 I/O, 파일 I/O, 긴 계산 등은 Lock 밖에서 수행하고, Lock 안에서는 메모리 복사나 상태 변경 같은 빠른 작업만 해야 한다.

### 읽기와 처리를 분리하라

데이터를 읽는 것(I/O)과 처리하는 것(파싱)을 분리하면 각 단계별로 Lock을 짧게 잡을 수 있다. 버퍼링은 이 분리를 가능하게 하는 핵심 기법이다.

### 삽질도 가치가 있다

타임아웃을 100ms, 2초로 바꿔보고, 소켓 옵션으로 바꿔보고, 데드락도 발견하고... 결국 최종 해결책은 "읽기 방식 자체를 바꾸는 것"이었지만, 삽질 과정에서 데드락 버그와 에러 처리 누락 같은 다른 문제들도 발견해서 함께 고칠 수 있었다.

---

> 다음 글에서는 버퍼링으로도 해결되지 않은 **남은 문제**와, 이를 **rwlock**으로 완전히 해결하는 과정을 다룹니다.
