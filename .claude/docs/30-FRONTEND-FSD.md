# 프론트엔드 규칙 — Next.js (App Router) + FSD

FSD(Feature-Sliced Design)는 프론트엔드판 DDD 다. **레이어 = 의존 방향**, **슬라이스 = 도메인 경계**,
**공개 API(`index.ts`) = Facade**. 백엔드와 같은 원리를 프론트 어휘로 쓴 것으로 이해하면 된다.

| 백엔드 개념 | 프론트 대응 |
|---|---|
| 바운디드 컨텍스트 | 슬라이스 (`entities/booking`, `features/create-booking`) |
| Facade (컨텍스트 공개 API) | 슬라이스의 `index.ts` |
| 애그리거트/엔티티 | `entities/<name>/model` 의 타입 + 파생 로직 |
| UseCase | `features/<verb-noun>` |
| ACL (외부 모델 번역) | `entities/<name>/api` 의 DTO → 도메인 모델 매퍼 |
| 유비쿼터스 언어 | **백엔드와 동일한 단어를 슬라이스명·타입명에 그대로 쓴다** |

---

## 1. 레이어와 디렉터리

```
src/
├── app/                     # Next.js App Router — 라우팅 껍데기만
│   ├── layout.tsx
│   ├── page.tsx
│   ├── bookings/
│   │   └── page.tsx         # → views/booking-list 를 렌더링만
│   └── _providers/          # FSD 'app' 레이어: Provider·전역 스타일 (private 폴더, 라우팅 제외)
├── views/                   # FSD 'pages' 레이어 (Next 예약어 회피용 이름)
│   └── booking-list/
├── widgets/                 # 여러 feature/entity 를 조합한 화면 블록
│   └── booking-table/
├── features/                # 사용자 의도 = UseCase
│   ├── create-booking/
│   └── cancel-booking/
├── entities/                # 도메인 모델
│   └── booking/
└── shared/                  # 도메인 지식 없는 재사용 자원
    ├── ui/  api/  lib/  config/
```

### 왜 `views/` 인가

Next.js App Router 는 `app/` 을, 구 Pages Router 는 `pages/` 를 예약어로 쓴다. FSD 의 `pages` 레이어와
이름이 충돌하므로 **`views/`** 로 쓴다. FSD `app` 레이어(Provider·전역 스타일)는
`src/app/_providers/` 에 둔다 — `_` 접두사 폴더는 Next 가 라우트로 해석하지 않는다.

### 의존 방향 (위 → 아래만 허용)

```
app  →  views  →  widgets  →  features  →  entities  →  shared
```

- **아래에서 위로는 절대 금지.** `entities` 가 `features` 를 import 하면 위반.
- **같은 레이어의 슬라이스끼리도 금지.** `features/create-booking` → `features/cancel-booking` 위반.
  → 공통이 필요하면 아래 레이어(`entities` / `shared`)로 내린다.
- 예외는 `entities` 레이어의 **cross-import 전용 공개 API** (`entities/booking/@x/customer.ts`) 뿐이다.
  이것도 남발하면 슬라이스 경계가 잘못 그어진 것이다.

---

## 2. 슬라이스와 세그먼트

슬라이스 내부는 **세그먼트**로 나눈다.

| 세그먼트 | 담는 것 |
|---|---|
| `ui/` | 컴포넌트, 스타일 |
| `model/` | 타입, 상태(store), 파생 로직, 검증 — **도메인 규칙이 여기 산다** |
| `api/` | 서버 요청, DTO ↔ 도메인 모델 매퍼, 쿼리 키 |
| `lib/` | 이 슬라이스 전용 유틸 |
| `config/` | 상수, 플래그 |
| `index.ts` | **공개 API. 밖에서는 여기로만 들어온다** |

```
entities/booking/
├── model/
│   ├── types.ts             # Booking, BookingStatus …
│   ├── booking-status.ts    # 상태 전이 규칙 (백엔드와 같은 규칙을 표현)
│   └── selectors.ts
├── api/
│   ├── booking.dto.ts       # 서버 응답 그대로
│   ├── booking.mapper.ts    # DTO → 도메인 모델  ← ACL
│   └── queries.ts           # useBookingQuery …
├── ui/
│   └── BookingStatusBadge.tsx
└── index.ts
```

```ts
// entities/booking/index.ts  — 공개 API = Facade
export type { Booking, BookingStatus } from './model/types';
export { canTransitionTo } from './model/booking-status';
export { useBookingQuery, bookingKeys } from './api/queries';
export { BookingStatusBadge } from './ui/BookingStatusBadge';
// mapper·dto 는 내보내지 않는다 — 슬라이스 내부 구현이다
```

**금지: 깊은 import**

```ts
import { Booking } from '@/entities/booking';                   // ✅
import { Booking } from '@/entities/booking/model/types';       // ❌ 위반
```

---

## 3. 레이어별 책임

### `entities/` — 도메인 모델

- 서버 DTO 를 그대로 화면까지 흘려보내지 않는다. **`api/mapper.ts` 에서 도메인 모델로 번역**한다(ACL).
  서버 스키마가 바뀌어도 고칠 파일은 매퍼 하나여야 한다.
- 상태 전이·표시 규칙 같은 **도메인 판단은 `model/` 에** 둔다. 컴포넌트 안 `if` 로 흩뿌리지 않는다.
- **백엔드 유비쿼터스 언어를 그대로 쓴다.** 백엔드가 `Booking.status = CONFIRMED` 면 프론트도
  `confirmed`/`CONFIRMED` 다. 프론트에서 `isDone` 같은 사투리를 만들지 않는다.
- 엔티티는 **자기 CRUD 훅까지만** 갖는다. "예약을 취소하고 알림을 보낸다" 같은 시나리오는 `features` 다.

### `features/` — 사용자 의도 (UseCase)

- 이름은 **동사-명사**: `create-booking`, `cancel-booking`, `filter-booking-list`.
- 하나의 feature = 하나의 사용자 의도. mutation + 낙관적 업데이트 + 폼 검증 + 성공/실패 처리까지 포함.
- feature 끼리 직접 부르지 않는다. 조합은 `widgets` / `views` 의 몫이다.

```
features/create-booking/
├── model/
│   ├── schema.ts            # zod 스키마 (입력 형식 검증)
│   └── use-create-booking.ts
├── api/create-booking.mutation.ts
├── ui/CreateBookingForm.tsx
└── index.ts
```

### `widgets/` — 조합 블록

- 여러 entity/feature 를 붙여 만든 독립적 화면 조각(테이블 + 필터 + 액션 버튼).
- 자체 비즈니스 규칙을 갖지 않는다. 규칙이 생기면 아래로 내린다.

### `views/` — 페이지 구성

- 위젯 배치, 페이지 단위 데이터 프리페치, 레이아웃. 여기에 도메인 로직을 쓰지 않는다.

### `app/` — Next.js 라우팅 껍데기

```tsx
// src/app/bookings/page.tsx
import { BookingListPage } from '@/views/booking-list';

export default function Page() {
  return <BookingListPage />;
}
```

- `page.tsx` / `layout.tsx` / `route.ts` 는 **10줄을 넘기지 않는다.** 넘으면 `views` 로 내려야 한다.
- `metadata`, `generateStaticParams`, route handler 정도만 여기 산다.

### `shared/` — 도메인을 모르는 것만

- 디자인 시스템 컴포넌트, HTTP 클라이언트, 포맷터, 타입 유틸.
- **`shared` 에 도메인 단어가 등장하면 위반이다.** `shared/ui/BookingCard.tsx` 는 `entities/booking/ui` 로.

---

## 4. 서버 컴포넌트 / 데이터 페칭 규칙

- 기본은 **서버 컴포넌트**. `'use client'` 는 상호작용이 필요한 **가장 낮은 컴포넌트**에만 붙인다.
  슬라이스 `index.ts` 나 위젯 루트에 붙이면 트리 전체가 클라이언트로 내려간다.
- 서버 데이터 페칭은 `views` 또는 서버 컴포넌트에서, 클라이언트 상태는 TanStack Query 로.
  **쿼리 키는 반드시 `entities/<name>/api/queries.ts` 에서 정의해 내보낸다** (`bookingKeys.detail(id)`).
  키를 여기저기 인라인으로 쓰면 무효화가 반드시 깨진다.
- Server Action 은 `features/<name>/api/` 에 두고 `'use server'` 를 파일 상단에 붙인다.
  Server Action 은 **백엔드 Facade 를 호출하는 얇은 어댑터**이지, 비즈니스 로직 자리가 아니다.

---

## 5. 상태 분류

| 종류 | 도구 | 위치 |
|---|---|---|
| 서버 상태 | TanStack Query | `entities/*/api` (조회), `features/*/api` (변경) |
| 전역 클라이언트 상태 | zustand 등 | `entities/*/model` 또는 `shared` |
| 폼 상태 | react-hook-form + zod | `features/*/model` |
| URL 상태 (필터·페이지) | `useSearchParams` | `features/*/model` |

> 서버 상태를 전역 스토어에 복사하지 않는다. 캐시 무효화 대신 수동 동기화를 하는 순간 버그가 시작된다.

---

## 6. 규칙을 기계로 강제

```bash
npx steiger ./src        # FSD 공식 린터: 레이어 위반·공개 API 누락·고아 슬라이스 검출
```

`eslint.config.js` — 깊은 import / 역방향 의존 차단:

```js
// eslint-plugin-boundaries
settings: {
  'boundaries/elements': [
    { type: 'app',      pattern: 'src/app/*' },
    { type: 'views',    pattern: 'src/views/*' },
    { type: 'widgets',  pattern: 'src/widgets/*' },
    { type: 'features', pattern: 'src/features/*' },
    { type: 'entities', pattern: 'src/entities/*' },
    { type: 'shared',   pattern: 'src/shared/*' },
  ],
},
rules: {
  'boundaries/element-types': ['error', {
    default: 'disallow',
    rules: [
      { from: 'app',      allow: ['views', 'widgets', 'features', 'entities', 'shared'] },
      { from: 'views',    allow: ['widgets', 'features', 'entities', 'shared'] },
      { from: 'widgets',  allow: ['features', 'entities', 'shared'] },
      { from: 'features', allow: ['entities', 'shared'] },
      { from: 'entities', allow: ['shared'] },
      { from: 'shared',   allow: ['shared'] },
    ],
  }],
  'no-restricted-imports': ['error', {
    patterns: [{
      group: ['@/entities/*/*', '@/features/*/*', '@/widgets/*/*', '@/views/*/*'],
      message: '슬라이스 내부 깊은 import 금지 — index.ts 공개 API 를 쓰세요.',
    }],
  }],
}
```

---

## 7. 체크리스트 (커밋 전)

- [ ] `npx steiger ./src` 통과 / ESLint boundaries 통과
- [ ] 새 슬라이스에 `index.ts` 공개 API 존재, 내부 파일 직접 import 0건
- [ ] 같은 레이어 슬라이스 간 import 0건
- [ ] `app/` 의 `page.tsx` 가 10줄 이하 (뷰 위임만)
- [ ] 서버 DTO 가 `entities/*/api/mapper` 를 거쳐 도메인 모델로 변환됨
- [ ] `shared/` 에 도메인 단어 없음
- [ ] `'use client'` 가 필요한 최소 컴포넌트에만
- [ ] 쿼리 키가 `entities/*/api/queries.ts` 에 집중
- [ ] 타입·상태값 이름이 백엔드 유비쿼터스 언어와 일치
- [ ] feature `model` 단위 테스트 + 주요 시나리오 e2e 존재 ([40-TDD.md](40-TDD.md))
