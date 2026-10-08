---
type: concept
topics:
  - "[[wiki/topics/desk-pet|Desk Pet]]"
status: draft
sources:
  - "raw/desk-pet-backend-architecture.md"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Backend Architecture

## Summary

Desk Pet은 Raspberry Pi 기반 음성 데스크 펫으로, 사용자가 PTT 버튼을 누르고 말하면 서버가
음성을 텍스트로 바꾸고 의도를 분류해 업무(task)를 생성·조회·완료·수정하고, 마감이 다가오면
먼저 알림(`proactive_alert`)을 보낸다. 백엔드는 Ports & Adapters(hexagonal) 구조로,
업무의 진실원(source of truth)은 외부 시스템인 Eureka Process API이며 백엔드는
`TaskPort` 추상화를 통해서만 접근한다. LLM/MCP는 해석과 도구 선택만 하고 데이터를 직접
변경하지 않는다.

## Details

### 레이어 구성

| Layer | 구성 요소 | 역할 |
| --- | --- | --- |
| Device / Hardware | User / PTT, Hardware, Audio Capture | PTT·마이크·LED·LCD·스피커, WAV/PCM 녹음 |
| Local Processing | STT, Lightweight Intent Model, LLM Service, MCP Client | 음성→텍스트(ko-KR), 의도 분류, 복잡 요청 분석, 도구 호출 |
| Application | Intent / LLM Orchestrator, TaskService, ReminderService, OutputService | 검증·라우팅, 업무 처리, 마감 알림 정책, 응답 생성 |
| Ports | TaskPort, OutputPublisherPort | Application ↔ 외부 경계 추상화 |
| Infrastructure / Adapter | SchedulerAdapter(driver), EurekaTaskAdapter, WebSocketAdapter | 주기 트리거, HTTP 매핑, WebSocket push |
| External Systems | Eureka Process API, Raspberry Pi | 업무 진실원(REST), Edge-TTS 합성·하드웨어 출력 |

### 기본 요청 경로

1. Raspberry Pi가 음성을 녹음해 `POST /api/voice`로 서버에 업로드한다(최대 녹음 15초).
2. 서버 측 STT가 텍스트로 변환한다(목표 1초, 최대 5초).
3. Lightweight Intent Model이 의도·confidence·간단한 파라미터를 뽑고 `requiresLLM`을
   판정한다(목표 500ms). 신뢰도가 낮거나 복잡한 요청만 LLM Service로 넘긴다.
4. LLM Service는 `taskName`·`dueAt`·`priority`·`reminderAt`을 추출하고 모호하거나 빠진
   정보를 감지해 재질문을 만들며, 필요하면 MCP 도구를 고른다. MCP Client가 구조화된 도구
   호출을 수행한다.
5. Intent / LLM Orchestrator가 AI·MCP 결과를 **검증한 뒤** Application Service로
   라우팅한다. 재질문(clarification)이 필요한 경우에는 데이터를 변경하지 않고 바로
   OutputService로 보낸다.
6. TaskService → `TaskPort` → `EurekaTaskAdapter` → Eureka Process API 순으로 업무를 처리한다.
7. OutputService는 TTS용 문장과 LCD 표정/텍스트만 만든다. 일반 응답은 HTTP Response로
   Pi에 돌려주고, 음성 합성(Edge-TTS)과 LED/LCD 제어는 Pi가 담당한다.

### 설계 원칙

- **LLM/MCP는 직접 쓰지 않는다**: LLM이 도구를 제안·선택해도 업무 변경은 Orchestrator
  검증 → TaskService/ReminderService → TaskPort 경로로만 일어난다. LLM/MCP에서 Eureka로
  가는 직접 변경 경로는 없다.
- **경량 모델 우선**: 단순 요청은 Lightweight Intent Model에서 끝내고, LLM은 복잡하거나
  confidence가 낮은 요청에만 쓴다.
- **응답 채널 분리**: 사용자 요청에 대한 일반 응답은 HTTP, 서버가 먼저 보내는 출력
  (`output`, `proactive_alert`)만 `OutputPublisherPort` → `WebSocketAdapter`
  (`/ws/hardware/{user_id}`)로 보낸다. WebSocket 전송 성공은 기기 재생 확인(ACK)이 아니다.
- **실패 시 추정하지 않는다**: Eureka 등록이 실패하면 성공으로 처리하지 않고, 조회 실패 시
  기존 값을 추정하거나 임의 업무를 만들지 않는다.

### 마감 알림 정책 (ReminderService)

- `SchedulerAdapter`가 polling으로 `ReminderService`를 주기 실행만 한다.
- 완료(`done`) 업무는 평가에서 제외한다.
- 사용자 지정 `reminderAt`이 있으면 그 값을, 없으면 `dueAt` 1시간 전을 기본값으로 쓴다.
- 중복 알림을 막은 뒤 OutputService → OutputPublisherPort → WebSocketAdapter → Pi로
  `proactive_alert`(`type: deadline_approaching`)를 보낸다.

### 인터페이스 계약 요약

원문에는 21개 계약/이벤트가 정의되어 있다(전체 필드·타임아웃·오류 정책은 raw 캡처 참조).

- Desk Pet Backend API(`/api/*`): `POST /api/tasks`, `GET /api/tasks`,
  `GET /api/tasks/{itemId}/status`, `POST /api/tasks/{itemId}/complete`,
  `PUT /api/tasks/{itemId}` — 모두 TaskService → TaskPort 경유.
- `TaskPort` 메서드: `create_task()`, `get_tasks()`, `get_task_status()`,
  `complete_task()`, `update_task()`.
- Eureka Process API(`/_api_/*`): `EurekaTaskAdapter`가 업무 생성·목록·상태·완료·수정을
  HTTP로 매핑한다(목표 1초, 최대 2초). 업무 상태는 `todo | doing | done | hold`.
- 하드웨어 출력: LED 상태는 `IDLE`(초록 점등)·`LISTENING`(노랑 점멸)·`SPEAKING`(파랑 점멸)·`ERROR`.

## Connections

- [[wiki/topics/desk-pet|Desk Pet]] — 프로젝트 토픽 페이지
- 관련 개념(아직 wiki 노트 없음): hexagonal architecture(Ports & Adapters), MCP(Model Context
  Protocol), intent classification

## Open Questions

- 중복 알림 방지 상태의 저장 구현체가 미확정이다(원문: "Persistence remains undecided").
- `proactive_alert` 전달 실패 시의 "재처리 정책" 내용이 원문에 정의되어 있지 않다.
- MCP Client의 "외부 분석/읽기 도구 접근" 범위(읽기 전용 도구가 Eureka를 직접 조회할 수
  있는지)가 다이어그램에 명시되지 않았다 — inference, 확인 필요.
- 원문 다이어그램의 엣지 key 11·12·17이 비어 있어, 편집 과정에서 제거된 연결이 있는지
  확인이 필요하다 — needs-update.
