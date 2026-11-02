---
type: query-output
created: "2026-11-01"
question: "10-29 회의 A4(클라우드에 /api/voice 먼저 열기)·A5(바뀐 요청 형식 반영)는 어디까지 됐고, 배포 환경은 어떻게 되어 있는가?"
sources:
  - "https://github.com/hansung-capstone-team-6-clear/desk-pet-backend/pull/6 — 2026-11-01 머지된 요청 형식 변경 PR"
  - "[[projects/eureka-desk-pet/meetings/2026-10-29-mentor-meeting-note|10-29 회의록]] — A4·A5 액션 아이템"
  - "[[projects/eureka-desk-pet/contracts|레이어 간 계약]] § 0·1 — request_id·user_id 규칙"
  - "2026-11-01 EC2 배포 후 /health·/api/voice 직접 호출 결과 (이서현 PC에서 실행)"
---

# 11/01 백엔드 요청 형식 변경과 AWS 배포

## Answer

10-29 회의 결정대로 요청에서 `request_id`를 빼고 `user_id`를 선택값으로 바꾼 코드가 PR #6으로 `main`에 머지됐다(A5의 코드 부분).
이 `main`을 캡스톤용 AWS 계정의 EC2 한 대에 Docker로 띄웠고, **실제 Gemini와 실제 Eureka에 연결된 `/api/voice`가 외부에서 동작한다**(A4).
하드웨어 팀은 지금 이 주소로 붙어 작업할 수 있다. 다만 계정 제한 때문에 서울이 아니라 **시드니 리전**에 있고,
응답 시간이 6~8초로 9초 제한에 가깝다.

## Evidence

### 1. 요청 형식 변경 (PR #6, 2026-11-01 머지)

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| `request_id` | Pi가 만들어 요청에 넣음 (필수) | **Backend가 만들어 응답에만** 넣음. 형식 `req_YYYYMMDD_HHMMSS_xxxx` (KST) |
| `user_id` | 필수 | **선택**. 생략하면 기본 사용자 `"default"` |
| 바뀐 파일 | — | `api/voice.py`, `schemas.py`(`TextRequest`), `tests/test_smoke.py`, `api/README.md`, `docs/interfaces/backend_interfaces.md` |

- `request_id`는 `api/voice.py`의 `new_request_id()`가 만든다. `services/tasks.handle()`의 시그니처는 그대로다.
- 로컬 `pytest` 38개 통과. `ruff` 경고 2개는 기존 `tests/eval/run_eval.py`의 것이라 이번 변경과 무관하다.
- 레포에는 CI가 없어 PR에서 자동 검사는 돌지 않았다.

### 2. 배포 환경

| 항목 | 값 |
| --- | --- |
| AWS 계정 | 캡스톤용 계정(팀 공용 구글 계정으로 가입, 무료 플랜 — 크레딧 100달러, 2027-05-01 만료) |
| 배포용 IAM 사용자 | `deskpet-deploy` (`AmazonEC2FullAccess`만) |
| 리전 | **ap-southeast-2 (시드니)** |
| 인스턴스 | `deskpet-backend`, t3.micro, Ubuntu 24.04 |
| 주소 | `http://3.104.122.146:8000` — `/api/voice`, `/api/text`, `/health`, `/docs` |
| 보안 그룹 | `deskpet-backend-sg` — 22번은 배포한 PC의 IP만, 8000번은 전체 공개 |
| 실행 방식 | `docker compose up -d --build` (레포의 `Dockerfile`·`docker-compose.yml` 그대로) |
| 코드 전달 | 레포가 비공개라 서버에서 clone하지 않고, PC에서 `git archive origin/main`으로 묶어 `scp`로 복사 |
| `.env` | 서버에만 있음(권한 600). `EUREKA_MODE=real`, Gemini·Eureka 키, **서버 전용 `DEVICE_TOKEN`** |

- SSH 키(`deskpet-key.pem`)와 기기 토큰 파일은 배포한 이서현 PC의 `~/.ssh/`에만 있다. vault와 GitHub에는 올리지 않는다.
- 기기 토큰은 하드웨어 담당(김리윤·김지훈)에게 DM으로만 전달한다.

### 3. 동작 확인 (2026-11-01, 서버 직접 호출)

| 요청 | 결과 | 시간 |
| --- | --- | --- |
| `GET /health` | `{"status":"ok"}` | — |
| `POST /api/voice` 토큰 없음 | HTTP 401 | — |
| "오늘 할 일 알려줘" (Eureka mock일 때) | `OK`, `task_list`, 업무 0건 | 7.7초 (재시작 직후 첫 요청) |
| "내일 오후 3시까지 보고서 작성 추가해줘" (Eureka mock일 때) | `OK`, `task_created`, 표정 `happy` | 3.4초 |
| "이번 주 업무 알려줘" (**실제 Eureka**) | `OK`, `task_list`, 실제 Eureka의 10-28 검증용 업무 2건 | 6.3초 |

- 실제 Eureka로 바꾼 뒤에는 조회만 했다. 등록 테스트는 실제 계정에 업무가 쌓이므로 하지 않았다.
- Gemini 키를 넣기 전에는 모든 요청이 `status: "ERROR"`("다시 한 번 말씀해 주세요")였다. 로그 원인은 API 키 없음이었다.

## Notes

- **서울 리전을 쓸 수 없다.** `deskpet-deploy`로 확인했을 때 시드니만 EC2 호출이 되고, 서울·도쿄·오사카·싱가포르·미국 동부·서부는
  상위 조직의 SCP(서비스 제어 정책)로 거부됐다. 직접 가입한 계정이라, **무료 플랜 계정에 거는 제한으로 추정**한다(확인 안 됨).
  유료 플랜으로 바꾸면 서울로 옮기기로 했다. 옮길 때 바뀌는 것은 서버 주소뿐이고, SQLite 데이터(알림 설정·이력)는 옮겨지지 않는다.
- **응답 시간이 9초 제한에 가깝다.** 10-29 회의에서 공유된 AI 측정값(중앙값 1.29초)보다 훨씬 길다. 측정 구간이 다르고
  (이번 값은 음성 업로드·STT·Eureka 호출·시드니 왕복을 모두 포함), 측정 횟수가 적어 원인은 아직 나누지 않았다.
- **퍼블릭 IP는 고정이 아니다.** 인스턴스를 중지했다가 켜면 바뀐다. 하드웨어 연동 기간에는 중지하지 않거나 탄력적 IP를 붙인다.
- **이제 테스트 요청이 실제 Eureka에 업무를 만든다.** 하드웨어 연동 테스트 업무는 이름에 "테스트"를 붙이고 끝나면 정리한다.
- 무료 플랜이므로 크레딧을 넘는 요금은 청구되지 않는다. 유료로 바꾸면 크레딧 초과분이 카드로 청구되니 그 전에 예산 알림을 건다.

## Follow-ups

1. **이서현:** 하드웨어 팀에 주소·기기 토큰(DM)·요청 형식(`audio`만 필수, `user_id` 선택)을 전달한다 → A5 완료.
2. **이서현:** `contracts.md` § 1의 요청 예시가 PR #6 코드와 같은지 다시 확인한다.
3. **김리윤·김지훈:** 이 주소로 1번(음성 요청)·2번(응답 출력)을 붙여 보고, 9초를 넘는 응답이 있으면 기록한다.
4. **이서현:** 응답 시간을 구간별(업로드·Gemini·Eureka)로 나눠 재고, 자주 9초를 넘으면 서울 이전을 앞당긴다.
5. **팀:** 유료 플랜 전환 여부를 정한다. 전환하면 예산 알림을 먼저 걸고 서울에 다시 배포한 뒤 시드니 인스턴스를 종료한다.
6. **팀:** 루트 계정 MFA를 켜고, 팀원별 IAM 사용자를 따로 만든다(루트 비밀번호 공유 금지).
7. **이서현·박시현:** 서버의 Gemini 키를 개인 키에서 캡스톤 전용 키로 바꾼다.
