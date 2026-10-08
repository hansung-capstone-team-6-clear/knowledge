---
title: "🤖 03. AI & 데이터"
source: "Notion export (팀 워크스페이스) — Capstone Eureka Process/🤖 03 AI & 데이터"
author: "클리어픽(6팀)"
created: 2026-10-08
published:
description: "경량 Intent 분류·LLM 복합 요청·Structured Output·지원 Intent·AI 모델 후보와 PoC·STT/Intent/LLM 인터페이스. Notion HTML export를 Markdown으로 변환 — 개인 이메일 제거, 이미지·첨부 파일 미포함."
tags:
  - notion-export
  - capstone
  - desk-pet
---

# 🤖 03. AI & 데이터

## 1. AI 처리 구조

### 1-1. 경량 Intent 분류

STT를 통해 변환된 사용자 텍스트를 경량 Intent 모델에 전달하여 요청 유형을 우선 분류한다.
- 업무 등록

- 업무 목록 조회

- 업무 완료

- 업무 상태 조회

- 마감 기한 변경 및 연기

- 알림 설정 및 변경

- 기타 / 불명확 요청

단순하고 명확한 요청은 Intent 분류 결과를 기반으로 Backend 기능으로 연결한다.

복합 요청이거나 문맥 해석이 필요하거나 Intent 신뢰도가 기준 이하인 경우 LLM 처리 단계로 전달한다.

**목표 처리시간: 500ms 이내**

### 1-2. LLM 기반 복합 요청 처리

LLM은 경량 Intent 모델만으로 처리하기 어려운 복합 요청과 추가적인 문맥 해석이 필요한 요청을 처리한다.

주요 역할은 다음과 같다.
- 복합 자연어 요청 해석

- 업무명, 날짜, 시간, 우선순위 등 파라미터 추출

- 여러 요청이 포함된 발화 분석

- 필수 정보 누락 여부 판단

- 필요한 외부 Tool 또는 기능 결정

- 불명확한 요청에 대한 재질문 생성

예:

`“9월 27일 오후 3시까지 보고서 검토해야 해. 마감 2시간 전에 알려줘.”`

↓
- 업무명: 보고서 검토

- 마감 기한: 2026-09-27 15:00

- 알림 시점: 120분 전

### 1-3. Structured Output

AI가 해석한 결과는 Backend가 직접 사용할 수 있도록 구조화된 형태로 반환한다.
```
{
"intent": "CREATE_TASK",
"need_clarification": false,
"parameters": {
"task_name": "보고서 검토",
"due_at": "2026-09-27T15:00:00",
"reminder_offset_minutes": 120
}
}
```

사용자가 알림 시간을 지정하지 않은 경우 프로젝트 정책에 따라 `reminder_offset_minutes`에 기본값 `60`을 적용한다.

필요한 정보가 부족하거나 대상 업무를 특정할 수 없는 경우 임의로 실행하지 않고 `need_clarification` 상태를 반환한다.

### 1-4. 재질문 및 예외처리

예를 들어,

`“보고서 알림 설정해줘.”`

와 같이 어떤 업무를 의미하는지 특정할 수 없는 요청은 바로 실행하지 않는다.
```
{
"intent": "SET_REMINDER",
"need_clarification": true,
"clarification_message": "어떤 보고서를 말씀하시는 건가요?"
}
```

이후 Backend가 후보 업무 정보를 제공하면 사용자에게 필요한 추가 질문을 수행하고, 대상이 확정된 이후에만 실제 데이터 변경을 요청한다.

## 2. 지원 Intent 및 데이터 정의

| Intent | 설명 | 예시 |
| --- | --- | --- |
| `CREATE_TASK` | 업무 등록 | “내일까지 보고서 작성 추가해줘.” |
| `GET_TASKS` | 업무 목록 조회 | “오늘 할 일 알려줘.” |
| `COMPLETE_TASK` | 업무 완료 | “보고서 완료했어.” |
| `GET_TASK_STATUS` | 업무 진행 상태 조회 | “보고서 지금 상태가 뭐야?” |
| `UPDATE_TASK` | 업무 마감 변경/연기 | “보고서 마감 두 시간 미뤄줘.” |
| `SET_REMINDER` | 마감 알림 설정 | “보고서 한 시간 전에 알려줘.” |
| `UPDATE_REMINDER` | 기존 알림 변경 | “두 시간 전 말고 30분 전에 알려줘.” |
| `UNKNOWN` | 지원하지 않거나 불명확한 요청 | 분류 불가능한 요청 |

## 3. AI 입출력 데이터

### 3-1. 정상 처리
```
{
"request_id": "req_20260930_001",
"stt_text": "9월 27일 오후 3시까지 보고서 검토해야 해. 마감 2시간 전에 알려줘.",
"current_time": "2026-09-25T10:00:00"
}
```

AI 처리 결과 :
```
{
"intent": "CREATE_TASK",
"need_clarification": false,
"parameters": {
"task_name": "보고서 검토",
"due_at": "2026-09-27T15:00:00",
"reminder_offset_minutes": 120
}
}
```

### 3-2. 불명확 요청
```
{
"intent": "SET_REMINDER",
"need_clarification": true,
"clarification_message": "어떤 보고서를 말씀하시는 건가요?",
"parameters": null
}
```

## 4. AI 모델 및 평가

| 영역 | 후보 |
| --- | --- |
| STT | Faster-Whisper / Whisper 계열 |
| 경량 Intent | 경량 텍스트 분류 모델 |
| LLM | 클라우드 LLM 또는 로컬 LLM 후보 |
| 로컬 LLM 검증 | Ollama 기반 모델 |
| Structured Output | Python / Pydantic |
| 평가 | 자체 테스트셋 기반 정확도 및 Latency 측정 |

### PoC 1. Intent 및 파라미터 추출 검증

다양한 사용자 발화를 입력하여 Intent 분류와 업무명, 마감일시, 알림 시점 등의 파라미터 추출 여부를 검증한다.

측정 항목:
- Intent 분류 정확도

- 업무명 추출 정확도

- 날짜/시간 추출 정확도

- 알림 기본값 적용 여부

### PoC 2. 불명확 요청 재질문 검증

대상 업무 또는 필수 정보가 부족한 발화를 입력하여 임의의 데이터 변경 없이 재질문 단계로 전환되는지 검증한다.

측정 항목:
- `need_clarification` 판단 정확도

- 잘못된 업무 등록/수정 발생 여부

- 추가 입력 이후 기존 요청과 연결 가능 여부

### PoC 3. AI 처리 Latency 측정

STT 결과가 생성된 이후 Intent 분류 및 필요한 경우 LLM 처리가 완료될 때까지의 시간을 측정한다.

목표:
- 경량 Intent 분류: **500ms 이내**

- LLM 처리: 실제 모델 선정 이후 목표값 확정

- 전체 음성 응답: 시스템 Latency 정책과 연계하여 측정

### 2.STT

**역할 및 책임**

사용자의 음성 데이터를 텍스트로 변환한다.
- 음성 데이터 수신

- 음성 데이터 유효성 검사

- 음성 → 텍스트 변환

- 요청 ID와 인식 결과 연결

- 무음 및 음성 인식 실패 처리

#### (1) 음성 인식 인터페이스 : SpeechToText
```
{
  "interface": "SpeechToText",
  "description": "사용자의 음성 데이터를 텍스트로 변환한다.",
  "input": {
    "requestId": "string",
    "audioData": "binary",
    "audioFormat": "WAV | PCM",
    "language": "ko-KR"
  },
  "output": {
    "requestId": "string",
    "text": "string",
    "status": "SUCCESS | NO_SPEECH | STT_ERROR"
  },
  "callMethod": "Raspberry Pi에서 STT 처리 모듈 또는 Backend STT 서비스 호출",
  "processing": "asynchronous",
  "errorHandling": {
    "noSpeech": "NO_SPEECH 반환",
    "unclearSpeech": "사용자에게 재입력 요청",
    "serviceError": "STT_ERROR 반환 및 오류 로그 기록"
  },
  "timeout": {
    "targetMs": 1000,
    "maxMs": 5000,
    "action": "최대 처리 시간을 초과하면 요청 중단"
  }
}
```

### 3. Lightweight Intent Model

**역할 및 책임**

STT로 변환된 사용자 텍스트의 의도를 빠르게 분류하고 LLM 호출 여부를 결정한다.
- 사용자 Intent 분류

- 단순 명령 / 복합 명령 구분

- 기본적인 명령 처리

- LLM 호출 필요 여부 판단

- 불명확한 요청 감지

#### (1) Intent 분류 인터페이스 : ClassifyIntent
```
{
  "interface": "ClassifyIntent",
  "description": "사용자 요청의 Intent를 분류하고 LLM 호출 필요 여부를 판단한다.",
  "input": {
    "requestId": "string",
    "text": "string"
  },
  "output": {
    "requestId": "string",
    "intent": "string",
    "confidence": "number",
    "requiresLLM": "boolean"
  },
  "callMethod": "Backend Internal Function 또는 Lightweight Model",
  "processing": "synchronous",
  "errorHandling": {
    "lowConfidence": "LLM 분석 단계로 전달",
    "missingInformation": "임의로 명령을 실행하지 않고 LLM 또는 사용자 재질문 단계로 전달"
  },
  "timeout": {
    "targetMs": 500,
    "maxMs": 1000
  }
}
```

#### (2) 단순 명령 처리 인터페이스 : HandleSimpleCommand
```
{
  "interface": "HandleSimpleCommand",
  "description": "LLM 호출이 필요하지 않은 단순 명령을 처리한다.",
  "input": {
    "intent": "string",
    "command": "string",
    "parameters": "object"
  },
  "output": {
    "success": "boolean",
    "responseText": "string | null",
    "robotCommand": "object | null"
  },
  "callMethod": "Backend Internal Function",
  "processing": "synchronous",
  "errorHandling": {
    "requiresExternalData": "LLM Layer로 전달",
    "requiresContext": "LLM Layer로 전달"
  },
  "timeout": {
    "maxMs": 1000
  }
}
```

---

### 4. LLM / MCP

**역할 및 책임**

복합적인 자연어 요청을 분석하고 필요한 외부 기능을 결정한다.
- 복합 사용자 요청 해석

- 업무명 / 날짜 / 시간 / 우선순위 추출

- 필요한 정보 누락 여부 판단

- 대화 맥락 활용

- MCP Tool 선택

- 외부 시스템 호출 결과 해석

- 불명확한 요청에 대한 재질문 생성

#### (1) 복합 요청 분석 인터페이스 : AnalyzeUserRequest
```
{
  "interface": "AnalyzeUserRequest",
  "description": "복합적인 사용자 요청을 분석하여 Intent와 필요한 작업을 결정한다.",
  "input": {
    "requestId": "string",
    "text": "string",
    "currentDateTime": "ISO-8601 datetime",
    "conversationContext": "object | null"
  },
  "output": {
    "intent": "string",
    "parameters": {
      "taskName": "string | null",
      "dueAt": "ISO-8601 datetime | null",
      "priority": "string | null",
      "reminderAt": "ISO-8601 datetime | null"
    },
    "requiredTool": "string | null",
    "requiresClarification": "boolean",
    "clarificationQuestion": "string | null"
  },
  "callMethod": "Backend Server → LLM API",
  "processing": "asynchronous",
  "errorHandling": {
    "missingRequiredInformation": "사용자에게 추가 질문",
    "ambiguousDateOrTask": "날짜나 업무 정보를 임의로 생성하지 않고 확인 요청",
    "llmFailure": "처리 실패 상태 반환"
  },
  "timeout": {
    "maxMs": 5000,
    "action": "최대 처리 시간을 초과하면 처리 실패 안내"
  }
}
```

#### (2) 외부 도구 호출 인터페이스 : CallMCPTool
```
{
  "interface": "CallMCPTool",
  "description": "LLM이 선택한 MCP Tool을 호출하고 실행 결과를 반환한다.",
  "input": {
    "requestId": "string",
    "toolName": "string",
    "operation": "string",
    "parameters": "object"
  },
  "output": {
    "requestId": "string",
    "success": "boolean",
    "result": "object | null",
    "error": {
      "code": "string | null",
      "message": "string | null"
    }
  },
  "callMethod": "Backend MCP Client → MCP Tool",
  "processing": "asynchronous",
  "errorHandling": {
    "toolFailure": "오류 코드와 원인을 Backend에 반환",
    "permissionDenied": "권한이 없는 기능은 실행하지 않음",
    "duplicateRequest": "데이터 변경 요청은 requestId를 통해 중복 실행 방지"
  },
  "timeout": {
    "minMs": 3000,
    "maxMs": 5000
  }
}
```

---

---

▸ 인라인 댓글

**블록 텍스트**: 사용자가 알림 시간을 지정하지 않은 경우 프로젝트 정책에 따라 `reminder_offset_minutes`에 기본값 `60`을 적용한다.
- **서현 ** Oct 7, 2026, 11:23 PMAI → backend정책 책임으로 이동
- **김지훈 ** Oct 7, 2026, 11:30 PMㅇㅇㄴㅇ
- **서현 ** Oct 7, 2026, 11:35 PM?

---

**블록 텍스트**: Intent 분류 인터페이스 : ClassifyIntent
- **서현 ** Oct 7, 2026, 11:36 PMtaskName/taskId 없음

---

**블록 텍스트**: "reminderAt": "ISO-8601 datetime | null"
- **서현 ** Oct 7, 2026, 11:25 PM

`"reminder_offset_minutes": 120` 앞부분 통일할것

---
