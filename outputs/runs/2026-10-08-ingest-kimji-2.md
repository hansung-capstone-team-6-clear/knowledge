---
type: run-log
kind: ingest
run_date: "2026-10-08"
author: kimji
summary: "AI–HW 충돌 검토 엑셀 1건 → 충돌 검토 노트 신설, 하드웨어 노트 5건에 충돌 항목 교차 링크. PDF·HTML은 기존 원본과 동일해 생략"
pr:
processed: 1
new_notes: 1
updated_notes: 5
tags: [desk-pet, hardware]
sources:
  - "raw/xlsx/deskpet_AI_HW_충돌검토.xlsx"
  - "raw/deskpet_AI_HW_충돌검토.md"
notes:
  - "[[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-device-states|장치 상태]]"
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]"
---

# 2026-10-08 ingest 2 — AI–HW 충돌 검토표

## Summary

사용자가 PDF·HTML·XLSX 3건을 다시 건넸다. PDF는 `raw/pdf/`의 보존 원본과 바이트 단위로 같고 HTML도 같은 날 1차 실행(`outputs/runs/2026-10-08-ingest-kimji.md`)에서 처리돼 중복 게이트로 건너뛰었다. 새 자료인 `deskpet_AI_HW_충돌검토.xlsx`(2시트, 충돌 13건)를 처리해 충돌 검토 노트를 만들고 기존 하드웨어 노트에 해당 항목을 연결했다.

## Details

- XLSX 원본은 `raw/xlsx/`에 이름 그대로 보존하고 `source_xlsx`·`source_sha256`으로 변환 MD와 짝지었다. 이 머신에 `openpyxl`이 없어 Python 표준 라이브러리(zipfile+XML)로 셀 값을 읽었다(`converted_by: manual`). 수식 셀은 계산값과 수식을 함께 남겼다.
- 신규 노트 `desk-pet-ai-hw-conflict-review`: 필수 5 · 결정 필요 5 · 충돌 없음 3, 원본 기준 전부 미착수.
- 갱신 노트 5건: `sources`·`related`에 충돌 검토를 더하고 해당 충돌 번호를 Connections/Open Questions에 연결(처리 시간 16초 vs 10초, `conversationId`·`clarification`, `+09:00`, 재질문 LCD 표정).
- 비교 상대인 "AI 처리 구조 문서"는 vault에 없어 링크하지 못했다.

## Dropped / Issues

- 중복으로 생략: `데스크펫 하드웨어 정리안 (MVP).pdf`, `데스크펫 MVP 아키텍처.html`.
- 엑셀 상태 열이 갱신되면 충돌 검토 노트를 다시 맞춰야 한다(needs-update).
- push·PR은 1차 실행과 같은 사유(`gh` 미인증, 리뷰어 설정이 템플릿 값)로 보류.
