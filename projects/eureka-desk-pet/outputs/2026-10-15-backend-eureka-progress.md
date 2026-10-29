---
type: query-output
created: "2026-10-15"
question: "현재 백엔드의 Eureka 업무 등록·목록 조회는 어디까지 구현하고 검증했는가?"
sources:
  - "2026-10-15 사용자가 공유한 터미널 출력: Eureka 등록 응답 ID 1000013, 이후 원본 목록과 백엔드 목록에서 동일 ID 확인"
  - "desk-pet-backend 로컬 작업 트리: src/desk_pet_backend/eureka.py, services/tasks.py, api/voice.py, db.py, tests/test_smoke.py"
  - "[[projects/eureka-desk-pet/contracts|1차 인터페이스 계약]]"
---

# 백엔드 Eureka 등록·조회 검증 결과

## Answer

2026-10-15에 **사용자가 현재 백엔드의 `EurekaClient`로 실제 Eureka Process API에 테스트 업무를 등록하고, 같은 ID가 목록에 나오는 것까지 확인했다.** 근거는 사용자가 공유한 터미널 출력이다. 등록 응답 ID는 `1000013`이었다. 이후 Eureka 원본 목록에서 해당 ID와 상태 `todo`를 확인했고, 백엔드의 `list_open_tasks()`에서도 같은 ID를 확인했다.

이 결과는 **Eureka 등록·목록 조회 경로의 수동 검증**이다. `/api/text`를 통한 AI 포함 전체 흐름이나 `/api/voice` 음성 흐름의 성공을 뜻하지 않는다.

## Evidence

| 확인 항목 | 2026-10-15 결과 | 범위 |
| --- | --- | --- |
| 연결·목록 조회 | 실제 Eureka에서 미완료 업무 3건 조회 | 테스트 키와 설정으로 목록 API 호출 성공 |
| 업무 등록 | `EurekaClient.create_task()`가 테스트 업무를 등록하고 ID `1000013` 반환 | 실제 Eureka에 업무 1건 생성 |
| 등록 직후 목록 | 바로 이어진 `list_open_tasks()`에서는 해당 ID가 나오지 않음 | 이 시점만으로 실패 여부를 판단할 수 없음 |
| 이후 원본 목록 | 총 5건 중 ID `1000013` 존재, 상태 판정 `todo` | Eureka 응답의 `list`와 `stage$$`를 확인 |
| 이후 백엔드 목록 | `list_open_tasks()`에서 ID `1000013` 확인 | 응답 필드·마감 시각·상태의 `Task` 변환까지 통과 |
| 자동 테스트 | 로컬 `pytest` 14개 통과 | 실제 서버가 아닌 모의 Eureka·모의 HTTP 응답 기반 |

실제 API 테스트에는 `src/desk_pet_backend/eureka.py`의 `EurekaClient`를 직접 호출했다. 로컬 8001 포트의 `mocks/mock_server`는 사용하지 않았다. 등록 직후 목록에 없었던 원인은 아직 확인되지 않았다. **목록 반영 지연은 가능한 설명이지만 확정된 원인은 아니다.**

## Notes

- 백엔드의 변경 파일은 `eureka.py`, `services/tasks.py`, `api/voice.py`, `db.py`, `tests/test_smoke.py`, `pyproject.toml`이다. 2026-10-15 현재 **로컬 미커밋 변경**이며 팀 백엔드 저장소에 반영된 코드로 취급하면 안 된다.
- 구현된 업무 흐름은 `create_task`에 한정된다. `services/ai.py`의 정식 `analyze()`는 미구현이어서 `/api/text`의 실제 사용자 입력 시연은 아직 불가능하다. `/api/voice`는 501을 반환한다.
- 완료·마감 변경·상태 조회, 100건 이상 목록 페이지 처리, 요청 재시도 시 중복 등록 방지는 남아 있다.
- API 키와 `.env`의 값은 이 문서에 기록하지 않았다. 기존 9월 PoC 결과는 [[wiki/eureka-process-api|Eureka Process API]]에 별도로 정리돼 있다.
- 이 문서를 작성하면서 실제 Eureka API 호출을 다시 실행하지 않았다.

## Follow-ups

1. AI 담당자와 `Decision` 출력 계약을 맞추고 `/api/text`에서 등록까지 끝까지 검증한다.
2. 실제 Eureka 목록의 등록 직후 가시성을 재측정하고, 필요하다면 등록 확인·재시도 정책을 정한다.
3. 업무 조회·완료를 구현한 다음 `/api/voice`와 선제 알림을 연결한다.
