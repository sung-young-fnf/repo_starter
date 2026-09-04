# DDD 공통 규칙 (언어 무관)

NestJS·FastAPI 어느 쪽이든 **이 문서의 규칙이 상위**다. 언어별 문서(20/21)는 이 규칙을 그 언어로
어떻게 쓰는지만 다룬다.

---

## 0. 언제 DDD를 쓰고, 언제 쓰지 않나

DDD는 **비즈니스 규칙이 복잡한 곳**에 쓰는 도구다. 전부에 바르면 비용만 든다.

| 상황 | 선택 |
|---|---|
| 상태 전이·정책·불변식이 있다 (예약, 정산, 주문, 권한) | **DDD 풀세트** |
| 사실상 CRUD + 조회 (설정값, 코드 테이블, 관리자 목록) | 얇은 서비스 + 리포지토리. 애그리거트 만들지 말 것 |
| 읽기 전용 대시보드·리포트 | **CQRS 의 Q 쪽만** — 도메인 모델 거치지 말고 전용 read model / SQL 직행 |

> 판단이 애매하면 계획 단계에서 `plan-reviewer` 에게 "이 컨텍스트는 DDD 과설계 아닌가"를 명시적으로 묻는다.

---

## 1. 전략적 설계 (Strategic Design)

코드를 쓰기 전에 **경계부터** 긋는다. 이걸 건너뛰고 폴더만 DDD처럼 만드는 게 가장 흔한 실패다.

### 1.1 서브도메인 분류

| 종류 | 뜻 | 투자 |
|---|---|---|
| Core (핵심) | 이 제품이 돈 버는 이유 | 최고 품질. DDD 풀세트 |
| Supporting (지원) | 필요하지만 차별점은 아님 | 단순하게 |
| Generic (일반) | 인증, 결제, 알림 등 | **사서 쓴다.** 직접 만들지 않는다 |

### 1.2 바운디드 컨텍스트

- 하나의 컨텍스트 = **하나의 유비쿼터스 언어가 일관되게 통하는 범위**.
- 같은 단어가 다른 뜻이면 **컨텍스트를 나눠야 한다는 신호**다.
  - 예: 예약 컨텍스트의 `Customer`(연락처·노쇼 이력) ≠ 정산 컨텍스트의 `Customer`(사업자번호·세금 정보).
    같은 테이블을 보더라도 **모델은 각자 갖는다.**
- 컨텍스트는 폴더/모듈/패키지로 **물리적으로 분리**한다. `src/contexts/<context>/`.

### 1.3 컨텍스트 맵 — 관계를 명시한다

`docs/domain/context-map.md` 에 표로 남긴다.

| 관계 | 의미 | 구현 |
|---|---|---|
| Customer–Supplier | 하류가 상류에 요구할 수 있음 | 상류 Facade 직접 호출 |
| Conformist | 상류 모델을 그대로 수용 | 상류 DTO 그대로 사용 |
| **ACL** (Anti-Corruption Layer) | 외부/레거시 모델이 우리 모델을 오염시키지 못하게 차단 | `infrastructure/acl/` 에 번역기 |
| Published Language | 이벤트/스키마로 계약 | 도메인 이벤트 + 스키마 버저닝 |

> **외부 API(결제사, 메시지, 레거시 DB)는 예외 없이 ACL을 통과시킨다.** 외부 응답 타입이 도메인
> 레이어에 등장하는 순간 그 도메인은 외부 스펙에 종속된다.

### 1.4 유비쿼터스 언어

- `docs/domain/<context>/ubiquitous-language.md` 에 **용어 표**를 유지한다.
- 컬럼: `한국어 용어 | 코드 식별자 | 정의 | 아닌 것(혼동 주의) | 상태값`
- **코드·테스트·API·프론트 폴더명이 전부 이 표의 식별자를 쓴다.** 번역이 끼어드는 순간 지식이 샌다.
- 기획서에 없는 단어를 코드가 발명하면(`XxxManager`, `XxxProcessor`, `data`, `info`) 그건 모델링이
  덜 끝났다는 뜻이다.

---

## 2. 전술적 설계 (Tactical Design)

### 2.1 빌딩 블록

| 블록 | 정체성 | 규칙 |
|---|---|---|
| **Value Object** | 값이 곧 정체성 | **불변**. 생성자에서 검증. 동등성은 값 비교. 되도록 VO를 먼저 만든다 |
| **Entity** | ID로 식별 | 상태 변경은 **의도를 드러내는 메서드**로만 (`confirm()`, `cancel(reason)`) |
| **Aggregate** | 일관성 경계 | 루트를 통해서만 내부 접근. 트랜잭션 단위 |
| **Repository** | 애그리거트 단위 저장소 | 인터페이스는 **도메인**에, 구현은 **인프라**에. 애그리거트당 1개 |
| **Domain Service** | 여러 애그리거트에 걸친 순수 규칙 | 상태를 갖지 않음. **여기에 트랜잭션·IO를 넣지 않는다** |
| **Domain Event** | 도메인에서 일어난 사실 | **과거형** 이름. 페이로드는 ID + 최소 정보 + 발생 시각 |
| **Factory** | 복잡한 생성 | 생성 규칙이 3줄 넘으면 분리 |
| **Specification** | 조합 가능한 조건 | 조회 조건이 여기저기 복붙될 때 도입 |

### 2.2 애그리거트 설계 4대 규칙

1. **진짜 불변식만 경계 안에 넣는다.** "예약 정원을 초과할 수 없다" 같은 *반드시 즉시 일관되어야 하는*
   규칙만. "통계가 맞아야 한다"는 불변식이 아니다.
2. **작게 설계한다.** 애그리거트가 컬렉션 수천 건을 들고 있으면 잘못 그은 것이다.
3. **다른 애그리거트는 ID로만 참조한다.** 객체 참조 금지.
   - ❌ `booking.customer.name`  → ✅ `booking.customerId` + 필요 시 조회 모델
4. **경계 밖은 결과적 일관성.** 한 트랜잭션에 애그리거트 하나. 나머지는 **도메인 이벤트**로 전파한다.

> **트랜잭션 1개 = 애그리거트 1개.** 예외는 주석으로 이유를 남기고 `plan-reviewer` 승인을 받는다.

### 2.3 도메인 이벤트

- 이름: `BookingConfirmed`, `PaymentFailed` — **과거형**, 컨텍스트의 언어로.
- 페이로드: 애그리거트 ID, 발생 시각, 소비자가 재조회 없이 판단할 최소 필드. 엔티티 통째로 싣지 않는다.
- 발행: 애그리거트가 이벤트를 **모아 두고**, 저장 성공 후 디스패치한다. 실패해도 안 나가야 한다.
- 같은 프로세스 내 소비는 동기, 컨텍스트 경계를 넘으면 아웃박스/큐를 검토한다.

---

## 3. 레이어와 의존 방향

```
        presentation  (controller / router / http DTO)
              │  호출
              ▼
        application   (UseCase · Facade · Command/Query · outbound Port)
              │  호출
              ▼
           domain     (Entity · VO · Aggregate · Domain Service · Repository 인터페이스 · Event)
              ▲
              │ 구현 (의존성 역전)
        infrastructure (ORM · Repository 구현 · 외부 API 어댑터 · ACL)
```

### 3.1 허용 / 금지 표

| From ↓ / To → | domain | application | infrastructure | presentation |
|---|---|---|---|---|
| **domain** | ✅ | ❌ | ❌ | ❌ |
| **application** | ✅ | ✅ | ❌ (포트 인터페이스로만) | ❌ |
| **infrastructure** | ✅ | ✅ (포트 구현 목적) | ✅ | ❌ |
| **presentation** | ❌ (**Facade만**) | ✅ Facade만 | ❌ | ✅ |

- **presentation → application 은 Facade 한 점으로만 들어간다.** 컨트롤러가 UseCase·리포지토리·
  도메인 서비스를 직접 호출하면 위반.
- **컨텍스트 A → 컨텍스트 B 도 Facade 또는 이벤트로만.** B의 리포지토리·엔티티를 A가 import 하면 위반.
- 구현체 배선은 **컴포지션 루트**(NestJS Module / FastAPI 컨테이너) 한 곳에서만 한다.

### 3.2 Facade 규칙 (백엔드 필수 패턴)

Facade = **바운디드 컨텍스트의 공개 API**. 하나의 컨텍스트에 원칙적으로 하나.

**Facade가 하는 일**
- 컨텍스트 바깥(컨트롤러, 다른 컨텍스트)에 노출할 오퍼레이션을 **유스케이스 언어로** 정의
- 여러 UseCase 조합 / 순서 제어
- **트랜잭션 경계**를 연다 (UnitOfWork)
- 도메인 예외 → 애플리케이션 결과 타입으로 변환

**Facade가 하지 않는 일**
- ❌ 비즈니스 규칙 판단 (→ 도메인)
- ❌ ORM·HTTP·큐 직접 호출 (→ 포트)
- ❌ HTTP 상태 코드·요청 객체 취급 (→ presentation)
- ❌ 다른 컨텍스트의 내부 접근

> Facade가 200줄을 넘거나 `if` 로 비즈니스 분기를 하고 있으면 로직이 새어 들어온 것이다.

---

## 4. 폴더 구조 (공통 형태)

```
src/contexts/<context>/
├── domain/
│   ├── model/          # 엔티티, VO, 애그리거트, 도메인 이벤트
│   ├── repository/     # 리포지토리 "인터페이스"
│   ├── service/        # 도메인 서비스
│   └── error/          # 도메인 예외
├── application/
│   ├── <context>.facade.*      # ★ 유일한 진입점
│   ├── usecase/
│   ├── dto/            # Command / Query / Result (도메인 타입 노출 금지)
│   └── port/           # 아웃바운드 포트 인터페이스 (알림, 스토리지, 트랜잭션)
├── infrastructure/
│   ├── persistence/    # ORM 모델, 리포지토리 구현, 매퍼
│   ├── external/       # 외부 API 어댑터
│   └── acl/            # 외부 모델 → 도메인 모델 번역
└── presentation/
    └── http/           # 컨트롤러/라우터, 요청·응답 DTO
```

- `shared/kernel/` (공유 커널)은 **정말 모든 컨텍스트가 같은 뜻으로 쓰는 것만** — `Money`, `Email`,
  `DateRange` 정도. 여기가 부풀면 컨텍스트 분리가 실패한 것이다.

---

## 5. 금지 목록 (리뷰에서 즉시 지적)

| # | 금지 | 왜 |
|---|---|---|
| 1 | 도메인 레이어에 프레임워크 import (`@nestjs/*`, `sqlalchemy`, `pydantic`, ORM 데코레이터) | 도메인이 프레임워크 수명에 묶인다 |
| 2 | ORM 모델을 도메인 엔티티로 겸용 | 스키마 변경이 도메인 규칙을 흔든다. **매퍼로 분리** |
| 3 | 컨트롤러 → UseCase/리포지토리/도메인 서비스 직접 호출 | Facade 경계 붕괴 |
| 4 | setter 만 있는 엔티티 + 서비스에 로직 (빈약한 도메인 모델) | DDD 폴더만 흉내 낸 트랜잭션 스크립트 |
| 5 | 리포지토리에 비즈니스 로직 (`findAndApproveExpired()`) | 규칙이 SQL 속으로 숨는다 |
| 6 | 도메인 서비스에서 IO/트랜잭션 수행 | 도메인이 테스트 불가능해진다 |
| 7 | 애그리거트 간 객체 참조 | 경계·트랜잭션이 무너진다 |
| 8 | Facade가 도메인 엔티티를 그대로 반환 | 내부 모델이 외부 계약이 된다. **Result DTO로 변환** |
| 9 | 한 트랜잭션에서 애그리거트 2개 이상 수정 | 잠금 경합·부분 실패 |
| 10 | 다른 컨텍스트의 테이블 직접 조회 | 컨텍스트 경계 무시. Facade/이벤트/read model 사용 |
| 11 | `Manager`, `Processor`, `Util`, `Helper`, `data`, `info` 네이밍 | 유비쿼터스 언어 부재의 증상 |
| 12 | 로그·에러에 상관관계 컨텍스트(traceId/requestId) 누락 | 프로덕션에서 추적 불가 |

---

## 6. 에러 처리

- **도메인 예외**는 도메인 언어로 (`BookingAlreadyCancelled`), HTTP를 모른다.
- **application** 에서 도메인 예외를 결과/애플리케이션 예외로 변환.
- **presentation** 에서만 HTTP 상태 코드로 매핑. 매핑 표를 컨텍스트별로 한 곳에 모은다.
- 예상 가능한 실패(정책 위반)와 버그(불변식 깨짐)를 타입으로 구분한다.

## 7. 로깅 규칙

- 모든 로그는 **요청 컨텍스트(traceId·userId)를 실어서** 남긴다. 컨텍스트 없는 로그는 리뷰에서 지적한다.
- 도메인 레이어는 로깅하지 않는다. 로깅은 application/infrastructure의 관심사다.
- 로그 메시지는 유비쿼터스 언어를 쓴다.

---

## 8. 규칙을 기계로 강제하기

프로즈 규칙은 반드시 썩는다. **CI에서 의존 방향을 깨면 빌드가 실패해야 한다.**

- TypeScript: `dependency-cruiser` (설정 예시 → [20-BACKEND-NESTJS.md](20-BACKEND-NESTJS.md) §7)
- Python: `import-linter` (설정 예시 → [21-BACKEND-PYTHON.md](21-BACKEND-PYTHON.md) §7)
- 프론트: `steiger` + `eslint-plugin-boundaries` (→ [30-FRONTEND-FSD.md](30-FRONTEND-FSD.md) §6)

`ddd-reviewer` 는 이 린터를 **실제로 실행해서** 근거로 삼는다. 눈으로만 본 리뷰는 근거가 아니다.
