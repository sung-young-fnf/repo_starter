---
name: write-integration-test
description: 실제 DB·실제 HTTP 를 경유하는 통합 테스트를 프로젝트 컨벤션대로 작성한다. NestJS(supertest)와 FastAPI(httpx) 양쪽 구조, 데이터 정리, DB 직접 조회 검증, 실패 경로·트랜잭션 롤백 검증을 강제한다.
---

# /write-integration-test — 통합 테스트 작성

목의 통합 테스트는 통합 테스트가 아니다. **실제 DB, 실제 라우팅, 실제 트랜잭션**을 통과시킨다.

## 시작 전

1. `.claude/docs/40-TDD.md` §5 를 읽는다.
2. **저장소의 가장 최근 통합 테스트를 찾아 전문을 읽는다.** 그 형식이 이 문서보다 우선이다.
3. 대상 컨트롤러/라우터와 DTO 를 읽어 엔드포인트·상태코드·에러 코드를 확인한다.
4. 테스트 DB 기동 방법을 확인한다 (docker-compose / testcontainers / 로컬 DB, 환경변수).

---

## 필수 요건 (빠지면 통합 테스트가 아니다)

- [ ] **실제 DB** 를 쓴다. 리포지토리를 Fake 로 바꾸지 않는다.
- [ ] **HTTP 를 경유**한다 (supertest / httpx). Facade 직접 호출은 UseCase 테스트다.
- [ ] **DB 를 직접 조회**해 저장 상태를 검증한다. API 응답만 믿지 않는다.
- [ ] **데이터 정리**가 보장된다 (트랜잭션 롤백 또는 명시적 cleanup).
- [ ] **실패 경로**가 있다: 400 / 401 / 403 / 404 / 409.
- [ ] **트랜잭션 롤백 검증**이 있다: 중간에 실패하면 아무것도 저장되지 않았는지 DB 로 확인.
- [ ] 테스트 이름이 **유비쿼터스 언어**로 되어 있다.

---

## A. NestJS

```
test/integration/<context>.e2e-spec.ts
```

구조

```ts
describe('Booking API (integration)', () => {
  let app: INestApplication;
  let db: DataSource;              // 또는 PrismaClient — DB 직접 조회용
  let token: string;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
      // 외부 시스템(결제·메일)만 스텁으로 대체. 리포지토리는 절대 대체하지 않는다.
      .overrideProvider(NOTIFICATION).useClass(StubNotificationAdapter)
      .compile();

    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ transform: true }));  // 프로덕션과 동일하게
    await app.init();

    db = moduleRef.get(DataSource);
    token = await issueTestToken(app, { role: 'member' });
  });

  afterEach(async () => { await cleanupBookings(db); });   // 정리 보장
  afterAll(async () => { await app.close(); });

  describe('POST /bookings', () => {
    it('유효한 요청이면_예약이 PENDING 으로 저장된다', async () => {
      const res = await request(app.getHttpServer())
        .post('/bookings')
        .set('Authorization', `Bearer ${token}`)
        .send({ customerId, start, end })
        .expect(201);

      // 응답만이 아니라 DB 를 본다
      const row = await db.query('SELECT status FROM bookings WHERE id = $1', [res.body.id]);
      expect(row[0].status).toBe('PENDING');
    });

    it('시작 시각이 과거면_400 BOOKING_IN_PAST 이고_아무것도 저장되지 않는다', async () => {
      await request(app.getHttpServer()).post('/bookings')
        .set('Authorization', `Bearer ${token}`)
        .send({ customerId, start: pastDate, end: pastDate })
        .expect(400)
        .expect((r) => expect(r.body.code).toBe('BOOKING_IN_PAST'));

      const rows = await db.query('SELECT * FROM bookings WHERE customer_id = $1', [customerId]);
      expect(rows).toHaveLength(0);          // 롤백 검증
    });

    it('같은 슬롯에 확정 예약이 있으면_409 를 반환한다', async () => { /* … */ });
    it('토큰이 없으면_401 을 반환한다', async () => { /* … */ });
  });
});
```

- 헬퍼는 `helperCreateBooking` / `cleanupBookings` 처럼 **의도가 드러나는 이름**으로 파일 하단에 모은다.
- 실행: `npm run test:integration` (없으면 `package.json` 에 스크립트를 추가 제안한다)

---

## B. FastAPI

```
tests/integration/test_<context>_api.py
```

```python
@pytest_asyncio.fixture
async def client(app, db_session) -> AsyncGenerator[AsyncClient, None]:
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c


@pytest.mark.asyncio
async def test_예약_생성_유효한_요청이면_PENDING으로_저장된다(client, db, auth_headers):
    res = await client.post("/bookings", json={...}, headers=auth_headers)
    assert res.status_code == 201

    row = await db.execute(
        text("SELECT status FROM bookings WHERE id = :id"), {"id": res.json()["id"]}
    )
    assert row.scalar_one() == "PENDING"          # DB 직접 검증


@pytest.mark.asyncio
async def test_예약_생성_시작시각이_과거면_400이고_저장되지_않는다(client, db, auth_headers):
    res = await client.post("/bookings", json={...past...}, headers=auth_headers)
    assert res.status_code == 400
    assert res.json()["code"] == "BOOKING_IN_PAST"

    count = await db.execute(text("SELECT count(*) FROM bookings"))
    assert count.scalar_one() == 0                 # 롤백 검증
```

- 정리는 `conftest.py` 에서 **트랜잭션 롤백 픽스처**로 처리하는 것을 기본으로 한다.
  (테스트마다 트랜잭션을 열고 끝나면 롤백 → 순서 의존이 사라진다)
- 외부 시스템만 `dependency_overrides` 로 스텁 교체한다. **세션·리포지토리는 실물 유지.**
- 실행: `pytest tests/integration -v`

---

## 반드시 넣는 시나리오

| 종류 | 예 |
|---|---|
| 정상 생성 | 201 + DB 저장 확인 |
| 목록/상세 조회 | 200 + 필드 매핑 확인 |
| 상태 전이 성공 | 확정 → DB status 변경 확인 |
| **상태 전이 실패** | 취소된 예약 확정 → 409 + **DB 상태 불변** 확인 |
| 형식 오류 | 400 + 에러 코드 |
| 인증/권한 | 401 / 403 |
| 없는 리소스 | 404 |
| 중복 | 409 |
| **트랜잭션 롤백** | 중간 실패 → 어떤 행도 남지 않음 |
| 동시성(해당 시) | 같은 슬롯 동시 요청 → 하나만 성공 |

---

## 마무리

1. 테스트를 **실행해서** 통과를 확인한다. 못 돌리면 그 사실과 필요한 준비(DB 기동 등)를 보고한다.
2. `skip` / `only` 가 남아 있지 않은지 확인한다.
3. `TASKS.md` 반영 여부를 사용자에게 묻는다.

## 하지 않는 것

- 리포지토리를 목/Fake 로 대체하기
- 응답 JSON 만 검증하고 DB 를 보지 않기
- 성공 경로만 쓰기
- 테스트 간 데이터 공유에 의존해 순서를 강제하기 (불가피하면 이유를 주석으로 남긴다)
- 프로덕션 DB 를 향해 테스트 실행 — 연결 문자열을 반드시 확인한다
