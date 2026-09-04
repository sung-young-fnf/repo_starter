# 개발 규칙 체계 개요

이 저장소에서 코드를 쓰기 전에 읽는 문서다. **규칙 문서(docs) → 에이전트(agents) → 스킬(skills)** 세
층으로 되어 있고, 사람이 매번 챙겨야 할 아키텍처 규율을 에이전트 파이프라인이 대신 검사한다.

## 스택 전제

| 영역 | 스택 | 아키텍처 |
|---|---|---|
| 프론트엔드 | Next.js (App Router) + TypeScript | **FSD** (Feature-Sliced Design) |
| 백엔드 (Node) | NestJS + TypeScript | **DDD + Ports & Adapters + Facade** |
| 백엔드 (Python) | FastAPI + SQLAlchemy | **DDD + Ports & Adapters + Facade** |
| 개발 방식 | 전 영역 | **TDD** (Red → Green → Refactor) |

두 백엔드는 언어만 다르고 **레이어 구조·의존 방향·Facade 규칙은 동일**하다. 문서도 그렇게 쪼개 두었다.

## 문서 지도

| 문서 | 언제 읽나 |
|---|---|
| [01-WORKFLOW.md](01-WORKFLOW.md) | **작업을 시작할 때** — 프로젝트 시작 / 기능 추가 두 시나리오의 순서, 커밋·PR 시점 |
| [10-DDD-CORE.md](10-DDD-CORE.md) | 도메인 모델링 시작 전 — 전략/전술 설계, 애그리거트 규칙, 레이어 의존 방향 (**언어 무관, 필수**) |
| [20-BACKEND-NESTJS.md](20-BACKEND-NESTJS.md) | NestJS 코드 작성 전 — 폴더 구조, Facade, DI 토큰, 트랜잭션 |
| [21-BACKEND-PYTHON.md](21-BACKEND-PYTHON.md) | FastAPI 코드 작성 전 — 동일 규칙의 Python 매핑 |
| [30-FRONTEND-FSD.md](30-FRONTEND-FSD.md) | Next.js 코드 작성 전 — FSD 레이어/슬라이스/공개 API, 백엔드 도메인과의 언어 일치 |
| [40-TDD.md](40-TDD.md) | 테스트 작성 전 — 테스트 계층, 무엇을 목으로 대체하고 무엇을 실물로 쓰나 |
| [50-PLAN-CHECKLIST.md](50-PLAN-CHECKLIST.md) | **구현 계획을 세울 때마다** — plan 단계에서 통과해야 하는 체크리스트 |

## 에이전트 (`.claude/agents/`)

| 에이전트 | 역할 | 호출 시점 |
|---|---|---|
| `domain-modeler` | 요구사항 → 바운디드 컨텍스트·애그리거트·도메인 이벤트 도출 | 새 도메인/기능 착수 시 |
| `plan-reviewer` | 구현 계획을 코드 작성 **전에** 검증 | 계획 수립 직후 (`/ddd-plan` Phase 2에서 자동) |
| `ddd-reviewer` | 백엔드 DDD·Facade 경계 위반 검사 | **커밋 전 필수** |
| `fsd-reviewer` | 프론트 FSD 레이어·슬라이스·공개 API 위반 검사 | **커밋 전 필수** |
| `ddd-refactor` | 발견된 위반을 실제로 코드 수정 | reviewer가 위반을 찾았을 때 |
| `api-spec-updater` | API 스펙 문서 + changelog 동기화 | controller/DTO/route 변경 후 |
| `http-file-generator` | `.http` 수동 테스트 파일 생성 | 새 엔드포인트 추가/DTO 변경 후 |
| `web-research-specialist` | 에러·기술 선택지 인터넷 리서치 | 라이브러리 에러, 기술 비교 |

## 스킬 (`.claude/skills/`)

| 스킬 | 역할 |
|---|---|
| `/domain-model` | 이벤트스토밍 방식으로 도메인 모델을 뽑아 `docs/domain/`에 문서화 |
| `/ddd-plan` | 병렬 탐색 → 계획 + 자동 리뷰 → TDD 실행. **모든 기능 구현의 시작점** |
| `/ddd-scaffold` | 바운디드 컨텍스트(백엔드) 또는 FSD 슬라이스(프론트) 뼈대 생성 |
| `/write-integration-test` | 실제 DB·실제 서버 대상 통합 테스트 작성 |

## 개발 흐름

```
새 기능 요청
│
├─ (도메인이 새롭거나 모호하면) /domain-model
│    └─ domain-modeler ─→ docs/domain/<context>/ (유비쿼터스 언어, 애그리거트, 이벤트)
│
├─ /ddd-plan                                    ← 모든 구현의 진입점
│    ├─ Phase 1: 탐색 에이전트 3개 병렬 (코드 / 테스트 / 패턴)
│    ├─ Phase 2: 계획 작성 → plan-reviewer 자동 리뷰 → 사용자 승인
│    └─ Phase 3: 테스트 먼저 → 구현 → DI 배선 → 전체 통과까지 반복
│
├─ 테스트 작성
│    └─ /write-integration-test
│
├─ 구현
│    └─ 뼈대가 필요하면 /ddd-scaffold
│
├─ 리뷰 (커밋 전 필수 게이트)
│    ├─ ddd-reviewer   (백엔드 변경 시)
│    ├─ fsd-reviewer   (프론트 변경 시)
│    └─ 위반 발견 → ddd-refactor 로 수정 → 재리뷰
│
├─ 문서 동기화
│    ├─ api-spec-updater
│    └─ http-file-generator
│
└─ 커밋 & PR  (형식은 01-WORKFLOW.md W6 · W8)
```

## 이 체계가 지키려는 것

1. **의존 방향은 자동으로 검사한다.** 사람 눈이 아니라 린터(`dependency-cruiser`, `import-linter`,
   `steiger`)와 리뷰 에이전트가 잡는다. 규칙이 프로즈로만 있으면 반드시 썩는다.
2. **도메인 레이어는 프레임워크를 모른다.** NestJS 데코레이터, SQLAlchemy 모델, Pydantic, React —
   전부 도메인 바깥.
3. **경계를 넘는 통로는 하나다.** 백엔드는 Facade, 프론트는 슬라이스의 `index.ts`.
4. **테스트가 먼저다.** 실패하는 테스트 없이 프로덕션 코드를 쓰지 않는다.
