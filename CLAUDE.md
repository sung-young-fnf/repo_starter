# 프로젝트 규칙

이 저장소의 아키텍처 규칙은 `.claude/docs/` 에 있다. **코드를 쓰기 전에 해당 문서를 읽는다.**

| 상황 | 읽을 문서 |
|---|---|
| 도메인 모델링·기능 착수 | `.claude/docs/10-DDD-CORE.md` (항상) |
| NestJS 백엔드 작업 | `.claude/docs/20-BACKEND-NESTJS.md` |
| FastAPI/Python 백엔드 작업 | `.claude/docs/21-BACKEND-PYTHON.md` |
| Next.js 프론트 작업 | `.claude/docs/30-FRONTEND-FSD.md` |
| 테스트 작성 | `.claude/docs/40-TDD.md` |
| **구현 계획 수립** | `.claude/docs/50-PLAN-CHECKLIST.md` |
| 전체 체계 지도 | `.claude/docs/00-OVERVIEW.md` |
| **작업 순서 (시작 ~ PR 머지)** | `.claude/docs/01-WORKFLOW.md` |

## 아키텍처 요약

- 백엔드: **DDD + Ports & Adapters**, 컨텍스트의 공개 진입점은 **Facade** 하나.
  컨트롤러는 Facade 만 호출하고, 트랜잭션은 Facade 메서드에서 연다.
- 프론트: **FSD**. `app → views → widgets → features → entities → shared` 한 방향 의존,
  슬라이스 밖에서는 `index.ts` 공개 API 로만 들어간다.
- 도메인 레이어는 프레임워크·ORM 을 모른다. 시간·랜덤·ID 는 주입받는다.
- 개발은 **TDD**. 실패하는 테스트 없이 프로덕션 코드를 쓰지 않는다.

## 워크플로우

1. 도메인이 새롭거나 모호하면 → `/domain-model`
2. 모든 기능 구현의 시작 → **`/ddd-plan`** (병렬 탐색 → 계획 + `plan-reviewer` 자동 리뷰 → 승인 → TDD 실행)
3. 뼈대가 필요하면 → `/ddd-scaffold`
4. 통합 테스트 → `/write-integration-test`
5. **커밋 전 필수 게이트** → 백엔드 변경은 `ddd-reviewer`, 프론트 변경은 `fsd-reviewer`.
   위반이 나오면 `ddd-refactor` 로 수정하고 재리뷰.
6. API 가 바뀌었으면 → `api-spec-updater`, `http-file-generator`

## 산출물 위치

```
docs/domain/          도메인 모델 (유비쿼터스 언어·애그리거트·이벤트)
docs/api-spec/        API 스펙 + changelog
docs/code-review/     리뷰 리포트
http/                 수동 테스트용 .http 파일
```

> 커밋/PR 메시지 규칙은 사용자 전역 `~/.claude/CLAUDE.md` 의 Conventional Commits 규칙을 따른다.
> `TASKS.md` 형식 규칙도 마찬가지다.
