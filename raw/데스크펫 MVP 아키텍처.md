---
title: "데스크펫 MVP 아키텍처"
source: "팀 내부 문서 — 데스크펫 MVP 아키텍처.html (로컬 HTML 페이지, URL 없음)"
author: "캡스톤 6팀"
created: 2026-10-08
published:
description: "Pi와 Backend가 음성 요청(HTTP)과 마감 알림(WebSocket) 두 통로만 쓰는 데스크펫 MVP 4계층 아키텍처. HTML 페이지를 Markdown으로 옮김(SVG 다이어그램은 텍스트로 서술)."
tags:
  - desk-pet
  - hardware
  - architecture
---

# 데스크펫 MVP 아키텍처

캡스톤 6팀 · Hardware · MVP

Pi와 Backend는 통로 두 개만 씁니다. 음성 요청은 녹음이 끝난 뒤 HTTP로 한 번에 올리고 응답까지 받습니다. 마감 알림은 Backend가 WebSocket으로 먼저 보냅니다.

## 4계층 다이어그램 (원본 SVG의 텍스트 전사)

범례: 파랑 = 음성 요청 (HTTP) · 주황 = 마감 알림 (WebSocket) · 회색 = 내부 호출 · 회색 점선 = 연결 확인 (PING, 30초)

### 1계층 · 하드웨어

- 마이크 — ICS-43434 · I2S
- PTT 버튼 — OBSF-30 · GPIO 17
- 앰프 + 스피커 — MAX98357A · I2S · 3W
- LED 링 24구 — WS2812B · GPIO 12
- LCD 1.69인치 — ST7789 · SPI0
- 서보 SG90 — MVP 제외 (점선 박스)

### 2계층 · Pi 소프트웨어 (Python)

- IAudioCapture — WAV 16kHz · 최대 15초
- IButtonInput — 눌림·뗌, 디바운싱
- IDeviceStateManager — 상태 6종 전환, THINKING은 Pi가 직접 (강조 박스)
- IHardwareOutput — Edge-TTS 합성 → 스피커 / LED 상태색 · 밝기 30% / LCD 표정 · 텍스트
- IVoiceUpload — `POST /api/voice`
- IBackendConnection — asyncio + websockets
- IDeviceStatusMonitor — Wi-Fi·장치 점검 → ERROR

### 3계층 · Backend

- `POST /api/voice` — WAV 받고 최종 출력으로 응답
- STT — Faster-Whisper
- Intent · Local LLM — 슬롯 추출, 불명확하면 재질문
- WebSocket — `/ws/hardware/{user_id}`
- Scheduler — 마감 임박 감지, done 제외
- Eureka API Adapter — 업무 등록·조회·완료·수정
- DB — 알림 이력 · 미전달 알림

### 4계층 · 외부

- Eureka Process — 업무 데이터 원본

### 연결 (화살표)

- 마이크 → IAudioCapture, PTT 버튼 → IButtonInput (회색)
- IAudioCapture·IButtonInput → IDeviceStateManager → IHardwareOutput (회색)
- IHardwareOutput → 앰프+스피커, LED 링, LCD (회색)
- IDeviceStatusMonitor → IDeviceStateManager (회색)
- IAudioCapture → IVoiceUpload → Backend `POST /api/voice` ("WAV 업로드", 파랑)
- Backend `POST /api/voice` → IVoiceUpload ("응답 output", 파랑) → IDeviceStateManager (파랑)
- `POST /api/voice` → STT → Intent · Local LLM → Eureka API Adapter (파랑)
- Eureka API Adapter → Eureka Process ("API", 회색)
- Eureka API Adapter → Scheduler ("Polling", 주황) → WebSocket (주황)
- Backend WebSocket → IBackendConnection ("알림", 주황) → IDeviceStateManager (주황)
- IBackendConnection → Backend WebSocket ("PING", 회색 점선)
- Scheduler → DB (회색)

## 두 가지 흐름

### 음성 요청 — 사용자가 버튼을 눌러 말할 때

1. 버튼을 누르면 `LISTENING`, 녹음을 시작합니다 (최대 15초).
2. 버튼을 떼면 Pi가 스스로 `THINKING`으로 바꿉니다.
3. WAV를 `POST /api/voice`로 올립니다. 5초 발화는 약 160KB라 0.1~0.3초면 끝납니다.
4. Backend가 STT, Intent·LLM, Eureka 처리를 마치고 최종 출력을 응답으로 돌려줍니다.
5. Pi는 `SPEAKING`으로 바꾸고 문장을 Edge-TTS로 합성해 재생합니다.
6. 재생이 끝나면 `IDLE`로 돌아갑니다. 10초 안에 응답이 없으면 `ERROR`.

### 마감 알림 — Backend가 먼저 알려줄 때

1. Scheduler가 Eureka를 Polling해 마감과 알림 설정을 비교합니다.
2. 마감 임박(기본 1시간 전)이고 완료되지 않은 업무를 찾습니다.
3. WebSocket으로 `proactive_alert`를 보냅니다.
4. Pi는 `ALERT`로 바꾸고 LCD 표시와 TTS를 재생합니다.
5. 끝나면 `IDLE`. 연결이 끊겨 있었다면 다시 연결될 때 Backend가 재전송합니다.

## 장치 상태

| 상태 | 표시 |
|---|---|
| IDLE | 초록 점등 · 대기 |
| LISTENING | 노랑 점멸 · 버튼 누르는 중 |
| THINKING | 흰색 점멸 · 응답 대기 (제안) |
| SPEAKING | 파랑 점멸 · TTS 재생 |
| ALERT | 주황 빠른 점멸 · 마감 알림 (제안) |
| ERROR | 빨강 점멸 · 실패·연결 끊김 (제안) |

## 설계 결정

| 결정 | 이유 |
|---|---|
| 음성은 녹음 후 파일 업로드 | Faster-Whisper는 음성을 다 받은 뒤 인식하므로, 조각으로 보내도 줄어드는 시간은 업로드 0.1~0.3초뿐입니다. |
| TTS는 문장만 받고 Pi가 합성 | Mock Server가 이미 문장을 보내고, 오디오를 주고받는 코드가 필요 없습니다. |
| LED 색은 Pi가 결정 | 색 정의가 한 곳에 모여 LISTENING과 ALERT의 노랑 중복 같은 충돌이 생기지 않습니다. |
| WebSocket 이벤트는 Mock 이름 유지 | `connected`, `proactive_alert`, `output`, `PONG`을 그대로 써서 기존 테스트 코드를 재사용합니다. |
| 서보는 MVP 제외 | LED와 PWM 장치를 같이 쓰는 문제, 앰프와 5V 핀을 같이 쓰는 문제가 함께 사라집니다. |

## 스트리밍으로 바꾸는 기준

- 전체 흐름이 완성되면 업로드, STT, LLM, TTS 구간별 시간을 실제로 잽니다.
- 업로드가 병목일 때만 IVoiceUpload를 WebSocket 스트리밍으로 바꿉니다. 나머지 인터페이스는 그대로 둡니다.
- 체감 속도를 줄이는 데는 TTS를 문장 단위로 나눠 첫 문장부터 재생하는 쪽이 더 효과적입니다.
