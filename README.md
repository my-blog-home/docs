# Ylog 요구사항·설계 문서

**Ylog**는 회원마다 블로그를 하나씩 갖고 글을 쓰고, 읽고, 댓글로 소통하는 블로그 서비스입니다.
이 저장소에는 **코드를 쓰기 전에 먼저 정한 요구사항과 설계 문서**가 있습니다. 코드는 [my-blog-home/blog-app](https://github.com/my-blog-home/blog-app)에 있습니다.

> 기술 스택과 고른 이유, 설계 결정, 버전별 진행 기록, 트러블슈팅, 피드백 정리와 다음 과제는
> **[blog-app README](https://github.com/my-blog-home/blog-app#readme)** 에 모아 두었습니다. 이 문서는 "요구사항을 어떻게 정했는가"를 다룹니다.

## 요구사항을 정한 방법

GitHub **[Spec Kit](https://github.com/github/spec-kit)** 의 순서를 따랐습니다. 문서가 먼저 있고, 코드는 그 문서를 원천으로 만듭니다.

```
constitution (원칙) → spec (무엇을, 왜) → clarify (애매한 점 확정) → plan·research (어떻게)
→ data-model·contracts (표, API, 화면) → tasks (할 일 목록) → 구현(blog-app)
```

1. **원칙 정하기** — [`.specify/memory/constitution.md`](.specify/memory/constitution.md)
   1. 요구사항 문서가 원천이다
   2. 규칙과 권한은 서버가 지킨다 (양보 불가)
   3. 단순함을 고른다
   4. 숫자는 한곳에서 바꾼다
   5. 바꿔 끼울 수 있게 외부에 기댄다
   6. 누구나 읽을 수 있게 쓴다

   이 원칙이 이후 기술 선택(서버 세션, 설정값 한곳 모으기, 저장소 인터페이스 등)의 기준이 되었습니다.

2. **사용자 시나리오와 기능 요구사항** — [`specs/001-blog-platform/spec.md`](specs/001-blog-platform/spec.md)
   - 처음 범위: **US1~US8** (가입·로그인, 글쓰기, 찾아 읽기, 댓글·좋아요, 계정 관리, 블로그 관리, 통계, 태그·이미지·신고)과 FR-001~057
   - 시나리오마다 우선순위(P1~P3)와 "이렇게 되면 통과" 기준을 적어 MVP(P1) 먼저 만들 수 있게 했습니다.

3. **구현 계획과 기술 결정** — [`plan.md`](specs/001-blog-platform/plan.md), [`research.md`](specs/001-blog-platform/research.md)
   - 결정마다 무엇을 골랐는지, 왜 골랐는지, 다른 후보는 무엇이었는지를 R-01~R-17로 남겼습니다(인증 방식, 인증번호 저장, 이미지 저장, 검색, 통계 집계, DB 변경 관리 등).

4. **데이터·API·화면 설계** — [`data-model.md`](specs/001-blog-platform/data-model.md), [`contracts/api.md`](specs/001-blog-platform/contracts/api.md), [`contracts/screens.md`](specs/001-blog-platform/contracts/screens.md)

5. **할 일 목록** — [`tasks.md`](specs/001-blog-platform/tasks.md)
   - 시나리오별로 나눈 작업(T001~). 구현이 끝난 항목은 체크해 진행 상황을 문서에서 바로 볼 수 있게 했습니다.

## 처음 범위 이후에 더한 것

MVP를 만든 뒤 참고 시안(pblog)에 있던 기능을 요구사항에 더했습니다. 코드보다 문서를 먼저 고친다는 원칙을 지키기 위해서입니다.

- **US9~US13**: 주제와 임시저장, 답글·비밀 댓글·댓글 신고, 구독과 내 활동, 첫 화면 둘러보기(핫한 글·인기 검색어·인기 블로거), 관리자 운영
- **FR-058~087**, 할 일 **T108~T132**
- 정해 둔 규칙이 바뀐 경우(예: 비밀번호 8~10자 → 8자 이상)도 문서를 먼저 고친 뒤 코드에 반영했습니다.

그 뒤의 화면·운영 개선(목록형·피드형 보기, 다크 모드, 정책 페이지, 릴리스 노트 등)은 버전마다 blog-app의 릴리스 노트와 README의 진행 기록에 남겼습니다.

## 폴더

| 위치 | 내용 |
|---|---|
| `specs/001-blog-platform/` | Spec Kit 문서: spec, plan, research, data-model, contracts, tasks, checklists |
| `.specify/` | Spec Kit 설정과 프로젝트 원칙(constitution) |
| `docs/상세/` | Spec Kit 이전에 쓴 기능별 상세 요구사항(인증·계정·블로그·탐색·소통·관리·공통 규칙) |
| `docs/가이드/` | 기술 설명(서버, 도커, 로컬 환경, MinIO 현황, 배포 준비)과 아키텍처 그림 |
| `.claude/skills/` | Claude Code에서 쓰는 Spec Kit 명령(`/speckit-*`) |

## 문서를 고칠 때

1. 요구사항이 바뀌면 `spec.md`를 먼저 고치고, 영향을 받는 `plan`·`data-model`·`contracts`·`tasks`를 맞춥니다.
2. 이 저장소는 PR로만 `main`에 병합합니다.
3. 코드 변경은 blog-app 저장소에서 하고, 완료된 할 일은 `tasks.md`에 체크합니다.
