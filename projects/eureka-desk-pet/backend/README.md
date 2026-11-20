# Backend 레이어

- 코드: `hansung-capstone-team-6-clear/desk-pet-backend`
- 계약: `../contracts.md` (전체)
- 구조: `../architecture.md`

## 레포 구조

`hansung-capstone-team-6-clear/desk-pet-backend#1`에서 셋업했다. 폴더별 역할은 레포의 README가 정본이다.

```text
src/desk_pet_backend/
├── main.py      앱 진입점, /health
├── config.py    환경 변수
├── schemas.py   contracts.md를 코드로 옮긴 Pydantic 모델
├── api/         ① 입구: /api/voice, /api/text, GET /api/tasks (Controller 역할)
├── services/    ② 규칙·흐름: tasks.py, task_cache.py(업무 목록 캐시), ai.py
├── eureka.py    ③ 바깥 연결: EurekaClient(real) + EurekaMock(mock)
└── db.py        ③ 바깥 연결: SQLite (reminders · pending)
tests/           자동 테스트, eval/ 평가셋, integration/ PoC 스크립트
mocks/, docs/    이전 단계 참고용 (1차에서 안 씀)
```

- 호출 방향은 `api/ → services/ → eureka.py, db.py` 한쪽뿐이다.
- 이전 알림 설계의 코드(`services/reminder.py`, `api/ws.py`의 알림 전송, `alerts` 테이블)는 삭제했다 (PR #41). `api/ws.py`는 Hardware 앱이 아직 연결하므로 `connected`·`PING`/`PONG`만 남겼고, Pi가 연결을 끊으면 지운다.
- MVC를 그대로 쓰지 않는 이유: 화면(View)이 없는 API 서버다. 응답 JSON 모양(`schemas.py`)이 View에 가장 가깝다.
- 기능별 폴더 대신 역할별 폴더를 쓰는 이유: 기능이 많지 않다. 5개 이상이 되면 기능별 폴더를 검토한다.
- 헥사고날 Port는 쓰지 않는다. 교체가 필요한 곳은 Eureka(real/mock) 하나이고, `eureka.py`의 `get_eureka()`로 충분하다.

셋업 PR에서 함께 정리한 것: 빈 헥사고날 폴더 삭제, `.env.example`을 `EUREKA_API_KEY`(x-api-key) 기준으로 변경,
`pyproject.toml` BOM 제거(설치 실패 원인), MCP·OpenAI·APScheduler 의존성 제거, `mocks/`·`docs/`를 참고용으로 표시.

## 2026-11-06 결정의 진행 상황

형식과 규칙은 `../contracts.md` § 1·6이 정본이다. 같은 날 결정한 업무 목록 캐시와 `GET /api/tasks`는 구현됐다 (아래 2026-11-11 현황).

| 변경 | 내용 | 상태 |
| --- | --- | --- |
| 시간 제한 | 처리 전체 7초 → 쓰기 전까지 15초 + Eureka 쓰기 3초. **Pi 20초 변경과 함께 배포한다** | Backend 구현 (PR #39), Pi 구현 전 |
| 쓰기 도중 시간 초과 (제안) | 처리됐는데 실패로 답하지 않게 "처리 여부를 확인하지 못했어요"로 답하고 `pending`을 지운다 | 미구현. 지금은 일반 오류로 답하고 목록을 바로 다시 읽는다 |
| 코드 삭제 | `reminder.py`, `ws.py`의 알림 전송, `alerts` 테이블, `POLL_INTERVAL_SECONDS` | 완료 (PR #41) |

## 구현 현황 (2026-11-20)

- 처리 제한을 쓰기 전까지 15초, Eureka 쓰기 3초로 나눴다 (PR #39). Pi의 20초 실패·10초 지연 안내는 하드웨어 쪽 구현 전이다.
- `set_reminder`를 실행한다 (PR #40). 알림 간격을 SQLite에 저장하고 `reminder_set`으로 답하며, `GET /api/tasks`에 바로 반영된다.
- 이전 선제 알림 코드를 삭제했다 (PR #41).
- 삭제 확인 답("응", "아니")의 판정을 AI의 `confirm`으로 바꿨다 (PR #42). Backend는 답이 같은 업무에 대한 것인지만 본다.
- **PR #43 (리뷰 중)**: `list_tasks`가 AI가 계산한 `date_from`·`date_to`로 업무를 거른다(Backend는 날짜를 계산하지 않음). 알림 켜기·끄기를 `reminder_enabled`와 `reminders.enabled` 열로 추가한다.
  AI 프롬프트가 바뀌기 전까지는 목록이 항상 전체로 나오고 알림 끄기는 되묻는다.

## 구현 현황 (2026-11-11)

- `GET /api/tasks`와 업무 목록 캐시를 구현해 서버에 배포했다 (PR #36, #37). 실제 Eureka로 등록·마감 변경·완료 직후 목록에 반영되는 것과, 완료한 업무가 약 10초 뒤 재조회 뒤에도 빠져 있는 것을 확인했다.
- 텍스트 요청 5건에서 목록 조회 시간은 0ms였다 (11/06에는 약 1.3초). 완료 1건은 Gemini가 6.4초 걸려 처리 시간이 7.0초였다. 7초 제한 조정이 여전히 필요하다.
- 음성 삭제(확인 질문 포함)는 Backend 처리만 있다 (PR #34). AI 프롬프트가 `delete_task`를 내보내야 동작한다.

## 구현 현황 (2026-11-06)

- `complete_task`와 `update_due`도 `tasks.handle()`에서 실행한다. `set_reminder`는 AI 출력 계약에 있지만 아직 지원하지 않는다.
- 등록·조회·수정·완료·재질문을 실제 서버, Gemini, Eureka에서 텍스트와 녹음 파일로 시험했다. Pi 마이크·TTS를 포함한 실물 흐름은 아직이다.
- 새 업무 이름 검증과 재질문 발화 보존은 [PR #15](https://github.com/hansung-capstone-team-6-clear/desk-pet-backend/pull/15)에서 리뷰 중이다.

## 구현 현황 (2026-10-28)

- [Backend PR #3](https://github.com/hansung-capstone-team-6-clear/desk-pet-backend/pull/3)이 `main`에 머지됐다. 실제 Eureka 업무 등록·미완료 목록 조회와 `tasks.handle()`의 `create_task`, `list_tasks`, `clarify` 경로가 구현됐다. 실제 Eureka에 등록한 업무 ID가 후속 목록에 나타나는 수동 검증도 했다.
- `POST /api/text`와 `POST /api/voice`가 `tasks.handle()`을 호출한다. 로컬 Swagger에서 정상 WAV 1건이 HTTP 200, 응답 본문 `status: OK`, `type: task_list`로 처리된 사례가 있다. 이는 Pi 마이크·TTS를 포함한 실물 전체 흐름 검증은 아니다.
- AI PR #4도 `main`에 머지됐다. 두 PR이 합쳐진 최신 `main`의 텍스트·음성 경로는 별도 재검증이 필요하다. 세부 근거와 담당별 다음 작업은 [[projects/eureka-desk-pet/outputs/2026-10-28-development-progress|10/28 개발 진행 현황]]을 본다.

## 구현 체크리스트 — Mock에서 실제 Eureka로

1. `GET /api/tasks`는 `status != done`인 업무를 준다. Mock v2는 `doing`만 대상이라 실제 Eureka에서 `todo`인 업무의 알림이 빠진다. `hold` 제외 여부는 팀이 정한다.
2. 알림 간격은 메모리가 아니라 SQLite에 둔다. 재시작하면 알림 설정이 사라진다.
3. 목록 조회 `limit=100` → 100개를 넘으면 다음 페이지까지 읽는다.
4. 입력 검증 오류가 500으로 올 수 있으므로 상태 코드와 응답 본문을 함께 확인한다.
5. 시간 변환(epoch ms ↔ +09:00)은 `eureka.py`에서만 한다. 다른 곳에서 변환하면 알림이 9시간 어긋난다.

## 응답 시간

2026-11-06 실제 서버 측정이다 (21:10~21:21, 58건, 성공 건 중앙값). 음성은 6건뿐이고, 네트워크는 Pi가 아니라 개발 PC에서 쟀다. Gemini는 혼잡할 때 훨씬 느려진다.

| 구간 | 텍스트 | 음성 |
| --- | --- | --- |
| 업로드·네트워크 | 0.9초 | 1.7초 |
| Eureka 목록 읽기 | 1.3초 | 1.3초 |
| Gemini | 1.7초 | 3.2초 |
| Eureka 쓰기 | 등록 1.1초, 수정·완료 0.5~0.6초 | 등록 1.1초, 수정·완료 0.5~0.6초 |

쓰기는 로그의 전체 시간에서 읽기와 Gemini를 뺀 값이다. 합하면 음성 등록이 약 7.2초다. 2026-11-11 캐시 배포 뒤 목록 읽기는 0초가 됐다.

버튼을 떼는 즉시 Pi가 THINKING 표정을 띄워 체감 지연을 줄인다. 시간 제한은 `../contracts.md` § 1을 본다.

## 1차 실행 환경

- 캡스톤 AWS 계정의 EC2(시드니)에서 레포의 `Dockerfile` / `docker-compose.yml`로 실행한다 → [[projects/eureka-desk-pet/outputs/2026-11-01-backend-aws-deploy|11/01 배포]].
- 알림 시연: "마감 3분 뒤 업무 + 2분 전 알림"으로 등록 → Pi가 목록을 받아 시간이 되면 알림.
- HTTPS는 2차다 (`../architecture.md` § 2차 고도화 1번).
