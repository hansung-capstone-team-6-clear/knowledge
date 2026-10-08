---
type: run-log
kind: ingest
run_date: "2026-10-08"
author: sensior101
summary: "Desk Pet 백엔드 아키텍처 개정 다이어그램(archify HTML) 1건을 raw 캡처로 추출, 기존 desk-pet wiki 3편에 개정 사항 반영"
pr: 3
processed: 1
new_notes: 0
updated_notes: 3
tags: [desk-pet, capstone, architecture]
sources:
  - "raw/desk-pet-backend-architecture.md"
notes:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
  - "[[wiki/proactive-deadline-alert|Proactive Deadline Alert]]"
---

# 2026-10-08 ingest — sensior101

## Summary

사용자가 직접 지정한 archify 3.0.1 HTML 다이어그램(`backend_architecture.html`) 1건을 처리했다.
`Clippings/` 경유가 아닌 단건 요청이라 `Clippings/`의 기존 클리핑은 이번 실행 범위에서 제외했다.
결과: `raw/desk-pet-backend-architecture.md`(추출 캡처) 신규, 기존 wiki 3편 갱신.

## Details

- 원본 HTML은 약 820KB이며 대부분이 임베디드 JetBrains Mono 폰트와 archify 뷰어 JS다. raw/에는 HTML 레인이
  없으므로 원본 바이너리를 보관하지 않고, 의미 콘텐츠만 기계 추출한 Markdown 캡처를 레인 2(repo-doc 스냅샷)의
  capture header 형식으로 남겼다(`commit`/`branch`는 해당 없음 → `n/a`).
- 추출 대상: SVG `<text>` 문자열, `<path data-edge-*>` 속성(엣지 23개), Notes 패널, 임베디드
  `desk-pet-interface-contracts` JSON(계약 21개 + reminderPolicy). 값은 원문 그대로이고 뷰어 UI 문자열은 제외했다.
- 처음에는 `wiki/desk-pet-backend-architecture.md`와 `wiki/topics/desk-pet.md`를 신설했으나, 같은 날 먼저 머지된
  riyunkim ingest(PR #1, `eureka-desk-pet` 토픽 7편)와 겹쳐 PR #3이 충돌했다. master를 병합하면서 중복 노트·토픽을
  제거하고, 이 원문을 기존 `raw/Capstone Eureka Process - 02 desk-pet-architecture 다이어그램.md`의 **개정판**으로 보고
  차이만 기존 노트에 반영했다:
  - System Architecture: LLM Service·MCP Client 분리, Orchestrator 검증, `ReminderRepositoryPort`·`DatabaseAdapter`·
    `MockTaskAdapter` 제거(영속화 미확정), HTTP 응답/WebSocket push 분리, Pi의 Edge-TTS.
  - Interface Contracts: 세 번째 `proactive_alert` 형식(`request_id`, `led` 없음), `SendOutputCommand` push 전용화,
    개정판 계약 구성 차이.
  - Proactive Deadline Alert: Scheduler → ReminderService 컴포넌트 경로, 기본 60분이 ReminderService 정책으로 명시됨.
- `docs/raw-index.*` 재생성.

## Dropped / Issues

- 원본 HTML(인터랙티브 뷰어) 자체는 vault에 보관하지 않았다.
- 새 needs-update: `proactive_alert` 스키마 3종 공존, 알림 저장소 미확정, TTS 위치(서버 vs Pi Edge-TTS) 불일치.
- `gh` CLI 미인증 상태에서 시작해 author slug는 `git config user.name`에서 얻었다(이후 로그인 확인, gh login 동일).
