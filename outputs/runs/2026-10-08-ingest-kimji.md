---
type: run-log
kind: ingest
run_date: "2026-10-08"
author: kimji
summary: "데스크펫 하드웨어 정리안 PDF·MVP 아키텍처 HTML 2건 → Hardware 토픽 신설, wiki 노트 6건 생성"
pr:
processed: 2
new_notes: 6
updated_notes: 0
tags: [desk-pet, hardware]
sources:
  - "raw/pdf/데스크펫 하드웨어 정리안 (MVP).pdf"
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/데스크펫 MVP 아키텍처.md"
notes:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-hardware|Desk Pet Hardware]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-device-states|장치 상태]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]"
  - "[[wiki/topics/hardware|Hardware]]"
---

# 2026-10-08 ingest — 데스크펫 하드웨어 문서

## Summary

사용자가 직접 건넨 팀 문서 2건(하드웨어 정리안 PDF 12쪽, MVP 아키텍처 HTML)을 처리했다. `wiki/topics/hardware` 토픽을 새로 만들고 데스크펫 하드웨어 노트 6건을 작성했으며, `wiki/INDEX.md`·`wiki/TOPIC_MAP.md`에 등록했다.

## Details

- PDF: `pdf2md-ingest` 절차를 따랐다. 원본은 `raw/pdf/`에 이름 그대로 보존(sha256 frontmatter 고정). 이 머신에 `pdftoppm`·`uv`가 없어 S2(pymupdf4llm)·S4(페이지 렌더 전사)를 쓰지 못했고, poppler `pdftotext`로 텍스트를 뽑은 뒤 표를 수동 재구성했다(`converted_by: S2`, `conversion_note`에 방식 명시). 원문의 취소선·형광 표기가 겹쳐 셀이 섞인 표는 HTML 문서와 대조해 복원했다. 인라인 멘션 칩은 추출되지 않아 `[칩]`으로 남겼다.
- HTML: 대응 raw 레인이 없어 Markdown으로 옮겨 raw 루트에 두었다(clipper 7키 frontmatter, `source`는 로컬 팀 문서 표기). SVG 다이어그램은 노드·화살표를 텍스트로 전사했다. HTML 원본 파일 자체는 보존하지 않았다.
- Clippings 단계는 거치지 않고 변환물을 바로 raw/에 두었다. 기존 `Clippings/`의 미처리 1건은 이번 요청 범위가 아니라 건드리지 않았다.
- 노트 설계: 한 개념 한 파일 — 아키텍처, 부품·핀맵·예산, 인터페이스 7종, 장치 상태, Pi–Backend 규약, 결정 기록.
- 개인 데이터: 원문 알림 예시의 팀원 식별값(`user_id`, 호칭)은 wiki 노트에서 자리표시자로 바꿨다. raw 사본은 원문 그대로 둔다.

## Dropped / Issues

- `needs-update` 표시: Backend 합의 2건(`POST /api/voice` 응답 형식, Edge-TTS 방식·`request_id`), 디자인 색 확정, LED 링 치수, 사운드카드 실물 테스트, Edge-TTS 합성 시간 실측.
- PR 미생성: `gh` 미인증, git `user.name`/`user.email` 미설정으로 커밋·PR은 사용자 확인 후 진행.
