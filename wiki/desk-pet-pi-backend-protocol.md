---
type: protocol
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/데스크펫 MVP 아키텍처.md"
  - "raw/deskpet_AI_HW_충돌검토.md"
related:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-device-states|장치 상태]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Pi–Backend Protocol

## Summary

Pi와 Backend는 두 통로만 쓴다. Backend → Pi 알림·출력은 WebSocket `/ws/hardware/{user_id}`, Pi → Backend 음성 요청은 HTTP `POST /api/voice` 한 번이다. 이벤트 이름은 MVP Mock Server 형식(`connected`·`proactive_alert`·`output`·`PONG`)을 그대로 쓰고, 추적용 `request_id`만 추가했다. 기기는 WebSocket 주소의 `user_id`로만 식별한다(MVP는 사용자 1명당 펫 1대).

## Details

### 공통 필드

`event` · `type` · `user_id` · `request_id` · `output`. `output`은 `tts`(문장)와 `lcd`(`expression`, `text`)를 담는다. Backend가 `led` 필드를 보내도 Pi는 무시한다 — LED 색은 Pi의 상태표가 정한다([[wiki/desk-pet-device-states|장치 상태]]).

### Pi가 받는 메시지 (WebSocket)

| event | 언제 | Pi 동작 |
|---|---|---|
| `connected` | 접속 직후 | `IDLE` 유지 |
| `proactive_alert` | 마감 임박(기본 1시간 전) | `ALERT` → tts 재생 → `IDLE` |
| `output` | Backend가 먼저 보내는 일반 출력 | `SPEAKING`(`type: error`면 `ERROR`) → tts 재생 → `IDLE` |
| `PONG` | Pi의 PING 응답 | 연결 유지 확인 |

폐기된 이벤트: `state`(Pi가 스스로 `THINKING` 전환), `response`·`error`(HTTP 응답으로 대체).

`proactive_alert` 예시 (원문 예시에서 사용자 식별값을 자리표시자로 바꿈):

```json
{
  "event": "proactive_alert",
  "type": "deadline_approaching",
  "user_id": "<user_id>",
  "request_id": "alert_<task_id>_<due_at>",
  "output": {
    "tts": "<이름> 님, 보고서 검토 마감이 1시간 남았어요!",
    "lcd": { "expression": "alert", "text": "마감 1시간 전" }
  }
}
```

### Pi가 보내는 메시지

| 통로 | 메시지 | 언제 |
|---|---|---|
| WebSocket (텍스트) | `PING` | 30초마다 |
| HTTP | `POST /api/voice` | 버튼을 뗀 뒤 녹음 파일 업로드 |

폐기: WebSocket `hello`·`audio_start`·바이너리·`audio_end`·`device_status`. 장치 상태는 Pi 안에서 LED와 로그로만 처리한다.

### `POST /api/voice`

- 요청: `multipart/form-data` — `audio`(WAV, 16kHz · mono · 16bit, 최대 15초), `user_id`, `request_id`.
- Backend는 STT부터 업무 처리까지 끝낸 뒤 출력을 응답 본문에 담는다.
- `status`: `OK` · `NO_SPEECH` · `STT_ERROR` · `ERROR`.
- Pi는 10초 안에 응답이 없으면 `ERROR`를 표시하고 `IDLE`로 복귀한다.

```json
{
  "request_id": "req_20261007_001",
  "status": "OK",
  "stt_text": "내일 오후 3시까지 보고서 작성 추가해줘",
  "type": "task_created",
  "output": {
    "tts": "보고서 작성 업무를 등록했어요.",
    "lcd": { "expression": "happy", "text": "업무 등록 완료" }
  }
}
```

### 연결 유지

`IBackendConnection`은 Python `asyncio` + `websockets`로 상시 연결한다. PING 30초 주기, PONG이 없으면 끊김으로 보고 3초 간격으로 재연결한다. 재연결 시 Backend가 미전달 알림을 재전송한다. JSON 파싱에 실패한 메시지는 버리고 로그만 남긴다.

## Connections

- 전체 흐름 속 위치: [[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]
- 이 메시지를 다루는 Pi 인터페이스: [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]] (4) `IBackendConnection`, (5) `IVoiceUpload`
- 이전 설계(WebSocket PCM 스트리밍, Backend `state` 이벤트)에서 바뀐 이유: [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]
- AI 처리 구조 문서와의 형식 충돌(상태값 `SUCCESS`/`OK`, 시간대, 재질문 연결): [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]

## Open Questions

- `POST /api/voice` 신설과 응답에 `output` 포함 — Backend 담당 합의 대기. Mock Server에는 파일을 받아 고정 문장을 돌려주는 가짜 엔드포인트부터 추가할 예정 — needs-update.
- 모든 메시지에 `request_id` 추가 — Backend 합의 대기 — needs-update.
- `type` 값 목록은 `task_created | task_completed | ... | error`로만 적혀 있어 전체 목록이 미정이다.
- AI 쪽 최악 처리 시간이 약 16초라 Pi의 10초 대기와 충돌한다. Backend 상한 9초 이하 + Pi 대기를 그보다 길게 하는 안이 나와 있다 — [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 1번, needs-update.
- 재질문 답변을 이전 요청과 잇는 값이 없다. `conversationId` 입력과 `type: clarification` 추가가 제안됐다 — [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 2번, needs-update.
- 시각 필드는 Mock과 같이 `+09:00`을 붙이는 쪽으로 AI 문서를 맞추자는 제안이 있다 — [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 8번.
