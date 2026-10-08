---
type: concept
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 01 기획.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
related:
  - "[[wiki/proactive-deadline-alert|Proactive Deadline Alert]]"
  - "[[wiki/intent-routing-and-structured-output|Intent Routing & Structured Output]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Requirements & Policies

## Summary

데스크 펫의 요구사항(기능 23건 FR-01~23, 비기능 12건 NFR-01~12), 구현 범위, 운영 정책을 정리한 노트다. 핵심 범위는
"음성으로 업무를 관리하고, 마감이 가까워지면 펫이 먼저 알려주는 기능까지"이며, Second Brain 기반 장기 기억은 확장 기능으로
분류된다. 정책의 공통 원칙은 **모호하면 실행하지 말고 묻는다**, **실제 성공을 확인하기 전에는 성공이라 말하지 않는다**,
**응답은 발화 종료 후 3초 안에 시작한다**이다.

## Details

### 기능 요구사항 묶음

| 묶음 | ID | 내용 (모두 Must, 표시한 것만 Should) |
| --- | --- | --- |
| 음성 입력 | FR-01~04 | PTT 녹음, STT, Intent 분류, 복합 요청 슬롯 추출 |
| 업무 관리 | FR-05~09 | 등록(`itemId` 반환), 목록, 완료(`done` 확인), 상태(`todo/doing/done/hold`), 마감 변경·연기 |
| 마감 알림 | FR-10~15 | 업무별 알림 시점, 기본 60분 전, 주기적 Polling 탐지, 요청 없는 능동 알림, 완료 업무 제외, 중복 방지 |
| 맥락 | FR-16~17 (Should) | Second Brain에 메모·회의 기록 저장, "어디까지 했었지?" 류 과거 맥락 검색 |
| 출력 | FR-18~20 | LED `IDLE/LISTENING/SPEAKING`, LCD 표정·텍스트(Should), TTS 결과 안내 |
| 안정성 | FR-21~23 | 미전달 알림 보존(Should), 불명확 명령 재질문, 단순 명령은 LLM 없이(Should) |

### 비기능 요구사항

- 반응성: 버튼→LED 100ms(NFR-02), STT 1초(NFR-03), Intent 0.5초(NFR-04), Eureka 1초·최대 2초(NFR-05).
- 견고성: Timeout 시 무한 대기 금지(NFR-06), 일부 장치·서비스 오류가 전체를 멈추지 않음(NFR-08), 요청 ID 추적(NFR-09),
  데이터 변경 중복 실행 방지(NFR-10).
- 보안·구조: API Key 하드코딩 금지 — `.env`/환경변수(NFR-07), HTTPS 기본(NFR-11), 계층 간 인터페이스 분리(NFR-12).

### 응답 시간 예산

| 구간 | 목표 | 상한 |
| --- | --- | --- |
| 버튼 → LED 반응 | 100ms | 200ms |
| 발화 종료 → STT 결과 | 1초 | 1.5초 |
| Intent 분석 | 500ms | 1초 |
| Backend 내부 처리 | 300ms | 500ms |
| Eureka API 응답 | 1초 | 2초 |
| TTS 생성 시작 | 1초 | 1.5초 |
| **발화 종료 → 음성 응답 시작** | **3초** | **5초** |

### 정책

- **명령 처리**: 명확하면 바로 실행. 대상·날짜·시간이 부족하면 값을 만들지 않고 재질문. 대상은 명확한데 알림 시점만 없으면 기본 1시간 전.
- **사용자 확인**: 대상이 하나로 특정되지 않을 때, 날짜 해석이 여러 가지일 때, 기존 중요 값을 크게 바꿀 때만 묻는다. 명확한 등록·완료·연기는 실행 후 결과만 안내.
- **성공/실패 안내**: 외부 시스템 처리 성공을 확인하기 전에 성공이라 안내하지 않는다.
- **Fallback**: STT 실패 → 재입력 요청 / 경량 모델 미확정 → LLM / LLM 미확정 → 재질문 / MCP·Eureka 실패 → 변경 완료로 간주하지 않음 / 알림 전달 실패 → 미전달 저장 후 복구 시 최신 상태 확인 뒤 필요할 때만 재전송.
- **Eureka 관련**: Eureka API 형식을 Pi에 직접 노출하지 않고 Backend Adapter로 감싼다. API Key는 Backend `.env`에만 둔다.
- **컨텍스트 저장**: 업무 기본 정보는 Eureka, 메모·회의 기록·마지막 작업·알림 설정·전달 이력은 Second Brain/Backend.

### 구현 범위와 제외

- 포함: PTT 음성 입력, STT, 경량 Intent + LLM, 업무 CRUD, 사용자 지정·기본 마감 알림, Polling, 능동 알림, TTS·LED·LCD 출력, Eureka 연동, WebSocket 알림, 네트워크·Timeout 처리.
- 제외: 행동 패턴 학습, 장기 개인화 모델, AI 우선순위 추천, AI 일정 자동 생성, 이메일·캘린더 연동, 다중 업무 시스템 통합, 완전 오프라인 AI, 다중 계정·권한, 모바일 앱·웹 대시보드.

### 10/01 멘토 피드백에 따른 범위 조정

기본 기능을 안정적으로 만드는 것이 최우선이다. 업무 연기, 업무 상태 저장, 과거 맥락 저장 기능은 줄이고, 네트워크
장애·복잡한 예외는 후순위로 미루며, 성능 기준은 구현 후 실측으로 조정한다.

## Connections

- 대표 시나리오 → [[proactive-deadline-alert|Proactive Deadline Alert]]
- 재질문·구조화 출력 → [[intent-routing-and-structured-output|Intent Routing & Structured Output]]
- 인터페이스별 Timeout → [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]

## Open Questions

- 10/01 범위 축소(업무 연기 제외 등)가 요구사항 표에 아직 반영되지 않았다 — FR-09(마감 변경·연기)는 여전히 Must, FR-21은 Should (needs-update).
- E2E 응답 목표가 문서마다 다르다: 정책 3초(상한 5초) vs 9/30 PoC 기준 1.5초 미만. Eureka 지연도 정책 1초 vs PoC 기준 300ms.
- 요구사항 건수: 중간발표 대본은 "기능 24건 내외"로 적었지만 표는 FR-01~23(23건)이다.
