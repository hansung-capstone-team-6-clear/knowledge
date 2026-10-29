---
type: project
status: active
goal: "Eureka Process 업무를 음성으로 등록·조회·완료하고, 마감이 임박하면 먼저 알려주는 AI 데스크 펫을 12월 전시 데모까지 완성한다."
due: 2026-12-31
milestones:
  - name: "1차 — 음성 명령·업무 시나리오·MVP 범위 정의"
    due: 2026-09-30
    done: true
  - name: "2차 — STT·TTS 및 Eureka Process API 연동 프로토타입"
    due: 2026-10-31
    done: false
  - name: "3차 — Raspberry Pi·버튼·LED 통합 및 선제 알림 구현"
    due: 2026-11-30
    done: false
  - name: "4차 — 소음·오인식 테스트, 전시 데모, 최종 발표"
    due: 2026-12-31
    done: false
next_action: "클라우드 /api/voice 먼저 공개, 하드웨어 0·1·2번 구현, 바뀐 요청·응답 형식을 contracts.md에 반영, 다음 주~다다음 주 하드웨어–백엔드 연동 테스트 (10-29 회의)"
---

# eureka-desk-pet

## Status

Active — 2차(10월) 프로토타입 단계. 10/01 기업 미팅부터 본격 개발을 시작했다. 9월 산출물(시나리오·요구사항·정책·
4계층 구조·인터페이스 초안)과 Eureka API PoC, Mock Server 검증이 끝났다.

2026-10-08 멘토 기술 검토: 구현 가능, 단 AI·백엔드 레이어 단순화 필요
→ [[projects/eureka-desk-pet/outputs/2026-10-08-tech-review|기술 검토]]. 구현 기준은 `architecture.md`와 `contracts.md`다.

## Purpose

한성대학교 기업연계 AI 캡스톤 디자인 수업에서 클리어픽(6팀)이 협업 기업 레몬클라우드와 진행하는 프로젝트.
"단순한 음성 장난감이 아니라, 실제 업무 상태를 조회·변경하는 대화형 물리 인터페이스"를 목표로 한다.

## Current Context

### 팀 구성

| 팀원 | 담당 | 주요 역할 |
| --- | --- | --- |
| 김리윤 (팀장) | 기획·자원조사 & 하드웨어 | 3D 프린팅 자원 조사, 협업 환경 구축, 전체 기획 정리, 하드웨어 |
| 김지훈 | 하드웨어 & 임베디드 | 핀맵, 부품·학교 자원 조사, STT/TTS 구조 검토 |
| 박시현 | AI & 로컬 LLM | 로컬 LLM 구동 테스트, 에이전트 판단 흐름, AI 평가 기준 |
| 이서현 | 아키텍처 & 서버 | 4계층 구조도, 응답 시간 기준, Eureka Process 테스트 |

지도: 담당 교수 1인, 레몬클라우드 멘토 1인. 매주 기업 대면 미팅.

### 협업 방식 (10/01 합의)

- GitHub Organization에 역할별 저장소 `hardware`, `backend`, `ai`를 두고, 기능별 Branch → Pull Request로 병합한다.
- Git Convention(브랜치·커밋·PR·Merge 방식, 공용 Branch 직접 수정 금지, 리뷰 방식)을 먼저 정한다.
- 기업 미팅마다 각자 ① 이번 주 구현 기능 ② 실행 결과·시연 ③ 기술 이슈 ④ 다음 주 계획을 준비하고, 어려운 이슈는 미리 멘토에게 질문한다.
- 프로젝트 문서는 Notion에서 내보내 이 knowledge vault에 Wiki화한다(2026-10-08 첫 반영).
- 팀 규칙(9/10): 지각·무단결석 금지, 고정 톡방 1시간 내 답장, 적극 참여, 생성형 AI 결과 그대로 사용 금지, 상대 의견 존중.

### 10/01 역할별 과제

- 공통: Git/GitHub 학습, 담당 영역 구조 다이어그램·기술 스택 확정, 역할 간 인터페이스 충돌 검토.
- Backend: 기술 스택 확정, Docker 환경 검토, 인터페이스 JSON 재정리, Mock Backend 수정.
- AI: LLM 구조 후보 결정, Local LLM 사용 여부, 경량 모델 + Local LLM 구조, Tool/Skill 구조 조사.
- Hardware: 부품 확정과 예산안(부품 / LLM 구독 비용 분리), 구조 다이어그램, Backend와의 JSON 인터페이스.

## Folder Structure

```text
projects/eureka-desk-pet/
├── README.md          ← 지금 이 파일: 프로젝트 상태(frontmatter)·팀·폴더 안내·문서 규칙
├── architecture.md    ← 1차(MVP) 구조, 레이어 책임, 처리 흐름, 2차 고도화 로드맵 (레이어 공통)
├── contracts.md       ← Pi·Backend·AI·DB 사이 형식의 단일 진실원 (10/07 하드웨어 결정 기반)
├── hardware/
│   └── README.md      ← 하드웨어 레이어: 1차 범위, 핀맵, 먼저 확인할 것, Plan B, Hardware 레포 구조
├── backend/
│   └── README.md      ← 백엔드 레이어: desk-pet-backend 구조, Eureka 연동 체크리스트, 응답 시간 예산
├── ai/
│   └── README.md      ← AI 레이어: 모델 선택, 프롬프트 입력, 평가셋, 로컬 LLM 비교 실험
├── meetings/          ← 주간 기업(멘토) 미팅 회의록 — README가 회의 목록
└── outputs/           ← 날짜가 붙는 조사·검토 결과 (스냅샷 — 고치지 않고 새 파일 추가)
```

> 용어: 이 폴더 문서의 **1차(MVP) / 2차(고도화)**는 구현 단계다. 위 frontmatter `milestones`의 1~4차는 월별 로드맵이고,
> 1차 MVP는 로드맵 2~3차(10~11월) 동안 만든다.

## How to Use

### 무엇을 어디에 쓰나

| 쓰려는 것 | 위치 |
| --- | --- |
| 레이어 공통 구조·흐름 변경 | `architecture.md` |
| 레이어 간 JSON, AI 출력 형식, DB 스키마 | `contracts.md` |
| 레이어 내부 결정, 체크리스트, 실험 결과 | `<layer>/README.md` — 길어지면 같은 폴더에 `kebab-case.md` 추가 |
| 날짜가 붙는 조사·검토 결론 | `outputs/YYYY-MM-DD-<slug>.md` (`templates/query-output.md` 형식) |
| 기업(멘토) 미팅 회의록 | `meetings/YYYY-MM-DD-<slug>-meeting-note.md` + `meetings/README.md` 목록 갱신 |
| Notion 등 외부 문서 원문 | vault ingest → `raw/` 원문 + `wiki/desk-pet-*` 정리 |
| 코드 실행법, 환경 변수 목록 | 각 코드 레포의 README |

### 규칙

1. 레이어 간 형식을 바꾸려면 `contracts.md` PR을 먼저 올리고, merge 후 각 코드 레포에 반영한다.
2. 1차 범위 밖 아이디어는 `architecture.md` § 2차 고도화에 한 줄로 추가만 한다.
3. 연락처 같은 개인 정보, API 키·토큰, `.env` 내용은 커밋하지 않는다.
4. 파일명은 영어 kebab-case, 본문은 한국어.
5. 모든 변경은 브랜치 + PR로 올리고, merge는 사람이 승인한다.

### 레이어 담당과 코드 레포

| 레이어 | 문서 | 코드 레포 (구조) | 담당 |
| --- | --- | --- | --- |
| 하드웨어 | `hardware/` | `hansung-capstone-team-6-clear/Hardware` (`setup/` → `parts/` → `app/`) | 김지훈, 김리윤 |
| 백엔드 | `backend/` | `hansung-capstone-team-6-clear/desk-pet-backend` (`api/` → `services/` → `eureka.py`·`db.py`) | 이서현 |
| AI | `ai/` | `desk-pet-backend`의 `services/ai.py` | 박시현 |
| 공통 | `architecture.md`, `contracts.md` | — | 김리윤 주관, 전원 리뷰 |

10/01 합의의 `hardware`·`backend`·`ai` 3개 저장소 중 AI는 별도 레포 대신 백엔드 모듈로 둔다. LLM 호출은 Backend
프로세스 안에서 도는 함수 하나라서, 레포를 나누면 버전 맞추기만 늘어난다.

## Related Wiki

- [[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]] — 토픽 허브
- [[desk-pet-system-architecture|Desk Pet System Architecture]]
- [[proactive-deadline-alert|Proactive Deadline Alert]]
- [[intent-routing-and-structured-output|Intent Routing & Structured Output]]
- [[eureka-process-api|Eureka Process API]]
- [[desk-pet-hardware|Desk Pet Hardware]]
- [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]
- [[desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]]
- [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]] — `contracts.md` § 1·2의 기반
- [[wiki/desk-pet-device-states|장치 상태]] — `contracts.md` § 5의 기반
- [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]] — 10/07 하드웨어 결정 16건
- [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] — `contracts.md`가 닫은 항목은 contracts § 변경 이력 참고

## Log

- 2026-09-03 — 수업 OT, 팀 결성(팀명 클리어픽), 희망 주제 선택, 수행계획서 제출.
- 2026-09-10 — 주제 확인, 팀 가치관("한 가지를 배우고 두 가지를 만든다")·팀 규칙 결정.
- 2026-09-16 — 시나리오 아이디어 4종(업무 등록·조회, 팀원 간 위임, 이동 중 다음 일정, 출근 직후 오늘 일정), 외형 아이디어, 기업 질문 목록.
- 2026-09-17 — 기업 미팅: 공용 Claude 계정·예산, Raspberry Pi 확보 방식, 역할 분담, 개발 환경 통일, 매주 대면 미팅.
- 2026-09-23 — 역할 분담.
- 2026-09-28 — Eureka Process API PoC(전 기능 성공, 대부분 1초 이내).
- 2026-09-29~30 — 4계층 구조도·레이어별 인터페이스, 대표 시나리오를 "마감 임박 선제 알림"으로 단일화, Mock Server 검증, Raspberry Pi 대여 완료.
- 2026-10-01 — 중간 발표, 기업 미팅(범위 축소·GitHub 협업·인터페이스 JSON·Docker/Lambda·Knowledge 활용 피드백).
- 2026-10-07 — 팀 회의, 03. AI 문서 필드명·책임 위치 리뷰 댓글.
- 2026-10-08 — 멘토 기술 검토, 1차 최소 비용 조사, 1차 아키텍처·계약(`contracts.md`) 작성. 코드 레포 구조 셋업 PR:
  `desk-pet-backend#1`(api → services → eureka·db), `Hardware#1`(setup → parts → app).
- 2026-10-15 — Pi 기본 세팅 완료(완료 기준 4개 통과, OS는 trixie), GPIO17 버튼 테스트 통과. 남은 하드웨어 일은 오디오 실물 테스트
  → [[projects/eureka-desk-pet/hardware/README|Hardware 레이어]].
  같은 날 온라인 멘토 미팅(시험기간 약식) — 백엔드 1차 범위 3개 기능으로 축소, AI 평가셋 20개·Gemini 테스트,
  다음 회의는 시험(10-23 종료) 후 대면 → [[projects/eureka-desk-pet/meetings/2026-10-15-mentor-meeting-note|10-15 회의록]].
- 2026-10-29 — 온라인 멘토 미팅. 하드웨어는 녹음·전송·출력만, 처리는 모두 백엔드로 흐름 정리. 요청에서 `request_id` 제거(응답에만),
  WebSocket 선제 알림 1차 제외, 하드웨어 0·2번 김리윤 / 1번 김지훈, 백엔드 API·Eureka·출력 이서현 / 의도 분석·재질문 박시현
  → [[projects/eureka-desk-pet/meetings/2026-10-29-mentor-meeting-note|10-29 회의록]].

## Outputs

- 원문(Notion export): `raw/Capstone Eureka Process*.md`, `raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md`,
  `raw/LLM Wiki - AI 데스크 펫 프로젝트.md`
- 코드 첨부(`deskpet_mock_server.zip`, `Eureka_Process_API_Test.zip`)는 vault에 넣지 않았다 — 역할별 GitHub 저장소가 정본이다.
- [[projects/eureka-desk-pet/outputs/2026-10-08-tech-review|2026-10-08 기술 검토]]
- [[projects/eureka-desk-pet/outputs/2026-10-08-mvp-cost-research|2026-10-08 1차 최소 비용 조사]]
- [[projects/eureka-desk-pet/meetings/README|회의록 목록]] — 최신: [[projects/eureka-desk-pet/meetings/2026-10-29-mentor-meeting-note|2026-10-29 멘토 미팅]]
