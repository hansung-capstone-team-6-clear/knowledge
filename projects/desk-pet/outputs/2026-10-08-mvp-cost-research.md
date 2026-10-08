---
type: query-output
created: "2026-10-08"
question: "데스크펫 1차 구현(MVP)을 최소 비용으로 하려면 구성요소별로 무엇을 쓰고, 월 운영비는 얼마인가?"
sources:
  - "[[projects/desk-pet/architecture|desk-pet 1차 아키텍처]] — 비용을 산정한 구조"
  - "Notion export: Capstone Eureka Process / 05 Raspberry Pi § 하드웨어 예산 (2026-10-08 내보내기)"
  - "https://ai.google.dev/gemini-api/docs/pricing"
  - "https://cloud.google.com/text-to-speech/pricing"
  - "https://aws.amazon.com/about-aws/whats-new/2025/07/aws-free-tier-credits-month-free-plan/"
  - "https://www.doit.com/compute/spot/ap-northeast-2/g4dn.xlarge"
  - "https://diyai.io/ai-tools/speech-to-text/openai-whisper-api-pricing-2026/"
  - "https://wikidocs.net/blog/@jaehong/9554/"
  - "https://www.morphllm.com/gemini-api-pricing"
  - "Anthropic Claude API 모델 가격표 (2026-09-25 기준)"
---

# 데스크펫 1차 최소 비용 조사

## Answer

**1차는 월 운영비 0원으로 구현할 수 있다.** 조건은 노트북 호스팅, Gemini 무료 등급, Pi에서 edge-tts, SQLite 조합이다.
일회성 비용은 하드웨어 부품 46,020원이고, 하드웨어 예산 200,000원의 약 23%다.
무료 경로가 막혀 유료로 바꿔도 월 약 $1.5~5.6 수준이다. 반대로 GPU 셀프호스팅은 상시 가동 시 월 약 $472라 1차에서 제외한다.

| 조합 | 호스팅 | 음성 인식 + 해석 | TTS | 월 운영비 (월 3,000회) |
| --- | --- | --- | --- | --- |
| **A. 1차 기본** | 팀원 노트북 Docker | Gemini 3.1 Flash-Lite 무료 등급 (오디오 직접 입력) | Pi에서 edge-tts | **0원** |
| B. 유료 전환 (Gemini) | 노트북 | Gemini 3.1 Flash-Lite 유료 등급 | edge-tts 또는 Google TTS 무료 한도 | 약 $1.5 |
| C. 유료 전환 (STT 분리 + Claude) | 노트북 | `gpt-4o-mini-transcribe` + Claude Haiku 4.5 | edge-tts 또는 Google TTS 무료 한도 | 약 $5.6 |
| 제외: GPU 셀프호스팅 | EC2 g4dn.xlarge (서울) | 7~8B 로컬 LLM | — | 상시 약 $472 |

A에서 B·C로 넘어가는 조건은 `architecture.md` § 1차 기술 선택에 있다.

## Evidence

### 사용량 전제 (추정)

| 항목 | 값 |
| --- | --- |
| 음성 요청 | 하루 100회 × 30일 = 월 3,000회 (개발·테스트·시연 합산) |
| 발화 길이 | 평균 5초 → 월 250분 |
| LLM 입력 / 출력 | 요청당 텍스트 맥락 1,200토큰 / 출력 80토큰 |
| TTS 문자 수 | 응답 평균 40자 × 3,000 + 알림 300회 × 30자 ≈ 월 13만 자 |

### 구성요소별 가격

| 구성요소 | 서비스 | 가격 | 출처 |
| --- | --- | --- | --- |
| LLM (오디오 입력 가능) | Gemini 3.1 Flash-Lite | 무료 등급 있음. 유료: 텍스트 입력 $0.25, 오디오 입력 $0.50, 출력 $1.50 (1M 토큰당) | Google 공식 가격표 |
| LLM | Gemini 3.5 Flash-Lite | 무료 등급 있음. 유료: 입력 $0.30 (모든 형식), 출력 $2.50 (1M 토큰당) | Google 공식 가격표 |
| LLM | Claude Haiku 4.5 | 입력 $1, 출력 $5 (1M 토큰당). 오디오 입력 없음 → STT 별도 | Anthropic 가격표 |
| STT | OpenAI `gpt-4o-mini-transcribe` | 분당 $0.003 (`whisper-1`은 $0.006) | 2차 출처 (diyai.io) — 공식 가격표 재확인 필요 |
| TTS | edge-tts (`ko-KR-SunHiNeural`) | 무료, API 키 불필요. Microsoft Edge 읽기 기능을 쓰는 비공식 라이브러리 | wikidocs 블로그 |
| TTS | Google Cloud TTS | 매월 Standard 400만 자, WaveNet·Neural2·Chirp 3 HD 각 100만 자 무료. 이후 Standard $4 / 1M 자 | Google 공식 가격표 |
| 호스팅 | AWS 무료 플랜 (2025-07 개편) | 가입 시 크레딧 $100 + 활동 보상 최대 $100, 가입 후 6개월 또는 크레딧 소진 시 종료 | AWS 공지 |
| GPU | EC2 g4dn.xlarge (T4 16GB), 서울 | 온디맨드 $0.647/시간 | DoiT 가격 페이지 |

### 계산

| 조합 | 계산 | 월 비용 |
| --- | --- | --- |
| B | 3,000 × (1,200 × $0.25 + 80 × $1.50) / 1M = $1.26, 오디오 5초분 토큰 비용 추가 (토큰 수 미확인, 요청당 $0.0001 이하로 추정) | 약 $1.5 |
| C | Claude: 3,000 × (1,200 × $1 + 80 × $5) / 1M = $4.80, STT: 250분 × $0.003 = $0.75 | 약 $5.6 |
| TTS를 Google로 바꿀 때 | 월 13만 자 < Standard 무료 400만 자 | 0원 |
| GPU 상시 | $0.647 × 730시간 | 약 $472 |
| GPU 실험 시간만 | $0.647 × 4시간 × 22일 | 약 $57 |

### 하드웨어 (일회성)

Notion "05 Raspberry Pi" § 하드웨어 예산 기준.

| 부품 | 금액 |
| --- | --- |
| ICS-43434 마이크 | 11,040원 |
| MAX98357A 앰프 | 9,090원 |
| ST7789 1.69" LCD | 8,090원 |
| 3W 4Ω 스피커 | 7,000원 |
| WS2812B 24구 LED 링 | 4,800원 (+ 배송비 3,000원) |
| PTT 버튼 (OBSF-30) | 3,000원 |
| **합계** | **46,020원** |

Raspberry Pi 4B, 서보 모터, 3D 프린터는 교내 자원(대여)이다.

## Notes

1. **Gemini 무료 등급의 데이터 사용**: 공식 가격표에 무료 등급 입력은 "제품 개선에 사용됨"으로 표시된다.
   테스트 workspace의 업무만 쓰고, 실제 회사 업무나 개인 정보는 넣지 않는다.
2. **무료 등급 요청 제한**: 비공식 출처(morphllm) 기준 분당 5~15회, 하루 최대 약 1,000회로 모델마다 다르다
   (`needs-update` — AI Studio에서 실제 한도 확인). 월 3,000회(하루 100회) 사용은 이 범위 안이다.
3. **edge-tts는 비공식**이라 예고 없이 막힐 수 있다. 막히면 Google Cloud TTS 무료 한도로 바꾼다.
   이 경우 키가 필요하므로 합성 위치를 Backend로 옮길지 팀이 정한다.
4. **AWS 무료 플랜은 가입 후 6개월**이다. 2차 외부 배포 직전(11월 이후)에 가입해야 12월 최종 발표까지 기간이 남는다.
5. 오디오 입력 토큰 수와 Gemini의 한국어 음성 인식 정확도는 확인하지 않았다 (`needs-update` — 평가셋으로 측정).
6. 가격은 2026-10-08 조회 기준이다. 모델 이름과 가격은 자주 바뀌므로 전환 시점에 공식 가격표를 다시 확인한다.

## Follow-ups

- 10/18까지 평가셋 20개로 조합 A의 정확도·지연 측정 → 결과를 `ai/eval-YYYY-MM-DD.md`에 기록
- 조합 A에서 B·C로 전환하면 이 문서를 고치지 말고 새 날짜의 output을 추가한다
