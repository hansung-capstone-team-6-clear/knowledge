---
source_pdf: "raw/pdf/데스크펫 하드웨어 정리안 (MVP).pdf"
source_sha256: "0c12ea0b63f10e657a6f94331d0758b10350ccd7f6003e9af723653d0cd9c263"
converted_by: S2
converted_at: "2026-10-08"
pages: 12
conversion_note: "poppler pdftotext 추출 후 수동 표 재구성. 원문의 취소선(변경 전)·형광(변경 후) 표기가 겹쳐 추출된 표 셀은 열 단위로 복원했고, 문서 내 인라인 칩(멘션)은 추출되지 않아 [칩]으로 표시했다."
---

# 데스크펫 하드웨어 정리안 (MVP)

<!-- page 1 -->

클리어픽 · Oct 7, 2026 · [칩]

## 결정 요약

[칩]는 MVP Mock Server 형식을 그대로 쓰고, 서보를 빼고, 음성은 파일 업로드로 가는 쪽으로 정리했습니다. 충돌 16건 중 14건은 하드웨어 쪽에서 확정할 수 있고, [칩]와 Backend 합의가 필요한 건 2건(1번 음성 업로드, 7번 TTS)입니다.

표기: 취소선 = 변경 전, 형광 표시 = 변경 후

| # | 항목 | 변경 전 | 변경 후 | 이유 |
|---|---|---|---|---|
| 1 | 음성 전송 | WebSocket PCM 스트리밍 (audio_start → 바이너리 → audio_end) | 녹음이 끝난 뒤 WAV를 HTTP POST /api/voice로 한 번에 업로드 | 청크 처리가 필요 없고, Mock에 엔드포인트 1개만 추가하면 됨 |
| 2 | Backend→Pi 이벤트 | proactive_alert · state · response · error | Mock 이름 유지: connected · proactive_alert · output · PONG (오류는 output의 type: error) | Mock Server와 테스트 코드를 그대로 씀 |
| 3 | 처리 중 표시 | Backend가 state 이벤트를 보냄 | 버튼을 떼면 Pi가 스스로 THINKING으로 전환 | 메시지 1종이 줄고 반응이 즉시 나타남 |
| 4 | 기기 식별 | hello 메시지 + device_id "pet-01" | WebSocket 주소의 user_id만 사용 (/ws/hardware/{user_id}) | MVP는 사용자 1명당 펫 1대 |
| 5 | 메시지 공통 필드 | v · device_id · task 객체 | event · type · user_id · request_id · output | Mock 형식에 추적용 request_id만 더함 |
| 6 | LED 색 결정 | Backend가 led {color, mode}를 지정 | Pi가 이벤트로 상태를 정하고 색은 Pi의 상태표를 따름 (Backend의 led 필드는 무시) | 색 정의가 한 곳에 모여 노랑 중복 문제가 사라짐 |
| 7 | TTS | 서버 합성 스트리밍 / wav 주소 (미정) | Backend는 문장(tts)만 보내고 Pi가 Edge-TTS로 합성 | Mock이 이미 문장을 보내고, 오디오 전송이 필요 없음 |

<!-- page 2 -->

| # | 항목 | 변경 전 | 변경 후 | 이유 |
|---|---|---|---|---|
| 8 | Pi→Backend 메시지 | hello · audio_start · 바이너리 · audio_end · device_status | WebSocket은 PING만, 음성은 HTTP | 장치 상태는 Pi 안에서 LED와 로그로 처리 |
| 9 | LED 상태 수 (IHardwareOutput) | 3종(디자인) / 4종 | 6종: IDLE · LISTENING · THINKING · SPEAKING · ALERT · ERROR | 디자인 표정과 알림·오류를 모두 포함 |
| 10 | 서보 | SG90 1~2개, GPIO13 | MVP에서 제외 (여유가 있으면 추가 기능) | PWM 충돌과 5V 공유 문제가 한 번에 해결됨 |
| 11 | VAD | Pi에서 Silero VAD 실행 | 없음 (버튼으로 녹음 구간 지정) | PTT 방식이라 불필요 |
| 12 | Pi 통신 라이브러리 | websocket-client (동기) | asyncio + websockets (비동기) | Mock의 hardware_client.py가 이미 이 방식 |
| 13 | 성능 목표 | 버튼 100ms, 음성·LCD 동기화 50ms 등 | 목표 대신 구현 후 실측값 기록 | 10/01 미팅 결정 |
| 14 | 구매 합계 | 약 40,020원 (20%) | 약 46,020원 (23%) | 버튼과 LED 배송비 포함 |
| 15 | Pi 사양 | 4GB/8GB | 4GB | 대여받은 실물 기준 |
| 16 | LLM 방식 | OpenAI/Claude API Streaming 추천 | 서버에서 실행하는 Local LLM (10/01 미팅) | AI 담당 결정 반영, Pi에는 영향 없음 |

## 하드웨어 핀맵

서보를 빼면서 PWM·5V 공유 문제가 사라졌고, 빠져 있던 버튼·마이크 L/R·LCD 백라이트 연결을 새로 정했습니다.

| 부품명 | 모듈 핀명 | 라즈베리 파이 핀 (GPIO / 물리 핀) | 비고 |
|---|---|---|---|
| 마이크 ICS-43434 | VCC / GND | 3.3V (Pin 1) / GND (Pin 9) | 전원 3.3V |
| 마이크 ICS-43434 | SCK / WS / SD | GPIO 18 (Pin 12) / GPIO 19 (Pin 35) / GPIO 20 (Pin 38) | I2S BCLK, LRCLK, Data In |

<!-- page 3 -->

| 부품명 | 모듈 핀명 | 라즈베리 파이 핀 (GPIO / 물리 핀) | 비고 |
|---|---|---|---|
| 마이크 ICS-43434 (L/R) | L/R | GND 연결 | 왼쪽 채널로 고정, 떠 있으면 소리가 안 들어올 수 있음 |
| 앰프 MAX98357A | Vin / GND | 5V (Pin 2) / GND (Pin 6) | 5V 전원, 3W 스피커 직결 |
| 앰프 MAX98357A | BCLK / LRC / DIN | GPIO 18 (Pin 12) / GPIO 19 (Pin 35) / GPIO 21 (Pin 40) | I2S BCLK·LRCLK 공유, Data Out |
| ST7789 LCD (1.69인치) | VCC / GND | 3.3V (Pin 17) / GND (Pin 14) | 전원 3.3V |
| ST7789 LCD (1.69인치) | SCL / SDA | GPIO 11 (Pin 23) / GPIO 10 (Pin 19) | SPI0_SCLK / SPI0_MOSI |
| ST7789 LCD (1.69인치) | CS / DC / RST | GPIO 8 (Pin 24) / GPIO 24 (Pin 18) / GPIO 25 (Pin 22) | SPI0_CE0 / Data Command / Reset |
| ST7789 LCD (백라이트) | BLK | 3.3V 직결 | 밝기 조절이 필요하면 GPIO 23으로 변경 |
| WS2812B LED 링 (24구) | VCC / GND / DIN | 5V (Pin 4) / GND (Pin 20) / GPIO 12 (Pin 32) | PWM0, I2S GPIO 18과 분리. 밝기 상한 30% |
| 버튼 PTT (OBSF-30) | 신호 / GND | GPIO 17 (Pin 11) / GND (Pin 25) | 내부 풀업, 누르면 LOW |
| 서보 모터 (SG90) | VCC / GND / PWM | 5V (Pin 2) / GND (Pin 39) / GPIO 13 (Pin 33) | MVP 제외, GPIO 13·5V Pin 2 공유 문제 해소 |

## 부팅 설정 (신규)

`/boot/firmware/config.txt`에 추가합니다. 마이크와 앰프를 한 번에 여는 설정은 실물 녹음·재생 테스트로 확인해야 합니다.

```
dtparam=audio=off
dtparam=spi=on
dtoverlay=googlevoicehat-soundcard
```

## 예산

구매 합계는 ~~40,020원 (약 20%)~~ 46,020원 (예산 200,000원의 약 23%)이고, 전원 어댑터와 하우징 재료비는 견적 후 더합니다.

<!-- page 4 -->

| 구분 | 부품 | 수량 | 금액 | 상태 |
|---|---|---|---|---|
| 메인 보드 | Raspberry Pi 4B (4GB) | 1 | 0원 (교내 대여) | 대여 완료 |
| 마이크 | ICS-43434 I2S MEMS 마이크 | 1 | 11,040원 | 구매 예정 |
| 앰프 | MAX98357A I2S 3W 앰프 | 1 | 9,090원 | 구매 예정 |
| 스피커 | 소형 스피커 4Ω 3W (35×25mm) | 1 | 7,000원 | 구매 예정 |
| 디스플레이 | 1.69인치 ST7789 IPS LCD (240×280, SPI) | 1 | 8,090원 | 구매 예정 |
| 상태 링 | WS2812B LED 24구 | 1 | 4,800원 + 배송비 3,000원 | 외경·내경 확인 후 구매 예정 |
| 버튼 | 산와 30mm 아케이드 버튼 (OBSF-30) | 1 | 3,000원 | 구매 예정 |
| 모터 | 서보 SG90 1~2개 | - | 0원 (학과 키트) | MVP 제외 |
| 하우징 | PLA 3D 프린팅 맞춤 하우징 | 1식 | 프린터 무료, 재료비 견적 | 부품 치수 확정 후 설계 |
| 전원 | 어댑터 5V 3A USB-C | 1 | 견적 필요 (보유 시 0원) | 추가 필요 |
| 저장장치 | microSD 32GB 이상 | 1 | 0원 | 보유 (사용 중) |
| 배선 | 점퍼선·브레드보드 | 1식 | 학과 키트 확인 | 추가 필요 |
| 레벨 시프터 | 74AHCT125 (LED 신호가 불안정할 때만) | 1 | 견적 필요 | 선택 |

서보를 빼면서 별도 5V 전원은 필요 없어졌습니다. LED 밝기를 30%로 제한하면 Pi용 5V 3A 어댑터 하나로 충분합니다.

## LED·LCD 상태표

상태는 6개이고, 어느 상태로 갈지는 Pi가 버튼과 받은 이벤트로 직접 정합니다. Backend는 LED 색을 지정하지 않습니다.

<!-- page 5 -->

| 상태 | 들어가는 조건 | LED (color / mode) | LCD expression |
|---|---|---|---|
| IDLE | 부팅 완료, 출력 완료 | green / solid | idle |
| LISTENING | 버튼 누름 | yellow / blink | listening |
| THINKING | 버튼을 뗀 직후 (Pi가 직접 전환) | white / blink | thinking |
| SPEAKING | output 수신 후 TTS 재생 중 | blue / blink | speaking |
| ALERT | proactive_alert 수신 | ~~yellow / blink~~ orange / 빠른 blink | alert |
| ERROR | output의 type이 error, Wi-Fi 끊김, 장치 오류 | red / blink | surprised |

ALERT는 LISTENING과 같은 노랑 점멸이었기 때문에 주황 빠른 점멸으로 바꿨습니다. THINKING·ALERT·ERROR의 색은 제안값이라 디자인 담당과 확인하면 됩니다.

## Backend 메시지

Pi와 Backend는 통로 2개만 씁니다. 알림은 WebSocket으로 받고, 음성 요청은 HTTP 한 번으로 보내고 응답까지 받습니다.

### Pi가 받는 메시지 (WebSocket /ws/hardware/{user_id})

| event | 언제 | Pi가 할 일 |
|---|---|---|
| connected | 접속 직후 | IDLE 유지 |
| proactive_alert | 마감 임박 (기본 1시간 전) | ALERT → tts 재생 → IDLE |
| output | Backend가 먼저 보내는 일반 출력 | SPEAKING(오류면 ERROR) → tts 재생 → IDLE |
| PONG | Pi의 PING에 대한 응답 | 연결 유지 확인 |
| ~~state~~ | ~~버튼을 뗀 직후~~ | 삭제: Pi가 스스로 THINKING 전환 |
| ~~response, error~~ | ~~음성 요청 결과~~ | 삭제: HTTP 응답으로 받음 |

<!-- page 6 -->

```json
{
  "event": "proactive_alert",
  "type": "deadline_approaching",
  "user_id": "team-park-sihyeon",
  "request_id": "alert_<task_id>_<due_at>",
  "output": {
    "tts": "시현 님, 보고서 검토 마감이 1시간 남았어요!",
    "lcd": { "expression": "alert", "text": "마감 1시간 전" }
  }
}
```

Mock Server 형식 그대로이고, 바뀐 점은 request_id 추가와 led 필드 생략(보내도 Pi가 무시) 두 가지입니다.

### Pi가 보내는 메시지

| 통로 | 메시지 | 언제 |
|---|---|---|
| WebSocket (텍스트) | PING | 30초마다 연결 확인 |
| HTTP | POST /api/voice | 버튼을 뗀 뒤 녹음 파일 업로드 |
| ~~WebSocket (바이너리)~~ | ~~hello, audio_start, 바이너리, audio_end, device_status~~ | 삭제 |

### 음성 요청 (신규): POST /api/voice

multipart/form-data로 audio(WAV, 16kHz · mono · 16bit, 최대 15초), user_id, request_id를 보냅니다. Backend는 STT부터 업무 처리까지 끝낸 뒤 출력을 응답 본문에 담아 돌려줍니다.

```json
{
  "request_id": "req_20261007_001",
  "status": "OK",
  "stt_text": "내일 오후 3시까지 보고서 작성 추가해줘",
  "type": "task_created",
  "output": {
    "tts": "보고서 작성 업무를 등록했어요.",
    "lcd": { "expression": "happy", "text": "업무 등록 완료" }
  }
}
```

status는 OK, NO_SPEECH, STT_ERROR, ERROR 중 하나입니다. Pi는 10초 안에 응답이 없으면 ERROR를 표시하고 IDLE로 돌아갑니다.

<!-- page 7 -->

## 하드웨어 인터페이스 (1)~(7)

위 결정에 맞춰 7개 인터페이스를 고쳤습니다. JSON 안에는 표시를 넣을 수 없어서, 각 인터페이스 아래에 바뀐 부분을 따로 적었습니다.

### (1) IButtonInput

```json
{
  "interface": "IButtonInput",
  "description": "Push-to-Talk 버튼의 눌림 및 해제 이벤트를 감지한다.",
  "input": { "buttonState": "pressed | released" },
  "output": { "event": "BUTTON_PRESSED | BUTTON_RELEASED" },
  "callMethod": "GPIO 17 인터럽트 (내부 풀업, 누르면 LOW)",
  "processing": "asynchronous event-driven",
  "errorHandling": {
    "debouncing": "짧은 시간 동안 반복되는 입력은 디바운싱 처리",
    "deviceError": "입력 장치 오류 발생 시 로그 기록"
  },
  "timeout": { "maxRecordingTimeSec": 15, "action": "최대 녹음 시간을 초과하면 녹음을 강제 종료" }
}
```

변경: callMethod GPIO Event 또는 Interrupt → GPIO 17 인터럽트 (내부 풀업, 누르면 LOW)

### (2) IAudioCapture

```json
{
  "interface": "IAudioCapture",
  "description": "버튼 이벤트에 따라 사용자 음성을 녹음하고 WAV 파일을 만든다.",
  "input": { "event": "START_RECORDING | STOP_RECORDING" },
  "output": {
    "requestId": "string",
    "audioFormat": "WAV (16kHz, mono, 16bit)",
    "audioData": "binary",
    "durationMs": "number"
  },
  "callMethod": "ALSA (I2S, googlevoicehat-soundcard)",
  "processing": "asynchronous",
  "errorHandling": {
    "microphoneConnectionFailure": "MIC_ERROR",
    "audioGenerationFailure": "AUDIO_CAPTURE_ERROR"
  },
  "timeout": { "maxRecordingTimeSec": 15, "action": "최대 녹음 시간 초과 시 녹음을 종료하고 현재까지의 음성 데이터를 처리" }
}
```

<!-- page 8 -->

변경: audioFormat WAV | PCM, sampleRate 가변 → WAV 16kHz · mono · 16bit 고정

### (3) IHardwareOutput

```json
{
  "interface": "IHardwareOutput",
  "description": "스피커, LED, LCD를 제어하여 사용자에게 시스템 상태와 처리 결과를 전달한다.",
  "input": {
    "ttsText": "string | null",
    "ledState": "IDLE | LISTENING | THINKING | SPEAKING | ALERT | ERROR",
    "lcdExpression": "idle | listening | thinking | speaking | alert | surprised | happy",
    "lcdText": "string | null"
  },
  "output": { "audioPlayed": "boolean", "ledUpdated": "boolean", "lcdUpdated": "boolean" },
  "callMethod": "Edge-TTS 합성 후 ALSA 재생, rpi_ws281x(GPIO 12), ST7789 SPI0",
  "processing": "asynchronous",
  "errorHandling": {
    "partialDeviceFailure": "특정 출력 장치에 오류가 발생해도 정상 동작하는 다른 출력 장치로 결과를 안내",
    "ttsFailure": "합성 실패 시 LCD 텍스트로만 안내"
  },
  "timeout": { "action": "출력이 비정상적으로 지속되면 종료하고 IDLE로 복귀" },
  "ledBrightnessMax": 0.3,
  "ledStates": {
    "IDLE": { "color": "green", "mode": "solid" },
    "LISTENING": { "color": "yellow", "mode": "blink" },
    "THINKING": { "color": "white", "mode": "blink" },
    "SPEAKING": { "color": "blue", "mode": "blink" },
    "ALERT": { "color": "orange", "mode": "fast_blink" },
    "ERROR": { "color": "red", "mode": "blink" }
  }
}
```

변경: ttsAudio (binary) → ttsText (문장, Pi가 Edge-TTS로 합성) · ledState 4종 → 6종 (THINKING·ALERT 추가) · lcdExpression 값 목록, ttsFailure, 밝기 상한 0.3 추가

<!-- page 9 -->

### (4) IBackendConnection

```json
{
  "interface": "IBackendConnection",
  "description": "Backend와 WebSocket 연결을 유지하고 알림·출력 이벤트를 수신한다.",
  "input": { "wsUrl": "ws://{host}:{port}/ws/hardware/{userId}" },
  "output": {
    "receivedEvent": "connected | proactive_alert | output | PONG",
    "payload": "object",
    "connectionState": "CONNECTED | DISCONNECTED | RECONNECTING"
  },
  "callMethod": "WebSocket, Python asyncio + websockets",
  "processing": "asynchronous, persistent connection",
  "errorHandling": {
    "connectionLost": "3초 간격으로 재연결, 재연결 시 Backend가 미전달 알림을 재전송",
    "invalidMessage": "JSON 파싱 실패 시 버리고 로그 기록"
  },
  "timeout": { "heartbeatIntervalSec": 30, "action": "PONG이 없으면 연결 끊김으로 보고 재연결" }
}
```

변경: 수신 전용 (Pi는 PING만 전송) · receivedEvent에서 ack 제외 · 라이브러리 asyncio + websockets 명시

<!-- page 10 -->

### (5) IVoiceUpload

```json
{
  "interface": "IVoiceUpload",
  "description": "녹음한 WAV를 Backend로 보내고, 처리가 끝난 출력을 응답으로 받는다.",
  "input": { "requestId": "string", "userId": "string", "audio": "WAV binary" },
  "output": {
    "requestId": "string",
    "status": "OK | NO_SPEECH | STT_ERROR | ERROR",
    "sttText": "string | null",
    "type": "task_created | task_completed | ... | error",
    "output": { "tts": "string", "lcd": { "expression": "string", "text": "string" } }
  },
  "callMethod": "HTTP POST /api/voice (multipart/form-data)",
  "processing": "asynchronous",
  "errorHandling": {
    "uploadFailure": "ERROR 상태 표시 후 재입력 안내",
    "noSpeech": "다시 말해 달라는 안내 출력"
  },
  "timeout": { "maxWaitSec": 10, "action": "응답이 없으면 ERROR 표시 후 IDLE로 복귀" }
}
```

변경: 응답 STT 텍스트만 → STT 텍스트 + 최종 출력(output) · 대기 시간 5초 → 10초 (STT와 LLM 처리까지 포함)

<!-- page 11 -->

### (6) IDeviceStatusMonitor

```json
{
  "interface": "IDeviceStatusMonitor",
  "description": "Wi-Fi와 장치 상태를 주기적으로 확인하고, 문제가 있으면 Pi에서 바로 표시한다.",
  "input": { "checkIntervalSec": 60 },
  "output": {
    "wifi": "connected | disconnected",
    "mic": "ok | error",
    "speaker": "ok | error",
    "led": "ok | error",
    "lcd": "ok | error"
  },
  "callMethod": "Raspberry Pi 시스템 명령 및 장치 초기화 결과 확인",
  "processing": "periodic + on change",
  "errorHandling": {
    "wifiDisconnected": "ERROR 상태 표시, 재연결되면 IDLE",
    "deviceFailure": "실패한 장치를 제외하고 나머지로 동작, 로그 기록"
  }
}
```

변경: Backend로 device_status 전송 → Pi 안에서만 처리 (LED·로그)

### (7) IDeviceStateManager

```json
{
  "interface": "IDeviceStateManager",
  "description": "버튼과 수신 이벤트에 따라 장치 상태를 바꾸고, 상태에 맞는 LED·LCD 출력을 요청한다.",
  "states": ["IDLE", "LISTENING", "THINKING", "SPEAKING", "ALERT", "ERROR"],
  "transitions": {
    "IDLE → LISTENING": "BUTTON_PRESSED",
    "LISTENING → THINKING": "BUTTON_RELEASED 또는 15초 초과",
    "THINKING → SPEAKING": "POST /api/voice 응답 수신",
    "IDLE → ALERT": "proactive_alert 수신",
    "SPEAKING / ALERT → IDLE": "TTS 재생 완료",
    "ANY → ERROR": "장치 오류, 통신 실패, 응답 시간 초과"
  },
  "errorHandling": {
    "invalidTransition": "THINKING 중 버튼 입력은 무시",
    "alertWhileBusy": "IDLE이 아닐 때 온 알림은 현재 출력이 끝난 뒤 재생",
    "stuckState": "한 상태가 비정상적으로 길어지면 IDLE로 강제 복귀"
  }
}
```

<!-- page 12 -->

변경: 원본 문서에 없던 인터페이스 추가 (원본은 (6)이 두 번 들어가 있었음) · 알림 대기 규칙 추가 (alertWhileBusy)

## 남은 결정 사항

Backend 담당과 합의할 것 2건이 가장 급하고, 나머지는 부품이 오면 실물로 확인하면 됩니다.

- Backend: POST /api/voice를 만들고 응답에 최종 출력(output)까지 담을지 합의. Mock Server에는 파일을 받아 고정 문장을 돌려주는 가짜 엔드포인트부터 추가
- Backend: TTS는 문장만 보내고 Pi가 Edge-TTS로 합성하는 방식에 합의, 모든 메시지에 request_id 추가
- 디자인: THINKING(흰색 점멸), ALERT(주황 빠른 점멸), ERROR(빨간 점멸) 색 확정
- 하드웨어: LED 링 외경·내경을 확인해 85mm 하우징과 30mm 버튼에 맞는지 보고 구매
- 하드웨어: 부품이 오면 googlevoicehat-soundcard로 녹음·재생을 모두 테스트
- 하드웨어: Pi에서 Edge-TTS 합성 시간을 재서, 너무 느리면 서버 합성으로 다시 검토
- 공통: 원본 문서의 LLM 비교표와 VAD 추천 문구를 이 결정에 맞게 수정
