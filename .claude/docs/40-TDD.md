# TDD 규칙

**실패하는 테스트 없이 프로덕션 코드를 쓰지 않는다.** 이게 이 문서의 전부고, 나머지는 그걸 실제로
가능하게 만드는 구조 이야기다.

DDD 와 TDD 는 같은 것을 요구한다 — **도메인이 IO 를 모르면 테스트가 빨라지고, 테스트가 빠르면 설계가
개선된다.** 도메인 테스트에 DB 나 목이 필요하다면 그건 테스트 문제가 아니라 **설계가 잘못된 신호**다.

---

## 1. 사이클

```
RED     실패하는 테스트를 먼저 쓴다. 컴파일조차 안 되는 상태가 정상이다.
GREEN   테스트를 통과시키는 가장 단순한 코드를 쓴다. 예쁘게 만들지 않는다.
REFACTOR 테스트가 초록인 상태를 유지하며 구조를 고친다. 이때 DDD 규칙을 적용한다.
```

- 한 사이클은 **작다.** 한 번에 테스트 1~3개.
- RED 단계에서 **테스트가 실제로 실패하는 것을 눈으로 확인**한다. 통과해 버리면 그 테스트는 아무것도
  검증하지 않고 있다.
- 아직 구현이 없어 컨텍스트가 깨지는 테스트는 일단 **skip 으로 커밋 가능한 상태**를 유지한다
  (Jest/Vitest `it.skip`, pytest `@pytest.mark.skip`). 구현하면서 skip 을 하나씩 지운다.
  **skip 이 남은 채로 커밋하지 않는다.**

---

## 2. 테스트 계층 — 무엇을 어디까지 실물로 쓰나

| 계층 | 대상 | 의존성 처리 | 개수 | 속도 |
|---|---|---|---|---|
| **도메인 단위** | 엔티티·VO·도메인 서비스 | **아무것도 대체 안 함** (순수 함수) | 가장 많이 | ms |
| **UseCase** | 유스케이스 1개 | 리포지토리·포트를 **인메모리 Fake** 로 | 많이 | ms |
| **통합** | Facade + 실제 DB + 실제 라우팅 | **실물 DB** (testcontainers / 로컬 DB) | 시나리오당 1~N | 초 |
| **계약(선택)** | 외부 API 어댑터 | 녹화된 응답 / 스텁 서버 | 어댑터당 소수 | ms |
| **E2E (프론트)** | 브라우저 → 백엔드 | 실물 스택 | 핵심 플로우만 | 분 |

### Mock 이 아니라 Fake

목 프레임워크(`jest.fn()`, `unittest.mock`)로 리포지토리를 흉내 내지 않는다. **인메모리 구현체를 만든다.**

```ts
// test/fake/in-memory-booking.repository.ts
export class InMemoryBookingRepository implements BookingRepository {
  private readonly rows = new Map<string, Booking>();

  async findById(id: BookingId) { return this.rows.get(id.value) ?? null; }
  async findConflicting(slot: TimeSlot) {
    return [...this.rows.values()].filter((b) => b.overlaps(slot));
  }
  async save(booking: Booking) { this.rows.set(booking.id.value, booking); }
}
```

```python
# tests/fakes.py
class InMemoryBookingRepository:
    def __init__(self) -> None:
        self._rows: dict[str, Booking] = {}

    async def find_by_id(self, id: BookingId) -> Booking | None:
        return self._rows.get(id.value)

    async def save(self, booking: Booking) -> None:
        self._rows[booking.id.value] = booking
```

왜 Fake 인가: 목은 **호출 순서와 인자**를 검증하게 만들어 테스트를 구현 세부에 묶는다. 리팩터링하면
로직이 그대로여도 테스트가 깨진다. Fake 는 **결과 상태**를 검증하게 해 준다.
목은 **외부 경계(결제사, 메일 발송)에서 "보냈는지"를 확인할 때만** 쓴다.

### 시간·랜덤은 주입한다

`FixedClock(2026-01-01T00:00:00Z)`, `SequentialIdGenerator`. 도메인이 `new Date()` 를 부르면
테스트가 시간에 흔들린다 — 그래서 [20](20-BACKEND-NESTJS.md)/[21](21-BACKEND-PYTHON.md) 에서
`now` 를 파라미터로 받게 한 것이다.

---

## 3. 무엇을 테스트하나

**테스트하는 것**
- 도메인 규칙과 그 경계값 (상태 전이 표의 **모든 칸**, 정원 초과 직전/직후)
- 불변식 위반 시 **어떤 예외가** 나는가
- UseCase 의 성공 경로 + 각 실패 경로
- Facade 트랜잭션: 중간에 실패하면 **아무것도 저장되지 않는가** (통합 테스트에서 DB 직접 조회로 확인)
- 발행된 도메인 이벤트의 종류와 페이로드

**테스트하지 않는 것**
- 프레임워크 동작 (`@IsUUID()` 가 UUID 를 검증하는지)
- ORM 이 SQL 을 만드는지
- getter/setter, 매퍼의 필드 1:1 복사 (매퍼는 통합 테스트가 자연히 커버한다)
- 커버리지 숫자를 채우기 위한 테스트

> 커버리지 목표는 **도메인 레이어 90%+**, 나머지는 숫자를 목표로 삼지 않는다.

---

## 4. 테스트 이름

```
<대상>_<상황>_<기대>
```

- ✅ `confirm_pending 상태에서_CONFIRMED 로 전이한다`
- ✅ `create_시작 시각이 과거면_BookingInPast 를 던진다`
- ❌ `test1`, `test_confirm`, `works correctly`

**이름은 유비쿼터스 언어로 쓴다.** 테스트 목록이 곧 도메인 명세가 되어야 한다.

---

## 5. 통합 테스트 필수 구조

세부 작성법은 `/write-integration-test` 스킬이 안내한다. 최소 요건:

- **실제 DB** 를 쓴다. 리포지토리를 목으로 바꾼 "통합" 테스트는 통합 테스트가 아니다.
- **HTTP 를 통해** 진입한다 (supertest / httpx). Facade 를 직접 부르는 건 UseCase 테스트다.
- **DB 를 직접 조회**해 저장 상태를 검증한다. API 응답만 믿지 않는다.
- 테스트마다 **데이터 정리**를 보장한다 (트랜잭션 롤백 또는 명시적 cleanup). 순서 의존은 최후의 수단.
- 실패 경로도 다룬다: 409 충돌, 404, 권한 없음, 트랜잭션 롤백.

---

## 6. 프론트엔드 TDD

| 대상 | 도구 | 방식 |
|---|---|---|
| `entities/*/model` 순수 로직 | Vitest | 단위. 렌더링 없음 |
| `entities/*/api/mapper` | Vitest | DTO 픽스처 → 도메인 모델 변환 검증 |
| `features/*` | Vitest + Testing Library + MSW | 사용자 관점(`getByRole`)으로. 서버는 MSW 로 스텁 |
| `widgets/` `views/` | Playwright | 핵심 플로우만 |

- 컴포넌트 내부 상태나 구현을 단언하지 않는다. **사용자가 보는 것**을 단언한다.
- MSW 핸들러는 `shared/api/mocks/` 또는 슬라이스별로 두고, **실제 DTO 타입을 그대로 사용**한다.
  타입이 어긋나면 여기서 먼저 깨져야 한다.
- 스냅샷 테스트는 기본적으로 쓰지 않는다. 쓰더라도 인라인 스냅샷 + 작은 단위로.

---

## 7. 커밋 전 게이트

```bash
# NestJS
npm run test:unit && npm run test:integration && npx depcruise src --config .dependency-cruiser.js

# Python
pytest -m "not e2e" && lint-imports && mypy app

# Next.js
npm run test && npx steiger ./src && npm run lint
```

- [ ] 새 코드에 대응하는 실패했던 테스트가 있었다
- [ ] `skip` / `only` 가 남아 있지 않다
- [ ] 도메인 테스트가 DB·네트워크 없이 돈다
- [ ] 통합 테스트가 DB 직접 조회로 상태를 검증한다
- [ ] 테스트 이름이 유비쿼터스 언어로 되어 있다
