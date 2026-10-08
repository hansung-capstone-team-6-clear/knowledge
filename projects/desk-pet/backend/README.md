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
├── api/         ① 입구: /voice, /text, /ws (Controller 역할)
├── services/    ② 규칙·흐름: tasks.py, ai.py, reminder.py
├── eureka.py    ③ 바깥 연결: EurekaClient(real) + EurekaMock(mock)
└── db.py        ③ 바깥 연결: SQLite (reminders · alerts · pending)
tests/           자동 테스트, eval/ 평가셋, integration/ PoC 스크립트
mocks/, docs/    이전 단계 참고용 (1차에서 안 씀)
```

- 호출 방향은 `api/ → services/ → eureka.py, db.py` 한쪽뿐이다.
- MVC를 그대로 쓰지 않는 이유: 화면(View)이 없는 API 서버다. 응답 JSON 모양(`schemas.py`)이 View에 가장 가깝다.
- 기능별 폴더 대신 역할별 폴더를 쓰는 이유: 기능이 두 개(음성 요청, 마감 알림)뿐이다. 5개 이상이 되면 기능별 폴더를 검토한다.
- 헥사고날 Port는 쓰지 않는다. 교체가 필요한 곳은 Eureka(real/mock) 하나이고, `eureka.py`의 `get_eureka()`로 충분하다.

셋업 PR에서 함께 정리한 것: 빈 헥사고날 폴더 삭제, `.env.example`을 `EUREKA_API_KEY`(x-api-key) 기준으로 변경,
`pyproject.toml` BOM 제거(설치 실패 원인), MCP·OpenAI·APScheduler 의존성 제거, `mocks/`·`docs/`를 참고용으로 표시.

## 구현 체크리스트 — Mock에서 실제 Eureka로

1. 알림 대상은 `status != done`. Mock v2는 `doing`만 대상이라 실제 Eureka에서 `todo`인 업무의 알림이 빠진다. `hold` 제외 여부는 팀이 정한다.
2. 중복 방지 키는 (itemId, dueAt, offsetMinutes) — SQLite `alerts` PRIMARY KEY. 키에 dueAt이 없으면 마감 연기 후 알림이 다시 오지 않는다.
3. 재연결 시 미전달 알림은 저장해 둔 문장을 그대로 보내지 말고, 상태를 다시 확인하고 남은 시간을 다시 계산해서 보낸다.
4. 상태는 메모리가 아니라 SQLite에 둔다. 재시작하면 알림 설정이 사라진다.
5. 목록 조회 `limit=100` → 100개를 넘으면 다음 페이지까지 읽는다.
6. 폴링 간격은 60초 (Mock v2 기본값 300초는 알림이 최대 5분 늦는다).
7. 입력 검증 오류가 500으로 올 수 있으므로 상태 코드와 응답 본문을 함께 확인한다.
8. 시간 변환(epoch ms ↔ +09:00)은 `eureka.py`에서만 한다. 다른 곳에서 변환하면 알림이 9시간 어긋난다.

## 응답 시간 예산 (1차 목표 5초)

| 구간 | 목표 | 근거 |
| --- | --- | --- |
| 업로드 (WAV 5초 분량) | 0.3초 | 추정 |
| LLM 1회 (오디오 포함) | 2초 | 추정 — 평가셋으로 측정 |
| Eureka 호출 | 1초 | PoC 실측 0.126~1.135초 |
| TTS 합성 시작 (Pi, edge-tts) | 1초 | 추정 |
| 합계 | 약 4.3초 | |

버튼을 떼는 즉시 Pi가 THINKING 표정을 띄워 체감 지연을 줄인다.

## 1차 실행 환경

- 레포의 `Dockerfile` / `docker-compose.yml`로 팀원 노트북에서 실행하고, Pi와 같은 Wi-Fi에 둔다.
- 알림 시연: "마감 3분 뒤 업무 + 2분 전 알림"으로 등록 → 1분 안에 Pi에 알림.
- 외부 배포(HTTPS/WSS)는 2차다 (`../architecture.md` § 2차 고도화 1번).
