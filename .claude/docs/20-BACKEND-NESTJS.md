# 백엔드 규칙 — NestJS (TypeScript)

전제: [10-DDD-CORE.md](10-DDD-CORE.md) 를 먼저 읽었다고 가정한다. 여기서는 그 규칙의 **NestJS 구현**만 다룬다.

---

## 1. 디렉터리

```
src/
├── contexts/
│   └── booking/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── booking.ts                  # 애그리거트 루트
│       │   │   ├── booking-id.vo.ts
│       │   │   ├── booking-status.vo.ts
│       │   │   └── booking-confirmed.event.ts
│       │   ├── repository/
│       │   │   └── booking.repository.ts       # 인터페이스 + DI 토큰
│       │   ├── service/
│       │   │   └── booking-slot.policy.ts
│       │   └── error/
│       │       └── booking.error.ts
│       ├── application/
│       │   ├── booking.facade.ts               # ★ 유일한 진입점
│       │   ├── usecase/
│       │   │   ├── create-booking.usecase.ts
│       │   │   └── cancel-booking.usecase.ts
│       │   ├── dto/
│       │   │   ├── create-booking.command.ts
│       │   │   └── booking.result.ts
│       │   └── port/
│       │       ├── notification.port.ts
│       │       └── transaction.port.ts
│       ├── infrastructure/
│       │   ├── persistence/
│       │   │   ├── booking.orm-entity.ts
│       │   │   ├── booking.typeorm-repository.ts
│       │   │   └── booking.mapper.ts
│       │   ├── external/
│       │   └── acl/
│       ├── presentation/
│       │   └── http/
│       │       ├── booking.controller.ts
│       │       └── dto/
│       │           ├── create-booking.request.ts
│       │           └── booking.response.ts
│       └── booking.module.ts                   # 컴포지션 루트
├── shared/
│   ├── kernel/          # Money, Email, DateRange …
│   └── infrastructure/  # 로거, 트랜잭션 구현, 공통 필터
└── main.ts
```

**파일명 규칙**: `<이름>.<역할>.ts` — `.vo.ts` `.event.ts` `.usecase.ts` `.facade.ts` `.port.ts`
`.repository.ts` `.orm-entity.ts` `.mapper.ts` `.request.ts` `.response.ts`.
역할이 파일명에 없으면 리뷰에서 지적한다.

---

## 2. 도메인 레이어 — 프레임워크가 없다

```ts
// domain/model/booking-status.vo.ts
export class BookingStatus {
  private static readonly TRANSITIONS: Record<string, readonly string[]> = {
    PENDING:   ['CONFIRMED', 'CANCELLED'],
    CONFIRMED: ['CANCELLED', 'COMPLETED'],
    CANCELLED: [],
    COMPLETED: [],
  };

  private constructor(readonly value: string) {}

  static pending(): BookingStatus {
    return new BookingStatus('PENDING');
  }

  static of(value: string): BookingStatus {
    if (!(value in BookingStatus.TRANSITIONS)) {
      throw new InvalidBookingStatus(value);
    }
    return new BookingStatus(value);
  }

  canTransitionTo(next: BookingStatus): boolean {
    return BookingStatus.TRANSITIONS[this.value].includes(next.value);
  }

  equals(other: BookingStatus): boolean {
    return this.value === other.value;
  }
}
```

```ts
// domain/model/booking.ts  — 애그리거트 루트
export class Booking {
  private readonly events: DomainEvent[] = [];

  private constructor(
    readonly id: BookingId,
    readonly customerId: CustomerId,      // 다른 애그리거트는 ID로만 참조
    private slot: TimeSlot,
    private status: BookingStatus,
  ) {}

  /** 생성도 도메인 규칙이다. 밖에서 new 로 만들지 않는다. */
  static create(id: BookingId, customerId: CustomerId, slot: TimeSlot, now: Date): Booking {
    if (slot.startsBefore(now)) throw new BookingInPast(slot);

    const booking = new Booking(id, customerId, slot, BookingStatus.pending());
    booking.record(new BookingCreated(id.value, customerId.value, slot.start, now));
    return booking;
  }

  /** 의도를 드러내는 메서드로만 상태를 바꾼다. setter 금지. */
  confirm(now: Date): void {
    const next = BookingStatus.of('CONFIRMED');
    if (!this.status.canTransitionTo(next)) {
      throw new BookingTransitionNotAllowed(this.status, next);
    }
    this.status = next;
    this.record(new BookingConfirmed(this.id.value, now));
  }

  cancel(reason: CancelReason, now: Date): void {
    /* … 동일한 형태 … */
  }

  private record(event: DomainEvent): void {
    this.events.push(event);
  }

  /** 저장이 성공한 뒤 인프라가 꺼내 간다. */
  pullEvents(): DomainEvent[] {
    return this.events.splice(0);
  }
}
```

**금지**

- `@Entity()` `@Column()` `@Injectable()` 등 데코레이터를 도메인에 붙이지 않는다.
- `Date.now()` / `Math.random()` / `uuid()` 를 도메인 안에서 호출하지 않는다 → **파라미터로 주입**
  (`now: Date`, `id: BookingId`). 테스트 가능성이 여기서 갈린다.
- `null` 을 상태로 쓰지 않는다. VO 나 명시적 상태값으로 표현한다.

---

## 3. 포트와 DI 토큰

NestJS 는 인터페이스를 런타임에 주입할 수 없다. **토큰을 인터페이스 옆에 함께 정의**한다.

```ts
// domain/repository/booking.repository.ts
export const BOOKING_REPOSITORY = Symbol('BookingRepository');

export interface BookingRepository {
  findById(id: BookingId): Promise<Booking | null>;
  findConflicting(slot: TimeSlot): Promise<Booking[]>;
  save(booking: Booking): Promise<void>;
}
```

```ts
// application/port/transaction.port.ts
export const TRANSACTION = Symbol('TransactionPort');

export interface TransactionPort {
  run<T>(fn: () => Promise<T>): Promise<T>;
}
```

- 리포지토리 인터페이스는 **domain** 에, 그 밖의 아웃바운드 포트(알림·스토리지·시계·트랜잭션)는
  **application/port** 에 둔다.
- 포트 메서드는 도메인 타입을 주고받는다. ORM 타입·HTTP 타입 금지.

---

## 4. UseCase — 시나리오 하나

```ts
// application/usecase/create-booking.usecase.ts
@Injectable()
export class CreateBookingUseCase {
  constructor(
    @Inject(BOOKING_REPOSITORY) private readonly bookings: BookingRepository,
    private readonly slotPolicy: BookingSlotPolicy,      // 도메인 서비스
    @Inject(CLOCK) private readonly clock: ClockPort,
  ) {}

  async execute(command: CreateBookingCommand): Promise<BookingResult> {
    const slot = TimeSlot.of(command.start, command.end);
    const conflicts = await this.bookings.findConflicting(slot);

    this.slotPolicy.ensureAvailable(slot, conflicts);      // 규칙 판단은 도메인이

    const booking = Booking.create(
      BookingId.next(),
      CustomerId.of(command.customerId),
      slot,
      this.clock.now(),
    );
    await this.bookings.save(booking);

    return BookingResult.from(booking);                    // 엔티티를 그대로 내보내지 않는다
  }
}
```

- UseCase 는 **입력 Command → 출력 Result**. HTTP 를 모른다.
- 분기가 늘어나면 그건 대개 도메인 규칙이다. UseCase 가 아니라 도메인으로 옮긴다.
- UseCase 는 트랜잭션을 열지 않는다 (§5).

---

## 5. Facade — 컨텍스트의 공개 API이자 트랜잭션 경계

```ts
// application/booking.facade.ts
@Injectable()
export class BookingFacade {
  constructor(
    private readonly createBooking: CreateBookingUseCase,
    private readonly cancelBooking: CancelBookingUseCase,
    private readonly getBooking: GetBookingQuery,
    @Inject(TRANSACTION) private readonly tx: TransactionPort,
  ) {}

  /** 쓰기: 트랜잭션 경계는 여기서 한 번만 열린다. */
  create(command: CreateBookingCommand): Promise<BookingResult> {
    return this.tx.run(() => this.createBooking.execute(command));
  }

  cancel(command: CancelBookingCommand): Promise<BookingResult> {
    return this.tx.run(() => this.cancelBooking.execute(command));
  }

  /** 읽기: 트랜잭션 없이 read model 직행 가능 */
  findById(id: string): Promise<BookingView | null> {
    return this.getBooking.execute(id);
  }
}
```

규칙

- **트랜잭션은 Facade 메서드당 1개.** 중첩 금지. UseCase 안에서 `tx.run` 호출 금지.
- 한 Facade 메서드가 UseCase 를 여러 개 부른다면 **애그리거트를 여러 개 수정하고 있지 않은지** 확인한다.
  수정한다면 이벤트/사가로 나누는 게 맞는지 계획 단계에서 판단한다.
- 다른 컨텍스트가 이 컨텍스트를 쓸 때도 **`BookingFacade` 만** 주입받는다.
- Facade 는 `Result` / `View` DTO 만 반환한다. `Booking` 엔티티 반환 금지.
- Facade 가 200줄을 넘거나 `if` 로 비즈니스 분기를 하면 로직이 새어 들어온 것이다.

---

## 6. presentation / infrastructure

```ts
// presentation/http/booking.controller.ts
@Controller('bookings')
export class BookingController {
  constructor(private readonly facade: BookingFacade) {}   // Facade 외 주입 금지

  @Post()
  async create(@Body() body: CreateBookingRequest): Promise<BookingResponse> {
    const result = await this.facade.create(body.toCommand());
    return BookingResponse.from(result);
  }
}
```

- 요청 DTO 에서만 `class-validator` 를 쓴다 (`@IsUUID()`, `@IsISO8601()`). 도메인은 자기 생성자에서
  스스로 검증한다. **양쪽 다 한다** — 형식 검증(presentation)과 규칙 검증(domain)은 다른 일이다.
- `toCommand()` / `from()` 변환은 DTO 클래스가 갖는다. 컨트롤러에 매핑 코드를 늘어놓지 않는다.
- 도메인 예외 → HTTP 매핑은 컨텍스트별 `ExceptionFilter` 한 곳에서 한다.

```ts
// infrastructure/persistence/booking.mapper.ts
export class BookingMapper {
  static toDomain(row: BookingOrmEntity): Booking { /* … */ }
  static toOrm(booking: Booking): BookingOrmEntity { /* … */ }
}
```

- **ORM 엔티티와 도메인 엔티티는 항상 별개 클래스**다. 매퍼 없이 통과시키면 위반.
- 리포지토리 구현은 `save()` 후 `booking.pullEvents()` 를 꺼내 이벤트 디스패처로 넘긴다.

```ts
// booking.module.ts — 배선은 여기 한 곳에서만
@Module({
  imports: [TypeOrmModule.forFeature([BookingOrmEntity])],
  controllers: [BookingController],
  providers: [
    BookingFacade,
    CreateBookingUseCase,
    CancelBookingUseCase,
    GetBookingQuery,
    BookingSlotPolicy,
    { provide: BOOKING_REPOSITORY, useClass: BookingTypeormRepository },
    { provide: NOTIFICATION, useClass: SlackNotificationAdapter },
  ],
  exports: [BookingFacade],           // ★ Facade 만 export
})
export class BookingModule {}
```

> `exports` 에 Facade 이외의 것이 있으면 컨텍스트 캡슐화가 깨진 것이다.

---

## 7. 의존 방향을 CI 로 강제

`.dependency-cruiser.js`:

```js
module.exports = {
  forbidden: [
    {
      name: 'domain-no-outward',
      severity: 'error',
      from: { path: '^src/contexts/[^/]+/domain' },
      to: { path: '^src/contexts/[^/]+/(application|infrastructure|presentation)' },
    },
    {
      name: 'domain-no-framework',
      severity: 'error',
      from: { path: '^src/contexts/[^/]+/domain' },
      to: {
        dependencyTypes: ['npm'],
        path: '^(@nestjs|typeorm|@prisma|class-validator|axios)',
      },
    },
    {
      name: 'application-no-infra',
      severity: 'error',
      from: { path: '^src/contexts/[^/]+/application' },
      to: { path: '^src/contexts/[^/]+/infrastructure' },
    },
    {
      name: 'presentation-facade-only',
      severity: 'error',
      comment: '컨트롤러는 Facade 로만 들어간다',
      from: { path: '^src/contexts/[^/]+/presentation' },
      to: { path: '^src/contexts/[^/]+/application/(usecase|port)' },
    },
    {
      name: 'cross-context-facade-only',
      severity: 'error',
      comment: '다른 컨텍스트의 내부(domain/infra/presentation)를 직접 import 금지 — Facade 만 허용',
      from: { path: '^src/contexts/([^/]+)/' },
      to: {
        // $1 = from 에서 잡은 컨텍스트명. 자기 컨텍스트 내부는 pathNot 으로 제외한다.
        path: '^src/contexts/[^/]+/(domain|infrastructure|presentation)',
        pathNot: '^src/contexts/$1/',
      },
    },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    tsConfig: { fileName: 'tsconfig.json' },
  },
};
```

```bash
npx depcruise src --config .dependency-cruiser.js   # CI 필수, 커밋 전 게이트
```

---

## 8. 체크리스트 (커밋 전)

- [ ] 도메인에 프레임워크/ORM import 0건 (`depcruise` 통과)
- [ ] 컨트롤러가 주입받는 것은 Facade 뿐
- [ ] Module `exports` 에 Facade 만
- [ ] 트랜잭션은 Facade 메서드당 1번, UseCase 안에는 없음
- [ ] ORM 엔티티 ↔ 도메인 엔티티 매퍼 존재
- [ ] Facade 반환 타입에 도메인 엔티티 없음
- [ ] 새 포트에 DI 토큰 정의 + Module 배선 완료
- [ ] 도메인 단위 테스트 + UseCase 테스트 + 통합 테스트 존재 ([40-TDD.md](40-TDD.md))
- [ ] 로그에 traceId / userId 포함
