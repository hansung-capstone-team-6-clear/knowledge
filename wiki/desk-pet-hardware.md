---
type: concept
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 05 Raspberry Pi.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 09-16 회의기록.md"
related:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Hardware

## Summary

데스크 펫 디바이스는 **Raspberry Pi 4B**에 I2S 마이크·I2S 앰프+스피커·SPI LCD·WS2812B LED 링·Push-to-Talk
버튼(선택적으로 SG90 서보)을 붙인 구성이다. 모델 추론은 Pi에서 하지 않고 **VAD만 로컬, STT·LLM·TTS는 서버/API**로
보내는 하이브리드가 권장안이다. 구매 부품은 약 4.3만 원으로, 예산 20만 원의 약 20% 수준이며 Pi·서보·3D 프린터는
교내 자원을 쓴다.

## Details

### GPIO 핀맵 (Raspberry Pi 4B)

| 부품 | 모듈 핀 | Pi 핀 (GPIO / 물리) | 신호 |
| --- | --- | --- | --- |
| ICS-43434 마이크 | VCC / GND | 3.3V (1) / GND (9) | 3.3V 전원 |
| ICS-43434 마이크 | SCK / WS / SD | GPIO18 (12) / GPIO19 (35) / GPIO20 (38) | I2S BCLK, LRCLK, Data In |
| MAX98357A 앰프 | Vin / GND | 5V (2) / GND (6) | 5V, 3W 스피커 직결 |
| MAX98357A 앰프 | BCLK / LRC / DIN | GPIO18 (12) / GPIO19 (35) / GPIO21 (40) | I2S 클럭 공유, Data Out |
| ST7789 LCD 1.69" | VCC / GND | 3.3V (17) / GND (14) | 3.3V |
| ST7789 LCD 1.69" | SCL / SDA | GPIO11 (23) / GPIO10 (19) | SPI0_SCLK / SPI0_MOSI |
| ST7789 LCD 1.69" | CS / DC / RST | GPIO8 (24) / GPIO24 (18) / GPIO25 (22) | SPI0_CE0 / Data·Command / Reset |
| WS2812B LED 24구 | VCC / GND / DIN | 5V (4) / GND (20) / GPIO12 (32) | PWM0 — I2S의 GPIO18과 분리 |
| SG90 서보 | VCC / GND / PWM | 5V (2) / GND (39) / GPIO13 (33) | PWM1 — 목·고개 동작 |

마이크와 앰프가 I2S 클럭(GPIO18/19)을 공유하고, LED는 PWM0(GPIO12)으로 옮겨 **I2S 오디오와 PWM LED 신호 간섭을
피한 것**이 설계 포인트다.

### 온디바이스 vs 클라우드

| 구분 | 완전 온디바이스 | 서버/API | 권장 하이브리드 |
| --- | --- | --- | --- |
| STT | Whisper.cpp (tiny/base) | Whisper API / Faster-Whisper | Silero VAD 로컬 + Faster-Whisper 서버 |
| LLM | Ollama Llama 3 8B (느림) | GPT-4o-mini / Claude / vLLM | API Streaming |
| TTS | Piper / Coqui | ElevenLabs / Edge-TTS | Edge-TTS 또는 ElevenLabs Streaming |
| 지연 | 3~6초 이상 | 1.5~2.5초 | 0.8~1.2초 |
| Pi 부하 | CPU 100%, 발열, LCD 프레임 드롭 | 통신만, 쾌적 | VAD만 로컬, 균형 |

### 부품과 예산

| 부품 | 규격 | 금액 |
| --- | --- | --- |
| 마이크 | ICS-43434 I2S MEMS | 11,040원 |
| 앰프 | MAX98357A I2S Class-D 3W | 9,090원 |
| LCD | ST7789 1.69" IPS 240×280 SPI | 8,090원 |
| 스피커 | 4Ω 3W 소형 사각 35×25mm | 7,000원 |
| LED | WS2812B 네오픽셀 링 24구 | 4,800원 (+배송 3,000원) |
| 버튼 | 산와 30mm 아케이드 버튼 OBSF-30 | 3,000원 |
| 메인보드 | Raspberry Pi 4B 4GB | 교내 대여 완료 |
| 서보 | SG90 | 학과 실습 키트 |
| 하우징 | PLA 3D 프린팅 | 상상파크 메이커스페이스 |

부품 합계 40,020원 + 버튼 3,000원. 10/01 멘토 피드백에 따라 부품 비용과 LLM 구독 비용을 나눠 예산을 다시 짜야 한다.

### 디바이스 소프트웨어 스택 (9/30 안)

Raspberry Pi OS 64-bit, Python 3.10+, `RPi.GPIO`/`pigpio`, `spidev`/`luma.lcd`, `PyAudio`, `websocket-client`.
Pi가 담당하는 인터페이스(IButtonInput, IAudioCapture, IHardwareOutput, IBackendConnection, IVoiceUpload,
IDeviceStatusMonitor)는 [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]에 있다.

### LED 상태 정의

`IDLE` 초록 점등, `LISTENING` 노랑 점멸, `SPEAKING` 파랑 점멸, `ERROR`(색 미정). 녹음 최대 15초, 버튼은 디바운싱.

### 외형

9/16 아이디어(앵무새·학교 마스코트, 고양이 귀 시계, 정육면체·원기둥, 얼굴+몸통)를 거쳐 **큐브 모양**으로 정리됐다.
LCD를 넣어야 해서 평평한 면이 필요하고, 교내 3D 프린터 출력 크기에 제약이 있다.

## Connections

- 계층 위치 → [[desk-pet-system-architecture|Desk Pet System Architecture]] (Device / Hardware Layer)
- 출력 사용 시나리오 → [[proactive-deadline-alert|Proactive Deadline Alert]]

## Open Questions

- LCD 크기 표기 불일치: 핀맵·예산은 1.69", 9/30 기술 스택 표는 2.0" (needs-update).
- 서보 모터 포함 여부: 핀맵에는 있지만 9/30 MVP 범위는 "서보 관절 구동 제외, 시청각 피드백에 집중".
- 9/17 회의 메모에 "라즈베리 파이 + 러스트"가 있으나 이후 문서는 모두 Python 기준이다.
