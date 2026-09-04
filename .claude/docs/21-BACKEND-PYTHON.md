# 백엔드 규칙 — Python (FastAPI + SQLAlchemy)

전제: [10-DDD-CORE.md](10-DDD-CORE.md) 를 먼저 읽었다고 가정한다.
**레이어·의존 방향·Facade 규칙은 NestJS 와 완전히 동일**하다. 여기서는 Python 매핑만 다룬다.

| 개념 | NestJS | Python |
|---|---|---|
| 포트 인터페이스 | `interface` + `Symbol` 토큰 | `typing.Protocol` (또는 ABC) |
| DI | Nest Module provider | 컴포지션 루트 + FastAPI `Depends` (또는 `dependency-injector`) |
| VO | `class` + private 생성자 | `@dataclass(frozen=True, slots=True)` |
| 트랜잭션 | `TransactionPort.run()` | `UnitOfWork` 컨텍스트 매니저 |
| 요청 검증 | `class-validator` DTO | Pydantic 모델 (**presentation 에서만**) |

---

## 1. 디렉터리

```
app/
├── contexts/
│   └── booking/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── booking.py              # 애그리거트 루트
│       │   │   ├── booking_id.py
│       │   │   ├── booking_status.py
│       │   │   └── events.py
│       │   ├── repository.py               # Protocol (인터페이스)
│       │   ├── service/
│       │   │   └── slot_policy.py
│       │   └── errors.py
│       ├── application/
│       │   ├── facade.py                   # ★ 유일한 진입점
│       │   ├── usecase/
│       │   │   ├── create_booking.py
│       │   │   └── cancel_booking.py
│       │   ├── dto.py                      # Command / Query / Result (dataclass)
│       │   └── port.py                     # Notification, Clock, UnitOfWork Protocol
│       ├── infrastructure/
│       │   ├── persistence/
│       │   │   ├── models.py               # SQLAlchemy 모델
│       │   │   ├── repository.py           # 리포지토리 구현
│       │   │   └── mapper.py
│       │   ├── external/
│       │   └── acl/
│       ├── presentation/
│       │   ├── router.py                   # FastAPI APIRouter
│       │   └── schema.py                   # Pydantic 요청/응답
│       └── container.py                    # 컴포지션 루트
├── shared/
│   ├── kernel/
│   └── infrastructure/
└── main.py
```

---

## 2. 도메인 레이어 — 프레임워크가 없다

```python
# domain/model/booking_status.py
from __future__ import annotations
from dataclasses import dataclass
from typing import ClassVar, Mapping


@dataclass(frozen=True, slots=True)
class BookingStatus:
    value: str

    _TRANSITIONS: ClassVar[Mapping[str, frozenset[str]]] = {
        "PENDING": frozenset({"CONFIRMED", "CANCELLED"}),
        "CONFIRMED": frozenset({"CANCELLED", "COMPLETED"}),
        "CANCELLED": frozenset(),
        "COMPLETED": frozenset(),
    }

    def __post_init__(self) -> None:
        if self.value not in self._TRANSITIONS:
            raise InvalidBookingStatus(self.value)

    @classmethod
    def pending(cls) -> "BookingStatus":
        return cls("PENDING")

    def can_transition_to(self, nxt: "BookingStatus") -> bool:
        return nxt.value in self._TRANSITIONS[self.value]
```

```python
# domain/model/booking.py — 애그리거트 루트
from datetime import datetime


class Booking:
    def __init__(
        self,
        id: BookingId,
        customer_id: CustomerId,      # 다른 애그리거트는 ID로만 참조
        slot: TimeSlot,
        status: BookingStatus,
    ) -> None:
        self.id = id
        self.customer_id = customer_id
        self._slot = slot
        self._status = status
        self._events: list[DomainEvent] = []

    @classmethod
    def create(
        cls, id: BookingId, customer_id: CustomerId, slot: TimeSlot, now: datetime
    ) -> "Booking":
        if slot.starts_before(now):
            raise BookingInPast(slot)

        booking = cls(id, customer_id, slot, BookingStatus.pending())
        booking._record(BookingCreated(id.value, customer_id.value, slot.start, now))
        return booking

    def confirm(self, now: datetime) -> None:
        nxt = BookingStatus("CONFIRMED")
        if not self._status.can_transition_to(nxt):
            raise BookingTransitionNotAllowed(self._status, nxt)
        self._status = nxt
        self._record(BookingConfirmed(self.id.value, now))

    def _record(self, event: DomainEvent) -> None:
        self._events.append(event)

    def pull_events(self) -> list[DomainEvent]:
        events, self._events = self._events, []
        return events
```

**금지**

- 도메인 모듈에서 `sqlalchemy`, `pydantic`, `fastapi` import — 예외 없음.
- `datetime.now()` / `uuid4()` 를 도메인 안에서 호출 — **파라미터로 주입**한다 (`now: datetime`).
- 도메인 엔티티를 SQLAlchemy `Base` 상속으로 만들기. `imperative mapping` 을 쓰더라도 **매퍼로 분리**
  하는 쪽을 기본으로 한다.

---

## 3. 포트 — Protocol

```python
# domain/repository.py
from typing import Protocol


class BookingRepository(Protocol):
    async def find_by_id(self, id: BookingId) -> Booking | None: ...
    async def find_conflicting(self, slot: TimeSlot) -> list[Booking]: ...
    async def save(self, booking: Booking) -> None: ...
```

```python
# application/port.py
from typing import Protocol
from contextlib import AbstractAsyncContextManager


class UnitOfWork(Protocol):
    def begin(self) -> AbstractAsyncContextManager[None]: ...


class Clock(Protocol):
    def now(self) -> datetime: ...


class NotificationPort(Protocol):
    async def notify(self, to: CustomerId, message: str) -> None: ...
```

`Protocol` 을 쓰면 구현체가 상속을 선언하지 않아도 되고, 테스트용 가짜 구현을 3줄로 만들 수 있다.

---

## 4. UseCase

```python
# application/usecase/create_booking.py
class CreateBookingUseCase:
    def __init__(
        self,
        bookings: BookingRepository,
        slot_policy: BookingSlotPolicy,   # 도메인 서비스
        clock: Clock,
    ) -> None:
        self._bookings = bookings
        self._slot_policy = slot_policy
        self._clock = clock

    async def execute(self, command: CreateBookingCommand) -> BookingResult:
        slot = TimeSlot.of(command.start, command.end)
        conflicts = await self._bookings.find_conflicting(slot)

        self._slot_policy.ensure_available(slot, conflicts)   # 규칙 판단은 도메인이

        booking = Booking.create(
            BookingId.next(),
            CustomerId(command.customer_id),
            slot,
            self._clock.now(),
        )
        await self._bookings.save(booking)
        return BookingResult.from_domain(booking)
```

- Command / Result 는 **Pydantic 이 아니라 `@dataclass(frozen=True)`** 로 만든다.
  Pydantic 은 presentation 경계 전용이다.
- UseCase 는 트랜잭션을 열지 않는다 (§5).

---

## 5. Facade — 공개 API + 트랜잭션 경계

```python
# application/facade.py
class BookingFacade:
    def __init__(
        self,
        create_booking: CreateBookingUseCase,
        cancel_booking: CancelBookingUseCase,
        get_booking: GetBookingQuery,
        uow: UnitOfWork,
    ) -> None:
        self._create_booking = create_booking
        self._cancel_booking = cancel_booking
        self._get_booking = get_booking
        self._uow = uow

    async def create(self, command: CreateBookingCommand) -> BookingResult:
        async with self._uow.begin():                     # 트랜잭션 경계는 여기 한 곳
            return await self._create_booking.execute(command)

    async def cancel(self, command: CancelBookingCommand) -> BookingResult:
        async with self._uow.begin():
            return await self._cancel_booking.execute(command)

    async def find_by_id(self, booking_id: str) -> BookingView | None:
        return await self._get_booking.execute(booking_id)   # 읽기는 트랜잭션 없이
```

- 라우터·다른 컨텍스트는 **`BookingFacade` 만** 참조한다.
- Facade 는 `Result` / `View` dataclass 만 반환한다. `Booking` 엔티티 반환 금지.

---

## 6. presentation / infrastructure / 컴포지션 루트

```python
# presentation/schema.py
from pydantic import BaseModel, Field


class CreateBookingRequest(BaseModel):
    customer_id: str = Field(min_length=1)
    start: datetime
    end: datetime

    def to_command(self) -> CreateBookingCommand:
        return CreateBookingCommand(
            customer_id=self.customer_id, start=self.start, end=self.end
        )


class BookingResponse(BaseModel):
    id: str
    status: str

    @classmethod
    def from_result(cls, result: BookingResult) -> "BookingResponse":
        return cls(id=result.id, status=result.status)
```

```python
# presentation/router.py
router = APIRouter(prefix="/bookings", tags=["booking"])


@router.post("", response_model=BookingResponse, status_code=201)
async def create_booking(
    body: CreateBookingRequest,
    facade: BookingFacade = Depends(get_booking_facade),   # Facade 만 주입
) -> BookingResponse:
    result = await facade.create(body.to_command())
    return BookingResponse.from_result(result)
```

- 라우터가 UseCase·리포지토리·세션(`AsyncSession`)을 직접 `Depends` 로 받으면 **위반**이다.
- 도메인 예외 → HTTP 매핑은 `app.add_exception_handler` 한 곳에서 표로 관리한다.

```python
# container.py — 배선은 여기서만
def get_booking_facade(session: AsyncSession = Depends(get_session)) -> BookingFacade:
    repo = SqlAlchemyBookingRepository(session)
    clock = SystemClock()
    return BookingFacade(
        create_booking=CreateBookingUseCase(repo, BookingSlotPolicy(), clock),
        cancel_booking=CancelBookingUseCase(repo, clock),
        get_booking=GetBookingQuery(session),
        uow=SqlAlchemyUnitOfWork(session),
    )
```

- 프로젝트가 커지면 `dependency-injector` 의 `DeclarativeContainer` 로 옮긴다. 그래도
  **배선 지점은 계속 한 곳**이어야 한다.
- 리포지토리 구현은 `save()` 후 `booking.pull_events()` 로 이벤트를 꺼내 디스패처에 넘긴다.

---

## 7. 의존 방향을 CI 로 강제 — import-linter

`.importlinter`:

```ini
[importlinter]
root_packages = app

[importlinter:contract:booking-layers]
name = booking 레이어 의존 방향
type = layers
layers =
    app.contexts.booking.presentation
    app.contexts.booking.application
    app.contexts.booking.domain

[importlinter:contract:domain-is-pure]
name = 도메인은 프레임워크를 모른다
type = forbidden
source_modules =
    app.contexts.booking.domain
forbidden_modules =
    sqlalchemy
    pydantic
    fastapi

[importlinter:contract:presentation-no-infra]
name = presentation 은 infrastructure 를 직접 쓰지 않는다
type = forbidden
source_modules =
    app.contexts.booking.presentation
forbidden_modules =
    app.contexts.booking.infrastructure

[importlinter:contract:contexts-independent]
name = 컨텍스트 내부는 서로 직접 참조하지 않는다 (Facade 만 허용)
type = independence
modules =
    app.contexts.booking.domain
    app.contexts.settlement.domain
```

```bash
lint-imports        # CI 필수, 커밋 전 게이트
```

> 컨텍스트를 추가할 때마다 `booking-layers` 계약을 복제해 넣는다. `/ddd-scaffold` 가 이 작업까지 한다.

---

## 8. 체크리스트 (커밋 전)

- [ ] `lint-imports` 통과
- [ ] 도메인에 `sqlalchemy` / `pydantic` / `fastapi` import 0건
- [ ] 라우터가 `Depends` 로 받는 것은 Facade 뿐 (`AsyncSession` 직접 주입 없음)
- [ ] Command / Result 는 dataclass, Pydantic 은 presentation 에만
- [ ] SQLAlchemy 모델 ↔ 도메인 엔티티 매퍼 존재
- [ ] 트랜잭션(`uow.begin()`)은 Facade 메서드당 1번
- [ ] 도메인 단위 테스트 + UseCase 테스트 + 통합 테스트 존재 ([40-TDD.md](40-TDD.md))
- [ ] `mypy --strict` 통과 (도메인/애플리케이션 레이어는 예외 없이)
