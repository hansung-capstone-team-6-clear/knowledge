---
type: query-output
created: "2026-10-28"
question: "2026-10-28 기준 AI·백엔드 개발은 어디까지 진행됐고, 기업 미팅 전 무엇을 확인해야 하는가?"
sources:
  - "https://github.com/hansung-capstone-team-6-clear/desk-pet-backend/pull/3 — 2026-10-28 머지된 Backend PR"
  - "https://github.com/hansung-capstone-team-6-clear/desk-pet-backend/pull/4 — 2026-10-28 머지된 AI PR"
  - "desk-pet-backend의 feat/backend-eureka-voice, feat/ai-analyze 브랜치 코드와 tests/eval/ 파일 — 2026-10-28 읽기 전용 확인"
  - "사용자가 공유한 실제 Eureka 등록·목록 조회 터미널 출력과 로컬 Swagger 음성 요청 응답"
  - "[[projects/eureka-desk-pet/contracts|1차 인터페이스 계약]]"
---

# 10/28 개발 진행 현황

## Answer

2026-10-28에 Backend PR #3과 AI PR #4가 순서대로 `main`에 머지됐다. 두 PR의 변경 파일은 겹치지 않고 `analyze(audio, text, now, open_tasks, pending)` 호출 및 `schemas.Decision`의 action·필드명은 맞는다. 다만 **머지된 최신 `main`을 새로 실행한 통합 테스트 결과는 아직 확인하지 않았다.** 이전 로컬 Swagger 성공 사례는 머지 전 함께 있던 코드로 수행한 테스트다.

이서현 담당 백엔드는 Eureka 실제 등록·미완료 목록 조회, `create_task`·`list_tasks`·`clarify`, 텍스트·WAV 요청 입구까지 진행됐다. 박시현 담당 AI는 Gemini 호출 함수와 20문항 평가셋·실행기를 작성했다. 완료·마감 변경·알림 설정 실행, 선제 알림, Pi 실물 연결, AI 평가 목표 달성은 남아 있다.

## Evidence

| 항목 | 확인된 상태 | 남은 검증·작업 |
| --- | --- | --- |
| PR 반영 | Backend #3은 2026-10-28 15:48 KST, AI #4는 15:53 KST `main`에 머지 | 최신 `main`에서 함께 설치·실행해 회귀 확인 |
| Eureka 등록·목록 | 실제 Eureka 등록 ID `1000013`이 후속 원본 목록과 백엔드 목록에 나타난 수동 검증 기록 | 등록 직후 목록 반영 시간과 다른 쓰기 동작 검증 |
| 백엔드 요청 | `POST /api/text`, `POST /api/voice`가 `tasks.handle()`에 연결. 정상 WAV를 사용한 로컬 Swagger 응답은 HTTP 200, `status: OK`, `type: task_list` | 최신 `main`에서 텍스트·음성 재실행, Pi 마이크 입력·TTS 검증 |
| 업무 action | `tasks.handle()`은 등록·조회·재질문 처리 | `complete_task`, `update_due`, `set_reminder`의 검증·Eureka/DB 실행 구현 |
| AI 모듈 | `src/desk_pet_backend/services/ai.py`의 비동기 `analyze()` 구현, 공통 `schemas.Decision` 사용 | Gemini 오류·429 대응과 최신 통합 경로 재검증 |
| AI 평가 | `utterances.jsonl` 20건과 `run_eval.py` 준비 | 실제 정답 수·지연 중앙값 측정 및 `ai/eval-YYYY-MM-DD.md` 기록. 목표는 18/20 이상, 중앙값 2초 이하 |
| 선제 알림·실행 환경 | WebSocket 기본 연결·PING/PONG, Dockerfile·docker-compose.yml 존재 | 폴링·미전달 재전송·앱 시작 시 작업 실행, Docker 컨테이너 및 Pi 실물 시연 검증 |

## Notes

- AI 평가셋 `create-02`는 기준 시각 2026-10-08에 이미 지난 2026-09-27 마감을 `create_task` 정답으로 둔다. 백엔드는 과거 마감 등록을 거절하므로 문항의 날짜 또는 기대 동작을 맞춰야 한다.
- AI가 `complete_task`, `update_due`, `set_reminder`를 올바르게 반환해도 현재 백엔드는 이 세 action을 지원하지 않아 오류 응답을 돌려준다. 두 PR의 머지와 전체 기능 완성은 구분한다.
- 2026-10-15의 [[projects/eureka-desk-pet/outputs/2026-10-15-backend-eureka-progress|Eureka 검증]]과 [[projects/eureka-desk-pet/outputs/2026-10-15-feedback-reflection|피드백 반영]] 문서는 당시 상태를 담은 스냅샷이다. 이후 진행은 이 문서로 이어서 본다.
- 이 문서를 작성하면서 실제 Eureka·Gemini 호출, Docker 실행, Pi 실물 테스트는 하지 않았다. API 키와 `.env` 값도 기록하지 않았다.

## Follow-ups

1. **이서현·박시현:** 최신 `main`을 함께 설치해 `/api/text`와 `/api/voice`의 등록·조회·재질문을 재검증하고 오류 응답과 소요 시간을 기록한다.
2. **박시현:** 평가 문항의 과거 마감 불일치를 수정한 뒤 20건을 실행해 정확도·지연·실패 문항을 `ai/eval-YYYY-MM-DD.md`에 기록한다.
3. **이서현:** 완료·마감 변경·알림 설정 action을 기존 Eureka/DB 모듈로 연결하고 성공·모호함·실패를 검증한다.
4. **이서현·하드웨어 담당:** 선제 알림 폴링·WebSocket 전달을 완성하고 Docker와 Pi 실물에서 음성 요청·알림 시연을 검증한다.
