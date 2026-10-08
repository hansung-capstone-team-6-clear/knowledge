---
type: project
status: active
goal: "Eureka Process와 연동되는 업무관리 AI 데스크펫을 최소 비용 1차 구현(MVP)으로 완성한 뒤 단계적으로 고도화한다."
due:
milestones:
  - name: "contracts.md 확정 + /text → LLM → Eureka 연결 (발화 20개 중 18개 성공)"
    due: 2026-10-18
    done: false
  - name: "/voice 음성 E2E 5초 이내 + Pi LED/LCD·WS 수신"
    due: 2026-10-31
    done: false
  - name: "선제 알림 실연동(SQLite) + Pi와 Backend 실제 연결"
    due: 2026-11-14
    done: false
  - name: "하우징 조립 + 데모 시나리오 4개 리허설"
    due: 2026-11-28
    done: false
  - name: "소음·오인식 테스트 + 최종 발표"
    due:
    done: false
next_action: "팀 회의에서 contracts.md를 검토해 1차 인터페이스로 확정한다."
---

# desk-pet

## Status

Active · 1차(MVP) 설계 확정 단계. 2026-10-08 기술 검토 결과 **구현 가능, 단 AI·백엔드 레이어 단순화 필요**
→ [[projects/desk-pet/outputs/2026-10-08-tech-review|기술 검토]].

## Purpose

한성대학교 기업연계 AI 캡스톤 6팀(협업 기업: 레몬클라우드) 프로젝트. 사용자가 음성으로 Eureka Process
업무를 등록·조회·완료하고, 마감이 다가오면 데스크펫이 먼저 알려주는 물리 인터페이스를 만든다.

진행 원칙은 하나다. **1차 = 최소 비용으로 끝까지 동작하는 MVP, 2차 = 고도화.** 2차 항목은
`architecture.md` § 2차 고도화 목록에 적어 두기만 하고, 1차 완료 전에는 착수하지 않는다.

## Current Context

- 실행 단위는 2개: Raspberry Pi 프로그램 1개, Backend 프로세스 1개. 업무 원본은 Eureka Process.
- 1차 월 운영비 목표 0원 (노트북 호스팅 + 무료 등급 API). 근거:
  [[projects/desk-pet/outputs/2026-10-08-mvp-cost-research|1차 최소 비용 조사]].
- 코드 레포: `hansung-capstone-team-6-clear/Hardware`, `hansung-capstone-team-6-clear/desk-pet-backend`.
  AI는 별도 레포 없이 backend 안의 모듈로 둔다.
- 설계·조사·결정 문서의 최종 위치는 이 폴더다 (backend 레포 `docs/README.md`의 이전 계획과 같음).

## Folder Structure

```text
projects/desk-pet/
├── README.md          ← 지금 이 파일: 프로젝트 상태(frontmatter)·폴더 안내·문서 규칙
├── architecture.md    ← 1차 아키텍처, 레이어 책임, 처리 흐름, 2차 고도화 로드맵 (레이어 공통)
├── contracts.md       ← 레이어 간 주고받는 JSON·AI 출력·DB 스키마의 단일 진실원 (레이어 공통)
├── hardware/
│   └── README.md      ← 하드웨어 레이어: 1차 범위, 핀맵, 먼저 확인할 것, Plan B
├── backend/
│   └── README.md      ← 백엔드 레이어: 모듈 구성, 레포 정리 항목, Eureka 연동 체크리스트
├── ai/
│   └── README.md      ← AI 레이어: 모델 선택, 프롬프트 입력, 평가셋, 로컬 LLM 비교 실험(2차)
└── outputs/           ← 날짜가 붙는 조사·검토 결과 (스냅샷, 수정하지 않고 새 파일 추가)
    ├── 2026-10-08-tech-review.md
    └── 2026-10-08-mvp-cost-research.md
```

## How to Use

### 무엇을 어디에 쓰나

| 쓰려는 것 | 위치 |
| --- | --- |
| 레이어 공통 구조·흐름 변경 | `architecture.md` |
| 레이어 간 JSON, AI 출력 형식, DB 스키마 | `contracts.md` |
| 레이어 내부 결정, 체크리스트, 실험 결과 | `<layer>/README.md` — 길어지면 같은 폴더에 `kebab-case.md` 추가 |
| 날짜가 붙는 조사·검토·회의 결론 | `outputs/YYYY-MM-DD-<slug>.md` (`templates/query-output.md` 형식) |
| 코드 실행법, 환경 변수 목록 | 각 코드 레포의 README |
| 여러 프로젝트에서 재사용할 개념 (예: I2S, WebSocket) | `wiki/` — vault ingest/승격 절차로 |

### 규칙

1. 레이어 간 인터페이스를 바꾸려면 `contracts.md` PR을 먼저 올리고, merge 후 각 코드 레포에 반영한다.
2. 1차 범위 밖 아이디어는 `architecture.md` § 2차 고도화에 한 줄로 추가만 한다.
3. 연락처 같은 개인 정보, API 키·토큰, `.env` 내용은 커밋하지 않는다.
4. 파일명은 영어 kebab-case, 본문은 한국어.
5. 모든 변경은 브랜치 + PR로 올리고, merge는 사람이 승인한다.

### 레이어 담당

| 레이어 | 문서 | 코드 | 담당 역할 |
| --- | --- | --- | --- |
| 하드웨어 | `hardware/` | `Hardware` 레포 | 하드웨어·임베디드 |
| 백엔드 | `backend/` | `desk-pet-backend` 레포 | 아키텍처·서버 |
| AI | `ai/` | `desk-pet-backend` 레포의 `services/ai.py` | AI·LLM |
| 공통 | `architecture.md`, `contracts.md` | — | 팀장(기획) 주관, 전원 리뷰 |

### 왜 레이어별 프로젝트로 나누지 않았나

- 프로젝트 상태(`status`·`milestones`·`next_action`)는 제품 단위로 하나여야 진행 상황이 한눈에 보인다.
- 10/01 기업 미팅 피드백대로 레이어 간 충돌을 막으려면 계약 문서가 한 곳(`contracts.md`)에 있어야 한다.
- 코드 레포는 이미 실행 단위별(`Hardware`, `desk-pet-backend`)로 나뉘어 있으므로, 문서는 한 프로젝트 안에서
  레이어 폴더로만 나눈다. AI 레포는 만들지 않는다 — AI는 backend 프로세스 안에서 도는 모듈이다.

## Related Wiki

- [[wiki/INDEX|Wiki Index]] — 데스크펫 관련 개념 문서는 아직 없다.

## Log

- 2026-10-08: 프로젝트 폴더 생성. 기술 검토, 1차 최소 비용 조사, 1차 아키텍처·계약 초안 작성.
- 2026-10-08: 코드 레포 구조 셋업 PR — `desk-pet-backend#1`(api → services → eureka·db), `Hardware#1`(setup → parts → app).

## Outputs

- [[projects/desk-pet/outputs/2026-10-08-tech-review|2026-10-08 기술 검토]]
- [[projects/desk-pet/outputs/2026-10-08-mvp-cost-research|2026-10-08 1차 최소 비용 조사]]
