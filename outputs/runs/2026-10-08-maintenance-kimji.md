---
type: run-log
kind: maintenance
run_date: "2026-10-08"
author: kimji
summary: "master 병합 후 데스크펫 중복 노트 정리: 하드웨어 노트 2건을 팀원 노트에 통합·삭제, hardware 토픽을 eureka-desk-pet 하위로 이동"
pr: 4
processed: 2
new_notes: 0
updated_notes: 9
tags: [desk-pet, hardware]
sources:
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/데스크펫 MVP 아키텍처.md"
notes:
  - "[[wiki/desk-pet-hardware|Desk Pet Hardware]]"
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
  - "[[wiki/topics/hardware|Hardware]]"
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
---

# 2026-10-08 maintenance — 데스크펫 중복 노트 통합

## Summary

PR #4(`hardware` → `master`)가 팀원 ingest(#1, #3)와 충돌해 `master`를 병합했다. 병합 후 같은 주제의 노트가 두 벌 생겨, VAULT_RULES의 "기존 노트 갱신 우선" 원칙에 따라 하드웨어 쪽 노트 2건을 팀원 노트에 통합하고 삭제했다.

## Details

- `desk-pet-hardware-components` → `desk-pet-hardware`: 핀맵에 마이크 L/R·LCD 백라이트·PTT 버튼 행 추가, 서보는 MVP 제외 표기. 부팅 설정 신설. 부품표를 10/07 기준(합계 46,020원, 상태 열, 전원·저장장치·배선·레벨 시프터)으로 교체. 온디바이스 표는 초기 조사안으로 남기고 MVP 확정안을 덧붙임. LED 상태 6종으로 갱신. 서보 포함 여부 Open Question 닫음.
- `desk-pet-mvp-architecture` → `desk-pet-system-architecture`: "하드웨어 MVP 아키텍처" 절 신설(4계층 표, 두 흐름, 설계 결정, 스트리밍 전환 기준). 음성 전송·TTS 위치 Open Question을 10/07 확정 내용으로 갱신, 처리 시간 충돌 추가.
- `desk-pet-interface-contracts`: Hardware 행이 9/30 원안임을 밝히고 10/07 변경을 요약, `ledState` 질문 닫음, `proactive_alert` 스키마 질문에 하드웨어 측 선택을 기록.
- 유지한 하드웨어 고유 노트: 인터페이스 7종 상세, 장치 상태, Pi–Backend 규약, MVP 결정 기록, AI–HW 충돌 검토. 삭제 노트로 가던 링크는 통합 대상 노트로 바꿨다(run-log 2건 포함).
- 토픽: `hardware`를 `eureka-desk-pet` 하위로 옮기고 INDEX·TOPIC_MAP에서 중첩 표기, INDEX의 중복 항목 제거.

## Dropped / Issues

- 삭제: `wiki/desk-pet-hardware-components.md`, `wiki/desk-pet-mvp-architecture.md` (내용은 위 노트로 이전, 원문은 git 이력에 남음).
- 팀원 노트 본문의 다른 절(외형, 멘토 권장 사항 등)은 손대지 않았다.
