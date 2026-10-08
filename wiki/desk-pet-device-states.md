---
type: concept
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/데스크펫 MVP 아키텍처.md"
  - "raw/deskpet_AI_HW_충돌검토.md"
related:
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Device States

## Summary

데스크펫은 `IDLE` · `LISTENING` · `THINKING` · `SPEAKING` · `ALERT` · `ERROR` 6개 상태를 가진다. 어느 상태로 갈지는 Pi가 버튼 입력과 받은 이벤트로 직접 정하고(`IDeviceStateManager`), 상태별 LED 색과 LCD 표정도 Pi의 상태표가 결정한다. Backend는 LED 색을 지정하지 않는다.

## Details

### 상태표

| 상태 | 들어가는 조건 | LED (color / mode) | LCD expression |
|---|---|---|---|
| `IDLE` | 부팅 완료, 출력 완료 | green / solid | idle |
| `LISTENING` | 버튼 누름 | yellow / blink | listening |
| `THINKING` | 버튼을 뗀 직후 (Pi가 직접 전환) | white / blink | thinking |
| `SPEAKING` | `output` 수신 후 TTS 재생 중 | blue / blink | speaking |
| `ALERT` | `proactive_alert` 수신 | orange / fast_blink | alert |
| `ERROR` | `output`의 `type: error`, Wi-Fi 끊김, 장치 오류 | red / blink | surprised |

- LED 밝기 상한은 30%(`ledBrightnessMax: 0.3`)다. 전원 예산 때문이다([[wiki/desk-pet-hardware-components|부품·핀맵·예산]]).
- `ALERT`는 원래 `LISTENING`과 같은 노랑 점멸이라 구분이 안 돼 주황 빠른 점멸로 바꿨다.
- LCD 표정에는 상태 외에 `happy`도 있다(예: 업무 등록 완료 응답).
- `THINKING`·`ALERT`·`ERROR` 색은 제안값이며 디자인 담당 확정 전이다.

### 전이 규칙

| 전이 | 트리거 |
|---|---|
| `IDLE → LISTENING` | `BUTTON_PRESSED` |
| `LISTENING → THINKING` | `BUTTON_RELEASED` 또는 녹음 15초 초과 |
| `THINKING → SPEAKING` | `POST /api/voice` 응답 수신 |
| `IDLE → ALERT` | `proactive_alert` 수신 |
| `SPEAKING / ALERT → IDLE` | TTS 재생 완료 |
| `ANY → ERROR` | 장치 오류, 통신 실패, 응답 시간 초과(10초) |

### 예외 처리

- `invalidTransition`: `THINKING` 중 버튼 입력은 무시한다.
- `alertWhileBusy`: `IDLE`이 아닐 때 온 알림은 현재 출력이 끝난 뒤 재생한다.
- `stuckState`: 한 상태가 비정상적으로 길어지면 `IDLE`로 강제 복귀한다.
- Wi-Fi가 끊기면 `ERROR`, 재연결되면 `IDLE`(`IDeviceStatusMonitor`, 60초 주기 점검).

## Connections

- 상태 관리 인터페이스 정의: [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]] (7) `IDeviceStateManager`, 출력은 (3) `IHardwareOutput`
- 상태를 바꾸는 이벤트: [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]
- 흐름 안에서의 상태 변화: [[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]]

## Open Questions

- 디자인 담당과 `THINKING`(흰색 점멸), `ALERT`(주황 빠른 점멸), `ERROR`(빨강 점멸) 색 확정 필요 — needs-update.
- `stuckState`의 "비정상적으로 길어지면" 기준 시간이 정해지지 않았다.
- 재질문 상황의 LCD 표정이 정해지지 않았다(`thinking` 재사용 또는 `question` 추가) — [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]] 10번.
