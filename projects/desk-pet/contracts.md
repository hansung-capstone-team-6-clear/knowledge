# desk-pet 레이어 간 계약 (1차)

> 레이어 간에 주고받는 형식의 **단일 진실원**. 바꾸려면 이 파일 PR을 먼저 올리고, merge 후 각 코드 레포에 반영한다.
> 상태: **초안 (2026-10-08)** — 팀 검토 후 확정. 구조와 흐름은 `architecture.md`를 본다.

## 0. 공통 규칙

1. JSON 필드 이름은 camelCase.
2. 시각은 ISO-8601 + `+09:00` (예: `2026-10-09T15:00:00+09:00`). Eureka의 epoch ms 변환은 Backend `eureka.py`에서만 한다.
3. 알림 시점은 "마감 N분 전"을 뜻하는 정수 `offsetMinutes` 하나로 표현한다. 절대 시각(`reminderAt`)과
   `reminder_offset_minutes`, `alert_before_minutes` 같은 기존 이름은 쓰지 않는다.
4. Pi → Backend 요청과 WS 연결은 헤더 `X-Device-Token`으로 인증한다. 토큰 값은 `.env`에만 두고 커밋하지 않는다.
5. 1차는 같은 Wi-Fi 안에서 HTTP/WS를 쓴다. 외부 배포 시 HTTPS/WSS로 바꾼다 (`architecture.md` § 1차에서 뺀 것).

## 1. Pi → Backend: 음성 요청

`POST /voice` — `multipart/form-data`

| 필드 | 형식 |
| --- | --- |
| `audio` | WAV, 16kHz, mono, 16bit, 최대 15초 |
| `deviceId` | string (예: `pet-01`) |

`POST /text` — `application/json`. 개발·평가·비상 시연용이며 응답은 `/voice`와 같다.

```json
{ "deviceId": "pet-01", "text": "내일 오후 3시까지 보고서 검토 추가해줘" }
```

응답 (`/voice`, `/text` 공통, HTTP 200):

```json
{ "requestId": "r-20261008-0001", "type": "task_created", "text": "보고서 검토 업무를 등록했어요. 마감은 내일 오후 3시예요." }
```

| `type` | 의미 |
| --- | --- |
| `task_created`, `task_completed`, `task_updated`, `task_list`, `reminder_set` | 처리 성공 |
| `clarify` | 재질문. `text`가 질문 |
| `no_speech` | 음성을 알아듣지 못함. 다시 말해 달라는 안내 |
| `error` | 처리 실패. `text`는 사용자 안내 문구 |

- Eureka 호출이 실패하거나 타임아웃이면 성공 문구를 보내지 않고 `error`로 응답한다.
- Pi는 10초 안에 응답이 없으면 ERROR 상태를 보여 주고 IDLE로 돌아간다.

## 2. Backend → Pi: 선제 알림 (WebSocket)

연결: `ws://<backend-host>:<port>/ws/{deviceId}` + 헤더 `X-Device-Token`

서버 → Pi:

```json
{
  "event": "proactive_alert",
  "itemId": "1000010",
  "taskName": "보고서 검토",
  "remainingMinutes": 60,
  "text": "보고서 검토 마감이 1시간 남았어요."
}
```

- Pi는 30초마다 `{"event": "ping"}`을 보내고, 서버는 `{"event": "pong"}`으로 답한다. 응답이 없으면 Pi가 5초 간격으로 재연결한다.
- 1차는 서버의 `send` 성공을 전달 완료로 본다. Pi의 수신 확인(ack)은 2차.
- 재연결 시 서버는 미전달 알림의 업무 상태를 다시 확인하고 `remainingMinutes`를 새로 계산해 보낸다.

## 3. Backend 내부: AI 출력

`ai.py`는 LLM을 1회 호출하고, 출력은 아래 JSON 스키마로 강제한다. 도구 호출(function calling) 대신 스키마 하나를 쓰는
이유: 공급자와 무관하게 동작하고, Pydantic 검증이 한 번으로 끝나며, 로그용 `transcript`를 같은 호출에서 얻는다.

LLM 입력:

1. 오디오(`/voice`) 또는 텍스트(`/text`)
2. 현재 시각 (+09:00, 요일 포함)
3. 미완료 업무 최대 30개: `[{ "itemId", "name", "dueAt", "status" }]`
4. `pending` — 재질문 대기 중인 원래 요청 (있을 때만)

LLM 출력:

```json
{
  "transcript": "내일 오후 3시까지 보고서 검토 추가해줘",
  "action": "create_task",
  "args": { "name": "보고서 검토", "dueAt": "2026-10-09T15:00:00+09:00", "offsetMinutes": null },
  "question": null
}
```

| `action` | `args` | 흡수한 기존 Intent |
| --- | --- | --- |
| `create_task` | `name`, `dueAt`, `offsetMinutes`(선택) | CREATE_TASK |
| `list_tasks` | `range`: `today` · `tomorrow` · `week` · `all` | GET_TASKS, GET_TASK_STATUS |
| `complete_task` | `itemId` | COMPLETE_TASK |
| `update_due` | `itemId`, `dueAt` | UPDATE_TASK |
| `set_reminder` | `itemId`, `offsetMinutes` | SET_REMINDER, UPDATE_REMINDER |
| `clarify` | `candidates`(선택, itemId 배열) + `question` 필수 | UNKNOWN, need_clarification |

Backend `tasks.py`가 강제하는 정책 (LLM에 맡기지 않는다):

1. `offsetMinutes`가 없으면 60.
2. `itemId`가 입력으로 준 미완료 목록에 없으면 실행하지 않고 `clarify`.
3. `dueAt`이 과거면 `clarify`.
4. 스키마 검증에 실패하면 `error` ("다시 한 번 말씀해 주세요").
5. Eureka가 2xx를 반환하고 응답 본문까지 확인한 뒤에만 성공 `type`을 돌려준다.

## 4. Backend ↔ Eureka 매핑

2026-09 PoC 스크립트(`desk-pet-backend` 레포 `tests/integration/Eureka_Process_API_Test/`)의 실제 호출 기준.

| 계약 필드 | Eureka API 필드 | 비고 |
| --- | --- | --- |
| `itemId` | `id` | |
| `name` | `name` | |
| `dueAt` (ISO +09:00) | `dueAt` (epoch ms) | 변환은 `eureka.py`에서만 |
| `status` | 목록: `stage$$`, 단건: `GET /_api_/items/{id}/status`의 `status`·`total`·`done` | `todo` · `doing` · `done` · `hold` |

| 동작 | Eureka 호출 |
| --- | --- |
| 등록 | `POST /_api_/items/0/start` — `{processId, name, dueAt, priority}` |
| 목록 | `GET /_api_/items/0/list?sites&detail=true&limit=100` — 100개를 넘으면 다음 페이지 (페이지 파라미터는 `needs-update`) |
| 완료 | `POST /_api_/items/{id}/complete` |
| 상태 | `GET /_api_/items/{id}/status` |
| 마감 변경 | `PUT /_api_/items/{id}` — `{dueAt}` |

- 인증 헤더는 `x-api-key` (`EUREKA_API_KEY`). 아이디·비밀번호 방식이 아니다.
- 입력 검증 오류가 500으로 올 수 있으므로 상태 코드와 응답 본문을 함께 확인한다 (PoC 결과).
- Eureka는 Webhook과 마감 범위 검색을 지원하지 않는다 → 60초 폴링 + Backend 필터링.

## 5. Pi 상태 매핑

| 상태 | 언제 | LED (WS2812B) | LCD 표정 |
| --- | --- | --- | --- |
| IDLE | 대기 | 초록 점등 | 기본 |
| LISTENING | 버튼 누르는 동안 (Pi가 직접) | 노랑 점멸 | 듣는 중 |
| THINKING | 버튼 뗀 뒤 응답 대기 (Pi가 직접) | 노랑 점등 | 생각 중 (…) |
| SPEAKING | 성공 `type` 또는 `clarify` 재생 중 | 파랑 점멸 | 말하는 중 |
| ALERT | `proactive_alert` 재생 중 | **주황** 점멸 | 안내/알림 + 업무명 |
| ERROR | `error`, `no_speech`, 응답 타임아웃, 연결 끊김 | **빨강** 점등 | 오류 문구 |

재생이 끝나면 IDLE로 돌아간다. 표정 에셋은 팀 디자인 시안(기본·듣는 중·말하는 중·생각 중·놀람·안내/알림)을 그대로 쓴다.

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
  device_id TEXT PRIMARY KEY,
  payload TEXT NOT NULL,        -- 재질문 전 원래 요청 (JSON)
  expires_at INTEGER NOT NULL   -- 60초 후
);
```

- 중복 알림 방지는 `alerts`의 PRIMARY KEY가 맡는다. 키에 `due_at`이 있어 마감을 연기하면 알림이 다시 온다.
- 업무명·마감·상태는 저장하지 않는다. 원본은 Eureka다.

## 변경 이력

- 2026-10-08: 초안 작성.
