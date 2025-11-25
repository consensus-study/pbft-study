# PBFT (Practical Byzantine Fault Tolerance) 완벽 가이드

> **원 논문**: "Practical Byzantine Fault Tolerance" - Miguel Castro and Barbara Liskov (OSDI '99)

---

## 목차

1. [역사적 배경](#1-역사적-배경)
2. [비잔틴 장군 문제](#2-비잔틴-장군-문제)
3. [왜 3f+1 노드가 필요한가?](#3-왜-3f1-노드가-필요한가)
4. [View란 무엇인가?](#4-view란-무엇인가)
5. [시스템 모델](#5-시스템-모델)
6. [3-Phase Protocol 완전 정복](#6-3-phase-protocol-완전-정복)
7. [왜 투표를 두 번 하나? (PREPARE vs COMMIT)](#7-왜-투표를-두-번-하나-prepare-vs-commit)
8. [View Change 프로토콜](#8-view-change-프로토콜)
9. [Garbage Collection & Checkpoints](#9-garbage-collection--checkpoints)
10. [Safety와 Liveness](#10-safety와-liveness)
11. [성능 최적화](#11-성능-최적화)
12. [Tendermint와 비교](#12-tendermint와-비교)
13. [구현 예시](#13-구현-예시)

---

## 1. 역사적 배경

### BFT 연구는 어떻게 시작됐을까?

```
1982  Lamport, Shostak, Pease - "비잔틴 장군 문제" 정의
        └─ 모든 것의 시작. "악의적인 노드가 있어도 합의할 수 있을까?"

1985  FLP 불가능성 정리
        └─ "비동기 네트워크에서 합의는 불가능하다" (충격!)

1989  Paxos (Lamport)
        └─ Crash fault만 허용. 악의적 노드는 못 막음.

1999  PBFT (Castro & Liskov)
        └─ 드디어! 실용적인 비잔틴 장애 허용 알고리즘 등장

2014  Tendermint (Kwon)
        └─ PBFT + PoS. 블록체인에 적용.

2019  HotStuff (Facebook)
        └─ 메시지 복잡도를 O(n)으로 줄임
```

### PBFT가 왜 중요한가?

1999년 이전에도 BFT 알고리즘은 있었어요. 하지만 너무 느려서 실제로 쓸 수가 없었죠. Castro와 Liskov는 이전 알고리즘보다 **10배 이상 빠른** PBFT를 만들었고, 실제로 NFS(파일 시스템)에 적용해서 **겨우 3%의 오버헤드**만 발생한다는 걸 증명했어요.

---

## 2. 비잔틴 장군 문제

### 문제 상황

여러 장군이 적의 도시를 공격하려고 해요. 모든 장군이 동시에 공격해야 이기고, 일부만 공격하면 져요.

```
                    ┌─────────────┐
                    │   사령관     │
                    │  (Primary)  │
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
       ┌────────┐    ┌────────┐    ┌────────┐
       │ 장군 A │    │ 장군 B │    │ 장군 C │
       │ (정직) │    │ (정직) │    │ (배신) │
       └────────┘    └────────┘    └────────┘
            │              │              │
            ▼              ▼              ▼
        "공격!"        "공격!"       "후퇴?"
```

문제는 **배신자(비잔틴 노드)**가 있다는 거예요:

-   다른 장군들에게 서로 다른 메시지를 보낼 수 있음
-   메시지를 안 보낼 수도 있음
-   거짓말을 할 수 있음

### 비잔틴 장애란?

| 장애 유형           | 설명                                          | 심각도   |
| ------------------- | --------------------------------------------- | -------- |
| Crash Fault         | 노드가 그냥 멈춤                              | 낮음     |
| Omission Fault      | 메시지를 안 보내거나 안 받음                  | 중간     |
| **Byzantine Fault** | 뭐든 할 수 있음 (거짓말, 조작, 선택적 무응답) | **최고** |

Byzantine Fault가 가장 일반적인 모델인 이유:

-   소프트웨어 버그 (예측 불가능한 동작)
-   해커의 공격
-   하드웨어 오류

---

## 3. 왜 3f+1 노드가 필요한가?

이 부분이 PBFT의 핵심이에요. 천천히 따라와 보세요.

### 핵심 문제: "기다림의 딜레마"

4명의 노드가 있고, 1명이 악의적(f=1)이라고 해봐요.

**질문: 클라이언트가 요청을 보내면, 몇 개의 응답을 기다려야 할까요?**

### Step 1: 왜 모든 노드 응답을 기다릴 수 없나?

```
노드 A: 응답 ✓
노드 B: 응답 ✓
노드 C: 응답 ✓
노드 D: (악의적) 일부러 응답 안 함

문제: 4개 응답을 기다리면 시스템이 영원히 멈춤!
```

악의적 노드가 응답을 안 하면 끝이에요. 그래서 **일부 응답만으로 진행**해야 해요.

최대 f개가 응답 안 할 수 있으니까, `n - f = 4 - 1 = 3`개 응답만 기다리면 됩니다.

### Step 2: 3개 응답 중 누가 정직한지 어떻게 알아?

```
시나리오 A: 정직한 노드들만 응답
─────────────────────────────────
응답한 노드: A(정직), B(정직), C(정직)
응답 안 함:  D(악의적)

결과: 3개 모두 정직 → 안전! ✓


시나리오 B: 악의적 노드도 응답에 포함
─────────────────────────────────
응답한 노드: A(정직), B(정직), D(악의적)
응답 안 함:  C(정직) - 네트워크 지연

결과: 2개 정직, 1개 악의적
      → 정직(2) > 악의적(1) → 다수결로 안전! ✓
```

### Step 3: 수학으로 정리

```
전제: n = 3f + 1, 기다리는 응답 수 = n - f = 2f + 1

최악의 경우:
┌─────────────────────────────────────────────────────┐
│                                                     │
│  응답한 2f+1개 노드 중:                              │
│                                                     │
│  • 최대 f개가 악의적일 수 있음                       │
│  • 나머지는 반드시 정직: (2f+1) - f = f+1개          │
│                                                     │
│  비교:                                              │
│  • 정직한 응답: f+1개                               │
│  • 악의적 응답: 최대 f개                            │
│                                                     │
│  → f+1 > f 이므로 정직한 쪽이 항상 다수! ✓          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 구체적 예시

**f=1 (1개 장애 허용)**

```
n = 3(1) + 1 = 4 노드
기다리는 응답 = 2(1) + 1 = 3개

노드 구성: 정직 3명 + 악의적 1명

[정직A] [정직B] [정직C] [악의D]

3개 응답을 받았을 때 가능한 조합:

Case 1: A, B, C 응답 → 정직 3, 악의 0 → 안전 ✓
Case 2: A, B, D 응답 → 정직 2, 악의 1 → 다수결 안전 ✓
Case 3: A, C, D 응답 → 정직 2, 악의 1 → 다수결 안전 ✓
Case 4: B, C, D 응답 → 정직 2, 악의 1 → 다수결 안전 ✓

→ 어떤 경우든 정직한 노드가 다수!
```

### 왜 3f가 아니라 3f+1인가?

만약 n = 3f라면? (예: f=1, n=3)

```
[정직A] [정직B] [악의C]

기다리는 응답 = 3 - 1 = 2개

가능한 조합:
Case: A, C 응답 → 정직 1, 악의 1

문제: 1 vs 1 → 다수결 불가! ❌
```

그래서 **+1이 필요**해요.

### 요약 표

| f (장애) | n (노드) | 쿼럼 | 최소 정직 | 최대 악의 |  결과   |
| :------: | :------: | :--: | :-------: | :-------: | :-----: |
|    1     |    4     |  3   |     2     |     1     | 2 > 1 ✓ |
|    2     |    7     |  5   |     3     |     2     | 3 > 2 ✓ |
|    3     |    10    |  7   |     4     |     3     | 4 > 3 ✓ |

---

## 4. View란 무엇인가?

View는 처음에 헷갈리는 개념인데, 간단히 말하면 **"리더(Primary)의 임기"**예요.

### 기본 개념

```
View = "누가 리더인가"를 나타내는 번호

View 0 → 노드 0이 리더 (Primary)
View 1 → 노드 1이 리더
View 2 → 노드 2이 리더
View 3 → 노드 3이 리더
View 4 → 노드 0이 다시 리더 (순환)

공식: Primary = View % 노드수
```

### 비유: 국회 회기

```
View = 국회 회기
Primary = 의장

제1회기 (View 0): 의장 = 김의원
   → 김의원이 안건 순서를 정함
   → 다른 의원들은 그 순서대로 투표

[김의원이 일을 안 하거나 이상하게 행동]
   → "의장 불신임!" (View Change)

제2회기 (View 1): 의장 = 이의원
   → 이의원이 새로운 의장으로 안건 순서 정함
```

### View Change가 일어나는 경우

```
View 0: Primary = 노드 0

문제 상황:
❌ Primary가 응답을 안 함 (죽었거나 네트워크 문제)
❌ Primary가 이상한 메시지를 보냄 (악의적)
❌ Primary가 특정 요청만 무시함 (검열)

→ Backup 노드들: "야, Primary 이상한데?"
→ 타임아웃 후 VIEW-CHANGE 메시지 전송
→ 2f+1 노드가 동의하면 View 1로 이동

View 1: Primary = 노드 1 (새 리더!)
```

### 왜 View가 필요한가?

```
문제: 리더가 없으면?

모든 노드가 동시에 순서를 제안하면...

노드 A: "요청 1번이 먼저!"
노드 B: "아니, 요청 2번이 먼저!"
노드 C: "요청 3번이 먼저야!"

→ 혼란! 합의 불가능!


해결: 한 번에 한 명의 리더만!

View 0에서는 노드 0만 순서를 제안할 수 있음
다른 노드들은 제안을 검증하고 동의/거부만 함

→ 질서 있는 합의 가능! ✓
```

---

## 5. 시스템 모델

### 네트워크 가정

PBFT는 **비동기 네트워크**를 가정해요. 즉, 인터넷처럼 불안정한 환경에서도 동작해야 해요.

```
┌─────────────────────────────────────────────────────────┐
│                   PBFT 네트워크 모델                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   가정:                                                  │
│   • 메시지가 손실될 수 있음                               │
│   • 메시지가 지연될 수 있음                               │
│   • 메시지가 중복될 수 있음                               │
│   • 메시지 순서가 바뀔 수 있음                            │
│   • 노드가 악의적으로 행동할 수 있음                       │
│                                                         │
│   보장:                                                  │
│   • 암호학적 기법(서명, 해시)은 깨지지 않음                │
│   • 노드 실패는 독립적 (동시에 다 같이 해킹당하진 않음)     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### FLP 불가능성과 PBFT의 해결책

1985년 FLP 정리에 의하면, **비동기 시스템에서 합의는 불가능**해요. 그럼 PBFT는 어떻게 동작할까요?

```
Safety (안전성): 항상 보장
───────────────────────────
• 잘못된 결과가 절대 나오지 않음
• 네트워크가 완전히 끊겨도 유지
• "차라리 멈추더라도 틀린 답은 안 낸다"

Liveness (활성): 약간의 가정 필요
───────────────────────────
• 메시지가 "결국" 전달된다는 가정
• 네트워크가 영원히 끊기지는 않음
• "언젠가는 응답이 온다"

결론:
네트워크 파티션이 발생하면?
→ Safety 유지 (틀린 결과 없음)
→ Liveness 포기 (시스템이 멈출 수 있음)
```

### 암호학적 기법

| 기법            | 용도              | 속도          |
| --------------- | ----------------- | ------------- |
| **MAC**         | 일반 메시지 인증  | 빠름 (10.3μs) |
| **디지털 서명** | View Change 증거  | 느림 (43ms)   |
| **해시 함수**   | 메시지 다이제스트 | 매우 빠름     |

---

## 6. 3-Phase Protocol 완전 정복

### 전체 흐름 먼저 보기

```
Client                Primary              Backups (1,2,3)
   │                     │                       │
   │───REQUEST──────────►│                       │
   │                     │                       │
   │                     │──PRE-PREPARE─────────►│
   │                     │                       │
   │                     │◄─────PREPARE──────────│
   │                     │                       │
   │                     │◄─────COMMIT───────────│
   │                     │──────COMMIT──────────►│
   │                     │                       │
   │                     │       EXECUTE         │
   │                     │                       │
   │◄────────────────REPLY───────────────────────│
```

**중요: 합의(3-Phase)와 실행(Execute)은 분리되어 있어요!**

```
합의 과정 (PRE-PREPARE → PREPARE → COMMIT)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"이 요청을 몇 번째로 실행할지" 순서만 결정!

실행 (EXECUTE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
합의가 끝난 후, 순서대로 실제 연산 수행!
```

---

### Phase 0: REQUEST (클라이언트 → Primary)

```
REQUEST⟨o, t, c⟩σc
```

| 파라미터 | 이름      | 설명              | 예시                    |
| :------: | --------- | ----------------- | ----------------------- |
|   `o`    | Operation | 실행할 연산       | `"transfer(A, B, 100)"` |
|   `t`    | Timestamp | 요청 시간         | `1699900000`            |
|   `c`    | Client ID | 클라이언트 식별자 | `"client_42"`           |
|   `σc`   | Signature | 클라이언트 서명   | (암호화된 값)           |

**왜 이 파라미터들이 필요한가?**

-   `o`: 뭘 실행할지 알아야 하니까
-   `t`: 같은 클라이언트의 요청 순서 구분 + 중복 요청 방지
-   `c`: 응답을 누구에게 보낼지
-   `σc`: 진짜 그 클라이언트가 보낸 건지 검증

---

### Phase 1: PRE-PREPARE (Primary → 모든 Backup)

```
⟨⟨PRE-PREPARE, v, n, d⟩σp, m⟩
```

| 파라미터 | 이름            | 설명             | 예시             |
| :------: | --------------- | ---------------- | ---------------- |
|   `v`    | View            | 현재 View 번호   | `0`              |
|   `n`    | Sequence Number | 할당된 순서 번호 | `42`             |
|   `d`    | Digest          | 요청의 해시값    | `"a1b2c3..."`    |
|   `m`    | Message         | 원본 REQUEST     | `REQUEST⟨o,t,c⟩` |

**이 단계의 의미:**

> "나(Primary)는 이 요청을 42번째로 실행하자고 제안해!"

**메시지 전송:**

```
Primary(노드0) ──PRE-PREPARE──► 노드1, 노드2, 노드3
                               (모든 Backup에게)

메시지 수: N-1 = 3개
```

**왜 d(digest)와 m(원본)을 둘 다 보내나?**

```
PRE-PREPARE = ⟨헤더, 본문⟩

헤더: ⟨PRE-PREPARE, v, n, d⟩
     → 작은 크기
     → PREPARE/COMMIT에서는 d만 사용
     → View Change 증거로 사용

본문: m (원본 요청)
     → 실행에 필요
     → 크기가 클 수 있음
```

---

### Phase 2: PREPARE (Backup들 → 모든 노드)

```
PREPARE⟨v, n, d, i⟩σi
```

| 파라미터 | 이름            | 설명               |
| :------: | --------------- | ------------------ |
|   `v`    | View            | 현재 View          |
|   `n`    | Sequence Number | PRE-PREPARE와 동일 |
|   `d`    | Digest          | PRE-PREPARE와 동일 |
|   `i`    | Replica ID      | 보내는 노드 ID     |

**이 단계의 의미:**

> "나(노드 i)는 View v에서 시퀀스 번호 n에 이 요청이 배정되는 것에 동의해!"

**⚠️ 중요: Primary는 PREPARE를 보내지 않아요!**

왜? PRE-PREPARE가 이미 "동의"의 의미를 담고 있으니까요.

```
PRE-PREPARE = "이 순서 제안해" + "나(Primary)는 동의해"
PREPARE     = "나도 이 순서에 동의해"

→ Primary가 PREPARE까지 보내면 중복!
```

**메시지 전송:**

```
노드0 (Primary): PREPARE 안 보냄! ❌ (PRE-PREPARE로 이미 함)

노드1 (Backup): → 노드0, 노드2, 노드3에게 PREPARE (3개)
노드2 (Backup): → 노드0, 노드1, 노드3에게 PREPARE (3개)
노드3 (Backup): → 노드0, 노드1, 노드2에게 PREPARE (3개)

메시지 수: (N-1) × (N-1) = 3 × 3 = 9개
           ─────   ─────
             │       └── 자기 제외한 모든 노드에게
             └── Backup만 보냄 (Primary 제외)
```

**Prepared 조건:**

```
prepared(m, v, n, i) = TRUE 가 되려면:

1. 원본 요청 m이 있어야 함
2. PRE-PREPARE⟨v, n, D(m)⟩이 있어야 함 (Primary에서)
3. 2f개의 PREPARE⟨v, n, D(m), j⟩가 있어야 함 (다른 Backup들에서)

예시 (f=1, n=4):
• 1개 PRE-PREPARE (Primary)
• 2개 PREPARE (다른 Backup 2개)
→ Total: 3개 노드가 같은 (v, n, d)에 동의 ✓
```

---

### Phase 3: COMMIT (모든 노드 → 모든 노드)

```
COMMIT⟨v, n, d, i⟩σi
```

파라미터는 PREPARE와 동일해요.

**이 단계의 의미:**

> "나(노드 i)는 prepared 상태가 됐어! 이 순서 확정해도 좋아!"

**⚠️ 중요: Primary도 COMMIT을 보내요!**

왜? PREPARE와 다르게, "prepared가 됐다"는 새로운 정보니까요.

```
PREPARE의 질문: "이 순서 동의해?"
→ Primary는 PRE-PREPARE로 이미 답함

COMMIT의 질문: "너 prepared 됐어?"
→ 이건 새로운 질문! Primary도 답해야 함
```

**메시지 전송:**

```
노드0 (Primary): → 노드1, 노드2, 노드3에게 COMMIT (3개) ✓
노드1 (Backup):  → 노드0, 노드2, 노드3에게 COMMIT (3개)
노드2 (Backup):  → 노드0, 노드1, 노드3에게 COMMIT (3개)
노드3 (Backup):  → 노드0, 노드1, 노드2에게 COMMIT (3개)

메시지 수: N × (N-1) = 4 × 3 = 12개
           ─   ─────
           │     └── 자기 제외한 모든 노드에게
           └── 모든 노드가 보냄 (Primary 포함!)
```

**Committed-local 조건:**

```
committed-local(m, v, n, i) = TRUE 가 되려면:

1. prepared(m, v, n, i) = TRUE
2. 2f+1개의 COMMIT⟨v, n, D(m), j⟩ 수신

예시 (f=1, n=4):
• prepared = TRUE (이미 완료)
• 3개 COMMIT 수신
→ 실행 가능! 🎉
```

---

### Phase 4: EXECUTE & REPLY

committed-local이 되면 드디어 실행!

```
실행 조건:
1. committed-local = TRUE
2. 시퀀스 번호가 n보다 작은 모든 요청이 이미 실행됨

→ 순차 실행 보장!
```

**순차 실행 예시:**

```
상황: 노드가 다음 순서로 committed-local 됨

시간 순서:
┌─────┐   ┌─────┐   ┌─────┐
│ n=42│   │ n=44│   │ n=43│
│ 먼저│   │ 두번째│   │ 세번째│
│도착 │   │ 도착 │   │ 도착 │
└─────┘   └─────┘   └─────┘

실행 순서:
1. n=42 실행 ✓ (바로 가능)
2. n=44 대기... (n=43이 아직!)
3. n=43 도착 → n=43 실행 ✓
4. n=44 실행 ✓ (이제 가능!)

→ 항상 42 → 43 → 44 순서로 실행됨!
```

**REPLY 메시지:**

```
REPLY⟨v, t, c, i, r⟩σi

v: View
t: 원본 REQUEST의 타임스탬프 (매칭용)
c: Client ID
i: 응답 보낸 노드 ID
r: 실행 결과
```

클라이언트는 **f+1개의 동일한 응답**을 기다려요. 왜? f+1개 중 최소 1개는 정직한 노드니까!

---

### 메시지 복잡도 정리

|    단계     |  보내는 노드  | 받는 노드 | 메시지 수 |  n=4   |
| :---------: | :-----------: | :-------: | :-------: | :----: |
| PRE-PREPARE |  1 (Primary)  |    N-1    |    N-1    |   3    |
|   PREPARE   | N-1 (Backup)  |    N-1    |  (N-1)²   |   9    |
|   COMMIT    | N (모든 노드) |    N-1    |  N(N-1)   |   12   |
|  **총합**   |               |           | **O(N²)** | **24** |

---

## 7. 왜 투표를 두 번 하나? (PREPARE vs COMMIT)

이게 PBFT에서 가장 헷갈리는 부분이에요. 스토리로 설명할게요.

### 등장인물

```
4명의 노드 (f=1, 1개 장애 허용)

👑 노드 0 (Primary in View 0): 앨리스
👤 노드 1 (Backup): 밥
👤 노드 2 (Backup): 찰리
😈 노드 3 (악의적): 데이브
```

### 시나리오: PREPARE만 있다면?

**Act 1: View 0에서 정상 진행**

```
앨리스(Primary): "이 요청을 42번으로 하자!"
                 PRE-PREPARE 전송

밥, 찰리: PREPARE 전송
데이브: 😈 (무시)
```

**Act 2: 일부만 prepared 상태**

```
앨리스: prepared = TRUE ✓ (PREPARE 2개 받음)
밥:     prepared = TRUE ✓ (PREPARE 2개 받음)
찰리:   prepared = FALSE (네트워크 지연으로 PREPARE 1개만 받음)
데이브: 😈
```

**Act 3: 갑자기 앨리스가 네트워크에서 사라짐!**

```
밥, 찰리, 데이브: "앨리스 응답 없어! View Change 하자!"

View 1로 이동, 새 Primary = 밥
```

**Act 4: 문제 발생! 🚨**

```
밥이 받은 VIEW-CHANGE 정보:

자신(밥):  prepared = [(v=0, n=42, 이체요청)]
찰리:      prepared = [] (빈 목록, 아직 prepared 아니었음)
데이브:    prepared = [] (거짓말)

밥의 고민:
"2f+1 = 3개 VIEW-CHANGE 받았는데...
 42번에 대해 prepared인 건 나 혼자뿐이네?

 이게 진짜 합의된 건가?
 아니면 내가 혼자 착각한 건가?"
```

**최악의 경우:**

```
만약 View 0에서 앨리스가 42번을 실행했다면?

View 0: 앨리스가 42번 = "이체 요청" 실행함
View 1: 밥이 "증거 부족하네" → 42번에 다른 요청 배정

결과:
앨리스: 이체 완료 (A: 900원)
밥/찰리: 다른 작업 (A: 1000원)

💥 상태 불일치! Safety 위반! 💥
```

### COMMIT이 있으면 해결!

**핵심 차이:**

```
PREPARE (2f개 필요):
─────────────────────
"같은 View 내에서" 순서 동의
→ View가 바뀌면? 정보가 사라질 수 있음!


COMMIT (2f+1개 필요):
─────────────────────
"View가 바뀌어도" 순서 보장
→ 2f+1개가 COMMIT을 보냈으면
→ 최소 f+1개 정직한 노드가 이 정보를 가짐
→ View Change 시 반드시 전달됨!
```

**수학적 보장 (비둘기집 원리):**

```
COMMIT 보낸 정직 노드: 최소 f+1개
VIEW-CHANGE 보낸 정직 노드: 최소 f+1개
전체 정직 노드: 2f+1개

(f+1) + (f+1) = 2f+2 > 2f+1

→ 두 집합이 겹침! 최소 1개는 둘 다 해당!
→ committed 정보는 View Change 시 반드시 전달됨! ✅
```

### 비유로 이해하기

```
PREPARE = 회의에서 손 드는 것
         → 회의록에 안 남음
         → 담당자 바뀌면 "누가 찬성했더라?"

COMMIT = 공식 서명
        → 문서로 남음
        → 담당자가 바뀌어도 서명 문서는 유효!
```

### 정리

```
1차 투표 (PREPARE): "이 순서에 동의해?"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• 같은 View 내에서 충돌 방지
• View가 바뀌면 보장 안 됨

2차 투표 (COMMIT): "prepared 됐어!"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• View가 바뀌어도 정보 보존
• View Change 때 반드시 전달됨
```

---

## 8. View Change 프로토콜

Primary가 문제가 생기면 어떻게 할까요?

### 트리거 조건

```
타이머 만료:
─────────────
Backup이 요청을 받고 일정 시간 내에 실행이 안 되면
→ "Primary가 문제인 것 같아"
→ VIEW-CHANGE 메시지 브로드캐스트
```

### VIEW-CHANGE 메시지

```
VIEW-CHANGE⟨v+1, n, C, P, i⟩σi

v+1: 새로운 View 번호
n:   마지막 stable checkpoint의 시퀀스 번호
C:   checkpoint 증명 (2f+1 서명)
P:   prepared 메시지들의 집합
i:   보내는 노드 ID
```

### NEW-VIEW 메시지

새 Primary가 2f+1개의 VIEW-CHANGE를 받으면:

```
NEW-VIEW⟨v+1, V, O⟩σp

v+1: 새 View 번호
V:   받은 VIEW-CHANGE 메시지들
O:   새로 생성할 PRE-PREPARE 메시지들
```

### View Change 과정

```
1. Backup들이 VIEW-CHANGE 전송
   └─ "나는 View v+1로 가고 싶어, 내가 알고 있는 정보는 이거야"

2. 새 Primary가 2f+1개 VIEW-CHANGE 수집
   └─ 충분한 동의 확보

3. 새 Primary가 NEW-VIEW 생성
   └─ 이전 View에서 prepared 됐던 요청들 복구
   └─ 빈 slot은 null 요청으로 채움

4. NEW-VIEW 브로드캐스트
   └─ 모든 노드가 새 View로 이동
   └─ 정상 프로토콜 재개
```

---

## 9. Garbage Collection & Checkpoints

메시지 로그가 무한히 커지면 안 되겠죠?

### Checkpoint 프로토콜

```
매 K개 요청마다 (예: K=100):

1. 노드가 상태 다이제스트 계산
2. CHECKPOINT⟨n, d, i⟩ 브로드캐스트
3. 2f+1개의 같은 다이제스트 수신 → stable checkpoint
4. stable checkpoint 이전 메시지 삭제 가능!
```

### Water Marks

```
Low Water Mark (h): 마지막 stable checkpoint
High Water Mark (H): h + L (L은 윈도우 크기)

유효한 시퀀스 번호: h < n ≤ H

예시:
h = 100, L = 200
→ 101~300번 요청만 처리 가능
→ 100 이하: 이미 체크포인트됨
→ 301 이상: 아직 처리하면 안 됨
```

---

## 10. Safety와 Liveness

### Safety (안전성)

> "잘못된 결과가 절대 나오지 않는다"

```
보장하는 것:
• 모든 정직한 노드가 같은 순서로 요청을 실행
• committed된 요청은 View가 바뀌어도 유지
• 충돌하는 두 요청이 같은 시퀀스 번호를 가질 수 없음

증명 핵심:
• 2f+1 쿼럼끼리는 최소 1개 정직한 노드에서 겹침
• 정직한 노드는 같은 (v, n)에 다른 요청에 동의 안 함
```

### Liveness (활성)

> "요청이 결국 처리된다"

```
보장하는 것:
• 클라이언트의 요청이 무한히 지연되지 않음
• View Change가 결국 완료됨

필요한 가정:
• 네트워크가 영원히 끊기지 않음
• 메시지가 "결국" 전달됨
• Primary가 악의적이면 View Change 발생
```

---

## 11. 성능 최적화

### MAC vs 디지털 서명

```
디지털 서명: 43ms (4000배 느림!)
MAC:        10.3μs

→ 일반 작업에는 MAC 사용
→ View Change에만 디지털 서명 (부인 방지 필요)
```

### Read-only 최적화

읽기 전용 요청은 상태를 바꾸지 않으니까:

```
일반 요청: 5 message delays (PRE-PREPARE → PREPARE → COMMIT → REPLY)
Read-only: 1 message delay (바로 REPLY!)

조건: f+1개의 같은 응답 필요 (상태 변경 없으니 순서 중요 X)
```

### Tentative Execution

```
일반: COMMIT 완료 후 실행 (5 delays)
최적화: PREPARE 완료 후 임시 실행 (4 delays)

주의: 임시 결과는 롤백될 수 있음
      committed 후에 최종 확정
```

---

## 12. Tendermint와 비교

PBFT를 이해했으면 Tendermint는 쉬워요!

| 구분          | PBFT               | Tendermint                    |
| ------------- | ------------------ | ----------------------------- |
| **연도**      | 1999               | 2014                          |
| **대상**      | 일반 분산 시스템   | 블록체인                      |
| **단위**      | 개별 요청          | 블록                          |
| **멤버십**    | 정적 (고정된 노드) | 동적 (검증자 변경 가능)       |
| **인센티브**  | 없음               | PoS (스테이킹)                |
| **프로토콜**  | 3-Phase            | Propose → Prevote → Precommit |
| **장애 허용** | f < n/3            | f < n/3                       |

### Tendermint의 혁신

```
1. 블록 기반 합의
   └─ 개별 트랜잭션이 아닌 블록 단위로 합의
   └─ 효율성 향상

2. PoS 통합
   └─ 검증자가 토큰을 스테이킹
   └─ 악의적 행동 시 슬래싱

3. ABCI (Application Blockchain Interface)
   └─ 합의 엔진과 애플리케이션 분리
   └─ 어떤 언어로든 앱 개발 가능

4. 동적 검증자
   └─ 검증자가 들어오고 나갈 수 있음
   └─ PBFT의 정적 멤버십 한계 극복
```

---

## 13. 구현 예시

### 메시지 구조체 (Go)

```go
package pbft

import (
    "crypto"
    "time"
)

// 클라이언트 요청
type RequestMsg struct {
    Operation []byte    // 실행할 연산
    Timestamp time.Time // 요청 시간
    ClientID  string    // 클라이언트 식별자
    Signature []byte    // 클라이언트 서명
}

// Pre-prepare 메시지 (Primary만 전송)
type PrePrepareMsg struct {
    View      uint64     // 현재 View
    SeqNum    uint64     // 시퀀스 번호
    Digest    []byte     // 요청의 해시
    Request   *RequestMsg // 원본 요청
    ReplicaID uint64     // Primary ID
    Signature []byte
}

// Prepare 메시지 (Backup들만 전송)
type PrepareMsg struct {
    View      uint64
    SeqNum    uint64
    Digest    []byte
    ReplicaID uint64 // 보내는 Backup ID
    Signature []byte
}

// Commit 메시지 (모든 노드가 전송)
type CommitMsg struct {
    View      uint64
    SeqNum    uint64
    Digest    []byte
    ReplicaID uint64 // 보내는 노드 ID
    Signature []byte
}

// Reply 메시지 (클라이언트에게)
type ReplyMsg struct {
    View      uint64
    Timestamp time.Time // REQUEST의 타임스탬프 (매칭용)
    ClientID  string
    ReplicaID uint64
    Result    []byte
    Signature []byte
}
```

### 노드 상태

```go
type ReplicaState struct {
    // 기본 정보
    ID          uint64
    View        uint64
    SeqNum      uint64
    F           int // 허용 장애 수

    // 로그
    MessageLog  map[uint64]*LogEntry
    PrepareLog  map[string]int // key: "view-seq-digest"
    CommitLog   map[string]int

    // 실행 상태
    LastExecuted uint64

    // Checkpoint
    LowWaterMark  uint64
    HighWaterMark uint64
}

func (s *ReplicaState) IsPrimary() bool {
    return s.ID == s.View % uint64(3*s.F + 1)
}

func (s *ReplicaState) IsPrepared(view, seq uint64, digest []byte) bool {
    key := makeKey(view, seq, digest)

    // PRE-PREPARE 있는지 확인
    entry := s.MessageLog[seq]
    if entry == nil || entry.PrePrepare == nil {
        return false
    }

    // 2f개의 PREPARE 있는지 확인
    return s.PrepareLog[key] >= 2*s.F
}
```

### 3-Phase Protocol 구현

```go
// Primary가 요청 처리
func (n *Node) HandleRequest(req *RequestMsg) error {
    if !n.state.IsPrimary() {
        return n.forwardToPrimary(req)
    }

    // 시퀀스 번호 할당
    seq := n.state.nextSeqNum()
    digest := computeDigest(req)

    // PRE-PREPARE 생성 및 전송
    prePrepare := &PrePrepareMsg{
        View:      n.state.View,
        SeqNum:    seq,
        Digest:    digest,
        Request:   req,
        ReplicaID: n.state.ID,
    }

    // N-1개 Backup에게 전송
    return n.broadcastToBackups(prePrepare)
}

// Backup이 PRE-PREPARE 처리
func (n *Node) HandlePrePrepare(msg *PrePrepareMsg) error {
    // 검증: View, 시퀀스 번호, 다이제스트 등
    if !n.validatePrePrepare(msg) {
        return errors.New("invalid pre-prepare")
    }

    // PREPARE 생성 (Backup만!)
    prepare := &PrepareMsg{
        View:      msg.View,
        SeqNum:    msg.SeqNum,
        Digest:    msg.Digest,
        ReplicaID: n.state.ID,
    }

    // 자신 제외 모든 노드에게 전송 (N-1개)
    return n.broadcastToOthers(prepare)
}

// PREPARE 처리
func (n *Node) HandlePrepare(msg *PrepareMsg) error {
    if !n.validatePrepare(msg) {
        return errors.New("invalid prepare")
    }

    // 카운트 증가
    key := makeKey(msg.View, msg.SeqNum, msg.Digest)
    n.state.PrepareLog[key]++

    // prepared 조건 확인 (2f개)
    if n.state.IsPrepared(msg.View, msg.SeqNum, msg.Digest) {
        // COMMIT 전송 (모든 노드, Primary 포함)
        commit := &CommitMsg{
            View:      msg.View,
            SeqNum:    msg.SeqNum,
            Digest:    msg.Digest,
            ReplicaID: n.state.ID,
        }
        return n.broadcastToOthers(commit)
    }

    return nil
}

// COMMIT 처리
func (n *Node) HandleCommit(msg *CommitMsg) error {
    if !n.validateCommit(msg) {
        return errors.New("invalid commit")
    }

    key := makeKey(msg.View, msg.SeqNum, msg.Digest)
    n.state.CommitLog[key]++

    // committed-local 조건 확인 (2f+1개)
    if n.state.CommitLog[key] >= 2*n.state.F + 1 {
        return n.tryExecute(msg.SeqNum)
    }

    return nil
}

// 순차 실행
func (n *Node) tryExecute(upTo uint64) error {
    for seq := n.state.LastExecuted + 1; seq <= upTo; seq++ {
        entry := n.state.MessageLog[seq]
        if entry == nil || !entry.Committed {
            break // 순서대로 실행해야 함!
        }

        // 실제 연산 실행
        result := n.executor.Execute(entry.Request.Operation)
        entry.Executed = true
        n.state.LastExecuted = seq

        // 클라이언트에게 응답
        reply := &ReplyMsg{
            View:      n.state.View,
            Timestamp: entry.Request.Timestamp,
            ClientID:  entry.Request.ClientID,
            ReplicaID: n.state.ID,
            Result:    result,
        }
        n.sendToClient(entry.Request.ClientID, reply)
    }
    return nil
}
```

---

## 용어 정리

| 용어                | 정의                                            |
| ------------------- | ----------------------------------------------- |
| **Byzantine Fault** | 노드가 뭐든 할 수 있는 장애 (거짓말, 무응답 등) |
| **Replica**         | 상태 머신의 복제본 (= 노드)                     |
| **Primary**         | 현재 View의 리더, 순서 제안자                   |
| **Backup**          | Primary가 아닌 노드들                           |
| **View**            | 리더의 "임기", Primary를 결정하는 번호          |
| **Quorum**          | 합의에 필요한 최소 노드 수 (2f+1)               |
| **prepared**        | 현재 View에서 순서 동의됨                       |
| **committed**       | View가 바뀌어도 순서 보장됨                     |
| **Safety**          | 틀린 결과가 절대 안 나옴                        |
| **Liveness**        | 요청이 결국 처리됨                              |

---

## 핵심 공식 정리

```
n ≥ 3f + 1          전체 노드 수
Quorum = 2f + 1     합의 필요 노드 수
Primary = View % n  현재 리더

PREPARE 조건:   PRE-PREPARE 1개 + PREPARE 2f개
COMMIT 조건:    prepared + COMMIT 2f+1개
클라이언트:     f+1개 동일 응답 대기
```

---
