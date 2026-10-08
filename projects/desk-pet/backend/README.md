# Backend 레이어

- 코드: `hansung-capstone-team-6-clear/desk-pet-backend`
- 계약: `../contracts.md` (전체)
- 구조: `../architecture.md`

## 1차 모듈 구성 (제안)

```text
src/desk_pet_backend/
  main.py      FastAPI: /voice · /text · /ws/{deviceId} · /health, 디바이스 토큰 확인
  ai.py        LLM 1회 호출 → contracts § 3 JSON, 스키마 검증
  tasks.py     action 실행 + 정책 + 응답 문장 템플릿
  eureka.py    Eureka HTTP 어댑터 (EUREKA_MODE=real 또는 mock)
  reminder.py  60초 폴링 루프, 알림 판단, 재연결 시 재전송
  db.py        SQLite (reminders · alerts · pending)
  config.py    환경 변수 읽기
mocks/         Eureka mock (eureka.py의 mock 모드가 사용)
tests/         단위 테스트 + tests/eval/ 평가셋
```

헥사고날 계층(Port·Adapter·Service)은 1차에서 쓰지 않는다. 교체가 실제로 필요한 곳은 Eureka(real/mock) 하나뿐이고,
그건 `eureka.py` 안의 모드 전환으로 충분하다.

## 레포에서 정리할 것 (2026-10-08 기준)

| 항목 | 현재 | 1차 제안 |
| --- | --- | --- |
| 디렉터리 | 헥사고날 빈 폴더 (`application/ports`, `application/services`, `domain/*`, `infrastructure/*`) | 위 모듈 7개 평면 구조 |
| `.env.example` | `EUREKA_USERNAME`/`EUREKA_PASSWORD`, `OPENAI_*`, `MCP_*` | PoC의 실제 인증은 `x-api-key` → `EUREKA_API_KEY`. `LLM_API_KEY`, `DEVICE_TOKEN`, `EUREKA_MODE` 추가, `MCP_*` 제거 |
| Mock 서버 | 레포 `mocks/mock_server` (v0.1, 상태값 `completed`)와 Notion 첨부 Mock v2 (상태값 `doing`) 두 버전 공존 | 하나로 합치고 상태값은 Eureka 실제 값 (`todo`/`doing`/`done`/`hold`) |
| 인터페이스 문서 | `docs/interfaces/backend_interfaces.md` (REST 5개, Intent, MCP) | `contracts.md`로 대체 |
| `docs/` | 임시 보관, knowledge 레포로 이전 예정 | 이전 위치는 이 vault의 `projects/desk-pet/`. 이전 확인 후 레포 `docs/` 삭제 |

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
