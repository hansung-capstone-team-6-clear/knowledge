---
title: "💻 04. Backend"
source: "Notion export (팀 워크스페이스) — Capstone Eureka Process/💻 04 Backend"
author: "클리어픽(6팀)"
created: 2026-10-08
published:
description: "Eureka Process API PoC 결과와 제약·Mock Server 테스트·Backend/Eureka/Output Controller 인터페이스·데이터 흐름. Notion HTML export를 Markdown으로 변환 — 개인 이메일 제거, 이미지·첨부 파일 미포함."
tags:
  - notion-export
  - capstone
  - desk-pet
---

# 💻 04. Backend

deskpet_mock_server.zip · Eureka_Process_API_Test.zip

## 1. Eureka Process PoC

### 1-1. 테스트 목적

Eureka Process API를 실제 운영 환경에서 호출하여 다음 항목을 확인하였다.
- API Key 기반 인증 정상 여부

- Process 및 업무 데이터 조회 가능 여부

- 업무 등록·조회·수정·완료·삭제 기능 동작 여부

- API 응답 속도

- 데스크펫 백엔드 연동 시 고려해야 할 제약사항

### 1-2. 테스트 환경

| 항목 | 내용 |
| --- | --- |
| 테스트 환경 | Windows PC |
| 개발 언어 | Python |
| HTTP Client | requests |
| 환경변수 처리 | python-dotenv |
| Eureka 환경 | 운영 환경 |
| Base URL | `https://api.eureka.codes/flw-v1` |
| Process ID | `general-work-v1@10204` |

### 1-3. 테스트 절차

Poc는 다음 순서로 진행하였다.

`API Key 확인` → `Process 확인` → `업무 등록` → `업무 목록 조회` → `업무 완료` → `상태 확인` → `마감 연기` → `업무 삭제`

### 1-4. 테스트 결과

| 테스트 종류 | 소요 시간 | 성고 여부 |
| --- | --- | --- |
| session (API Key 인증 확인) | 0.191 / 0.468 sec | 성공 |
| Process 목록 | 0.301 ~ 1.135 sec | 성공 |
| 업무 등록 | 0.672 sec | 성공 |
| 업무 조회 | 0.279 sec | 성공 |
| 업무 완료 | 0.230 sec | 성공 |
| 상태 조회 | 0.183 sec | 성공 |
| 마감 연기 | 0.126 sec | 성공 |
| 업무 삭제 | 0.404 sec | 성공 |

전체 테스트에서 API 호출 자체는 정상적으로 동작하였다.

대부분의 요청은 약 **0.1~0.7초 내외**로 응답하였으며, Process 목록 조회에서 최대 약 **1.1초**의 응답 시간이 확인되었다.

### 1-5. PoC 과정에서 확인한 Eureka Process 제약사항

#### (1) 마감 기한 범위 검색 미지원

Eureka API에서는 다음과 같은 조건으로 업무를 직접 조회하는 기능이 제공되지 않는다.
- 1시간 이내 마감 업무

- 오늘 마감 업무

- 특정 시간 범위 내 마감 업무

따라서 Backend에서 업무 목록을 조회한 후 각 업무의 `dueAt` 값을 기준으로 직접 필터링해야 한다.

#### (2) Push / Webhook 미지원

Eureka Process에서는 업무 상태 변경이나 마감 시점 도달 시 Backend로 자동 이벤트를 전달하는 Push 또는 Webhook 기능을 제공하지 않는다.

따라서 데스크펫 Backend가 일정 주기로 Eureka API를 호출하는 **Polling 방식**이 필요하다.

#### (3) 업무 상태 관리 방식

업무 완료 여부는 Item 자체의 단순 상태값보다 **하위 Stage 상태를 기반으로 관리되는 구조**를 가진다.

→ 개별 업무의 정확한 상태 확인은 `GET /_api_/items/{itemId}/status `API를 사용할 수 있다.

단, Polling 과정에서 여러 업무의 상태를 반복적으로 확인할 경우 각 업무마다 Status API를 호출하면 요청 횟수가 크게 증가한다.

→ 따라서 대량 조회 상황에서는 업무 목록 데이터에 포함된 `stage$$` 값을 이용하여 상태를 판단하고, 필요한 경우에만 개별 Status API를 호출하는 방식이 적합하다.

#### (4) 오류 응답 처리

입력 데이터 검증 오류가 항상 `400 Bad Request`로 반환되지는 않았다.
일부 잘못된 요청의 경우 서버에서 `500` 응답이 반환될 수 있다.

따라서 Backend에서는 HTTP Status Code만으로 요청 성공 여부를 판단하지 않고 다음 정보를 함께 확인해야 한다.
- HTTP Status Code

- Response Body

- Eureka에서 반환한 오류 메시지

→ `Status Code 확인 + Response Body 검증`방식으로 오류를 처리해야 한다.

## 2. Mock Server 테스트

### 2-1. 테스트 목적

실제 LLM 및 데스크펫 하드웨어가 완전히 구현되기 전에 **전체 시스템의 데이터 흐름과 인터페이스를 선행 검증하기 위한 목적으로** Mock Server를 구성하였다.

Mock 환경에서는 실제 Eureka Process, LLM, TTS 및 하드웨어 동작 일부를 단순화하여 다음 기능의 정상 동작 여부를 확인하였다.
- 업무 데이터 등록 및 조회

- 업무 상태 변경

- 마감 시점 감지

- 능동 알림 이벤트 생성

- WebSocket을 통한 `proactive_alert` 전송

- 데스크펫에서 사용할 LED / LCD / TTS 출력 데이터 구조 확인

### 2-2. 테스트 환경

| 항목 | 내용 |
| --- | --- |
| 테스트 환경 | Windows PC |
| 개발 언어 | Python |
| Backend 형태 | Mock Server |
| 통신 방식 | HTTP REST API / WebSocket |
| 업무 데이터 | Mock Data |
| 알림 이벤트 | `proactive_alert` |
| 실제 하드웨어 및 LLM | 미연동 (print() 함수로 임시로 구현해놓았다.) |

### 2-3. Mock Server 구현 기능
- **업무 등록 :** `POST /api/tasks`

- **업무 목록 조회 : **`GET /api/tasks`

- **업무 완료 처리 : **`POST /api/tasks/{id}/complete`

- **업무 상태 조회 : **`GET /api/tasks/{id}/status`

- **업무 정보 수정 : **`PUT /api/tasks/{id}`

### 2-4. 마감 알림 테스트

Mock Server에서는 등록된 업무의 마감 기한을 기준으로 알림 조건을 확인하도록 구성하였다.
```
업무 등록
→ Backend 업무 확인
→ 현재 시간과 마감 시간 비교
→ 알림 임계치 진입
→ proactive_alert 이벤트 생성
→ WebSocket Push
```

기본 시나리오에서는 업무 마감 **1시간 전**을 알림 조건으로 설정하여 테스트하였다.

알림 조건에 도달한 업무가 발견되면 Backend에서 다음과 같은 이벤트를 생성하도록 구성하였다.
```
{
  "event": "proactive_alert",
  "type": "deadline_approaching",
  "user_id": "team-park-sihyeon",
  "output": {
    "tts": "시현 님, 보고서 검토 마감이 1시간 남았어요!",
    "led": {
      "color": "yellow",
      "mode": "blink"
    },
    "lcd": {
      "expression": "alert",
      "text": "마감 1시간 전"
    }
  }
}
```

Mock 단계에서는 `tts` 문장을 임시로 생성하여 사용하였다.

최종 시스템에서는 해당 문장을 **LLM이 상황 및 사용자 맥락을 기반으로 생성하도록 연결할 예정**이다.

### 2-5. WebSocket 테스트

Backend에서 생성된 `proactive_alert` 이벤트를 WebSocket을 통해 클라이언트에 전달하는 테스트를 수행하였다.

테스트 결과 다음 흐름이 정상적으로 동작하는 것을 확인하였다.
```
Backend
→ WebSocket Connection
→ proactive_alert Push
→ Client Event 수신
```

이를 통해 사용자의 별도 요청 없이 Backend에서 데스크펫으로 능동적인 알림 이벤트를 전달할 수 있음을 확인하였다.

### 2-6. 테스트 결과

Mock 테스트를 통해 다음 기능의 정상 동작을 확인하였다.

| 테스트 항목 | 결과 |
| --- | --- |
| 업무 등록 | 성공 |
| 업무 목록 조회 | 성공 |
| 업무 완료 처리 | 성공 |
| 업무 상태 조회 | 성공 |
| 업무 마감 수정 | 성공 |
| 마감 임박 업무 감지 | 성공 |
| 완료 업무 알림 제외 | 성공 |
| `proactive_alert` 이벤트 생성 | 성공 |
| WebSocket 이벤트 전송 | 성공 |
| 클라이언트 이벤트 수신 | 성공 |

### 2-7. 실제 Eureka 연동 시 변경되는 부분

Mock Server에서는 자체 업무 데이터를 사용하였지만 실제 구현에서는 해당 업무 데이터를 Eureka Process에서 조회하도록 변경된다.

**Mock 구조 :**
```
Mock 업무 데이터
→ Backend
→ 마감 조건 판단
→ WebSocket
→ Desk Pet
```

**실제 구조 :**
```
Eureka Process
→ Backend Polling
→ 업무 목록 및 상태 확인
→ 마감 조건 판단
→ WebSocket
→ Desk Pet
```

따라서 Mock 테스트에서 검증한 Backend 이후의 알림 흐름은 실제 Eureka 연동 환경에서도 동일하게 활용할 수 있다.

### 5. Backend API

**역할 및 책임**

Raspberry Pi, STT, AI 모델 및 Eureka Process 사이의 요청·응답 흐름을 관리한다.
- Raspberry Pi 요청 수신

- STT / Intent / LLM 처리 흐름 관리

- Eureka Process API 호출

- Scheduler / Polling 실행

- 마감 임박 이벤트 생성

- 사용자 지정 알림 시점 관리

- 알림 전달 상태 관리

- Timeout 및 오류 처리

- API Key 및 환경변수 관리

- Latency 측정 및 로그 기록

#### (1) 업무 등록 인터페이스 : CreateTaskAPI
```
{
  "interface": "CreateTaskAPI",
  "endpoint": "POST /api/tasks",
  "description": "새로운 업무를 등록한다.",
  "input": {
    "requestId": "string",
    "taskName": "string",
    "dueAt": "ISO-8601 datetime",
    "priority": "string",
    "reminderAt": "ISO-8601 datetime | null"
  },
  "output": {
    "success": "boolean",
    "itemId": "string | null",
    "status": "string | null",
    "message": "string"
  },
  "callMethod": "Raspberry Pi → Backend HTTPS",
  "processing": "asynchronous",
  "errorHandling": {
    "missingRequiredInformation": "사용자에게 추가 입력 요청",
    "eurekaFailure": "실제 등록되지 않은 업무를 성공으로 안내하지 않음"
  },
  "timeout": {
    "backendTargetMs": 500,
    "externalRequest": "전체 시스템 응답시간 정책 적용"
  }
}
```

#### (2) 업무 조회 인터페이스 : GetTasksAPI
```
{
  "interface": "GetTasksAPI",
  "endpoint": "GET /api/tasks",
  "description": "등록된 업무 목록을 조회한다.",
  "input": {
    "requestId": "string",
    "filter": {
      "status": "string | null",
      "dueBefore": "ISO-8601 datetime | null",
      "priority": "string | null"
    }
  },
  "output": {
    "success": "boolean",
    "tasks": [
      {
        "itemId": "string",
        "taskName": "string",
        "dueAt": "ISO-8601 datetime",
        "priority": "string",
        "status": "todo | doing | done | hold"
      }
    ]
  },
  "callMethod": "Raspberry Pi → Backend HTTPS",
  "processing": "asynchronous",
  "errorHandling": {
    "queryFailure": "임의의 업무 데이터를 생성하지 않고 조회 실패 상태 반환"
  },
  "timeout": {
    "maxMs": 5000
  }
}
```

#### (3) 업무 완료 인터페이스 : CompleteTaskAPI
```
{
  "interface": "CompleteTaskAPI",
  "endpoint": "POST /api/tasks/{itemId}/complete",
  "description": "지정된 업무를 완료 상태로 변경한다.",
  "input": {
    "itemId": "string"
  },
  "output": {
    "success": "boolean",
    "itemId": "string",
    "status": "done | null"
  },
  "callMethod": "Raspberry Pi → Backend → Eureka Process",
  "processing": "asynchronous",
  "errorHandling": {
    "taskNotFound": "완료 처리하지 않고 오류 반환",
    "ambiguousTask": "사용자에게 대상 업무 확인 요청"
  },
  "timeout": {
    "maxMs": 5000
  }
}
```

#### (4) 업무 상태 확인 인터페이스 : GetTaskStatusAPI
```
{
  "interface": "GetTaskStatusAPI",
  "endpoint": "GET /api/tasks/{itemId}/status",
  "description": "업무의 현재 진행 상태를 확인한다.",
  "input": {
    "itemId": "string"
  },
  "output": {
    "success": "boolean",
    "itemId": "string",
    "status": "todo | doing | done | hold",
    "totalStages": "number",
    "completedStages": "number"
  },
  "callMethod": "Raspberry Pi → Backend → Eureka Process",
  "processing": "asynchronous",
  "errorHandling": {
    "statusQueryFailure": "기존 값을 추정하지 않고 조회 실패 상태 반환"
  },
  "timeout": {
    "maxMs": 5000
  }
}
```

#### (5) 업무 수정 인터페이스 : UpdateTaskAPI
```
{
  "interface": "UpdateTaskAPI",
  "endpoint": "PUT /api/tasks/{itemId}",
  "description": "업무명, 마감 기한 또는 우선순위를 수정한다.",
  "input": {
    "itemId": "string",
    "taskName": "string | null",
    "dueAt": "ISO-8601 datetime | null",
    "priority": "string | null"
  },
  "output": {
    "success": "boolean",
    "task": {
      "itemId": "string",
      "taskName": "string",
      "dueAt": "ISO-8601 datetime",
      "priority": "string"
    }
  },
  "callMethod": "Raspberry Pi → Backend → Eureka Process",
  "processing": "asynchronous",
  "errorHandling": {
    "taskNotFound": "업무를 변경하지 않고 오류 반환",
    "invalidInput": "변경하지 않고 입력 오류 상태 반환"
  },
  "timeout": {
    "maxMs": 5000
  }
}
```

---

### 6. Eureka Process

**역할 및 책임**

업무의 기준 정보를 저장하고 관리한다.
- 업무 등록

- 업무 목록 조회

- 업무 완료 처리

- 업무 진행 상태 관리

- 마감 기한 변경

- 업무명 및 우선순위 변경

- Item ID 및 Stage ID 관리

#### (1) 업무 등록 인터페이스 : CreateTask
```
{
  "interface": "CreateTask",
  "endpoint": "POST /_api_/items/0/start",
  "input": {
    "processId": "string",
    "taskName": "string",
    "dueAt": "ISO-8601 datetime",
    "priority": "string"
  },
  "output": {
    "itemId": "string",
    "stageIds": ["string"],
    "task": "object"
  },
  "callMethod": "Backend → Eureka Process HTTP API",
  "processing": "asynchronous",
  "errorHandling": {
    "requestFailure": "HTTP 상태 코드 및 응답 본문 확인",
    "missingRequiredValue": "업무 등록 중단"
  },
  "timeout": {
    "targetMs": 1000,
    "maxMs": 2000
  }
}
```

#### (2) 업무 조회 인터페이스 : GetTasks
```
{
  "interface": "GetTasks",
  "endpoint": "GET /_api_/items/0/list?sites&detail=true",
  "input": {
    "workspace": "string",
    "options": "object"
  },
  "output": {
    "items": ["object"],
    "stages": ["object"]
  },
  "callMethod": "Backend → Eureka Process HTTP API",
  "processing": "asynchronous",
  "errorHandling": {
    "apiFailure": "Backend에 조회 실패 상태 반환"
  },
  "timeout": {
    "targetMs": 1000,
    "maxMs": 2000
  }
}
```

#### (3) 업무 완료 인터페이스 : CompleteTask
```
{
  "interface": "CompleteTask",
  "endpoint": "POST /_api_/items/{itemId}/complete",
  "input": {
    "itemId": "string"
  },
  "output": {
    "itemId": "string",
    "status": "string",
    "stages": ["object"]
  },
  "callMethod": "Backend → Eureka Process HTTP API",
  "processing": "asynchronous",
  "errorHandling": {
    "invalidState": "hold 등 완료할 수 없는 상태인 경우 오류 반환",
    "taskNotFound": "존재하지 않는 업무 ID 오류 반환"
  },
  "timeout": {
    "targetMs": 1000,
    "maxMs": 2000
  }
}
```

#### (4) 업무 상태 확인 인터페이스 : GetTaskStatus
```
{
  "interface": "GetTaskStatus",
  "endpoint": "GET /_api_/items/{itemId}/status",
  "input": {
    "itemId": "string"
  },
  "output": {
    "itemId": "string",
    "status": "todo | doing | done | hold",
    "totalStages": "number",
    "completedStages": "number"
  },
  "callMethod": "Backend → Eureka Process HTTP API",
  "processing": "asynchronous",
  "errorHandling": {
    "taskNotFound": "상태 조회 실패 반환",
    "apiFailure": "상태 조회 실패 반환"
  },
  "timeout": {
    "targetMs": 1000,
    "maxMs": 2000
  }
}
```

#### (5) 업무 수정 및 연기 인터페이스 : UpdateTask
```
{
  "interface": "UpdateTask",
  "endpoint": "PUT /_api_/items/{itemId}",
  "input": {
    "itemId": "string",
    "dueAt": "ISO-8601 datetime | null",
    "taskName": "string | null",
    "priority": "string | null"
  },
  "output": {
    "success": "boolean",
    "task": "object"
  },
  "callMethod": "Backend → Eureka Process HTTP API",
  "processing": "asynchronous",
  "errorHandling": {
    "invalidValue": "변경하지 않고 오류 반환",
    "taskNotFound": "존재하지 않는 업무 ID 오류 반환"
  },
  "timeout": {
    "targetMs": 1000,
    "maxMs": 2000
  }
}
```

---

### 7. Output Controller

**역할 및 책임**

Backend의 처리 결과를 사용자에게 전달할 수 있는 음성 및 시각 출력 명령으로 변환한다.
- TTS 문장 생성

- LED 상태 결정

- LCD 표정 결정

- LCD 표시 텍스트 생성

- Hardware Layer로 출력 명령 전달

#### (1) 출력 생성 인터페이스 : GenerateOutput
```
{
  "interface": "GenerateOutput",
  "description": "Backend 처리 결과를 데스크펫 출력 형식으로 변환한다.",
  "input": {
    "responseType": "string",
    "taskName": "string | null",
    "result": "object",
    "remainingTime": "number | null"
  },
  "output": {
    "ttsText": "string",
    "ledState": "IDLE | LISTENING | SPEAKING | ERROR",
    "lcdExpressionId": "string",
    "lcdText": "string"
  },
  "callMethod": "Backend Internal Function",
  "processing": "synchronous",
  "errorHandling": {
    "partialGenerationFailure": "사용 가능한 출력 채널을 통해 최소한의 사용자 피드백 제공"
  },
  "timeout": {
    "targetMs": 200,
    "maxMs": 500
  }
}
```

#### (2) 하드웨어 전달 인터페이스 : SendOutputCommand
```
{
  "interface": "SendOutputCommand",
  "description": "생성된 출력 명령을 Raspberry Pi로 전달한다.",
  "input": {
    "requestId": "string",
    "ttsText": "string | null",
    "ledState": "IDLE | LISTENING | SPEAKING | ERROR",
    "lcdExpressionId": "string | null",
    "lcdText": "string | null"
  },
  "output": {
    "success": "boolean",
    "audioOutput": "boolean",
    "ledOutput": "boolean",
    "lcdOutput": "boolean"
  },
  "callMethod": "Backend → Raspberry Pi Network Request",
  "processing": "asynchronous",
  "errorHandling": {
    "networkFailure": "출력 전달 실패 상태 기록",
    "proactiveAlertFailure": "전달하지 못한 능동 알림을 미전달 상태로 기록",
    "normalResponseFailure": "사용자에게 네트워크 연결 오류 상태 표시"
  },
  "timeout": {
    "action": "일정 시간 동안 Raspberry Pi의 응답이 없으면 요청 종료 후 네트워크 오류로 처리"
  }
}
```

---

### 8. 전체 시스템 데이터 흐름
```
{
  "requestFlow": [
    {
      "step": 1,
      "from": "User",
      "to": "Hardware",
      "action": "Push-to-Talk 버튼 입력"
    },
    {
      "step": 2,
      "from": "Hardware",
      "to": "STT",
      "action": "음성 데이터 전달"
    },
    {
      "step": 3,
      "from": "STT",
      "to": "Lightweight Intent Model",
      "action": "인식된 텍스트 전달"
    },
    {
      "step": 4,
      "from": "Lightweight Intent Model",
      "to": "Backend",
      "action": "Intent 및 LLM 호출 필요 여부 전달"
    },
    {
      "step": 5,
      "condition": "requiresLLM == true",
      "from": "Backend",
      "to": "LLM",
      "action": "복합 요청 분석"
    },
    {
      "step": 6,
      "condition": "externalToolRequired == true",
      "from": "LLM",
      "to": "MCP Tool",
      "action": "필요한 외부 기능 호출"
    },
    {
      "step": 7,
      "condition": "taskOperation == true",
      "from": "Backend",
      "to": "Eureka Process",
      "action": "업무 등록, 조회, 완료, 수정 또는 상태 확인"
    },
    {
      "step": 8,
      "from": "Backend",
      "to": "Output Controller",
      "action": "처리 결과 전달"
    },
    {
      "step": 9,
      "from": "Output Controller",
      "to": "Hardware",
      "action": "TTS, LED, LCD 출력 명령 전달"
    },
    {
      "step": 10,
      "from": "Hardware",
      "to": "User",
      "action": "음성 및 시각적 결과 출력"
    }
  ]
}
```

### 9. 마감 알림 처리 흐름
```
{
  "deadlineReminderFlow": {
    "scheduler": {
      "method": "polling",
      "managedBy": "Backend"
    },
    "process": [
      "Backend가 주기적으로 Eureka Process의 미완료 업무를 조회한다.",
      "업무의 dueAt과 현재 시간을 비교한다.",
      "사용자 지정 알림 시간이 존재하면 해당 시간을 기준으로 알림 여부를 판단한다.",
      "사용자 지정 알림 시간이 없으면 기본값인 마감 1시간 전을 사용한다.",
      "이미 완료된 업무는 알림 대상에서 제외한다.",
      "알림 조건을 만족하면 proactive alert 이벤트를 생성한다.",
      "Output Controller가 TTS, LED, LCD 출력 데이터를 생성한다.",
      "Backend가 Raspberry Pi로 출력 명령을 전달한다.",
      "전달에 실패한 알림은 미전달 상태로 기록하여 네트워크 복구 후 재처리할 수 있도록 한다."
    ],
    "defaultReminder": {
      "beforeDueMinutes": 60
    }
  }
}
```
