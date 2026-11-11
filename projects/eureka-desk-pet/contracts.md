# eureka-desk-pet 레이어 간 계약 (1차)

> 레이어 간에 주고받는 형식의 **단일 진실원**. 바꾸려면 이 파일 PR을 먼저 올리고, merge 후 각 코드 레포에 반영한다.
> 상태: **초안 (2026-10-08)**, 2026-10-29 멘토 미팅과 2026-11-06 팀 결정 반영. 구조와 흐름은 `architecture.md`를 본다.
> 2026-11-06 결정 중 `GET /api/tasks`와 업무 목록 캐시는 Backend에 구현해 2026-11-11 서버에 배포했다. 시간 제한과 Pi 알람은 **아직 구현 전**이다. 각 절에 현재 동작을 함께 적는다.
>
> 기반: [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]과 [[wiki/desk-pet-device-states|장치 상태]]
> (2026-10-07 하드웨어 결정, MVP Mock Server 형식 유지). 이 문서는 그 형식을 그대로 쓰고, Backend 내부(AI 출력·Eureka 매핑·DB)와
> [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]의 미결 항목에 대한 답을 더한 것이다.

## 0. 공통 규칙

1. JSON 필드 이름은 snake_case (Mock Server 형식 그대로).
2. 시각은 ISO-8601 + `+09:00` (예: `2026-10-09T15:00:00+09:00`). Eureka의 epoch ms 변환은 Backend `eureka.py`에서만 한다.
3. 알림 시점은 "마감 N분 전"을 뜻하는 정수 `offset_minutes` 하나로 표현한다. `reminderAt`(절대 시각), `reminder_offset_minutes`,
   `alert_before_minutes` 같은 기존 이름은 쓰지 않는다.
4. 기기는 `user_id`로 식별한다 (MVP는 사용자 1명당 펫 1대). 1차는 로그인 없이 Eureka 계정 하나를 쓴다.
   요청의 `user_id`는 선택값이고, Eureka API 키는 Backend `.env`에만 둔다 (§ 4).
5. 요청 추적용 `request_id`는 **Pi가 만들어 보내고 Backend가 응답에 그대로 돌려준다.** Pi가 생략하면 Backend가 만든다.
   Backend가 만드는 음성 요청 id는 `req_20261029_153000_a1b2` 형식이다.
6. **추가 — 인증**: Pi → Backend 요청에 헤더 `X-Device-Token`을 붙인다. 값은 양쪽 `.env`에만 두고 커밋하지 않는다.
7. 1차는 HTTP를 쓴다. HTTPS 전환은 2차다 (`architecture.md` § 1차에서 뺀 것).

## 1. Pi → Backend: 음성 요청

`POST /api/voice` — `multipart/form-data`

| 필드 | 형식 |
| --- | --- |
| `audio` | WAV, 16kHz, mono, 16bit, 최대 15초 |
| `user_id` | string, **선택**. 생략하면 Backend는 기본 사용자 하나로 처리한다 |
| `request_id` | string, **선택**. 보내면 응답에 그대로 돌려주고, 생략하면 Backend가 만든다 |

`POST /api/text` — `application/json`. **추가** — 개발·평가·비상 시연용. 응답은 `/api/voice`와 같다.

```json
{ "user_id": "pet-01", "text": "내일 오후 3시까지 보고서 작성 추가해줘" }
```

응답 (두 엔드포인트 공통, HTTP 200). 모양은 `desk-pet-backend` `schemas.VoiceResponse` 그대로다:

```json
{
  "request_id": "req_20261029_153000_a1b2",
  "status": "OK",
  "stt_text": "내일 오후 3시까지 보고서 작성 추가해줘",
  "type": "task_created",
  "output": {
    "tts": "보고서 작성 업무를 등록했어요. 마감은 내일 오후 3시예요.",
    "lcd": { "expression": "happy", "text": "업무 등록 완료" }
  }
}
```

| `status` | 의미 |
| --- | --- |
| `OK` | 처리됨 — `type` 참고 |
| `NO_SPEECH` | 말소리가 없음. `output.tts`는 다시 말해 달라는 안내 |
| `STT_ERROR` | 음성 인식 실패 |
| `ERROR` | 처리 실패 (Eureka 실패, 시간 초과 등). 성공 문구를 보내지 않는다 |

| `type` | 의미 |
| --- | --- |
| `task_created`, `task_completed`, `task_updated`, `task_deleted`, `task_list`, `reminder_set` | 처리 성공 |
| `task_delete_cancelled` | 삭제 확인에 "아니"로 답함. 재질문이 아니라 대기로 돌아간다 |
| `clarification` | 재질문. `output.tts`가 질문 문장 |
| `error` | `status`가 `OK`가 아닐 때 |

- **시간**. 2026-11-06에 아래 정책으로 바꾸기로 했다 (구현 전). 시간은 Pi의 업로드 시작부터 잰다.

  | 항목 | 현재 동작 | 결정 (구현 전) |
  | --- | --- | --- |
  | 지연 안내 | 없음 | 10초가 지나도 응답이 없으면 Pi가 "조금 더 기다려주세요" 안내. LCD와 TTS 중 무엇을 쓸지는 하드웨어 팀이 PR 리뷰에서 정한다 |
  | 요청 실패 | Pi 10초 초과 | Pi 20초 초과 |
  | Backend 처리 제한 | 업로드·검증 뒤 읽기·Gemini·쓰기 전체 7초 | Pi 20초 안에 응답이 도착하도록 조정. 제안값: 쓰기 전까지 15초 + Eureka 쓰기 3초 (Pi 업로드 실측 후 확정) |
  | LLM 재시도 | Gemini 503일 때 1회 | 1회. 대상 오류(시간 초과, 깨진 JSON 포함 여부)는 AI 담당과 정한다 |

  처리 제한을 넘으면 `ERROR`를 반환한다. Backend 제한을 Pi 변경보다 먼저 늘리면 Pi가 끊은 뒤에도 Backend가 업무를 실행할 수 있으므로 **두 변경은 함께 배포한다.**
- **현재 반환 범위**. `OK`와 `ERROR`를 반환한다. `NO_SPEECH`·`STT_ERROR`는 스키마에 정의되어 있고 반환 분기는 없다. `reminder_set`도 현재 처리에서 생성하지 않는다.
- **재질문 연결**: Pi는 새 필드를 보내지 않는다. Backend가 `user_id`별로(생략하면 기본 사용자로) 원래 요청을 60초 동안
  기억했다가(`pending`) 다음 요청에 이어 붙인다. `conversationId`는 필요 없다.
- **`output.lcd`**: LCD에 띄울 표정(`expression`)과 짧은 문구(`text`). `null`일 수 있다 — 그러면 Pi는 상태별 기본 표정을 쓴다 (§ 5).
  10-29 회의 슬라이드 초안의 `output.expression`·`output.text`(평평한 모양)는 쓰지 않는다.
- **`output.lcd.expression`**: `idle`, `listening`, `thinking`, `speaking`, `alert`, `surprised`, `happy`. 재질문 때는 `thinking`을 쓴다.
- Backend가 `led` 필드를 보내지 않는다. LED 색은 Pi가 정한다 (§ 5).
- **처리 결과는 `status` 문자열로 준다.** HTTP 오류 코드는 요청 자체가 잘못됐을 때만 쓰고, 이때 본문은 `{"detail": "<사유>"}`다.

| HTTP | 언제 |
| --- | --- |
| 401 | `X-Device-Token`이 없거나 틀림 |
| 413 | 파일이 1MB를 넘거나 음성이 15초를 넘음 |
| 422 | 16kHz·모노·16bit PCM WAV가 아니거나 비어 있음, 필수 필드 누락 |

## 2. Pi → Backend: 업무 목록 (마감 임박 알람)

> **2026-11-06 결정 (Backend 구현됨, Pi 구현 전):** 마감 임박 알림은 WebSocket 푸시 대신 **Pi가 스스로 시간을 확인해 울린다.** Backend는 업무 목록만 준다.
> 10-29의 "선제 알림 1차 제외"를 대체한다. 지금 코드에는 이전 설계의 `WS /ws/hardware/{user_id}`가 남아 있고 삭제 예정이다. Pi는 여기에 연결하지 않는다.

`GET /api/tasks` + 헤더 `X-Device-Token`. 응답은 아래와 같다 (Backend PR #36). 업무는 마감순이다.

```json
{
  "tasks": [
    {
      "item_id": "1000046",
      "name": "보고서 검토",
      "due_at": "2026-11-20T17:00:00+09:00",
      "offset_minutes": 60
    }
  ]
}
```

| 담당 | 하는 일 |
| --- | --- |
| Backend | 미완료 업무의 이름·마감 시각·알림 간격 전달. 간격을 따로 지정하지 않은 업무는 기본 60분 |
| Pi | 목록 저장과 갱신, `마감 − offset_minutes ≤ 지금 < 마감`이면 알림 재생·LCD 표시, 보낸 기록 저장, 재부팅 후 복구 |

- 마감은 시각까지 저장한다. Eureka `dueAt`은 epoch ms 값이다 (2026-11-06 원본 확인).
- 알림 문장은 Pi가 만든다. 예: "보고서 검토 마감이 1시간 남았어요."
- 중복 방지는 Pi의 보낸 기록이 맡는다. 키는 (`item_id`, `due_at`, `offset_minutes`)다. 마감을 바꾸면 키가 달라져 새 마감 기준으로 다시 알린다.
- Pi는 부팅 때와 등록·수정·완료 응답을 받은 뒤 목록을 갱신한다. 그 밖의 갱신 주기는 Hardware PR에서 정한다.
- Backend는 목록을 캐시에서 내보낸다 (§ 4). Eureka 웹에서 바꾼 내용은 최대 하루 늦게 반영될 수 있다 (팀 전체 리뷰 항목).
- 언제 울릴지는 Pi가 판단하므로 Backend는 마감이 지난 업무도 거르지 않는다. 알람 대상은 Pi가 위 조건으로 고른다.
- 목록을 못 받으면 Backend는 503 `{"detail": "업무 목록을 가져오지 못했습니다"}`를 준다. 토큰 오류는 401이다. Pi는 저장해 둔 목록을 계속 쓴다.

## 3. Backend 내부: AI 출력

`services/ai.py`는 LLM을 1회 호출하고, 출력은 아래 JSON 스키마로 강제한다. 도구 호출(function calling) 대신 스키마 하나를 쓰는
이유: 공급자와 무관하게 동작하고, Pydantic 검증이 한 번으로 끝나며, 응답의 `stt_text`로 쓸 `transcript`를 같은 호출에서 얻는다.

LLM 입력:

1. 오디오(`/api/voice`) 또는 텍스트(`/api/text`)
2. 현재 시각 (+09:00, 요일 포함)
3. 미완료 업무 최대 30개: `[{ "item_id", "name", "due_at", "status" }]`
4. `pending` — 재질문 대기 중인 원래 요청 (있을 때만)

LLM 출력:

```json
{
  "transcript": "내일 오후 3시까지 보고서 작성 추가해줘",
  "action": "create_task",
  "args": { "name": "보고서 작성", "due_at": "2026-10-09T15:00:00+09:00", "offset_minutes": null },
  "question": null
}
```

| `action` | `args` | 응답 `type` | 흡수한 기존 Intent |
| --- | --- | --- | --- |
| `create_task` | `name`, `due_at`, `offset_minutes`(선택) | `task_created` | CREATE_TASK |
| `list_tasks` | `range`: `today` · `tomorrow` · `week` · `all` | `task_list` | GET_TASKS, GET_TASK_STATUS |
| `complete_task` | `item_id` | `task_completed` | COMPLETE_TASK |
| `update_due` | `item_id`, `due_at` | `task_updated` | UPDATE_TASK |
| `delete_task` | `item_id` | 확인 질문 `clarification` → `task_deleted` | — (2026-11-06 추가, FR-24 변경) |
| `set_reminder` | `item_id`, `offset_minutes` | `reminder_set` | SET_REMINDER, UPDATE_REMINDER |
| `clarify` | `candidates`(선택, item_id 배열) + `question` 필수 | `clarification` | UNKNOWN, need_clarification |

현재 실행하는 action은 `create_task`·`list_tasks`·`complete_task`·`update_due`·`delete_task`·`clarify`다. `set_reminder`는 스키마와 프롬프트에 있으며 실행 시 미지원 오류를 반환한다. `delete_task`는 프롬프트 반영 전이라 AI가 아직 내보내지 않는다.

**삭제 확인**: `delete_task`는 바로 실행하지 않는다. Backend가 업무 이름과 마감을 말하며 되묻고 `pending`에 대상 `item_id`를 60초 보관한다. 다음 발화가 "응"·"네" 같은 긍정 답이면 삭제하고, "아니" 같은 부정 답이면 취소한다. 다른 말이면 대기를 지우고 새 요청으로 처리한다. 답 판정은 LLM이 아니라 Backend가 받아쓴 문장으로 한다.

Backend `services/tasks.py`가 강제하는 정책 (LLM에 맡기지 않는다):

1. `offset_minutes`가 없으면 60.
2. `item_id`가 백엔드가 조회한 미완료 목록에 없으면 실행하지 않고 `clarification`.
3. `due_at`이 과거면 `clarification`.
4. 스키마 검증에 실패하면 `status: ERROR` ("다시 한 번 말씀해 주세요").
5. Eureka가 2xx를 반환하고 응답 본문까지 확인한 뒤에만 성공 `type`을 돌려준다.
6. `output.tts`는 항상 Backend 템플릿이 만든다 (LLM을 두 번 부르지 않는다). 재질문만 LLM의 `question`을 쓴다.

**현재 구현 — Gemini API**. `services/ai.py`가 모델 설정을 읽고 오디오 또는 텍스트와 맥락으로 구조화 JSON을 한 번 받는다.

**이전 LLM 논의** ([[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 9번): 팀 결정은 "서버에서 Local LLM"
([[wiki/desk-pet-hardware-mvp-decisions|MVP 결정]] 16번)이고, 멘토 제안은 "1차는 무료 등급 API, Local LLM은 비교 실험"이다
(`ai/README.md`, [[projects/eureka-desk-pet/outputs/2026-10-08-tech-review|기술 검토]] § 로컬 LLM 비교). 위 입출력 형식은 어느 쪽이든 같다.

## 4. Backend ↔ Eureka 매핑

2026-09 PoC 스크립트(`desk-pet-backend` 레포 `tests/integration/Eureka_Process_API_Test/`)의 실제 호출 기준.

| 계약 필드 | Eureka API 필드 | 비고 |
| --- | --- | --- |
| `item_id` | `id` | |
| `name` | `name` | |
| `due_at` (ISO +09:00) | `dueAt` (epoch ms) | 변환은 `eureka.py`에서만 |
| `status` | 목록: `stage$$`, 단건: `GET /_api_/items/{id}/status`의 `status`·`total`·`done` | `todo` · `doing` · `done` · `hold` |

| 동작 | Eureka 호출 |
| --- | --- |
| 등록 | `POST /_api_/items/0/start` — `{processId, name, dueAt, priority} (`priority=high`)` |
| 목록 | `GET /_api_/items/0/list?sites&detail=true&limit=100` — 현재 원본 100건 이상이면 완료 필터 전에 오류. 페이지 처리 방식 확인 필요 |
| 완료 | `POST /_api_/items/{id}/complete` |
| 상태 | `GET /_api_/items/{id}/status` (PoC, 실연동 클라이언트 미구현) |
| 마감 변경 | `PUT /_api_/items/{id}` — `{dueAt}` (2026-11-06 실제 Eureka에서 저장 확인) |
| 삭제 | `DELETE /_api_/items/{id}` — 응답 `{"id": "<id>"}`. 이후 단건 조회는 404, 목록에서 빠짐 (2026-11-06 확인). 복구 가능 여부는 미확인 |

- 인증 헤더는 `x-api-key` (`EUREKA_API_KEY`). 아이디·비밀번호 방식이 아니다.
- 입력 검증 오류가 500으로 올 수 있으므로 상태 코드와 응답 본문을 함께 확인한다 (PoC 결과).
- Eureka는 Webhook과 마감 범위 검색을 지원하지 않는다.

**목록 조회 방식**. 2026-11-06 결정에 따라 요청마다 읽지 않고 캐시를 쓴다 (Backend PR #37, 2026-11-11 배포). 같은 날 서버 시험에서 요청당 목록 조회 시간이 약 1.3초에서 0초가 됐다.

| 시점 | 동작 |
| --- | --- |
| 서버 시작 시, 하루 1회 | 전체 목록을 조회해 로컬 캐시에 저장 |
| 요청 처리, `GET /api/tasks` | 캐시를 읽음 (Eureka 호출 없음) |
| 쓰기 성공 | 캐시를 바로 수정하고 약 10초 뒤 전체 조회. 변경 직후 조회에 수정 전 값이 나올 수 있어서다 |
| 쓰기 실패 | 즉시 전체 조회 |
| 조회 실패 | 기존 캐시 유지. 캐시가 아직 없으면 오류 |

여러 번 쓰면 마지막 쓰기 기준으로 한 번만 다시 조회한다. 조회 도중 쓰기가 있었으면 그 조회 결과는 버린다. `TASK_CACHE_SECONDS=0`이면 캐시를 끄고 요청마다 읽는다.

## 5. Pi 상태

[[wiki/desk-pet-device-states|장치 상태]]를 그대로 쓴다. 상태와 LED 색은 Pi가 정한다.

| 상태 | 들어가는 조건 | LED (color / mode) | LCD expression |
| --- | --- | --- | --- |
| `IDLE` | 부팅 완료, 출력 완료 | green / solid | idle |
| `LISTENING` | 버튼 누름 | yellow / blink | listening |
| `THINKING` | 버튼을 뗀 직후 (Pi가 직접) | white / blink | thinking |
| `SPEAKING` | `/api/voice` 응답이 `status: OK` | blue / blink | 응답의 `output.lcd.expression` |
| `ALERT` | Pi의 알람 시각 도달 (§ 2) — 구현 전 | orange / fast_blink | alert |
| `ERROR` | `status`가 `OK`가 아님, 응답 시간 초과(현재 10초, 변경 후 20초 — § 1), Wi-Fi·장치 오류 | red / blink | surprised |

- LED 밝기 상한 30%. `THINKING`·`ALERT`·`ERROR` 색은 제안값이다 (디자인 확정 전).
- `THINKING` 중 버튼 입력은 무시한다. `IDLE`이 아닐 때 알람 시각이 되면 지금 출력이 끝난 뒤 재생한다.
- `THINKING`에서 10초가 지나면 지연 안내를 낸다 (§ 1, 구현 전). 상태는 그대로다.
- 재생(`output.tts`)이 끝나면 `IDLE`로 돌아간다.

## 6. Backend 저장소 (SQLite)

```sql
CREATE TABLE reminders (
  item_id TEXT PRIMARY KEY,
  offset_min INTEGER NOT NULL DEFAULT 60
);

CREATE TABLE pending (
  user_id TEXT PRIMARY KEY,
  payload TEXT NOT NULL,        -- 재질문 전 원래 요청 (JSON)
  expires_at INTEGER NOT NULL   -- 60초 후
);
```

- `pending`과 `reminders`를 사용한다. `reminders`의 알림 간격은 `GET /api/tasks`로 Pi에 전달한다.
- `alerts` 테이블은 쓰지 않기로 했다 (2026-11-06). 코드에는 정의가 남아 있고 구현 때 삭제한다. 알림 기록은 Pi가 저장한다 (§ 2).
- 업무명·마감·상태는 SQLite에 저장하지 않는다. 원본은 Eureka이고, 메모리 캐시에는 읽기용 사본을 둔다 (§ 4).

## 변경 이력

- 2026-10-08: 초안 작성. 같은 날 10/07 하드웨어 결정(Mock 형식: `/api/voice`, `/ws/hardware/{user_id}`, `PING`/`PONG`,
  `output` 봉투, snake_case)에 맞춰 이름과 모양을 정렬했다.
  [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 중 이 계약이 답을 정한 항목: 1(9초 상한), 2(재질문 연결 — `pending`),
  3(STT는 Backend), 4(`robotCommand` 없음), 5(`OK`), 6(`output.tts`는 Backend 템플릿), 7(WAV 고정), 8(`+09:00`), 10(재질문 표정 `thinking`).
  남은 항목: 9(LLM 방식).
- 2026-10-29: [[projects/eureka-desk-pet/meetings/2026-10-29-mentor-meeting-note|10-29 멘토 미팅]] 결정 반영.
  `request_id`를 요청에서 빼고 Backend가 만들어 응답에만 넣는다(§ 0-5, § 1). `user_id`는 선택값, 로그인 없이 계정 하나(§ 0-4).
  WebSocket 선제 알림은 1차 제외(§ 2, § 5 `ALERT`). 응답 모양은 슬라이드 초안 대신 현재 Backend `schemas.VoiceResponse`를 그대로 두고,
  실제 코드의 HTTP 오류 코드(401·413·422)를 § 1에 적었다.
  코드 반영 필요: Backend `api/voice.py`·`schemas.TextRequest`(`request_id` 생성, `user_id` 선택), Hardware `backend_client.send_voice()`(`request_id` 제거).
- 2026-11-04: `request_id` 방식을 Pi 쪽에 맞춰 변경. Pi가 보낸 값을 응답에 그대로 돌려주고, 생략하면 Backend가 만든다(§ 0-5, § 1).
  10-29 결정의 `request_id` 항목을 대체한다. Hardware `send_voice()`의 `request_id` 제거는 더 이상 필요 없다.

- 2026-11-05: 백엔드 코드와 대조해 처리 시간, Gemini 호출, 실행 가능한 업무 동작, WebSocket 및 저장소 구현 범위를 기록했다.
- 2026-11-06: `update_due` 실행 구현 반영(§ 3, § 4). 형식 변경은 없다. 같은 날 실제 Eureka에서 저장을 확인했다.
- 2026-11-06: 팀 결정 반영 (구현 전). Pi 지연 안내 10초·실패 20초, Backend 처리 제한 조정, LLM 재시도 1회(§ 1). WebSocket 알림 대신 `GET /api/tasks` + Pi 자체 알람(§ 2, § 5, 하드웨어 팀 동의). 목록 캐시(§ 4). `alerts` 테이블 제외(§ 6). 남은 리뷰: 지연 안내 방식(하드웨어 팀), Eureka 웹 변경의 반영 지연(전체).
- 2026-11-06: 음성 삭제 추가 — `delete_task`, `task_deleted`, `task_delete_cancelled`, 확인 질문 필수(§ 1, § 3, § 4). 기획서 FR-24 "삭제 제외"를 바꾸는 결정이다.
- 2026-11-11: `GET /api/tasks`(§ 2)와 목록 캐시(§ 4)를 Backend에 구현해 서버에 배포했다. 목록 조회 실패 시 503(§ 2)을 적었다. 형식 변경은 없다.
