---
title: "🏗️ 02. 시스템 설계"
source: "Notion export (팀 워크스페이스) — Capstone Eureka Process/🏗️ 02 시스템 설계"
author: "클리어픽(6팀)"
created: 2026-10-08
published:
description: "4계층 구조도 설명과 일반/선제 알림/복합 요청/예외 처리 흐름. Notion HTML export를 Markdown으로 변환 — 개인 이메일 제거, 이미지·첨부 파일 미포함."
tags:
  - notion-export
  - capstone
  - desk-pet
---

# 🏗️ 02. 시스템 설계

## 1. 4계층 구조도_(이미지 생략)_

1계층 **Device / Hardware Layer**는 Raspberry Pi를 기반으로 사용자와 데스크펫 사이의 물리적인 입출력을 담당한다. Push-to-Talk 버튼 입력, 마이크를 통한 음성 수집, 스피커 출력, LED 상태 표시, LCD 표정 및 텍스트 출력 등을 수행한다.

2계층 **Voice & Local Processing Layer**는 Raspberry Pi 또는 로컬 환경에서 동작하며 Push-to-Talk 상태 관리, 기본 하드웨어 제어, 네트워크 통신을 담당한다. 수집된 음성 데이터는 WebSocket을 통해 상위 계층으로 전달되며, Backend에서 전달된 응답 및 제어 명령을 다시 하드웨어에 반영한다.

3계층 **AI & Backend Layer**는 시스템의 핵심 처리 계층이다. STT를 통해 음성을 텍스트로 변환하고, 경량 Intent 모델을 이용해 사용자의 요청 유형을 우선 분류한다. 단순 요청은 Backend 로직에서 처리하며, 복합적인 요청이나 추가 문맥 해석이 필요한 경우 LLM/MCP를 통해 처리한다. 또한 Scheduler / Polling을 이용한 마감 임박 감지, Eureka Process API 연동, 알림 이력 관리, WebSocket 기반 `proactive_alert` 이벤트 생성을 담당한다.

4계층 **External / Process Layer**는 외부 업무 관리 및 기록 시스템과의 연동을 담당한다. Eureka Process Cloud를 통해 업무 등록, 조회, 완료, 수정 및 상태 관리 기능을 제공하며, 회의 기록, 알림 설정 및 이력 관리에 활용된다.

Backend 처리 결과는 **Output Controller**를 통해 음성 및 시각 출력 명령으로 변환된다. Output Controller는 TTS 음성 출력, LED 상태 명령, LCD 표정 및 텍스트 출력을 생성하여 Raspberry Pi에 전달한다. 또한 Scheduler가 마감 임박 조건을 감지하면 Backend에서 `proactive_alert` 이벤트를 생성하고 WebSocket을 통해 데스크펫에 전달하여 사용자의 별도 요청 없이 능동적인 알림을 제공한다.

## 2. 주요 처리 흐름

전체 처리 흐름은 다음과 같다.

`사용자 음성 입력 → STT → 경량 Intent 분류 → 필요 시 LLM/MCP 처리 → Eureka Process / Second Brain 연동 → Backend 처리 → Output Controller → TTS / LED / LCD 출력`

### 2-1. 일반 음성 요청 처리 흐름

Push-to-Talk 버튼 입력

→ 마이크 음성 녹음

→ Backend로 음성 데이터 전송

→ STT를 통한 음성 → 텍스트 변환

→ 경량 Intent 모델을 통한 요청 유형 분류

→ 단순 요청은 Backend에서 직접 처리

→ 복합 요청 또는 추가 문맥 해석이 필요한 요청은 LLM으로 전달

→ 필요 시 Eureka Process 또는 Second Brain 조회

→ 처리 결과 생성

→ TTS / LED / LCD 출력 정보 생성

→ Raspberry Pi로 결과 전달

→ 음성 및 시각 출력

### 2-2. 마감 임박 선제 알림 흐름

Backend Scheduler / Polling

→ Eureka Process 업무 목록 및 마감 정보 조회

→ 업무 상태 및 마감 시간 확인

→ 사용자 지정 알림 시점과 비교

→ 완료된 업무 제외

→ 알림 조건 충족 여부 판단

→ `proactive_alert` 이벤트 생성

→ WebSocket을 통해 Raspberry Pi로 전달

→ LED / LCD / TTS를 통한 능동 알림 출력

→ 알림 발송 이력 저장

### 2-3. 복합 요청 처리 흐름

STT 결과 수신

→ 경량 Intent 모델에서 요청 분류

→ 복합 요청 또는 신뢰도 기준 미달 판단

→ LLM으로 요청 전달

→ 업무명 / 날짜 / 시간 / 우선순위 등 정보 추출

→ 필요한 MCP Tool 또는 외부 시스템 선택

→ Eureka Process / Second Brain 호출

→ 실행 결과 해석

→ 사용자 응답 생성

→ Output Controller를 통해 출력 정보 생성

### 2-4. 예외 처리 흐름

네트워크 오류 또는 외부 API 실패 발생

→ 오류 유형 확인

→ Timeout 또는 재시도 정책 적용

→ 데이터 변경 요청의 실제 반영 여부 확인

→ 필요한 경우 사용자에게 재입력 또는 재확인 요청

→ 능동 알림 전달 실패 시 미전달 상태 저장

→ 네트워크 복구 후 최신 업무 상태 재확인 후 알림 재전송

## 3. 시스템 구조 및 인터페이스 정의

 desk-pet-architecture.html
