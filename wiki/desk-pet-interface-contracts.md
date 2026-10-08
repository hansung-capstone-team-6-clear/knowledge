---
type: protocol
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 03 AI & 데이터.md"
  - "raw/Capstone Eureka Process - 04 Backend.md"
  - "raw/Capstone Eureka Process - 05 Raspberry Pi.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
  - "raw/desk-pet-backend-architecture.md"
related:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/eureka-process-api|Eureka Process API]]"
  - "[[wiki/intent-routing-and-structured-output|Intent Routing & Structured Output]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Interface Contracts

## Summary

하드웨어·AI·Backend를 서로 다른 사람이 독립적으로 개발하므로, 계층 사이 데이터 전달을 **JSON 인터페이스**로 고정해
충돌을 막는다(10/01 멘토 피드백). 각 인터페이스는 `interface`, `input`, `output`, `callMethod`, `processing`,
`errorHandling`, `timeout` 필드로 기술한다. 이 노트는 인터페이스 목록과 핵심 이벤트를 모은 색인이며, 필드 원문은
각 raw 문서에 있다.

## Details

### 인터페이스 목록

| 영역 | 인터페이스 | 방향 / 호출 | Timeout |
| --- | --- | --- | --- |
| Hardware | `IButtonInput` | GPIO 이벤트 → `BUTTON_PRESSED/RELEASED` | 녹음 최대 15초 |
| Hardware | `IAudioCapture` | 녹음 → `requestId`, WAV/PCM, `sampleRate` | 최대 15초 |
| Hardware | `IHardwareOutput` | 스피커·LED·LCD 제어 | 비정상 지속 시 IDLE 복귀 |
| Hardware | `IBackendConnection` | Pi → Backend WebSocket `ws://{host}:{port}/ws/hardware/{userId}` | heartbeat 30초 |
| Hardware | `IVoiceUpload` | `HTTPS POST /api/voice` (multipart) → STT 결과 | 5초 |
| Hardware | `IDeviceStatusMonitor` | 60초 주기 + 변경 시 `device_status` 보고 | — |
| STT | `SpeechToText` | 음성 → 텍스트, `SUCCESS / NO_SPEECH / STT_ERROR` | 목표 1초, 최대 5초 |
| AI | `ClassifyIntent` | 텍스트 → intent, `confidence`, `requiresLLM` | 목표 0.5초, 최대 1초 |
| AI | `HandleSimpleCommand` | LLM 불필요 명령 처리 | 최대 1초 |
| AI | `AnalyzeUserRequest` | Backend → LLM API, 파라미터·`requiredTool`·재질문 | 최대 5초 |
| AI | `CallMCPTool` | Backend MCP Client → Tool, `requestId`로 중복 방지 | 3~5초 |
| Backend API | `CreateTaskAPI` | `POST /api/tasks` | 내부 0.5초 |
| Backend API | `GetTasksAPI` | `GET /api/tasks` (status·dueBefore·priority 필터) | 5초 |
| Backend API | `CompleteTaskAPI` | `POST /api/tasks/{itemId}/complete` | 5초 |
| Backend API | `GetTaskStatusAPI` | `GET /api/tasks/{itemId}/status` | 5초 |
| Backend API | `UpdateTaskAPI` | `PUT /api/tasks/{itemId}` | 5초 |
| Eureka | `CreateTask` / `GetTasks` / `CompleteTask` / `GetTaskStatus` / `UpdateTask` | Backend → Eureka HTTP ([[eureka-process-api\|Eureka Process API]]) | 목표 1초, 최대 2초 |
| Second Brain | `SaveWorkRecord` / `UpdateWorkContext` / `SearchContext` / `SaveReminderSetting` | MCP Tool 또는 Backend 내부 서비스 | 3초 |
| Output | `GenerateOutput` | 결과 → `ttsText`, `ledState`, `lcdExpressionId`, `lcdText` | 목표 0.2초, 최대 0.5초 |
| Output | `SendOutputCommand` | Backend → Pi, 실패 시 능동 알림은 미전달 기록 | 무응답 시 네트워크 오류 |

> Hardware 행은 9/30 원안 기준이다. 2026-10-07 하드웨어 정리안은 `IAudioCapture`를 WAV 16kHz·mono·16bit로 고정하고, `IVoiceUpload` 응답에
> 최종 `output`을 포함·대기 10초로 늘렸으며, `IDeviceStatusMonitor`는 Backend 보고 없이 Pi 내부에서 처리하고, `IBackendConnection`에서 `ack`를 뺐고,
> `IDeviceStateManager`를 추가했다 — [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]. AI 쪽 `SUCCESS`는 `OK`로 맞추자는 제안이 있다
> ([[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 5번).

공통 오류 원칙: 외부 처리 실패를 성공으로 안내하지 않고, 대상이 모호하면 변경하지 않고 확인을 요청하며, 조회 실패 시
값을 추정하지 않는다.

### WebSocket 이벤트

Pi가 받는 이벤트: `connected`, `proactive_alert`, `output`, `PONG`, `ack`. 연결 상태는 `CONNECTED / DISCONNECTED /
RECONNECTING`이고, 재연결 시 Backend가 미전달 알림을 재전송한다.

**`proactive_alert` (Mock Server 구현 형식)**

```json
{
  "event": "proactive_alert",
  "type": "deadline_approaching",
  "user_id": "team-park-sihyeon",
  "output": {
    "tts": "시현 님, 보고서 검토 마감이 1시간 남았어요!",
    "led": { "color": "yellow", "mode": "blink" },
    "lcd": { "expression": "alert", "text": "마감 1시간 전" }
  }
}
```

9/30 하드웨어 안은 `event_type: "PROACTIVE_ALERT"`, `task_id`, `due_at`, `remaining_minutes`,
`hardware_control.{led_state, lcd_expression, lcd_text}`, `audio_stream_url` 형식을 제안했다. Mock의 `tts` 문장은 임시이고,
최종적으로는 LLM이 상황·맥락으로 생성할 예정이다.

개정 다이어그램에 임베드된 계약(2026-10-08, `raw/desk-pet-backend-architecture.md`, 21개)은 세 번째 형식을 쓴다:
`event`, `type: deadline_approaching`, `user_id`, `request_id`, `output.tts`, `output.lcd.expression: alert`,
`output.lcd.text` — **`led` 필드가 없고** `request_id`가 추가됐다. 엔드포인트는 `/ws/hardware/{user_id}`.
같은 개정판에서 `SendOutputCommand`는 **서버 주도 push 전용**으로 좁혀졌다(일반 음성 요청의 HTTP Response는 이 계약을
거치지 않음). WebSocket 전송 성공은 기기 재생 확인이 아니며, 재생 ACK는 가정하지 않는다. `GenerateOutput`은 TTS 문장만
만들고 음성 합성·LED 제어는 Pi 책임이다. 개정판에는 `IBackendConnection`·`IVoiceUpload`·`IDeviceStatusMonitor`·
`HandleSimpleCommand`·Second Brain 계약이 없고, 대신 `ParseSimpleRequest`(경량 파싱 전용)가 추가됐다.

### 전체 요청 흐름 (10단계)

User → Hardware(PTT) → STT → Lightweight Intent → Backend → (`requiresLLM`) LLM → (`externalToolRequired`) MCP Tool →
(`taskOperation`) Eureka → Output Controller → Hardware → User.

### 10/01 피드백: JSON으로 고정할 경계

Hardware ↔ Backend, Backend ↔ AI, Backend ↔ Eureka Process, Backend ↔ Scheduler, Backend → Hardware 알림.

## Connections

- 계층 구조 → [[desk-pet-system-architecture|Desk Pet System Architecture]]
- AI 출력 스키마 → [[intent-routing-and-structured-output|Intent Routing & Structured Output]]
- 알림 이벤트의 쓰임 → [[proactive-deadline-alert|Proactive Deadline Alert]]
- Pi 측 장치 → [[desk-pet-hardware|Desk Pet Hardware]]

## Open Questions

- `proactive_alert` 스키마가 세 가지(Mock 형식 vs 9/30 `PROACTIVE_ALERT` 형식 vs 개정 다이어그램의 `request_id`·무`led` 형식)로 공존한다 — 10/07 하드웨어 정리안은 세 번째(Mock + `request_id`, `led`는 보내도 무시)를 택했다 — [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]. Backend 합의 대기 (needs-update).
- `proactive_alert` 전달 실패 시 "재처리 정책"이 개정 계약에 이름만 있고 내용이 없다.
- 필드 표기 혼용: Backend API는 camelCase(`taskName`, `dueAt`), AI 출력은 snake_case(`task_name`, `due_at`).
- `ledState` 알림 전용 상태 부재 → 10/07 정리안에서 6종(`THINKING`·`ALERT` 추가)으로 확장됐다 — [[wiki/desk-pet-device-states|장치 상태]].
- WebSocket URL이 `ws://`로 적혀 있으나 NFR-11은 HTTPS(암호화) 통신을 기본으로 한다 → `wss://` 여부 확인.
