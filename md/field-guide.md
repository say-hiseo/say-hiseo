# 🚚 Field Guide

### AI 기반 운송 현장 암묵지 디지털 자산화 서비스

| 기간 | 역할 |
| :---: | :---: |
| 2026.08 ~ 진행 중 | 프론트엔드 100% · 인프라 100% · 백엔드 도메인 및 API 설계 참여 |

**기술 스택**

`React` `TypeScript` `Vite` `Spring Boot` `PostgreSQL` `JPA` `QueryDSL` `Docker` `Docker Compose` `Naver Cloud Platform` `Swagger(springdoc)`

<br/>

## 📌 Service

운송 기사가 진입로·안전·대기 시간처럼 현장에서만 알 수 있는 암묵지를 음성으로 제보하면 AI가 이를 구조화합니다.
이후 다른 운송인의 반복 검증을 거쳐 믿을 수 있는 현장 지식으로 쌓아 가는 서비스입니다.

<br/>

## 🔥 Troubleshooting

### 1. 검증되지 않은 제보가 그대로 노출되는 문제 → 지식 생명주기 기반 도메인 설계

#### ❗ 문제 상황

- 운송 현장의 암묵지를 서비스 데이터로 쌓을 때 **"제보 하나 = 지식 하나"**로 저장하면, 사실 여부가 확인되지 않은 정보까지 사용자에게 노출될 수 있음
- 위치 정보가 빠진 불완전한 제보와 아직 아무도 확인하지 않은 정보를 똑같이 취급하면, 데이터가 많이 쌓여도 서비스가 제공하는 정보 자체를 믿기 어렵다고 판단

#### 🛠️ 해결 과정

암묵지가 **수집 → 검증 → 신뢰할 수 있는 지식**으로 바뀌는 생명주기를 중심으로 도메인과 ERD 설계

**① 제보와 지식의 역할 분리**

- `TACIT_REPORT`: 사용자가 제보한 원본 정보
- `TACIT_KNOWLEDGE`: 검증 과정을 거쳐 서비스에서 활용하는 지식

**② 상태 기반 관리**

- 제보 단계에서 필수 정보가 있는지에 따라 `COMPLETE` / `INCOMPLETE` 상태로 구분
- 이후 방문자의 검증 결과를 `support_count` / `conflict_count`로 누적하는 `VALIDATION` 구조 설계
- 지식의 신뢰 상태를 아래와 같이 구분하고 상태 전이 규칙 정의

| 상태 | 의미 |
| :---: | :--- |
| `UNVERIFIED` | 아직 검증되지 않은 지식 |
| `VERIFIED` | 반복 검증으로 신뢰가 확보된 지식 |
| `CONFLICT` | 검증 결과가 서로 엇갈리는 지식 |
| `EXPIRED` | 더 이상 유효하지 않은 지식 |

**③ 검증 질문 우선순위 설계**

- 검증 질문을 점수에 따라 무작위로 고르지 않고, **`INCOMPLETE` 정보 보완 → `UNVERIFIED` 지식 검증** 순으로 우선순위를 정해 불필요한 반복 질문을 줄임

#### ✅ 결과

- 제보를 단순히 저장하는 구조가 아니라, 불확실한 원본 정보와 검증을 거친 지식을 분리하고 **반복 검증에 따라 신뢰 상태가 바뀌는 도메인 구조** 설계

> 💡 데이터의 양을 늘리는 것보다 **어떤 상태의 데이터를 사용자에게 제공할지 정의하고 관리하는 관점**을 익혔습니다.

<br/>

### 2. Apple Silicon(arm64) ↔ NCP(amd64) Docker Image 호환 문제

#### ❗ 문제 상황

- 로컬 Mac의 Apple Silicon(`arm64`)에서는 정상적으로 빌드되던 Docker Image가 Naver Cloud Platform의 `amd64` 서버에서 실행되지 않음
- `no match for platform in manifest` 로그를 확인한 결과, 사용 중인 Base Image가 서버 Architecture를 지원하지 않는 것이 원인

#### 🛠️ 해결 과정

**① 원인 범위 특정**

- 에러 로그를 바탕으로 애플리케이션 코드가 아닌 **Container Image Architecture 문제**로 범위를 좁힘
- 이미지 태그를 용량 위주로만 골라 실제 배포 환경의 Architecture 호환성을 고려하지 못했다는 점을 확인

**② Multi-Architecture Base Image로 변경**

- `amd64`와 `arm64`를 모두 지원하는 Base Image로 바꿔 실행 문제 해결

**③ 배포 구조 정리**

- Build 환경과 Runtime 환경을 분리한 **Multi-stage Dockerfile** 적용
  - Frontend: `bun` Build → `nginx` 정적 서빙
  - Backend: Gradle Build → JRE Runtime
- Spring Boot · React · PostgreSQL을 `docker-compose.yml`로 구성해 같은 배포 구조 사용
- `.env`와 환경변수 주입 방식으로 Secret과 실행 환경 설정 분리

#### ✅ 결과

- 같은 Dockerfile과 Docker Compose 구성으로 **로컬 `arm64` 환경과 NCP `amd64` 서버에서 모두 실행되는 배포 구조** 완성
- `git pull → docker compose up -d --build` 기반 재배포 절차를 정리해 팀 내 배포 방식 표준화

> 💡 문제가 생겼을 때 애플리케이션 코드만 보지 않고 **운영 환경 · Architecture · Container까지 범위를 넓혀 원인을 추적하는 관점**을 익혔습니다.

<br/>

## ➕ 그 외

- AI Coding Agent를 활용해 전체 Frontend 화면과 반응형 UI 구현
- 요구사항 정의 → Prompt 작성 → 생성 결과 검증 → Backend API 연동까지 전 과정 수행
- AI가 생성한 결과를 그대로 쓰지 않고, 기존 요구사항·인증 구조와 충돌하는지 확인한 뒤 수정

<br/>

[⬅️ 메인으로 돌아가기](../README.md)
