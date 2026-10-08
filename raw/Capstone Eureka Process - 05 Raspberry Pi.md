---
title: "⚡05. Raspberry Pi"
source: "Notion export (팀 워크스페이스) — Capstone Eureka Process/⚡05 Raspberry Pi"
author: "클리어픽(6팀)"
created: 2026-10-08
published:
description: "Raspberry Pi 4B 핀맵·온디바이스/클라우드 비교·하드웨어 예산·Hardware 인터페이스. Notion HTML export를 Markdown으로 변환 — 개인 이메일 제거, 이미지·첨부 파일 미포함."
tags:
  - notion-export
  - capstone
  - desk-pet
---

# ⚡05.Raspberry Pi

## 1. 하드웨어 핀맵 (Raspberry Pi 4B GPIO 연결표)_(이미지 생략)_

| 부품명 | 모듈 핀명 | 라즈베리 파이 핀 (GPIO / 물리 핀) | 비고 / 신호 종류 |
| --- | --- | --- | --- |
| **ICS-43434 마이크** | VCC / GND | 3.3V (Pin 1) / GND (Pin 9) | 3.3V 전원 |
| **ICS-43434 마이크** | SCK / WS / SD | GPIO 18 (Pin 12) / GPIO 19 (Pin 35) / GPIO 20 (Pin 38) | I2S BCLK, LRCLK, Data In |
| **MAX98357A 앰프** | Vin / GND | 5V (Pin 2) / GND (Pin 6) | 5V 전원, 3W 스피커 직결 |
| **MAX98357A 앰프** | BCLK / LRC / DIN | GPIO 18 (Pin 12) / GPIO 19 (Pin 35) / GPIO 21 (Pin 40) | I2S BCLK/LRCLK 공유, Data Out |
| **ST7789 LCD (1.69")** | VCC / GND | 3.3V (Pin 17) / GND (Pin 14) | 3.3V SPI 전원 |
| **ST7789 LCD (1.69")** | SCL / SDA | GPIO 11 (Pin 23) / GPIO 10 (Pin 19) | SPI0_SCLK / SPI0_MOSI |
| **ST7789 LCD (1.69")** | CS / DC / RST | GPIO 8 (Pin 24) / GPIO 24 (Pin 18) / GPIO 25 (Pin 22) | SPI0_CE0 / Data Command / Reset |
| **WS2812B LED (24구)** | VCC / GND / DIN | 5V (Pin 4) / GND (Pin 20) / GPIO 12 (Pin 32) | PWM0 — I2S GPIO 18과 분리 |
| **서보 모터 (SG90 등)** | VCC / GND / PWM | 5V (Pin 2) / GND (Pin 39) / GPIO 13 (Pin 33) | PWM1 — 목·고개 동작용 제어 |
| 구분 | 완전 온디바이스 (Local Pi) | 클라우드 / 외부 서버 연동 (API / Server) | 추천 하이브리드 방식 |
| STT 모델 | Whisper.cpp (tiny/base) | OpenAI Whisper API / Faster-Whisper | Silero VAD는 로컬에서 수행하고, Faster-Whisper는 서버 또는 API에서 수행 |
| LLM / Brain | Ollama (Llama 3 8B, 처리 속도 느림) | OpenAI GPT-4o-mini / Claude / Custom vLLM | OpenAI 또는 Claude API를 Streaming 방식으로 사용 |
| TTS 모델 | Piper TTS / Coqui TTS | ElevenLabs / Edge-TTS | Edge-TTS 또는 ElevenLabs를 Streaming 방식으로 사용 |
| 응답 지연 (Latency) | 3~6초 이상, 매우 느림 | 1.5~2.5초 | 0.8~1.2초, 최적 |
| Raspberry Pi 4B 부하 | CPU 사용률 100%, 발열 심함, LCD 프레임 드롭 발생 | 네트워크 통신만 수행하여 비교적 쾌적함 | VAD만 로컬에서 수행하여 성능과 반응성의 균형 확보 |
| 종합 평가 | 모델 실행으로 인한 성능 저하가 큼 | 성능은 좋지만 네트워크 의존성 존재 | **Silero VAD는 로컬, STT·LLM·TTS는 서버/API에서 처리하는 방식이 가장 적합** |

## 2. 하드웨어 예산

### 2-1. 주요 하드웨어 구성

| 구분 | 부품명 및 규격 | 수량 | 단가 / 비고 | 구매 / 확보 링크 |
| --- | --- | --- | --- | --- |
| 메인 보드 | Raspberry Pi 4 Model B (4GB) | 1개 | 보유 자원 또는 교내 대여 | 대여 완료 |
| 마이크 | ICS-43434 I2S MEMS 마이크 모듈 | 1개 | 11,040원 | http://coupang.com/vp/products/9438404384버튼  |
| 앰프 | MAX98357A I2S Class-D 3W 오디오 앰프 | 1개 | 9,090원 | https://www.coupang.com/vp/products/7206616397 |
| 스피커 | 소형 사각 스피커 4Ω 3W (35×25mm) | 1개 | 7,000원, 무료배송 | http://coupang.com/vp/products/7628449301 |
| 디스플레이 | 1.69인치 ST7789 IPS LCD 모듈 (240×280, SPI) | 1개 | 8,090원 | https://www.coupang.com/vp/products/9458784096 |
| 상태 LED | 아두이노 RGB LED 링 24구 WS2812B 네오픽셀 | 1개 | 4,800원, 배송비 3,000원 | https://www.coupang.com/vp/products/9656438280 |
| 모터 | SG90 서보 모터 (고개·목 동작용) | 1~2개 | 학과 실습 키트 활용 | 학과 및 상상파크 자원 활용 |
| 하우징 | PLA 3D 프린팅 맞춤 하우징 | 1식 | 교내 3D 프린터 활용 | 한성대학교 상상파크 메이커스페이스 |
| 버튼 | 아이에스티몰 산와 30mm 아케이드 게임 버튼 (OBSF-30) | 1개 | 3,000원 | https://www.coupang.com/vp/products/204404568 |

### 2-2. 구매 예정 부품 비용

| 부품 | 금액 |
| --- | --- |
| ICS-43434 I2S MEMS 마이크 | 11,040원 |
| MAX98357A I2S 앰프 모듈 | 9,090원 |
| ST7789 1.69인치 IPS LCD | 8,090원 |
| 3W 4Ω 소형 스피커 | 7,000원 |
| WS2812B 네오픽셀 LED 링 24구 | 4,800원 |
| **총 구매 비용** | **40,020원** |
| 버튼 추가  | +3,000원 |

전체 예산은 **200,000원**이며, Raspberry Pi, 서보 모터, 3D 프린팅 장비 등은 교내 보유 자원과 학과 기자재를 활용하여 구매 비용을 최소화하였다.

현재 확정된 구매 부품의 총비용은 **약 40,020원**으로, 전체 예산의 약 **20% 수준**이다.

프로젝트 비용 절감을 위해 다음과 같은 교내 자원을 활용한다.
- Raspberry Pi 4 대여

- SG90 서보 모터 및 일부 실습 부품

- 한성대학교 상상파크 메이커스페이스 3D 프린터

- PLA 기반 맞춤형 데스크펫 하우징 제작

### 1. Hardware

**역할 및 책임**

사용자와 데스크펫 사이의 물리적인 입출력을 담당한다.
- Push-to-Talk 버튼 입력 감지

- 사용자 음성 녹음

- 음성 데이터 생성 및 전송

- 스피커 출력

- LED 상태 제어

- LCD 표정 및 텍스트 출력

- Wi-Fi 및 하드웨어 상태 감지

#### (1) 버튼 입력 인터페이스 : IButtonInput
```
{
  "interface": "IButtonInput",
  "description": "Push-to-Talk 버튼의 눌림 및 해제 이벤트를 감지한다.",
  "input": {
    "buttonState": "pressed | released"
  },
  "output": {
    "event": "BUTTON_PRESSED | BUTTON_RELEASED"
  },
  "callMethod": "Raspberry Pi GPIO Event 또는 Interrupt",
  "processing": "asynchronous event-driven",
  "errorHandling": {
    "debouncing": "짧은 시간 동안 반복되는 입력은 디바운싱 처리",
    "deviceError": "입력 장치 오류 발생 시 로그 기록"
  },
  "timeout": {
    "maxRecordingTimeSec": 15,
    "action": "최대 녹음 시간을 초과하면 녹음을 강제 종료"
  }
}
```

#### (2) 음성 입력 인터페이스 : IAudioCapture
```
{
  "interface": "IAudioCapture",
  "description": "버튼 이벤트에 따라 사용자 음성을 녹음하고 음성 데이터를 생성한다.",
  "input": {
    "event": "START_RECORDING | STOP_RECORDING"
  },
  "output": {
    "requestId": "string",
    "audioFormat": "WAV | PCM",
    "audioData": "binary",
    "durationMs": "number",
    "sampleRate": "number"
  },
  "callMethod": "Raspberry Pi Internal Audio Device API",
  "processing": "asynchronous",
  "errorHandling": {
    "microphoneConnectionFailure": "MIC_ERROR",
    "audioGenerationFailure": "AUDIO_CAPTURE_ERROR"
  },
  "timeout": {
    "maxRecordingTimeSec": 15,
    "action": "최대 녹음 시간 초과 시 녹음을 종료하고 현재까지의 음성 데이터를 처리"
  }
}
```

#### (3) 출력 장치 제어 인터페이스 : IHardwareOutput
```
{
  "interface": "IHardwareOutput",
  "description": "스피커, LED, LCD를 제어하여 사용자에게 시스템 상태와 처리 결과를 전달한다.",
  "input": {
    "ttsAudio": "binary | null",
    "ledState": "IDLE | LISTENING | SPEAKING | ERROR",
    "lcdExpression": "string | null",
    "lcdText": "string | null"
  },
  "output": {
    "audioPlayed": "boolean",
    "ledUpdated": "boolean",
    "lcdUpdated": "boolean"
  },
  "callMethod": "Raspberry Pi Internal Device Control Function",
  "processing": "asynchronous",
  "errorHandling": {
    "partialDeviceFailure": "특정 출력 장치에 오류가 발생해도 정상 동작하는 다른 출력 장치를 통해 가능한 범위에서 사용자에게 결과를 안내"
  },
  "timeout": {
    "action": "음성 출력 또는 장치 제어가 비정상적으로 지속될 경우 출력을 종료하고 IDLE 상태로 복귀"
  },
  "ledStates": {
    "IDLE": {
      "color": "green",
      "mode": "solid"
    },
    "LISTENING": {
      "color": "yellow",
      "mode": "blink"
    },
    "SPEAKING": {
      "color": "blue",
      "mode": "blink"
    }
  }
}
```

#### (4) 네트워크 통신 인터페이스 : IBackendConnection
```
{
  "interface": "IBackendConnection",
  "description": "Backend와 WebSocket 연결을 유지하고, 출력 명령(proactive_alert, output)을 수신하며 장치 이벤트를 전송한다.",
  "input": {
    "wsUrl": "ws://{host}:{port}/ws/hardware/{userId}",
    "userId": "string",
    "sendMessage": "object | null"
  },
  "output": {
    "receivedEvent": "connected | proactive_alert | output | PONG | ack",
    "payload": "object",
    "connectionState": "CONNECTED | DISCONNECTED | RECONNECTING"
  },
  "callMethod": "WebSocket (Raspberry Pi → Backend)",
  "processing": "asynchronous, persistent connection",
  "errorHandling": {
    "connectionLost": "일정 간격으로 재연결 시도, 재연결 시 Backend가 미전달 알림을 재전송",
    "invalidMessage": "JSON 파싱 실패 시 메시지를 버리고 로그 기록"
  },
  "timeout": {
    "heartbeatIntervalSec": 30,
    "action": "PING에 PONG 응답이 없으면 연결 끊김으로 간주하고 재연결"
  }
}
```

#### (5) 음성 전송 인터페이스 : IVoiceUpload
```
{
  "interface": "IVoiceUpload",
  "description": "IAudioCapture가 생성한 음성 데이터를 Backend로 전송하고 STT 결과를 받는다.",
  "input": {
    "requestId": "string",
    "userId": "string",
    "audioFormat": "WAV",
    "audioData": "binary"
  },
  "output": {
    "requestId": "string",
    "status": "OK | NO_SPEECH | STT_ERROR",
    "text": "string | null"
  },
  "callMethod": "HTTPS POST /api/voice (multipart/form-data)",
  "processing": "asynchronous",
  "errorHandling": {
    "uploadFailure": "UPLOAD_ERROR, LED ERROR 표시 후 재입력 안내",
    "noSpeech": "NO_SPEECH 수신 시 다시 말해 달라는 안내 출력"
  },
  "timeout": {
    "maxWaitSec": 5,
    "action": "응답이 없으면 요청을 실패로 처리하고 IDLE로 복귀"
  }
}
```

#### (6) 장치 상태 감지 인터페이스 : IDeviceStatusMonitor
```
{
  "interface": "IDeviceStatusMonitor",
  "description": "Wi-Fi 연결, 마이크·스피커·LED·LCD 상태를 주기적으로 확인하고 변경 시 Backend에 보고한다.",
  "input": {
    "checkIntervalSec": 60
  },
  "output": {
    "event": "device_status",
    "wifi": "connected | disconnected",
    "mic": "ok | error",
    "speaker": "ok | error",
    "led": "ok | error",
    "lcd": "ok | error"
  },
  "callMethod": "Raspberry Pi 시스템 명령 및 장치 초기화 결과 확인",
  "processing": "periodic + on change",
  "errorHandling": {
    "wifiDisconnected": "LED ERROR 표시, 재연결 후 device_status 재전송",
    "deviceFailure": "실패한 장치를 제외하고 나머지 장치로 동작 유지"
  }
}
```

#### (6) 장치 상태 감지 인터페이스 : IDeviceStatusMonitor
