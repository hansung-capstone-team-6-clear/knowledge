---
type: concept
topics:
  - "[[wiki/topics/hardware|Hardware]]"
status: draft
sources:
  - "raw/데스크펫 하드웨어 정리안 (MVP).md"
related:
  - "[[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]]"
  - "[[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]"
created: "2026-10-08"
updated: "2026-10-08"
---

# Desk Pet Hardware Components

## Summary

데스크펫 MVP 하드웨어는 Raspberry Pi 4B(4GB)에 I2S 마이크(ICS-43434)·I2S 앰프(MAX98357A)+3W 스피커, SPI LCD(ST7789 1.69인치), WS2812B LED 링 24구, PTT 아케이드 버튼을 붙인 구성이다. 서보(SG90)는 MVP에서 뺐다. 구매 합계는 46,020원으로 예산 200,000원의 약 23%이며, 전원 어댑터와 하우징 재료비는 견적 후 더한다(2026-10-07 기준).

## Details

### 핀맵

| 부품 | 모듈 핀 | Pi 핀 (GPIO / 물리) | 비고 |
|---|---|---|---|
| 마이크 ICS-43434 | VCC / GND | 3.3V (Pin 1) / GND (Pin 9) | 전원 3.3V |
| 마이크 ICS-43434 | SCK / WS / SD | GPIO 18 (Pin 12) / GPIO 19 (Pin 35) / GPIO 20 (Pin 38) | I2S BCLK, LRCLK, Data In |
| 마이크 ICS-43434 | L/R | GND | 왼쪽 채널 고정. 떠 있으면 소리가 안 들어올 수 있음 |
| 앰프 MAX98357A | Vin / GND | 5V (Pin 2) / GND (Pin 6) | 5V 전원, 3W 스피커 직결 |
| 앰프 MAX98357A | BCLK / LRC / DIN | GPIO 18 (Pin 12) / GPIO 19 (Pin 35) / GPIO 21 (Pin 40) | 마이크와 BCLK·LRCLK 공유, Data Out |
| LCD ST7789 | VCC / GND | 3.3V (Pin 17) / GND (Pin 14) | 전원 3.3V |
| LCD ST7789 | SCL / SDA | GPIO 11 (Pin 23) / GPIO 10 (Pin 19) | SPI0_SCLK / SPI0_MOSI |
| LCD ST7789 | CS / DC / RST | GPIO 8 (Pin 24) / GPIO 24 (Pin 18) / GPIO 25 (Pin 22) | SPI0_CE0 / Data Command / Reset |
| LCD ST7789 백라이트 | BLK | 3.3V 직결 | 밝기 조절이 필요하면 GPIO 23으로 변경 |
| LED WS2812B 링 24구 | VCC / GND / DIN | 5V (Pin 4) / GND (Pin 20) / GPIO 12 (Pin 32) | PWM0. I2S의 GPIO 18과 분리. 밝기 상한 30% |
| PTT 버튼 OBSF-30 | 신호 / GND | GPIO 17 (Pin 11) / GND (Pin 25) | 내부 풀업, 누르면 LOW |
| (제외) 서보 SG90 | VCC / GND / PWM | 5V (Pin 2) / GND (Pin 39) / GPIO 13 (Pin 33) | MVP 제외 — GPIO 13·5V Pin 2 공유 문제 해소 |

마이크와 앰프는 같은 I2S 버스(BCLK GPIO 18, LRCLK GPIO 19)를 공유하고 데이터 선만 나뉜다(마이크 In GPIO 20, 앰프 Out GPIO 21). LED 링을 GPIO 12(PWM0)로 둔 것은 I2S가 쓰는 GPIO 18과 겹치지 않게 하기 위해서다.

### 부팅 설정

`/boot/firmware/config.txt`에 추가한다.

```
dtparam=audio=off
dtparam=spi=on
dtoverlay=googlevoicehat-soundcard
```

`googlevoicehat-soundcard` 오버레이 하나로 마이크와 앰프를 함께 여는 설정이며, 실물 녹음·재생 테스트로 확인해야 한다.

### 부품 목록과 예산

| 구분 | 부품 | 수량 | 금액 | 상태 |
|---|---|---|---|---|
| 메인 보드 | Raspberry Pi 4B (4GB) | 1 | 0원 (교내 대여) | 대여 완료 |
| 마이크 | ICS-43434 I2S MEMS | 1 | 11,040원 | 구매 예정 |
| 앰프 | MAX98357A I2S 3W | 1 | 9,090원 | 구매 예정 |
| 스피커 | 4Ω 3W (35×25mm) | 1 | 7,000원 | 구매 예정 |
| 디스플레이 | ST7789 1.69인치 IPS LCD (240×280, SPI) | 1 | 8,090원 | 구매 예정 |
| 상태 링 | WS2812B LED 24구 | 1 | 4,800원 + 배송비 3,000원 | 외경·내경 확인 후 구매 예정 |
| 버튼 | 산와 30mm 아케이드 버튼 (OBSF-30) | 1 | 3,000원 | 구매 예정 |
| 모터 | 서보 SG90 1~2개 | - | 0원 (학과 키트) | MVP 제외 |
| 하우징 | PLA 3D 프린팅 맞춤 하우징 | 1식 | 프린터 무료, 재료비 견적 | 부품 치수 확정 후 설계 |
| 전원 | 어댑터 5V 3A USB-C | 1 | 견적 필요 (보유 시 0원) | 추가 필요 |
| 저장장치 | microSD 32GB 이상 | 1 | 0원 | 보유 (사용 중) |
| 배선 | 점퍼선·브레드보드 | 1식 | 학과 키트 확인 | 추가 필요 |
| 레벨 시프터 | 74AHCT125 (LED 신호가 불안정할 때만) | 1 | 견적 필요 | 선택 |

구매 합계 46,020원은 마이크·앰프·스피커·LCD·LED(배송비 포함)·버튼의 합이다.

### 전원

서보를 빼면서 별도 5V 전원이 필요 없어졌다. LED 밝기를 30%로 제한하면 Pi용 5V 3A 어댑터 하나로 충분하다.

## Connections

- 각 부품을 다루는 소프트웨어 인터페이스: [[wiki/desk-pet-hardware-interfaces|하드웨어 인터페이스]]
- 전체 계층 구조: [[wiki/desk-pet-mvp-architecture|데스크펫 MVP 아키텍처]]
- 서보 제외·Pi 4GB 확정 등의 결정 배경: [[wiki/desk-pet-hardware-mvp-decisions|MVP 결정 기록]]

## Open Questions

- LED 링 외경·내경이 85mm 하우징과 30mm 버튼에 맞는지 확인 후 구매 — needs-update.
- 부품 도착 후 `googlevoicehat-soundcard`로 녹음·재생 동시 동작 테스트 — needs-update.
- 전원 어댑터·하우징 재료비 견적이 아직 합계에 없다.
