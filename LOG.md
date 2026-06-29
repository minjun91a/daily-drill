# 검증 드릴 오답 로그

> AI가 "자신 있게 틀리는" 패턴을 모은다. mint-agent trust layer의 검증 규칙 후보이자 포트폴리오 자산.

**형식(항목마다 고정):** 문제(코드) → 내 예측(그대로) → 실제 → 교정(고친 코드) → 교훈. **틀린 것만** 기록한다. 예측은 *원문 그대로* 보존한다.

---

## 2026-06-29 — SQL · 유저별 총 결제액 쿼리 리뷰

> ⚠️ **이 항목은 이전 세션에서 진행되어 원본 SQL을 보유하지 않음.** 아래 **문제 코드는 기록된 오답·교훈에 맞춘 재구성본**이고, "내 예측/실제/교훈"은 당시 기록 원문이다. 교정은 당시 미수행(진단 단계까지만 하던 시기) → *참고용 정답 fix*로 사후 보강.

**문제 (재구성본)**
```sql
-- 유저별 총 결제액: 결제 완료('done') 주문 금액 합
SELECT u.name, SUM(o.amount) AS total
FROM users u
JOIN orders   o ON o.user_id  = u.id
JOIN payments p ON p.order_id = o.id     -- 주문 1 : 결제 N
WHERE o.status = 'done'
GROUP BY u.name;
```

### 오답 1 — JOIN 행 복제 ≠ WHERE 필터
- **내 예측(그대로)**: "`status = 'done'` 조건을 걸었기 때문에 금액이 실제보다 크게 나올 수 없다."
- **실제**: 크게 나올 수 있다. 원인은 주문 필터(WHERE)가 아니라 **JOIN이 행을 복제**하는 것. 한 주문에 결제가 2건이면 그 주문의 `o.amount`가 2번 더해진다. status 필터는 *어떤 주문을 포함할지*만 거를 뿐, 행 복제를 막지 못한다.
- **교정 (참고 — 당시엔 진단까지만)**:
  ```sql
  SELECT u.id, u.name, SUM(o.amount) AS total
  FROM users u
  JOIN orders o ON o.user_id = u.id
  WHERE o.status = 'done'
    AND EXISTS (SELECT 1 FROM payments p WHERE p.order_id = o.id)  -- 복제 없이 '결제 있음'만 필터
  GROUP BY u.id, u.name;                                            -- 식별자로 묶기
  ```
- **교훈**: JOIN을 거칠 때마다 "이 JOIN이 행을 늘리나?"를 먼저 묻는다. '결제 있음' 필터는 행을 늘리는 JOIN 대신 `EXISTS`로, 또는 결제를 주문 단위로 먼저 집계. WHERE로는 못 막는다.

### 오답 2 — 집계 GROUP BY는 식별자(id)로
- **내 예측(그대로)**: "SELECT 컬럼이 2개라서 GROUP BY에 두 컬럼 모두 써야 한다."
- **실제**: SUM은 집계라 GROUP BY에 안 넣어도 문법은 통과한다. 진짜 위험은 **name이 유저를 유일하게 식별하지 못한다**는 것 — 동명이인이 있으면 서로 다른 사람의 결제액이 한 줄로 합쳐진다.
- **교정 (고친 코드)**: `GROUP BY u.id, u.name` (식별은 이름이 아니라 id, 표시용으로 name 동반).
- **교훈**: 사람을 식별하는 건 이름이 아니라 id.

### 잘한 것
- INNER JOIN이라 결제 없는 'done' 주문은 결과에서 빠진다는 것을 정확히 예측함. (모두 살리려면 LEFT JOIN → amount NULL)

---

## 2026-06-29 (복기) — 틀린 개념을 다른 유형으로 재출제, 합격까지

### 복기 2 — 집계는 식별자로 GROUP BY  ·  **1회 합격**
**문제**
```sql
SELECT b.name AS brand, SUM(p.stock) AS total_stock
FROM products p
JOIN brands b ON b.id = p.brand_id
GROUP BY b.name;
```
- **내 예측(그대로)**: "브랜드별 재고를 정확히 보여줄 수 없다. 중복된 브랜드명이면 그 재고 수량을 합쳐서 보여주게 되기 때문에 group by 를 name 이 아닌 각 name 별 id 로 group by 를 해야 정확한 재고 파악이 된다."
- **실제**: 정확. 서로 다른 `brand_id`가 같은 이름을 가지면 한 줄로 합쳐져 잘못된 합계가 된다.
- **교정 (고친 코드)**: `GROUP BY b.id, b.name`.
- **교훈**: 묶는 기준은 라벨(name)이 아니라 식별자(id). ✅

### 복기 1 — JOIN 행 복제  ·  **3회 만에 합격** (변형 2개)

**변형 A (1차)**
```sql
SELECT p.name, SUM(s.amount) AS revenue
FROM products p
JOIN promotions pr ON pr.product_id = p.id   -- 상품 1 : 프로모션 N
JOIN sales      s  ON s.product_id  = p.id   -- 상품 1 : 판매 N
WHERE p.in_stock = TRUE
GROUP BY p.id, p.name;
```
- **내 예측(그대로)**: "revenue는 실제 매출과 일치하다. 그 이유는 각 프로모션 별 각 매출을 행에 추가되서 출력이 되지만, 조건부가 true 값만을 부르기 때문에 실제 매출만을 출력 되기 때문이다." → ❌ **오답 1과 같은 함정 반복** (WHERE가 복제를 막아준다고 오판).

**변형 B (2~3차)**
```sql
SELECT c.name, SUM(o.amount) AS total
FROM customers c
JOIN orders  o  ON o.customer_id = c.id
JOIN coupons cp ON cp.customer_id = c.id     -- SELECT/WHERE에 안 쓰이는데 JOIN됨
WHERE c.region = 'A'
GROUP BY c.id, c.name;
```
데이터: 김(고객1) 주문 2건(1000·2000) + 쿠폰 3장. 실제 주문액 = 3000.
- **내 예측 변천(그대로)**:
  - **2차**: "실제 총 주문액은 0원(쿠폰 때문). 쿼리가 출력하는 total은 3000. 둘이 다른 이유는 coupons의 쿠폰값 때문이고, where는 customer 구분을 위한 조건이라 이 문제엔 적용되지 않는다." → ❌ ("WHERE 무관"은 맞힘 / 그러나 JOIN을 안 펼쳐 행 수·합계를 거꾸로 봄, 쿠폰이 금액을 0으로 만든다는 오해)
  - **3차** (6행 직접 전개 후): "total은 9000. 3배이며 이 배수는 coupon 개수와 동일. 각 주문별 각 쿠폰을 적용하도록 join이 걸려 경우의 수로 부풀고, where로는 막지 못한다." → ✅ **진단 합격**
- **실제**: 주문 2 × 쿠폰 3 = 6행 복제 → `SUM(o.amount)` = (1000+2000)×3 = **9000** (실제 3000의 3배 = 쿠폰 수).
- **교정 (내가 직접 작성)**:
  ```sql
  SELECT c.name, SUM(o.amount) AS total
  FROM customers c
  JOIN orders o ON o.customer_id = c.id   -- 안 쓰는 coupons JOIN 제거(근본 원인)
  WHERE c.region = 'A'
  GROUP BY c.id, c.name;
  ```
  - `SUM(DISTINCT o.amount)` 함정 회피 — 서로 다른 주문이 우연히 같은 금액이면 누락된다. 근본 수정은 **불필요한 JOIN 제거**.
  - 작성 시 `customer_id` → `customers_id` 오타 1건. → **내가 짠 교정 코드도 검증 대상**이라는 메타 교훈.
- **교훈**: 집계 전 "이 JOIN이 행을 늘리나?"를 먼저 묻는다. SELECT/WHERE에 안 쓰는 조인은 지운다.

---

## 강화된 교훈 (누적)
- 집계 쿼리는 `SUM`/`COUNT` 전에 **"이 JOIN이 행을 늘리나?"** 를 먼저 묻는다.
- 의심되면 **`GROUP BY`/`SUM`을 떼고 JOIN 결과를 한 행씩 펼쳐서** 직접 센다. (머릿속 직관 < 실제 행 전개)
- `WHERE`는 *어떤 행을 통과시킬지*만 거른다. **이미 복제된 행을 다시 합쳐 줄이지 못한다.**
- **교정도 검증한다** (학습 완료 = 진단 → 교정 → 교정 검증): 손쉬운 `SUM(DISTINCT)`는 값이 우연히 겹치면 또 틀린다 → 근본은 *불필요한 JOIN 제거*. 그리고 내가 방금 짠 fix도 컬럼명까지 한 번 더 읽는다.
- 묶는 기준은 라벨(name)이 아니라 **식별자(id)**.

### 다음 개선 포인트
- 코드를 위→아래 직관으로만 읽지 말고, **데이터를 머릿속에서 한 행씩 굴려 반례를 떠올리기**. "order 10에 payment 2건이면? 동명이인이면?"
- 다음 SQL 날: **"JOIN 행 복제"를 스캐폴드 없이 cold로** 재확인 — 혼자 join을 펼쳐 진단 + 교정까지 가면 진짜 마스터.
