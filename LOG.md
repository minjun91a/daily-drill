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

## 2026-06-30 — pandas · merge fan-out (도시명 붙여 총 주문액)

**문제 (검토한 코드)**
```python
import pandas as pd

orders = pd.DataFrame({
    "order_id":  [1, 2, 3],
    "city_code": ["A", "A", "B"],
    "amount":    [100, 200, 300],
})
cities = pd.DataFrame({
    "city_code": ["A", "A", "B"],
    "city_name": ["서울", "Seoul", "부산"],   # 'A'가 2줄 — 더러운 매핑 테이블
})
merged = orders.merge(cities, on="city_code", how="left")
total = merged["amount"].sum()
print(total)
```
실제 주문액 합 = 600.

### 오답 — merge 행 복제(fan-out)는 how(inner/left)로 막을 수 없다
- **내 예측 변천(그대로)**:
  - 1차: "merged 4행 / total 600. how=left라 짝 없는 행까지 포함되어 모두 더한 값." → ❌ (cities의 'A' 중복을 못 봄, fan-out 자체를 놓침)
  - 2차: "merged 9행 / total 1800." → ❌ (과교정 — 전체 조합 cross join으로 오해)
  - 3차: "merged 7행 / total 1200." → ❌ ('A' 매칭을 3줄로 오산, 실제 2줄)
  - 4차: "merged 5행 / total 900." → ✅ **진단 합격**
- **실제**: `cities`에 city_code 'A'가 2줄 → 'A' 주문(order 1·2)이 각각 2행으로 복제. merged = 2+2+1 = **5행**, `SUM` = 100×2+200×2+300×1 = **900** (실제 600의 1.5배). merge는 **키가 같은 행끼리만** 짝지음(cross join 아님).
- **교정 변천**:
  - 1차: `how="inner"`로 변경 → ❌ 세 주문 모두 매칭되므로 inner=left, 여전히 5행/900. **join 방식(inner/left)은 복제를 제어하지 않는다.**
  - 2차: merge 제거(정답 방향) — 그러나 `total = merged["amount"].sum()`에서 지운 `merged`를 그대로 참조 → ❌ `NameError`(+ 콤마 누락 `SyntaxError`).
  - 3차 (합격):
    ```python
    total = orders["amount"].sum()   # total엔 도시명 불필요 → cities/merge 제거
    print(total)                      # 600
    ```
- **교훈**:
  - `merge` = SQL `JOIN`. **오른쪽 키가 중복이면 행이 복제(fan-out)된다. `how`(inner/left/right/outer)는 *짝 없는 행* 처리만 바꿀 뿐, *짝 여러 개의 복제*를 막지 못한다.**
  - 집계 metric(total)에 안 쓰는 테이블은 merge하지 않는다. 도시명도 필요하면 lookup을 `drop_duplicates("city_code")`로 1행/키 정제 후 merge(또는 `validate="m:1"`로 사전 검증).
  - 변수를 지웠으면 그 변수를 참조하는 줄도 같이 고친다(`NameError` 자가검증).

### 🔁 복습 (cold 재확인) — 2026-06-30
같은 개념(merge fan-out)을 새 시나리오로 **힌트 없이** 재출제: `employees × departments`, dept_id 20이 departments에 2줄(개발/Development).
- **cold 1차(그대로)**: "merged 7행 / total 2300 / *left merge라서 모든 행이 더해짐* / 교정 `salary.sum()`" → ❌ — total 2300은 맞혔으나 **행 수 7 오답 + 원인을 다시 'how=left'로 회귀**(오늘 오전 고친 함정). 교정도 `salary` 미정의(`NameError`).
- **나사 한 번 조인 뒤**("오른쪽 키 `departments.dept_id`가 유니크한가?"): 중복키(20) 발견 → emp3가 2행 복제 → **5행 / 2300**, 원인을 *중복 키*로 정확히 지목, 교정 `employees['salary'].sum()`(=1800, 동작) → ✅ **합격**.
- **남은 습관**: merge를 보면 *제일 먼저* "오른쪽 조인 키가 유니크한가(1:1? 1:N?)"를 묻기. 코드 가드는 `merge(..., validate="m:1")` — 더러운 lookup이면 `MergeError`로 즉시 잡힘(③ 교정 검증 실전 도구).

---

## 강화된 교훈 (누적)
- 집계 쿼리는 `SUM`/`COUNT` 전에 **"이 JOIN이 행을 늘리나?"** 를 먼저 묻는다.
- 의심되면 **`GROUP BY`/`SUM`을 떼고 JOIN 결과를 한 행씩 펼쳐서** 직접 센다. (머릿속 직관 < 실제 행 전개)
- `WHERE`는 *어떤 행을 통과시킬지*만 거른다. **이미 복제된 행을 다시 합쳐 줄이지 못한다.**
- **교정도 검증한다** (학습 완료 = 진단 → 교정 → 교정 검증): 손쉬운 `SUM(DISTINCT)`는 값이 우연히 겹치면 또 틀린다 → 근본은 *불필요한 JOIN 제거*. 그리고 내가 방금 짠 fix도 컬럼명까지 한 번 더 읽는다.
- 묶는 기준은 라벨(name)이 아니라 **식별자(id)**.
- **(pandas) `merge` = `JOIN`** — 오른쪽 키가 중복이면 fan-out 복제. `how`(inner/left)는 *짝 없는 행* 처리만 바꿀 뿐 복제를 못 막는다. 집계에 안 쓰는 테이블은 merge하지 않는다(`validate="m:1"`로 사전 검증 가능).

### 다음 개선 포인트
- 코드를 위→아래 직관으로만 읽지 말고, **데이터를 머릿속에서 한 행씩 굴려 반례를 떠올리기**. "order 10에 payment 2건이면? 동명이인이면?"
- 다음 SQL 날: **"JOIN 행 복제"를 스캐폴드 없이 cold로** 재확인 — 혼자 join을 펼쳐 진단 + 교정까지 가면 진짜 마스터.
- pandas `merge` fan-out: 2026-06-30 cold 재확인 → 1차에 'how=left' 오해로 회귀, 나사 한 번 조인 뒤 합격. **개념 OK. 남은 건 "오른쪽 키 유니크?"를 *자동으로* 묻는 습관 + `validate=` 가드 손에 익히기.**

---

## 🔁 복습 섹션 (정규 문제에서 파생 — 같은 개념, 다른 각도)

> 오늘 정규 SQL의 두 함정(JOIN 복제 / 식별자 집계)을 새 스키마로 다시 정의하고,
> 버그 쿼리 → 점진적 수정의 과정으로 풀었다. (집 이동 중 진행분)

### 스키마 & 데이터
```
customers          orders                       coupons
id | name          id  | customer_id | amount    id | order_id
1  | 박지훈         101 | 1           | 10000     1  | 101
2  | 박지훈         102 | 2           | 20000     2  | 101
3  | 이수아         103 | 3           | 5000      3  | 102
```
(order 101엔 쿠폰 2개, 102엔 1개, 103엔 0개)

### 버그 쿼리 — "고객별 결제 총액"이 목적
```sql
SELECT c.name, SUM(o.amount) AS total
FROM customers c
JOIN orders o   ON o.customer_id = c.id
JOIN coupons cp ON cp.order_id = o.id
GROUP BY c.name;
```

**예측 → 검증**
- 내 예측: 박지훈/박지훈/이수아 3줄, total 45000 (101×2 + 102 + 103)
- 실제 출력: **딱 한 줄** → `박지훈 | 40000`
- 어긋난 지점 3개:
  1. `GROUP BY c.name`은 동명이인을 **합친다** (DISTINCT 여부와 무관). 박지훈 두 명 → 한 그룹.
  2. order101이 쿠폰 2개로 **2행 복제** → `SUM(o.amount)`가 10000을 두 번 더함.
  3. order103은 쿠폰이 없어 **INNER JOIN에서 탈락** → 5000은 결과에 존재하지 않음.

### 개념 정리 (질문하며 잡은 것)
- **JOIN은 "합치기"가 아니라 "짝 있는 행만 남기기".** order103이 사라진 건 orders에서 삭제된 게 아니라, 쿼리의 시야(INNER JOIN)에서 짝을 못 찾았을 뿐. → 데이터의 사실 ≠ 쿼리의 시야.
- **JOIN은 거르기 + 복제 양면을 가진다.** WHERE는 줄이기만:
  ```
  WHERE : 행을 거른다 (줄이기만)
  JOIN  : 행을 거른다 + 짝이 여러 개면 늘린다 (양방향)
  ```
- 주문을 살리려면 `INNER JOIN` → `LEFT JOIN` (거르기 스위치 OFF, order103이 NULL 달고 생존).

### 수정 1 — coupons 제거 (가장 깨끗)
결제 총액에 쿠폰 테이블은 필요 없다.
```sql
SELECT c.id, c.name, SUM(o.amount) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id;   -- 동명이인 분리
```

### 수정 2 — 쿠폰 수도 같이 보고 싶다면 (fan-out 방지 패턴)
쿠폰을 **주문 단위로 먼저 집계해 1행으로 접은 뒤** LEFT JOIN → 복제 차단.
```sql
SELECT c.id, c.name,
       SUM(coupon_cnt.cp_total) AS coupon_count,
       SUM(o.amount)            AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
LEFT JOIN (
    SELECT order_id, COUNT(*) AS cp_total
    FROM coupons
    GROUP BY order_id
) AS coupon_cnt ON coupon_cnt.order_id = o.id
GROUP BY c.id;
```
| c.id | name | coupon_count | total |
|---|---|---|---|
| 1 | 박지훈 | 2 | 10000 |
| 2 | 박지훈 | 1 | 20000 |
| 3 | 이수아 | NULL→0 | 5000 |

(이수아 쿠폰 NULL은 `COALESCE(SUM(...), 0)`으로 0 처리 가능)

### 마지막 의문 — c.name이 GROUP BY에 없는데 왜 OK?
- 엄격히는 **표준 SQL 위반**. MySQL이 `ONLY_FULL_GROUP_BY`를 껐을 때 봐주는 것.
- 봐주는 이유: `c.id`가 **PK**라 id가 정해지면 name이 자동으로 유일 → **함수적 종속(functional dependency)**.
- 위험: DB/설정 따라 `ERROR 1055`로 터지거나, 종속 안 된 컬럼이면 **아무 행이나 조용히** 집어옴(자신 있게 틀린 값).
- **안전 습관**: 보여줄 비집계 컬럼은 전부 GROUP BY에 → `GROUP BY c.id, c.name`.

### 복습 핵심
- GROUP BY는 같은 값을 **묶는다**(분리가 아님). 동명이인 합침의 원인.
- fan-out(복제)은 **미리 집계해서 1행으로 접어 붙이면** 막힌다 — 실무 표준 패턴.
- **GROUP BY 쓰면 SELECT의 모든 컬럼은 (1) 그룹키 or (2) 집계함수**. 예외처럼 보이는 건 함수적 종속 봐주기일 뿐.
- "규칙이 통과한다 ≠ 옳다." 왜 통과하는지를 물어야 한다.

---

## Day 01 복습 — LOG 추가분

**복습 (파생)**
- `GROUP BY name`은 동명이인을 **합친다**(묶기지 분리가 아님, DISTINCT 무관).
- JOIN = "합치기"가 아니라 **"짝 있는 행만 남기기"**. 짝 없는 행(쿠폰 0개 주문)은 INNER JOIN에서 탈락. → 데이터의 사실 ≠ 쿼리의 시야.
- JOIN은 **거르기 + 복제 양면**. WHERE는 줄이기만. JOIN 볼 때마다 "거르나, 늘리나, 둘 다인가" 묻기.
- fan-out(복제) 차단 = **서브쿼리로 주문 단위 미리 집계해 1행으로 접어 붙이기**.
- `GROUP BY` 쓰면 SELECT 모든 컬럼은 (1)그룹키 or (2)집계함수. MySQL이 봐주는 건 PK 함수적 종속일 뿐.

**메타 교훈**
- 데이터를 한 행씩 굴려 반례 떠올리기.
- "규칙이 통과한다 ≠ 옳다." 왜 통과하는지를 묻는 습관이 검증의 핵심.
