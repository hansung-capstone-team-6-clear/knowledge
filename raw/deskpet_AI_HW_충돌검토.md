---
source_xlsx: "raw/xlsx/deskpet_AI_HW_충돌검토.xlsx"
source_sha256: "2194ead51d7fe74c3c370d1bed0ef1382a7dae295ac56ac669225db283bab29f"
converted_by: manual
converted_at: "2026-10-08"
sheets: 2
conversion_note: "Python 표준 라이브러리(zipfile+XML)로 셀 값을 추출해 시트별 MD 표로 옮겼다. 수식 셀은 저장된 계산값과 수식을 함께 적었다. 셀 서식·드롭다운(상태 열 데이터 유효성 검사)은 옮기지 않았다."
---

# AI 문서 ↔ 하드웨어 정리안(MVP) 충돌 검토

## 시트 1: 충돌 검토

출처: AI 처리 구조 문서, 데스크펫 하드웨어 정리안(MVP) PDF (2026-10-07). '상태' 열(I)만 드롭다운으로 수정하세요.

| # | 구분 | 항목 | AI 문서 | 하드웨어 정리안 (PDF) | PDF 쪽 | 고칠 쪽 | 조치 | 상태 |
|---|---|---|---|---|---|---|---|---|
| 1 | 필수 | 처리 시간 | STT 5초 + Intent 1초 + LLM 5초 + MCP 5초 → 최악 약 16초 | Pi는 10초만 기다리고 ERROR 처리 | p.6, p.10 | 둘 다 | Backend 전체 상한을 9초 이하로 정하고 Pi 대기를 그보다 길게. 시간 초과 요청은 request_id로 중복 실행 방지 | 미착수 |
| 2 | 필수 | 재질문 연결 | 추가 답변을 기존 요청과 연결 (PoC 2) | /api/voice 입력에 대화를 잇는 값이 없고 type에 clarification 없음 | p.10 | 하드웨어 | IVoiceUpload에 conversationId, type에 clarification 추가 | 미착수 |
| 3 | 필수 | STT 위치 | Raspberry Pi에서 STT 또는 Backend | Pi는 WAV만 업로드 | p.6 | AI | 'Raspberry Pi에서' 부분 삭제 | 미착수 |
| 4 | 필수 | robotCommand | HandleSimpleCommand 출력에 robotCommand | Pi는 output.tts·output.lcd만 받고 LED 색은 Pi가 결정 | p.4~5 | AI | robotCommand 삭제 또는 output 형식으로 변경 | 미착수 |
| 5 | 필수 | 상태값 이름 | SUCCESS · NO_SPEECH · STT_ERROR | OK · NO_SPEECH · STT_ERROR · ERROR (Mock도 OK) | p.6, p.10 | AI | SUCCESS를 OK로 변경 | 미착수 |
| 6 | 결정 필요 | 안내 문장(tts) 작성 주체 | responseText, clarification_message만 있고 성공 안내 문장 작성 주체 없음 | Backend가 output.tts를 항상 채운다고 가정 | p.6 | AI/Backend | Output Controller가 항상 output.tts를 만든다고 명시 | 미착수 |
| 7 | 결정 필요 | 음성 형식 | WAV \| PCM | WAV 16kHz · mono · 16bit 고정 | p.7 | AI | 하드웨어 형식에 맞춤 | 미착수 |
| 8 | 결정 필요 | 시간대 | 2026-09-27T15:00:00 (시간대 없음) | +09:00 표기 (Mock 동일) | p.6 | AI | 모든 시각에 +09:00 표기 | 미착수 |
| 9 | 결정 필요 | LLM 방식 | 클라우드 또는 로컬 후보, LLM API 호출 | 서버에서 Local LLM (10/01 미팅) | p.2 #16 | 둘 다 | 확정된 쪽으로 통일 | 미착수 |
| 10 | 결정 필요 | 재질문 시 LCD 표정 | (해당 없음) | lcdExpression 목록에 질문용 표정 없음 | p.8 | 하드웨어 | thinking 사용 또는 question 추가 | 미착수 |
| 11 | 충돌 없음 | request_id | AI 입력에 request_id 사용 | Pi가 request_id 생성 | p.6 | - | 조치 없음 | 해당 없음 |
| 12 | 충돌 없음 | VAD | 언급 없음 | 사용 안 함 (PTT) | p.2 #11 | - | 조치 없음 | 해당 없음 |
| 13 | 충돌 없음 | 불명확 요청 정책 | 임의 실행 없이 재질문 | THINKING → SPEAKING 흐름과 일치 | p.11 | - | 조치 없음 (재질문 연결은 2번에서 처리) | 해당 없음 |

## 시트 2: 요약

| 구분 | 건수 | 완료 | 남은 건수 |
|---|---|---|---|
| 필수 | 5 | 0 | 5 |
| 결정 필요 | 5 | 0 | 5 |
| 충돌 없음 | 3 | 0 | 0 |
| 합계 | 13 | 0 | 10 |

수식: 건수 `=COUNTIF('충돌 검토'!$B$5:$B$17,<구분>)`, 완료 `=COUNTIFS(구분 열=<구분>, 상태 열="완료")`, 남은 건수 `=IF(<구분>="충돌 없음",0,건수-완료)`, 합계 `=SUM(...)`.

'건수·완료'는 '충돌 검토' 시트의 구분·상태 열에서 자동 계산됩니다.
