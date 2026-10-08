---
type: concept
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/데스크펫 MVP 아키텍처.md"
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/deskpet_AI_HW_충돌검토.md"
related:
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-device-states|장치 상태]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-hardware-components|부품·핀맵·예산]]"
  - "[[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]"
  - "[[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet MVP Architecture

## Summary

데스크펫 MVP는 하드웨어 · Pi 소프트웨어(Python) · Backend · 외부(Eureka) 4계층으로 나뉜다. Pi와 Backend 사이에는 통로가 두 개뿐이다. 음성 요청은 녹음이 끝난 뒤 WAV 파일을 HTTP `POST /api/voice`로 한 번에 올리고 같은 응답으로 최종 출력을 받는다. 마감 알림은 Backend가 WebSocket `/ws/hardware/{user_id}`로 먼저 보낸다(push). Pi는 30초마다 PING으로 연결을 확인한다.

## Details

### 계층 구성

| 계층 | 구성 요소 |
|---|---|
| 1 · 하드웨어 | 마이크 ICS-43434(I2S), PTT 버튼 OBSF-30(GPIO 17), 앰프 MAX98357A + 3W 스피커(I2S), WS2812B LED 링 24구(GPIO 12), ST7789 1.69인치 LCD(SPI0). 서보 SG90은 MVP 제외 |
| 2 · Pi 소프트웨어 | `IAudioCapture`, `IButtonInput`, `IDeviceStateManager`(중심), `IHardwareOutput`, `IVoiceUpload`, `IBackendConnection`, `IDeviceStatusMonitor` |
| 3 · Backend | `POST /api/voice`, STT(Faster-Whisper), Intent · Local LLM(슬롯 추출, 불명확하면 재질문), WebSocket, Scheduler(마감 임박 감지, done 제외), Eureka API Adapter, DB(알림 이력 · 미전달 알림) |
| 4 · 외부 | Eureka Process — 업무 데이터 원본 |

Pi 쪽 인터페이스 상세는 [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]], 배선은 [[wiki/desk-pet-hardware-components|부품·핀맵·예산]]에 있다.

### 흐름 1 — 음성 요청 (사용자 → Backend)

1. 버튼을 누르면 `LISTENING`, 녹음 시작(최대 15초).
2. 버튼을 떼면 Pi가 스스로 `THINKING`으로 전환한다(Backend 이벤트를 기다리지 않음).
3. WAV를 `POST /api/voice`로 업로드. 5초 발화가 약 160KB라 업로드는 0.1~0.3초.
4. Backend가 STT → Intent·LLM → Eureka 처리를 끝내고 최종 출력(`output`)을 응답 본문에 담는다.
5. Pi는 `SPEAKING`으로 전환하고 `tts` 문장을 Edge-TTS로 합성해 재생한다.
6. 재생이 끝나면 `IDLE`. 10초 안에 응답이 없으면 `ERROR`.

### 흐름 2 — 마감 알림 (Backend → Pi)

1. Scheduler가 Eureka를 Polling해 마감과 알림 설정을 비교한다.
2. 마감 임박(기본 1시간 전)이면서 완료되지 않은 업무를 찾는다.
3. WebSocket으로 `proactive_alert`를 보낸다.
4. Pi는 `ALERT`로 전환하고 LCD 표시와 TTS를 재생한 뒤 `IDLE`로 돌아간다.
5. 연결이 끊겨 있었다면 재연결 시 Backend가 미전달 알림을 재전송한다(DB에 보관).

### 설계 결정

| 결정 | 이유 |
|---|---|
| 음성은 녹음 후 파일 업로드 | Faster-Whisper는 음성을 다 받은 뒤 인식하므로 스트리밍으로 줄일 수 있는 시간은 업로드 0.1~0.3초뿐 |
| TTS는 문장만 받고 Pi가 합성 | Mock Server가 이미 문장을 보내며, 오디오 송수신 코드가 필요 없음 |
| LED 색은 Pi가 결정 | 색 정의가 한 곳에 모여 LISTENING/ALERT 노랑 중복 같은 충돌이 없어짐 |
| WebSocket 이벤트는 Mock 이름 유지 | `connected`·`proactive_alert`·`output`·`PONG`을 그대로 써서 기존 테스트 코드 재사용 |
| 서보는 MVP 제외 | LED와 PWM 공유, 앰프와 5V 핀 공유 문제가 함께 사라짐 |

### 스트리밍 전환 기준

- 전체 흐름이 완성되면 업로드·STT·LLM·TTS 구간별 시간을 실측한다.
- 업로드가 병목일 때만 `IVoiceUpload`를 WebSocket 스트리밍으로 바꾸고 나머지 인터페이스는 유지한다.
- 체감 속도 개선에는 TTS를 문장 단위로 나눠 첫 문장부터 재생하는 편이 더 효과적이다.

## Connections

- 메시지 형식과 필드: [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]
- 상태 전이와 LED/LCD 표시: [[wiki/desk-pet-device-states|장치 상태]]
- 이 구조로 정리되기까지의 16개 변경 항목: [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]
- AI 처리 구조 문서와 이 구조의 충돌 13건: [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]

## Open Questions

- `POST /api/voice`가 응답에 최종 출력까지 담는 방식은 Backend 담당과 합의 전이다(2026-10-07 기준) — needs-update.
- 10초 응답 제한이 STT + Local LLM + Eureka 처리를 실제로 감당하는지 실측 전이다.
- AI 처리 구조 문서는 최악 약 16초(STT 5 + Intent 1 + LLM 5 + MCP 5)를 잡고 있어 위 10초 제한과 정면으로 부딪힌다 — [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 1번.
