---
layout: post
title: "Claude API로 PR 코드 리뷰 자동 검증 서버 만들기"
date: 2026-03-18 12:00:00 +0900
categories: [백엔드]
tags: [Claude, AI, Bitbucket, FastAPI, Docker, Python, 코드리뷰, 자동화]
mermaid: true
---

> Bitbucket webhook 권한도 없고, 외부 서버도 없는 상황에서 — PR 코드 리뷰를 Claude가 자동으로 검증하고, 유효한 항목은 직접 코드에 반영까지 해주는 서버를 만든 기록.

---

## 배경

팀에서 PR 리뷰가 올라오면 항상 이런 고민이 생겼다.

- 리뷰어가 지적한 내용이 실제로 타당한가?
- 이전에 이미 논의된 내용을 다시 올린 건 아닌가?
- 유효한 이슈 중 어떤 걸 먼저 처리해야 하나?

이걸 매번 사람이 판단하는 건 피로감이 크다. Claude API를 써서 자동으로 검증하고, 결과를 저장해두면 어떨까 싶었다.

목표는 간단했다.

```
임차섭 리뷰어가 "PR 리뷰 결과" 키워드로 댓글을 달면
→ Claude가 리뷰 내용을 검증하고
→ 결과를 파일로 저장
→ 브라우저에서 확인
```

---

## 첫 번째 벽: webhook을 설정할 수가 없다

가장 자연스러운 방법은 Bitbucket webhook이었다. 댓글이 달리면 서버로 POST 요청이 오는 구조.

근데 문제가 있었다.

1. 저장소 Settings 권한이 없다 → webhook 직접 등록 불가
2. 서버가 로컬이다 → Bitbucket이 접근할 공인 URL이 없다

### 대안: Bitbucket API 폴링

webhook 대신 주기적으로 API를 긁어서 새 댓글을 확인하는 방식으로 방향을 바꿨다.

```
60초마다 → 전체 저장소 조회 → 열린 PR 조회 → 댓글 확인 → 조건 맞으면 Claude 호출
```

폴링이라 실시간은 아니지만, 코드 리뷰 검증 용도로는 1분 딜레이가 전혀 문제없었다.

핵심은 **이미 처리한 댓글을 다시 처리하지 않는 것**. 처리한 댓글 ID를 파일에 저장해서 서버 재시작 후에도 유지되게 했다.

```python
# results/.seen_comments.json 에 댓글 ID 목록 영속화
def load_seen_ids() -> set[int]:
    if os.path.exists(SEEN_FILE):
        with open(SEEN_FILE, "r") as f:
            return set(json.load(f))
    return set()
```

---

## 구조

```
Bitbucket API (폴링)
        ↓
  poller.py (60초 간격)
  - PR 작성자 = 박지수 필터
  - 리뷰어 = 임차섭 필터
  - 키워드 "PR 리뷰 결과" 필터
        ↓
  claude_validator.py
  - PR diff (실제 코드 변경사항)
  - 이전 댓글 이력
  - 검증 대상 리뷰
        ↓
  result_writer.py
  → results/{repo}_pr{id}_{timestamp}.md
        ↓
  viewer (FastAPI)
  → localhost:8000 브라우저에서 확인
```

---

## Claude에게 뭘 주느냐가 핵심이다

처음엔 리뷰 댓글 텍스트만 Claude에게 넘겼다. 결과가 나오긴 하는데, 실제 코드를 보지 않으니 "기술적으로 타당한지"만 판단했다. 코드에 문제가 실제로 있는지는 알 수 없었다.

### PR diff 추가

Bitbucket API로 PR diff를 가져와서 같이 넘겼다.

```
GET /2.0/repositories/{workspace}/{repo}/pullrequests/{id}/diff
```

이제 Claude는 리뷰어가 지적한 코드가 실제로 변경사항에 존재하는지 확인할 수 있게 됐다.

### 이전 댓글 이력 추가

같은 PR의 이전 댓글도 같이 넘겼다. "이미 수정 완료"라고 답한 이슈를 리뷰어가 다시 올리는 경우를 잡아내기 위해서다.

최종적으로 Claude에게 넘기는 입력:

```
1. PR diff (실제 코드 변경사항)
2. 이전 댓글 이력 (시간순)
3. 검증 대상 리뷰 (최신 댓글)
```

---

## 검증 결과 형식

Claude 출력을 구조화하기 위해 시스템 프롬프트에 판정 기준과 출력 형식을 명시했다.

**판정 기준:**

| 판정 | 기준 |
|------|------|
| ✅ 유효 | 주장이 타당하며 실제 수정 필요 |
| ❌ 무효 — 이미 반영됨 | 이전 댓글에서 수정 완료 처리됨 |
| ❌ 무효 — 이전 검토에서 기각됨 | 이전에 유지 결정된 이슈 재제기 |
| ❌ 무효 — 사실 오류 | diff와 다른 전제로 지적 |
| ❌ 무효 — diff에 없음 | 지적한 코드가 변경사항에 없음 |
| ⚠️ 부분 유효 | 일부만 유효하거나 개선 권고 수준 |

**액션 분류:**

- **A. 바로 수정 가능**: 정답이 하나인 경우 (오타, 잘못된 명칭 등)
- **B. 인간이 결정해야 할 일**: 트레이드오프나 비즈니스 판단 필요
- **C. 수정 불필요**: 무효 이슈

---

## HTML 뷰어

결과 파일이 쌓이면 어딘가서 봐야 한다. FastAPI에 간단한 HTML 뷰어를 붙였다.

```
localhost:8000        → 전체 결과 목록
localhost:8000/results/{파일명}  → 상세 보기 + 코드 적용 폼
```

상세 페이지에서 marked.js로 마크다운을 렌더링하고, B 항목(인간이 결정해야 할 일)에 대한 입력 폼을 같이 보여줬다.

B 항목 폼 예시:

```
┌──────────────────────────────────────────┐
│ 🔸 이슈 2 — 캐시 전략 선택              │
│  - 문제: 조회 성능 저하                  │
│  - 대안 A: Redis 캐시                    │
│  - 대안 B: Local Cache                   │
│ 결정 입력                                │
│ [ textarea                              ] │
└──────────────────────────────────────────┘
│ 추가 요청사항 (선택)                      │
│ [ textarea                              ] │
│ [  Apply 및 Push  ]                       │
```

---

## Apply: 리뷰 반영을 코드에 직접

유효 이슈 + 인간 결정을 입력하고 Apply를 누르면 실제 코드를 수정하고 push까지 한다.

### 전체 파일 교체 대신 git patch

처음엔 Claude에게 "수정된 파일 전체 내용을 JSON으로 줘"라고 했다. 근데 이러면 문제가 있다.

- Claude가 파일 전체를 재작성하다가 의도치 않은 부분까지 건드릴 수 있다
- 변경이 정확히 어디서 일어났는지 추적하기 어렵다

**unified diff(patch) 방식으로 바꿨다.**

Claude에게 unified diff 형식으로만 출력하게 하고:

```diff
COMMIT_MESSAGE: fix: UserService 트랜잭션 처리 보완

--- a/src/main/java/com/example/UserService.java
+++ b/src/main/java/com/example/UserService.java
@@ -45,6 +45,7 @@
     public void updateUserAuth(Long userId) {
+        @Transactional
         userRepository.updateStatus(userId);
```

`git apply --check`로 먼저 유효성을 검사하고, 통과하면 `git apply`로 적용한다.

```python
_git(["apply", "--check", "review.patch"], cwd=temp_dir)  # 검증
_git(["apply", "review.patch"], cwd=temp_dir)              # 적용
_git(["commit", "-m", commit_message], cwd=temp_dir)
_git(["push", "origin", branch], cwd=temp_dir)
```

이렇게 하면 리뷰에서 언급한 라인만 정확하게 수정된다.

### SSH 인증

push는 Bitbucket App Password(HTTPS) 또는 SSH 키로 인증한다. SSH 방식을 택했다.

Dockerfile에서 빌드 시 SSH 키를 컨테이너에 심고, Bitbucket known_hosts를 등록해둔다.

```dockerfile
COPY ssh_key /root/.ssh/id_ed25519
RUN chmod 600 /root/.ssh/id_ed25519
RUN ssh-keyscan bitbucket.org >> /root/.ssh/known_hosts
```

clone URL은 `git@bitbucket.org:workspace/repo.git` 형식을 쓴다.

---

## Apply 흐름 전체

```
Apply 버튼 클릭
  ↓
Bitbucket API → PR 브랜치명, diffstat 조회
  ↓
git clone --depth=1 --branch {branch} (SSH)
  ↓
변경된 파일 읽기 (라인 번호 포함)
  ↓
Claude → unified diff 생성
         (✅ 유효 항목만, 인간 결정 반영)
  ↓
git apply --check (패치 유효성 검사)
  ↓
git apply → git commit → git push
  ↓
임시 디렉토리 정리
  ↓
결과 반환 (브랜치, 커밋 메시지, 수정 파일 목록)
```

---

## .env 구성

```
ANTHROPIC_API_KEY=...
BITBUCKET_EMAIL=...
BITBUCKET_TOKEN=...        # 읽기 권한 (폴링용)
BITBUCKET_WORKSPACE=telcoware
POLL_INTERVAL=60
PR_AUTHOR=박지수
REVIEW_AUTHOR=임차섭
GIT_AUTHOR_NAME=jayasou
GIT_AUTHOR_EMAIL=jayasou@telcoware.com
```

---

## 실행

```bash
docker compose up --build
```

서버가 뜨면 즉시 폴링을 시작한다.

```
[INFO] Bitbucket 폴링 시작 (간격: 60초, 워크스페이스: telcoware)
[INFO] 대상: PR작성자=박지수, 리뷰어=임차섭, 키워드='PR 리뷰 결과'
[INFO] 기존 처리 댓글 9개 로드
[INFO] 폴링 시작 — 26개 저장소 확인 중
[INFO] [sks-smtsty-backend] PR #259 (박지수) 리뷰 감지 — Claude 검증 시작
[INFO] diff 크기: 18,432 bytes
[INFO] 검증 결과 저장: /app/results/sks-smtsty-backend_pr259_20260318_054446.md
```

브라우저에서 `http://localhost:8000` 접속하면 결과 목록을 볼 수 있다.

---

## 마무리

webhook 권한도 없고 외부 서버도 없는 제약 안에서, 폴링 방식으로 우회했다. 핵심은 Claude에게 **리뷰 텍스트만 주지 않고 실제 코드(diff)를 같이 주는 것**이었다. 그래야 "코드에 실제로 문제가 있는가"를 판단할 수 있다.

Apply 기능은 아직 실전 투입 전이다. `git apply --check`로 패치 검증을 먼저 하는 구조라 실패해도 코드가 망가지진 않는다. SSH 키 쓰기 권한이 확인되면 실전에서도 써볼 예정이다.
