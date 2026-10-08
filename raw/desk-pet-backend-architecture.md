---
title: "Desk Pet Backend Architecture Diagram"
source: "archify 3.0.1 HTML export — backend_architecture.html"
commit: "n/a"
branch: "n/a"
captured: 2026-10-08
author: sensior101
note: "archify가 생성한 단일 HTML(약 820KB, 대부분 임베디드 폰트·뷰어 JS)에서 의미 콘텐츠만 기계 추출한 캡처본. 뷰어 UI 문자열·CSS·폰트·뷰어 스크립트는 제외했고, 다이어그램 텍스트·엣지 속성·Notes·Interface Contracts JSON은 원문 값 그대로 옮겼다. 이후 무수정."
---

# Desk Pet Backend Architecture

Backend orchestration · Eureka Process integration · proactive deadline alerts

## Diagram text (SVG `<text>` in document order)

```text
External Input Reference
User / PTT
voice command / feedback
Hardware
PTT · microphone · LED · LCD · speaker
Audio Capture
Raspberry Pi · WAV recording
STT
server speech → text · ko-KR
LLM Service
MCP Client
Lightweight Intent Model
classify · confidence · parameters
Intent / LLM Orchestrator
validate AI / MCP · route services
clarification → no data mutation
TaskService
create · list · status · complete · update
ReminderService
deadline check · deduplication · reminder policy
OutputService
user response · TTS text · LCD output
TaskPort
Application ↔ Eureka abstraction
OutputPublisherPort
server-initiated output / proactive_alert
asynchronous publishing only
SchedulerAdapter
periodic reminder trigger
driver
EurekaTaskAdapter
TaskPort implementation · HTTP mapping
adapter
WebSocketAdapter
output / proactive_alert delivery
adapter
Eureka Process API
task source of truth · REST
Raspberry Pi
Edge-TTS · LED · LCD · speaker
input
PCM
POST /api/voice · audio
text
intent
calls
uses
HTTP / response
triggers
queries
output
publishes
proactive_alert / output
via implementation
via implementation
complex / low confidence
analysis · no tool
reminder intent
task result
HTTP Response · normal reply
clarification
tool
MCP result
Device / Hardware Layer
Local Processing Layer
Application Layer
Ports
Infrastructure / Adapter Layer
External Systems
Legend
Frontend
Backend
Cloud
Message bus
External
```

## Edges (`data-edge-from` → `data-edge-to` : `data-edge-label`)

| key | from | to | label |
| --- | --- | --- | --- |
| 0 | user | hardware | input |
| 1 | hardware | audio-capture | PCM |
| 2 | audio-capture | stt | POST /api/voice · audio |
| 3 | stt | intent-model | text |
| 4 | intent-model | orchestrator | intent |
| 5 | orchestrator | task-service | calls |
| 6 | task-service | task-port | uses |
| 7 | task-port | eureka-adapter | via implementation |
| 8 | eureka-adapter | eureka-api | HTTP / response |
| 9 | scheduler | reminder-service | triggers |
| 10 | reminder-service | task-port | queries |
| 13 | reminder-service | output-service | output |
| 14 | output-service | output-port | publishes |
| 15 | output-port | websocket-adapter | via implementation |
| 16 | websocket-adapter | raspberry-pi | proactive_alert / output |
| 18 | intent-model | llm-service | complex / low confidence |
| 19 | llm-service | orchestrator | analysis · no tool needed |
| 20 | orchestrator | reminder-service | reminder intent |
| 21 | task-service | output-service | task result |
| 22 | output-service | raspberry-pi | HTTP Response · normal reply |
| 23 | orchestrator | output-service | clarification · no mutation |
| 24 | llm-service | mcp-client | selected structured tool call |
| 25 | mcp-client | orchestrator | tool call / result · validate and route |

## Notes & index panel

### Primary request path

- External Pi capture → HTTP voice upload → server STT / intent. LLM Service selects MCP tools when needed.
- MCP Client → Orchestrator: validate AI / MCP results, then route Application Services. Clarification means no data mutation.
- TaskService → TaskPort → EurekaTaskAdapter. OutputService creates TTS text / LCD content; Pi performs speech synthesis and hardware output.
### Deadline policy

- SchedulerAdapter periodically triggers ReminderService.
- Completed tasks are excluded from reminder evaluation.
- When no custom time exists, the default reminder is one hour before dueAt.
- Deduplicated alert: OutputService → OutputPublisherPort → WebSocketAdapter → Pi. Persistence remains undecided.
### Port / adapter boundary

- TaskPort contains create_task, get_tasks, get_task_status, complete_task, and update_task.
- EurekaTaskAdapter implements TaskPort. Runtime arrows lead to Eureka Process API, the task source of truth.
- Normal response: OutputService → Raspberry Pi via HTTP. Server push: OutputPublisherPort → WebSocketAdapter.

## Interface Contracts (embedded `desk-pet-interface-contracts` data)

### reminderPolicy

```json
[
  "Polling 사용 · SchedulerAdapter는 주기 실행만 담당",
  "completed 업무 제외",
  "사용자 지정 reminder가 있으면 해당 값 사용",
  "없으면 기본 dueAt 1시간 전",
  "중복 알림 방지 후 필요 시 OutputService 호출 · 저장 구현체는 미확정",
  "SchedulerAdapter → ReminderService → OutputService → OutputPublisherPort → WebSocketAdapter → Raspberry Pi로 proactive_alert 전달"
]
```

### TaskPort note

- TaskPort methods: create_task() · get_tasks() · get_task_status() · complete_task() · update_task()

### contracts

```json
[
  {
    "group": "Hardware",
    "interface": "IButtonInput",
    "endpoint": "Internal interface · Raspberry Pi GPIO",
    "callMethod": "Raspberry Pi GPIO Event 또는 Interrupt",
    "processing": "asynchronous event-driven",
    "input": [
      "buttonState: pressed | released"
    ],
    "output": [
      "event: BUTTON_PRESSED | BUTTON_RELEASED"
    ],
    "timeout": "maxRecordingTimeSec: 15 · 초과 시 녹음 강제 종료",
    "error": [
      "debouncing: 반복 입력 디바운싱",
      "deviceError: 입력 장치 오류 로그"
    ]
  },
  {
    "group": "Hardware",
    "interface": "IAudioCapture",
    "endpoint": "Internal interface · Raspberry Pi audio device",
    "callMethod": "Raspberry Pi Internal Audio Device API",
    "processing": "asynchronous",
    "input": [
      "event: START_RECORDING | STOP_RECORDING"
    ],
    "output": [
      "requestId: string",
      "audioFormat: WAV | PCM",
      "audioData: binary",
      "durationMs: number",
      "sampleRate: number"
    ],
    "timeout": "maxRecordingTimeSec: 15 · 초과 시 현재까지의 음성 처리",
    "error": [
      "microphoneConnectionFailure: MIC_ERROR",
      "audioGenerationFailure: AUDIO_CAPTURE_ERROR"
    ]
  },
  {
    "group": "Hardware",
    "interface": "IHardwareOutput",
    "endpoint": "Internal interface · device control",
    "callMethod": "Raspberry Pi: Edge-TTS synthesis → audio playback / LED / LCD control",
    "processing": "asynchronous",
    "input": [
      "ttsText: string | null",
      "ledState: IDLE | LISTENING | SPEAKING | ERROR",
      "lcdExpression: string | null",
      "lcdText: string | null"
    ],
    "output": [
      "audioPlayed: boolean",
      "ledUpdated: boolean",
      "lcdUpdated: boolean"
    ],
    "timeout": "비정상 지속 시 출력 종료 후 IDLE 복귀",
    "error": [
      "partialDeviceFailure: 가능한 출력 채널로 최소 피드백 제공"
    ],
    "note": "LED states: IDLE=green/solid · LISTENING=yellow/blink · SPEAKING=blue/blink"
  },
  {
    "group": "STT",
    "interface": "SpeechToText",
    "endpoint": "Internal service interface",
    "callMethod": "Audio Capture → Backend STT service (server-side)",
    "processing": "asynchronous",
    "input": [
      "requestId: string",
      "audioData: binary",
      "audioFormat: WAV | PCM",
      "language: ko-KR"
    ],
    "output": [
      "requestId: string",
      "text: string",
      "status: SUCCESS | NO_SPEECH | STT_ERROR"
    ],
    "timeout": "targetMs: 1000 · maxMs: 5000 · 초과 시 요청 중단",
    "error": [
      "NO_SPEECH 반환",
      "불명확한 음성은 재입력 요청",
      "STT_ERROR 반환 및 로그 기록"
    ]
  },
  {
    "group": "Lightweight Intent Model",
    "interface": "ClassifyIntent",
    "endpoint": "Internal function / lightweight model",
    "callMethod": "Backend Internal Function 또는 Lightweight Model",
    "processing": "synchronous",
    "input": [
      "requestId: string",
      "text: string"
    ],
    "output": [
      "requestId: string",
      "intent: string",
      "confidence: number",
      "parameters: object | null",
      "requiresLLM: boolean"
    ],
    "timeout": "targetMs: 500 · maxMs: 1000",
    "error": [
      "lowConfidence: LLM 분석으로 전달",
      "missingInformation: 임의 실행 없이 재질문 단계로 전달"
    ],
    "note": "Intent classification, confidence, simple parameter extraction, requiresLLM decision only. Actual execution and response generation belong to Application Services."
  },
  {
    "group": "Lightweight Intent Model",
    "interface": "ParseSimpleRequest",
    "endpoint": "Internal lightweight parsing interface",
    "callMethod": "Lightweight Intent Model → Intent / LLM Orchestrator (or LLM Service when requiresLLM)",
    "processing": "synchronous",
    "input": [
      "requestId: string",
      "text: string"
    ],
    "output": [
      "requestId: string",
      "intent: string",
      "confidence: number",
      "parameters: object | null",
      "requiresLLM: boolean"
    ],
    "timeout": "maxMs: 1000",
    "error": [
      "lowConfidence → LLM Service",
      "missingInformation → Orchestrator clarification"
    ],
    "note": "Classification and simple parameter extraction only. No task execution, response generation, or hardware command generation."
  },
  {
    "group": "Intent / LLM",
    "interface": "AnalyzeUserRequest",
    "endpoint": "LLM analysis interface",
    "callMethod": "Lightweight Intent Model → LLM Service → Intent / LLM Orchestrator; selected tool → MCP Client → Intent / LLM Orchestrator",
    "processing": "asynchronous",
    "input": [
      "requestId: string",
      "text: string",
      "currentDateTime: ISO-8601 datetime",
      "conversationContext: object | null"
    ],
    "output": [
      "intent: string",
      "parameters: taskName / dueAt / priority / reminderAt",
      "requiredTool: string | null",
      "requiresClarification: boolean",
      "clarificationQuestion: string | null"
    ],
    "timeout": "maxMs: 5000 · 초과 시 처리 실패 안내",
    "error": [
      "missingRequiredInformation: 추가 질문",
      "ambiguousDateOrTask: 확인 요청",
      "llmFailure: 처리 실패 상태 반환"
    ],
    "note": "Complex request interpretation; taskName / dueAt / priority / reminderAt extraction; ambiguity and missing-information detection; clarification generation; MCP tool selection. No direct data mutation or Eureka API access. reminderAt 미지정 시 null; 기본 정책은 ReminderService 적용."
  },
  {
    "group": "Intent / LLM",
    "interface": "CallMCPTool",
    "endpoint": "MCP tool interface",
    "callMethod": "LLM Service → MCP Client → Intent / LLM Orchestrator → Application Service (task operations)",
    "processing": "asynchronous",
    "input": [
      "requestId: string",
      "toolName: string",
      "operation: string",
      "parameters: object"
    ],
    "output": [
      "requestId: string",
      "success: boolean",
      "result: object | null",
      "error: code / message"
    ],
    "timeout": "minMs: 3000 · maxMs: 5000",
    "error": [
      "toolFailure: 오류 코드와 원인 반환",
      "permissionDenied: 권한 없는 기능 실행 금지",
      "duplicateRequest: requestId로 중복 방지"
    ],
    "note": "MCP is in use. LLM proposes/selects a tool; MCP Client handles the structured invocation interface and external analysis/read-tool access. Task-mutating tools are delegated through Orchestrator validation and TaskService / ReminderService, then TaskPort → EurekaTaskAdapter → Eureka Process API. MCP results are validated by the Orchestrator. Clarification required: no data mutation. No direct LLM/MCP → Eureka mutation."
  },
  {
    "group": "Desk Pet Backend API",
    "interface": "CreateTaskAPI",
    "endpoint": "POST /api/tasks",
    "callMethod": "Desk Pet Backend API → TaskService → TaskPort",
    "processing": "asynchronous",
    "input": [
      "requestId",
      "taskName",
      "dueAt: ISO-8601 datetime",
      "priority",
      "reminderAt: ISO-8601 datetime | null"
    ],
    "output": [
      "success",
      "itemId: string | null",
      "status: string | null",
      "message"
    ],
    "timeout": "backendTargetMs: 500 · 외부 요청은 시스템 응답시간 정책 적용",
    "error": [
      "missingRequiredInformation: 추가 입력 요청",
      "eurekaFailure: 등록되지 않은 업무를 성공 처리하지 않음"
    ]
  },
  {
    "group": "Desk Pet Backend API",
    "interface": "GetTasksAPI",
    "endpoint": "GET /api/tasks",
    "callMethod": "Desk Pet Backend API → TaskService → TaskPort",
    "processing": "asynchronous",
    "input": [
      "requestId",
      "filter: status / dueBefore / priority"
    ],
    "output": [
      "success",
      "tasks: itemId / taskName / dueAt / priority / status(todo|doing|done|hold)"
    ],
    "timeout": "maxMs: 5000",
    "error": [
      "queryFailure: 임의 업무를 생성하지 않고 조회 실패 반환"
    ]
  },
  {
    "group": "Desk Pet Backend API",
    "interface": "CompleteTaskAPI",
    "endpoint": "POST /api/tasks/{itemId}/complete",
    "callMethod": "Desk Pet Backend API → TaskService → TaskPort",
    "processing": "asynchronous",
    "input": [
      "itemId: string"
    ],
    "output": [
      "success",
      "itemId: string",
      "status: done | null"
    ],
    "timeout": "maxMs: 5000",
    "error": [
      "taskNotFound: 완료 처리하지 않음",
      "ambiguousTask: 대상 업무 확인 요청"
    ]
  },
  {
    "group": "Desk Pet Backend API",
    "interface": "GetTaskStatusAPI",
    "endpoint": "GET /api/tasks/{itemId}/status",
    "callMethod": "Desk Pet Backend API → TaskService → TaskPort",
    "processing": "asynchronous",
    "input": [
      "itemId: string"
    ],
    "output": [
      "success",
      "itemId: string",
      "status: todo | doing | done | hold",
      "totalStages",
      "completedStages"
    ],
    "timeout": "maxMs: 5000",
    "error": [
      "statusQueryFailure: 기존 값을 추정하지 않고 실패 반환"
    ]
  },
  {
    "group": "Desk Pet Backend API",
    "interface": "UpdateTaskAPI",
    "endpoint": "PUT /api/tasks/{itemId}",
    "callMethod": "Desk Pet Backend API → TaskService → TaskPort",
    "processing": "asynchronous",
    "input": [
      "itemId",
      "taskName: string | null",
      "dueAt: ISO-8601 datetime | null",
      "priority: string | null"
    ],
    "output": [
      "success",
      "task: itemId / taskName / dueAt / priority"
    ],
    "timeout": "maxMs: 5000",
    "error": [
      "taskNotFound: 변경하지 않음",
      "invalidInput: 변경하지 않고 입력 오류 반환"
    ]
  },
  {
    "group": "Eureka Process API",
    "interface": "CreateTask",
    "endpoint": "POST /_api_/items/0/start",
    "callMethod": "EurekaTaskAdapter → Eureka Process HTTP API",
    "processing": "asynchronous",
    "input": [
      "processId",
      "taskName",
      "dueAt: ISO-8601 datetime",
      "priority"
    ],
    "output": [
      "itemId",
      "stageIds: string[]",
      "task: object"
    ],
    "timeout": "targetMs: 1000 · maxMs: 2000",
    "error": [
      "requestFailure: HTTP 상태 코드와 응답 본문 확인",
      "missingRequiredValue: 등록 중단"
    ]
  },
  {
    "group": "Eureka Process API",
    "interface": "GetTasks",
    "endpoint": "GET /_api_/items/0/list?sites&detail=true",
    "callMethod": "EurekaTaskAdapter → Eureka Process HTTP API",
    "processing": "asynchronous",
    "input": [
      "workspace",
      "options"
    ],
    "output": [
      "items: object[]",
      "stages: object[]"
    ],
    "timeout": "targetMs: 1000 · maxMs: 2000",
    "error": [
      "apiFailure: Backend에 조회 실패 반환"
    ]
  },
  {
    "group": "Eureka Process API",
    "interface": "CompleteTask",
    "endpoint": "POST /_api_/items/{itemId}/complete",
    "callMethod": "EurekaTaskAdapter → Eureka Process HTTP API",
    "processing": "asynchronous",
    "input": [
      "itemId: string"
    ],
    "output": [
      "itemId",
      "status",
      "stages: object[]"
    ],
    "timeout": "targetMs: 1000 · maxMs: 2000",
    "error": [
      "invalidState: hold 등 완료 불가 상태 오류",
      "taskNotFound: 존재하지 않는 업무 ID 오류"
    ]
  },
  {
    "group": "Eureka Process API",
    "interface": "GetTaskStatus",
    "endpoint": "GET /_api_/items/{itemId}/status",
    "callMethod": "EurekaTaskAdapter → Eureka Process HTTP API",
    "processing": "asynchronous",
    "input": [
      "itemId: string"
    ],
    "output": [
      "itemId",
      "status: todo | doing | done | hold",
      "total",
      "done"
    ],
    "timeout": "targetMs: 1000 · maxMs: 2000",
    "error": [
      "taskNotFound: 상태 조회 실패",
      "apiFailure: 상태 조회 실패"
    ]
  },
  {
    "group": "Eureka Process API",
    "interface": "UpdateTask",
    "endpoint": "PUT /_api_/items/{itemId}",
    "callMethod": "EurekaTaskAdapter → Eureka Process HTTP API",
    "processing": "asynchronous",
    "input": [
      "itemId",
      "dueAt: ISO-8601 datetime | null",
      "taskName: string | null",
      "priority: string | null"
    ],
    "output": [
      "success",
      "task: object"
    ],
    "timeout": "targetMs: 1000 · maxMs: 2000",
    "error": [
      "invalidValue: 변경하지 않음",
      "taskNotFound: 존재하지 않는 업무 ID 오류"
    ]
  },
  {
    "group": "Output / WebSocket",
    "interface": "GenerateOutput",
    "endpoint": "Internal application interface",
    "callMethod": "OutputService internal operation",
    "processing": "synchronous",
    "input": [
      "responseType",
      "taskName: string | null",
      "result: object",
      "remainingTime: number | null"
    ],
    "output": [
      "type: string",
      "tts: string | null",
      "lcd.expression: string | null",
      "lcd.text: string | null"
    ],
    "timeout": "targetMs: 200 · maxMs: 500",
    "error": [
      "partialGenerationFailure: 사용 가능한 채널로 최소 피드백 제공"
    ],
    "note": "TTS용 문장만 생성. 음성 합성 / LED 제어는 Raspberry Pi 책임. 일반 응답은 HTTP로 Pi에 반환; server-initiated output / proactive_alert만 OutputPublisherPort 사용."
  },
  {
    "group": "Output / WebSocket",
    "interface": "SendOutputCommand",
    "endpoint": "Server-initiated WebSocket push only",
    "callMethod": "OutputService → OutputPublisherPort → WebSocketAdapter → Raspberry Pi",
    "processing": "asynchronous",
    "input": [
      "event: output | proactive_alert",
      "type: string",
      "user_id: string",
      "request_id: string",
      "output.tts: string | null",
      "output.lcd.expression: string | null",
      "output.lcd.text: string | null"
    ],
    "output": [
      "WebSocket send success / failure (not device playback confirmation)"
    ],
    "timeout": "WebSocket send failure is a delivery failure; no device playback ACK is assumed",
    "error": [
      "networkFailure: 전달 실패 기록",
      "proactiveAlertFailure: 미전달 상태 기록"
    ],
    "note": "일반 음성 요청의 HTTP Response는 이 계약을 거치지 않음."
  },
  {
    "group": "Output / WebSocket",
    "interface": "proactive_alert WebSocket event",
    "endpoint": "/ws/hardware/{user_id}",
    "callMethod": "OutputService → OutputPublisherPort → WebSocketAdapter → Raspberry Pi",
    "processing": "asynchronous",
    "input": [
      "event: proactive_alert"
    ],
    "output": [
      "event: proactive_alert",
      "type: deadline_approaching",
      "user_id: string",
      "request_id: string",
      "output.tts: string",
      "output.lcd.expression: alert",
      "output.lcd.text: string"
    ],
    "error": [
      "전달 실패 시 미전달 알림으로 기록 후 재처리 정책 적용"
    ]
  }
]
```
