# Implementation Plan: 개인 블로그 서비스 (MVP)

**Branch**: `001-blog-platform` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-blog-platform/spec.md`

## Summary

여러 회원이 각자 블로그 1개를 갖고 글을 쓰고(공개/비공개, 분류, 태그, 이미지), 방문자는 로그인 없이 공개 글을 읽고 검색하며,
회원은 댓글·좋아요·신고로 소통하고, 블로그 주인은 관리 화면에서 글·분류·댓글·통계를 관리하는 서비스다.

기술 방향은 `docs/상세/` 각 파일의 `구현 방식`과 `docs/가이드/`를 따른다: **단일 Spring Boot 서버 + React 화면 + PostgreSQL**,
로그인은 **서버 세션(쿠키, DB 저장)**, 만료가 필요한 임시 값(인증번호, 조회수 중복 방지)은 **Redis**, 이미지 파일은
**S3 방식 저장소(MinIO, 서버 디스크로 교체 가능)**, 메일은 **SMTP**. 결정 근거는 [research.md](research.md)에 있다.

## Technical Context

**Language/Version**: Java 21 (서버), TypeScript 5 (화면) — 버전은 가안, 원천 문서는 언어만 정함

**Primary Dependencies**: Spring Boot 3.x, Spring Security, Spring Session JDBC, Spring Data JPA, Spring Data Redis, Spring Mail, Bean Validation, AWS SDK for Java v2(S3) / React 18 + Vite, React Router, 마크다운 렌더러 + HTML 정화(sanitize)

**Storage**: PostgreSQL 16(업무 데이터, 로그인 세션), Redis 7(인증번호·제한·중복 방지 키, 자동 만료), 이미지 저장소(MinIO 가안 / 서버 디스크)

**Testing**: JUnit 5 + Spring Boot Test, Testcontainers(PostgreSQL, Redis), MockMvc(권한·계약 테스트) / Vitest + Testing Library(화면 규칙)

**Target Platform**: Linux 서버 1대(배포 위치 미정), 최신 데스크톱·모바일 브라우저(폭 360px 이상)

**Project Type**: 웹 애플리케이션 (backend + frontend)

**Performance Goals**: 글 목록·글 상세 2초 이내 (NF-09)

**Constraints**: 단일 서버, MSA 금지, 외부 서비스는 SMTP만 예외, 기본값은 `application.yml` 한 곳, 2주·1인 구현 규모

**Scale/Scope**: 동시 접속 수백 명 이하, 회원 수천 명 이하, 화면 약 20개(방문자 7, 회원 6, 관리 6, 공통 1)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 원칙 | 점검 내용 | 결과 |
|---|---|---|
| I. 요구사항 문서가 원천 | spec의 FR 모두 원천 ID 연결, data-model·contracts에도 FR 번호 표기 | ✅ 통과 |
| II. 서버가 규칙·권한을 지킴 | 모든 쓰기 API에 서버 권한 검사 명시(contracts), 비공개 글은 조회 쿼리 조건으로 제외, 계정 존재 비노출 응답 통일, BCrypt, CSRF·HttpOnly·세션 재발급 | ✅ 통과 |
| III. 단순함 | 단일 서버 + 모듈 방향(`comment·stats·search → post → blog → user`), 세션 방식, 관리자·소셜 로그인 등 범위 밖 기능 제외 | ✅ 통과 |
| IV. 숫자는 한곳에서 | `blog.limits.*` 설정 묶음 하나로 모든 기본값 관리(research R-09) | ✅ 통과 |
| V. 바꿔 끼울 수 있게 | `ImageStorage`, `MailSender` 인터페이스, 접속 정보는 프로필(`dev`/`prod`)과 환경변수 | ✅ 통과 |
| VI. 누구나 읽을 수 있게 | 문서 한국어, 안내 문구는 원천 문서 표를 그대로 사용, 360px 대응 | ✅ 통과 |

**Phase 1 후 재점검**: data-model은 회원–블로그를 1:N 구조로 두되 서버가 1개로 제한(원천 03 `구현 방식`)하고, 새 표는 원천 문서가
예고한 범위만 만들었다. 위반 없음 → Complexity Tracking 비움.

## Project Structure

### Documentation (this feature)

```text
specs/001-blog-platform/
├── spec.md              # 요구사항 (/speckit-specify, clarify 반영)
├── plan.md              # 이 파일
├── research.md          # 기술 결정과 근거
├── data-model.md        # 엔터티·필드·제약·삭제 규칙
├── quickstart.md        # 끝까지 동작하는지 확인하는 시나리오
├── contracts/
│   ├── api.md           # REST API 목록, 권한, 오류 규칙
│   └── screens.md       # 화면 목록과 화면별 상태·문구
├── checklists/requirements.md
└── tasks.md             # 다음 단계 (/speckit-tasks)
```

### Source Code (repository root)

이 저장소(`my-blog-home/docs`)는 문서 저장소다. 코드는 별도 저장소에 아래 구조로 둔다고 가정한다.

```text
backend/
├── build.gradle
└── src/
    ├── main/java/.../blog/
    │   ├── common/        # 설정(기본값 묶음), 오류 응답, 보안 설정, 시간(한국 시간)
    │   ├── user/          # 회원, 가입, 이메일 인증, 로그인 잠금, 마이페이지, 비밀번호 찾기
    │   ├── blog/          # 블로그, 분류, 블로그 설정
    │   ├── post/          # 글, 공개 범위, 태그, 이미지, 좋아요, 신고, 목록
    │   ├── comment/       # 댓글, 새 댓글 표시
    │   ├── search/        # 검색
    │   └── stats/         # 조회수·방문자 집계, 대시보드, 통계
    ├── main/resources/application.yml (+ application-dev.yml, application-prod.yml)
    └── test/java/...      # 모듈별 단위 테스트, 통합 테스트(Testcontainers), API 계약 테스트
frontend/
└── src/
    ├── pages/            # contracts/screens.md의 화면 단위
    ├── components/
    ├── api/              # contracts/api.md 호출
    └── messages.ts       # 안내 문구 모음
docker-compose.yml        # 개발용 PostgreSQL, Redis, MinIO
```

**Structure Decision**: 원천 문서의 "단일 서버 + React 화면"에 맞춰 web application 구조를 고른다. 서버 모듈은
`docs/가이드/06` 3-2의 방향(`comment·stats·search → post → blog → user`)으로만 의존한다.

## 구현 순서 (spec 우선순위 기준)

| 단계 | 범위 | 주요 FR | 끝났다는 기준 |
|---|---|---|---|
| 1 | 기반: 프로젝트, compose, 보안·세션 설정, 기본값 묶음, 오류 응답 | FR-053~057 | 빈 서버가 세션 쿠키·CSRF와 함께 뜬다 |
| 2 | US1 가입·인증·로그인 | FR-001~015 | quickstart 시나리오 A |
| 3 | US2 블로그·분류·글 | FR-022~033 | 시나리오 B |
| 4 | US3 목록·상세·검색 | FR-034~037 | 시나리오 C |
| 5 | US4 댓글·좋아요 | FR-038~040 | 시나리오 D |
| 6 | US5 계정·비밀번호 찾기 | FR-016~021 | 시나리오 E |
| 7 | US6·US7 관리 화면·통계 | FR-044~051 | 시나리오 F |
| 8 | US8 태그·이미지·신고 | FR-041~043 | 시나리오 G |

## 남은 결정 (구현을 막지는 않음)

| 항목 | 지금 쓰는 값 | 정해지면 바뀌는 곳 |
|---|---|---|
| SMTP 계정(Gmail / 네이버) | 개발은 로그 출력 구현 | `MailSender` 구현과 prod 설정 |
| 이미지 저장소 | 개발은 MinIO, 배포는 미정 | `ImageStorage` 구현 |
| 배포 위치, 화면 파일 제공(Nginx / Spring) | 미정 | prod 설정, Dockerfile |
| 로그인 유지 절대 최대 기간 | 없음 | 세션 설정 |

## Complexity Tracking

> 헌법 위반 없음. 비워 둔다.
