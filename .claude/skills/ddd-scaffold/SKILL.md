---
name: ddd-scaffold
description: 새 바운디드 컨텍스트(NestJS/FastAPI) 또는 새 FSD 슬라이스(Next.js)의 뼈대를 규칙에 맞게 생성한다. 레이어 폴더, Facade, 포트, DI 배선, 공개 API index.ts, 린터 계약, 테스트 스캐폴드까지 함께 만든다.
---

# /ddd-scaffold — 뼈대 생성

빈 구조를 손으로 만들다 보면 반드시 뭔가 빠진다. **배선과 린터 계약까지** 함께 만든다.

## 시작 전 확인

사용자에게 다음을 확인한다 (`AskUserQuestion` 사용 가능).

1. **대상**: 백엔드 컨텍스트 / 프론트 슬라이스 / 둘 다
2. **이름**: 컨텍스트명 또는 슬라이스명
   - `docs/domain/*/ubiquitous-language.md` 가 있으면 **거기 있는 식별자를 쓴다.**
     없는 이름이면 "용어표에 없는 이름입니다. 추가할까요?" 라고 먼저 묻는다.
3. **프론트 슬라이스면 레이어**: `entities` / `features` / `widgets` / `views`
   - 판단 근거를 한 줄로 확인한다. 도메인 모델이면 entities, 사용자 의도면 features.

기존 코드가 있으면 **가장 최근에 만들어진 컨텍스트/슬라이스를 읽고 그 형식을 그대로 따른다.**
이 문서의 예시보다 저장소의 실제 관례가 우선이다.

---

## A. 백엔드 컨텍스트 (NestJS)

`.claude/docs/20-BACKEND-NESTJS.md` 의 구조를 따른다.

```
src/contexts/<context>/
├── domain/
│   ├── model/<context>.ts              # 애그리거트 루트 (private 생성자 + static create)
│   ├── model/<context>-id.vo.ts
│   ├── model/<context>-status.vo.ts    # 상태가 있으면 전이표 포함
│   ├── model/events.ts
│   ├── repository/<context>.repository.ts   # interface + Symbol 토큰
│   ├── service/                        # 필요할 때만
│   └── error/<context>.error.ts
├── application/
│   ├── <context>.facade.ts             # ★
│   ├── usecase/create-<context>.usecase.ts
│   ├── dto/create-<context>.command.ts
│   ├── dto/<context>.result.ts
│   └── port/                           # 필요할 때만
├── infrastructure/persistence/
│   ├── <context>.orm-entity.ts
│   ├── <context>.typeorm-repository.ts
│   └── <context>.mapper.ts
├── presentation/http/
│   ├── <context>.controller.ts
│   └── dto/{create-<context>.request.ts, <context>.response.ts}
└── <context>.module.ts
```

함께 만드는 것 — **이걸 빼면 스캐폴딩이 아니다.**

- `<context>.module.ts` 에 providers 전체 배선 + `exports: [<Context>Facade]`
- 루트 `AppModule` 의 `imports` 에 새 모듈 추가
- `.dependency-cruiser.js` 가 이미 컨텍스트 전체를 정규식으로 덮는지 확인 (안 덮으면 규칙 추가)
- 테스트 스캐폴드:
  - `test/fake/in-memory-<context>.repository.ts`
  - `<context>.spec.ts` (도메인 — 상태 전이표의 **모든 칸**을 `it.skip` 으로 미리 나열)
  - `create-<context>.usecase.spec.ts`
  - `test/integration/<context>.e2e-spec.ts`

## B. 백엔드 컨텍스트 (Python)

`.claude/docs/21-BACKEND-PYTHON.md` 구조를 따른다. 추가로 반드시 함께 만든다.

- `container.py` 팩토리 + `main.py` 의 `include_router`
- `.importlinter` 에 이 컨텍스트의 `layers` 계약 + `domain-is-pure` forbidden 계약 **복제 추가**
- `tests/<context>/` 에 `conftest.py`(Fake 픽스처), 도메인/UseCase/통합 테스트 스캐폴드
- 모든 `__init__.py`

---

## C. 프론트 FSD 슬라이스

`.claude/docs/30-FRONTEND-FSD.md` 구조를 따른다.

### entities
```
src/entities/<name>/
├── model/types.ts              # 도메인 모델 타입 (백엔드 용어 그대로)
├── model/<name>-status.ts      # 상태 전이 규칙
├── api/<name>.dto.ts           # 서버 응답 타입
├── api/<name>.mapper.ts        # DTO → 도메인 모델  ← ACL
├── api/queries.ts              # 쿼리 키 + useQuery
├── ui/                         # 표시 전용 컴포넌트
├── index.ts                    # ★ 공개 API — mapper/dto 는 내보내지 않는다
└── __tests__/<name>.mapper.test.ts
```

### features (이름은 동사-명사)
```
src/features/<verb-noun>/
├── model/schema.ts             # zod
├── model/use-<verb-noun>.ts
├── api/<verb-noun>.mutation.ts # invalidateQueries 대상 명시
├── ui/<VerbNoun>Form.tsx       # 'use client' 는 여기 (최하단)
├── index.ts
└── __tests__/<verb-noun>.test.tsx   # Testing Library + MSW
```

함께 하는 것
- 라우트가 필요하면 `src/app/<route>/page.tsx` — **위임 10줄 이하**
- MSW 핸들러 등록
- `npx steiger ./src` 실행해 통과 확인

---

## 생성 후 필수 검증

만들고 끝내지 않는다. 실제로 돌려 본다.

```bash
# NestJS
npx depcruise src --config .dependency-cruiser.js && npm run build && npm test

# Python
lint-imports && mypy app && pytest -m "not e2e"

# Next.js
npx steiger ./src && npm run lint && npm run test
```

## 보고

```
생성: src/contexts/booking/ (파일 18개)
배선: booking.module.ts → AppModule imports 추가
린터: .dependency-cruiser.js 규칙 적용 확인 ✅
테스트 스캐폴드: 도메인 6건 / UseCase 3건 / 통합 2건 — 전부 skip 상태
검증: depcruise ✅ / build ✅ / test 0 passed, 11 skipped

다음: /ddd-plan 으로 첫 기능을 계획하고 skip 을 하나씩 떼세요.
```

## 원칙

- **빈 파일을 만들지 않는다.** 각 파일에 규칙에 맞는 최소 구현 또는 명확한 TODO 를 넣는다.
- **테스트는 skip 상태로 목록을 먼저 만든다.** 그게 다음 작업 목록이 된다.
- 도메인 파일에 프레임워크 import 를 넣지 않는다.
- 배선(Module/container)과 린터 계약을 빼먹지 않는다 — 여기가 가장 자주 빠진다.
- 이미 있는 컨텍스트/슬라이스를 덮어쓰지 않는다. 존재하면 **확인하고 멈춘다.**
