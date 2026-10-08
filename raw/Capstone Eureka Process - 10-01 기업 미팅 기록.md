---
title: "10/01 기업 미팅 기록"
source: "Notion export (팀 워크스페이스) — Capstone Eureka Process/10 01 기업 미팅 기록"
author: "클리어픽(6팀)"
created: 2026-10-08
published:
description: "멘토 피드백 — 범위 축소·GitHub 협업·인터페이스 JSON·LLM 역할·Knowledge 활용·Docker/Lambda·역할별 할 일. Notion HTML export를 Markdown으로 변환 — 개인 이메일 제거, 이미지·첨부 파일 미포함."
tags:
  - notion-export
  - capstone
  - desk-pet
---

# 10/01 기업 미팅 기록

## 1. 개발 방향 정리

현재 프로젝트의 가장 중요한 목표는 **기본 기능을 안정적으로 구현하는 것**이다.

추가 기능이나 예외 상황 처리는 기본 기능 구현이 완료된 이후 여유가 있을 경우 진행한다.
- 실제 구현 범위를 현재보다 줄이기 (ex. 업무 연기 기능, 업무 상태 저장 기능, 업무 과거 맥락 저장 기능)

- 핵심 기능 우선 개발

- 네트워크 장애, 복잡한 예외 상황 등은 후순위 처리

- 응답 속도 등 성능 기준은 우선 구현 후 실제 테스트 결과를 바탕으로 개선

- 이번 주부터 본격적으로 개발 시작

## 2. GitHub 협업

### 2-1. 기본 개념
- **Git**
  - 코드 및 문서의 변경 이력 관리

- **GitHub**
  - Git Repository를 팀원들과 공유하고 협업하기 위한 플랫폼

- **Branch**
  - 각자 독립적으로 기능을 개발하기 위한 작업 공간

- **Pull Request**
  - 개발한 Branch를 기존 Branch에 병합하기 전에 검토 및 승인을 요청하는 과정

본격적인 개발 시작 전 팀원 모두 GitHub 활용 방법을 숙지한다.

### 2-2. Repository 구성

GitHub Organization 내부에 역할별 Repository를 분리하여 관리한다.
- `hardware`

- `backend`

- `ai`

각 담당자는 자신이 맡은 Repository에서 개발을 진행하며, 필요한 경우 기능별 Branch를 생성하여 작업한다.

### 2-3. Git Convention

팀원 간 작업 방식을 통일하기 위해 프로젝트 시작 단계에서 **Git Convention(협업 규칙)** 을 정의하는 것을 권장한다.
- Branch 이름 규칙

- Commit Message 규칙

- Pull Request 작성 방식

- Merge 방식

- 공용 Branch 직접 수정 여부

- 코드 리뷰 방식

## 3. 과제 피드백

### 3-1. 인터페이스 정의 수정

각 파트가 서로 독립적으로 개발되기 때문에 통신 규약을 명확하게 정의해야 한다.** → **서로 충돌되는 것 방지

특히 다음 영역 사이의 데이터 전달 구조를 **JSON 형식**으로 정리한다.
- Hardware ↔ Backend

- Backend ↔ AI

- Backend ↔ Eureka Process

- Backend ↔ Scheduler

- Backend → Hardware 알림

### 3-2. 구조 다이어그램 작성

각 담당자가 자신의 개발 영역에 대한 구조 다이어그램을 별도로 작성한다.

단순 시스템 전체 구조뿐만 아니라 실제 구현 수준의 구조까지 작성한다.

### 3-3. LLM 모델 역할 및 채택

현재 결정된 흐름 구조 :
```
사용자 발화
    ↓
STT
    ↓
경량 모델
    ↓
발화 / 작업 유형 분류
    ↓
Local LLM
    ↓
복잡한 자연어 해석
    ↓
Tool / Skill 실행
    ↓
JSON 결과
```

- 모델 별 역할
  - 경량 모델 : 사용자 발화의 의도 및 작업 유형 분류

  - LLM 모델 : 복잡한 명령 해석, 필요한 Tool/Skill 선택, 응답 생성

- LLM 모델 채택
  - LLM 모델 결정 시 검토할 사항들 : 응답 시간, 모델 실행 환경, 메모리 사용량, 모델 크기, 백엔드와의 통신 방식 등

  - Local LLM을 실제 서비스 구조에 포함할 경우 **Local LLM을 호출하고 관리하기 위한 Backend / Serving 구조를 직접 구현해야 할 가능성이 있음**. (참고사항)

### 3-4. 역할별 기술 Stack 정리 필요
- 각 역할별로 사용할 프레임워크 및 기술 구성을 정리해야 함.

### 3-5. 예산 계획 구체적으로 설정
- 부품 사용 비용 / LLM 구독 비용 나눠서 계산

## 4. Knowledge / Second Brain 활용 방식

프로젝트 관련 문서를 **LLM Wiki / Knowledge 형태로 관리**한다.

현재 Notion에 작성된 프로젝트 문서도 Export하여 Markdown 형태로 변환한 뒤 Knowledge에 포함하는 방법을 검토한다.
```
Notion
 ↓
전체 내보내기
 ↓
Markdown
 ↓
내용 정리 / Wiki화
 ↓
knowledge/
```

LLM은 Knowledge 전체를 매번 읽는 것이 아니라 필요한 내용을 검색하여 Context에 포함하는 방식으로 사용한다.

→ 자세한 활용방식은 Github 내용 참고

## 5. 권장 사항

### 5-1. Backend Docker 도입

Backend 개발 환경을 Docker Container로 구성하는 것을 권장받음.

Docker는 Backend 실행에 필요한 환경 전체를 하나의 Container 안에 구성하여 개발자의 PC나 서버 환경이 달라도 거의 동일하게 실행할 수 있게 한다.

장점:
- 팀원별 개발 환경 차이 감소

- 서버 배포 간편

- 동일한 버전의 Python 및 Library 사용 가능

- 실행 환경 재현 가능

### 5-2. Backend 배포

Backend는 AWS Lambda 업로드 검토

주요 고려 사항:
- 서버 비용 최소화

- Scheduler 실행 방식

- WebSocket 사용 가능 여부

- Eureka API 연동

- LLM 서버와의 통신

- 지속 실행이 필요한 기능과 Lambda 구조의 적합성

### 5-3. Tool / Skill 구조

LLM에게 모든 기능을 직접 수행하게 하지 않고 특정 기능을 **Tool / Skill 단위로 분리**하는 방식을 활용한다.

예시 :
```
create_task
update_task
complete_task
get_task_list
search_knowledge
create_alert
```

## 6. 이번 주 해야 할 일

이번주부터 개발 시작하고, 기업 미팅 때마다 각자 다음 내용을 준비한다.
```
1. 이번 주 구현한 기능
2. 실제 실행 결과 / 시연
3. 해결이 필요한 기술적 이슈
4. 다음주 구현 계획
```

미리 해결하기 어려운 기술적 이슈를 정리하여 기업 미팅 전 미리 카톡을 통해 멘토님께 질문한다.

### 6-1. 공통 과제
- Git / GitHub 사용법 별도 학습

- Branch / Pull Request 방식 익히기

- 각 담당 영역 구조 다이어그램 작성

- 각 담당 영역 기술 Stack 확정

- 역할 간 인터페이스 충돌 여부 검토

### 6-2. Backend 담당자
- Backend 기술 Stack 확정

- Docker 환경 구성 검토

- 인터페이스 JSON 재정리

- 백엔드 구조 다이어그램 작성

- 기존 Mock Backend 수정

### 6-3. AI 담당자
- 사용할 LLM 구조 후보 결정

- Local LLM 사용 여부 검토

- llm 구조 다이어그램 작성

- 경량 모델 + Local LLM 구조 검토

- Tool / Skill 구조 조사

### 6-4. Hardware 담당자
- 필요한 부품 확정 및 예산안 작성

- Hardware 구조 다이어그램 작성

- Backend와 주고받을 JSON 인터페이스 정의

 knowledge 활용
