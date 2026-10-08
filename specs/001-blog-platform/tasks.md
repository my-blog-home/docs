---

description: "개인 블로그 서비스 (MVP) 구현 할 일 목록"
---

# Tasks: 개인 블로그 서비스 (MVP)

**Input**: Design documents from `/specs/001-blog-platform/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: spec은 테스트를 따로 요구하지 않지만, 헌법 "개발 흐름과 품질 기준"이 **권한 검사, 비공개 글 노출, 인증번호·로그인 잠금, 계정 존재 비노출**은 자동 테스트로 확인하라고 한다(SHOULD). 그래서 이 네 가지에 대한 테스트 작업만 넣었다. 나머지는 [quickstart.md](quickstart.md)로 확인한다.

**Organization**: 사용자 시나리오(US1~US8)별로 묶었다. 각 단계가 끝나면 quickstart의 해당 시나리오로 따로 확인할 수 있다.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 다른 파일을 다루고 앞선 미완료 작업에 기대지 않아 동시에 할 수 있음
- **[Story]**: 연결된 사용자 시나리오 (US1~US8)

## Path Conventions

코드는 이 문서 저장소가 아닌 별도 코드 저장소에 둔다(plan.md). 경로가 길어 아래 줄임말을 쓴다.

| 줄임말 | 실제 경로 |
|---|---|
| `api/` | `backend/src/main/java/com/myblog/` |
| `res/` | `backend/src/main/resources/` |
| `test/` | `backend/src/test/java/com/myblog/` |
| `web/` | `frontend/src/` |

서버 모듈은 `common`, `user`, `blog`, `post`, `comment`, `search`, `stats`이며 의존 방향은 `comment·stats·search → post → blog → user`만 허용한다(plan.md).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 빈 프로젝트를 만들고 개발용 부품을 띄운다

- [x] T001 코드 저장소에 plan.md 구조대로 `backend/`, `frontend/` 폴더와 루트 `README.md`, `.gitignore`(`.env`, 빌드 결과물 제외) 만들기
- [x] T002 Spring Boot 3 + Java 21 Gradle 프로젝트 만들기, 의존성(Web, Security, Session JDBC, Data JPA, Data Redis, Mail, Validation, PostgreSQL 드라이버, Flyway, AWS SDK v2 S3, Testcontainers) 추가 in `backend/build.gradle`
- [x] T003 [P] React 18 + Vite + TypeScript 프로젝트 만들기, React Router·마크다운 렌더러·HTML 정화 라이브러리·Vitest 추가 in `frontend/package.json`
- [x] T004 [P] 개발용 PostgreSQL 16, Redis 7, MinIO(버전 고정, 포트 9000은 localhost에만) 정의 in `docker-compose.yml`, 비밀번호는 `.env.example`
- [x] T005 [P] Vite 개발 서버에서 `/api`, `/images`를 서버로 넘기는 프록시 설정 in `frontend/vite.config.ts` (research R-02)
- [ ] T006 [P] 서버·화면 코드 서식 도구(Spotless, ESLint, Prettier) 설정 in `backend/build.gradle`, `frontend/.eslintrc.cjs`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 모든 사용자 시나리오가 기대는 공통 기반

**⚠️ CRITICAL**: 이 단계가 끝나야 사용자 시나리오 작업을 시작할 수 있다

- [x] T007 프로필별 접속 설정과 `Asia/Seoul` 시간대 설정 in `res/application.yml`, `res/application-dev.yml`, `res/application-prod.yml`(접속 정보는 환경변수) (헌법 V)
- [x] T008 원천 문서 `기본값` 표의 모든 숫자를 `blog.limits.*`로 모으고 `@ConfigurationProperties`로 읽기 in `res/application.yml`, `api/common/config/BlogLimits.java` (research R-09, FR-056)
- [x] T009 [P] 기본값을 화면에 내려 주는 `GET /api/config/limits` in `api/common/config/LimitsController.java`
- [x] T010 [P] 공통 오류 응답(`code`, `message`, `fieldErrors`)과 예외 → 상태 코드 변환 in `api/common/error/ErrorResponse.java`, `api/common/error/GlobalExceptionHandler.java` (contracts/api.md 공통 규칙)
- [x] T011 [P] 원천 문서 `안내 문구` 표의 서버 문구를 한곳에 모으기 in `res/messages.properties`
- [x] T012 Flyway 첫 마이그레이션: `member` 표 (`email VARCHAR(254) NOT NULL UNIQUE(lower(email))`, `nickname VARCHAR(10) NOT NULL UNIQUE(lower(nickname))`, `bio VARCHAR(100) NULL`, `password_hash VARCHAR(100) NOT NULL`, `failed_login_count SMALLINT NOT NULL DEFAULT 0`, `locked_until TIMESTAMPTZ NULL`, `created_at`, `updated_at`) in `res/db/migration/V1__member.sql`
- [x] T013 Flyway 마이그레이션: `blog`(`owner_id NOT NULL FK ON DELETE CASCADE`, `name VARCHAR(30) NOT NULL`, `description VARCHAR(200) NULL`, `comments_last_viewed_at TIMESTAMPTZ NULL`), `category`(`name VARCHAR(20) NOT NULL UNIQUE(blog_id, lower(name))`, `sort_order INT NOT NULL`, `color_index SMALLINT NOT NULL`, `is_default BOOLEAN NOT NULL DEFAULT false`) in `res/db/migration/V2__blog_category.sql`
- [x] T014 Spring Session JDBC 표 생성(공식 스키마) in `res/db/migration/V3__spring_session.sql`
- [x] T015 Spring Security 설정: 세션 쿠키 `HttpOnly`·`SameSite=Lax`·`Secure`(prod)·만료 7일, 마지막 사용 후 7일 세션 만료, 로그인 시 세션 ID 재발급, CSRF 토큰 쿠키 + `X-XSRF-TOKEN` 헤더, 로그인 필요 API는 `401` JSON in `api/common/security/SecurityConfig.java` (FR-013, FR-055, research R-01·R-02)
- [x] T016 [P] BCrypt 비밀번호 인코더(강도 10~12) 빈 in `api/common/security/PasswordConfig.java` (FR-054)
- [x] T017 [P] 로그인한 회원 정보를 꺼내는 도우미와 회원별 세션 삭제 도우미(`FindByIndexNameSessionRepository`) in `api/common/security/CurrentMember.java`, `api/common/security/SessionTerminator.java` (FR-017, FR-021)
- [x] T018 [P] Redis 연결과 키 이름 도우미 in `api/common/redis/RedisKeys.java`
- [x] T019 [P] `MailSender` 인터페이스와 `LogMailSender`(dev, 로그에 인증번호), `SmtpMailSender`(prod, 5초 시간 제한) in `api/common/mail/` (research R-06, 헌법 V)
- [x] T020 [P] 화면 공통: API 호출 도우미(CSRF 헤더, 오류 응답 해석), 기본값 불러오기, 안내 문구 모음 in `web/api/client.ts`, `web/api/limits.ts`, `web/messages.ts`
- [x] T021 [P] 화면 공통 틀: 라우터, 머리글(로그인 전·후), 360px 대응 기본 스타일 in `web/App.tsx`, `web/components/Header.tsx`, `web/styles/base.css` (FR-052)
- [x] T022 [P] 통합 테스트 기반: PostgreSQL·Redis Testcontainers와 로그인된 MockMvc 도우미 in `test/support/IntegrationTestBase.java`

**Checkpoint**: 빈 서버가 세션 쿠키·CSRF와 함께 뜨고, 화면이 머리글을 보여 준다

---

## Phase 3: User Story 1 - 이메일 인증으로 가입하고 로그인하기 (Priority: P1) 🎯 MVP

**Goal**: 이메일 인증을 마친 사람만 가입하고, 가입하면 블로그가 생기며, 로그인·로그아웃·잠금이 동작한다

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 A

### Tests for User Story 1 (헌법 지정 항목)

- [x] T023 [P] [US1] 인증번호 규칙 통합 테스트: 10분 만료, 맞으면 폐기, 5회 틀리면 폐기, 1분·하루 5번 제한, 인증 없이·30분 지나 가입 시 `403` in `test/user/SignupVerificationIT.java` (FR-006~008)
- [x] T024 [P] [US1] 로그인 잠금·비노출 통합 테스트: 없는 이메일과 틀린 비밀번호의 응답이 같음, 5회 실패 후 `423`, 성공 시 횟수 0, 없는 이메일은 잠금 기록 없음 in `test/user/LoginLockIT.java` (FR-011, FR-012)

### Implementation for User Story 1

- [x] T025 [P] [US1] `Member` 엔터티와 저장소(이메일·닉네임 소문자 비교 조회) in `api/user/domain/Member.java`, `api/user/domain/MemberRepository.java`
- [x] T026 [P] [US1] `Blog`, `Category` 엔터티와 저장소 in `api/blog/domain/Blog.java`, `api/blog/domain/Category.java`, `api/blog/domain/*Repository.java`
- [x] T027 [US1] 입력 규칙 검사기: 이메일 형식·공백 제거·소문자, 닉네임 `2~10자 한글·영문·숫자`, 비밀번호 정규식(research R-03) in `api/user/validation/` (FR-002~004)
- [x] T028 [US1] 인증번호 서비스: `SecureRandom` 6자리(O·0·I·1 제외), Redis `verify:{purpose}:{email}:code|fail|cooldown|daily|verified` 키 처리, 용도 `signup`/`reset` in `api/user/verification/VerificationService.java` (research R-04·R-05, FR-006~008)
- [x] T029 [US1] 블로그 생성 서비스: "{닉네임}의 블로그" + "미분류"(`is_default=true`, `sort_order=1`) 함께 만들기, 회원당 1개 제한 in `api/blog/service/BlogCreationService.java` (FR-009, FR-022)
- [x] T030 [US1] 가입 서비스: 발송 전 이메일 중복 확인, 가입 시 인증됨 표시 재확인, 이메일 중복 재확인, BCrypt 저장, 블로그 생성, 인증됨 표시 삭제를 한 트랜잭션으로 in `api/user/service/SignupService.java` (FR-001~009)
- [x] T031 [US1] 가입 API: `POST /api/auth/signup/verification`, `/confirm`, `POST /api/auth/signup`, `GET /api/members/nickname-availability` in `api/user/web/SignupController.java`
- [x] T032 [US1] 로그인 서비스: 잠금 확인 → 비밀번호 비교 → 실패 횟수 증가(5회면 `locked_until = now + 10분`) / 성공 시 초기화, 없는 이메일은 기록 없이 같은 실패 응답 in `api/user/service/LoginService.java` (FR-010~012, research R-07)
- [x] T033 [US1] 로그인·로그아웃·내 상태 API: `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` in `api/user/web/AuthController.java` (FR-010, FR-014)
- [x] T034 [P] [US1] 회원가입 화면: 인증번호 받기·확인 단계, 이메일 잠금과 `이메일 변경`, 다시 받기 대기 시간, 비밀번호 규칙 4개 실시간 표시, 오류 칸으로 커서 이동·비밀번호 칸만 비우기, 버튼 중복 클릭 방지 in `web/pages/SignupPage.tsx` (FR-001~005)
- [x] T035 [P] [US1] 로그인 화면·로그인 모달, 로그인 후 원래 화면으로 돌아가기, 회원 전용 기능 클릭 시 모달 띄우는 도우미 in `web/pages/LoginPage.tsx`, `web/components/LoginModal.tsx`, `web/auth/requireLogin.ts` (FR-010, FR-015)
- [x] T036 [US1] 머리글 로그인 상태 연동(사용자 메뉴, 로그아웃 후 회원 화면이면 첫 화면으로) in `web/components/Header.tsx`, `web/auth/AuthContext.tsx` (FR-014)

**Checkpoint**: 시나리오 A 통과. 가입·로그인만으로 시연 가능

---

## Phase 4: User Story 2 - 내 블로그에 글 쓰고 고치기 (Priority: P1)

**Goal**: 블로그 주인이 분류를 관리하고 글을 쓰고·고치고·지우며, 비공개 글은 본인만 본다

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 B

### Tests for User Story 2 (헌법 지정 항목)

- [x] T037 [P] [US2] 글 권한·비공개 통합 테스트: 남의 글 수정·삭제·수정용 조회가 `404`, 남의 비공개 글 상세 `404`, 비공개 글이 남의 목록에 없음, CSRF 헤더 없으면 `403` in `test/post/PostAuthorizationIT.java` (FR-029, FR-032, FR-055)
- [x] T038 [P] [US2] 분류 권한 통합 테스트: 남의 분류 추가·수정·삭제 `404`, 글 있는 분류·미분류 삭제 `409` in `test/blog/CategoryAuthorizationIT.java` (FR-024, FR-025)

### Implementation for User Story 2

- [x] T039 [US2] Flyway 마이그레이션: `post`(`category_id NOT NULL FK ON DELETE RESTRICT`, `title VARCHAR(100) NOT NULL`, `body TEXT NOT NULL`, `visibility VARCHAR(10) NOT NULL DEFAULT 'PUBLIC' CHECK IN ('PUBLIC','PRIVATE')`, `view_count BIGINT NOT NULL DEFAULT 0`, `updated_at TIMESTAMPTZ NULL`)와 색인 `(blog_id, visibility, created_at DESC, id DESC)` in `res/db/migration/V4__post.sql`
- [x] T040 [P] [US2] `Post` 엔터티와 저장소 in `api/post/domain/Post.java`, `api/post/domain/PostRepository.java`
- [x] T041 [US2] 분류 서비스: 추가(맨 아래, 색 자동 배정), 이름 변경(블로그 안 대소문자 무시 중복 금지), 위·아래 이동(이웃과 `sort_order` 교환), 삭제 조건 검사, 주인 확인 in `api/blog/service/CategoryService.java` (FR-024~026)
- [x] T042 [US2] 블로그·분류 API: `GET/PATCH /api/blogs/{id}`, `POST /api/blogs/{id}/categories`, `PATCH/DELETE /api/categories/{id}`, `POST /api/categories/{id}/move` in `api/blog/web/BlogController.java`, `api/blog/web/CategoryController.java`
- [x] T043 [US2] 글 서비스: 작성(자기 블로그만, 제목 `1~100자` 공백 제거, 본문 `1~10,000자`, 분류 기본값 = 마지막 쓴 분류), 수정(내용 바뀐 경우만 `updated_at`), 삭제(한 트랜잭션), 볼 수 있는지 판단(남이면 `404`) in `api/post/service/PostService.java` (FR-027~030, FR-032)
- [x] T044 [US2] 글 API: `POST /api/blogs/{id}/posts`, `GET/PUT/DELETE /api/posts/{id}`, `GET /api/posts/{id}/edit` in `api/post/web/PostController.java`
- [x] T045 [P] [US2] 마크다운 표시 컴포넌트: 마크다운 → HTML → 허용 태그만 남기는 정화, `javascript:` 링크 제거 in `web/components/MarkdownView.tsx` (research R-10, FR-053)
- [x] T046 [P] [US2] 글쓰기·수정 화면: 제목, 분류, 공개 여부(비공개→공개 확인), 마크다운 본문, 저장 중 잠금, 실패 시 입력 유지, 이탈 확인 in `web/pages/PostEditorPage.tsx` (FR-027, FR-033)
- [x] T047 [P] [US2] 글 상세 화면(기본): 분류·제목·블로그 이름·작성/수정 시각·본문, 작성자만 수정·삭제, 삭제 확인 후 내 블로그 목록으로, "존재하지 않는 글입니다" in `web/pages/PostDetailPage.tsx` (FR-031, FR-032)

**Checkpoint**: 시나리오 B 통과. US1 + US2로 "가입해서 글 쓰기" 시연 가능

---

## Phase 5: User Story 3 - 방문자가 글을 찾아 읽기 (Priority: P1)

**Goal**: 로그인 없이 블로그 화면, 분류별 목록, 이전/다음 글, 검색을 쓴다

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 C

### Tests for User Story 3 (헌법 지정 항목)

- [x] T048 [P] [US3] 비공개 노출 통합 테스트: 방문자의 블로그 목록·분류 글 수·검색·이전/다음 글에 비공개 글이 0건, 주인 목록에는 포함 in `test/post/PrivatePostExposureIT.java` (FR-026, FR-032, SC-004)

### Implementation for User Story 3

- [x] T049 [US3] 목록 조회: 보는 사람 기준(방문자=공개만, 주인=전체), 분류 필터, `(created_at DESC, id DESC)` 정렬, 10개씩, 범위 밖 페이지는 마지막 페이지, 미리보기 `100자`(마크다운 기호 제거, 줄바꿈→공백) in `api/post/service/PostQueryService.java` (FR-034, FR-035)
- [x] T050 [US3] 이전/다음 글 조회(같은 블로그 공개 글, `(created_at, id)` 기준) in `api/post/service/PostQueryService.java` (FR-031)
- [x] T051 [P] [US3] 검색 서비스: 검색어 `2~50자`, 공백으로 단어 나누기, 단어마다 `ILIKE` AND, `%`·`_`·`\` 이스케이프, 공개 글만 in `api/search/SearchService.java` (FR-036, research R-11)
- [x] T052 [US3] 목록·검색 API: `GET /api/blogs/{id}/posts`, `GET /api/search` in `api/post/web/PostListController.java`, `api/search/SearchController.java`
- [x] T053 [P] [US3] 공통 페이지 번호 컴포넌트(현재 강조, 처음·끝에서 이전·다음 비활성) in `web/components/Pagination.tsx` (CF-10-3)
- [x] T054 [P] [US3] 블로그 화면: 이름·소개, 분류 목록(글 수), "N개의 글", 분류 선택 유지, 비공개 표시(주인), 빈 목록 안내 in `web/pages/BlogPage.tsx` (FR-022, FR-026, FR-033~035)
- [x] T055 [P] [US3] 첫 화면(전체 최근 공개 글)과 검색 결과 화면(검색어 유지, "검색 결과 N건", 2자 미만 안내) in `web/pages/HomePage.tsx`, `web/pages/SearchPage.tsx` (FR-036, FR-037)
- [x] T056 [US3] 글 상세에 이전/다음 글과 `목록으로`(그 분류 목록) 추가 in `web/pages/PostDetailPage.tsx` (FR-031)

**Checkpoint**: 시나리오 C 통과. P1 MVP 완성 — 가입, 쓰기, 읽기, 검색

---

## Phase 6: User Story 4 - 댓글과 좋아요로 소통하기 (Priority: P2)

**Goal**: 회원이 댓글을 달고 좋아요를 누르며, 작성자와 블로그 주인이 댓글을 지운다

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 D

### Tests for User Story 4 (헌법 지정 항목)

- [ ] T057 [P] [US4] 댓글·좋아요 권한 통합 테스트: 작성자도 블로그 주인도 아닌 회원의 댓글 삭제 `404`, 비회원 등록 `401`, 자기 글 좋아요 `400`, 5초 안 재등록 `429` in `test/comment/CommentAuthorizationIT.java` (FR-038~040)

### Implementation for User Story 4

- [ ] T058 [US4] Flyway 마이그레이션: `comment`(`author_id NULL FK ON DELETE SET NULL`, `content VARCHAR(500) NOT NULL`), `post_like`(`UNIQUE(post_id, member_id)`) in `res/db/migration/V5__comment_like.sql`
- [ ] T059 [P] [US4] `Comment`, `PostLike` 엔터티와 저장소 in `api/comment/domain/`, `api/post/domain/PostLike.java`
- [ ] T060 [US4] 댓글 서비스: 내용 `1~500자`(공백만 불가), Redis `comment-cooldown:{memberId}` 5초, 삭제 권한(작성자 또는 글의 블로그 주인), 글을 볼 수 있을 때만 조회 in `api/comment/service/CommentService.java` (FR-038, FR-039)
- [ ] T061 [US4] 좋아요 서비스: 켜고 끄기, 자기 글 금지, 개수와 내 상태 in `api/post/service/LikeService.java` (FR-040)
- [ ] T062 [US4] 댓글·좋아요 API: `GET/POST /api/posts/{id}/comments`, `DELETE /api/comments/{id}`, `PUT /api/posts/{id}/like`, 글 상세 응답에 `likeCount`·`likedByMe`·`commentCount` 추가 in `api/comment/web/CommentController.java`, `api/post/web/LikeController.java`
- [ ] T063 [US4] 글 상세에 댓글 목록(오래된 순, "탈퇴한 사용자"), 입력칸(비회원 안내), 삭제 확인, 좋아요 버튼 추가 in `web/pages/PostDetailPage.tsx`, `web/components/CommentSection.tsx`, `web/components/LikeButton.tsx`

**Checkpoint**: 시나리오 D 통과

---

## Phase 7: User Story 5 - 내 계정 관리와 비밀번호 찾기 (Priority: P2)

**Goal**: 마이페이지(정보 수정, 비밀번호 변경, 탈퇴)와 비밀번호 찾기

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 E

### Tests for User Story 5 (헌법 지정 항목)

- [ ] T064 [P] [US5] 비밀번호 찾기 비노출 통합 테스트: 가입·미가입·탈퇴 이메일의 응답·제한 동작이 같고 메일은 가입된 이메일에만 발송, 변경 후 모든 세션 삭제와 잠금 해제 in `test/user/PasswordResetIT.java` (FR-020, FR-021, SC-005)
- [ ] T065 [P] [US5] 비밀번호 변경·탈퇴 통합 테스트: 다른 세션만 끊김, 현재 비밀번호 오입력이 로그인 실패에 합산, 탈퇴 후 남의 글 댓글 작성자 NULL·재가입 가능 in `test/user/AccountManagementIT.java` (FR-017~019)

### Implementation for User Story 5

- [ ] T066 [US5] 마이페이지 서비스: 정보 조회, 닉네임(자기 닉네임은 중복 아님)·소개 `0~100자` 수정 in `api/user/service/MyPageService.java` (FR-016)
- [ ] T067 [US5] 비밀번호 변경 서비스: 현재 비밀번호 확인(틀리면 실패 횟수 +1), 새 비밀번호 규칙·현재와 다름, 지금 세션 빼고 삭제 in `api/user/service/PasswordChangeService.java` (FR-017, FR-018)
- [ ] T068 [US5] 탈퇴 서비스: 비밀번호 확인, research R-14 순서대로 한 트랜잭션 삭제·NULL 처리, 이미지 파일은 트랜잭션 후 삭제, 모든 세션 삭제 in `api/user/service/WithdrawalService.java` (FR-019)
- [ ] T069 [US5] 비밀번호 찾기 서비스: `reset` 용도 인증번호(가입 안 된 이메일은 번호 저장 없이 제한만 증가), 30분 안 재확인, 한 트랜잭션으로 비밀번호·실패 횟수·잠금 초기화, 모든 세션 삭제, 인증됨 표시 삭제 in `api/user/service/PasswordResetService.java` (FR-020, FR-021)
- [ ] T070 [US5] 계정 API: `GET/PATCH/DELETE /api/me`, `PUT /api/me/password`, `POST /api/auth/password-reset/verification`, `/confirm`, `POST /api/auth/password-reset` in `api/user/web/MyPageController.java`, `api/user/web/PasswordResetController.java`
- [ ] T071 [P] [US5] 마이페이지 화면: 내 정보(바뀐 것 없으면 저장 비활성, 이탈 확인), 비밀번호 변경, 탈퇴(안내·체크·최종 확인), 내 블로그 바로가기 in `web/pages/MyPage.tsx` (FR-016~019)
- [ ] T072 [P] [US5] 비밀번호 찾기 화면(로그인 상태면 첫 화면으로, 항상 같은 안내, 완료 후 로그인 화면) in `web/pages/PasswordResetPage.tsx` (FR-020, FR-021)

**Checkpoint**: 시나리오 E 통과

---

## Phase 8: User Story 6 - 블로그 관리 화면에서 운영하기 (Priority: P3)

**Goal**: 관리 화면의 글 관리, 분류 관리, 댓글 관리(새 댓글 표시), 설정

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 F의 F1~F3

### Tests for User Story 6 (헌법 지정 항목)

- [ ] T073 [P] [US6] 관리 API 권한 통합 테스트: `/api/manage/blogs/{id}/**`를 남이 부르면 `404`, 비회원 `401` in `test/stats/ManageAuthorizationIT.java` (FR-044)

### Implementation for User Story 6

- [ ] T074 [US6] 관리 공통 권한 검사(블로그 주인인지, 아니면 `404`) in `api/blog/service/BlogOwnerGuard.java` (FR-044, NF-02)
- [ ] T075 [US6] 글 관리 조회: 비공개 포함 내 글, 공개 여부·분류 거르기, 조회수·댓글 수 포함, 10개씩 in `api/post/service/ManagePostQueryService.java` (FR-046)
- [ ] T076 [US6] 새 댓글 서비스: `comments_last_viewed_at`(없으면 블로그 생성 시각) 이후 + 작성자 ≠ 주인인 댓글 수, 댓글 관리 조회 시 `isNew` 계산 후 시각 갱신 in `api/comment/service/NewCommentService.java` (FR-049, research R-13)
- [ ] T077 [US6] 관리 API: `GET /api/manage/blogs/{id}/posts`, `GET /api/manage/blogs/{id}/comments`(내용 앞 `50자`), `GET /api/auth/me`에 `newCommentCount` 추가 in `api/blog/web/ManageController.java`, `api/user/web/AuthController.java` (FR-046, FR-048, FR-049)
- [ ] T078 [P] [US6] 관리 화면 틀: 왼쪽 메뉴(좁은 화면은 위쪽 가로 목록), 새 댓글 숫자, `내 블로그 보기`, `글쓰기` in `web/pages/manage/ManageLayout.tsx` (FR-044)
- [ ] T079 [P] [US6] 글 관리 화면 in `web/pages/manage/ManagePostsPage.tsx` (FR-046)
- [ ] T080 [P] [US6] 분류 관리 화면: 색 점·이름·글 수, 이름 변경, 위·아래, 삭제 안내, 하단 고정 안내 in `web/pages/manage/ManageCategoriesPage.tsx` (FR-047)
- [ ] T081 [P] [US6] 댓글 관리 화면: 최신순, `NEW`, 글 제목 → 댓글 위치 이동, 삭제 in `web/pages/manage/ManageCommentsPage.tsx` (FR-048)
- [ ] T082 [P] [US6] 설정 화면: 이름·소개 "n/200" in `web/pages/manage/ManageSettingsPage.tsx` (FR-022)
- [ ] T083 [US6] 머리글 사용자 메뉴에 새 댓글 숫자와 `블로그 관리`·`마이페이지` 항목 추가 in `web/components/Header.tsx` (FR-049)

**Checkpoint**: 시나리오 F1~F3 통과

---

## Phase 9: User Story 7 - 방문 통계 보기 (Priority: P3)

**Goal**: 조회수·방문자 집계와 대시보드·통계 화면

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 F의 F4~F6

### Implementation for User Story 7

- [ ] T084 [US7] Flyway 마이그레이션: `daily_stats`(`stat_date DATE NOT NULL`, `view_count INT NOT NULL DEFAULT 0`, `visitor_count INT NOT NULL DEFAULT 0`, `UNIQUE(blog_id, stat_date)`) in `res/db/migration/V6__daily_stats.sql`
- [ ] T085 [US7] 방문자 구분값: 로그인 회원 `m:{id}`, 비회원 쿠키 `vid`(UUID, 1년) `v:{uuid}` 발급 필터 in `api/stats/VisitorKeyFilter.java` (research R-12)
- [ ] T086 [US7] 집계 서비스: 블로그 주인 본인 제외, Redis `view:{postId}:{visitor}` 30분 / `visit:{blogId}:{visitor}:{yyyyMMdd}` 한국 시간 자정까지, `daily_stats` UPSERT와 `post.view_count + 1`, 글 상세 조회 때 호출 in `api/stats/ViewCountService.java`, `api/post/web/PostController.java` (FR-050)
- [ ] T087 [US7] 대시보드·통계 조회: 오늘·어제·누적(합계), 30일 일별(없는 날 0), 최근 7일 인기 공개 글 5, 최근 글 5, 기간 7/30일 일별 조회수·방문자·댓글 수 in `api/stats/StatsQueryService.java` (FR-045, FR-051)
- [ ] T088 [US7] 대시보드·통계 API: `GET /api/manage/blogs/{id}/dashboard`, `GET /api/manage/blogs/{id}/stats?days=` in `api/stats/StatsController.java`
- [ ] T089 [P] [US7] 대시보드 화면(숫자 카드, 30일 선 그래프, 인기 글, 최근 글, 빈 상태) in `web/pages/manage/DashboardPage.tsx` (FR-045)
- [ ] T090 [P] [US7] 통계 화면(7/30일 선택, 두 그래프, 마우스를 올리면 그날 숫자) in `web/pages/manage/StatsPage.tsx` (FR-051)

**Checkpoint**: 시나리오 F4~F6 통과

---

## Phase 10: User Story 8 - 태그·이미지·신고 (Priority: P3)

**Goal**: 글에 태그와 이미지를 붙이고, 회원이 남의 글을 신고한다

**Independent Test**: [quickstart.md](quickstart.md) 시나리오 G

### Implementation for User Story 8

- [ ] T091 [US8] Flyway 마이그레이션: `tag`(`name VARCHAR(15) NOT NULL UNIQUE(lower(name))`), `post_tag`(PK(post_id, tag_id)), `report`(`reason VARCHAR(20) NOT NULL CHECK IN ('SPAM','ABUSE','ADULT','ETC')`, `detail VARCHAR(200) NULL`, `UNIQUE(post_id, reporter_id)`), `post_image`(`post_id NULL`, `storage_key VARCHAR(100) NOT NULL UNIQUE`, `content_type VARCHAR(20) NOT NULL`, `size_bytes INT NOT NULL CHECK ≤ 5MB`) in `res/db/migration/V7__tag_report_image.sql`
- [ ] T092 [P] [US8] `Tag`, `PostTag`, `Report`, `PostImage` 엔터티와 저장소 in `api/post/domain/`
- [ ] T093 [US8] 태그 처리: 앞 `#` 제거, `1~15자`, 공백·쉼표 불가, 대소문자 무시, 글당 최대 5개, 같은 글 중복 제거, 글 저장·수정 때 연결 in `api/post/service/TagService.java`, `api/post/service/PostService.java` (FR-041)
- [ ] T094 [US8] 태그별 공개 글 목록 API `GET /api/tags/{name}/posts` in `api/post/web/TagController.java` (FR-041)
- [ ] T095 [US8] 신고 서비스·API: 자기 글 금지, 사유 4종, 기타 설명 `0~200자`, 중복 `409` in `api/post/service/ReportService.java`, `api/post/web/ReportController.java` (FR-042)
- [ ] T096 [P] [US8] `ImageStorage` 인터페이스와 `S3ImageStorage`(MinIO), `DiskImageStorage` in `api/post/image/` (research R-08, 헌법 V)
- [ ] T097 [US8] 이미지 업로드 서비스·API: 매직 넘버로 jpg·png·gif·webp 확인, `5MB` 이하, UUID 파일 이름, 글당 `10장`, 글 저장 시 본문에 있는 이미지만 연결, `GET /images/{key}`(공개 글 또는 본인만) in `api/post/image/ImageService.java`, `api/post/web/ImageController.java` (FR-043)
- [ ] T098 [US8] 글 삭제·탈퇴 때 이미지 파일 삭제, 24시간 지난 미연결 이미지 하루 한 번 정리(`@Scheduled`) in `api/post/image/ImageCleanupJob.java` (FR-030, research R-08)
- [ ] T099 [P] [US8] 글쓰기 화면에 태그 입력(최대 5개)과 이미지 올리기(조건 안내, 본문에 주소 삽입) 추가 in `web/pages/PostEditorPage.tsx`, `web/components/TagInput.tsx`, `web/components/ImageUploadButton.tsx`
- [ ] T100 [P] [US8] 글 상세에 태그 목록과 신고 버튼(자기 글이면 숨김, 사유 선택 모달), 태그별 목록 화면 in `web/pages/PostDetailPage.tsx`, `web/components/ReportModal.tsx`, `web/pages/TagPage.tsx`

**Checkpoint**: 시나리오 G 통과. 모든 사용자 시나리오 완료

---

## Phase 11: Polish & Cross-Cutting Concerns

**Purpose**: 여러 시나리오에 걸친 마무리

- [ ] T101 [P] 스크립트 실행 차단 점검: 본문·댓글에 `<script>`, `javascript:` 링크, `onerror` 속성을 넣어도 실행되지 않음을 확인하는 화면 테스트 in `frontend/src/components/MarkdownView.test.tsx` (FR-053, quickstart S1)
- [ ] T102 [P] 비밀번호 원문이 로그·오류에 남지 않는지 점검(요청 로그에서 `password` 칸 가리기) in `api/common/logging/` (FR-054, quickstart S5)
- [ ] T103 [P] 360px 화면 점검과 수정(관리 화면 메뉴, 표, 그래프) in `web/styles/` (FR-052, SC-007)
- [ ] T104 성능 점검: 글 1,000개 블로그에서 목록·상세 2초 이내 확인, 느리면 색인·쿼리 수정 in `res/db/migration/` (SC-003)
- [ ] T105 [P] 안내 문구가 원천 문서 `안내 문구` 표와 같은지 대조 in `res/messages.properties`, `web/messages.ts` (FR-057)
- [ ] T106 [P] 코드 저장소 README에 실행 방법(compose, dev 프로필, 로그에서 인증번호 보기) 정리 in `README.md`
- [ ] T107 [quickstart.md](quickstart.md) 시나리오 A~G와 보안·성능 점검을 처음부터 끝까지 다시 실행

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 바로 시작
- **Foundational (Phase 2)**: Setup 후. 모든 사용자 시나리오를 막는다
- **사용자 시나리오 (Phase 3~10)**: Foundational 후. 아래 시나리오 간 의존을 따른다
- **Polish (Phase 11)**: 원하는 시나리오가 모두 끝난 뒤

### User Story Dependencies

원천 문서상 글은 가입해야 생기는 블로그에 쓰므로, 이 서비스는 시나리오끼리 완전히 독립적이지 않다.

| 시나리오 | 먼저 끝나야 하는 것 | 이유 |
|---|---|---|
| US1 가입·로그인 | Foundational | |
| US2 글쓰기 | US1 | 로그인과 가입 시 만든 블로그가 필요 |
| US3 읽기·검색 | US2 | 읽을 글(`post` 표)이 필요 |
| US4 댓글·좋아요 | US2 (상세 화면은 US3과 같은 파일) | |
| US5 계정 | US1 (탈퇴 처리의 완성은 US4·US8 표가 생긴 뒤 보강) | |
| US6 관리 화면 | US2, US4 | 글·댓글 관리 대상 |
| US7 통계 | US6 (관리 화면 틀) | |
| US8 태그·이미지·신고 | US2 | |

권장 순서: US1 → US2 → US3 → (US4, US5, US8 순서 무관) → US6 → US7

### Within Each User Story

- 헌법 지정 테스트를 먼저 쓰고 실패하는 것을 확인한 뒤 구현
- 마이그레이션 → 엔터티 → 서비스 → API → 화면
- 시나리오가 끝나면 quickstart로 확인하고 다음으로

### Parallel Opportunities

- Setup의 T003~T006, Foundational의 [P] 작업은 동시에 할 수 있다
- 각 시나리오에서 테스트([P])와 엔터티([P])는 동시에, 화면([P])은 API가 정해지면 서버 작업과 동시에 할 수 있다
- US4, US5, US8은 US3 이후 서로 동시에 진행할 수 있다(같은 파일 `PostDetailPage.tsx`, `PostService.java`만 조심)

---

## Parallel Example: User Story 1

```bash
# 테스트 먼저 (동시에):
Task: "인증번호 규칙 통합 테스트 in test/user/SignupVerificationIT.java"
Task: "로그인 잠금·비노출 통합 테스트 in test/user/LoginLockIT.java"

# 엔터티 (동시에):
Task: "Member 엔터티와 저장소 in api/user/domain/"
Task: "Blog, Category 엔터티와 저장소 in api/blog/domain/"

# 서버 API가 정해진 뒤 화면 (서버 서비스 작업과 동시에):
Task: "회원가입 화면 in web/pages/SignupPage.tsx"
Task: "로그인 화면·모달 in web/pages/LoginPage.tsx"
```

---

## Implementation Strategy

### MVP First (P1: US1 → US2 → US3)

1. Phase 1 Setup, Phase 2 Foundational
2. Phase 3 US1 → 시나리오 A 확인 (가입·로그인만 시연 가능)
3. Phase 4 US2 → 시나리오 B 확인 (글쓰기)
4. Phase 5 US3 → 시나리오 C 확인 → **P1 MVP 완성: 가입, 쓰기, 읽기, 검색**
5. 시간이 모자라면 여기서 멈추고 시연한다

### Incremental Delivery

1. MVP 이후 US4(댓글·좋아요) → US5(계정) → US8(태그·이미지·신고) → US6(관리) → US7(통계)
2. 각 단계마다 quickstart 해당 시나리오로 확인한 뒤 다음으로
3. 2주·1인 일정에서는 P3(US6~US8)는 범위를 줄일 수 있다. 줄이면 spec의 우선순위 표와 이 문서를 같이 고친다

---

## 구현 기록

### 2026-10-08: MVP(US1~US3) 구현

코드는 이 Mac의 로컬 저장소 `~/Documents/blog-app`에 있다(원격 저장소 없음).

- T001~T056 중 T006(서식 도구)만 남았다. 서버 통합 테스트 28개, 화면 테스트 6개가 통과했고, 브라우저와 API로 quickstart 시나리오 A~C를 확인했다.
- 계획과 다르게 한 것:
  - T004: compose에 MinIO는 넣지 않았다. 이미지 업로드(US8)를 할 때 추가한다.
  - T011: 서버 안내 문구는 `messages.properties` 대신 `common/error/Messages.java` 상수로 모았다.
  - T022: Docker가 없어 Testcontainers 대신 로컬 PostgreSQL의 `myblog_test` DB와 Redis 15번 DB를 쓴다.
  - T041~T042: 분류 관리 화면은 관리 화면(US6) 전이라 블로그 화면 옆에 주인에게만 보이게 넣었다.
  - 글 삭제 API는 `204` 대신 `200 {blogId}`를 돌려준다. 화면이 지운 뒤 블로그 목록으로 돌아가기 위해서다.
  - 비밀번호 찾기(US5)에서 가입되지 않은 이메일도 화면 흐름이 같도록, 그 이메일에는 아무도 모르는 번호를 저장한다(research R-04 보강).

## Notes

- 작업 수: 총 107개 (Setup 6, Foundational 16, US1 14, US2 11, US3 9, US4 7, US5 9, US6 11, US7 7, US8 10, Polish 7)
- [P] = 다른 파일, 앞선 미완료 작업에 기대지 않음
- 숫자 제약은 data-model.md의 값을 그대로 옮겼다. 구현 때 바꾸려면 T008의 기본값 묶음과 원천 문서 `기본값` 표를 함께 고친다
- 작업 하나 또는 논리적 묶음마다 커밋한다
