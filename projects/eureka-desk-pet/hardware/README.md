# Hardware 레이어

- 코드: `hansung-capstone-team-6-clear/Hardware`
- 계약: `../contracts.md` § 1 (음성 요청·시간 제한), § 2 (업무 목록·마감 알람), § 5 (상태 매핑)
- 구조: `../architecture.md`

## 1차 범위

| 포함 | 제외 (2차) |
| --- | --- |
| Raspberry Pi 4B, PTT 버튼, I2S 마이크(ICS-43434), I2S 앰프(MAX98357A) + 스피커, ST7789 LCD, WS2812B LED 링 | 서보 모터(SG90), wake word, VAD, 오프라인 동작 |

Pi 프로그램은 업무를 판단하지 않는다. 녹음·업로드, 응답 문장을 TTS로 재생, 상태 매핑표대로 LED/LCD 표시, 지연 안내, 마감 알람을 맡는다.
LLM API 키는 Pi에 두지 않는다 (디바이스 토큰만).

## 결정된 변경 (2026-11-06, 구현 전)

하드웨어 팀이 동의했다. 형식은 `../contracts.md` § 1·2·5가 정본이다. WebSocket은 쓰지 않는다.

| 변경 | Pi가 할 일 |
| --- | --- |
| 응답 대기 | 10초 → 20초. 20초를 넘으면 실패 처리 (`ERROR`) |
| 지연 안내 | 10초가 지나도 응답이 없으면 "조금 더 기다려주세요". **LCD와 TTS 중 무엇을 쓸지는 PR 리뷰에서 정한다** |
| 목록 동기화 | 부팅 때와 등록·수정·완료 응답을 받은 뒤 `GET /api/tasks` |
| 마감 알람 | `마감 − offset_minutes ≤ 지금 < 마감`이면 알림 재생·LCD 표시 |
| 보낸 기록 | (`item_id`, `due_at`, `offset_minutes`) 기준으로 파일에 저장, 재부팅 후 복구 |

응답 대기 변경은 Backend 시간 제한 변경과 함께 배포한다. Pi가 10초에 끊은 뒤에도 Backend가 업무를 실행하는 일을 막기 위해서다.

## 핀맵 (1차안)

Notion "05 Raspberry Pi" 핀맵 **표**를 기준으로 하고, 버튼을 추가하고 서보를 뺐다. 표의 GPIO와 물리 핀 번호는 Pi 4B
핀 배열과 일치한다. **Notion의 배선 그림은 핀 라벨이 실제와 다르니 그림을 보고 배선하지 않는다.**

| 부품 | 신호 | GPIO (물리 핀) | 전원 |
| --- | --- | --- | --- |
| ICS-43434 마이크 | BCLK / LRCLK / DATA IN | GPIO18 (12) / GPIO19 (35) / GPIO20 (38) | 3.3V (1), GND (9) |
| MAX98357A 앰프 | BCLK / LRCLK / DATA OUT | GPIO18 (12) / GPIO19 (35) / GPIO21 (40) — 마이크와 클럭 공유 | 5V (2), GND (6) |
| ST7789 LCD | SCLK / MOSI / CS / DC / RST | GPIO11 (23) / GPIO10 (19) / GPIO8 (24) / GPIO24 (18) / GPIO25 (22) | 3.3V (17), GND (14) |
| WS2812B LED 링 | DIN | GPIO12 (32) | 5V (4), GND (20) |
| PTT 버튼 (3핀 모듈, 확인) | IN, active-low | GPIO17 (11) | 3.3V (1) — 5V 금지, GND (6) |

## 진행 상황 (2026-10-15)

- **Pi 기본 세팅 완료** — 완료 기준 4개 통과: SSH 접속, `arecord -l`/`aplay -l`에 I2S 카드, `/dev/spidev0.0`, `pip install -r requirements.txt`.
- **OS**: Raspberry Pi OS trixie (Debian 13). 설계 문서의 bookworm과 다르다.
- **config.txt** (`/boot/firmware/`): `dtparam=spi=on`, `dtparam=audio=off`, `dtoverlay=googlevoicehat-soundcard` 적용.
- **I2S 사운드 카드**: card 2 (`sndrpigooglevoi`). 녹음·재생 테스트는 `plughw:2,0`을 쓴다.
  카드 인식만 확인했고, 실제 마이크·앰프 동작은 부품 도착 후 확인한다.
- **버튼 테스트 통과** (GPIO17): 3핀 모듈, active-low(평소 1, 누르면 0), 채터링 없음.
  앱 코드는 `Button(17, pull_up=True)`를 그대로 쓸 수 있다.
- **남은 일**: 오디오 실물 테스트 (아래 1번 완료 기준).

## 먼저 확인할 것 (10/18까지)

1. **오디오가 1순위 리스크다.** 마이크와 앰프가 I2S 컨트롤러 하나를 공유하므로 녹음과 재생을 함께 여는 overlay가 하나 필요하다.
   `dtoverlay=googlevoicehat-soundcard` 적용, 카드 인식 확인 (10/15). 실물 녹음·재생은 미확인.
   완료 기준: `arecord`로 5초 녹음 → `aplay`로 재생 성공.
2. `config.txt`(Bookworm·trixie는 `/boot/firmware/`, 이전 OS는 `/boot/`)에 `dtparam=audio=off` (10/15 적용) — 온보드 오디오가 PWM을 써서 LED(PWM0)와 충돌할 수 있다.
3. LED 밝기는 소프트웨어로 30% 이하로 둔다. 24개 최대 밝기는 약 1.4A(개당 약 60mA)이고 Pi 5V 핀을 앰프와 같이 쓴다.
   Pi의 3.3V 신호로 LED가 불안정하면 레벨 시프터를 단다.
4. 하우징 실측: 디자인 시안은 약 85mm 큐브인데 Pi 4B 보드가 85.6 × 56.5mm이고, 24구 링은 보통 외경이 약 66mm다.
   부품을 실측한 뒤 하우징 크기나 링 크기(12/16구)를 정한다. 밀폐형 PLA면 통풍구를 둔다.

## Plan B

| 문제 | 대안 |
| --- | --- |
| I2S overlay가 끝내 안 됨 | USB 마이크 + USB 스피커 (또는 USB 사운드카드) — 드라이버 불필요 |
| LCD 드라이버 문제 | 1차 시연은 LED + 음성만으로 |
| 버튼 부품 지연 | 브레드보드용 택트 스위치 |

## Hardware 레포 구조

`hansung-capstone-team-6-clear/Hardware#1`에서 셋업했다. 폴더 순서가 곧 작업 순서다. 폴더별 역할은 레포의 README가 정본이다.

```text
Hardware/
├── setup/      ① Pi 처음 준비: OS, 패키지, config.txt, 파이썬 환경
├── parts/      ② 부품 단독 테스트: audio(마이크+앰프+스피커) → button → led → lcd
└── app/        ③ 통합 프로그램
    ├── main.py            흐름: 버튼·알람 이벤트 → 상태 → 녹음·업로드·재생
    ├── states.py          상태 6개별 LED 색·LCD 표정 (contracts § 5)
    ├── backend_client.py  POST /api/voice, GET /api/tasks
    ├── tts.py             edge-tts(`ko-KR-SunHiNeural`) 합성·재생
    ├── devices/           button · audio · led · lcd
    └── assets/faces/      표정 PNG (240×280)
```

- MVC를 쓰지 않는 이유: 화면을 그리는 앱이 아니라 "이벤트 → 상태 → 장치" 프로그램이다. 비유하면 `states.py`가 View, `main.py`가 Controller, 백엔드가 Model이다.
- 마이크는 앰프와 I2S 클럭을 공유해서 `parts/audio`로 묶었다.
- `parts/`에서 검증한 코드를 `app/devices/`로 옮긴다. `main.py`는 `devices/` 함수만 부른다.

## 이 폴더에 둘 문서

- 부품 테스트 결과, 배선 결정, 하우징 치수 → 이 README 또는 `hardware/<topic>.md`
- 코드 실행법, 설치 명령 → Hardware 레포 README
- 하드웨어 메모는 이 폴더에 둔다 (`wiki/`는 여러 프로젝트에서 재사용할 개념 문서 자리다)
