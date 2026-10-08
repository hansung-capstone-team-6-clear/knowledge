# Wiki Index

This index is the entry point for vault queries and ingest deduplication.

## Topics

- [[wiki/topics/hardware|Hardware]]
- [[wiki/topics/knowledge-management|Knowledge Management]]

## Articles

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
