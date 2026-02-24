---
layout: post
title: "Mutex에서 RWLock으로 - 동시성 문제의 마지막 퍼즐"
date: 2026-02-23 01:00:00 +0900
categories: [동시성]
tags: [C, RWLock, Mutex, TCP, FullDuplex, 멀티스레드, 동시성]
---

> 이전 글에서 버퍼링 기반 읽기로 Lock 보유 시간을 40초에서 500ms로 줄였다. 하지만 이야기는 여기서 끝나지 않았다.

---

## 버퍼링 이후에도 남은 문제

버퍼링을 도입한 후, MQ 연결이 끊기는 현상은 사라졌다. 하지만 코드를 다시 들여다보니 한 가지가 마음에 걸렸다.

```c
// read_and_process() - 소켓 읽기 부분
pthread_mutex_lock(&connection_mutex);   // Lock 획득
rc = recv(socket, temp_buffer, recv_len, 0);  // 최대 500ms 블로킹
pthread_mutex_unlock(&connection_mutex); // Lock 해제
```

Read Thread가 `recv()`에서 **최대 500ms** 동안 Lock을 잡고 있다. 이 500ms 동안 다른 스레드에서 메시지를 보내려면 **같은 mutex**를 잡아야 한다.

```c
// send_message() - 메시지 전송 부분
pthread_mutex_lock(&connection_mutex);   // ← 여기서 대기!
rc = mq_write_frame(conn, &frame, pool);
pthread_mutex_unlock(&connection_mutex);
```

```
시간 →

Read Thread:   [===== Lock 잡고 recv() 대기 (500ms) =====]
                                                           [=== Lock ===]
Main Thread:        [ Lock 대기... 대기... 대기... ]
                    ↑ 메시지 보내고 싶은데 못 보냄

Conn Thread:            [ Lock 대기... 대기... ]
                        ↑ Heartbeat 보내고 싶은데 못 보냄
```

40초에서 500ms로 줄인 것은 큰 개선이었지만, **recv()를 하는 동안 send()를 못 하는 구조** 자체는 그대로였다. 500ms라는 지연은 연결이 끊길 정도는 아니지만, 메시지 전송의 응답 시간에 영향을 줄 수 있다.

> "recv()하는 동안 send()를 정말 못 하는 게 맞나?"

이 질문이 해결의 실마리가 되었다.

---

## TCP 소켓은 Full-Duplex다

TCP 소켓에 대해 알아보니, 중요한 사실을 발견했다.

> **TCP 소켓은 Full-Duplex**이다. 즉, 하나의 소켓에서 **읽기(recv)와 쓰기(send)를 동시에** 할 수 있다.

내부적으로 TCP 소켓은 **송신 버퍼와 수신 버퍼가 분리**되어 있다.

```
┌──────────────────────────────────────┐
│            TCP 소켓 내부              │
│                                      │
│  ┌──────────────┐  ┌──────────────┐  │
│  │  수신 버퍼    │  │  송신 버퍼    │  │
│  │  (recv용)    │  │  (send용)    │  │
│  │              │  │              │  │
│  │  ← 서버에서  │  │  → 서버로    │  │
│  │    데이터     │  │    데이터     │  │
│  └──────────────┘  └──────────────┘  │
│        ↑                  ↑          │
│    recv()가           send()가       │
│    여기서 읽음        여기에 씀       │
│                                      │
│    서로 다른 버퍼 → 간섭 없음!       │
└──────────────────────────────────────┘
```

`recv()`는 수신 버퍼에서 데이터를 꺼내고, `send()`는 송신 버퍼에 데이터를 넣는다. 서로 다른 버퍼를 사용하므로 **동시에 실행해도 안전**하다.

그런데 우리 코드는 `recv()`와 `send()`를 **같은 mutex**로 직렬화하고 있었다. 동시에 실행해도 되는 것을 굳이 기다리게 만들고 있었던 셈이다.

### 그러면 Lock이 아예 필요 없나?

아니다. Lock이 필요한 경우가 있다.

- **recv()와 send()**: 동시 실행 가능 → Lock 불필요 (서로 다른 버퍼)
- **send()와 send()**: 동시 실행 불가 → Lock 필요 (같은 송신 버퍼에 두 스레드가 쓰면 데이터가 인터리빙됨)
- **disconnect()**: recv/send가 모두 끝난 후에 실행해야 함 → 독점 Lock 필요

정리하면:

| 동작 | 필요한 보호 수준 |
|------|----------------|
| recv() | connection이 살아있는지만 보장 |
| send() | connection 보장 + 다른 send()와 직렬화 |
| disconnect() | 모든 I/O가 끝난 후 독점 실행 |

이 패턴... 어디서 본 적 있지 않은가?

---

## RWLock이 딱 맞는 상황

**Reader-Writer Lock (RWLock)**은 정확히 이런 상황을 위해 존재한다.

```
┌─────────────────────────────────────────────────────┐
│                    RWLock                             │
│                                                      │
│  Read Lock (공유)        Write Lock (독점)            │
│  ┌─────────────────┐    ┌─────────────────┐         │
│  │ 여러 스레드가    │    │ 하나의 스레드만  │         │
│  │ 동시에 획득 가능 │    │ 획득 가능        │         │
│  │                 │    │                 │         │
│  │ 다른 Read Lock  │    │ 모든 Read Lock이 │         │
│  │ 과 공존 가능    │    │ 해제될 때까지    │         │
│  │                 │    │ 대기             │         │
│  └─────────────────┘    └─────────────────┘         │
└─────────────────────────────────────────────────────┘
```

- **Read Lock (rdlock)**: 여러 스레드가 동시에 잡을 수 있다
- **Write Lock (wrlock)**: 하나의 스레드만 잡을 수 있고, 모든 Read Lock이 풀릴 때까지 대기한다

이걸 우리 상황에 대입하면:

| 동작 | Lock 종류 | 이유 |
|------|----------|------|
| recv() | **rdlock** | connection 보호. send()와 동시 실행 가능 |
| send() | **write_mutex + rdlock** | write_mutex로 다른 send()와 직렬화 + rdlock으로 connection 보호 |
| disconnect() | **wrlock** | 모든 rdlock(recv/send)이 해제될 때까지 대기 후 독점 실행 |

```
시간 →

Read Thread:   [=== rdlock: recv() ===]          [=== rdlock: recv() ===]
Main Thread:       [== rdlock: send() ==]    동시 실행!
Conn Thread:           [= rdlock: heartbeat =]

                                              ... 모든 rdlock 해제 후 ...
Disconnect:                                                 [== wrlock ==]
```

recv()와 send()가 동시에 실행된다! 500ms 대기가 사라진다.

---

## 구현

### 1. 인프라 추가

```c
#include <pthread.h>

// 기존 mutex (write 직렬화 용도로 유지)
static pthread_mutex_t connection_mutex = PTHREAD_MUTEX_INITIALIZER;

// 새로 추가: connection lifecycle 보호
static pthread_rwlock_t connection_rwlock = PTHREAD_RWLOCK_INITIALIZER;
```

`connection_mutex`는 삭제하지 않고 **send() 간 직렬화**를 위해 유지한다. `connection_rwlock`이 connection의 생존을 보장하는 역할을 맡는다.

### 2. Helper 함수 3개

**safe_write()** - 안전한 쓰기

```c
static int safe_write(mq_frame *frame) {
    pthread_mutex_lock(&connection_mutex);       // 1. 다른 send()와 직렬화
    pthread_rwlock_rdlock(&connection_rwlock);    // 2. connection 보호 (공유)
    if (conn == NULL) {
        pthread_rwlock_unlock(&connection_rwlock);
        pthread_mutex_unlock(&connection_mutex);
        return -1;
    }
    int rc = mq_write_frame(conn, frame);
    pthread_rwlock_unlock(&connection_rwlock);
    pthread_mutex_unlock(&connection_mutex);
    return rc;
}
```

**safe_write_raw()** - Heartbeat 전용 쓰기

```c
static int safe_write_raw(const char *data, size_t size) {
    pthread_mutex_lock(&connection_mutex);
    pthread_rwlock_rdlock(&connection_rwlock);
    if (conn == NULL) {
        pthread_rwlock_unlock(&connection_rwlock);
        pthread_mutex_unlock(&connection_mutex);
        return -1;
    }
    int rc = mq_write_raw(conn, data, size);
    pthread_rwlock_unlock(&connection_rwlock);
    pthread_mutex_unlock(&connection_mutex);
    return rc;
}
```

**safe_disconnect()** - 안전한 연결 종료

```c
static void safe_disconnect(void) {
    pthread_rwlock_wrlock(&connection_rwlock);  // 독점! 모든 I/O 완료 대기
    if (conn != NULL) {
        mq_disconnect(&conn);
    }
    pthread_rwlock_unlock(&connection_rwlock);
}
```

### 3. 읽기 경로 변경

```c
// read_and_process() - 변경 전
pthread_mutex_lock(&connection_mutex);
rc = recv(conn->socket, temp_buffer, recv_len, 0);
pthread_mutex_unlock(&connection_mutex);

// read_and_process() - 변경 후
pthread_rwlock_rdlock(&connection_rwlock);   // rdlock!
rc = recv(conn->socket, temp_buffer, recv_len, 0);
pthread_rwlock_unlock(&connection_rwlock);
```

`mutex` → `rdlock`으로 바꾸는 것이 전부다. 이 한 줄의 변경으로 recv()와 send()가 동시 실행 가능해진다.

### 4. 쓰기 경로 변경

모든 `mq_write_frame()` 호출부를 `safe_write()`로 교체했다.

```c
// 변경 전 (10곳 모두 이 패턴)
pthread_mutex_lock(&connection_mutex);
rc = mq_write_frame(conn, &frame);
pthread_mutex_unlock(&connection_mutex);

// 변경 후
rc = safe_write(&frame);
```

### 5. 연결/해제 경로 변경

```c
// create_connection() - 연결 시
pthread_rwlock_wrlock(&connection_rwlock);   // wrlock! (독점)
if (conn != NULL) {
    mq_disconnect(&conn);
}
rc = mq_connect(&conn, ...);
pthread_rwlock_unlock(&connection_rwlock);

// disconnect 경로 (3곳) - 모두 safe_disconnect()로 교체
safe_disconnect();
```

---

## 작업 중 발견한 메모리 누수

rwlock 변경을 적용하면서 기존 코드를 하나하나 확인하다가, **메모리 누수**를 발견했다.

```c
// 변경 전 - send_message()
void *frame_mem = malloc(frame_size);
// ... frame 구성 ...

pthread_mutex_lock(&connection_mutex);
rc = mq_write_frame(conn, &frame);
pthread_mutex_unlock(&connection_mutex);
if (rc != 0) return rc;  // ← 여기가 문제!

free(frame_mem);  // ← 실패 시 여기에 도달하지 않음
```

`rc != 0`이면 즉시 `return`한다. 그런데 `return` 전에 `frame_mem`을 `free`하지 않는다. **매 실패마다 메모리가 새는 것**이다.

```c
// 변경 후
rc = safe_write(&frame);
if (rc != 0) {
    free(frame_mem);  // 실패해도 메모리 해제!
    return rc;
}

free(frame_mem);
```

같은 패턴이 5개 함수에 있었다:
- `send_message()`
- `send_message_with_headers()`
- `send_unsubscribe_msg()`
- `send_subscribe_msg()`
- `send_subscribe_msg_with_selector()`

모두 수정했다.

---

## Deadlock 안전성 검증

Lock이 2개 이상이면 항상 **Deadlock** 가능성을 검증해야 한다.

### Lock 획득 순서

```
safe_write():      mutex → rdlock     (순서: 1 → 2)
safe_write_raw():  mutex → rdlock     (순서: 1 → 2)
recv (read_and_process):   rdlock only (순서: 2만)
disconnect (safe_disconnect): wrlock only (순서: 2만)
connect (create_connection):  wrlock only (순서: 2만)
```

Lock 획득 순서가 항상 **mutex → rwlock**으로 일관된다. 역순으로 잡는 코드가 없으므로 Deadlock은 발생하지 않는다.

### 동시성 매트릭스

| | recv (rdlock) | send (mutex+rdlock) | disconnect (wrlock) |
|---|---|---|---|
| **recv** | 동시 가능 | 동시 가능 | 대기 |
| **send** | 동시 가능 | 직렬화 (mutex) | 대기 |
| **disconnect** | 대기 | 대기 | - |

- recv + send: 둘 다 rdlock이므로 동시 실행 가능
- send + send: mutex로 직렬화 (MQ 프레임 인터리빙 방지)
- disconnect: wrlock이므로 모든 rdlock이 해제될 때까지 대기 후 독점 실행

---

## 최종 결과

```
[Mutex만 사용 (버퍼링 적용 후)]
Read Thread:  [=== mutex: recv() (500ms) ===]
Main Thread:       [ mutex 대기... ][ send() ]
                   └── 최대 500ms 대기 ──┘

[RWLock 적용 후]
Read Thread:  [=== rdlock: recv() (500ms) ===]
Main Thread:      [= rdlock: send() =]            ← 동시 실행!
                  └── 대기 시간: 0 ──┘
```

| 항목 | Mutex | RWLock |
|------|-------|--------|
| recv 중 send | 최대 500ms 대기 | **즉시 실행** |
| recv 중 heartbeat | 최대 500ms 대기 | **즉시 실행** |
| send 중 send | 직렬화 | 직렬화 (동일) |
| disconnect 시 | 즉시 실행 | I/O 완료 대기 후 실행 (더 안전) |

---

## 전체 여정 요약

```
[원래 상태]
Lock 없음 + mq_read_frame() 1바이트씩 + 40초 타임아웃
→ Race Condition → Crash

    ↓ Lock 추가

[1차 시도]
Mutex + mq_read_frame() 1바이트씩 + 40초 타임아웃
→ Lock 경합 → Heartbeat 지연 → 연결 끊김

    ↓ 타임아웃 조정 삽질 (100ms → 2s → API 정리)
    ↓ 데드락 발견 및 수정

[2차 시도 - 버퍼링 도입]
Mutex + 버퍼링(청크 읽기) + 500ms 타임아웃
→ Lock 보유 시간 40초 → 500ms로 단축
→ 연결 안정화!
→ 하지만 recv 중 send 불가 (500ms 지연)

    ↓ TCP Full-Duplex 인사이트

[최종 - RWLock 도입]
RWLock(rdlock/wrlock) + 버퍼링 + 500ms 타임아웃
→ recv/send 동시 실행 가능
→ send 지연 시간: 0
→ disconnect는 wrlock으로 안전하게 보호
→ 메모리 누수도 추가 수정
```

---

## 교훈

### 동시성 문제에는 계층이 있다

처음에는 "Lock을 걸면 해결되겠지"라고 생각했지만, 실제로는 여러 계층의 문제가 중첩되어 있었다.

1. **보호 대상이 뭔지** 정확히 파악해야 한다 (connection의 생존 보장)
2. **어느 수준의 보호가 필요한지** 구분해야 한다 (공유 접근 vs 독점 접근)
3. **I/O와 Lock의 상호작용**을 고려해야 한다 (Blocking I/O + Lock = 위험)

### Mutex가 항상 답은 아니다

Mutex는 가장 간단한 동기화 도구이지만, 모든 상황에 최적은 아니다.

RWLock은 보통 "읽기가 많고 쓰기가 적은" **데이터 접근** 패턴에서 사용된다. 우리 경우는 조금 다르다. recv()와 send()는 데이터를 "읽고 쓰는" 것이지만, connection 입장에서 보면 둘 다 connection을 **"사용하는"** 것이다. disconnect()만 connection을 **"파괴하는"** 것이다.

| | 전통적 RWLock | 우리 코드의 RWLock |
|---|---|---|
| rdlock | 데이터를 **읽기만** 하는 스레드 | connection을 **사용하는** 스레드 (recv/send) |
| wrlock | 데이터를 **수정**하는 스레드 | connection을 **파괴/생성**하는 스레드 (disconnect/connect) |

**"공유 자원을 사용하는 것(공유 접근) vs 공유 자원 자체를 변경하는 것(독점 접근)"** 이라는 관점으로 보면, RWLock이 정확히 맞는 도구였다.

### 삽질이 쌓이면 설계가 된다

처음부터 RWLock을 떠올린 것은 아니었다. 타임아웃을 줄여보고, 데드락을 고치고, 버퍼링을 도입하고... 이 과정에서 "recv()와 send()는 동시에 해도 되는데 왜 기다리지?"라는 질문이 자연스럽게 나왔다. 문제를 깊이 이해하면 올바른 도구가 보인다.

### 리팩토링은 버그를 드러낸다

rwlock 적용을 위해 모든 `mq_write_frame()` 호출부를 `safe_write()`로 교체하면서, 기존 코드의 메모리 누수(frame_mem 미해제)를 발견할 수 있었다. 코드를 하나하나 확인하는 과정 자체가 숨어있던 버그를 찾아내는 기회가 된다.
