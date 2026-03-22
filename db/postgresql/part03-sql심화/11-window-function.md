# Chapter 11. Window Function

## 11.1 Window Function이란?

### 정의

```
Window Function (윈도우 함수):
각 행에 대해 관련된 행들의 집합(윈도우)에 대한 계산 수행

비유: 반 등수 매기기
┌────────────────────────────────┐
│ 학생 | 점수 | 반 전체에서 등수 │
│ Alice| 90   | 1위              │
│ Bob  | 85   | 2위              │
│ Carol| 80   | 3위              │
│                                │
│ 모든 행 유지 + 추가 정보 제공  │
└────────────────────────────────┘

집계 함수 vs 윈도우 함수:
- 집계: 그룹당 1행
- 윈도우: 모든 행 유지
```

### 기본 문법

```sql
함수() OVER (
  [PARTITION BY column]
  [ORDER BY column]
  [ROWS/RANGE ...]
)
```

---

## 11.2 ROW_NUMBER

### 기본 사용

```sql
-- 순번 부여
SELECT
  name,
  age,
  ROW_NUMBER() OVER (ORDER BY age DESC) AS row_num
FROM users;

-- 결과:
--   name   | age | row_num
-- ---------+-----+---------
--  Alice   | 35  | 1
--  Bob     | 30  | 2
--  Charlie | 28  | 3
--  David   | 25  | 4
```

### PARTITION BY

```sql
-- 국가별로 순번
SELECT
  name,
  country,
  age,
  ROW_NUMBER() OVER (PARTITION BY country ORDER BY age DESC) AS country_rank
FROM users
ORDER BY country, country_rank;

-- 결과:
--   name   | country | age | country_rank
-- ---------+---------+-----+--------------
--  Alice   | Korea   | 35  | 1
--  Charlie | Korea   | 28  | 2
--  Bob     | USA     | 30  | 1
--  David   | USA     | 25  | 2

-- 국가별로 다시 1부터 시작
```

### TOP N per Group

```sql
-- 카테고리별 인기 게시글 TOP 3
SELECT *
FROM (
  SELECT
    category_id,
    title,
    view_count,
    ROW_NUMBER() OVER (
      PARTITION BY category_id
      ORDER BY view_count DESC
    ) AS rn
  FROM posts
) ranked
WHERE rn <= 3;
```

---

## 11.3 RANK / DENSE_RANK

### RANK

```sql
-- 등수 (동점 처리, 다음 순위 건너뜀)
SELECT
  name,
  score,
  RANK() OVER (ORDER BY score DESC) AS rank
FROM students;

-- 결과:
--   name   | score | rank
-- ---------+-------+------
--  Alice   | 95    | 1
--  Bob     | 90    | 2
--  Charlie | 90    | 2    ← 동점
--  David   | 85    | 4    ← 3 건너뜀
--  Eve     | 80    | 5
```

### DENSE_RANK

```sql
-- 밀집 등수 (동점 처리, 다음 순위 연속)
SELECT
  name,
  score,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM students;

-- 결과:
--   name   | score | dense_rank
-- ---------+-------+------------
--  Alice   | 95    | 1
--  Bob     | 90    | 2
--  Charlie | 90    | 2    ← 동점
--  David   | 85    | 3    ← 연속
--  Eve     | 80    | 4
```

### ROW_NUMBER vs RANK vs DENSE_RANK

```sql
SELECT
  name,
  score,
  ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,
  RANK() OVER (ORDER BY score DESC) AS rank,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM students;

-- 결과:
--   name   | score | row_num | rank | dense_rank
-- ---------+-------+---------+------+------------
--  Alice   | 95    | 1       | 1    | 1
--  Bob     | 90    | 2       | 2    | 2
--  Charlie | 90    | 3       | 2    | 2
--  David   | 85    | 4       | 4    | 3
--  Eve     | 80    | 5       | 5    | 4

-- ROW_NUMBER: 항상 고유 (1,2,3,4,5)
-- RANK: 동점 같은 순위, 다음 건너뜀 (1,2,2,4,5)
-- DENSE_RANK: 동점 같은 순위, 연속 (1,2,2,3,4)
```

---

## 11.4 집계 윈도우 함수

### SUM OVER

```sql
-- 누적 합계
SELECT
  order_date,
  daily_sales,
  SUM(daily_sales) OVER (ORDER BY order_date) AS cumulative_sales
FROM (
  SELECT
    DATE(created_at) AS order_date,
    SUM(total_amount) AS daily_sales
  FROM orders
  GROUP BY DATE(created_at)
) daily;

-- 결과:
--  order_date | daily_sales | cumulative_sales
-- ------------+-------------+------------------
--  2024-01-01 | 1000        | 1000
--  2024-01-02 | 1500        | 2500
--  2024-01-03 | 2000        | 4500
```

### AVG OVER

```sql
-- 이동 평균 (7일)
SELECT
  order_date,
  daily_sales,
  AVG(daily_sales) OVER (
    ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7d
FROM daily_sales
ORDER BY order_date DESC;
```

### COUNT OVER

```sql
-- 현재까지 행 수
SELECT
  name,
  created_at,
  COUNT(*) OVER (ORDER BY created_at) AS running_count
FROM users
ORDER BY created_at;
```

---

## 11.5 LAG / LEAD

### LAG (이전 행)

```sql
-- 전일 대비
SELECT
  order_date,
  daily_sales,
  LAG(daily_sales) OVER (ORDER BY order_date) AS prev_day_sales,
  daily_sales - LAG(daily_sales) OVER (ORDER BY order_date) AS diff
FROM daily_sales;

-- 결과:
--  order_date | daily_sales | prev_day_sales | diff
-- ------------+-------------+----------------+------
--  2024-01-01 | 1000        | NULL           | NULL
--  2024-01-02 | 1500        | 1000           | 500
--  2024-01-03 | 2000        | 1500           | 500
```

### LEAD (다음 행)

```sql
-- 다음 행 값
SELECT
  order_date,
  daily_sales,
  LEAD(daily_sales) OVER (ORDER BY order_date) AS next_day_sales
FROM daily_sales;

-- 결과:
--  order_date | daily_sales | next_day_sales
-- ------------+-------------+----------------
--  2024-01-01 | 1000        | 1500
--  2024-01-02 | 1500        | 2000
--  2024-01-03 | 2000        | NULL
```

### 오프셋과 기본값

```sql
-- 2행 전, 기본값 0
SELECT
  order_date,
  daily_sales,
  LAG(daily_sales, 2, 0) OVER (ORDER BY order_date) AS two_days_ago
FROM daily_sales;

-- LAG(column, offset, default)
-- offset: 몇 행 전/후 (기본 1)
-- default: NULL일 때 기본값
```

---

## 11.6 FIRST_VALUE / LAST_VALUE

### FIRST_VALUE

```sql
-- 그룹 내 첫 번째 값
SELECT
  category_id,
  title,
  view_count,
  FIRST_VALUE(title) OVER (
    PARTITION BY category_id
    ORDER BY view_count DESC
  ) AS most_viewed_post
FROM posts;

-- 각 카테고리에서 가장 조회수 높은 게시글 이름
```

### LAST_VALUE

```sql
-- 그룹 내 마지막 값
SELECT
  category_id,
  title,
  view_count,
  LAST_VALUE(title) OVER (
    PARTITION BY category_id
    ORDER BY view_count DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS least_viewed_post
FROM posts;

-- ⚠️ ROWS BETWEEN ... 필수!
-- 기본값은 CURRENT ROW까지만
```

### NTH_VALUE

```sql
-- N번째 값
SELECT
  category_id,
  title,
  view_count,
  NTH_VALUE(title, 2) OVER (
    PARTITION BY category_id
    ORDER BY view_count DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS second_most_viewed
FROM posts;
```

---

## 11.7 NTILE

### 분위수

```sql
-- 4분위수
SELECT
  name,
  salary,
  NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;

-- 결과:
--   name   | salary | quartile
-- ---------+--------+----------
--  Alice   | 100000 | 1        ← 상위 25%
--  Bob     | 90000  | 1
--  Charlie | 80000  | 2        ← 25-50%
--  David   | 70000  | 2
--  Eve     | 60000  | 3        ← 50-75%
--  Frank   | 50000  | 3
--  Grace   | 40000  | 4        ← 하위 25%
--  Henry   | 30000  | 4
```

### 백분위수

```sql
-- 10분위수 (상위 10%, 20%, ...)
SELECT
  name,
  score,
  NTILE(10) OVER (ORDER BY score DESC) AS decile
FROM students;

-- decile = 1: 상위 10%
-- decile = 2: 상위 11-20%
-- ...
```

---

## 11.8 Window Frame

### ROWS vs RANGE

```sql
-- ROWS: 물리적 행
-- RANGE: 논리적 값 범위

-- 예시 데이터:
--  order_date | sales
-- ------------+-------
--  2024-01-01 | 100
--  2024-01-02 | 150
--  2024-01-02 | 200   ← 같은 날짜
--  2024-01-03 | 180

-- ROWS: 물리적 3행
SELECT
  order_date,
  sales,
  SUM(sales) OVER (
    ORDER BY order_date
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  ) AS sum_3rows
FROM orders;

-- RANGE: 같은 날짜 모두 포함
SELECT
  order_date,
  sales,
  SUM(sales) OVER (
    ORDER BY order_date
    RANGE BETWEEN INTERVAL '2 days' PRECEDING AND CURRENT ROW
  ) AS sum_3days
FROM orders;
```

### Frame 지정 방법

```sql
-- 전체 파티션
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- 처음부터 현재까지
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 이전 N행 ~ 현재
ROWS BETWEEN N PRECEDING AND CURRENT ROW

-- 현재 ~ 다음 N행
ROWS BETWEEN CURRENT ROW AND N FOLLOWING

-- 이전 N행 ~ 다음 M행
ROWS BETWEEN N PRECEDING AND M FOLLOWING

-- 기본값 (ORDER BY 있을 때)
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

---

## 11.9 실전 예제

### 순위 매기기

```sql
-- 월별 매출 TOP 사용자
WITH monthly_sales AS (
  SELECT
    DATE_TRUNC('month', order_date) AS month,
    user_id,
    SUM(total_amount) AS total_sales
  FROM orders
  WHERE status = 'completed'
  GROUP BY DATE_TRUNC('month', order_date), user_id
)
SELECT
  month,
  user_id,
  total_sales,
  RANK() OVER (PARTITION BY month ORDER BY total_sales DESC) AS sales_rank
FROM monthly_sales
WHERE RANK() OVER (PARTITION BY month ORDER BY total_sales DESC) <= 10
ORDER BY month DESC, sales_rank;
```

### 증감률 계산

```sql
-- 전월 대비 매출 증감률
WITH monthly_revenue AS (
  SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(total_amount) AS revenue
  FROM orders
  WHERE status = 'completed'
  GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
  ROUND(
    (revenue - LAG(revenue) OVER (ORDER BY month))
    / LAG(revenue) OVER (ORDER BY month) * 100,
    2
  ) AS growth_rate
FROM monthly_revenue
ORDER BY month DESC;
```

### 누적 통계

```sql
-- 일별 가입자 + 누적
WITH daily_signups AS (
  SELECT
    DATE(created_at) AS signup_date,
    COUNT(*) AS new_users
  FROM users
  GROUP BY DATE(created_at)
)
SELECT
  signup_date,
  new_users,
  SUM(new_users) OVER (ORDER BY signup_date) AS cumulative_users,
  AVG(new_users) OVER (
    ORDER BY signup_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7d
FROM daily_signups
ORDER BY signup_date DESC;
```

### Gap 분석

```sql
-- 구매 간격
SELECT
  user_id,
  order_date,
  LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS prev_order_date,
  order_date - LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS days_since_last_order
FROM orders
ORDER BY user_id, order_date;
```

---

## 11.10 페이지네이션

### OFFSET vs Window Function

```sql
-- ❌ OFFSET (느림, 페이지 커질수록)
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10 OFFSET 1000;
-- 1010개 행 스캔 후 1000개 버림

-- ✅ Window Function + WHERE (빠름)
WITH numbered AS (
  SELECT
    *,
    ROW_NUMBER() OVER (ORDER BY created_at DESC) AS rn
  FROM posts
)
SELECT * FROM numbered
WHERE rn BETWEEN 1001 AND 1010;
-- 필요한 행만 가져옴 (인덱스 활용)
```

---

## 11.11 중복 제거

### DISTINCT ON vs ROW_NUMBER

```sql
-- PostgreSQL 전용: DISTINCT ON
SELECT DISTINCT ON (user_id)
  user_id,
  order_date,
  total_amount
FROM orders
ORDER BY user_id, order_date DESC;
-- 사용자별 최근 주문 1개

-- 표준 SQL: ROW_NUMBER
SELECT *
FROM (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
  FROM orders
) ranked
WHERE rn = 1;
```

---

## 11.12 성능 최적화

### 인덱스

```sql
-- ORDER BY 컬럼에 인덱스
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);

-- PARTITION BY + ORDER BY
CREATE INDEX idx_posts_category_views ON posts(category_id, view_count DESC);
```

### 윈도우 함수 재사용

```sql
-- ❌ 비효율: 같은 윈도우 함수 반복
SELECT
  name,
  score,
  RANK() OVER (ORDER BY score DESC) AS rank,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank,
  score - AVG(score) OVER (ORDER BY score DESC) AS diff_from_avg
FROM students;

-- ✅ 효율: WINDOW 절 사용
SELECT
  name,
  score,
  RANK() OVER w AS rank,
  DENSE_RANK() OVER w AS dense_rank,
  score - AVG(score) OVER w AS diff_from_avg
FROM students
WINDOW w AS (ORDER BY score DESC);
```

---

## 11.13 복잡한 예제

### 연속 이벤트 찾기

```sql
-- 3일 연속 로그인 사용자
WITH daily_logins AS (
  SELECT DISTINCT
    user_id,
    DATE(login_at) AS login_date
  FROM login_logs
),
streaks AS (
  SELECT
    user_id,
    login_date,
    login_date - ROW_NUMBER() OVER (
      PARTITION BY user_id ORDER BY login_date
    )::INT AS streak_group
  FROM daily_logins
)
SELECT
  user_id,
  MIN(login_date) AS streak_start,
  MAX(login_date) AS streak_end,
  COUNT(*) AS streak_length
FROM streaks
GROUP BY user_id, streak_group
HAVING COUNT(*) >= 3
ORDER BY streak_length DESC;
```

### Funnel 분석

```sql
-- 전환 퍼널 (방문 → 가입 → 구매)
WITH events AS (
  SELECT
    user_id,
    event_type,
    created_at,
    LEAD(event_type) OVER (PARTITION BY user_id ORDER BY created_at) AS next_event,
    LEAD(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS next_event_time
  FROM user_events
  WHERE event_type IN ('visit', 'signup', 'purchase')
)
SELECT
  event_type,
  next_event,
  COUNT(*) AS count,
  AVG(EXTRACT(EPOCH FROM (next_event_time - created_at))) AS avg_time_seconds
FROM events
WHERE next_event IS NOT NULL
GROUP BY event_type, next_event
ORDER BY event_type;
```

---

## 핵심 요약

### 순위 함수
```sql
ROW_NUMBER()    -- 고유 순번 (1,2,3,4,5)
RANK()          -- 동점 처리, 건너뜀 (1,2,2,4,5)
DENSE_RANK()    -- 동점 처리, 연속 (1,2,2,3,4)
NTILE(n)        -- n분위수
```

### 값 참조
```sql
LAG(col, n)            -- n행 이전
LEAD(col, n)           -- n행 다음
FIRST_VALUE(col)       -- 첫 번째
LAST_VALUE(col)        -- 마지막
NTH_VALUE(col, n)      -- n번째
```

### 집계
```sql
SUM() OVER (...)       -- 누적 합계
AVG() OVER (...)       -- 이동 평균
COUNT() OVER (...)     -- 누적 개수
```

### 기본 문법
```sql
함수() OVER (
  PARTITION BY column    -- 그룹화
  ORDER BY column        -- 정렬
  ROWS BETWEEN ...       -- 범위
)
```

---

## 다음 단계

Window Function을 익혔으니, 이제 **CTE (Common Table Expression)**를 배워봅시다!

👉 [Chapter 12. CTE](12-CTE.md)

---

## 실습 과제

### 과제 1: 순위

```sql
-- 1. 점수 높은 순 등수 (RANK)
-- 2. 카테고리별 TOP 3 (ROW_NUMBER)
-- 3. 상위 25% (NTILE)
```

### 과제 2: 증감 분석

```sql
-- 1. 일별 가입자 + 누적
-- 2. 전일 대비 증감
-- 3. 7일 이동 평균
```

### 과제 3: 실전

```sql
-- 1. 사용자별 최근 주문
-- 2. 월별 매출 + 전월 대비 증감률
-- 3. 연속 3일 로그인 사용자
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!