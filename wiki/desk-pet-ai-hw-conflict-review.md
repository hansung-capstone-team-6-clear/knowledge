---
type: concept
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/deskpet_AI_HW_충돌검토.md"
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
related:
  - "[[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
  - "[[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet AI–Hardware Conflict Review

## Summary

AI 처리 구조 문서와 2026-10-07 하드웨어 정리안(MVP)을 대조한 검토표다. 13개 항목 중 필수 5건, 결정 필요 5건, 충돌 없음 3건이며, 2026-10-08 원본 기준으로 조치가 끝난 항목은 없다(남은 10건 모두 미착수). 가장 큰 쟁점은 처리 시간이다 — AI 쪽 최악 처리 시간이 약 16초인데 Pi는 10초만 기다린다.

## Details

### 필수 (5건)

| # | 항목 | 충돌 | 고칠 쪽 | 조치 |
|---|---|---|---|---|
| 1 | 처리 시간 | AI: STT 5초 + Intent 1초 + LLM 5초 + MCP 5초 → 최악 약 16초 / HW: Pi 대기 10초 후 `ERROR` | 둘 다 | Backend 전체 상한을 9초 이하로 정하고 Pi 대기를 그보다 길게. 시간 초과 요청은 `request_id`로 중복 실행 방지 |
| 2 | 재질문 연결 | AI: 추가 답변을 기존 요청과 연결(PoC 2) / HW: `/api/voice` 입력에 대화를 잇는 값이 없고 `type`에 `clarification` 없음 | 하드웨어 | `IVoiceUpload`에 `conversationId`, `type`에 `clarification` 추가 |
| 3 | STT 위치 | AI: "Raspberry Pi에서 STT 또는 Backend" / HW: Pi는 WAV만 업로드 | AI | "Raspberry Pi에서" 부분 삭제 |
| 4 | `robotCommand` | AI: `HandleSimpleCommand` 출력에 `robotCommand` / HW: Pi는 `output.tts`·`output.lcd`만 받고 LED 색은 Pi가 결정 | AI | `robotCommand` 삭제 또는 `output` 형식으로 변경 |
| 5 | 상태값 이름 | AI: `SUCCESS` · `NO_SPEECH` · `STT_ERROR` / HW: `OK` · `NO_SPEECH` · `STT_ERROR` · `ERROR` (Mock도 `OK`) | AI | `SUCCESS`를 `OK`로 변경 |

### 결정 필요 (5건)

| # | 항목 | 충돌 | 고칠 쪽 | 조치 |
|---|---|---|---|---|
| 6 | 안내 문장(tts) 작성 주체 | AI 문서에는 `responseText`, `clarification_message`만 있고 성공 안내 문장을 누가 쓰는지 없음 / HW는 Backend가 `output.tts`를 항상 채운다고 가정 | AI/Backend | Output Controller가 항상 `output.tts`를 만든다고 명시 |
| 7 | 음성 형식 | AI: `WAV \| PCM` / HW: WAV 16kHz · mono · 16bit 고정 | AI | 하드웨어 형식에 맞춤 |
| 8 | 시간대 | AI: `2026-09-27T15:00:00`(시간대 없음) / HW: `+09:00` 표기(Mock 동일) | AI | 모든 시각에 `+09:00` 표기 |
| 9 | LLM 방식 | AI: 클라우드 또는 로컬 후보, LLM API 호출 / HW: 서버에서 Local LLM(10/01 미팅) | 둘 다 | 확정된 쪽으로 통일 |
| 10 | 재질문 시 LCD 표정 | `lcdExpression` 목록에 질문용 표정 없음 | 하드웨어 | `thinking` 사용 또는 `question` 추가 |

### 충돌 없음 (3건)

- 11 `request_id`: AI 입력이 `request_id`를 쓰고 Pi가 생성한다 — 일치.
- 12 VAD: AI 문서에 언급 없음, HW는 PTT라 사용 안 함 — 일치.
- 13 불명확 요청 정책: AI는 임의 실행 없이 재질문, HW의 `THINKING → SPEAKING` 흐름과 맞는다. 재질문을 이전 요청과 잇는 문제는 2번에서 다룬다.

### 고칠 쪽 분포

AI 문서 수정 6건(3·4·5·7·8번 + 6번 공동), 하드웨어 수정 2건(2·10번), 양쪽 2건(1·9번). 하드웨어 쪽이 바꿀 것은 `IVoiceUpload` 입력·`type` 확장과 LCD 표정 하나뿐이고, 대부분은 AI 문서를 하드웨어 정리안·Mock 형식에 맞추는 작업이다.

## Connections

- 1·2·5·8번이 건드리는 메시지 형식: [[wiki/desk-pet-pi-backend-protocol|Pi–Backend 통신 규약]]
- 1·2번의 대상 인터페이스 `IVoiceUpload`, 10번의 `IHardwareOutput`: [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]
- 10번이 바꿀 LCD 표정 목록: [[wiki/desk-pet-device-states|장치 상태]]
- 비교 기준이 된 하드웨어 쪽 결정 16건: [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]

## Open Questions

- 비교 상대인 "AI 처리 구조 문서"는 아직 vault에 없다. 들어오면 이 노트에 링크한다.
- 모든 항목이 원본 기준 미착수다. 엑셀의 상태 열이 바뀌면 이 노트를 갱신해야 한다 — needs-update.
- 1번의 9초 상한을 STT·Local LLM·MCP 실측으로 맞출 수 있는지는 아직 근거가 없다.
