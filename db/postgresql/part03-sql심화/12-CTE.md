# Chapter 12. CTE (Common Table Expression)

## 12.1 CTE란?

### 정의

```
CTE (Common Table Expression):
WITH 절로 정의하는 임시 결과 집합

비유: 수학 문제의 중간 계산
┌────────────────────────────────┐
│ x = 5 + 3        (중간 계산)   │
│ y = x × 2        (x 재사용)    │
│ z = y + 10       (최종 답)     │
│                                │
│ CTE = x, y 같은 중간 결과      │
└────────────────────────────────┘
```

### 기본 문법

```sql
WITH cte_name AS (
  SELECT ...
)
SELECT * FROM cte_name;
```

---

## 12.2 단일 CTE

### 기본 예제

```sql
-- 평균 나이보다 많은 사용자
WITH avg_age_cte AS (
  SELECT AVG(age) AS avg_age
  FROM users
)
SELECT
  u.name,
  u.age,
  a.avg_age
FROM users u, avg_age_cte a
WHERE u.age > a.avg_age;
```

### CTE vs 서브쿼리

```sql
-- 서브쿼리 (읽기 어려움)
SELECT name, age
FROM users
WHERE age > (SELECT AVG(age) FROM users);

-- CTE (읽기 쉬움)
WITH avg_age AS (
  SELECT AVG(age) AS value FROM users
)
SELECT name, age
FROM users, avg_age
WHERE age > avg_age.value;
```

---

## 12.3 다중 CTE

### 여러 CTE 정의

```sql
WITH
  -- CTE 1: 사용자별 게시글 수
  user_posts AS (
    SELECT
      user_id,
      COUNT(*) AS post_count
    FROM posts
    GROUP BY user_id
  ),
  -- CTE 2: 활성 사용자
  active_users AS (
    SELECT user_id
    FROM user_posts
    WHERE post_count >= 5
  )
-- 메인 쿼리
SELECT
  u.name,
  up.post_count
FROM users u
INNER JOIN user_posts up ON u.id = up.user_id
WHERE u.id IN (SELECT user_id FROM active_users)
ORDER BY up.post_count DESC;
```

### CTE 체인

```sql
WITH
  -- 1단계: 일별 매출
  daily_sales AS (
    SELECT
      DATE(order_date) AS date,
      SUM(total_amount) AS sales
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE(order_date)
  ),
  -- 2단계: 이동 평균 (daily_sales 사용)
  sales_with_avg AS (
    SELECT
      date,
      sales,
      AVG(sales) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
      ) AS moving_avg_7d
    FROM daily_sales
  )
-- 3단계: 평균 이상만 조회
SELECT *
FROM sales_with_avg
WHERE sales > moving_avg_7d
ORDER BY date DESC;
```

---

## 12.4 재귀 CTE

### 기본 구조

```sql
WITH RECURSIVE cte_name AS (
  -- 1. 기본 케이스 (Anchor)
  SELECT ...

  UNION ALL

  -- 2. 재귀 케이스 (Recursive)
  SELECT ...
  FROM cte_name  -- 자기 자신 참조
  WHERE 종료_조건
)
SELECT * FROM cte_name;
```

### 숫자 생성

```sql
-- 1부터 10까지
WITH RECURSIVE numbers AS (
  -- 기본: 1
  SELECT 1 AS n

  UNION ALL

  -- 재귀: n + 1
  SELECT n + 1
  FROM numbers
  WHERE n < 10
)
SELECT * FROM numbers;

-- 결과: 1, 2, 3, ..., 10
```

### 날짜 범위 생성

```sql
-- 2024년 1월 전체
WITH RECURSIVE dates AS (
  SELECT '2024-01-01'::DATE AS date

  UNION ALL

  SELECT date + 1
  FROM dates
  WHERE date < '2024-01-31'
)
SELECT * FROM dates;

-- 더 간단한 방법: generate_series
SELECT generate_series(
  '2024-01-01'::DATE,
  '2024-01-31'::DATE,
  '1 day'::INTERVAL
)::DATE AS date;
```

---

## 12.5 계층 구조 (조직도)

### 하향식 (Top-Down)

```sql
-- CEO부터 모든 하위 직원
WITH RECURSIVE org_tree AS (
  -- 기본: CEO
  SELECT
    id,
    name,
    manager_id,
    1 AS level,
    name AS path
  FROM employees
  WHERE manager_id IS NULL  -- CEO

  UNION ALL

  -- 재귀: 하위 직원
  SELECT
    e.id,
    e.name,
    e.manager_id,
    o.level + 1,
    o.path || ' > ' || e.name
  FROM employees e
  INNER JOIN org_tree o ON e.manager_id = o.id
)
SELECT
  REPEAT('  ', level - 1) || name AS org_chart,
  level,
  path
FROM org_tree
ORDER BY path;

-- 결과:
-- org_chart      | level | path
-- ---------------+-------+------------------
-- CEO            | 1     | CEO
--   CTO          | 2     | CEO > CTO
--     개발팀장    | 3     | CEO > CTO > 개발팀장
--       개발자1   | 4     | CEO > CTO > 개발팀장 > 개발자1
```

### 상향식 (Bottom-Up)

```sql
-- 특정 직원부터 CEO까지
WITH RECURSIVE manager_chain AS (
  -- 기본: 개발자1
  SELECT
    id,
    name,
    manager_id,
    1 AS level
  FROM employees
  WHERE id = 5  -- 개발자1

  UNION ALL

  -- 재귀: 상위 매니저
  SELECT
    e.id,
    e.name,
    e.manager_id,
    m.level + 1
  FROM employees e
  INNER JOIN manager_chain m ON e.id = m.manager_id
)
SELECT * FROM manager_chain
ORDER BY level DESC;

-- 결과: CEO → CTO → 개발팀장 → 개발자1
```

---

## 12.6 그래프 탐색

### 친구 관계 (SNS)

```sql
-- Alice의 친구 + 친구의 친구
WITH RECURSIVE friend_network AS (
  -- 기본: 직접 친구
  SELECT
    user_id,
    friend_id,
    1 AS degree
  FROM friendships
  WHERE user_id = 1  -- Alice

  UNION

  -- 재귀: 친구의 친구
  SELECT
    f.user_id,
    fs.friend_id,
    fn.degree + 1
  FROM friend_network fn
  INNER JOIN friendships f ON fn.friend_id = f.user_id
  INNER JOIN friendships fs ON f.user_id = fs.user_id
  WHERE fn.degree < 3  -- 최대 3단계
)
SELECT DISTINCT
  u.name,
  fn.degree
FROM friend_network fn
INNER JOIN users u ON fn.friend_id = u.id
ORDER BY fn.degree, u.name;
```

### 무한 루프 방지

```sql
-- 순환 참조 방지
WITH RECURSIVE paths AS (
  SELECT
    id,
    name,
    manager_id,
    ARRAY[id] AS path  -- 경로 저장
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  SELECT
    e.id,
    e.name,
    e.manager_id,
    p.path || e.id
  FROM employees e
  INNER JOIN paths p ON e.manager_id = p.id
  WHERE NOT (e.id = ANY(p.path))  -- 순환 방지
)
SELECT * FROM paths;
```

---

## 12.7 실전 예제

### 카테고리 트리

```sql
-- 계층형 카테고리
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  parent_id INT REFERENCES categories(id)
);

INSERT INTO categories (name, parent_id) VALUES
  ('전자제품', NULL),
  ('컴퓨터', 1),
  ('노트북', 2),
  ('데스크톱', 2),
  ('가전', NULL);

-- 카테고리 계층 조회
WITH RECURSIVE category_tree AS (
  -- 최상위 카테고리
  SELECT
    id,
    name,
    parent_id,
    1 AS level,
    name AS full_path
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  -- 하위 카테고리
  SELECT
    c.id,
    c.name,
    c.parent_id,
    ct.level + 1,
    ct.full_path || ' > ' || c.name
  FROM categories c
  INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT
  REPEAT('--', level - 1) || name AS hierarchy,
  full_path
FROM category_tree
ORDER BY full_path;

-- 결과:
-- hierarchy     | full_path
-- --------------+-------------------------
-- 전자제품      | 전자제품
-- --컴퓨터      | 전자제품 > 컴퓨터
-- ----노트북    | 전자제품 > 컴퓨터 > 노트북
-- ----데스크톱  | 전자제품 > 컴퓨터 > 데스크톱
-- 가전          | 가전
```

### 댓글 트리

```sql
-- 대댓글 계층
WITH RECURSIVE comment_tree AS (
  -- 최상위 댓글
  SELECT
    id,
    parent_id,
    content,
    user_id,
    1 AS level
  FROM comments
  WHERE parent_id IS NULL
    AND post_id = 1

  UNION ALL

  -- 대댓글
  SELECT
    c.id,
    c.parent_id,
    c.content,
    c.user_id,
    ct.level + 1
  FROM comments c
  INNER JOIN comment_tree ct ON c.parent_id = ct.id
)
SELECT
  REPEAT('  ', level - 1) || content AS thread,
  level
FROM comment_tree
ORDER BY id;
```

---

## 12.8 데이터 생성

### 테스트 데이터

```sql
-- 100개 샘플 사용자 생성
WITH RECURSIVE user_generator AS (
  SELECT
    1 AS id,
    'user1@example.com' AS email,
    'User 1' AS name

  UNION ALL

  SELECT
    id + 1,
    'user' || (id + 1) || '@example.com',
    'User ' || (id + 1)
  FROM user_generator
  WHERE id < 100
)
INSERT INTO users (email, name)
SELECT email, name FROM user_generator;
```

### 시계열 데이터

```sql
-- 최근 30일 일별 통계 (데이터 없는 날도 표시)
WITH RECURSIVE dates AS (
  SELECT CURRENT_DATE - 30 AS date
  UNION ALL
  SELECT date + 1
  FROM dates
  WHERE date < CURRENT_DATE
)
SELECT
  d.date,
  COALESCE(COUNT(o.id), 0) AS order_count,
  COALESCE(SUM(o.total_amount), 0) AS total_sales
FROM dates d
LEFT JOIN orders o ON DATE(o.created_at) = d.date
GROUP BY d.date
ORDER BY d.date DESC;
```

---

## 12.9 복잡한 분석

### Funnel 분석

```sql
WITH
  -- 1단계: 이벤트별 사용자 수
  event_users AS (
    SELECT
      event_type,
      COUNT(DISTINCT user_id) AS user_count
    FROM user_events
    WHERE created_at >= '2024-01-01'
    GROUP BY event_type
  ),
  -- 2단계: 퍼널 순서
  funnel_order AS (
    VALUES
      ('page_view', 1),
      ('signup', 2),
      ('add_to_cart', 3),
      ('checkout', 4),
      ('purchase', 5)
  )
-- 3단계: 전환율 계산
SELECT
  fo.column1 AS event,
  eu.user_count,
  LAG(eu.user_count) OVER (ORDER BY fo.column2) AS prev_count,
  ROUND(
    eu.user_count::NUMERIC
    / LAG(eu.user_count) OVER (ORDER BY fo.column2) * 100,
    2
  ) AS conversion_rate
FROM funnel_order fo
LEFT JOIN event_users eu ON fo.column1 = eu.event_type
ORDER BY fo.column2;
```

### 코호트 분석

```sql
WITH
  -- 가입 월 코호트
  cohorts AS (
    SELECT
      id,
      DATE_TRUNC('month', created_at) AS cohort_month
    FROM users
  ),
  -- 월별 활동
  monthly_activity AS (
    SELECT
      user_id,
      DATE_TRUNC('month', login_at) AS activity_month
    FROM login_logs
  )
SELECT
  c.cohort_month,
  COUNT(DISTINCT c.id) AS cohort_size,
  COUNT(DISTINCT CASE
    WHEN ma.activity_month >= NOW() - INTERVAL '30 days' THEN ma.user_id
  END) AS active_users,
  ROUND(
    COUNT(DISTINCT CASE
      WHEN ma.activity_month >= NOW() - INTERVAL '30 days' THEN ma.user_id
    END)::NUMERIC / COUNT(DISTINCT c.id) * 100,
    2
  ) AS retention_rate
FROM cohorts c
LEFT JOIN monthly_activity ma ON c.id = ma.user_id
GROUP BY c.cohort_month
ORDER BY c.cohort_month DESC;
```

---

## 12.10 성능 최적화

### MATERIALIZED

```sql
-- 일반 CTE (매번 실행)
WITH expensive_query AS (
  SELECT ... -- 복잡한 쿼리
)
SELECT * FROM expensive_query
UNION ALL
SELECT * FROM expensive_query;
-- expensive_query 2번 실행!

-- MATERIALIZED (한 번만 실행, 임시 저장)
WITH expensive_query AS MATERIALIZED (
  SELECT ... -- 복잡한 쿼리
)
SELECT * FROM expensive_query
UNION ALL
SELECT * FROM expensive_query;
-- expensive_query 1번 실행, 결과 재사용
```

### 재귀 깊이 제한

```sql
-- 기본 최대 깊이: 100
-- 변경
SET max_stack_depth = '2MB';

-- 또는 쿼리에서 제한
WITH RECURSIVE tree AS (
  SELECT id, parent_id, 1 AS level
  FROM nodes
  WHERE parent_id IS NULL

  UNION ALL

  SELECT n.id, n.parent_id, t.level + 1
  FROM nodes n
  INNER JOIN tree t ON n.parent_id = t.id
  WHERE t.level < 10  -- 최대 10단계
)
SELECT * FROM tree;
```

---

## 12.11 CTE vs 서브쿼리 vs 임시 테이블

### 비교

```sql
-- CTE
WITH temp_data AS (
  SELECT * FROM large_table WHERE condition
)
SELECT * FROM temp_data;
-- 장점: 읽기 쉬움, 재사용 가능
-- 단점: 인덱스 없음

-- 서브쿼리
SELECT *
FROM (
  SELECT * FROM large_table WHERE condition
) AS temp_data;
-- 장점: 표준 SQL
-- 단점: 재사용 불가

-- 임시 테이블
CREATE TEMP TABLE temp_data AS
SELECT * FROM large_table WHERE condition;

CREATE INDEX idx_temp ON temp_data(column);

SELECT * FROM temp_data;
-- 장점: 인덱스 가능, 빠름
-- 단점: 관리 필요
```

### 선택 기준

```
CTE 사용:
✅ 가독성 중요
✅ 한 쿼리 내에서만 사용
✅ 재귀 필요
✅ 중간 크기 데이터

임시 테이블 사용:
✅ 대용량 데이터
✅ 여러 쿼리에서 재사용
✅ 인덱스 필요
✅ 복잡한 변환
```

---

## 핵심 요약

### 기본 CTE
```sql
WITH cte_name AS (
  SELECT ...
)
SELECT * FROM cte_name;
```

### 다중 CTE
```sql
WITH
  cte1 AS (...),
  cte2 AS (...)
SELECT * FROM cte1, cte2;
```

### 재귀 CTE
```sql
WITH RECURSIVE cte AS (
  SELECT ... -- 기본
  UNION ALL
  SELECT ... FROM cte WHERE ... -- 재귀
)
SELECT * FROM cte;
```

### 장점
```
- 가독성 향상
- 재사용 가능
- 복잡한 쿼리 단순화
- 재귀 쿼리 지원
```

---

## 다음 단계

CTE를 마스터했으니, 이제 **제약조건 (Constraints)**을 배워봅시다!

👉 [Chapter 13. 제약조건](../part04-제약조건과-인덱스/13-제약조건.md)

---

## 실습 과제

### 과제 1: 기본 CTE

```sql
-- 1. 평균 나이보다 많은 사용자
-- 2. 활성 사용자 + 게시글 수
-- 3. 다중 CTE 사용
```

### 과제 2: 재귀 CTE

```sql
-- 1. 1부터 100까지 숫자
-- 2. 조직도 계층 조회
-- 3. 카테고리 트리
```

### 과제 3: 실전 분석

```sql
-- 1. 일별 매출 + 이동 평균
-- 2. 코호트 분석
-- 3. 댓글 트리 출력
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 2-3시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!