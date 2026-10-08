---
title: "Desk Pet Robot Architecture (diagram)"
source: "Notion export (팀 워크스페이스) — Capstone Eureka Process/🏗️ 02 시스템 설계/desk-pet-architecture.html"
author: "클리어픽(6팀)"
created: 2026-10-08
published:
description: "02. 시스템 설계 페이지에 첨부된 인터랙티브 아키텍처 다이어그램(HTML)에서 노드·계층·연결 라벨을 텍스트로 추출. 다이어그램 뷰어 UI 문구와 스타일·스크립트는 제외."
tags:
  - notion-export
  - capstone
  - desk-pet
---

# Desk Pet Robot Architecture

Backend orchestration · Eureka Process integration · proactive deadline alerts

## Nodes (layer — node: responsibility)

| Layer | Node | Responsibility |
| --- | --- | --- |
| Architecture component | User | voice command / feedback |
| Device / Hardware Layer | Hardware | PTT · microphone · LED · LCD · speaker |
| Local Processing Layer | Audio Capture | PCM / WAV recording |
| Local Processing Layer | STT | speech → text · ko-KR |
| Local Processing Layer | Lightweight Intent Model | classify simple commands |
| Application Layer | Intent / LLM Orchestrator | route intent · extract slots · clarify |
| Application Layer | TaskService | create · list · status · complete · update |
| Application Layer | ReminderService | deadline check · deduplication · policy |
| Application Layer | OutputService | TTS · LED · LCD command generation |
| Ports (application port) | TaskPort | create_task · get_tasks · get_task_status · complete_task · update_task |
| Ports (application port) | ReminderRepositoryPort | reminder settings · delivery history |
| Ports (application port) | OutputPublisherPort | publish output event |
| Infrastructure / Adapter Layer (driver) | SchedulerAdapter | periodic reminder trigger |
| Infrastructure / Adapter Layer (adapter) | EurekaTaskAdapter | TaskPort implementation · HTTP mapping |
| Infrastructure / Adapter Layer (adapter) | DatabaseAdapter | reminder persistence |
| Infrastructure / Adapter Layer (adapter) | WebSocketAdapter | proactive_alert delivery |
| External Systems | Eureka Process API | task source of truth · REST |
| External Systems | Raspberry Pi | TTS · LED · LCD output |
| Test-only substitution (test path) | MockTaskAdapter | test-only TaskPort substitute |

## Edge labels

input · PCM audio · text · intent · calls · uses · implements · HTTP / response · triggers · queries · reads / writes · implements · output · publishes · implements · proactive_alert / output · test impl

## Legend

Frontend · Backend · Database · Cloud · Message bus · External
