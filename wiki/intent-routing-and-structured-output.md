---
type: pattern
topics:
  - "[[wiki/topics/eureka-desk-pet|AI Desk Pet (Capstone)]]"
status: draft
sources:
  - "raw/Capstone Eureka Process - 03 AI & 데이터.md"
  - "raw/LLM Wiki - AI 데스크 펫 프로젝트.md"
  - "raw/Eureka Process 연동형 업무관리 AI 데스크 펫 - 프로젝트 정리.md"
  - "raw/Capstone Eureka Process - 10-01 기업 미팅 기록.md"
related:
  - "[[wiki/desk-pet-system-architecture|Desk Pet System Architecture]]"
  - "[[wiki/desk-pet-interface-contracts|Desk Pet Interface Contracts]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Intent Routing & Structured Output

## Summary

데스크 펫 AI 계층의 처리 패턴. STT 텍스트를 먼저 **경량 Intent 모델**로 분류해 단순 명령은 LLM 없이 처리하고,
복합·모호하거나 신뢰도가 낮은 요청만 **LLM**으로 넘긴다. LLM은 업무 데이터를 직접 바꾸지 않고, 검증 가능한
**구조화된 JSON**(intent, 파라미터, 재질문 여부)만 반환한다. 실제 변경은 Backend가 검증한 뒤 Eureka/MCP가 수행한다.
목적은 응답 시간·비용 절감과 환각으로 인한 잘못된 업무 변경 방지다.

## Details

### 2단계 라우팅

```
STT 텍스트 → 경량 Intent 분류 (목표 500ms)
   ├─ 단순·명확 → Backend 직접 처리 (HandleSimpleCommand)
   └─ 복합 / 문맥 필요 / 신뢰도 미달 → LLM 분석 (AnalyzeUserRequest)
        → 필요 Tool 선택 → Backend 검증 → Eureka / MCP 실행 → 결과 확인 → 응답 생성
```

| 분류 | 예시 | 처리 |
| --- | --- | --- |
| 단순 명령 | "다시 말해줘" | 규칙 기반 |
| 단순 조회 | "현재 상태 알려줘" | 경량 분류 후 Backend |
| 업무 조회 | "오늘 할 일 뭐야?" | Intent 분류 후 API 조회 |
| 복합 등록 | "금요일 3시까지 보고서 검토하고 2시간 전에 알려줘" | LLM 정보 추출 후 API 호출 |
| 모호한 요청 | "그거 처리해줘" | LLM 또는 Backend가 재질문 |

10/01 기업 미팅에서 정리된 흐름도 같다: 사용자 발화 → STT → 경량 모델(발화·작업 유형 분류) → Local LLM(복잡한
자연어 해석) → Tool/Skill 실행 → JSON 결과. 멘토는 Local LLM을 쓰면 호출·관리용 Serving 구조를 직접 만들어야
할 수 있다고 짚었다.

### 지원 Intent

| Intent | 설명 | 예시 |
| --- | --- | --- |
| `CREATE_TASK` | 업무 등록 | "내일까지 보고서 작성 추가해줘." |
| `GET_TASKS` | 업무 목록 조회 | "오늘 할 일 알려줘." |
| `COMPLETE_TASK` | 업무 완료 | "보고서 완료했어." |
| `GET_TASK_STATUS` | 진행 상태 조회 | "보고서 지금 상태가 뭐야?" |
| `UPDATE_TASK` | 마감 변경·연기 | "보고서 마감 두 시간 미뤄줘." |
| `SET_REMINDER` | 마감 알림 설정 | "보고서 한 시간 전에 알려줘." |
| `UPDATE_REMINDER` | 기존 알림 변경 | "두 시간 전 말고 30분 전에 알려줘." |
| `UNKNOWN` | 미지원·불명확 | 분류 불가 |

### Structured Output

```json
{
  "intent": "CREATE_TASK",
  "need_clarification": false,
  "parameters": {
    "task_name": "보고서 검토",
    "due_at": "2026-09-27T15:00:00",
    "reminder_offset_minutes": 120
  }
}
```

모호한 경우에는 `"need_clarification": true`와 `clarification_message`("어떤 보고서를 말씀하시는 건가요?")를
반환하고 `parameters`는 `null`이다. 입력에는 `request_id`, `stt_text`, `current_time`을 함께 넘겨 상대 날짜를 해석한다.

**출력 검증 규칙 (LLM Wiki)**: intent는 허용 목록 안의 값, `task_name`은 비어 있지 않을 것, 날짜는 타임존 포함
표준 형식, 필수 정보가 없으면 API 미호출, 재질문 플래그가 참이면 데이터 변경 없이 질문, API 결과를 확인한 뒤에만
최종 응답 생성.

### LLM이 하는 일 / 하면 안 되는 일

- **한다**: 복합 요청 의도 파악, 업무명·날짜·시간·우선순위 추출, 누락 판단, Tool/API 호출 필요 여부 판단,
  결과를 사용자 문장으로 변환, 재질문 생성, 대화 맥락 기반 후속 요청 해석.
- **안 한다**: 확인되지 않은 업무 임의 등록·수정, API Key 생성·노출, 성공 확인 전 성공 안내, 모호할 때 임의 선택,
  기준 시각 없이 날짜 추측.

### System Prompt 초안 원칙

요청 분석 모듈로서 의도와 업무 정보를 구조화해 추출하고, 확인할 수 없는 정보를 생성하지 않으며, 대상·날짜가
불명확하면 재질문하고, 필수 정보가 모두 확인된 경우에만 데이터 변경 Tool을 제안하며, 실행 결과 확인 전에는 성공이라
말하지 않고, 항상 지정된 JSON 형식으로 답한다.

### 모델 후보와 평가

| 영역 | 후보 |
| --- | --- |
| STT | Faster-Whisper / Whisper API / Groq Whisper, 로컬 VAD는 Silero |
| LLM (클라우드) | GPT-4o-mini, Claude Haiku |
| LLM (로컬 검증) | Ollama·vLLM 기반 Llama 3 8B, Qwen 2.5 |
| TTS | Edge-TTS (대안 ElevenLabs) |
| Structured Output | Python + Pydantic (JSON Schema 강제) |
| 평가 도구 | Promptfoo, Ragas 검토 |

PoC 기준(박시현 9/30): 업무명·마감일시 추출 성공률 95% 이상, 알림 미지정 시 60분 기본값 100%, 모호 입력 재질문
100% 반환, Intent 500ms 이내·AI 계층 전체 1.5초 이내. 평가 항목은 Intent·정보 추출·재질문·Tool 선택 정확도,
응답 시간, 실행 안정성, 중복 실행 방지, 응답 품질이다.

## Connections

- 인터페이스 원문(ClassifyIntent, AnalyzeUserRequest, CallMCPTool) → [[desk-pet-interface-contracts|Desk Pet Interface Contracts]]
- 재질문·Fallback 정책 → [[desk-pet-requirements-and-policies|Desk Pet Requirements & Policies]]
- 대표 시나리오에서의 사용 → [[proactive-deadline-alert|Proactive Deadline Alert]]

## Open Questions

- **필드명 통일 필요 (needs-update)**: `reminder_offset_minutes`(03. AI) / `reminder_before_minutes`(LLM Wiki) /
  `reminderAt`(AnalyzeUserRequest), `need_clarification` / `needs_clarification` / `requiresClarification`가 섞여 있다.
  10/7 댓글에서 앞부분 통일이 요청됐다.
- `ClassifyIntent` 출력에 `taskName`/`taskId`가 없어 단순 명령의 대상 업무를 어떻게 넘길지 비어 있다(10/7 댓글).
- 최종 LLM 미선정 — 클라우드 API vs Local LLM. 10/01 기준 AI 담당 과제로 남아 있다.
- 피드백 출력(TTS·LED·LCD)을 AI가 생성할지 Output Controller가 생성할지 — 9/30 AI 안은 AI JSON에 `output_feedback`을
  포함했지만, 댓글에서 Output Controller가 따로 있다는 지적이 있었다.
