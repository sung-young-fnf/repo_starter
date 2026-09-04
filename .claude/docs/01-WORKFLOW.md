# 워크플로우 — 시작부터 PR 머지까지

`.claude/` 체계를 **실제로 어떤 순서로 쓰는지**를 다룬다. 규칙의 내용은 각 문서에, 규칙을 쓰는
순서는 여기에 있다.

커밋 메시지·PR 형식은 사용자 전역 `~/.claude/CLAUDE.md` 의 Conventional Commits 규칙을 따른다.
이 문서는 **그 규칙을 어느 시점에 적용하는지**만 정한다.

---

# 시나리오 1 — 프로젝트를 처음 시작할 때

준비 3단계(S1~S3)는 **1회성**이고, 코드가 아니라 골조를 세우는 일이다.
S4부터가 실제 개발이고, 그 뒤로는 영원히 시나리오 2를 반복한다.

## S0. 규칙 이식

```bash
cp -r repo_starter/.claude  my-project/
cp    repo_starter/CLAUDE.md my-project/
cd my-project && git init
```

```bash
git add .claude CLAUDE.md
git commit
```

```
chore: DDD/FSD 개발 규칙 문서·에이전트·스킬 도입

규칙이 사람 머릿속에만 있으면 프로젝트가 커질 때 반드시 흔들린다.
레이어 경계·Facade·FSD 의존 방향을 문서로 고정하고, 리뷰 에이전트가
커밋 전에 자동 검사하도록 체계를 먼저 깔았다.

문서 7종 / 에이전트 8종 / 스킬 4종. CLAUDE.md 가 세션 시작 시 자동 로드된다.
```

## S1. 도메인 모델링 — `/domain-model`

폴더 구조를 먼저 만들지 않는다. **경계부터** 긋는다.

```
/domain-model
헬스장 PT 예약 서비스. 회원이 트레이너 시간대를 예약하고,
노쇼하면 패널티가 쌓이고, 트레이너는 정산을 받는다.
(기획서 붙여넣기 또는 파일 경로)
```

- `domain-modeler` 가 이벤트스토밍을 돌린다 (사건 → 커맨드 → 애그리거트 → 정책 → 컨텍스트)
- **되묻는 질문에 반드시 답한다.** 여기서 트랜잭션 경계가 결정된다.
  - "정산의 '회원'과 예약의 '회원'이 같은 대상인가?" → 컨텍스트 분리 여부
  - "정원 초과는 절대 안 되나, 잠깐 초과됐다 조정돼도 되나?" → 즉시 일관성 vs 결과적 일관성
- 산출물: `docs/domain/context-map.md`, `docs/domain/<context>/{ubiquitous-language,aggregates,events}.md`

```bash
git add docs/domain && git commit
```

```
docs: booking·settlement 도메인 모델 정의

같은 '회원'이 예약에서는 노쇼 이력, 정산에서는 사업자 정보를 뜻해
한 모델로 묶으면 양쪽 규칙이 서로를 오염시킨다. 컨텍스트를 둘로 나눴다.

정원 초과는 즉시 일관성이 필요하다는 확인을 받아 Booking 애그리거트
안의 불변식으로 뒀고, 패널티 적립은 결과적 일관성으로 이벤트 분리했다.
```

> **이 문서가 앞으로 모든 이름의 기준이 된다.** 백엔드 컬럼, 프론트 타입, 테스트 이름이 전부
> 여기 식별자를 쓴다.

## S2. 뼈대 생성 — `/ddd-scaffold`

```
/ddd-scaffold
booking 컨텍스트, NestJS. 프론트는 entities/booking 슬라이스도 함께.
```

폴더만이 아니라 **가장 자주 빠지는 것**까지 만든다.

- `booking.module.ts` 배선 + `AppModule` 등록 + `exports: [BookingFacade]`
- `.dependency-cruiser.js` / `.importlinter` 계약
- `test/fake/in-memory-booking.repository.ts`
- **상태 전이표의 모든 칸을 `it.skip` 으로 나열한 테스트** ← 이게 다음 작업 목록이 된다

```bash
git commit   # chore(booking): 컨텍스트 스캐폴딩
```

## S3. 린터 설치 (1회, 생략 금지)

설정 파일은 생성되지만 **패키지는 직접 깐다.** 이걸 건너뛰면 규칙이 프로즈로만 남고 반드시 썩는다.

```bash
npm i -D dependency-cruiser              # NestJS
pip install import-linter                # Python
npm i -D steiger eslint-plugin-boundaries # Next.js
```

`package.json` 에 등록해 두면 리뷰 에이전트가 자동으로 실행한다.

```json
"scripts": {
  "arch": "depcruise src --config .dependency-cruiser.js",
  "fsd": "steiger ./src"
}
```

**CI 에도 넣는다.** 로컬 게이트만으로는 결국 새어 나간다.

```yaml
# .github/workflows/ci.yml
- run: npm run arch
- run: npm run fsd
- run: npm test
```

```bash
git commit   # chore: 아키텍처 린터 도입 및 CI 게이트 추가
```

## S4. 첫 기능 → 시나리오 2로

여기서부터는 시나리오 2와 완전히 같다. 단 두 가지가 다르다.

1. **Phase 1 탐색이 빈손으로 돌아온다.** 참조할 기존 코드가 없다. `docs/domain/` 과 규칙 문서가
   그 역할을 대신한다.
2. **첫 기능이 곧 패턴 레퍼런스가 된다.** 두 번째 `/ddd-plan` 부터 탐색 에이전트 C 가
   "가장 최근 UseCase 전문"으로 물어오는 게 이 코드다. 첫 번째를 대충 짜면 그 형태가 전체에 복제된다.

그래서 **첫 `/ddd-plan` 의 승인 검토를 가장 꼼꼼히** 한다. 특히 이 셋은 앞으로 수십 번 복사된다.

- Facade 메서드 시그니처 형태 — `create(command: XCommand): Promise<XResult>`
- 파일명 규칙 — `.usecase.ts` / `.facade.ts` / `.mapper.ts`
- 테스트 이름 형식 — `<대상>_<상황>_<기대>`

---

# 시나리오 2 — 기능이 추가될 때 (일상 반복)

```
브랜치 → /ddd-plan → 승인 → TDD 구현 → 리뷰 게이트 → 커밋 → 문서 동기화 → PR → 머지
```

## W1. 브랜치를 먼저 판다

```bash
git switch -c feat/booking-cancel
```

> 브랜치 네이밍(`<type>/<context>-<what>`)은 전역 규칙에 없는 제안이다. 팀 컨벤션이 있으면 그쪽을 따른다.
> **main 에서 바로 작업하지 않는다**는 것만 지킨다.

## W2. `/ddd-plan` 실행

```
/ddd-plan
예약 취소 API 추가. 24시간 이내 취소면 패널티 1회 부과.
```

| Phase | 자동으로 벌어지는 일 |
|---|---|
| 1 | 탐색 에이전트 3개 병렬 — ⓐ 컨텍스트 구조·DI 배선 위치 ⓑ 기존 테스트·Fake 현황 ⓒ 최근 UseCase/테스트 전문 |
| 2 | 계획 작성 → `plan-reviewer` 자동 호출 → REJECT 면 스스로 고쳐 재리뷰(최대 2회) → **승인 요청 후 정지** |
| 3 | 테스트 먼저 → domain → application → 배선 → presentation → 프론트 |

## W3. ★ 승인 검토 — 사람이 개입하는 유일한 지점

`plan-reviewer` 가 구조는 걸러 준다. **도메인 지식은 당신만 안다.** 이 세 가지만 봐도 절반이 걸러진다.

| 볼 것 | 이상 신호 |
|---|---|
| **애그리거트 · 불변식** | "24시간 이내" 판단이 UseCase 에 있으면 ❌ → 도메인으로 |
| **트랜잭션** | 예약 취소 + 패널티 = 애그리거트 2개. 이벤트로 나눴는가? |
| **테스트 목록 위치** | 구현 목록보다 아래에 있으면 TDD 가 아니다 |

```
ㅇㅇ 진행해
```
또는
```
패널티는 결과적 일관성으로 가도 돼. BookingCancelled 이벤트로 분리해줘
```

## W4. TDD 구현 (Phase 3 자동 진행)

```
RED    실패 테스트 작성 → 실패를 눈으로 확인 (통과해 버리면 그 테스트는 무의미)
GREEN  domain → application(UseCase → Facade) → infrastructure → presentation → 프론트
       각 단계마다 skip 을 하나씩 떼고 테스트 실행
전체   npm run test && npm run arch    (또는 pytest && lint-imports && mypy)
```

**`skip` 이 하나라도 남아 있으면 끝난 게 아니다.**

## W5. 커밋 전 게이트 — 통과가 커밋 조건

```
커밋 전에 리뷰해줘
```

- 백엔드 변경 → `ddd-reviewer` / 프론트 변경 → `fsd-reviewer`
- 린터를 **실제로 실행한 출력**을 근거로 판정하고 `docs/code-review/<날짜>-<범위>.md` 에 리포트를 남긴다
- 위반이 나오면:

```
ddd-refactor 로 고쳐줘
```

→ R1~R7 레시피대로 수정하고 매 단계 테스트를 돌려 동작이 안 바뀌었는지 확인한다 → **재리뷰**

> **위반이 남은 채로 커밋하지 않는다.** CONCERN/NIT 은 커밋을 막지 않는다.

## W6. 커밋

TDD 사이클이 자연스럽게 커밋 단위를 만든다. 쪼갤수록 리뷰가 쉬워진다.

```bash
git commit   # test(booking): 예약 취소 상태 전이·패널티 테스트 추가
git commit   # feat(booking): 예약 취소 시 24시간 패널티 부과
git commit   # feat(booking): DELETE /bookings/:id 엔드포인트 추가
git commit   # feat(booking): 예약 취소 기능 슬라이스 추가
```

본문에는 **diff 에 안 보이는 것** — 원인·판단 근거·검증 결과를 남긴다.

```
feat(booking): 예약 취소 시 24시간 패널티 부과

취소 정책이 없어 노쇼와 직전 취소가 구분되지 않았고, 트레이너 시간이
비어도 보상 근거가 없었다.

**Booking 애그리거트에 cancel(reason, now) 를 추가해 상태 전이와 패널티
발생 여부를 도메인에서 판단하게 했다.** 패널티 적립은 다른 애그리거트라
BookingCancelled 이벤트로 분리했다 — 한 트랜잭션에 애그리거트 하나 규칙.
동기 처리도 검토했으나 정산 컨텍스트 장애가 예약 취소를 막게 되어 버렸다.

도메인 테스트 6건, 통합 테스트 3건 통과. depcruise 위반 0건.

    ✔ 9 passing (1.2s)
    no dependency violations found (142 modules, 318 dependencies cruised)
```

> **다음 작업자가 오해할 지점**이 있으면 반드시 문장으로 남긴다. squash merge 는 커밋 본문을
> 그대로 머지 커밋에 넣으므로 **커밋 본문 품질이 곧 영구 기록**이다.

## W7. 문서 동기화

```
api-spec 갱신하고 .http 파일도 만들어줘
```

- `api-spec-updater` → `docs/api-spec/booking.md` + `changelog/booking/log_<날짜>.md`
  (breaking change 는 ⚠️ + 영향 범위 + 마이그레이션 방법)
- `http-file-generator` → `http/booking.http` (성공 + 401/404/409 + 상태 전이 실패 케이스)
- 새 용어가 생겼으면 `docs/domain/<context>/ubiquitous-language.md` 갱신

```bash
git commit   # docs: booking API 스펙 및 취소 엔드포인트 .http 갱신
```

## W8. PR

```bash
git push -u origin feat/booking-cancel
gh pr create
```

제목도 Conventional Commits 접두사로 시작한다 (squash merge 시 `(#21)` 자동 부착).

```markdown
feat(booking): 예약 취소 및 24시간 패널티 부과

## 요약
예약 취소 API 를 추가하고, 시작 24시간 이내 취소에 패널티 1회를 부과한다.

## 배경 / 원인
취소 정책이 없어 노쇼와 직전 취소가 구분되지 않았다. 트레이너 시간이 비어도
보상 근거가 없어 정산 문의가 반복됐다.

## 변경사항
- domain: `Booking.cancel(reason, now)` — 상태 전이 + 패널티 발생 판단
- domain: `BookingCancelled` 이벤트 추가
- application: `CancelBookingUseCase`, `BookingFacade.cancel()` (트랜잭션 경계)
- presentation: `DELETE /bookings/:id`
- frontend: `features/cancel-booking` 슬라이스 신규

## 검증
- 도메인 6건 / UseCase 3건 / 통합 3건 통과
- `depcruise` 위반 0건, `steiger` 위반 0건
- 롤백 검증: 패널티 부과 실패 시 예약 상태가 CONFIRMED 로 유지되는지 DB 직접 조회로 확인
- 아키텍처 리뷰: `docs/code-review/2026-09-04-booking.md` (PASS)

## 영향 범위 · 롤백
- 마이그레이션: `1725...-add-penalty.ts` — `down()` 으로 롤백 가능
- 프론트 `entities/booking` 응답 타입에 `cancelledAt` 추가 (하위 호환)

## 리뷰 포인트
- 패널티를 이벤트로 분리한 판단 — 동기 처리 대비 트레이드오프가 적절한지
- 24시간 기준을 도메인 상수로 뒀는데, 정책 테이블로 빼야 할 시점인지
```

`.github/pull_request_template.md` 가 있으면 그 구조를 우선하되 **"배경 / 원인"은 반드시 채운다.**

## W9. 리뷰 반영 → 머지

- 지적 반영 후 **`ddd-reviewer` / `fsd-reviewer` 를 다시 돌린다** (수정하다 경계가 무너지기 쉽다)
- CI 의 `arch` / `fsd` / `test` 가 초록인지 확인
- **squash merge**

## W10. 마무리

- `TASKS.md` 반영 (전역 규칙 형식 준수)

```
- [x] [booking] 예약 취소 API + 24시간 패널티 (2026-09-04)
- [ ] [booking] 패널티 정책 테이블화
```

- 브랜치 정리: `git branch -d feat/booking-cancel`

---

# 언제 이 워크플로우를 쓰지 않나

오타 수정, 로그 문구 변경, 상수값 조정, 스타일 — `/ddd-plan` 을 돌리지 않는다. 그냥 고치고
`fix:` / `style:` 로 커밋한다.

**비즈니스 규칙이 바뀌거나, 엔드포인트·슬라이스가 새로 생길 때**만 쓰는 도구다.

# 한눈에

```
[시나리오 1 · 1회성]
  S0 규칙 이식          →  chore: 규칙 도입
  S1 /domain-model      →  docs: 도메인 모델 정의
  S2 /ddd-scaffold      →  chore: 컨텍스트 스캐폴딩
  S3 린터 + CI          →  chore: 아키텍처 게이트 추가
  S4 ────────────────────┐
                         │
[시나리오 2 · 무한 반복] ↓
  W1 브랜치
  W2 /ddd-plan          →  탐색 → 계획 → plan-reviewer
  W3 ★ 승인 검토         →  사람이 개입하는 유일한 지점
  W4 TDD 구현           →  RED → GREEN → 전체 초록
  W5 리뷰 게이트         →  ddd-reviewer / fsd-reviewer → ddd-refactor
  W6 커밋               →  Conventional Commits + 원인 본문
  W7 문서 동기화         →  api-spec-updater / http-file-generator
  W8 PR                 →  요약·배경/원인·변경사항·검증·영향/롤백·리뷰포인트
  W9 반영 → 재리뷰 → squash merge
  W10 TASKS.md
```
