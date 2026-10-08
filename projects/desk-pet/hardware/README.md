# Hardware 레이어

- 코드: `hansung-capstone-team-6-clear/Hardware`
- 계약: `../contracts.md` § 1 (음성 요청), § 2 (선제 알림), § 5 (상태 매핑)
- 구조: `../architecture.md`

## 1차 범위

| 포함 | 제외 (2차) |
| --- | --- |
| Raspberry Pi 4B, PTT 버튼, I2S 마이크(ICS-43434), I2S 앰프(MAX98357A) + 스피커, ST7789 LCD, WS2812B LED 링 | 서보 모터(SG90), wake word, VAD, 오프라인 동작 |

Pi 프로그램은 판단하지 않는다. 녹음·업로드, 응답 문장을 TTS로 재생, 상태 매핑표대로 LED/LCD 표시, WS 재연결만 맡는다.
LLM API 키는 Pi에 두지 않는다 (디바이스 토큰만).

## 핀맵 (1차안)

Notion "05 Raspberry Pi" 핀맵 **표**를 기준으로 하고, 버튼을 추가하고 서보를 뺐다. 표의 GPIO와 물리 핀 번호는 Pi 4B
핀 배열과 일치한다. **Notion의 배선 그림은 핀 라벨이 실제와 다르니 그림을 보고 배선하지 않는다.**

| 부품 | 신호 | GPIO (물리 핀) | 전원 |
| --- | --- | --- | --- |
| ICS-43434 마이크 | BCLK / LRCLK / DATA IN | GPIO18 (12) / GPIO19 (35) / GPIO20 (38) | 3.3V (1), GND (9) |
| MAX98357A 앰프 | BCLK / LRCLK / DATA OUT | GPIO18 (12) / GPIO19 (35) / GPIO21 (40) — 마이크와 클럭 공유 | 5V (2), GND (6) |
| ST7789 LCD | SCLK / MOSI / CS / DC / RST | GPIO11 (23) / GPIO10 (19) / GPIO8 (24) / GPIO24 (18) / GPIO25 (22) | 3.3V (17), GND (14) |
| WS2812B LED 링 | DIN | GPIO12 (32) | 5V (4), GND (20) |
| PTT 버튼 (제안) | 입력, 내부 풀업 | GPIO17 (11) | GND (아무 GND 핀) |

## 먼저 확인할 것 (10/18까지)

1. **오디오가 1순위 리스크다.** 마이크와 앰프가 I2S 컨트롤러 하나를 공유하므로 녹음과 재생을 함께 여는 overlay가 하나 필요하다.
   후보는 `dtoverlay=googlevoicehat-soundcard` (추정 — 검증 필요).
   완료 기준: `arecord`로 5초 녹음 → `aplay`로 재생 성공.
2. `config.txt`(Bookworm은 `/boot/firmware/`, 이전 OS는 `/boot/`)에 `dtparam=audio=off` — 온보드 오디오가 PWM을 써서 LED(PWM0)와 충돌할 수 있다.
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
    ├── main.py            흐름: 버튼·알림 이벤트 → 상태 → 녹음·업로드·재생
    ├── states.py          상태 6개별 LED 색·LCD 표정 (contracts § 5)
    ├── backend_client.py  POST /voice, WS 수신 (30초 ping, 5초 재연결)
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
