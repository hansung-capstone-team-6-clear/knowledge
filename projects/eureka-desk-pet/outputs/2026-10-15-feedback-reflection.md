---
type: query-output
created: "2026-10-15"
question: "10/01 기업 미팅의 개발 피드백과 10/08 기술 검토를 MVP 설계와 구현에 어떻게 반영했는가?"
sources:
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
  - "[[projects/eureka-desk-pet/outputs/2026-10-08-tech-review|10/08 기술 검토]]"
  - "[[projects/eureka-desk-pet/architecture|1차 아키텍처]]"
  - "[[projects/eureka-desk-pet/contracts|1차 인터페이스 계약]]"
  - "[[projects/eureka-desk-pet/outputs/2026-10-15-backend-eureka-progress|백엔드 Eureka 검증 결과]]"
---

# 개발 피드백 반영 현황

## Answer

**10/01 기업 미팅 기록**은 기본 기능의 안정적 구현, 범위 축소, 파트 간 JSON 인터페이스와 담당 영역 구조 다이어그램 정리를 요구했다. **“기존 AI·백엔드 구조가 과하다”는 평가는 10/08 기술 검토 문서의 판단**이다. 두 출처를 같은 멘토 발언으로 합치지 않는다.

이를 바탕으로 10/08에 `architecture.md`에 단순화한 MVP 구조를 **제안**하고, `contracts.md`에 JSON 형식을 **초안**으로 정리했다. 설계 문서 작성과 팀 합의·실제 구현 완료는 서로 다른 단계다. 현재 백엔드에서 실제로 확인된 범위는 Eureka 업무 1건의 등록·목록 조회다.

## Evidence

| 출처와 피드백 | 문서에 반영한 내용 | 구현·검증 상태 |
| --- | --- | --- |
| 10/01: 기본 기능 우선, 구현 범위 축소 | `architecture.md`에서 1차 MVP와 2차 고도화를 구분하고, 업무 등록·조회·완료와 선제 알림을 중심 흐름으로 정리 | Eureka 등록·목록 조회만 실제 API로 수동 검증. 음성·완료·알림은 미완료 |
| 10/01: 파트 간 JSON 인터페이스 정리 | `contracts.md`에 Pi ↔ Backend, Backend ↔ AI, Backend ↔ Eureka 형식과 필드명 정리 | 문서는 초안. 실제 `/api/text` 전체 흐름은 AI 모듈 미구현으로 미검증 |
| 10/01: 담당 영역의 구현 수준 구조 다이어그램 | `architecture.md`에 실행 단위·모듈 책임·요청 및 알림 흐름을 텍스트로 정리 | 담당 영역별 최종 도식과 팀 검토 여부는 별도 확인 필요 |
| 10/08 기술 검토: AI·백엔드 구조 과다 | 경량 Intent → LLM → MCP의 다단계를 1차 제안에서 제외하고, AI 1회 JSON 판단 → Backend 검증·실행으로 단순화. 백엔드는 FastAPI 1개 프로세스의 기존 모듈을 사용 | 제안 문서와 코드 뼈대는 있으나 AI 연결 및 전체 시나리오는 미완료 |

10/01 기록에는 Local LLM의 역할·운영 방식 검토와 Tool/Skill 활용 제안도 있다. 1차에서 Local LLM·MCP를 제외한 것은 **10/08 기술 검토 후 작성한 MVP 제안**이며, 멘토의 확정 지시나 팀 최종 합의로 표현하지 않는다.

## Notes

- `architecture.md`는 2026-10-08 **제안**, `contracts.md`는 **초안**으로 표시돼 있다. 채택 여부와 모델 선택은 팀 확인이 필요하다.
- `contracts.md`의 필드명이나 책임 경계를 바꿀 때는 해당 문서를 먼저 검토해야 한다. 이 결과 문서가 인터페이스 계약을 대신하지 않는다.
- 10/01 미팅은 성능 기준을 우선 구현 후 실측으로 개선하라고 했다. 실제 음성 요청의 처리 시간은 아직 측정하지 않았다.
- 2026-10-15의 실제 Eureka 검증 범위와 남은 작업은 [[projects/eureka-desk-pet/outputs/2026-10-15-backend-eureka-progress|백엔드 Eureka 검증 결과]]에 기록했다.

## Follow-ups

1. 팀과 1차 MVP 범위, `architecture.md`, `contracts.md` 초안을 검토해 확정 여부를 기록한다.
2. 백엔드와 AI 담당자가 `Decision` 필드·오류 응답을 맞추고 `/api/text` 전체 경로를 검증한다.
3. 담당 영역 구조 다이어그램과 실제 구현 결과가 일치하는지 확인한 뒤, 음성·완료·알림 순서로 남은 기능을 검증한다.
