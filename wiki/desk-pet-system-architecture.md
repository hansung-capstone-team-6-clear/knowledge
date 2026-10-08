---
type: concept
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 02 시스템 설계.md"
  - "raw/Capstone Eureka Process - 02 desk-pet-architecture 다이어그램.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
related:
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
  - "[[wiki/proactive-deadline-alert|Proactive Deadline Alert]]"
  - "[[wiki/intent-routing-and-structured-output|Intent Routing & Structured Output]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet System Architecture

## Summary

Eureka Process 연동형 업무관리 AI 데스크 펫의 시스템 구조. 책상 위 Raspberry Pi 디바이스는 입출력만
맡고, 음성 인식·의도 해석·업무 처리·마감 감시는 Backend가 담당하는 **Edge–Backend 분리 4계층 구조**다.
Backend 내부는 Port/Adapter(헥사고날) 구조로 설계해 Eureka Process, DB, WebSocket 같은 외부 의존을
Adapter 뒤로 숨긴다. 계층 간 통신 규약은 [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]에
JSON으로 정의돼 있다.

## Details

### 4계층 구조 (02. 시스템 설계 기준)

| 계층 | 위치 | 담당 |
| --- | --- | --- |
| 1. Device / Hardware Layer | Raspberry Pi | Push-to-Talk 버튼, 마이크 녹음, 스피커, LED 링, LCD 표정·텍스트 |
| 2. Voice & Local Processing Layer | Raspberry Pi / 로컬 | PTT 상태 관리, 하드웨어 제어, 네트워크 통신(WebSocket), Backend 응답을 하드웨어에 반영 |
| 3. AI & Backend Layer | 서버 | STT, 경량 Intent 분류, LLM/MCP, Scheduler·Polling 마감 감지, Eureka 연동, 알림 이력, `proactive_alert` 생성 |
| 4. External / Process Layer | 외부 | Eureka Process Cloud(업무 CRUD·상태), 회의 기록·알림 설정·이력 저장소 |

Backend 처리 결과는 **Output Controller**가 TTS 문장·LED 상태·LCD 표정/텍스트 명령으로 변환해 Pi에 보낸다.

> 9/30 김지훈 안은 같은 구조를 Physical / Edge Engine / Application·Backend / AI & Service Engine으로
> 나눴다. 계층 이름만 다르고 경계(디바이스는 I/O, 서버는 판단)는 같다.

### 주요 처리 흐름

- **일반 음성 요청**: PTT → 녹음 → Backend 전송 → STT → 경량 Intent 분류 → (단순) Backend 직접 처리 /
  (복합) LLM → 필요 시 Eureka·Second Brain 조회 → Output Controller → Pi에서 TTS·LED·LCD 출력.
  상세는 [[intent-routing-and-structured-output|Intent Routing & Structured Output]].
- **마감 임박 선제 알림**: Scheduler가 Eureka를 Polling → 완료 업무 제외 → 알림 시점 비교 →
  `proactive_alert` → WebSocket으로 Pi에 Push → 이력 저장. 상세는 [[proactive-deadline-alert|Proactive Deadline Alert]].
- **예외**: Timeout·재시도 정책 적용, 데이터 변경의 실제 반영 여부 확인, 알림 전달 실패 시 미전달 저장 후 복구 시 재확인·재전송.

### Backend 내부 구조 (desk-pet-architecture 다이어그램)

| 층 | 구성요소 | 역할 |
| --- | --- | --- |
| Application | Intent / LLM Orchestrator | intent 라우팅, 슬롯 추출, 재질문 |
| Application | TaskService · ReminderService · OutputService | 업무 CRUD / 마감 확인·중복 방지·정책 / TTS·LED·LCD 명령 생성 |
| Ports | TaskPort · ReminderRepositoryPort · OutputPublisherPort | 업무 연산, 알림 설정·전달 이력, 출력 이벤트 발행 |
| Adapters | EurekaTaskAdapter · DatabaseAdapter · WebSocketAdapter · SchedulerAdapter | Eureka HTTP 매핑, 알림 영속화, `proactive_alert` 전달, 주기 트리거 |
| Test-only | MockTaskAdapter | 테스트용 TaskPort 대체 구현 |

`TaskPort`를 Eureka Adapter와 Mock Adapter가 모두 구현하므로, Mock Server에서 검증한 알림 흐름을 실제
Eureka 연동에서도 그대로 쓸 수 있다는 것이 이 구조의 핵심 근거다(04. Backend § 2-7). NFR-12("Eureka 또는 LLM
구현을 바꿔도 Hardware Layer를 직접 수정하지 않는다")와도 대응한다.

### 처리 위치 원칙

- **Pi / 로컬**: PTT 상태, 녹음, LED·LCD·스피커 제어, 장치·네트워크 상태 확인, 오프라인 시계, (가능하면) 경량 Intent.
- **Backend / Cloud**: 복합 요청 해석, LLM·MCP 호출, Eureka 연동, Second Brain 검색·저장, 마감 Scheduler, 알림·Context 관리.
- 하드웨어 조사 결과 Pi 4B에서 STT·LLM을 모두 돌리면 CPU 100%·발열·LCD 프레임 드롭이 생겨,
  **VAD만 로컬, STT·LLM·TTS는 서버/API**가 권장안이다([[desk-pet-hardware|Desk Pet Hardware]]).

### 멘토 권장 사항 (10/01)

- Backend는 Docker 컨테이너로 개발 환경을 통일한다.
- 배포는 AWS Lambda를 검토하되, 상시 실행이 필요한 Scheduler·WebSocket이 Lambda 구조에 맞는지 확인해야 한다.
- LLM에게 기능을 직접 맡기지 않고 `create_task`, `update_task`, `complete_task`, `get_task_list`,
  `search_knowledge`, `create_alert` 같은 Tool/Skill 단위로 분리한다.
- 각 담당자가 자기 영역의 구현 수준 구조 다이어그램을 따로 그린다.

## Connections

- 계층 간 JSON 규약 → [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]
- 대표 시나리오 → [[proactive-deadline-alert|Proactive Deadline Alert]]
- AI 계층 → [[intent-routing-and-structured-output|Intent Routing & Structured Output]]
- 외부 업무 시스템 → [[eureka-process-api|Eureka Process API]]
- 요구사항·정책 → [[desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]]

## Open Questions

- 배포 형태 미정: AWS Lambda에서 상시 WebSocket·Polling Scheduler를 어떻게 유지할지 (needs-update).
- Pi↔Backend 음성 전송 방식이 문서마다 다르다 — HTTPS `POST /api/voice` 업로드(IVoiceUpload) vs WebSocket PCM 스트림(9/30 안).
- Second Brain 구현체 미정 — 백엔드 DB, Notion API, 팀 knowledge vault 중 무엇을 쓸지.
