# 검증 드릴 오답 로그

> AI가 "자신 있게 틀리는" 패턴을 모은다. mint-agent trust layer의 검증 규칙 후보이자 포트폴리오 자산.

**항목 형식(통일):** 모든 항목(정규·복기·복습)은 같은 5필드 — **문제(코드) → 내 예측(그대로) → 실제 → 교정(고친 코드) → 교훈**. *틀린 것만* 기록하고, 예측은 **원문 그대로** 보존한다. 깊은 개념은 *보충*으로 덧붙인다.
**구조:** 날짜(Day)별 섹션 = `정규 → [복기] → [복습] → 그날의 핵심`. 날짜를 가로지르는 누적 교훈은 맨 끝 **누적** 섹션.

---

## 📅 2026-06-29 · Day 1 — SQL

### 정규

> ⚠️ 이전 세션 진행분 — 원본 SQL 미보유. 아래 **문제 코드는 재구성본**, "예측/실제/교훈"은 당시 기록 원문. 교정은 사후 보강.

#### 오답 1 — JOIN 행 복제 ≠ WHERE 필터
- **문제 (재구성본)**:
  ```sql
  -- 유저별 총 결제액: 결제 완료('done') 주문 금액 합
  SELECT u.name, SUM(o.amount) AS total
  FROM users u
  JOIN orders   o ON o.user_id  = u.id
  JOIN payments p ON p.order_id = o.id     -- 주문 1 : 결제 N
  WHERE o.status = 'done'
  GROUP BY u.name;
  ```
- **내 예측 (그대로)**: "`status = 'done'` 조건을 걸었기 때문에 금액이 실제보다 크게 나올 수 없다."
- **실제**: 크게 나올 수 있다. 원인은 WHERE가 아니라 **JOIN이 행을 복제**하는 것. 한 주문에 결제가 2건이면 `o.amount`가 2번 더해진다. status 필터는 *어떤 주문을 포함할지*만 거를 뿐 복제를 못 막는다.
- **교정 (고친 코드)**:
  ```sql
  SELECT u.id, u.name, SUM(o.amount) AS total
  FROM users u
  JOIN orders o ON o.user_id = u.id
  WHERE o.status = 'done'
    AND EXISTS (SELECT 1 FROM payments p WHERE p.order_id = o.id)  -- 복제 없이 '결제 있음'만 필터
  GROUP BY u.id, u.name;
  ```
- **교훈**: JOIN마다 "이 JOIN이 행을 늘리나?"를 먼저 묻는다. '결제 있음' 필터는 JOIN 대신 `EXISTS`로(또는 결제를 주문 단위로 먼저 집계). WHERE로는 못 막는다.

#### 오답 2 — 집계 GROUP BY는 식별자(id)로
- **문제**: 위 같은 쿼리의 `GROUP BY u.name`.
- **내 예측 (그대로)**: "SELECT 컬럼이 2개라서 GROUP BY에 두 컬럼 모두 써야 한다."
- **실제**: SUM은 집계라 GROUP BY에 안 넣어도 문법은 통과. 진짜 위험은 **name이 유저를 유일 식별 못 함** — 동명이인이면 다른 사람 결제액이 한 줄로 합쳐진다.
- **교정 (고친 코드)**: `GROUP BY u.id, u.name` (식별은 이름이 아니라 id, 표시용으로 name 동반).
- **교훈**: 사람을 식별하는 건 이름이 아니라 id.

> ✅ **잘한 것**: INNER JOIN이라 결제 없는 'done' 주문은 결과에서 빠진다는 걸 정확히 예측. (모두 살리려면 LEFT JOIN → amount NULL)

### 복기 (마스터리 루프 — 같은 개념 다른 유형으로 합격까지)

#### 복기 2 — 집계는 식별자로 GROUP BY · **1회 합격**
- **문제**:
  ```sql
  SELECT b.name AS brand, SUM(p.stock) AS total_stock
  FROM products p
  JOIN brands b ON b.id = p.brand_id
  GROUP BY b.name;
  ```
- **내 예측 (그대로)**: "브랜드별 재고를 정확히 보여줄 수 없다. 중복된 브랜드명이면 그 재고 수량을 합쳐서 보여주게 되기 때문에 group by 를 name 이 아닌 각 name 별 id 로 group by 를 해야 정확한 재고 파악이 된다."
- **실제**: 정확. 서로 다른 `brand_id`가 같은 이름이면 한 줄로 합쳐져 잘못된 합계.
- **교정 (고친 코드)**: `GROUP BY b.id, b.name`.
- **교훈**: 묶는 기준은 라벨(name)이 아니라 식별자(id). ✅

#### 복기 1 — JOIN 행 복제 · **3회 만에 합격** (변형 2개)
- **문제 (변형 A, 1차)**:
  ```sql
  SELECT p.name, SUM(s.amount) AS revenue
  FROM products p
  JOIN promotions pr ON pr.product_id = p.id   -- 상품 1 : 프로모션 N
  JOIN sales      s  ON s.product_id  = p.id   -- 상품 1 : 판매 N
  WHERE p.in_stock = TRUE
  GROUP BY p.id, p.name;
  ```
  **문제 (변형 B, 2~3차)**:
  ```sql
  SELECT c.name, SUM(o.amount) AS total
  FROM customers c
  JOIN orders  o  ON o.customer_id = c.id
  JOIN coupons cp ON cp.customer_id = c.id     -- SELECT/WHERE에 안 쓰이는데 JOIN됨
  WHERE c.region = 'A'
  GROUP BY c.id, c.name;
  ```
  데이터: 김(고객1) 주문 2건(1000·2000) + 쿠폰 3장. 실제 주문액 = 3000.
- **내 예측 변천 (그대로)**:
  - 1차(A): "revenue는 실제 매출과 일치하다. 그 이유는 각 프로모션 별 각 매출을 행에 추가되서 출력이 되지만, 조건부가 true 값만을 부르기 때문에 실제 매출만을 출력 되기 때문이다." → ❌ 오답 1과 같은 함정 반복.
  - 2차(B): "실제 총 주문액은 0원(쿠폰 때문). 쿼리 출력 total은 3000. … where는 customer 구분을 위한 조건이라 이 문제엔 적용되지 않는다." → ❌ (WHERE 무관은 맞힘 / JOIN 안 펼쳐 행수·합계 거꾸로)
  - 3차(B, 6행 전개): "total은 9000. 3배이며 이 배수는 coupon 개수와 동일. 각 주문별 각 쿠폰… 경우의 수로 부풀고, where로는 막지 못한다." → ✅ 진단 합격.
- **실제**: 주문 2 × 쿠폰 3 = 6행 복제 → `SUM(o.amount)` = (1000+2000)×3 = **9000** (실제 3000의 3배 = 쿠폰 수).
- **교정 (고친 코드)**:
  ```sql
  SELECT c.name, SUM(o.amount) AS total
  FROM customers c
  JOIN orders o ON o.customer_id = c.id   -- 안 쓰는 coupons JOIN 제거(근본 원인)
  WHERE c.region = 'A'
  GROUP BY c.id, c.name;
  ```
  `SUM(DISTINCT)` 함정 회피(다른 주문이 같은 금액이면 누락) · 작성 시 `customer_id→customers_id` 오타 1건(내 교정도 검증 대상).
- **교훈**: 집계 전 "이 JOIN이 행을 늘리나?". SELECT/WHERE에 안 쓰는 조인은 지운다.

### 복습 (파생 · 집 이동 중)

#### 복습 — 고객별 결제 총액 (JOIN 복제 + 동명이인 동시 함정)
- **문제**:
  ```sql
  SELECT c.name, SUM(o.amount) AS total
  FROM customers c
  JOIN orders o   ON o.customer_id = c.id
  JOIN coupons cp ON cp.order_id = o.id
  GROUP BY c.name;
  ```
  데이터:
  ```
  customers          orders                       coupons
  id | name          id  | customer_id | amount    id | order_id
  1  | 박지훈         101 | 1           | 10000     1  | 101
  2  | 박지훈         102 | 2           | 20000     2  | 101
  3  | 이수아         103 | 3           | 5000      3  | 102
  ```
  (order 101엔 쿠폰 2개, 102엔 1개, 103엔 0개)
- **내 예측 (그대로)**: 박지훈/박지훈/이수아 3줄, total 45000 (101×2 + 102 + 103).
- **실제**: **딱 한 줄** → `박지훈 | 40000`. 어긋난 3지점 —
  1. `GROUP BY c.name`이 동명이인(박지훈 2명)을 한 그룹으로 **합침**.
  2. order101이 쿠폰 2개로 **2행 복제** → 10000을 두 번 더함.
  3. order103은 쿠폰 0개 → **INNER JOIN에서 탈락**, 5000 사라짐.
- **교정 (고친 코드)**:
  - 수정 1 — 안 쓰는 coupons 제거(가장 깨끗):
    ```sql
    SELECT c.id, c.name, SUM(o.amount) AS total
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id;   -- 동명이인 분리
    ```
  - 수정 2 — 쿠폰 수도 보고 싶으면(fan-out 방지: 주문 단위로 먼저 1행 집계 후 LEFT JOIN):
    ```sql
    SELECT c.id, c.name,
           SUM(coupon_cnt.cp_total) AS coupon_count,
           SUM(o.amount)            AS total
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    LEFT JOIN (
        SELECT order_id, COUNT(*) AS cp_total
        FROM coupons GROUP BY order_id
    ) AS coupon_cnt ON coupon_cnt.order_id = o.id
    GROUP BY c.id;
    ```
    | c.id | name | coupon_count | total |
    |---|---|---|---|
    | 1 | 박지훈 | 2 | 10000 |
    | 2 | 박지훈 | 1 | 20000 |
    | 3 | 이수아 | NULL→0 | 5000 |
    (이수아 NULL은 `COALESCE(SUM(...), 0)`으로 0 처리)
- **교훈**:
  - JOIN은 "합치기"가 아니라 **"짝 있는 행만 남기기"** — order103 탈락은 데이터 삭제가 아니라 *쿼리의 시야*에서 짝을 못 찾은 것. 데이터의 사실 ≠ 쿼리의 시야. 살리려면 LEFT JOIN.
  - WHERE는 줄이기만, JOIN은 **거르기 + 복제 양면**.
  - fan-out 차단 = 서브쿼리로 주문 단위 미리 집계해 1행으로 접어 붙이기.
- **보충 — `c.name`이 GROUP BY에 없는데 왜 OK?**: 엄격히는 표준 SQL 위반(MySQL이 `ONLY_FULL_GROUP_BY` 끄면 봐줌). `c.id`가 PK라 id가 정해지면 name이 자동 유일 → **함수적 종속**. 위험: 설정 따라 `ERROR 1055` 또는 종속 안 된 컬럼이면 *아무 행이나 조용히* 집어옴. 안전 습관: 보여줄 비집계 컬럼은 전부 GROUP BY에.

### 그날의 핵심
- `GROUP BY name`은 동명이인을 **합친다**(묶기지 분리 아님, DISTINCT 무관) → 식별자 id로 묶기.
- JOIN = 짝 있는 행만 남기기 + 짝 여러 개면 복제. WHERE는 줄이기만. 볼 때마다 "거르나/늘리나/둘 다?".
- fan-out 차단 = 서브쿼리로 미리 1행 집계 후 붙이기 (실무 표준 패턴).
- "규칙이 통과한다 ≠ 옳다." 왜 통과하는지를 물어야 한다.

---

## 📅 2026-06-30 · Day 2 — pandas

### 정규

#### 오답 — merge 행 복제(fan-out)는 how(inner/left)로 못 막는다
- **문제 (검토한 코드)**:
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
- **내 예측 변천 (그대로)**:
  - 1차: "merged 4행 / total 600. how=left라 짝 없는 행까지 포함되어 모두 더한 값." → ❌ (cities 'A' 중복 못 봄)
  - 2차: "merged 9행 / total 1800." → ❌ (과교정 — 전체 조합 cross join 오해)
  - 3차: "merged 7행 / total 1200." → ❌ ('A' 매칭 3줄로 오산, 실제 2줄)
  - 4차: "merged 5행 / total 900." → ✅ 진단 합격.
- **실제**: `cities`에 'A'가 2줄 → 'A' 주문(order 1·2) 각각 2행 복제. merged = 2+2+1 = **5행**, `SUM` = 100×2+200×2+300×1 = **900**(실제 600의 1.5배). merge는 **키 같은 행끼리만**(cross join 아님).
- **교정 (고친 코드)**:
  ```python
  total = orders["amount"].sum()   # total엔 도시명 불필요 → cities/merge 제거
  print(total)                      # 600
  ```
  (교정 변천: ① `how="inner"` → ❌ 모두 매칭이라 inner=left, 여전히 900 → join 방식은 복제 제어 못 함. ② merge 제거했으나 지운 `merged` 참조 → `NameError` + 콤마 누락. ③ 위 코드로 합격.)
- **교훈**: `merge` = SQL `JOIN`. 오른쪽 키가 중복이면 **fan-out 복제**. `how`(inner/left/…)는 *짝 없는 행* 처리만 바꿀 뿐 복제를 못 막는다. 집계에 안 쓰는 테이블은 merge하지 않는다(필요하면 `drop_duplicates`로 1행/키 정제). 변수 지웠으면 참조 줄도 같이 고친다.

*복기 없음 — 정규 문제를 스캐폴드 트레이스로 그 자리에서 합격.*

### 복습 (cold 재확인 — 힌트 없이 같은 개념 새 시나리오)

#### 복습 — 부서명 붙여 총 급여 (merge fan-out)
- **문제**:
  ```python
  import pandas as pd

  employees = pd.DataFrame({
      "emp_id":  [1, 2, 3, 4],
      "dept_id": [10, 10, 20, 30],
      "salary":  [300, 400, 500, 600],
  })
  departments = pd.DataFrame({
      "dept_id":   [10, 20, 20, 30],
      "dept_name": ["영업", "개발", "Development", "인사"],   # dept_id 20이 2줄
  })
  merged = employees.merge(departments, on="dept_id", how="left")
  total_salary = merged["salary"].sum()
  print(total_salary)
  ```
  실제 급여 합 = 1800.
- **내 예측 변천 (그대로)**:
  - cold 1차: "merged 7행 / total 2300 / *left merge라서 모든 행이 더해짐* / 교정 `salary.sum()`" → ❌ (total 2300만 맞음 / 행수 7 오답 + 원인을 'how=left'로 회귀 = 오늘 오전 고친 함정 / `salary` 미정의 `NameError`)
  - 나사 조인 뒤("오른쪽 키 `departments.dept_id`가 유니크?"): 중복키 20 발견 → "5행 / 2300 / 원인은 중복 키" → ✅ 합격.
- **실제**: dept_id 20이 departments에 2줄 → emp3(dept20) 2행 복제. merged = 1+1+2+1 = **5행**, `SUM` = 300+400+500×2+600 = **2300**(실제 1800).
- **교정 (고친 코드)**: `total_salary = employees['salary'].sum()` (= 1800, 동작). 부서명도 필요하면 `departments.drop_duplicates("dept_id")` 후 merge.
- **교훈**: merge 보면 *제일 먼저* "오른쪽 조인 키가 유니크한가(1:1? 1:N?)". 코드 가드 `merge(..., validate="m:1")` — 더러운 lookup이면 `MergeError`로 즉시 잡힘(③ 교정 검증 도구).

### 그날의 핵심
- `merge` = `JOIN`. 오른쪽 키 중복 → fan-out. `how`는 *짝 없는 행* 처리만, 복제 제어 아님.
- 집계 metric에 안 쓰는 테이블은 안 붙인다. 붙여야 하면 lookup을 키당 1행으로 정제(`drop_duplicates`) + `validate="m:1"` 가드.
- 개념은 잡았으나 **cold에서 'how=left'로 한 번 회귀** → "오른쪽 키 유니크?" 자동 점검 습관화 필요.

---

## 📅 2026-07-01 · Day 3 — Python

### 정규

#### 오답 — mutable 기본 인자(`batch=[]`)는 호출 간에 공유된다
- **문제 (검토한 코드)**:
  ```python
  def add_event(event, batch=[]):
      batch.append(event)
      return batch

  a = add_event("login")
  b = add_event("click")
  c = add_event("logout")

  print(a); print(b); print(c)
  ```
- **내 예측 변천 (그대로)**:
  - 1차: "a=login, b=click, c=logout 따로 출력 / 원인은 for문 필요 / 교정도 for문." → ❌ (출력·원인·교정 빗나감. 단 '독립적 새 batch가 아니라 하나의 batch에 계속 쌓인다'는 핵심은 이때 이미 맞힘)
  - 2차: "셋 다 `['login', 'click', 'logout']` — a·b·c가 *같은 리스트를 가리키는 세 이름*이라서." → ✅ **진단 합격** (aliasing까지 정확)
- **실제**: `batch=[]` 기본값은 **함수 정의 시 딱 한 번** 생성 → 모든 호출이 그 한 리스트를 공유·누적. a·b·c는 같은 객체를 가리켜 셋 다 `['login', 'click', 'logout']`.
- **교정 (고친 코드)**:
  ```python
  def add_event(event):
      batch = []            # 호출마다 몸통이 실행 → 새 빈 리스트
      batch.append(event)
      return batch
  # → a=['login'], b=['click'], c=['logout'] (독립)
  ```
  교정 변천: ① for문 → ❌ (반복문은 원인 아님) · ② `batch='None'`(문자열!) + `if batch==1;` → ❌ (문법·개념 오류, 버그 그대로) · ③ 위 코드로 합격.
  넘겨받은 batch도 허용하는 **프로 버전**(선택 인자 `batch=None`) + 그 과정에서 만난 들여쓰기 함정은 아래 **`보충 — 프로 버전`** 참고.
- **교훈**: 기본 인자는 **정의 시 1번** 평가된다 → 리스트·딕트 같은 mutable을 기본값에 두면 호출 간 공유(버그). 매 호출 새 객체가 필요하면 **함수 몸통 안**에서 만든다(`None` 기본값 + `is None` 체크가 표준). `None`(값) ≠ `'None'`(문자열).
- **보충 — `None` 비교는 `is None`으로 (`== None` 아님)**: `is None` / `is not None`이 정답 습관(PEP 8 공식 권장). `== None`은 *문법 오류는 아니고* 평범한 경우 잘 동작하지만 **비권장·위험**. 직접 검증한 근거:

  | 케이스 | `== None` | `is None` |
  |--------|-----------|-----------|
  | `None` 자체 | `True` | `True` |
  | 빈 리스트 `[]` | `False` | `False` |
  | 커스텀 `__eq__` 객체 | `True` ⚠️ | `False` ✅ |
  | numpy 배열 | `[False False False]`(bool 아님) | `False` ✅ |
  | numpy `if arr == None:` | **`ValueError`** 💥 | 안전 |

  - None은 파이썬에 **딱 하나뿐인 객체(싱글턴)** → "그 None이냐"는 값(`==`)이 아니라 **정체성(`is`)** 질문.
  - `==`는 클래스가 `__eq__`를 덮으면 **속을 수 있다**(위 3행). `is`는 객체 정체성이라 안 속음.
  - **numpy/pandas 배열**은 `배열 == None`이 bool이 아니라 **원소별 결과** → `if`에서 `ValueError`. 데이터 실무에서 제일 자주 만나는 함정.

#### 보충 — 프로 버전: 선택 인자 `batch=None` (넘긴 batch도 이어붙이기)

간단판(`def add_event(event):` + 몸통에서 `batch = []`)은 *항상* 새 리스트라 남이 만든 리스트를 못 받는다. 프로 버전은 `batch`를 **선택 매개변수**로 남겨 "새로 시작"과 "기존 리스트에 이어붙이기"를 모두 지원한다.

**`()` 안 매개변수 해석** — 함수가 받는 재료 목록이고, `=` 유무가 필수/선택을 가른다:

| 매개변수 | 의미 |
|---|---|
| `event` (`=` 없음) | **필수** — 안 주면 에러 |
| `batch=None` (`=` 있음) | **선택** — 안 주면 자동으로 `None` |

**호출별 동작:**
```python
add_event("login")          # batch=None  → if 참 → 새 [] → append → ['login']
add_event("save", ["x"])    # batch=["x"] → if 거짓 → 바로 append → ['x', 'save']
```
기본값을 `[]`가 아니라 **`None`**으로 두는 이유: `[]`면 정의 시 1번 생성돼 호출 간 공유되는 버그(위 오답). `None`은 안전한 "안 줬음" 표시라, 몸통에서 그때그때 새 `[]`를 만든다.

**내가 겪은 함정 — 들여쓰기 한 칸이 로직을 바꿈:**
```python
# ❌ append가 if 안 → batch를 넘기면(if 거짓) append 통째로 건너뜀 → event 누락
def add_event(event, batch=None):
    if batch is None:
        batch = []
        batch.append(event)   # if 블록 안
    return batch

# ✅ append를 몸통으로 빼 두 경우 모두 실행
def add_event(event, batch=None):
    if batch is None:
        batch = []
    batch.append(event)       # if 밖 = 함수 몸통
    return batch
```
- 검증: `add_event("save", ["x"])` → ❌판은 `['x']`(save 누락), ✅판은 `['x', 'save']`.
- **교훈**: `if` 블록 안은 *조건이 참일 때만* 실행된다. **항상 해야 할 일(append)을 `if` 안에 두면 조건이 거짓일 때 조용히 누락**된다. 들여쓰기 = 코드의 실행 범위.

#### 보충 — 함수 읽기 기초: `return` · 지역변수(scope)

Day 3 코드를 파다 나온 기초 3가지. mutable 함정과는 별개로 "함수를 코드로 읽는 눈"의 토대.

**① `if`와 `return`은 독립** — `if`는 *실행 여부*, `return`은 *돌려줄 값*. 서로 무관하다.
```python
def 판정(x):
    if x > 0:          # 조건은 x
        y = "양수"
    else:
        y = "음수/0"
    return y           # 돌려주는 건 y — x와 전혀 다름
```
`if`에 쓴 변수와 `return` 값은 같을 수도, 완전히 다를 수도 있다. `add_event`가 `return batch`였던 건 "함수 목적이 batch를 돌려주는 것"이라서지 `if`가 batch를 언급해서가 아니다. (batch 길이를 원하면 `return len(batch)`도 정상.)

**② 반환값 읽는 법** — `return` 단어를 찾고 → 어느 경로가 실행되는지 보고 → 그 뒤 값을 계산. `return`을 하나도 안 만나면 `None`. `return`이 실행되면 함수는 *즉시* 끝난다(먼저 만난 return이 이김).
```python
def grade(score):
    if score >= 90:
        return "A"     # 먼저 참이면 여기서 끝
    if score >= 80:
        return "B"
    return "C"
# grade(95)->'A'   grade(85)->'B'   grade(50)->'C'   (경로 따라 다른 return)

def greet(name):
    print("hi", name)  # return 없음
# greet("x") 의 반환값 -> None
```
`return` 뒤엔 변수·값·계산식(`return len(batch)`)·함수호출·튜플(`return x, y`)·없음(→`None`) 다 가능.

**③ 지역변수(scope) + `return` vs `print`** — 함수 *안* 변수(`result`, `n`)는 함수 *밖*에서 안 보인다.
```python
def check(n):
    if n % 2 == 0:
        result = "짝수"
    else:
        result = "홀수"
    return result

print(result)        # ❌ NameError — result는 check 안에서만 삶
answer = check(7)    # ✅ 반환값을 받아서
print(answer)        #    출력  -> '홀수'
print(check(7))      # ✅ 또는 호출 결과를 바로  -> '홀수'
```
- **함수 밖에서는** 내부 변수 이름(`result`, `n`)이 아니라 **반환값**을 쓴다 → `print(check(7))`.
- **`return`** = 값을 함수 밖으로 *돌려줌*(화면엔 안 보임) vs **`print`** = 화면에 *보여줌*(돌려주는 것 아님). 관례: **값은 `return`, 출력은 밖에서.**

**④ 반환값은 "두 쪽" + 받아야 산다** — `return`엔 *내보내는 함수 쪽*과 *받는 호출부 쪽* 둘이 있고, **받는 쪽이 안 받으면 그 값은 사라진다.**
```python
check(7)            # 값은 나왔지만 아무도 안 받음 → 허공에 버려짐
answer = check(7)   # answer가 받아둠 → 이제 쓸 수 있음
print(check(7))     # print가 즉석에서 받아 화면에 보여줌
```
멘탈 모델(직접 잡음): `return` = "함수가 만든 결과값을 밖으로 되돌려준다". 그걸 **변수에 받거나 바로 쓰지 않으면 놓친다** — 내보내기와 받기는 항상 세트.

### 그날의 핵심
- mutable 기본 인자(`[]`, `{}`)는 **정의 시 1번 생성** → 호출 간 공유되는 함정. 매 호출 새 객체는 *몸통 안*에서 만든다.
- 변수 여러 개가 같은 리스트를 가리키면(aliasing) 하나를 바꿔도 전부 바뀐 것처럼 보인다.
- 감이 안 오면 대개 개념이 아직 덜 잡힌 것 — 핵심을 잡으면 고침은 한 줄일 때가 많다.
- (함수 기초) `if`≠`return`(독립) · 반환값은 `return` 뒤를 *경로 따라* 읽기 · 지역변수는 함수 밖에서 안 보임(scope) → 밖에선 반환값 사용. (↑ 보충 참고)

---

## 📚 누적 (전 기간)

### 강화된 교훈
- 집계 쿼리는 `SUM`/`COUNT` 전에 **"이 JOIN이 행을 늘리나?"** 를 먼저 묻는다.
- 의심되면 **`GROUP BY`/`SUM`을 떼고 JOIN 결과를 한 행씩 펼쳐서** 직접 센다. (머릿속 직관 < 실제 행 전개)
- `WHERE`는 *어떤 행을 통과시킬지*만 거른다. **이미 복제된 행을 다시 합쳐 줄이지 못한다.**
- **교정도 검증한다** (학습 완료 = 진단 → 교정 → 교정 검증): 손쉬운 `SUM(DISTINCT)`는 값이 우연히 겹치면 또 틀린다 → 근본은 *불필요한 JOIN 제거*. 그리고 내가 방금 짠 fix도 컬럼명까지 한 번 더 읽는다.
- 묶는 기준은 라벨(name)이 아니라 **식별자(id)**.
- **(pandas) `merge` = `JOIN`** — 오른쪽 키가 중복이면 fan-out 복제. `how`(inner/left)는 *짝 없는 행* 처리만 바꿀 뿐 복제를 못 막는다. 집계에 안 쓰는 테이블은 merge하지 않는다(`validate="m:1"`로 사전 검증 가능).
- **(Python) mutable 기본 인자**(`def f(x, items=[])`)는 정의 시 1번 생성 → 호출 간 공유되는 함정. 매 호출 새 객체가 필요하면 몸통 안에서(`items=None` + `if items is None: items=[]`).
- **(Python) `None` 비교는 항상 `is None` / `is not None`** (`== None` 아님). 이유: 싱글턴 정체성 / `==`는 `__eq__`에 속을 수 있음 / numpy·pandas 배열에선 `==None`이 원소별 → `if`에서 `ValueError`.

### 다음 개선 포인트
- 코드를 위→아래 직관으로만 읽지 말고, **데이터를 머릿속에서 한 행씩 굴려 반례를 떠올리기**. "order 10에 payment 2건이면? 동명이인이면?"
- 다음 SQL 날: **"JOIN 행 복제"를 스캐폴드 없이 cold로** 재확인 — 혼자 join을 펼쳐 진단 + 교정까지 가면 진짜 마스터.
- pandas `merge` fan-out: 2026-06-30 cold 재확인 → 1차에 'how=left' 오해로 회귀, 나사 한 번 조인 뒤 합격. **개념 OK. 남은 건 "오른쪽 키 유니크?"를 *자동으로* 묻는 습관 + `validate=` 가드 손에 익히기.**
