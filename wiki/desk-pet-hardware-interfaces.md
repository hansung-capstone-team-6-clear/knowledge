---
type: concept
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/데스크펫 MVP 아키텍처.md"
related:
  - "[[wiki/desk-pet-device-states|장치 상태]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-hardware-components|부품·핀맵·예산]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Hardware Interfaces

## Summary

Pi 쪽 Python 소프트웨어는 7개 인터페이스로 나뉜다. 입력 2개(`IButtonInput`, `IAudioCapture`), 출력 1개(`IHardwareOutput`), 통신 2개(`IBackendConnection`, `IVoiceUpload`), 점검 1개(`IDeviceStatusMonitor`), 그리고 이들을 묶는 상태 관리 `IDeviceStateManager`다. 모두 비동기로 동작한다. 전체 JSON 정의는 원문 `raw/데스크펫 하드웨어 정리안 (MVP).md`에 있다.

## Details

| # | 인터페이스 | 역할 | 호출 방식 | 주요 규칙 |
|---|---|---|---|---|
| 1 | `IButtonInput` | PTT 버튼 눌림·뗌 감지 → `BUTTON_PRESSED` / `BUTTON_RELEASED` | GPIO 17 인터럽트 (내부 풀업, 누르면 LOW) | 디바운싱, 녹음 15초 초과 시 강제 종료 |
| 2 | `IAudioCapture` | 버튼 이벤트에 따라 녹음해 WAV 생성 | ALSA (I2S, `googlevoicehat-soundcard`) | WAV 16kHz · mono · 16bit 고정, 오류 코드 `MIC_ERROR` / `AUDIO_CAPTURE_ERROR`, 15초 초과 시 그때까지의 음성 처리 |
| 3 | `IHardwareOutput` | 스피커·LED·LCD로 상태와 결과 전달 | Edge-TTS 합성 후 ALSA 재생, `rpi_ws281x`(GPIO 12), ST7789 SPI0 | 입력 `ttsText`·`ledState`(6종)·`lcdExpression`·`lcdText`. 장치 일부가 고장 나도 나머지로 안내, TTS 실패 시 LCD 텍스트만. 밝기 상한 0.3 |
| 4 | `IBackendConnection` | Backend WebSocket 상시 연결, 알림·출력 수신 | WebSocket, Python `asyncio` + `websockets` | 수신 전용(Pi는 PING만 보냄). 30초 heartbeat, 3초 간격 재연결, 재연결 시 미전달 알림 재전송 |
| 5 | `IVoiceUpload` | WAV를 Backend로 보내고 최종 출력을 응답으로 받음 | HTTP `POST /api/voice` (multipart/form-data) | 응답 `status` + `sttText` + `output`. 대기 10초, 초과 시 `ERROR` → `IDLE`. 무음이면 다시 말해 달라고 안내 |
| 6 | `IDeviceStatusMonitor` | Wi-Fi와 마이크·스피커·LED·LCD 상태 점검 | Pi 시스템 명령, 장치 초기화 결과 확인 | 60초 주기 + 변화 시. Wi-Fi 끊김 → `ERROR`, 재연결 → `IDLE`. 실패 장치는 제외하고 동작. Backend로 보내지 않음 |
| 7 | `IDeviceStateManager` | 버튼·수신 이벤트로 상태 전환, LED·LCD 출력 요청 | 내부 | 6개 상태와 전이표, `alertWhileBusy`·`stuckState` 규칙 — [[wiki/desk-pet-device-states|장치 상태]] |

### 2026-10-07 정리안에서 바뀐 점

- (1) 호출 방식을 "GPIO Event 또는 Interrupt"에서 GPIO 17 인터럽트로 확정.
- (2) 오디오 형식을 "WAV | PCM, sampleRate 가변"에서 WAV 16kHz · mono · 16bit로 고정.
- (3) `ttsAudio`(바이너리)를 `ttsText`(문장)로 바꾸고 Pi가 Edge-TTS로 합성. `ledState` 4종 → 6종(`THINKING`·`ALERT` 추가), `lcdExpression` 값 목록·`ttsFailure`·밝기 상한 추가.
- (4) 수신 전용으로 정리하고 `receivedEvent`에서 `ack` 제외, 라이브러리 명시.
- (5) 응답을 STT 텍스트만 → STT 텍스트 + 최종 출력으로, 대기 시간 5초 → 10초(STT·LLM 처리 포함).
- (6) `device_status`를 Backend로 보내던 것을 Pi 내부 처리(LED·로그)로.
- (7) 새로 추가. 원본 문서에는 (6)이 두 번 들어가 있었다. 알림 대기 규칙(`alertWhileBusy`) 추가.

## Connections

- 계층 다이어그램 속 배치: [[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]]
- (4)·(5)가 주고받는 메시지: [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]
- (1)·(2)·(3)이 구동하는 부품과 핀: [[wiki/desk-pet-hardware-components|부품·핀맵·예산]]

## Open Questions

- Pi에서 Edge-TTS 합성 시간을 실측해 너무 느리면 서버 합성으로 다시 검토 — needs-update.
- Edge-TTS는 온라인 서비스를 쓰므로 Wi-Fi 끊김 시 (3)의 `ttsFailure` 경로(LCD 텍스트만)로 떨어질 것으로 보인다 — 추론, 원문에 명시 없음.
