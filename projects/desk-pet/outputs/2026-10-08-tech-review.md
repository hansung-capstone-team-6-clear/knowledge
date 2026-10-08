---
type: query-output
created: "2026-10-08"
question: "데스크펫(Eureka Process 연동 업무관리 AI 데스크펫) 기획은 기술적으로 구현 가능한가? 레이어 구조가 과하지 않은가?"
sources:
  - "Notion export: Capstone Eureka Process — 01 기획, 02 시스템 설계(desk-pet-architecture.html 포함), 03 AI & 데이터, 04 Backend, 05 Raspberry Pi, 10/01 기업 미팅 기록 (2026-10-08 내보내기)"
  - "Notion 첨부: deskpet_mock_server.zip (Mock v2), Eureka_Process_API_Test.zip"
  - "hansung-capstone-team-6-clear/desk-pet-backend — docs/interfaces/backend_interfaces.md, mocks/mock_server, .env.example"
  - "hansung-capstone-team-6-clear/Hardware"
  - "https://github.com/geerlingguy/ai-benchmarks"
  - "https://stratospherelinuxips.readthedocs.io/en/develop/immune/research_rpi_llm_performance.html"
  - "https://medium.com/@omkar121212/ollama-vs-llama-cpp-on-raspberry-pi-5-8e7fbeb310de"
  - "https://huggingface.co/WJO/llama-cpp-b10355-colab-t4-sm75"
  - "https://www.doit.com/compute/spot/ap-northeast-2/g4dn.xlarge"
---

# 데스크펫 기술 검토

## Answer

**구현은 가능하다. 다만 기존 설계는 4인 3개월 규모에 비해 AI 레이어와 백엔드 구조가 과하다.**
진짜 리스크는 하드웨어나 Eureka 연동이 아니라, 아직 프로토타입이 없는 "음성 → 대상 업무(itemId) 특정 → 실행" 경로다.
기획서 1-2의 차별점은 "선제 알림 + 물리적 존재감"인데, 설계 복잡도는 차별점과 관계없는 AI 라우팅
(경량 Intent → LLM → MCP)에 몰려 있다. 단순화한 1차 구조는 [[projects/desk-pet/architecture|1차 아키텍처]]에 있다.

### 검증된 것과 안 된 것

| 구간 | 상태 | 근거 |
| --- | --- | --- |
| Eureka API CRUD | 검증됨 | PoC 8개 호출 모두 성공, 0.126~1.135초 (04 Backend 1-4) |
| 폴링 → WebSocket 선제 알림 | Mock으로 검증 | Mock v2 scheduler + WS push |
| 하드웨어 | 부품만 확정 | 배선·동작 미확인 |
| 한국어 STT, LLM 정보 추출, TTS | 검증 안 됨 | Mock은 `audio_text_hint`로 텍스트를 직접 넣음 |
| 업무명 → itemId 특정, 재질문 대화 | 설계 없음 | `ClassifyIntent`·`AnalyzeUserRequest` 출력에 itemId 없음 (팀 댓글도 지적) |

### 레이어별 판단

| 레이어 | 기존 설계 | 판단 | 1차 조치 |
| --- | --- | --- | --- |
| 1·2계층 (Pi) | 계층 2개로 분리 | 개념상 문제없음 | 실제 프로그램은 1개. Silero VAD 제외 (PTT가 발화 구간을 정함), Pi 로컬 Intent 처리(기획 4-3(5)) 제외 |
| 3계층 AI | STT → 경량 Intent → (신뢰도 미달 시) LLM → MCP | 과설계 (가장 큰 문제) | LLM 1회 → JSON → Backend 실행 |
| 3계층 Backend | 헥사고날: 계층 6개, 서비스 3개, Port 3개, Adapter 4개 | 담당자 1명 기준으로 과함 | FastAPI 프로세스 1개, 모듈 7개 |
| 저장소 | 알림 설정·이력을 "Second Brain 또는 백엔드" | Second Brain은 부적합 | SQLite 테이블 3개 |
| 배포 | Lambda 검토 | 가능하지만 MVP에는 무거움 | 1차는 Docker 1개, Lambda는 2차 검토 |

AI 레이어를 줄이는 이유:

1. **경량 Intent 모델은 얻는 게 거의 없다.** Intent 7개가 전부 날짜·업무명 추출을 요구한다("오늘 할 일"도 날짜 필요).
   결국 거의 모든 요청이 LLM으로 가고, 남는 건 학습 데이터 수집, 임계값 튜닝, 테스트 경로 하나 추가다.
2. **MCP가 할 일이 없다.** 도구가 전부 Backend 내부 함수다. 게다가 기존 흐름(04 Backend § 8)은 "LLM → MCP"와
   "Backend → Eureka" 두 경로로 Eureka에 접근한다. "성공 확인 전 성공 안내 금지"와 중복 실행 방지를 지키려면
   실행 주체는 Backend 하나여야 한다.
3. **로컬 LLM은 서비스 경로에서 뺀다.** 아래 § 로컬 LLM 비교 참고.

## Evidence

### 코딩 전에 맞춰야 할 불일치

| # | 불일치 | 위치 | 1차 처리 |
| --- | --- | --- | --- |
| 1 | 알림 시점 표현 세 가지: `reminder_offset_minutes` / `reminderAt` / `alert_before_minutes` | 03 AI, 04 CreateTaskAPI, Mock v2 | `offsetMinutes` 하나 (contracts § 0) |
| 2 | 문서는 시간대 없는 ISO 시각, 실제 Eureka는 epoch ms | PoC `eureka_task_create.py` | 변환은 `eureka.py`에서만 |
| 3 | 필드명: 문서·Mock은 `taskName`/`itemId`/`items`, 실제 API는 `name`/`id`/`list`/`stage$$`/`total`/`done` | PoC 스크립트 vs Mock | contracts § 4 매핑 |
| 4 | TTS 위치: Backend는 `ttsText`, Pi는 `ttsAudio: binary`를 받게 되어 있음 | 04 § 7, 05 (3) | backend 레포 결정대로 Pi가 TTS |
| 5 | `POST /api/voice`가 STT 텍스트만 반환, 최종 응답 경로 미정 | 05 (5) | `/voice` 응답 하나에 최종 결과 |
| 6 | LED 색 충돌: 듣는 중 = 노랑 점멸 = 마감 알림, 말하는 중 = 파랑 점멸 = 재전송 알림 | 05 ledStates, 01 3-4·3-5 | 알림 주황, 오류 빨강 (contracts § 5) |
| 7 | 응답시간: 구간 목표 합 3.8초 > 전체 목표 3초, 상한 합 6.5초 > 5초, LLM 구간은 예산에 없음 | 01 4-3(8) | Intent 단계 삭제, 1차 목표 5초 |
| 8 | `ws://` vs NFR-11 (HTTPS) | 05 (4) | 1차는 같은 Wi-Fi, 외부 배포 시 WSS |
| 9 | 4계층 그림에서 2→3계층 화살표가 "음성 텍스트 데이터"인데 STT는 3계층 | 02 이미지 | 1차 구조로 대체 |
| 10 | 차별점 "팀원 펫 간 알림 전파" vs 구현 제외 "다중 사용자 계정" | 01 1-2 vs 4-2(6) | 2차로 이동, 발표 문구 조정 |
| 11 | `.env.example`이 `EUREKA_USERNAME`/`PASSWORD`인데 PoC 실제 인증은 `x-api-key` | backend 레포 | `EUREKA_API_KEY` |

### Mock v2 → 실제 Eureka 전환 시 결함 (Notion 첨부 `deskpet_mock_server.zip`, `app/main.py`)

1. `main.py:530` `status != "doing"`이면 건너뜀 → 실제 Eureka에서 `todo`인 업무의 알림이 빠진다.
2. `main.py:121` 중복 판단 키가 (itemId, 알림 시점)뿐 → 마감을 연기해도 알림이 다시 오지 않는다.
3. `main.py:621` 재연결 시 저장한 알림을 그대로 재전송 → 기획 3-5(2)의 "최신 상태 재확인, 남은 시간 재계산"과 다르다.
4. `main.py:23` 상태가 전부 메모리 → 재시작하면 알림 설정이 사라진다.
5. `main.py:29` 기본 폴링 300초 → 알림이 최대 5분 늦는다.
6. PoC `eureka_task_list.py:22` `limit=100` → 업무가 100개를 넘으면 폴링에서 빠진다.

backend 레포의 `mocks/mock_server`는 다른 버전(v0.1, 상태값 `completed`)이다. 두 Mock을 하나로 합쳐야 한다.

### 하드웨어

1. 핀맵 **표**는 실제 Pi 4B 핀 배열과 일치한다. **배선 그림**은 라벨이 틀렸다 (예: 존재하지 않는 GPIO 28 표기).
2. PTT 버튼 핀이 핀맵에 없다 → GPIO17(pin 11) 제안.
3. I2S 마이크·앰프 클럭 공유는 맞지만 녹음·재생을 함께 여는 overlay가 하나 필요하다 (후보 `googlevoicehat-soundcard`, 추정).
4. WS2812B(PWM0)와 서보(PWM1)는 같은 PWM 블록을 써서 충돌할 수 있다 → 서보는 1차 제외.
5. LED 24개 최대 밝기 약 1.4A + 서보를 Pi 5V에서 같이 공급 → 밝기 30% 이하로 제한.
6. 디자인 약 85mm 큐브 vs Pi 4B 85.6mm, 24구 링 외경 약 66mm → 실측 필요.

상세: `projects/desk-pet/hardware/README.md`.

### 로컬 LLM 비교

전제(추정): 요청당 입력 1,200토큰, 출력 80토큰.

| | Pi 4B에서 직접 | 클라우드 GPU에 로컬 LLM | 클라우드 API |
| --- | --- | --- | --- |
| 실행 가능 모델 | 1~1.5B (RAM 4GB, OS 포함). 8B는 4bit로도 4GB 초과(추정) | 7~8B (T4 16GB) | 공급자 모델 |
| 생성 속도 | Pi 5 실측 1B 약 11 tok/s, 3B 약 4.7~5.2 tok/s. Pi 4는 더 느림 (추정 1/2~1/3) | Colab T4 7B 실측 약 35~39 tok/s | 측정 필요 |
| 입력 처리 속도 | Pi 5 실측 TinyLlama 1.1B 약 108 tok/s (llama.cpp) | GPU라 빠름 | — |
| 요청 1회 예상 시간 | 40~55초 (추정) | 2~3초 (추정) | 1~2초 (추정) |
| 비용 | 0원 | g4dn.xlarge 서울 $0.647/시간 → 상시 약 $472/월 | 월 수 달러 |
| 오프라인 이점 | Eureka 호출에 네트워크가 필요해 사실상 없음 | 없음 | 없음 |

결론: Pi 직접 실행은 5초 목표를 맞출 수 없고, GPU 셀프호스팅은 캡스톤 사용량에서 API보다 10~100배 비싸다.
로컬 LLM은 2차 비교 실험으로 둔다 (`projects/desk-pet/ai/README.md`). Pi 4 직접 측정한 공개 벤치마크는 찾지 못했다.

## Notes

1. **보안**: Notion 첨부 `Eureka_Process_API_Test.zip` 안에 실제 `EUREKA_API_KEY`가 든 `.env`가 포함돼 있다
   (값은 이 문서에 기록하지 않는다). 키 폐기·재발급과 Notion 첨부 교체가 필요하다. backend 레포는 `.gitignore`가
   `.env`를 막고 있고 `.env.example`만 추적한다.
2. 기존 문서의 "Ollama (Llama 3 8B)" 온디바이스 항목(05 Raspberry Pi)은 4GB Pi에서 실행할 수 없다.
3. 수치 중 "추정" 표시가 없는 것은 위 sources의 실측·공식 값이다.

## Follow-ups

- [[projects/desk-pet/architecture|1차 아키텍처]]와 `projects/desk-pet/contracts.md`를 팀 회의에서 확정
- Eureka API 키 재발급
- 10/18까지 평가셋 20개로 "음성 → itemId 특정 → 실행" 경로 검증
