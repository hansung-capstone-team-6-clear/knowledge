---
type: concept
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 05 Raspberry Pi.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 09-16 회의기록.md"
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
related:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-device-states|장치 상태]]"
  - "[[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Hardware

## Summary

데스크 펫 디바이스는 **Raspberry Pi 4B(4GB)**에 I2S 마이크·I2S 앰프+스피커·SPI LCD·WS2812B LED 링·Push-to-Talk
버튼을 붙인 구성이다. 2026-10-07 하드웨어 정리안(MVP)에서 **서보(SG90)는 제외**하고, Pi는 **녹음·출력만** 맡기로 했다 —
VAD 없음(PTT로 구간 지정), STT·Local LLM은 서버, TTS는 Backend가 보낸 문장을 **Pi가 Edge-TTS로 합성**한다.
구매 부품 합계는 46,020원으로 예산 20만 원의 약 23%이며, Pi·microSD·3D 프린터는 교내·보유 자원을 쓴다.
결정 16건의 변경 전후는 [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]에 있다.

## Details

### GPIO 핀맵 (Raspberry Pi 4B, 2026-10-07 MVP 기준)

| 부품 | 모듈 핀 | Pi 핀 (GPIO / 물리) | 신호 |
| --- | --- | --- | --- |
| ICS-43434 마이크 | VCC / GND | 3.3V (1) / GND (9) | 3.3V 전원 |
| ICS-43434 마이크 | SCK / WS / SD | GPIO18 (12) / GPIO19 (35) / GPIO20 (38) | I2S BCLK, LRCLK, Data In |
| ICS-43434 마이크 | L/R | GND | 왼쪽 채널 고정. 떠 있으면 소리가 안 들어올 수 있음 |
| MAX98357A 앰프 | Vin / GND | 5V (2) / GND (6) | 5V, 3W 스피커 직결 |
| MAX98357A 앰프 | BCLK / LRC / DIN | GPIO18 (12) / GPIO19 (35) / GPIO21 (40) | I2S 클럭 공유, Data Out |
| ST7789 LCD 1.69" | VCC / GND | 3.3V (17) / GND (14) | 3.3V |
| ST7789 LCD 1.69" | SCL / SDA | GPIO11 (23) / GPIO10 (19) | SPI0_SCLK / SPI0_MOSI |
| ST7789 LCD 1.69" | CS / DC / RST | GPIO8 (24) / GPIO24 (18) / GPIO25 (22) | SPI0_CE0 / Data·Command / Reset |
| ST7789 LCD 백라이트 | BLK | 3.3V 직결 | 밝기 조절이 필요하면 GPIO23으로 변경 |
| WS2812B LED 24구 | VCC / GND / DIN | 5V (4) / GND (20) / GPIO12 (32) | PWM0 — I2S의 GPIO18과 분리, 밝기 상한 30% |
| PTT 버튼 OBSF-30 | 신호 / GND | GPIO17 (11) / GND (25) | 내부 풀업, 누르면 LOW |
| SG90 서보 (MVP 제외) | VCC / GND / PWM | 5V (2) / GND (39) / GPIO13 (33) | 제외로 GPIO13 PWM·5V Pin 2 공유 문제 해소 |

마이크와 앰프가 I2S 클럭(GPIO18/19)을 공유하고, LED는 PWM0(GPIO12)으로 옮겨 **I2S 오디오와 PWM LED 신호 간섭을
피한 것**이 설계 포인트다. 10/07 정리안에서 빠져 있던 버튼·마이크 L/R·LCD 백라이트 연결을 새로 정했다.

### 부팅 설정

`/boot/firmware/config.txt`에 `dtparam=audio=off`, `dtparam=spi=on`, `dtoverlay=googlevoicehat-soundcard`를 추가한다.
오버레이 하나로 마이크와 앰프를 함께 여는 설정이라 실물 녹음·재생 테스트로 확인해야 한다.

### 온디바이스 vs 클라우드

| 구분 | 완전 온디바이스 | 서버/API | 권장 하이브리드 |
| --- | --- | --- | --- |
| STT | Whisper.cpp (tiny/base) | Whisper API / Faster-Whisper | Silero VAD 로컬 + Faster-Whisper 서버 |
| LLM | Ollama Llama 3 8B (느림) | GPT-4o-mini / Claude / vLLM | API Streaming |
| TTS | Piper / Coqui | ElevenLabs / Edge-TTS | Edge-TTS 또는 ElevenLabs Streaming |
| 지연 | 3~6초 이상 | 1.5~2.5초 | 0.8~1.2초 |
| Pi 부하 | CPU 100%, 발열, LCD 프레임 드롭 | 통신만, 쾌적 | VAD만 로컬, 균형 |

위 표는 초기 조사안이다. **MVP 확정(2026-10-07)**은 이와 다르다 — VAD는 PTT라 쓰지 않고, STT는 서버의 Faster-Whisper,
LLM은 서버의 Local LLM(10/01 미팅 AI 담당 결정), TTS는 Backend가 문장만 보내고 Pi가 Edge-TTS로 합성한다.

### 부품과 예산 (2026-10-07 기준)

| 구분 | 부품 | 수량 | 금액 | 상태 |
| --- | --- | --- | --- | --- |
| 메인보드 | Raspberry Pi 4B (4GB) | 1 | 0원 (교내 대여) | 대여 완료 |
| 마이크 | ICS-43434 I2S MEMS | 1 | 11,040원 | 구매 예정 |
| 앰프 | MAX98357A I2S Class-D 3W | 1 | 9,090원 | 구매 예정 |
| 스피커 | 4Ω 3W 소형 사각 35×25mm | 1 | 7,000원 | 구매 예정 |
| LCD | ST7789 1.69" IPS 240×280 SPI | 1 | 8,090원 | 구매 예정 |
| LED | WS2812B 네오픽셀 링 24구 | 1 | 4,800원 + 배송 3,000원 | 외경·내경 확인 후 구매 |
| 버튼 | 산와 30mm 아케이드 버튼 OBSF-30 | 1 | 3,000원 | 구매 예정 |
| 서보 | SG90 1~2개 | - | 0원 (학과 키트) | MVP 제외 |
| 하우징 | PLA 3D 프린팅 맞춤 하우징 | 1식 | 프린터 무료, 재료비 견적 | 부품 치수 확정 후 설계 |
| 전원 | 어댑터 5V 3A USB-C | 1 | 견적 필요 (보유 시 0원) | 추가 필요 |
| 저장장치 | microSD 32GB 이상 | 1 | 0원 | 보유 (사용 중) |
| 배선 | 점퍼선·브레드보드 | 1식 | 학과 키트 확인 | 추가 필요 |
| 레벨 시프터 | 74AHCT125 (LED 신호가 불안정할 때만) | 1 | 견적 필요 | 선택 |

구매 합계 46,020원(예산 20만 원의 약 23%) — 이전 표기 40,020원에 버튼과 LED 배송비를 더한 값이다. 전원 어댑터와 하우징
재료비는 견적 후 더한다. 10/01 멘토 피드백에 따라 부품 비용과 LLM 구독 비용을 나눠 예산을 다시 짜야 한다.

서보를 빼면서 별도 5V 전원이 필요 없어졌고, LED 밝기를 30%로 제한하면 Pi용 5V 3A 어댑터 하나로 충분하다.

### 디바이스 소프트웨어 스택 (9/30 안)

Raspberry Pi OS 64-bit, Python 3.10+, `RPi.GPIO`/`pigpio`, `spidev`/`luma.lcd`, `PyAudio`, `websocket-client`.
Pi가 담당하는 인터페이스(IButtonInput, IAudioCapture, IHardwareOutput, IBackendConnection, IVoiceUpload,
IDeviceStatusMonitor)는 [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]에 있다.
10/07 정리안에서 통신 라이브러리는 `websocket-client`(동기)에서 `asyncio` + `websockets`(비동기)로 바꿨고(Mock의 `hardware_client.py`와 같은 방식),
상태 관리 `IDeviceStateManager`를 더해 Pi 인터페이스가 7개가 됐다 — [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]].

### LED 상태 정의

10/07 정리안에서 4종 → 6종으로 늘었다: `IDLE` 초록 점등, `LISTENING` 노랑 점멸, `THINKING` 흰색 점멸, `SPEAKING` 파랑 점멸,
`ALERT` 주황 빠른 점멸, `ERROR` 빨강 점멸(`THINKING`·`ALERT`·`ERROR` 색은 제안값). 색은 Backend가 아니라 Pi의 상태표가 정한다.
상세와 전이 규칙은 [[wiki/desk-pet-device-states|장치 상태]]. 녹음 최대 15초, 버튼은 디바운싱.

### 외형

9/16 아이디어(앵무새·학교 마스코트, 고양이 귀 시계, 정육면체·원기둥, 얼굴+몸통)를 거쳐 **큐브 모양**으로 정리됐다.
LCD를 넣어야 해서 평평한 면이 필요하고, 교내 3D 프린터 출력 크기에 제약이 있다.

## Connections

- 계층 위치 → [[desk-pet-system-architecture|Desk Pet System Architecture]] (Device / Hardware Layer)
- 출력 사용 시나리오 → [[proactive-deadline-alert|Proactive Deadline Alert]]
- 하드웨어 하위 토픽 → [[wiki/topics/hardware|Hardware]] (인터페이스·상태·통신 규약·결정 기록·AI 문서 충돌 검토)

## Open Questions

- LCD 크기 표기 불일치: 핀맵·예산은 1.69", 9/30 기술 스택 표는 2.0". 10/07 정리안은 1.69"로 적었다.
- 서보 모터 포함 여부 → 10/07 정리안에서 MVP 제외로 확정됐다.
- LED 링 외경·내경이 85mm 하우징과 30mm 버튼에 맞는지 확인 후 구매 (needs-update).
- 부품 도착 후 `googlevoicehat-soundcard`로 녹음·재생 동시 동작 테스트, Pi의 Edge-TTS 합성 시간 실측 (needs-update).
- 9/17 회의 메모에 "라즈베리 파이 + 러스트"가 있으나 이후 문서는 모두 Python 기준이다.
