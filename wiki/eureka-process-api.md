---
type: tool
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 04 Backend.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 09-16 회의기록.md"
related:
  - "[[wiki/proactive-deadline-alert|Proactive Deadline Alert]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Eureka Process API

## Summary

Eureka Process는 협업 기업 레몬클라우드의 업무(프로세스) 관리 서비스이고, 데스크 펫 프로젝트에서 **업무 데이터의
기준 저장소(source of truth)**다. 업무명·마감·우선순위·Stage·상태가 여기 저장되고, Backend가 HTTP API(`x-api-key`
헤더 인증)로 등록·조회·완료·상태 확인·수정을 호출한다. 9/28 PoC에서 전 기능이 운영 환경에서 정상 동작했고
대부분 1초 안에 응답했다. 다만 **Push/Webhook과 마감 범위 검색이 없어** 마감 감지는 Backend Polling으로 해야 한다.

## Use Cases

| 기능 | Endpoint | 비고 |
| --- | --- | --- |
| 인증 확인 | `GET /_api_/session` | API Key 유효성 |
| 업무 등록 | `POST /_api_/items/0/start` | 입력 `processId`, `taskName`, `dueAt`, `priority` → `itemId`, `stageIds` 반환 |
| 업무 목록 | `GET /_api_/items/0/list?sites&detail=true` | items + stages, 범위 필터 없음 |
| 업무 완료 | `POST /_api_/items/{itemId}/complete` | 미완료 Stage 전부 `done`. `hold` 등은 완료 불가 오류 |
| 상태 조회 | `GET /_api_/items/{itemId}/status` | `todo / doing / done / hold`, `totalStages`, `completedStages` |
| 수정·연기 | `PUT /_api_/items/{itemId}` | `dueAt`, `taskName`, `priority` |

데스크 펫 Backend는 이 형식을 Pi에 직접 노출하지 않고 내부 Adapter(`EurekaTaskAdapter`)를 거쳐 자체 API
(`/api/tasks...`)로 감싼다. 계약 원문은 [[desk-pet-interface-contracts|Desk Pet Interface Contracts]].

### PoC 결과 (Python `requests` + `python-dotenv`, 운영 환경)

| 테스트 | 소요 시간 | 결과 |
| --- | --- | --- |
| session (API Key 확인) | 0.191 / 0.468 s | 성공 |
| Process 목록 | 0.301 ~ 1.135 s | 성공 |
| 업무 등록 | 0.672 s | 성공 |
| 업무 조회 | 0.279 s | 성공 |
| 업무 완료 | 0.230 s | 성공 |
| 상태 조회 | 0.183 s | 성공 |
| 마감 연기 | 0.126 s | 성공 |
| 업무 삭제 | 0.404 s | 성공 |

대부분 0.1~0.7초, 최대는 Process 목록의 약 1.1초다. 정책상 목표 1초·상한 2초 안에 든다.

## Setup Notes

- Base URL `https://api.eureka.codes/flw-v1`, 테스트 Process는 "일반 업무"(`general-work-v1@10204`).
- API Key는 코드·Pi·Git에 넣지 않고 Backend 서버의 `.env`/환경변수에서만 읽는다(NFR-07).
- **제약 1 — 마감 범위 검색 없음**: "1시간 이내/오늘 마감" 조회가 안 되므로 목록을 받아 `dueAt`으로 직접 필터링한다.
- **제약 2 — Push/Webhook 없음**: 상태 변경·마감 도달 이벤트가 오지 않으므로 Polling이 필요하다
  ([[proactive-deadline-alert|Proactive Deadline Alert]]).
- **제약 3 — 상태는 Stage 기반**: 업무 완료 여부는 Item 값이 아니라 하위 Stage 상태로 표현된다. 개별 확인은 status API,
  Polling처럼 대량 확인할 때는 목록의 `stage$$` 값으로 판단하고 필요할 때만 status API를 호출해 요청 수를 줄인다.
- **제약 4 — 오류 코드 불일치**: 입력 검증 오류가 400이 아니라 500으로 올 때가 있다. Status Code와 Response Body의
  오류 메시지를 함께 확인한다.
- 9/16 질문에 대한 답: 기존 Eureka Process용 MCP는 API 형태로 제공 가능하다는 답을 받았다.

## Related Concepts

- [[desk-pet-system-architecture|Desk Pet System Architecture]] — 4계층 중 External / Process Layer
- [[desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]] — FR-05~09, NFR-05·07

## Open Questions

- 업무 위임 시나리오(담당자 지정, "배정됨 → 확인됨")에 필요한 API가 있는지 확인되지 않았다 — 현재 MVP 범위 밖.
- Polling 주기별 호출량이 API 제한에 걸리는지 확인 필요 (needs-update).
