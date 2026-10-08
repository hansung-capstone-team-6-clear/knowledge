---
type: pattern
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 01 기획.md"
  - "raw/Capstone Eureka Process - 02 시스템 설계.md"
  - "raw/Capstone Eureka Process - 04 Backend.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
related:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/eureka-process-api|Eureka Process API]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Proactive Deadline Alert

## Summary

데스크 펫 프로젝트의 **대표 시나리오(최종 MVP)**. 사용자가 묻지 않아도 Backend가 Eureka Process 업무의
마감을 주기적으로 확인하다가, 사용자가 지정한 시점(미지정 시 **마감 1시간 전**)에 미완료 업무가 있으면
`proactive_alert` 이벤트를 WebSocket으로 데스크 펫에 Push해 LED·LCD·TTS로 먼저 알려준다.
일반 음성 챗봇과의 차별점("묻기 전에 먼저 알려준다")이 이 패턴에서 나온다.

## Details

### 왜 Polling인가

Eureka Process는 마감 도달·상태 변경을 알려주는 Push/Webhook이 없고, 마감 기한 범위 검색도 지원하지 않는다
([[eureka-process-api|Eureka Process API]] § 제약). 그래서 Backend Scheduler가 업무 목록을 주기적으로 가져와
`dueAt`을 직접 필터링한다.

### 데이터 책임 분리

| 저장소 | 담는 것 |
| --- | --- |
| Eureka Process | 업무명, 마감 기한(`dueAt`), 우선순위, Stage, 업무 상태 |
| Second Brain / Backend 저장소 | 사용자 지정 알림 시점(`itemId`에 연결), 알림 전달 여부, 미전달 알림, 업무 메모·과거 맥락 |

### 정상 흐름

1. 사용자: "9월 27일 오후 3시까지 보고서 검토해야 해. 마감 2시간 전에 알려줘."
2. STT → Intent/LLM이 업무명·마감·알림 시점(120분 전)을 추출 → Eureka에 업무 등록, 반환된 `itemId`와 알림 설정 저장.
3. Scheduler가 Eureka 업무 목록을 주기적으로 조회한다.
4. 알림 시점(오후 1시)에 업무가 미완료면 `proactive_alert` 생성 → WebSocket Push.
5. 펫: LED 알림 상태, LCD 마감 임박 표정·텍스트, TTS "보고서 검토 마감이 2시간 남았습니다." → 대기 복귀.
6. Backend는 전달 사실을 기록해 같은 알림이 다시 나가지 않게 한다.

### 예외 규칙

| 상황 | 처리 |
| --- | --- |
| 이미 완료한 업무 | 완료 시 Eureka의 미완료 Stage가 모두 `done` → 알림 시점에 상태를 재확인하고 알림 생략 |
| 네트워크 단절 | 이벤트를 `미전달`로 저장 → 재연결 시 Eureka 최신 상태·마감 재확인 → 남은 시간을 다시 계산해 전달(예: "1시간 30분 남았습니다") → 미전달 해제 |
| 알림 시간 변경 | 기존 설정 무효화 후 새 시점 저장, 이전 시점에는 알림 없음 |
| 대상 업무 불명확 | "어떤 보고서를 말씀하시는 건가요?"로 재질문, 확정 전에는 아무것도 바꾸지 않음. 시점 미지정이면 기본 60분 |

### 성공 조건 (요약)

- 지정 시점에 미완료 업무를 정확히 탐지하고 사용자 요청 없이 알림이 발생한다.
- 완료 업무는 알림 대상에서 제외되고, 같은 알림이 중복 발생하지 않는다.
- 안내 후 펫이 정상 대기 상태로 돌아온다.

### 검증 현황 (Mock Server, 9/30)

Mock Server(FastAPI, 인메모리 Task)에서 업무 등록 → 마감 조건 판단 → `proactive_alert` 생성 → WebSocket Push →
클라이언트 수신까지 10개 항목이 모두 성공했다. 이벤트 예시는
[[desk-pet-interface-contracts|Desk Pet Interface Contracts]] § proactive_alert. 실제 연동에서는 Mock 데이터 자리만
Eureka Polling으로 바뀐다.

## Connections

- 전체 구조 → [[desk-pet-system-architecture|Desk Pet System Architecture]]
- 알림·Fallback 정책 원문 → [[desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]] (FR-10~15, FR-21)
- 출력 하드웨어 → [[desk-pet-hardware|Desk Pet Hardware]]

## Open Questions

- **Polling 주기 미확정 (needs-update)**: 9/30 김지훈 안은 1분, 중간발표 대본과 Mock Server 기본값(`POLLING_SECONDS=300`)은 5분이다.
- 알림 LED 색이 문서마다 다르다 — 대표 시나리오는 노란색 점멸인데, LED 상태 정의에서 노란색 점멸은 `LISTENING`이고, 네트워크 복구 알림은 파란색 점멸이다. 알림 전용 상태(예: `PROACTIVE_ALERT`)를 따로 정할지 결정이 필요하다.
- 기본 알림값(60분) 적용 책임을 AI가 아닌 Backend 정책으로 옮기자는 의견(10/7 댓글)이 반영 대기 중이다.
- 알림 이력 저장소(Backend DB vs Second Brain)가 아직 둘 다 후보로 적혀 있다.
