---
name: ddd-plan
description: DDD + TDD 로 기능을 계획하고 구현하는 3단계 워크플로우. 병렬 탐색 → 계획 수립 + plan-reviewer 자동 리뷰 → 테스트 먼저 구현. 새 API·화면·기능 구현의 진입점으로 사용한다.
---

# /ddd-plan — DDD + TDD 구현 워크플로우

**모든 기능 구현은 여기서 시작한다.** 코드를 바로 쓰지 않는다.

사용자가 무엇을 만들지 아직 말하지 않았다면, **먼저 무엇을 만들지 물어보고 멈춘다.**

---

## Phase 0. 규칙 로드

다음을 읽는다 (해당하는 것만).

- `.claude/docs/10-DDD-CORE.md` — 항상
- `.claude/docs/50-PLAN-CHECKLIST.md` — 항상
- 백엔드면 `20-BACKEND-NESTJS.md` 또는 `21-BACKEND-PYTHON.md`
- 프론트면 `30-FRONTEND-FSD.md`
- `.claude/docs/40-TDD.md` — 항상
- `docs/domain/` 이 있으면 해당 컨텍스트의 유비쿼터스 언어·애그리거트 문서

> 도메인 자체가 새롭거나 모호하면 여기서 멈추고 **`/domain-model` 을 먼저 돌리자고 제안**한다.

---

## Phase 1. 병렬 탐색

Agent 도구로 탐색 에이전트 **3개를 한 메시지에서 동시에** 띄운다. (`subagent_type: "Explore"`)

**A. 코드 탐색**
> 대상 컨텍스트 `<context>` 의 현재 구조를 조사하라. 다음을 파일 경로와 함께 보고:
> Facade 메서드 목록 / UseCase 목록 / 도메인 엔티티·VO·상태 전이 / 리포지토리 인터페이스와 구현 /
> 포트와 DI 배선 위치(Module 또는 container) / 컨트롤러 엔드포인트 / ORM 모델과 매퍼.
> 프론트면: 관련 슬라이스(entities/features/widgets/views), 각 `index.ts` 공개 API, 쿼리 키 정의 위치.
> **코드를 수정하지 말고 사실만 보고하라.**

**B. 테스트 현황**
> `<context>` 관련 기존 테스트를 조사하라. 도메인 단위 / UseCase / 통합 / 프론트 테스트가
> 각각 어디에 어떤 이름으로 있는지, Fake 구현체(`InMemory*Repository` 등)가 이미 있는지,
> 테스트 실행 명령이 무엇인지(package.json scripts / pytest 설정) 보고하라.

**C. 패턴 레퍼런스**
> 이 저장소에서 **가장 최근에 추가된** 유스케이스 하나와 테스트 하나를 찾아 전문을 보고하라.
> 파일명 규칙, import 순서, 에러 처리 방식, 테스트 구조(픽스처·헬퍼·정리)를 그대로 인용하라.
> 새 코드가 따라야 할 본보기를 찾는 것이 목적이다.

세 결과를 받아 **현재 구조 요약**을 짧게 정리한다.

---

## Phase 2. 계획 수립 → 자동 리뷰 → 승인

### 2-1. 계획 작성

`50-PLAN-CHECKLIST.md` 의 A~F 항목에 **답을 문장으로** 쓴다. 형식:

```markdown
## 무엇을 만드나
<한 문단>

## 도메인 설계
- 컨텍스트: <기존/신규 + 근거>
- 애그리거트: <이름> — 불변식: <목록>
- 상태 전이: <표>
- 새 용어: <유비쿼터스 언어에 추가할 항목>
- 도메인 이벤트: <이름(과거형) — 소비자>
- 트랜잭션: <한 트랜잭션에 애그리거트 하나인가? 아니면 어떻게 나누나>

## 백엔드 변경 (레이어별)
| 레이어 | 파일 | 내용 |
|---|---|---|
| domain | src/contexts/booking/domain/model/booking.ts | confirm() 추가 |
| application | .../booking.facade.ts | confirm(command) — tx 경계 |
| ... |

- Facade 시그니처: `confirm(command: ConfirmBookingCommand): Promise<BookingResult>`
- 새 포트: <인터페이스 위치 / DI 토큰 / 구현체 / **배선 지점 파일명**>
- 마이그레이션: <파일 / 롤백 방법>
- 에러 → HTTP 매핑: <표>

## 프론트 변경 (FSD 레이어별)
| 레이어 | 슬라이스 | 내용 |
|---|---|---|
| entities | booking | status 전이 규칙 추가, mapper 갱신 |
| features | confirm-booking | 신규 — mutation + 낙관적 업데이트 |

- 공개 API(`index.ts`)에서 내보낼 것:
- 무효화할 쿼리 키:
- `'use client'` 경계:

## 테스트 계획 (구현보다 먼저 온다)
1. [도메인] `confirm_PENDING에서_CONFIRMED로 전이한다`
2. [도메인] `confirm_CANCELLED에서_BookingTransitionNotAllowed를 던진다`
3. [UseCase] `confirm_존재하지 않는 예약이면_BookingNotFound`  (InMemory 리포지토리)
4. [통합] `POST /bookings/{id}/confirm` 200 + DB 에서 status=CONFIRMED 확인
5. [통합] 이미 취소된 예약 → 409 + DB 상태 불변 확인
6. [프론트] `features/confirm-booking` — 성공 시 목록 갱신 (MSW)

## 실행 순서
1. 실패 테스트 1~2 작성 (skip 포함) → 실패 확인
2. 도메인 구현 → 1~2 통과
3. UseCase + Facade + 배선 → 3 통과
4. 컨트롤러 + 통합 테스트 → 4~5 통과
5. 프론트 → 6 통과
6. 린터(depcruise/lint-imports/steiger) + 전체 테스트

## 운영
- 로그(traceId), 권한, 동시성(락/유니크), N+1 여부, 롤백 방법
```

### 2-2. plan-reviewer 자동 호출

계획을 쓴 **직후** Agent 도구로 `plan-reviewer` 를 호출한다. 사용자가 시키지 않아도 호출한다.

전달할 것: 계획 전문 + Phase 1 탐색 결과 요약 + 대상 파일 경로.

- **REJECT** → 지적을 반영해 계획을 고치고 **다시 호출**한다. (최대 2회. 3회째도 REJECT 면
  쟁점을 사용자에게 그대로 보여 주고 판단을 요청한다.)
- **APPROVE WITH CHANGES** → 변경을 반영하고 다음으로 간다.
- **APPROVE** → 다음으로 간다.

### 2-3. 사용자 승인

계획과 리뷰 결과를 함께 제시하고 **승인을 기다린다.** 승인 없이 Phase 3 으로 넘어가지 않는다.

---

## Phase 3. TDD 실행

### Step 1 — 테스트를 먼저 쓴다
- 계획의 테스트 목록을 **전부** 작성한다. 아직 구현이 없으니 skip 을 붙여 커밋 가능한 상태로 둔다
  (`it.skip` / `@pytest.mark.skip`).
- 첫 1~2개는 skip 을 떼고 **실행해서 실패를 눈으로 확인**한다.
  → 실패하지 않으면 그 테스트는 아무것도 검증하지 않는 것이다. 다시 쓴다.
- Fake 구현체가 필요하면 여기서 만든다 (목이 아니라 인메모리 구현체 — `40-TDD.md` §2).

### Step 2 — 구현
- **안에서 바깥으로**: domain → application(UseCase → Facade) → infrastructure → presentation → 프론트.
- 한 단계 끝날 때마다 해당 skip 을 떼고 테스트를 돌린다.
- 도메인이 `new Date()`/`uuid()` 를 부르고 싶어지면 **주입으로 바꾼다.**
- DI 배선(Module `providers`/`exports`, `container.py`)을 **잊지 않는다** — 가장 자주 빠지는 항목이다.

### Step 3 — 전부 초록으로
```bash
# NestJS
npm run test:unit && npm run test:integration
npx depcruise src --config .dependency-cruiser.js

# Python
pytest -m "not e2e" && lint-imports && mypy app

# Next.js
npm run test && npx steiger ./src && npm run lint
```
- 통합 테스트는 **실제 DB 가 떠 있어야** 한다. 서버/DB 기동 방법을 확인하고 안내한다.
- 전부 통과할 때까지 반복한다. **skip 이 하나라도 남아 있으면 끝난 게 아니다.**

### Step 4 — 리뷰 게이트
- 백엔드 변경 → `ddd-reviewer`
- 프론트 변경 → `fsd-reviewer`
- 위반이 나오면 `ddd-refactor` 로 고치고 **재리뷰**한다.

### Step 5 — 문서 동기화
- API 가 바뀌었으면 `api-spec-updater`
- 엔드포인트가 추가/변경됐으면 `http-file-generator`
- 새 용어가 생겼으면 `docs/domain/<context>/ubiquitous-language.md` 갱신

### Step 6 — 마무리
- 작업이 끝났으면 사용자에게 **`TASKS.md` 반영 여부를 묻는다** (전역 규칙).
- 커밋은 사용자가 요청할 때만 한다.

---

## 이 워크플로우에서 하지 않는 것

- 계획 승인 전에 프로덕션 코드 쓰기
- 테스트 없이 구현하기
- 테스트를 느슨하게 고쳐서 통과시키기
- 계획에 없던 리팩터링을 끼워 넣기 (발견하면 **보고하고 별도 작업으로 남긴다**)
