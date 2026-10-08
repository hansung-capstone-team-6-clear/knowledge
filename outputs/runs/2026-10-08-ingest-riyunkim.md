---
type: run-log
kind: ingest
run_date: "2026-10-08"
author: riyunkim
summary: "캡스톤 데스크 펫 Notion export 2건(12개 원문) ingest — wiki 7편·토픽 1·프로젝트 README 신설"
pr:
processed: 12
new_notes: 7
updated_notes: 0
tags: [capstone, desk-pet, notion-export]
sources:
  - "raw/Capstone Eureka Process.md"
  - "raw/Capstone Eureka Process - 01 기획.md"
  - "raw/Capstone Eureka Process - 02 시스템 설계.md"
  - "raw/Capstone Eureka Process - 02 desk-pet-architecture 다이어그램.md"
  - "raw/Capstone Eureka Process - 03 AI & 데이터.md"
  - "raw/Capstone Eureka Process - 04 Backend.md"
  - "raw/Capstone Eureka Process - 05 Raspberry Pi.md"
  - "raw/Capstone Eureka Process - 09-16 회의기록.md"
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
  - "raw/Capstone Eureka Process - 작업 트래커.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/LLM Wiki - AI 데스크 펫 프로젝트.md"
notes:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/proactive-deadline-alert|Proactive Deadline Alert]]"
  - "[[wiki/intent-routing-and-structured-output|Intent Routing & Structured Output]]"
  - "[[wiki/eureka-process-api|Eureka Process API]]"
  - "[[wiki/desk-pet-hardware|Desk Pet Hardware]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
  - "[[wiki/desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]]"
---

# Ingest 2026-10-08 — 캡스톤 데스크 펫 Notion export

## Summary

팀 Notion 워크스페이스에서 내려받은 export zip 2건(`Capstone Eureka Process` 트리, `Eureka Process 연동형 업무관리 AI
데스크 펫 - 프로젝트 정리` 트리)을 Markdown으로 변환해 12개 원문을 `raw/`에 보존하고, 프로젝트 지식을 wiki 7편으로
컴파일했다. 새 토픽 `AI Desk Pet (Capstone)`과 프로젝트 노트 `projects/eureka-desk-pet/README.md`를 만들었다.

## Details

- **변환**: Notion HTML export를 표준 라이브러리 HTML 파서로 Markdown 변환(표·코드 블록·인라인 댓글 유지). 파일명은
  페이지 제목 기준, emoji 제거, `09/16` 같은 날짜의 `/`는 `-`로 바꿨다. 레인 1(루트 `*.md`) clipper 7키 frontmatter를
  붙이고 `source`에 Notion 페이지 경로를 적었다. `Clippings/` 시작 커밋 후 `raw/`로 이동했다.
- **정리 규칙**: 개인 이메일 주소는 `(이메일 제거)`로 치환, 원격 아바타 이미지 URL·Notion 빈 링크 제거, 쇼핑몰 링크의
  추적 쿼리 제거, 로컬 페이지 링크는 텍스트만 남김. 작업 트래커는 CSV(데이터베이스) + 내용이 있는 하위 페이지 5개를 한
  파일로 묶었고, Notion 기본 템플릿만 있는 하위 페이지는 생략했다.
- **아키텍처 다이어그램**: `desk-pet-architecture.html`(인터랙티브 뷰어)은 노드·계층·연결 라벨만 텍스트로 추출해
  별도 원문으로 보존했다.
- **노트 설계**: 재사용 가능한 설계 지식은 `wiki/`(아키텍처, 대표 시나리오 패턴, AI 라우팅 패턴, Eureka API, 하드웨어,
  인터페이스 계약, 요구사항·정책), 팀 구성·회의 로그·역할별 과제는 `projects/eureka-desk-pet/`로 분리했다.
  중복 원문(초기 "프로젝트 정리" 페이지가 01~05 문서의 초안을 대부분 포함)은 최신 문서를 우선하고 차이만 Open Questions에 남겼다.
- **중복 게이트**: 기존 `raw/`가 비어 있어 source 중복 없음. 기존 wiki 문서 없음(`wiki/Hardware`는 팀원이 만든 빈 파일로
  보여 손대지 않았다).

## Dropped / Issues

- **제외한 첨부**: `deskpet_mock_server.zip`, `Eureka_Process_API_Test.zip`(코드 — GitHub 저장소가 정본),
  이미지 PNG/JPG, `second_brain_실습 (1).pdf`와 `week2~4` HTML(레몬클라우드 교육 자료 — 프로젝트 문서가 아니고 외부 자료라 제외).
- **보안 주의**: `Eureka_Process_API_Test.zip`에 실제 값이 들어 있는 `.env`(`EUREKA_API_KEY`)가 포함돼 있다. vault에는
  넣지 않았지만 Notion 첨부로 공유돼 있으므로 키 교체와 첨부 정리가 필요하다.
- **needs-update**: Polling 주기(1분 vs 5분), AI 출력 필드명 통일, `proactive_alert` 스키마 이원화, 알림 LED 색 충돌,
  10/01 범위 축소의 요구사항 반영, LCD 크기 표기(1.69" vs 2.0"), 배포 형태(Lambda) — 각 노트 Open Questions 참고.
- `Clippings/`에 있던 기존 클리핑 2건(`Hermes Agent Masterclass.md`, `드디어 나에게 딱 맞았던 ...md`)은 이번 요청 범위가 아니라 처리하지 않았다.
- `templates/team-settings.yaml`의 `github.default_reviewer`가 템플릿 값(`steve-lemon`)이라 이 저장소 협업자가 아니다 — PR 리뷰어 미지정.
