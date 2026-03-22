# Chapter 9. 집계 함수와 GROUP BY

## 9.1 집계 함수란?

### 정의

```
집계 함수 (Aggregate Function):
여러 행을 하나의 결과로 요약

비유: 반 전체 평균 점수
┌────────────────────────────────┐
│ 학생 개별 점수 → 여러 행       │
│ 85, 90, 78, 95, 88            │
│                               │
│ 평균 점수 → 하나의 값         │
│ 87.2                          │
└────────────────────────────────┘
```

### 주요 집계 함수

```sql
COUNT()   -- 개수
SUM()     -- 합계
AVG()     -- 평균
MIN()     -- 최소값
MAX()     -- 최대값
```

---

## 9.2 COUNT

### 기본 사용

```sql
-- 전체 행 수
SELECT COUNT(*) FROM users;
-- 10

-- 특정 컬럼 (NULL 제외)
SELECT COUNT(age) FROM users;
-- age가 NULL인 행은 제외

-- NULL 포함 확인
SELECT
  COUNT(*) AS total,
  COUNT(age) AS age_count,
  COUNT(*) - COUNT(age) AS age_null_count
FROM users;
```

### DISTINCT

```sql
-- 중복 제거 개수
SELECT COUNT(DISTINCT country) FROM users;
-- 국가 종류 개수

SELECT COUNT(DISTINCT age) FROM users;
-- 나이 종류 개수

-- 여러 컬럼 조합
SELECT COUNT(DISTINCT (country, city)) FROM users;
```

### 조건부 COUNT

```sql
-- WHERE
SELECT COUNT(*) FROM users WHERE age > 30;

-- CASE
SELECT
  COUNT(CASE WHEN age < 20 THEN 1 END) AS under_20,
  COUNT(CASE WHEN age >= 20 AND age < 30 THEN 1 END) AS twenties,
  COUNT(CASE WHEN age >= 30 THEN 1 END) AS over_30
FROM users;
```

---

## 9.3 SUM

### 기본 사용

```sql
-- 합계
SELECT SUM(price) FROM orders;

-- 특정 조건
SELECT SUM(price) FROM orders WHERE status = 'completed';

-- NULL 처리
SELECT SUM(COALESCE(price, 0)) FROM orders;
-- NULL을 0으로 처리
```

### 계산된 값 합계

```sql
-- 수량 × 가격
SELECT SUM(quantity * price) AS total_revenue
FROM order_items;

-- 조건부 합계
SELECT
  SUM(CASE WHEN status = 'completed' THEN price ELSE 0 END) AS completed_sum,
  SUM(CASE WHEN status = 'pending' THEN price ELSE 0 END) AS pending_sum
FROM orders;
```

---

## 9.4 AVG

### 기본 사용

```sql
-- 평균
SELECT AVG(age) FROM users;
-- 27.5

-- 소수점 자릿수
SELECT ROUND(AVG(age), 2) AS avg_age FROM users;
-- 27.50

-- NULL 제외 주의!
SELECT
  AVG(rating) AS avg_with_null,
  AVG(COALESCE(rating, 0)) AS avg_with_zero
FROM products;
```

### 그룹별 평균

```sql
-- 국가별 평균 나이
SELECT
  country,
  ROUND(AVG(age), 1) AS avg_age
FROM users
GROUP BY country;
```

---

## 9.5 MIN / MAX

### 기본 사용

```sql
-- 최소값
SELECT MIN(age) FROM users;
-- 18

-- 최대값
SELECT MAX(age) FROM users;
-- 65

-- 날짜
SELECT
  MIN(created_at) AS first_signup,
  MAX(created_at) AS last_signup
FROM users;
```

### 문자열

```sql
-- 알파벳 순서
SELECT
  MIN(name) AS first_name,
  MAX(name) AS last_name
FROM users;
-- first_name: Alice
-- last_name: Zoe
```

---

## 9.6 GROUP BY

### 기본 문법

```sql
SELECT column, aggregate_function(column)
FROM table
GROUP BY column;
```

### 단일 컬럼 그룹화

```sql
-- 국가별 사용자 수
SELECT
  country,
  COUNT(*) AS user_count
FROM users
GROUP BY country;

-- 결과:
--  country | user_count
-- ---------+------------
--  Korea   | 5
--  Japan   | 3
--  USA     | 2
```

### 여러 컬럼 그룹화

```sql
-- 국가 + 성별
SELECT
  country,
  gender,
  COUNT(*) AS user_count
FROM users
GROUP BY country, gender
ORDER BY country, gender;

-- 결과:
--  country | gender | user_count
-- ---------+--------+------------
--  Korea   | F      | 3
--  Korea   | M      | 2
--  Japan   | F      | 1
--  Japan   | M      | 2
```

### 여러 집계 함수

```sql
SELECT
  country,
  COUNT(*) AS user_count,
  AVG(age) AS avg_age,
  MIN(age) AS min_age,
  MAX(age) AS max_age
FROM users
GROUP BY country
ORDER BY user_count DESC;
```

---

## 9.7 HAVING

### WHERE vs HAVING

```
WHERE:  그룹화 전 필터링
HAVING: 그룹화 후 필터링
```

### 기본 사용

```sql
-- 사용자 3명 이상인 국가만
SELECT
  country,
  COUNT(*) AS user_count
FROM users
GROUP BY country
HAVING COUNT(*) >= 3
ORDER BY user_count DESC;
```

### WHERE + HAVING

```sql
-- 20세 이상 중에서, 평균 나이 30세 이상인 국가
SELECT
  country,
  COUNT(*) AS user_count,
  ROUND(AVG(age), 1) AS avg_age
FROM users
WHERE age >= 20              -- 그룹화 전 필터
GROUP BY country
HAVING AVG(age) >= 30        -- 그룹화 후 필터
ORDER BY avg_age DESC;
```

### 복잡한 HAVING

```sql
-- 여러 조건
SELECT
  country,
  COUNT(*) AS user_count,
  AVG(age) AS avg_age
FROM users
GROUP BY country
HAVING COUNT(*) >= 5
   AND AVG(age) > 25
   AND MIN(age) >= 18;
```

---

## 9.8 실전 예제

### 사용자 통계

```sql
-- 국가별 통계
SELECT
  country,
  COUNT(*) AS total_users,
  COUNT(CASE WHEN gender = 'M' THEN 1 END) AS male_count,
  COUNT(CASE WHEN gender = 'F' THEN 1 END) AS female_count,
  ROUND(AVG(age), 1) AS avg_age,
  MIN(created_at) AS first_signup,
  MAX(created_at) AS last_signup
FROM users
GROUP BY country
ORDER BY total_users DESC;
```

### 매출 통계

```sql
-- 월별 매출
SELECT
  DATE_TRUNC('month', order_date) AS month,
  COUNT(*) AS order_count,
  SUM(total_amount) AS total_revenue,
  ROUND(AVG(total_amount), 2) AS avg_order_value,
  MAX(total_amount) AS max_order
FROM orders
WHERE status = 'completed'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month DESC;
```

### 상품 인기도

```sql
-- 판매량 TOP 10
SELECT
  p.name AS product_name,
  COUNT(oi.id) AS order_count,
  SUM(oi.quantity) AS total_sold,
  SUM(oi.quantity * oi.price) AS total_revenue
FROM products p
INNER JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id, p.name
ORDER BY total_sold DESC
LIMIT 10;
```

---

## 9.9 GROUP BY 고급

### ROLLUP

```sql
-- 소계와 총계 추가
SELECT
  country,
  city,
  COUNT(*) AS user_count
FROM users
GROUP BY ROLLUP(country, city)
ORDER BY country NULLS FIRST, city NULLS FIRST;

-- 결과:
--  country | city   | user_count
-- ---------+--------+------------
--  NULL    | NULL   | 10         ← 전체 총계
--  Korea   | NULL   | 5          ← 한국 소계
--  Korea   | Seoul  | 3
--  Korea   | Busan  | 2
--  Japan   | NULL   | 3          ← 일본 소계
--  Japan   | Tokyo  | 2
--  Japan   | Osaka  | 1
```

### CUBE

```sql
-- 모든 조합의 소계
SELECT
  country,
  gender,
  COUNT(*) AS user_count
FROM users
GROUP BY CUBE(country, gender)
ORDER BY country NULLS FIRST, gender NULLS FIRST;

-- 결과:
--  country | gender | user_count
-- ---------+--------+------------
--  NULL    | NULL   | 10         ← 전체
--  NULL    | F      | 5          ← 여성 전체
--  NULL    | M      | 5          ← 남성 전체
--  Korea   | NULL   | 5          ← 한국 전체
--  Korea   | F      | 3          ← 한국 여성
--  Korea   | M      | 2          ← 한국 남성
--  Japan   | NULL   | 3
--  Japan   | F      | 1
--  Japan   | M      | 2
```

### GROUPING SETS

```sql
-- 특정 조합만 그룹화
SELECT
  country,
  city,
  gender,
  COUNT(*) AS user_count
FROM users
GROUP BY GROUPING SETS (
  (country, city),      -- 국가+도시
  (country, gender),    -- 국가+성별
  (gender)              -- 성별만
)
ORDER BY country NULLS LAST, city NULLS LAST, gender NULLS LAST;
```

---

## 9.10 윈도우 함수 미리보기

### 집계 함수 vs 윈도우 함수

```sql
-- 집계 함수: 그룹당 1행
SELECT
  country,
  AVG(age) AS avg_age
FROM users
GROUP BY country;

-- 결과: 2행 (국가당 1행)
--  country | avg_age
-- ---------+---------
--  Korea   | 28.5
--  Japan   | 32.0

-- 윈도우 함수: 모든 행 유지
SELECT
  name,
  country,
  age,
  AVG(age) OVER (PARTITION BY country) AS country_avg_age
FROM users;

-- 결과: 10행 (모든 사용자)
--   name   | country | age | country_avg_age
-- ---------+---------+-----+-----------------
--  Alice   | Korea   | 25  | 28.5
--  Bob     | Korea   | 30  | 28.5
--  Charlie | Japan   | 32  | 32.0
--  ...

-- 자세한 내용은 Chapter 11에서!
```

---

## 9.11 성능 최적화

### 인덱스

```sql
-- GROUP BY 컬럼에 인덱스
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_orders_status ON orders(status);

-- 복합 인덱스
CREATE INDEX idx_users_country_gender ON users(country, gender);
```

### EXPLAIN 분석

```sql
-- 실행 계획 확인
EXPLAIN ANALYZE
SELECT
  country,
  COUNT(*) AS user_count
FROM users
GROUP BY country;

-- HashAggregate vs GroupAggregate
-- 적은 그룹: HashAggregate (빠름)
-- 많은 그룹: GroupAggregate (메모리 절약)
```

### 불필요한 GROUP BY 제거

```sql
-- ❌ 나쁜 예: 불필요한 GROUP BY
SELECT
  user_id,
  COUNT(*)
FROM posts
GROUP BY user_id;
-- user_id마다 1행이면 GROUP BY 불필요

-- ✅ 좋은 예
SELECT
  user_id,
  1 AS post_count
FROM posts;
```

---

## 9.12 함정과 주의사항

### SELECT 절 오류

```sql
-- ❌ 에러: name은 GROUP BY에 없음
SELECT
  country,
  name,              -- 에러!
  COUNT(*) AS user_count
FROM users
GROUP BY country;

-- ✅ 올바른 방법 1: GROUP BY에 추가
SELECT
  country,
  name,
  COUNT(*) AS user_count
FROM users
GROUP BY country, name;

-- ✅ 올바른 방법 2: 집계 함수 사용
SELECT
  country,
  STRING_AGG(name, ', ') AS names,
  COUNT(*) AS user_count
FROM users
GROUP BY country;
```

### HAVING vs WHERE

```sql
-- ❌ 비효율: HAVING으로 일반 필터
SELECT
  country,
  COUNT(*) AS user_count
FROM users
GROUP BY country
HAVING country = 'Korea';
-- 모든 국가를 그룹화한 후 필터 (비효율)

-- ✅ 효율: WHERE로 필터
SELECT
  country,
  COUNT(*) AS user_count
FROM users
WHERE country = 'Korea'
GROUP BY country;
-- 한국만 그룹화 (효율적)
```

### NULL 처리

```sql
-- NULL 그룹
SELECT
  country,
  COUNT(*) AS user_count
FROM users
GROUP BY country;

-- 결과:
--  country | user_count
-- ---------+------------
--  Korea   | 5
--  Japan   | 3
--  NULL    | 2          ← country가 NULL인 사용자들

-- NULL 제외
SELECT
  country,
  COUNT(*) AS user_count
FROM users
WHERE country IS NOT NULL
GROUP BY country;
```

---

## 9.13 문자열 집계

### STRING_AGG

```sql
-- 문자열 연결
SELECT
  country,
  STRING_AGG(name, ', ') AS names
FROM users
GROUP BY country;

-- 결과:
--  country |        names
-- ---------+---------------------
--  Korea   | Alice, Bob, Charlie
--  Japan   | David, Eve

-- 정렬 + 구분자
SELECT
  country,
  STRING_AGG(name, ', ' ORDER BY name) AS names
FROM users
GROUP BY country;
```

### ARRAY_AGG

```sql
-- 배열로 집계
SELECT
  country,
  ARRAY_AGG(name ORDER BY name) AS names
FROM users
GROUP BY country;

-- 결과:
--  country |        names
-- ---------+---------------------
--  Korea   | {Alice,Bob,Charlie}
--  Japan   | {David,Eve}
```

---

## 9.14 복잡한 통계 쿼리

### 일별 가입자 추이

```sql
SELECT
  DATE(created_at) AS signup_date,
  COUNT(*) AS new_users,
  SUM(COUNT(*)) OVER (ORDER BY DATE(created_at)) AS cumulative_users
FROM users
GROUP BY DATE(created_at)
ORDER BY signup_date;
```

### 코호트 분석

```sql
-- 가입 월별 사용자 통계
SELECT
  DATE_TRUNC('month', created_at) AS cohort_month,
  COUNT(*) AS users,
  COUNT(CASE WHEN last_login_at >= NOW() - INTERVAL '7 days' THEN 1 END) AS active_users,
  ROUND(
    COUNT(CASE WHEN last_login_at >= NOW() - INTERVAL '7 days' THEN 1 END)::NUMERIC
    / COUNT(*)::NUMERIC * 100,
    2
  ) AS retention_rate
FROM users
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY cohort_month DESC;
```

### RFM 분석

```sql
-- Recency, Frequency, Monetary
SELECT
  user_id,
  MAX(order_date) AS last_order_date,
  DATE_PART('day', NOW() - MAX(order_date)) AS recency,
  COUNT(*) AS frequency,
  SUM(total_amount) AS monetary
FROM orders
WHERE status = 'completed'
GROUP BY user_id
HAVING COUNT(*) >= 2  -- 2번 이상 구매
ORDER BY monetary DESC;
```

---

## 핵심 요약

### 집계 함수
```sql
COUNT(*)          -- 행 수
COUNT(column)     -- NULL 제외 개수
SUM(column)       -- 합계
AVG(column)       -- 평균
MIN(column)       -- 최소
MAX(column)       -- 최대
```

### GROUP BY
```sql
SELECT column, aggregate_function(column)
FROM table
WHERE condition        -- 그룹화 전 필터
GROUP BY column
HAVING condition       -- 그룹화 후 필터
ORDER BY column;
```

### 주의사항
```sql
-- SELECT 절: GROUP BY 컬럼 또는 집계 함수만
-- WHERE: 그룹화 전
-- HAVING: 그룹화 후
-- NULL은 별도 그룹
```

### 고급 기능
```sql
ROLLUP             -- 소계 + 총계
CUBE               -- 모든 조합
GROUPING SETS      -- 특정 조합
STRING_AGG         -- 문자열 연결
```

---

## 다음 단계

집계 함수와 GROUP BY를 익혔으니, 이제 **서브쿼리**를 배워봅시다!

👉 [Chapter 10. 서브쿼리](10-서브쿼리.md)

---

## 실습 과제

### 과제 1: 기본 집계

```sql
-- 1. 국가별 사용자 수
-- 2. 나이대별 사용자 수 (10대, 20대, ...)
-- 3. 평균 나이가 30세 이상인 국가
```

### 과제 2: 매출 통계

```sql
-- 1. 월별 총 매출
-- 2. 상품별 판매 수량 TOP 10
-- 3. 평균 주문 금액
```

### 과제 3: 복합 통계

```sql
-- 1. 국가+성별 사용자 수
-- 2. 카테고리별 평균 가격
-- 3. 사용자별 주문 횟수 + 총액
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 2-3시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!