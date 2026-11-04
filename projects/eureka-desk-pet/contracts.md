# eureka-desk-pet 레이어 간 계약 (1차)

> 레이어 간에 주고받는 형식의 **단일 진실원**. 바꾸려면 이 파일 PR을 먼저 올리고, merge 후 각 코드 레포에 반영한다.
> 상태: **초안 (2026-10-08)**, 2026-10-29 멘토 미팅 결정 반영 — 팀 검토 후 확정. 구조와 흐름은 `architecture.md`를 본다.
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
   Backend가 만드는 음성 요청 id는 `req_20261029_153000_a1b2`, 알림은 `alert_<item_id>_<due_at>` 형식이다.
6. **추가 — 인증**: Pi → Backend 요청과 WS 연결에 헤더 `X-Device-Token`을 붙인다. 값은 양쪽 `.env`에만 두고 커밋하지 않는다.
7. 1차는 같은 Wi-Fi 안에서 HTTP/WS를 쓴다. 외부 배포 시 HTTPS/WSS로 바꾼다 (`architecture.md` § 1차에서 뺀 것).

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
| `task_created`, `task_completed`, `task_updated`, `task_list`, `reminder_set` | 처리 성공 |
| `clarification` | 재질문. `output.tts`가 질문 문장 |
| `error` | `status`가 `OK`가 아닐 때 |

- **시간**. 계약 초안의 Backend 응답 상한은 9초이고 Pi는 업로드 시작부터 최대 10초를 기다린다. 현재 백엔드 코드는 업로드·입력 검증이 끝난 뒤 업무 처리에 7초 제한을 적용한다. 처리 제한을 넘으면 `ERROR`를 반환한다.
- **현재 반환 범위**. `OK`와 `ERROR`를 반환한다. `NO_SPEECH`·`STT_ERROR`는 스키마에 정의되어 있고 반환 분기는 없다. `task_updated`·`reminder_set`도 현재 처리에서 생성하지 않는다.
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

## 2. Backend ↔ Pi: WebSocket — 1차 제외

> **2026-10-29 결정: 선제 알림은 1차에서 뺀다.** 중요도가 낮아 하드웨어 작업 목록에서 WebSocket 항목을 모두 빼고,
> Backend는 여유가 있을 때 구현한다. 아래 형식은 2차 기준으로 유지한다. 코드에는 토큰 검사, 연결·`connected`·`PING`/`PONG`, 전송 함수가 있다. 폴러 연결, 알림 판단, 재전송은 미구현이다.

연결: `ws://<backend-host>:<port>/ws/hardware/{user_id}` + 헤더 `X-Device-Token`

| 방향 | 메시지 | 언제 |
| --- | --- | --- |
| Backend → Pi | `{"event": "connected", "user_id": "pet-01"}` | 접속 직후 |
| Backend → Pi | `proactive_alert` (아래) | 마감 임박 (기본 1시간 전) |
| Backend → Pi | `output` (`proactive_alert`와 같은 모양, `event`만 다름) | Backend가 먼저 보내는 일반 출력 — 1차에서는 쓰는 곳이 없다 |
| Pi → Backend | 텍스트 `PING` | 30초마다 |
| Backend → Pi | `{"event": "PONG"}` | `PING`에 대한 응답 |

```json
{
  "event": "proactive_alert",
  "type": "deadline_approaching",
  "user_id": "pet-01",
  "request_id": "alert_1000010_1791525600000",
  "output": {
    "tts": "보고서 검토 마감이 1시간 남았어요.",
    "lcd": { "expression": "alert", "text": "마감 1시간 전" }
  }
}
```

- `PONG`이 오지 않거나 연결이 끊기면 Pi가 3초 간격으로 다시 연결한다.
- 재연결되면 Backend가 미전달 알림의 업무 상태를 다시 확인하고, 남은 시간을 새로 계산해 보낸다.
- 선제 알림의 초기 구현에서는 Backend의 `send` 성공을 전달 완료로 본다. Pi 수신 확인(ack)은 후속 과제다.
- JSON 파싱에 실패한 메시지는 Pi가 버리고 로그만 남긴다.

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
| `set_reminder` | `item_id`, `offset_minutes` | `reminder_set` | SET_REMINDER, UPDATE_REMINDER |
| `clarify` | `candidates`(선택, item_id 배열) + `question` 필수 | `clarification` | UNKNOWN, need_clarification |

현재 실행하는 action은 `create_task`·`list_tasks`·`complete_task`·`clarify`다. `update_due`·`set_reminder`는 스키마와 프롬프트에 있으며 실행 시 미지원 오류를 반환한다.

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
| 마감 변경 | `PUT /_api_/items/{id}` — `{dueAt}` (PoC, 실연동 클라이언트 미구현) |

- 인증 헤더는 `x-api-key` (`EUREKA_API_KEY`). 아이디·비밀번호 방식이 아니다.
- 입력 검증 오류가 500으로 올 수 있으므로 상태 코드와 응답 본문을 함께 확인한다 (PoC 결과).
- Eureka는 Webhook과 마감 범위 검색을 지원하지 않는다 → 60초 폴링 + Backend 필터링.

## 5. Pi 상태

[[wiki/desk-pet-device-states|장치 상태]]를 그대로 쓴다. 상태와 LED 색은 Pi가 정한다.

| 상태 | 들어가는 조건 | LED (color / mode) | LCD expression |
| --- | --- | --- | --- |
| `IDLE` | 부팅 완료, 출력 완료 | green / solid | idle |
| `LISTENING` | 버튼 누름 | yellow / blink | listening |
| `THINKING` | 버튼을 뗀 직후 (Pi가 직접) | white / blink | thinking |
| `SPEAKING` | `/api/voice` 응답이 `status: OK` | blue / blink | 응답의 `output.lcd.expression` |
| `ALERT` | `proactive_alert` 수신 — **2차** (§ 2) | orange / fast_blink | alert |
| `ERROR` | `status`가 `OK`가 아님, 응답 10초 초과, Wi-Fi·장치 오류 | red / blink | surprised |

- LED 밝기 상한 30%. `THINKING`·`ALERT`·`ERROR` 색은 제안값이다 (디자인 확정 전).
- `THINKING` 중 버튼 입력은 무시한다. `IDLE`이 아닐 때 온 알림은 지금 출력이 끝난 뒤 재생한다.
- 재생(`output.tts`)이 끝나면 `IDLE`로 돌아간다.

## 6. Backend 저장소 (SQLite)

```sql
CREATE TABLE reminders (
  item_id TEXT PRIMARY KEY,
  offset_min INTEGER NOT NULL DEFAULT 60
);

CREATE TABLE alerts (
  item_id TEXT NOT NULL,
  due_at INTEGER NOT NULL,      -- epoch ms, Eureka 값 그대로
  offset_min INTEGER NOT NULL,
  status TEXT NOT NULL,         -- sent | undelivered
  sent_at INTEGER,
  PRIMARY KEY (item_id, due_at, offset_min)
);

CREATE TABLE pending (
  user_id TEXT PRIMARY KEY,
  payload TEXT NOT NULL,        -- 재질문 전 원래 요청 (JSON)
  expires_at INTEGER NOT NULL   -- 60초 후
);
```

- 현재 `pending`과 `reminders`를 사용한다. `alerts`는 테이블 정의 단계다.
- 2차의 중복 알림 방지는 `alerts`의 PRIMARY KEY가 맡는다. 키에 `due_at`이 있어 마감을 연기하면 알림이 다시 온다.
- 업무명·마감·상태는 저장하지 않는다. 원본은 Eureka다.

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
