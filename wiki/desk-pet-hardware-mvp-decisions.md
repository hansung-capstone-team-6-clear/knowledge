---
type: concept
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
  - "raw/deskpet_AI_HW_충돌검토.md"
related:
  - "[[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]]"
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Hardware MVP Decisions

## Summary

2026-10-07 하드웨어 정리안은 기존 설계 문서와 MVP Mock Server 사이의 충돌 16건을 정리했다. 방향은 세 가지다 — MVP Mock Server 형식을 그대로 쓰고, 서보를 빼고, 음성은 파일 업로드로 간다. 16건 중 14건은 하드웨어 쪽에서 확정할 수 있고, Backend 합의가 필요한 건은 2건(1번 음성 업로드, 7번 TTS)이다.

## Details

### 변경 항목 16건

| # | 항목 | 변경 전 | 변경 후 | 이유 |
|---|---|---|---|---|
| 1 | 음성 전송 | WebSocket PCM 스트리밍 (`audio_start` → 바이너리 → `audio_end`) | 녹음 후 WAV를 `POST /api/voice`로 한 번에 업로드 | 청크 처리 불필요, Mock에 엔드포인트 1개만 추가 |
| 2 | Backend→Pi 이벤트 | `proactive_alert`·`state`·`response`·`error` | Mock 이름 유지: `connected`·`proactive_alert`·`output`·`PONG` (오류는 `output`의 `type: error`) | Mock Server와 테스트 코드 재사용 |
| 3 | 처리 중 표시 | Backend가 `state` 이벤트를 보냄 | 버튼을 떼면 Pi가 스스로 `THINKING` 전환 | 메시지 1종 감소, 반응 즉시 |
| 4 | 기기 식별 | `hello` 메시지 + `device_id "pet-01"` | WebSocket 주소의 `user_id`만 사용 | MVP는 사용자 1명당 펫 1대 |
| 5 | 공통 필드 | `v`·`device_id`·`task` 객체 | `event`·`type`·`user_id`·`request_id`·`output` | Mock 형식에 추적용 `request_id`만 추가 |
| 6 | LED 색 결정 | Backend가 `led {color, mode}` 지정 | Pi 상태표가 결정, Backend `led` 무시 | 색 정의 일원화, 노랑 중복 해소 |
| 7 | TTS | 서버 합성 스트리밍 / wav 주소 (미정) | Backend는 문장(`tts`)만, Pi가 Edge-TTS로 합성 | Mock이 이미 문장을 보냄, 오디오 전송 불필요 |
| 8 | Pi→Backend 메시지 | `hello`·`audio_start`·바이너리·`audio_end`·`device_status` | WebSocket은 PING만, 음성은 HTTP | 장치 상태는 Pi 내부에서 LED·로그로 처리 |
| 9 | LED 상태 수 | 3종(디자인) / 4종 | 6종 | 디자인 표정과 알림·오류 모두 포함 |
| 10 | 서보 | SG90 1~2개, GPIO 13 | MVP 제외 (여유가 있으면 추가 기능) | PWM 충돌과 5V 공유 문제 동시 해결 |
| 11 | VAD | Pi에서 Silero VAD 실행 | 없음 (버튼으로 녹음 구간 지정) | PTT 방식이라 불필요 |
| 12 | Pi 통신 라이브러리 | `websocket-client` (동기) | `asyncio` + `websockets` (비동기) | Mock의 `hardware_client.py`가 이미 이 방식 |
| 13 | 성능 목표 | 버튼 100ms, 음성·LCD 동기화 50ms 등 | 목표 대신 구현 후 실측값 기록 | 10/01 미팅 결정 |
| 14 | 구매 합계 | 약 40,020원 (20%) | 약 46,020원 (23%) | 버튼과 LED 배송비 포함 |
| 15 | Pi 사양 | 4GB/8GB | 4GB | 대여받은 실물 기준 |
| 16 | LLM 방식 | OpenAI/Claude API Streaming 추천 | 서버에서 실행하는 Local LLM | 10/01 미팅의 AI 담당 결정 반영, Pi에는 영향 없음 |

### 남은 결정 사항 (2026-10-07 기준)

Backend 합의 2건이 가장 급하고, 나머지는 부품 도착 후 실물로 확인한다.

- [ ] Backend: `POST /api/voice` 신설과 응답에 최종 출력(`output`) 포함 합의. Mock Server에는 파일을 받아 고정 문장을 돌려주는 가짜 엔드포인트부터 추가.
- [ ] Backend: TTS는 문장만 보내고 Pi가 Edge-TTS로 합성하는 방식 합의, 모든 메시지에 `request_id` 추가.
- [ ] 디자인: `THINKING`(흰색 점멸)·`ALERT`(주황 빠른 점멸)·`ERROR`(빨강 점멸) 색 확정.
- [ ] 하드웨어: LED 링 외경·내경이 85mm 하우징과 30mm 버튼에 맞는지 확인 후 구매.
- [ ] 하드웨어: 부품 도착 후 `googlevoicehat-soundcard`로 녹음·재생 테스트.
- [ ] 하드웨어: Pi의 Edge-TTS 합성 시간 실측, 너무 느리면 서버 합성 재검토.
- [ ] 공통: 원본 설계 문서의 LLM 비교표와 VAD 추천 문구를 이 결정에 맞게 수정.

## Connections

- 결정이 반영된 결과 구조: [[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]]
- 항목 1·2·4·5·7·8의 결과 규약: [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]
- 항목 6·9의 결과 상태표: [[wiki/desk-pet-device-states|장치 상태]]
- 항목 10·14·15의 결과 부품표: [[wiki/desk-pet-hardware-components|부품·핀맵·예산]]
- 이 결정들을 AI 처리 구조 문서와 대조한 결과(13건 중 10건 조치 필요): [[wiki/desk-pet-ai-hw-conflict-review|AI–HW 충돌 검토]]

## Open Questions

- 위 체크리스트는 2026-10-07 시점 상태다. 각 항목이 닫히면 이 노트와 관련 노트를 갱신해야 한다 — needs-update.
