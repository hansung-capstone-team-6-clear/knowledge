---
type: concept
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 02 시스템 설계.md"
  - "raw/Capstone Eureka Process - 02 desk-pet-architecture 다이어그램.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
  - "raw/desk-pet-backend-architecture.md"
  - "raw/데스크펫 MVP 아키텍처.md"
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
related:
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
  - "[[wiki/proactive-deadline-alert|Proactive Deadline Alert]]"
  - "[[wiki/intent-routing-and-structured-output|Intent Routing & Structured Output]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]"
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

### 개정 다이어그램 (archify `Desk Pet Backend Architecture`, 2026-10-08 캡처)

같은 다이어그램의 개정판(`raw/desk-pet-backend-architecture.md`)은 위 Notion 첨부본과 다음이 다르다.

- **LLM Service·MCP Client가 별도 노드로 추가**: 경량 Intent Model이 복잡하거나 confidence가 낮은 요청만
  LLM Service로 넘기고, LLM이 고른 도구는 MCP Client가 구조화 호출한다. 두 결과 모두 Orchestrator가
  **검증한 뒤** Application Service로 라우팅한다. LLM/MCP → Eureka 직접 변경 경로는 없다.
- **재질문은 데이터 무변경**: clarification이 필요하면 Orchestrator가 바로 OutputService로 보낸다.
- **영속화 미확정으로 후퇴**: `ReminderRepositoryPort`·`DatabaseAdapter`가 빠지고 "Persistence remains undecided"로
  표기됐다. `MockTaskAdapter`(test-only)도 다이어그램에서 빠졌다(엣지 key 11·12·17 공백이 이 제거의 흔적).
- **응답 채널 분리**: 음성은 `POST /api/voice`로 업로드하고 STT는 서버에서 한다. 일반 응답은 HTTP Response로
  Pi에 돌려주고, 서버가 먼저 보내는 출력(`output`·`proactive_alert`)만 `OutputPublisherPort` → `WebSocketAdapter`를 쓴다.
- **TTS 합성 위치**: OutputService는 TTS 문장·LCD 표정/텍스트만 만들고, 음성 합성(Edge-TTS)과 LED·LCD 제어는 Pi가 맡는다.

### 하드웨어 MVP 아키텍처 (2026-10-07 정리안 · 「데스크펫 MVP 아키텍처」)

하드웨어 쪽 MVP 정리안은 위 개정 다이어그램과 같은 방향으로 Pi–Backend 경계를 확정했다. **통로는 두 개뿐**이다.

| 계층 | 구성 |
| --- | --- |
| 1 · 하드웨어 | 마이크 ICS-43434, PTT 버튼(GPIO17), 앰프 MAX98357A + 3W 스피커, LED 링 24구(GPIO12), LCD ST7789(SPI0). 서보 제외 |
| 2 · Pi 소프트웨어 (Python) | `IAudioCapture`, `IButtonInput`, `IDeviceStateManager`(중심), `IHardwareOutput`, `IVoiceUpload`, `IBackendConnection`, `IDeviceStatusMonitor` |
| 3 · Backend | `POST /api/voice`, STT(Faster-Whisper), Intent · Local LLM, WebSocket `/ws/hardware/{user_id}`, Scheduler, Eureka API Adapter, DB(알림 이력·미전달 알림) |
| 4 · 외부 | Eureka Process |

- **음성 요청 (HTTP)**: 버튼 누름 → `LISTENING`·녹음(최대 15초) → 버튼 뗌 → Pi가 스스로 `THINKING` → WAV를 `POST /api/voice`로
  업로드(5초 발화 약 160KB, 0.1~0.3초) → Backend가 STT·Intent·LLM·Eureka 처리 후 최종 `output`을 응답 본문으로 반환 →
  `SPEAKING`·Edge-TTS 재생 → `IDLE`. 10초 안에 응답이 없으면 `ERROR`.
- **마감 알림 (WebSocket)**: Scheduler가 Eureka를 Polling → 마감 임박(기본 1시간 전)·미완료 업무 → `proactive_alert` push →
  `ALERT`·LCD·TTS → `IDLE`. 연결이 끊겨 있었으면 재연결 시 Backend가 재전송. Pi는 30초마다 PING.

| 설계 결정 | 이유 |
| --- | --- |
| 음성은 녹음 후 파일 업로드 | Faster-Whisper는 음성을 다 받은 뒤 인식하므로 스트리밍으로 줄일 수 있는 건 업로드 0.1~0.3초뿐 |
| TTS는 문장만 받고 Pi가 합성 | Mock Server가 이미 문장을 보내고, 오디오 송수신 코드가 필요 없음 |
| LED 색은 Pi가 결정 | 색 정의가 한 곳에 모여 LISTENING/ALERT 노랑 중복 같은 충돌이 없어짐 |
| WebSocket 이벤트는 Mock 이름 유지 | `connected`·`proactive_alert`·`output`·`PONG` 그대로 써서 기존 테스트 코드 재사용 |
| 서보는 MVP 제외 | LED와 PWM 공유, 앰프와 5V 핀 공유 문제가 함께 사라짐 |

스트리밍 전환 기준: 전체 흐름 완성 후 업로드·STT·LLM·TTS 구간별 시간을 실측하고, 업로드가 병목일 때만 `IVoiceUpload`를
WebSocket 스트리밍으로 바꾼다. 체감 속도 개선에는 TTS를 문장 단위로 나눠 첫 문장부터 재생하는 편이 더 효과적이다.
메시지 형식은 [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]], 결정 배경은 [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]].

### 처리 위치 원칙

- **Pi / 로컬**: PTT 상태, 녹음, LED·LCD·스피커 제어, 장치·네트워크 상태 확인, 오프라인 시계, (가능하면) 경량 Intent.
- **Backend / Cloud**: 복합 요청 해석, LLM·MCP 호출, Eureka 연동, Second Brain 검색·저장, 마감 Scheduler, 알림·Context 관리.
- 초기 하드웨어 조사에서는 Pi 4B에서 STT·LLM을 모두 돌리면 CPU 100%·발열·LCD 프레임 드롭이 생겨,
  **VAD만 로컬, STT·LLM·TTS는 서버/API**를 권장했다. 10/07 MVP 확정안은 VAD 없음(PTT), TTS는 Pi의 Edge-TTS다([[desk-pet-hardware|Desk Pet Hardware]]).

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
- AI 처리 구조 문서와 하드웨어 정리안의 충돌 13건 → [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]

## Open Questions

- 배포 형태 미정: AWS Lambda에서 상시 WebSocket·Polling Scheduler를 어떻게 유지할지 (needs-update).
- Pi↔Backend 음성 전송 방식이 문서마다 다르다 — HTTPS `POST /api/voice` 업로드(IVoiceUpload) vs WebSocket PCM 스트림(9/30 안).
  개정 다이어그램(2026-10-08)과 하드웨어 MVP 정리안(10-07) 모두 HTTP 업로드 + 서버 STT다. 남은 건 Backend 담당 합의다 (needs-update).
- 알림 이력·중복 방지 상태 저장소: 개정 다이어그램에서 `DatabaseAdapter`가 빠져 미확정 상태다 (needs-update).
- TTS 위치: 초기 하드웨어 권장안은 서버/API TTS였으나 개정 다이어그램과 10/07 MVP 정리안은 Pi의 Edge-TTS다. Backend 합의와 Pi 합성 시간 실측이 남았다 (needs-update).
- 처리 시간: AI 쪽 최악 약 16초(STT 5 + Intent 1 + LLM 5 + MCP 5) vs Pi 대기 10초 — [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 1번.
- Second Brain 구현체 미정 — 백엔드 DB, Notion API, 팀 knowledge vault 중 무엇을 쓸지.
