---
type: run-log
kind: ingest
run_date: "2026-10-08"
author: sensior101
summary: "Desk Pet 백엔드 아키텍처 HTML(archify) 1건을 raw 캡처로 추출, wiki 노트 1개와 Desk Pet 토픽 신설"
pr:
processed: 1
new_notes: 1
updated_notes: 0
tags: [desk-pet, architecture]
sources:
  - "raw/desk-pet-backend-architecture.md"
notes:
  - "[[wiki/desk-pet-backend-architecture|Desk Pet Backend Architecture]]"
  - "[[wiki/topics/desk-pet|Desk Pet]]"
---

# 2026-10-08 ingest — sensior101

## Summary

사용자가 직접 지정한 archify 3.0.1 HTML 다이어그램(`backend_architecture.html`) 1건을 처리했다.
`Clippings/` 경유가 아닌 단건 요청이라 `Clippings/`의 기존 클리핑 2건은 이번 실행 범위에서 제외했다.
결과: `raw/desk-pet-backend-architecture.md`(추출 캡처), `wiki/desk-pet-backend-architecture.md`(신규, draft),
`wiki/topics/desk-pet.md`(신규 루트 토픽), `wiki/INDEX.md`·`wiki/TOPIC_MAP.md` 갱신.

## Details

- 원본 HTML은 약 820KB이며 대부분이 임베디드 JetBrains Mono 폰트와 archify 뷰어 JS다. raw/에는 HTML 레인이
  없으므로 원본 바이너리를 보관하지 않고, 의미 콘텐츠만 기계 추출한 Markdown 캡처를 레인 2(repo-doc 스냅샷)의
  capture header 형식으로 남겼다(`commit`/`branch`는 해당 없음 → `n/a`).
- 추출 대상: SVG `<text>` 문자열, `<path data-edge-*>` 속성(엣지 23개), Notes 패널, 임베디드
  `desk-pet-interface-contracts` JSON(계약 21개 + reminderPolicy). 값은 원문 그대로이고 뷰어 UI 문자열은 제외했다.
- 기존 wiki 문서가 없어 중복 게이트 해당 없음. 노트 타입은 아키텍처이므로 `concept`.
- 개인 데이터·자격증명 없음(엔드포인트 경로와 계약 스키마만 포함).

## Dropped / Issues

- 원본 HTML(인터랙티브 뷰어) 자체는 vault에 보관하지 않았다.
- wiki 노트 Open Questions: 중복 알림 저장소 미확정, `proactive_alert` 재처리 정책 미정의, MCP 읽기 도구 범위
  불명확(inference), 엣지 key 11·12·17 공백(needs-update).
- `gh` CLI 미인증으로 author slug는 `git config user.name`에서 얻었다.
