<!--
================================================================
Cursor 합치기 지침 (이 주석 블록은 합친 뒤 지워도 됨)
- 아래 "🔁 복습 섹션" 전체를, 오늘 정규 SQL을 기록한 파일
  (예: days/day01.md) 맨 아래에 그대로 이어붙여줘.
- 기존 정규 문제 내용은 절대 덮어쓰지 말 것. 추가만.
- 이 파일엔 정규 문제가 없으니 중복 걱정 없음. 복습만 들어있음.
- 붙인 뒤 LOG.md 갱신은 이 파일 맨 아래 [LOG 추가분] 참고.
================================================================
-->

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

<!--
================================================================
[LOG 추가분] — LOG.md 맨 아래에 이어붙일 내용.
이미 LOG.md에 Day 01 정규 블록이 있으면, 아래 "복습 (파생)"
이하만 그 블록 안에 추가해줘.
================================================================

## Day 01 — SQL  (※ 이미 있으면 이 헤더는 빼고 아래만 추가)

**복습 (파생)**
- `GROUP BY name`은 동명이인을 **합친다**(묶기지 분리가 아님, DISTINCT 무관).
- JOIN = "합치기"가 아니라 **"짝 있는 행만 남기기"**. 짝 없는 행(쿠폰 0개 주문)은 INNER JOIN에서 탈락. → 데이터의 사실 ≠ 쿼리의 시야.
- JOIN은 **거르기 + 복제 양면**. WHERE는 줄이기만. JOIN 볼 때마다 "거르나, 늘리나, 둘 다인가" 묻기.
- fan-out(복제) 차단 = **서브쿼리로 주문 단위 미리 집계해 1행으로 접어 붙이기**.
- `GROUP BY` 쓰면 SELECT 모든 컬럼은 (1)그룹키 or (2)집계함수. MySQL이 봐주는 건 PK 함수적 종속일 뿐.

**메타 교훈**
- 데이터를 한 행씩 굴려 반례 떠올리기.
- "규칙이 통과한다 ≠ 옳다." 왜 통과하는지를 묻는 습관이 검증의 핵심.
================================================================
-->
