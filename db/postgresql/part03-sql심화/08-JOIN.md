# Chapter 8. JOIN

## 8.1 JOIN이란?

### 정의

```
JOIN:
여러 테이블의 데이터를 연결하여 조회

비유: 퍼즐 맞추기
┌────────────────────────────────┐
│ users 테이블    posts 테이블   │
│                                │
│ id | name       id | user_id   │
│ 1  | Alice      1  | 1         │
│ 2  | Bob        2  | 1         │
│                3  | 2         │
│                                │
│ JOIN 결과:                     │
│ name  | post_id                │
│ Alice | 1                      │
│ Alice | 2                      │
│ Bob   | 3                      │
└────────────────────────────────┘
```

### JOIN 종류

```
INNER JOIN:   교집합 (양쪽 모두 있는 데이터)
LEFT JOIN:    왼쪽 테이블 기준
RIGHT JOIN:   오른쪽 테이블 기준
FULL JOIN:    합집합 (양쪽 모두)
CROSS JOIN:   모든 조합
```

---

## 8.2 INNER JOIN

### 기본 문법

```sql
SELECT columns
FROM table1
INNER JOIN table2 ON table1.column = table2.column;
```

### 예제 테이블

```sql
-- users 테이블
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255)
);

INSERT INTO users (name, email) VALUES
  ('Alice', 'alice@example.com'),
  ('Bob', 'bob@example.com'),
  ('Charlie', 'charlie@example.com');

-- posts 테이블
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  title VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO posts (user_id, title) VALUES
  (1, 'Alice의 첫 글'),
  (1, 'Alice의 두 번째 글'),
  (2, 'Bob의 글'),
  (NULL, '작성자 없는 글');  -- user_id가 NULL
```

### INNER JOIN 실행

```sql
-- 기본 INNER JOIN
SELECT
  users.name,
  posts.title
FROM users
INNER JOIN posts ON users.id = posts.user_id;

-- 결과:
--  name  |        title
-- -------+----------------------
--  Alice | Alice의 첫 글
--  Alice | Alice의 두 번째 글
--  Bob   | Bob의 글

-- Charlie (글 없음) 제외됨
-- "작성자 없는 글" (user_id NULL) 제외됨
```

### 테이블 별칭

```sql
-- 별칭 사용 (권장)
SELECT
  u.name,
  p.title,
  p.created_at
FROM users u
INNER JOIN posts p ON u.id = p.user_id;

-- AS 키워드 (선택)
SELECT
  u.name,
  p.title
FROM users AS u
INNER JOIN posts AS p ON u.id = p.user_id;
```

### 여러 컬럼 조건

```sql
-- 여러 조건
SELECT
  u.name,
  p.title
FROM users u
INNER JOIN posts p
  ON u.id = p.user_id
  AND p.created_at > '2024-01-01';

-- WHERE 추가
SELECT
  u.name,
  p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id
WHERE u.name LIKE 'A%';
-- Alice만
```

---

## 8.3 LEFT JOIN (LEFT OUTER JOIN)

### 개념

```
LEFT JOIN:
왼쪽 테이블의 모든 행 + 오른쪽 매칭되는 행
매칭 없으면 NULL
```

### 기본 예제

```sql
-- LEFT JOIN
SELECT
  u.name,
  p.title
FROM users u
LEFT JOIN posts p ON u.id = p.user_id;

-- 결과:
--   name   |        title
-- ---------+----------------------
--  Alice   | Alice의 첫 글
--  Alice   | Alice의 두 번째 글
--  Bob     | Bob의 글
--  Charlie | NULL                  ← 글 없어도 나옴

-- Charlie도 포함됨 (posts가 NULL)
```

### 글 없는 사용자 찾기

```sql
-- 글을 한 번도 안 쓴 사용자
SELECT
  u.name,
  COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name
HAVING COUNT(p.id) = 0;

-- 또는
SELECT u.name
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE p.id IS NULL;
```

---

## 8.4 RIGHT JOIN (RIGHT OUTER JOIN)

### 기본 예제

```sql
-- RIGHT JOIN (잘 안 씀, LEFT JOIN 선호)
SELECT
  u.name,
  p.title
FROM posts p
RIGHT JOIN users u ON p.user_id = u.id;

-- LEFT JOIN과 동일한 결과
SELECT
  u.name,
  p.title
FROM users u
LEFT JOIN posts p ON u.id = p.user_id;

-- 실무에서는 LEFT JOIN 사용 권장
```

---

## 8.5 FULL OUTER JOIN

### 개념

```
FULL OUTER JOIN:
양쪽 모두 포함
매칭 없으면 NULL
```

### 예제

```sql
-- FULL OUTER JOIN
SELECT
  u.name,
  p.title
FROM users u
FULL OUTER JOIN posts p ON u.id = p.user_id;

-- 결과:
--   name   |        title
-- ---------+----------------------
--  Alice   | Alice의 첫 글
--  Alice   | Alice의 두 번째 글
--  Bob     | Bob의 글
--  Charlie | NULL                  ← 글 없는 사용자
--  NULL    | 작성자 없는 글         ← user_id NULL인 글
```

### 매칭 안 된 데이터만

```sql
-- 한쪽만 있는 데이터
SELECT
  u.name,
  p.title
FROM users u
FULL OUTER JOIN posts p ON u.id = p.user_id
WHERE u.id IS NULL OR p.id IS NULL;

-- Charlie (글 없음)
-- "작성자 없는 글" (user_id NULL)
```

---

## 8.6 CROSS JOIN

### 개념

```
CROSS JOIN:
모든 조합 (카르테시안 곱)
users 3명 × posts 4개 = 12행
```

### 예제

```sql
-- CROSS JOIN
SELECT
  u.name,
  p.title
FROM users u
CROSS JOIN posts p;

-- 결과: 3명 × 4개 = 12행
-- Alice | Alice의 첫 글
-- Alice | Alice의 두 번째 글
-- Alice | Bob의 글
-- Alice | 작성자 없는 글
-- Bob   | Alice의 첫 글
-- Bob   | Alice의 두 번째 글
-- ...
```

### 사용 사례

```sql
-- 날짜 × 사용자 조합 (리포트용)
SELECT
  d.date,
  u.name
FROM (
  SELECT generate_series(
    '2024-01-01'::date,
    '2024-01-07'::date,
    '1 day'::interval
  )::date AS date
) d
CROSS JOIN users u
ORDER BY d.date, u.name;

-- 모든 날짜 × 모든 사용자 조합
```

---

## 8.7 다중 JOIN

### 3개 테이블

```sql
-- categories 테이블
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100)
);

INSERT INTO categories (name) VALUES
  ('기술'),
  ('일상');

-- posts에 category_id 추가
ALTER TABLE posts ADD COLUMN category_id INT REFERENCES categories(id);

UPDATE posts SET category_id = 1 WHERE id IN (1, 2);
UPDATE posts SET category_id = 2 WHERE id = 3;

-- 3개 테이블 JOIN
SELECT
  u.name AS author,
  c.name AS category,
  p.title
FROM posts p
INNER JOIN users u ON p.user_id = u.id
INNER JOIN categories c ON p.category_id = c.id;

-- 결과:
--  author |  category  |        title
-- --------+------------+----------------------
--  Alice  | 기술       | Alice의 첫 글
--  Alice  | 기술       | Alice의 두 번째 글
--  Bob    | 일상       | Bob의 글
```

### 4개 이상 테이블

```sql
-- comments 테이블
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  post_id INT REFERENCES posts(id),
  user_id INT REFERENCES users(id),
  content TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO comments (post_id, user_id, content) VALUES
  (1, 2, 'Bob의 댓글'),
  (1, 3, 'Charlie의 댓글');

-- 4개 테이블 JOIN
SELECT
  p.title AS post_title,
  post_author.name AS post_author,
  c.content AS comment,
  comment_author.name AS comment_author
FROM comments c
INNER JOIN posts p ON c.post_id = p.id
INNER JOIN users post_author ON p.user_id = post_author.id
INNER JOIN users comment_author ON c.user_id = comment_author.id;

-- 결과:
--    post_title     | post_author |    comment      | comment_author
-- ------------------+-------------+-----------------+----------------
--  Alice의 첫 글    | Alice       | Bob의 댓글      | Bob
--  Alice의 첫 글    | Alice       | Charlie의 댓글  | Charlie
```

---

## 8.8 SELF JOIN

### 개념

```
SELF JOIN:
같은 테이블을 자기 자신과 JOIN
```

### 예제: 조직도

```sql
-- employees 테이블
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  manager_id INT REFERENCES employees(id)
);

INSERT INTO employees (name, manager_id) VALUES
  ('CEO', NULL),
  ('CTO', 1),
  ('개발팀장', 2),
  ('개발자1', 3),
  ('개발자2', 3);

-- SELF JOIN: 직원과 매니저
SELECT
  e.name AS employee,
  m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- 결과:
--  employee  | manager
-- -----------+---------
--  CEO       | NULL
--  CTO       | CEO
--  개발팀장   | CTO
--  개발자1    | 개발팀장
--  개발자2    | 개발팀장
```

### 예제: 대댓글

```sql
-- 댓글에 parent_id 추가
ALTER TABLE comments ADD COLUMN parent_id INT REFERENCES comments(id);

UPDATE comments SET parent_id = NULL WHERE id = 1;
UPDATE comments SET parent_id = 1 WHERE id = 2;

-- 대댓글과 원댓글
SELECT
  c1.content AS comment,
  c2.content AS reply
FROM comments c1
LEFT JOIN comments c2 ON c1.id = c2.parent_id
WHERE c1.parent_id IS NULL;
```

---

## 8.9 실전 예제

### 블로그 통계

```sql
-- 사용자별 게시글 수
SELECT
  u.name,
  COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name
ORDER BY post_count DESC;

-- 결과:
--   name   | post_count
-- ---------+------------
--  Alice   | 2
--  Bob     | 1
--  Charlie | 0
```

### 카테고리별 게시글 수

```sql
SELECT
  c.name AS category,
  COUNT(p.id) AS post_count
FROM categories c
LEFT JOIN posts p ON c.id = p.category_id
GROUP BY c.id, c.name
ORDER BY post_count DESC;
```

### 최근 게시글과 작성자

```sql
SELECT
  p.title,
  u.name AS author,
  c.name AS category,
  p.created_at
FROM posts p
INNER JOIN users u ON p.user_id = u.id
LEFT JOIN categories c ON p.category_id = c.id
ORDER BY p.created_at DESC
LIMIT 10;
```

### 댓글 많은 게시글

```sql
SELECT
  p.title,
  u.name AS author,
  COUNT(cm.id) AS comment_count
FROM posts p
INNER JOIN users u ON p.user_id = u.id
LEFT JOIN comments cm ON p.id = cm.post_id
GROUP BY p.id, p.title, u.name
HAVING COUNT(cm.id) > 0
ORDER BY comment_count DESC;
```

---

## 8.10 서브쿼리 vs JOIN

### 서브쿼리

```sql
-- 서브쿼리 (느림)
SELECT
  name,
  (SELECT COUNT(*) FROM posts WHERE user_id = users.id) AS post_count
FROM users;
```

### JOIN (권장)

```sql
-- JOIN (빠름)
SELECT
  u.name,
  COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name;
```

### IN vs JOIN

```sql
-- IN + 서브쿼리
SELECT * FROM users
WHERE id IN (SELECT user_id FROM posts WHERE created_at > '2024-01-01');

-- JOIN
SELECT DISTINCT u.*
FROM users u
INNER JOIN posts p ON u.id = p.user_id
WHERE p.created_at > '2024-01-01';

-- 성능: JOIN이 일반적으로 더 빠름
```

---

## 8.11 JOIN 성능 최적화

### 인덱스

```sql
-- FK에 인덱스 (필수!)
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_category_id ON posts(category_id);
CREATE INDEX idx_comments_post_id ON comments(post_id);

-- JOIN 조건 컬럼에 인덱스
CREATE INDEX idx_posts_created_at ON posts(created_at);
```

### EXPLAIN 분석

```sql
-- 실행 계획 확인
EXPLAIN ANALYZE
SELECT
  u.name,
  p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id;

-- Seq Scan → Index Scan으로 변경되었는지 확인
```

### 불필요한 JOIN 제거

```sql
-- ❌ 나쁜 예: 불필요한 JOIN
SELECT u.name
FROM users u
INNER JOIN posts p ON u.id = p.user_id;
-- posts 데이터를 안 쓰는데 JOIN함

-- ✅ 좋은 예: WHERE + EXISTS
SELECT u.name
FROM users u
WHERE EXISTS (SELECT 1 FROM posts WHERE user_id = u.id);
```

---

## 8.12 JOIN 함정

### 중복 행

```sql
-- 문제: 댓글 많으면 행 중복
SELECT
  p.title,
  u.name
FROM posts p
INNER JOIN users u ON p.user_id = u.id
INNER JOIN comments c ON p.id = c.post_id;

-- 댓글 2개면 같은 게시글이 2번 나옴!

-- 해결: DISTINCT 또는 GROUP BY
SELECT DISTINCT
  p.title,
  u.name
FROM posts p
INNER JOIN users u ON p.user_id = u.id
INNER JOIN comments c ON p.id = c.post_id;
```

### NULL 처리

```sql
-- LEFT JOIN 후 COUNT
SELECT
  u.name,
  COUNT(p.id) AS post_count  -- COUNT(*)가 아닌 COUNT(p.id)
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name;

-- COUNT(*): NULL도 카운트 (잘못된 결과)
-- COUNT(p.id): NULL 제외 (올바른 결과)
```

---

## 8.13 복잡한 JOIN 예제

### 전자상거래

```sql
-- 주문 상세 정보
SELECT
  o.id AS order_id,
  u.name AS customer,
  p.name AS product,
  oi.quantity,
  oi.price,
  (oi.quantity * oi.price) AS subtotal,
  o.created_at AS order_date
FROM orders o
INNER JOIN users u ON o.user_id = u.id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
WHERE o.created_at >= '2024-01-01'
ORDER BY o.created_at DESC;
```

### SNS 팔로우

```sql
-- follows 테이블
CREATE TABLE follows (
  follower_id INT REFERENCES users(id),
  following_id INT REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (follower_id, following_id)
);

-- Alice가 팔로우하는 사람들
SELECT
  u.name AS following
FROM follows f
INNER JOIN users u ON f.following_id = u.id
WHERE f.follower_id = 1;  -- Alice

-- Alice를 팔로우하는 사람들
SELECT
  u.name AS follower
FROM follows f
INNER JOIN users u ON f.follower_id = u.id
WHERE f.following_id = 1;  -- Alice

-- 맞팔 (상호 팔로우)
SELECT
  u1.name AS user1,
  u2.name AS user2
FROM follows f1
INNER JOIN follows f2
  ON f1.follower_id = f2.following_id
  AND f1.following_id = f2.follower_id
INNER JOIN users u1 ON f1.follower_id = u1.id
INNER JOIN users u2 ON f1.following_id = u2.id
WHERE f1.follower_id < f1.following_id;  -- 중복 제거
```

---

## 핵심 요약

### JOIN 종류
```sql
INNER JOIN    -- 교집합 (양쪽 모두 있음)
LEFT JOIN     -- 왼쪽 기준 (오른쪽 NULL 가능)
RIGHT JOIN    -- 오른쪽 기준 (잘 안 씀)
FULL JOIN     -- 합집합 (양쪽 NULL 가능)
CROSS JOIN    -- 모든 조합
SELF JOIN     -- 같은 테이블
```

### 기본 문법
```sql
SELECT columns
FROM table1
INNER JOIN table2 ON table1.id = table2.fk
WHERE conditions;
```

### 성능
```sql
-- FK에 인덱스 필수
CREATE INDEX idx_fk ON table(fk_column);

-- EXPLAIN으로 확인
EXPLAIN ANALYZE SELECT ...;
```

### 주의사항
```sql
-- LEFT JOIN 후 COUNT
COUNT(right_table.id)  -- NULL 제외

-- 중복 행 방지
DISTINCT 또는 GROUP BY
```

---

## 다음 단계

JOIN을 익혔으니, 이제 **집계 함수와 GROUP BY**를 배워봅시다!

👉 [Chapter 9. 집계 함수와 GROUP BY](09-집계함수와-GROUPBY.md)

---

## 실습 과제

### 과제 1: 기본 JOIN

```sql
-- 1. INNER JOIN: 게시글과 작성자
-- 2. LEFT JOIN: 모든 사용자와 게시글 수
-- 3. 글 없는 사용자 찾기
```

### 과제 2: 다중 JOIN

```sql
-- 1. 게시글 + 작성자 + 카테고리
-- 2. 댓글 + 게시글 + 작성자
-- 3. 최신 댓글 10개 조회
```

### 과제 3: 통계

```sql
-- 1. 사용자별 게시글 수
-- 2. 카테고리별 게시글 수
-- 3. 댓글 많은 게시글 TOP 5
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 3시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!