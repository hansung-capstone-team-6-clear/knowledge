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
| `update_due` | `item_id`, `due_at` | 날짜만 바꾸면 원래 시각 유지 | id가 목록에 있는지, 마감이 있고 미래인지 (아니면 재질문) |
| `delete_task` | `item_id`, `confirm`(확인 질문에 답할 때만) | 지우기·취소 요청일 때만 낸다("완료했어"는 complete_task). 대상이 모호하면 clarify. 확인 답은 `yes`·`no`·`null` | id가 목록에 있는지, 확인 답은 `confirm`이 `yes`이고 `item_id`가 확인 대상과 같을 때만 삭제 |
| `set_reminder` | `item_id`, `offset_minutes` | "2시간 전" → 120 | id가 목록에 있는지, offset > 0 (PR #40) |
| `clarify` | `candidates` + **지금까지 확정된 `name`·`due_at`·`offset_minutes`** | 질문 문장은 짧은 한국어 | `question`을 TTS로, Decision 전체를 pending으로 60초 저장 |

**삭제 확인 답**은 AI가 판단한다 (desk-pet-backend PR #39·#42). `pending.action`이 `delete_task`이면 사용자가 확인 질문에 답하는 중이다.

| 답 | AI 출력 |
| --- | --- |
| 동의 ("응", "알겠어요", "지워줘") | `delete_task`, `item_id`는 pending과 같게, `confirm: "yes"` |
| 거절 ("아니", "그냥 둬") | `delete_task`, `item_id`는 pending과 같게, `confirm: "no"` |
| 알 수 없음 ("잠깐만", "음") | `delete_task`, `item_id`는 pending과 같게, `confirm: null` (확실한 거절이 아니면 `no`를 내지 않는다) |
| 새 요청 ("과제 제출 완료했어") | 새 action, `confirm` 없음 |

처음 삭제를 요청할 때는 `confirm`을 넣지 않는다. `delete_item_id`는 `analyze()`에 넘기기 전에 빼고, Backend가 AI의 `item_id`와 비교한다.
11/18 평가셋(39문항, 삭제 확인 답 5문항 포함)은 39/39를 통과했다.

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
| Gemini 응답이 깨진 JSON이거나 Decision과 다름 | 재시도, 다시 실패하면 `pydantic.ValidationError` | `tasks.handle()`이 잡아 `status: ERROR` |
| Gemini 503(혼잡) | 재시도, 다시 실패하면 `google.genai` 예외 | 같음 |
| 그 밖의 Gemini 오류·네트워크 오류 | `google.genai` 등 예외 (재시도 없음) | 같음 |
| 7초 초과 | (호출 중단) | `asyncio.timeout(7)` → `ERROR` |

`analyze()`는 503과 깨진 JSON에서만 재시도하며, **둘을 합쳐 1회**다 (503으로 재시도한 뒤 JSON이 깨져도 더 하지 않는다).
그 외 예외와 재시도 실패는 그대로 올린다. 재시도 시간도 `handle()`의 전체 제한 시간에 포함되며, 재시도가 일어나면 경고 로그를 남긴다.

깨진 JSON(`key must be a string` 등)은 응답 스키마(`response_json_schema`)로도 막아 보았으나 쓰지 않는다. 11/12 시험에서 스키마를 지정하면
JSON 오류는 사라졌지만 말한 마감(`due_at`)을 빠뜨리거나 `args`를 비우는 경우가 늘었다 (§ 7).

**2026-11-06 결정 (구현 전):** Gemini 처리 시간이 1~30초로 불규칙해 정상 호출도 `TimeoutError`로 끝나므로 시간 제한을 늘린다. Pi는 10초에 지연 안내, 20초에 실패 처리한다. LLM 재시도는 1회이고 대상 오류는 § 8에서 정한다. 재시도 시간은 계속 Backend 처리 제한에 포함한다 (`../contracts.md` § 1).

## 6. 시간 예산

| 구간 | 측정값 |
| --- | --- |
| `analyze()` 텍스트 | 중앙값 1.3~1.5초 (11/03), 1.7초 (11/06 서버) |
| `analyze()` 음성 | 2~3.5초 (11/03), 중앙값 3.2초 (11/06 서버, 6건) |
| 혼잡할 때 | 1~30초까지 불규칙 |
| `/api/voice` 전체 (EC2 시드니) | 조회 4.3~8초, 등록 약 7.2초 추정 (11/06) |
| 현재 제한 | 처리 7초 (Pi는 10초 대기) |
| 결정된 제한 (구현 전) | Pi 10초 지연 안내, 20초 실패 |

근거: [[projects/eureka-desk-pet/ai/eval-2026-11-03|11/03 AI 평가]], 2026-11-06 실제 서버 시험.

## 7. 결정된 항목

1. **새 업무 이름 검증**: AI가 가끔 업무 목록의 이름을 빌려 와 `create_task`를 낸다 (평가 § 4-1). `tasks.py`에서 검증한다 ([PR #15](https://github.com/hansung-capstone-team-6-clear/desk-pet-backend/pull/15) 리뷰 중). `pending`에 이름 검증용 `utterances`가 추가되지만 `analyze()`에 넘기기 전에 빼므로 입력은 그대로다.
2. **503 재시도**: 1회 재시도하고, 다시 실패하면 `ERROR`로 응답한다 (§ 5).
3. **맥락의 `open_tasks`**: 캐시 구현 후 Eureka 직접 조회 대신 캐시에서 온다. `analyze()` 입력 형식은 그대로다.
4. **깨진 JSON 재시도**: 이서현 요청(11/07)에 따라 503과 함께 재시도 대상에 넣는다. 재시도는 둘을 합쳐 1회 (§ 5).
5. **응답 스키마 미사용**: 11/12 시험에서 스키마 3가지(항목 null 허용, 항목 생략 가능, 최상위만 지정)를 비교했다.
   모두 `update_due`의 `due_at`이 빠지거나 `args`가 비었고, 스키마 없이 쓴 쪽만 값이 채워졌다. 대신 깨진 JSON을 재시도한다.

## 8. 미정 — 이서현과 정할 것

1. **`candidates` 사용**: AI는 후보 `item_id`를 주지만 Backend는 쓰지 않고 `question`만 읽는다. LCD에 후보 수를 보여 주는 등 쓸지.
2. `set_reminder` 처리 흐름 연결 여부 (AI 판단은 평가셋 문항 통과).
3. **호출 시간 초과 재시도와 호출 1번의 제한 시간**: 503·깨진 JSON은 정했다 (§ 7). 시간 초과도 재시도할지, 호출 1번에 제한 시간을 둘지.
