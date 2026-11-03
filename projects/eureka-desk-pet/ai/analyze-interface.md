---
type: interface
created: "2026-11-03"
author: 박시현
reviewers: [이서현]
sources:
  - "[[projects/eureka-desk-pet/meetings/2026-10-29-mentor-meeting-note|10-29 회의록]] A7 — 이서현–박시현 사이 함수 입출력 정리"
  - "[[projects/eureka-desk-pet/contracts|레이어 간 계약]] § 3 — AI 출력"
  - "desk-pet-backend src/desk_pet_backend/services/ai.py, services/tasks.py, db.py, schemas.py (main 9e5b645 + feat/clarify-prompt)"
---

# AI ↔ Backend 함수 입출력 (`analyze()`)

`/api/voice` 요청 하나는 아래 순서로 처리된다. **박시현(AI)** 담당은 ②이고, 나머지는 **이서현(Backend)** 담당이다.

```text
① tasks.handle()      입력 준비 — Eureka 미완료 업무, db pending 조회          (이서현)
② ai.analyze()        Gemini 1회 호출 → Decision                              (박시현)
③ tasks._handle()     정책 검증 → Eureka 실행 → tts·lcd 템플릿 → VoiceResponse  (이서현)
```

이 문서는 ①→②(입력)와 ②→③(출력) 사이의 약속이다. 형식 자체의 정본은 `contracts.md` § 3이고, 여기에는 두 함수가 서로 기대하는
동작을 적는다.

## 1. 함수

```python
async def analyze(*, audio: bytes | None, text: str | None, now: datetime,
                  open_tasks: list[Task], pending: dict | None) -> Decision
```

## 2. 입력 (Backend → AI)

| 인자 | 형식 | 만드는 곳 | 약속 |
| --- | --- | --- | --- |
| `audio` | WAV bytes (16kHz·모노·16bit, 15초·1MB 이하) | `api/voice.py` 검증 후 | `text`와 둘 중 하나는 반드시 있다. 둘 다 없으면 `ValueError` |
| `text` | str | `/api/text` | 개발·평가·시연용 |
| `now` | `datetime` (+09:00) | `tasks._handle()` | "내일", "다음 주 화요일" 계산 기준. 요일은 AI가 붙인다 |
| `open_tasks` | `list[Task]` (`item_id`, `name`, `due_at`, `status`) | `eureka.list_open_tasks()` | 완료(`done`) 제외. AI는 앞 30개만 프롬프트에 넣는다 |
| `pending` | dict 또는 `None` | `db.get_pending(user_id)` | **직전 재질문 때 받은 `Decision.model_dump(mode="json")` 그대로.** 60초 지나면 `None` |

## 3. 출력 (AI → Backend): `schemas.Decision`

```json
{"transcript": "보고서 작성 추가해줘", "action": "clarify",
 "args": {"name": "보고서 작성"}, "question": "보고서 작성의 마감은 언제로 할까요?"}
```

| 필드 | 약속 |
| --- | --- |
| `transcript` | 이번 발화를 받아 적은 문장 → 응답 `stt_text`. pending 문장을 섞지 않는다 |
| `action` | `create_task` · `list_tasks` · `complete_task` · `update_due` · `set_reminder` · `clarify` |
| `args` | 아래 표. 안 쓰는 키는 생략 (`None`) |
| `question` | `clarify`일 때만 문자열 → Backend가 그대로 TTS로 읽는다. 나머지는 `null` |

| `action` | AI가 채우는 `args` | AI가 보장하려는 것 (프롬프트) | Backend가 다시 확인하는 것 (코드) |
| --- | --- | --- | --- |
| `create_task` | `name`, `due_at`, `offset_minutes` | name·due_at이 모두 있을 때만 낸다. 날짜를 지어내지 않는다 | name·due_at 존재, 마감이 미래, offset > 0, 없으면 60 |
| `list_tasks` | `range` | today·tomorrow·week·all 중 하나 | 범위로 거르기, 최대 5개 읽기 |
| `complete_task` | `item_id` | open_tasks에 있는 id만 | id가 목록에 있는지 (없으면 재질문) |
| `update_due` | `item_id`, `due_at` | 날짜만 바꾸면 원래 시각 유지 | **처리 흐름 미구현** ("아직 지원하지 않아요") |
| `set_reminder` | `item_id`, `offset_minutes` | "2시간 전" → 120 | **처리 흐름 미구현** |
| `clarify` | `candidates` + **지금까지 확정된 `name`·`due_at`·`offset_minutes`** | 질문 문장은 짧은 한국어 | `question`을 TTS로, Decision 전체를 pending으로 60초 저장 |

## 4. 재질문(pending) 왕복

```text
요청 1  "보고서 작성 추가해줘"
  ② Decision(action=clarify, args={name: "보고서 작성"}, question="마감은 언제로 할까요?")
  ③ db.set_pending(user_id, Decision 전체) → 응답 type=clarification, 표정 thinking
요청 2  "내일 오후 3시까지"          (60초 안)
  ① db.get_pending(user_id) → 위 Decision
  ② pending + 이번 발화 → Decision(action=create_task, name="보고서 작성", due_at=내일 15:00)
  ③ 실행 성공 → db.clear_pending(user_id)
```

- 대답이 질문과 맞지 않으면(업무를 물었는데 시간만 답함) AI는 실행하지 않고 다시 `clarify`를 낸다. 이때 pending은 새 Decision으로
  덮어써진다. **clarify `args`에 확정된 값을 계속 넣는 이유**가 이것이다 — 덮어써져도 원래 요청의 이름·마감이 남는다.
- 대답이 전혀 다른 새 요청이면 AI는 pending을 무시하고 새 요청으로 처리한다. 성공하면 Backend가 pending을 지운다.

## 5. 오류

| 상황 | `analyze()` 동작 | Backend 처리 |
| --- | --- | --- |
| Gemini 응답이 스키마와 다름 | `pydantic.ValidationError` | `tasks.handle()`이 잡아 `status: ERROR` |
| Gemini 503(혼잡)·네트워크 오류 | `google.genai` 예외 | 같음 |
| 9초 초과 | (호출 중단) | `asyncio.timeout(9)` → `ERROR` |

`analyze()`는 예외를 잡지 않고 그대로 올린다. 재시도는 하지 않는다 (아래 미정 3).

## 6. 시간 예산

| 구간 | 측정값 |
| --- | --- |
| `analyze()` 텍스트 | 중앙값 1.3~1.5초 |
| `analyze()` 음성 | 2~3.5초 |
| `/api/voice` 전체 (EC2 시드니) | 6~8초 |
| 제한 | 9초 (Pi는 10초 대기) |

근거: [[projects/eureka-desk-pet/ai/eval-2026-11-03|11/03 AI 평가]], [[projects/eureka-desk-pet/outputs/2026-11-01-backend-aws-deploy|11/01 배포]].

## 7. 미정 — 이서현과 정할 것

1. **새 업무 이름 검증**: AI가 가끔 업무 목록의 이름을 빌려 와 `create_task`를 낸다 (평가 § 4-1). `name`이 이번 발화·pending 발화에
   없으면 재질문하는 코드 검사를 `tasks.py` 정책에 둘지 `ai.py` 후처리에 둘지.
2. **`candidates` 사용**: AI는 후보 `item_id`를 주지만 Backend는 쓰지 않고 `question`만 읽는다. LCD에 후보 수를 보여 주는 등 쓸지.
3. **503 재시도**: `analyze()` 안에서 1회 재시도할지 (지연 +2초 내외), 그대로 `ERROR`로 둘지.
4. `update_due`·`set_reminder` 처리 흐름 연결 일정 (AI 판단은 평가셋 6문항 모두 통과).
