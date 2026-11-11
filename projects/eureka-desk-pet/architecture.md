# eureka-desk-pet 1차 아키텍처 (최소 비용 MVP)

> 상태: **제안 (2026-10-08)** — 팀 검토 후 확정.
> 근거: [[projects/eureka-desk-pet/outputs/2026-10-08-tech-review|기술 검토]],
> [[projects/eureka-desk-pet/outputs/2026-10-08-mvp-cost-research|1차 최소 비용 조사]].
> 레이어 간 형식은 `contracts.md`가 정본이다.
> Pi 쪽 구조·통신·상태는 2026-10-07 하드웨어 결정([[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]])을 그대로 따른다 —
> 음성은 HTTP 한 번, TTS는 Pi(Edge-TTS), LED 색은 Pi, 서보·VAD 제외. 이 문서는 그 위에 Backend·AI를 단순화한 안이다.

## 원칙

1. **실행 단위는 2개다.** Raspberry Pi 프로그램 1개, Backend 프로세스 1개. 나머지는 외부 API.
2. **업무를 바꾸는 주체는 Backend 하나다.** LLM은 "무엇을 할지"를 JSON으로 답할 뿐이고,
   실행·검증·성공 판단은 Backend가 한다.
3. **1차는 월 운영비 0원을 목표로 한다.** 유료 전환은 § 1차 기술 선택의 전환 조건을 만족할 때만 한다.
4. **1차에 없는 기능은 § 2차 고도화에 적어 두고, 1차 완료 전에는 손대지 않는다.**

## 구조

```text
Raspberry Pi: 녹음 · WAV 업로드 · TTS/LED/LCD
  │ POST /api/voice + X-Device-Token    ▲ HTTP VoiceResponse
  ▼                                    │
FastAPI: 입력 검증 → tasks.py: 맥락 조회 → Decision 검증·실행 → 응답
                        ├─ eureka.py: 실연동 HTTP 또는 메모리 mock
                        ├─ db.py: SQLite pending · reminders
                        └─ ai.py: Gemini 오디오/텍스트 1회 → Decision

개발·평가: POST /api/text · 상태 확인: GET /health
마감 임박: Pi가 GET /api/tasks로 목록 수신 → 시간이 되면 Pi가 스스로 알림 (Pi 쪽 구현 전)
목록 조회: 서버 시작 시·하루 1회 Eureka 조회 → 로컬 캐시, 쓰기 시 캐시 수정 후 약 10초 뒤 재조회
```

## 레이어 책임

| 위치 | 맡는 일 | 맡지 않는 일 |
| --- | --- | --- |
| Pi | 버튼, 녹음, 업로드, TTS 합성·재생, LED/LCD 표시, 지연 안내, 마감 알람(목록 저장, 시간 확인, 보낸 기록, 재부팅 복구) | 업무 판단, Eureka 호출, LLM API 키 보관 |
| Backend `api/` | 엔드포인트, 디바이스 토큰 확인 | 비즈니스 규칙 |
| Backend `services/ai.py` | LLM 1회 호출, 출력 JSON 스키마 검증 | 업무 실행 |
| Backend `services/tasks.py` | action 실행, 기본 알림 60분, item_id 검증, Eureka 성공 확인 후 응답 | HTTP 세부 |
| Backend `services/task_cache.py` | Eureka 미완료 목록 캐시, 쓰기 뒤 캐시 수정·재조회 | 업무 판단 |
| Backend `eureka.py` | 요청·응답 필드 매핑, epoch ms 변환, 목록 100건 제한과 오류 본문 확인 | 정책 |
| Backend `db.py` | 알림 간격·재질문 대기 저장 | 업무 원본 (Eureka가 원본) |

`services/reminder.py`, `api/ws.py`의 알림 전송, `alerts` 테이블은 이전 설계의 코드다. 2026-11-06 결정으로 쓰지 않으며 구현 때 삭제한다.

코드 레포 구조는 각 레포 README가 정본이다: `desk-pet-backend`(`api/` → `services/` → `eureka.py`·`db.py`),
`Hardware`(`setup/` → `parts/` → `app/`).

LED 색과 LCD 표정은 Pi의 매핑표 한 곳에서만 정한다 (`contracts.md` § 5). Backend는 `status`·`type`과 `output`(tts 문장, lcd 표정·문구)만 보낸다.

## 처리 흐름

### 음성 요청 — 요청 한 번, 응답 한 번

1. 버튼을 누르면 Pi가 즉시 LISTENING으로 바꾸고 녹음한다. 버튼→LED 100ms 요구는 Pi 안에서 충족한다.
2. 버튼을 떼면 THINKING 표정으로 바꾸고 `POST /api/voice`로 WAV를 올린다.
3. Backend가 Eureka 미완료 목록과 `pending`(재질문 대기)을 조회한 뒤, `ai.py`가 오디오와 맥락을 LLM에
   **1회** 전달해 `{transcript, action, args}`를 받는다. 목록은 캐시에서 읽는다 (`contracts.md` § 4).
   LLM 호출은 실패 시 1회 재시도한다.
4. `tasks.py`가 검증·실행하고 결과를 템플릿 문장으로 만든다. 응답 문장을 위해 LLM을 두 번 부르지 않는다.
5. 응답 `{status, type, output}`을 받은 Pi가 `output.tts`를 Edge-TTS로 합성·재생하고 IDLE로 돌아간다.
6. 응답이 늦으면 Pi가 10초에 지연 안내를 내고 20초에 실패 처리한다 (`contracts.md` § 1, 구현 전).
   이전 정책은 Pi 10초·Backend 7초였는데, Gemini 처리 시간이 1~30초로 불규칙해 정상 호출도 `TimeoutError`로 끝나서 바꿨다.

### 마감 임박 알림 (2026-11-06 결정. Backend 목록 API는 2026-11-11 배포, Pi 구현 전)

폴링과 WebSocket 대신 Pi가 스스로 울린다. 마감 시각을 알고 있으면 알림 시각에 서버 연결이 필요 없기 때문이다. 하드웨어 팀이 동의했다.

1. Pi가 부팅 때와 쓰기 응답을 받은 뒤 `GET /api/tasks`로 목록을 받아 저장한다.
2. `마감 − offset_minutes ≤ 지금 < 마감`이고 보낸 기록이 없으면 대상이다.
3. Pi가 문장을 만들어 재생하고 LCD에 표시한 뒤 기록을 저장한다 ("보고서 검토 마감이 1시간 남았어요.").
4. 재부팅하면 저장한 목록과 기록으로 복구한다.

### 업무 목록 캐시 (2026-11-06 결정, 2026-11-11 배포)

요청마다 Eureka를 읽지 않고 서버 시작 시와 하루 1회 받아 둔 목록을 쓴다. 쓰기 성공 시 캐시를 바로 고치고 약 10초 뒤 다시 받는다.
규칙은 `contracts.md` § 4가 정본이다. Eureka 웹에서 바꾼 내용은 최대 하루 늦게 반영되며, 이 점은 팀 전체 리뷰 항목이다.

### 재질문

1. 대상 업무가 여러 개이거나 날짜가 모호하면 LLM이 `clarify`와 질문을 돌려주고, 응답 `type`은 `clarification`이 된다.
2. Backend는 원래 요청을 `pending`에 60초 동안 저장하고 질문을 응답한다.
3. 사용자가 다시 버튼을 눌러 답하면, 음성 요청 3단계에서 `pending`을 맥락으로 함께 넣는다.
   확정되면 실행하고 `pending`을 지운다.

## 1차 기술 선택 (최소 비용)

| 구성요소 | 1차 선택 | 월 비용 | 전환 조건 | 전환 대상 |
| --- | --- | --- | --- | --- |
| 호스팅 | AWS 무료 플랜 크레딧으로 EC2 (시드니) + Docker | 크레딧 내 0원 | 유료 플랜 전환 | 서울 리전 EC2 |
| 음성 인식 + 해석 | **현재 코드**: Gemini API에 오디오 또는 텍스트를 보내 JSON을 한 번 받는다. 모델은 설정으로 지정한다 (contracts § 3) | 0원 | 무료 한도 초과, 평가셋 18/20 미달, 실제 업무 데이터 필요 | Gemini 유료 (월 약 $1.5) 또는 STT 분리 + Claude Haiku 4.5 (월 약 $5.6) |
| TTS | Pi에서 edge-tts (`ko-KR-SunHiNeural`) | 0원 | 서비스 장애·차단 | Google Cloud TTS (Standard 월 400만 자 무료) |
| 저장소 | SQLite 파일 1개 | 0원 | Backend가 2대 이상이 될 때 | 2차에서 결정 |
| 하드웨어 | 구매 부품 + 교내 자원 (Pi 4B, 3D 프린터) | 일회성 46,020원 | I2S 오디오 실패 | USB 마이크·스피커 |

산정 근거와 출처는 [[projects/eureka-desk-pet/outputs/2026-10-08-mvp-cost-research|최소 비용 조사]]에 있다.
LLM 호출은 `ai.py`의 함수 하나 뒤에 두므로, 전환은 설정값 변경으로 끝나야 한다.

## 1차에서 뺀 것

| 항목 | 빼는 이유 | 2차에서 다시 넣는 조건 |
| --- | --- | --- |
| 경량 Intent 모델 | 거의 모든 발화가 날짜·업무명 추출을 요구해 결국 LLM을 호출 | 측정 결과 특정 발화의 LLM 지연이 병목일 때 규칙 기반 단축 경로 |
| MCP | 도구가 전부 Backend 내부 함수라 LLM 출력 JSON으로 충분 | 외부 클라이언트(예: Claude Desktop)에 도구를 공개할 때 |
| 로컬 LLM 실서비스 적용 | Pi 4B 1B 모델 요청당 40~55초(추정), GPU 상시 $472/월 | 비교 실험(`ai/README.md`)에서 품질·지연 기준을 넘을 때 |
| Output Controller | Pi 상태표 + 응답 `output`으로 대체 | — |
| Second Brain 연동 (FR-16/17) | 기획서에서도 "향후 확장"으로 분류 | 1차 완료 후 |
| 서보 모터 | 기능 요구사항에 없음, LED와 PWM 충돌 가능, 5V 전원 부담 | 1차 완료 후 |
| VAD, wake word | PTT 버튼이 발화 구간을 정함 | — |
| HTTPS (NFR-11) | 1차는 HTTP로 EC2에 연결한다 — **팀 합의 필요** | 운영 환경 전환 시 |
| WebSocket·SSE 푸시 | 마감 시각을 아는 Pi가 스스로 울리면 상시 연결이 필요 없다 (2026-11-06) | 서버가 먼저 알려야 할 기능이 생길 때 |
| Lambda 배포 | 스케줄러·상태 저장이 리소스 여러 개로 나뉨 | 운영 환경 전환 시 검토 |
| 다중 사용자·팀 알림 전파 | 기획서 4-2(6) 구현 제외 범위 | 2차, `user_id`와 기기 매핑부터 |

## 2차 고도화 (우선순위 순)

1. HTTPS 적용, 서울 리전 이전.
2. 모델 비교·유료 전환 — 평가셋의 정확도·지연 결과로 결정.
3. 로컬 LLM 비교 실험 — AI 평가 산출물. 팀원 GPU PC를 먼저 쓴다.
4. 요청 ID 기반 로그 추적 (NFR-09).
5. 서보 모터 표현, 하우징 개선.
6. 다중 디바이스·팀 알림 전파.
7. Second Brain 맥락 기억 (FR-16/17).
8. Lambda 등 서버리스 전환 검토.

## 1차 일정

| 기한 | Backend·AI | Pi | 완료 기준 |
| --- | --- | --- | --- |
| 10/18 | `contracts.md` 확정, `/api/text` → LLM → 실제 Eureka | 오디오 overlay, 버튼 | 평가셋 20개 중 18개 성공 · `arecord`/`aplay` 둘 다 동작 |
| 10/31 | `/api/voice` 전체 흐름 | LED/LCD 매핑 | 노트북 마이크 기준 5초 이내 응답 |
| 일정 별도 확인 | 음성 요청과 업무 처리 검증 | Pi와 Backend 실제 연결 | 등록·조회·완료·재질문 점검 |
| 11/28 | 시나리오 리허설 | 하우징 조립 | 기획서 시나리오 3개 + 재질문 시나리오 연속 성공 |
| 12월 초 (날짜 확정 필요) | 소음·오인식 테스트 | 마감 정리 | 최종 발표 |

마감 임박 알람은 음성 요청·응답 출력이 끝난 뒤 구현해 검증한다.

## 막혔을 때의 대안 (Plan B)

| 리스크 | 대안 |
| --- | --- |
| I2S 오디오 overlay 실패 | USB 마이크 + USB 스피커 (드라이버 불필요) |
| LLM 정확도 미달 | 데모 발화 고정 + 마감 변경 전 확인 질문 |
| 전시장 소음 | `/api/text`로 키보드 입력 시연 |
| Eureka 장애 | `EUREKA_MODE=mock` |
| 무료 API 한도·장애 | 유료 등급 전환 (§ 1차 기술 선택) |

## 기존 설계와의 대응

| 기존 (Notion 02 시스템 설계, backend 레포 docs) | 1차 |
| --- | --- |
| 1계층 Device + 2계층 Voice & Local Processing | Pi 프로그램 1개 |
| 3계층 STT / 경량 Intent / LLM·MCP / 사용자 요청 해석 | `services/ai.py` (LLM 1회) |
| 3계층 Backend API / Scheduler / DB / Eureka Adapter / WebSocket | `api/`, `db.py`, `eureka.py`. Scheduler·WebSocket은 Pi 자체 알람으로 대체 |
| Output Controller (TTS·LED·LCD 명령 생성) | 응답 `output` + Pi 상태표, TTS는 Pi |
| TaskService · ReminderService · OutputService + Port 3개 | `services/tasks.py` + `eureka.py`의 real/mock 전환. 알림 판단은 Pi로 이동 |
| Backend REST 5개 (`/api/tasks` 등) | 없음 — Pi는 `/api/voice`만 호출 |
| `backend_interfaces.md` § 10 미결정 1·2 (알림 저장, 중복 방지) | 알림 간격은 SQLite `reminders`, 보낸 기록과 중복 방지는 Pi |
| 미결정 3 (미전달 알림 재처리) | 재연결 시 상태 재확인 후 재전송 |
| 미결정 4 (AI 컴포넌트 간 인터페이스) | `ai.py` 함수 하나, 출력은 `contracts.md` § 3 |
| 10/07 하드웨어 결정 16건 (Mock 형식 유지, `/api/voice`, Pi TTS, 상태 6종) | 그대로 채택 — `contracts.md` § 1·2·5 |
| 10/07 결정 16번 (서버 Local LLM) | 현재 백엔드는 Gemini API를 사용한다. Local LLM은 비교 실험으로 남겨 둔다. |
| AI–HW 충돌 검토 13건 | 9번(LLM 방식)을 뺀 미결 항목은 `contracts.md`가 답을 정함 (§ 변경 이력) |
