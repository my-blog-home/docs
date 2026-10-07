# API Contract: 개인 블로그 서비스 (MVP)

> 화면(React)과 서버가 주고받는 REST API 목록이다. 가안이며, 구현하면서 이름은 바뀔 수 있지만 **권한과 오류 규칙은 지킨다.**
> 형식: JSON, 시각은 ISO-8601(UTC, 화면에서 한국 시간으로 표시).

## 공통 규칙

| 항목 | 규칙 | 근거 |
|---|---|---|
| 로그인 | 세션 쿠키. 로그인이 필요한 API에 쿠키가 없으면 `401` | FR-015 |
| CSRF | `GET` 외 모든 요청은 `X-XSRF-TOKEN` 헤더 필요. 없거나 틀리면 `403` | FR-055 |
| 권한 없음 | 남의 글·블로그를 고치려 하면 **`404`** ("존재하지 않는 글입니다"). 존재 여부를 숨기려고 `403` 대신 `404`를 쓴다 | FR-029, FR-032 |
| 비공개 글 | 작성자가 아니면 어느 API에서도 결과에 나오지 않고, 상세는 `404` | FR-032 |
| 입력 오류 | `400` + 칸별 오류 목록 | FR-005 |
| 제한 초과 | `429` + 안내 문구 (인증번호 다시 받기, 댓글 5초) | FR-007, FR-038 |
| 페이지 | `?page=1` 부터, 한 페이지 10개. 범위를 넘으면 마지막 페이지를 돌려준다 | FR-034, FR-035 |

오류 응답 모양:

```json
{
  "code": "VALIDATION_FAILED",
  "message": "입력값을 확인해 주세요",
  "fieldErrors": [{ "field": "nickname", "message": "닉네임은 한글, 영문, 숫자로 2~10자여야 합니다" }]
}
```

`message`는 원천 문서의 `안내 문구` 표를 그대로 쓴다. 계정이 있는지, 내부 구조가 어떤지 드러내는 내용은 넣지 않는다(FR-011, FR-020).

## 설정

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET | `/api/config/limits` | 누구나 | 글자 수·개수·용량 등 기본값 (research R-09) |

## 가입·이메일 인증 (US1)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| POST | `/api/auth/signup/verification` | 비회원 | `{nickname, email}` → `204`. 이메일 중복 `409`, 1분 제한·하루 한도 `429`, 발송 실패 `503` | FR-001~003, 006, 007 |
| POST | `/api/auth/signup/verification/confirm` | 비회원 | `{email, code}` → `204`. 틀림 `400`, 만료·5회 초과 `410` | FR-006~008 |
| POST | `/api/auth/signup` | 비회원 | `{nickname, email, password, passwordConfirm}` → `201`. 인증 없음·30분 지남 `403`, 그 사이 가입된 이메일 `409` | FR-001~005, 008, 009 |
| GET | `/api/members/nickname-availability?nickname=` | 누구나 | `{available}` (입력 중 안내용) | FR-003 |

## 로그인·세션 (US1)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| POST | `/api/auth/login` | 비회원 | `{email, password}` → `200 {member}`. 실패는 항상 `401` 같은 문구, 잠금 중 `423 {retryAfterMinutes}` | FR-010~013 |
| POST | `/api/auth/logout` | 회원 | `204` | FR-014 |
| GET | `/api/auth/me` | 누구나 | 로그인 `200 {id, nickname, blogId, newCommentCount}`, 아니면 `401` | FR-015, FR-049 |

> 잠금(`423`)은 **가입된 계정이 실제로 잠겼을 때만** 나온다. 이는 원천 문서(CF-02-6 "남은 시간을 안내")가 요구한 동작이며, 5회 연속 실패 뒤에만 보이므로 계정 존재를 알려 주는 정도는 받아들인다.

## 비밀번호 찾기 (US5)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| POST | `/api/auth/password-reset/verification` | 비회원 | `{email}` → 가입 여부와 무관하게 `204` (제한 초과만 `429`) | FR-020 |
| POST | `/api/auth/password-reset/verification/confirm` | 비회원 | `{email, code}` → `204` / `400` / `410` | FR-020 |
| POST | `/api/auth/password-reset` | 비회원 | `{email, newPassword, newPasswordConfirm}` → `204`. 모든 세션 삭제, 잠금 해제 | FR-020, 021 |

## 마이페이지 (US5)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| GET | `/api/me` | 회원 | `{email, nickname, bio, joinedAt, blogId}` | FR-016 |
| PATCH | `/api/me` | 회원 | `{nickname?, bio?}` → `200`. 닉네임 중복 `409` | FR-016 |
| PUT | `/api/me/password` | 회원 | `{currentPassword, newPassword, newPasswordConfirm}` → `204`. 현재 비밀번호 틀림 `400`(실패 횟수 +1), 다른 세션 삭제 | FR-017, 018 |
| DELETE | `/api/me` | 회원 | `{password, agreed: true}` → `204`. 모든 세션 삭제 | FR-019 |

## 블로그·분류 (US2, US6)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| GET | `/api/blogs/{blogId}` | 누구나 | `{id, name, description, ownerNickname, categories:[{id, name, colorIndex, postCount}]}` (글 수는 보는 사람 기준) | FR-022, 026 |
| PATCH | `/api/blogs/{blogId}` | 주인 | `{name, description}` | FR-022 |
| POST | `/api/blogs/{blogId}/categories` | 주인 | `{name}` → `201`. 중복 `409` | FR-024 |
| PATCH | `/api/categories/{id}` | 주인 | `{name}` | FR-024 |
| POST | `/api/categories/{id}/move` | 주인 | `{direction: "UP"\|"DOWN"}` | FR-024 |
| DELETE | `/api/categories/{id}` | 주인 | `204`. 글 있음·미분류 `409 {postCount}` | FR-025 |

## 글 (US2, US3)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| GET | `/api/blogs/{blogId}/posts?categoryId=&page=` | 누구나 | `{totalCount, page, totalPages, items:[{id, title, excerpt, categoryName, createdAt, visibility}]}` (주인에게만 비공개 포함) | FR-034, 035 |
| GET | `/api/posts/{postId}` | 누구나 | `{id, blog, category, title, body, visibility, createdAt, updatedAt, tags, likeCount, likedByMe, commentCount, prevPostId, nextPostId, editable}`. 볼 수 없으면 `404`. 열람 시 조회수 집계 | FR-031, 032, 050 |
| POST | `/api/blogs/{blogId}/posts` | 주인 | `{title, body, categoryId, visibility, tags[], imageIds[]}` → `201 {id}` | FR-027, 041, 043 |
| PUT | `/api/posts/{postId}` | 작성자 | 위와 같음 → `200`. 내용 변화 없으면 `updatedAt` 유지 | FR-028, 029 |
| DELETE | `/api/posts/{postId}` | 작성자 | `204`. 딸린 데이터 함께 삭제 | FR-030 |
| GET | `/api/posts/{postId}/edit` | 작성자 | 수정 화면용 원본. 남의 글 `404` | FR-029 |

## 검색·태그 (US3, US8)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| GET | `/api/search?q=&page=` | 누구나 | `{totalCount, items:[{id, blogName, categoryName, createdAt, title, excerpt}]}`. 2자 미만 `400` | FR-036 |
| GET | `/api/tags/{name}/posts?page=` | 누구나 | 그 태그가 붙은 공개 글 목록 | FR-041 |

## 댓글·좋아요·신고 (US4, US8)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| GET | `/api/posts/{postId}/comments` | 누구나(글을 볼 수 있을 때) | 오래된 순 `[{id, authorNickname\|null, content, createdAt, deletable}]` | FR-038 |
| POST | `/api/posts/{postId}/comments` | 회원 | `{content}` → `201`. 5초 안 재등록 `429` | FR-038 |
| DELETE | `/api/comments/{id}` | 댓글 작성자 또는 글의 블로그 주인 | `204`. 그 외 `404` | FR-039 |
| PUT | `/api/posts/{postId}/like` | 회원(자기 글 제외) | `{liked, likeCount}`. 누를 때마다 켜고 끔. 자기 글 `400` | FR-040 |
| POST | `/api/posts/{postId}/reports` | 회원(자기 글 제외) | `{reason, detail?}` → `201`. 이미 신고 `409` | FR-042 |

## 이미지 (US8)

| 메서드 | 경로 | 권한 | 요청 → 응답 | FR |
|---|---|---|---|---|
| POST | `/api/images` | 회원 | `multipart file` → `201 {id, url}`. 형식·용량 위반 `400` | FR-043 |
| GET | `/images/{storageKey}` | 누구나(공개 글 또는 본인) | 이미지 파일 | FR-043 |

## 블로그 관리·통계 (US6, US7) — 모두 블로그 주인만, 남이면 `404`

| 메서드 | 경로 | 요청 → 응답 | FR |
|---|---|---|---|
| GET | `/api/manage/blogs/{blogId}/dashboard` | `{views:{today, yesterday, total}, visitors:{…}, newCommentCount, daily30:[{date, views, visitors}], popular7:[{postId, title, views}], recent:[{postId, title, createdAt, visibility}]}` | FR-045 |
| GET | `/api/manage/blogs/{blogId}/posts?visibility=&categoryId=&page=` | `{items:[{id, title, categoryName, createdAt, visibility, viewCount, commentCount}]}` | FR-046 |
| GET | `/api/manage/blogs/{blogId}/comments?page=` | `{items:[{id, authorNickname\|null, createdAt, excerpt50, postId, postTitle, isNew}]}`. 호출하면 읽음 처리 | FR-048, 049 |
| GET | `/api/manage/blogs/{blogId}/stats?days=7\|30` | `{daily:[{date, views, visitors, comments}]}` | FR-051 |
