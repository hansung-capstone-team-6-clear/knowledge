# Wiki Index

This index is the entry point for vault queries and ingest deduplication.

## Topics

- [[wiki/topics/knowledge-management|Knowledge Management]]
- [[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]
- [[wiki/topics/hardware|Hardware]]

## Articles

### AI Desk Pet (Capstone)

- [[wiki/desk-pet-system-architecture|Desk Pet System Architecture]] — Edge–Backend 4계층 구조와 Port/Adapter Backend
- [[wiki/proactive-deadline-alert|Proactive Deadline Alert]] — 대표 시나리오: Polling 기반 마감 임박 선제 알림
- [[wiki/intent-routing-and-structured-output|Intent Routing & Structured Output]] — 경량 Intent → LLM 2단계 라우팅, JSON 출력, LLM 금지 사항
- [[wiki/eureka-process-api|Eureka Process API]] — 업무 기준 저장소 API, PoC 지연 시간, 제약 4가지
- [[wiki/desk-pet-hardware|Desk Pet Hardware]] — Raspberry Pi 4B 핀맵, 부품·예산, 온디바이스 vs 클라우드
- [[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]] — 계층 간 JSON 인터페이스 색인, WebSocket 이벤트
- [[wiki/desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]] — FR/NFR, 응답 시간 예산, 처리 정책, 구현 범위

### Hardware

- [[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]] — 4계층 구조, 음성(HTTP)·알림(WebSocket) 두 흐름, 설계 결정
- [[wiki/desk-pet-hardware-components|부품·핀맵·예산]] — Pi 4B 핀맵, 부팅 설정, BOM과 예산
- [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]] — Pi 쪽 7개 인터페이스 요약
- [[wiki/desk-pet-device-states|장치 상태]] — 6개 상태, LED/LCD 표시, 전이 규칙
- [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]] — WebSocket 이벤트, `POST /api/voice`
- [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]] — 2026-10-07 변경 16건과 남은 결정
- [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] — AI 처리 구조 문서와의 충돌 13건(필수 5 · 결정 필요 5)

## Maintenance

- Update this file after ingest creates or changes wiki articles.
- Prefer linking canonical wiki notes here rather than raw source files.
- Source provenance belongs in each note's `sources` frontmatter as `"raw/<file>.md"` strings.
