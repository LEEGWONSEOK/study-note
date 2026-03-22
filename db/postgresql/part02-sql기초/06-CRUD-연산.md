# Chapter 6. CRUD 연산

## 6.1 CRUD란?

### 정의

```
CRUD:
Create  - INSERT (생성)
Read    - SELECT (조회)
Update  - UPDATE (수정)
Delete  - DELETE (삭제)

비유: 메모장
┌──────────────────────────────┐
│ Create  → 메모 작성          │
│ Read    → 메모 읽기          │
│ Update  → 메모 수정          │
│ Delete  → 메모 삭제          │
└──────────────────────────────┘
```

---

## 6.2 INSERT (생성)

### 기본 문법

```sql
INSERT INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...);
```

### 단일 행 삽입

```sql
-- 테이블 준비
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  age INT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 모든 컬럼 지정
INSERT INTO users (email, name, age)
VALUES ('alice@example.com', 'Alice', 25);

-- 순서대로 모든 컬럼 (권장하지 않음)
INSERT INTO users
VALUES (DEFAULT, 'bob@example.com', 'Bob', 30, DEFAULT);

-- 일부 컬럼만 (NULL 허용 컬럼)
INSERT INTO users (email, name)
VALUES ('charlie@example.com', 'Charlie');
-- age는 NULL, created_at은 DEFAULT (NOW())
```

### 다중 행 삽입

```sql
-- 여러 행 한 번에 (권장)
INSERT INTO users (email, name, age) VALUES
  ('david@example.com', 'David', 28),
  ('eve@example.com', 'Eve', 32),
  ('frank@example.com', 'Frank', 27);

-- vs 여러 번 INSERT (느림)
INSERT INTO users (email, name, age) VALUES ('david@example.com', 'David', 28);
INSERT INTO users (email, name, age) VALUES ('eve@example.com', 'Eve', 32);
INSERT INTO users (email, name, age) VALUES ('frank@example.com', 'Frank', 27);
```

### RETURNING (삽입된 데이터 반환)

```sql
-- 생성된 ID 반환
INSERT INTO users (email, name, age)
VALUES ('grace@example.com', 'Grace', 29)
RETURNING id;
--  id
-- ----
--   7

-- 모든 컬럼 반환
INSERT INTO users (email, name, age)
VALUES ('henry@example.com', 'Henry', 35)
RETURNING *;

-- 특정 컬럼만 반환
INSERT INTO users (email, name, age)
VALUES ('iris@example.com', 'Iris', 26)
RETURNING id, email, created_at;
```

### INSERT ... SELECT

```sql
-- 다른 테이블에서 복사
CREATE TABLE users_backup (
  LIKE users INCLUDING ALL
);

INSERT INTO users_backup
SELECT * FROM users WHERE age > 30;

-- 조건부 복사
INSERT INTO users_backup (email, name, age)
SELECT email, name, age
FROM users
WHERE created_at > '2024-01-01';
```

### ON CONFLICT (UPSERT)

```sql
-- 중복 시 무시
INSERT INTO users (email, name, age)
VALUES ('alice@example.com', 'Alice Updated', 26)
ON CONFLICT (email) DO NOTHING;
-- email이 이미 있으면 무시

-- 중복 시 업데이트
INSERT INTO users (email, name, age)
VALUES ('alice@example.com', 'Alice Updated', 26)
ON CONFLICT (email) DO UPDATE
SET
  name = EXCLUDED.name,
  age = EXCLUDED.age;
-- EXCLUDED = 삽입하려던 새 값

-- 조건부 업데이트
INSERT INTO users (email, name, age)
VALUES ('alice@example.com', 'Alice Updated', 26)
ON CONFLICT (email) DO UPDATE
SET
  name = EXCLUDED.name,
  age = EXCLUDED.age
WHERE users.age < EXCLUDED.age;  -- 새 나이가 더 클 때만
```

---

## 6.3 SELECT (조회)

### 기본 조회

```sql
-- 모든 컬럼, 모든 행
SELECT * FROM users;

-- 특정 컬럼
SELECT email, name FROM users;

-- 컬럼 별칭
SELECT
  email AS user_email,
  name AS full_name,
  age AS user_age
FROM users;

-- 계산 컬럼
SELECT
  name,
  age,
  age + 10 AS age_in_10_years
FROM users;
```

### DISTINCT (중복 제거)

```sql
-- 중복 제거
SELECT DISTINCT age FROM users;
-- 25, 28, 30, 32 (중복된 나이는 한 번만)

-- 여러 컬럼 조합 중복 제거
SELECT DISTINCT age, name FROM users;
```

### LIMIT / OFFSET (페이징)

```sql
-- 처음 10개
SELECT * FROM users LIMIT 10;

-- 11~20번째
SELECT * FROM users LIMIT 10 OFFSET 10;

-- 페이지 계산
-- 1페이지 (1~10)
SELECT * FROM users LIMIT 10 OFFSET 0;

-- 2페이지 (11~20)
SELECT * FROM users LIMIT 10 OFFSET 10;

-- 3페이지 (21~30)
SELECT * FROM users LIMIT 10 OFFSET 20;

-- 공식: OFFSET = (page - 1) * page_size
```

### ORDER BY (정렬)

```sql
-- 오름차순 (기본)
SELECT * FROM users ORDER BY age;
SELECT * FROM users ORDER BY age ASC;

-- 내림차순
SELECT * FROM users ORDER BY age DESC;

-- 여러 컬럼
SELECT * FROM users
ORDER BY age DESC, name ASC;
-- age 내림차순 → 같으면 name 오름차순

-- NULL 위치
SELECT * FROM users ORDER BY age NULLS FIRST;
SELECT * FROM users ORDER BY age NULLS LAST;
```

### 집계 함수

```sql
-- COUNT: 개수
SELECT COUNT(*) FROM users;  -- 모든 행
SELECT COUNT(age) FROM users;  -- NULL 제외

-- SUM: 합계
SELECT SUM(age) FROM users;

-- AVG: 평균
SELECT AVG(age) FROM users;
SELECT ROUND(AVG(age), 2) AS avg_age FROM users;

-- MIN/MAX: 최소/최대
SELECT MIN(age) FROM users;
SELECT MAX(age) FROM users;

-- 여러 함수
SELECT
  COUNT(*) AS total_users,
  AVG(age) AS avg_age,
  MIN(age) AS youngest,
  MAX(age) AS oldest
FROM users;
```

---

## 6.4 UPDATE (수정)

### 기본 UPDATE

```sql
-- 단일 컬럼 수정
UPDATE users
SET age = 26
WHERE email = 'alice@example.com';

-- 여러 컬럼 수정
UPDATE users
SET
  name = 'Alice Kim',
  age = 26
WHERE email = 'alice@example.com';

-- 계산 값
UPDATE users
SET age = age + 1
WHERE id = 1;
```

### WHERE 조건

```sql
-- 특정 행만
UPDATE users
SET age = 30
WHERE id = 5;

-- 여러 행
UPDATE users
SET age = age + 1
WHERE age < 30;

-- ⚠️ WHERE 없으면 모든 행 수정!
UPDATE users SET age = 0;  -- 위험! 모든 나이가 0
```

### 서브쿼리 UPDATE

```sql
-- 다른 테이블 값으로 수정
UPDATE users
SET age = (
  SELECT AVG(age)::INT
  FROM users
)
WHERE age IS NULL;
```

### RETURNING

```sql
-- 수정된 행 반환
UPDATE users
SET age = age + 1
WHERE id = 1
RETURNING *;

-- 수정 전/후 비교
UPDATE users
SET age = age + 1
WHERE id = 1
RETURNING id, name, age AS new_age;
```

### UPDATE ... FROM

```sql
-- 다른 테이블과 JOIN하여 UPDATE
CREATE TABLE user_stats (
  user_id INT PRIMARY KEY,
  login_count INT DEFAULT 0
);

UPDATE users
SET age = age + 1
FROM user_stats
WHERE users.id = user_stats.user_id
  AND user_stats.login_count > 100;
```

---

## 6.5 DELETE (삭제)

### 기본 DELETE

```sql
-- 특정 행 삭제
DELETE FROM users
WHERE id = 10;

-- 조건부 삭제
DELETE FROM users
WHERE age < 18;

-- ⚠️ WHERE 없으면 모든 행 삭제!
DELETE FROM users;  -- 위험! 모든 데이터 삭제
```

### RETURNING

```sql
-- 삭제된 행 반환
DELETE FROM users
WHERE id = 10
RETURNING *;

-- 삭제된 행 개수
DELETE FROM users
WHERE age < 18
RETURNING id;
```

### DELETE ... USING

```sql
-- 다른 테이블과 JOIN하여 DELETE
DELETE FROM users
USING user_stats
WHERE users.id = user_stats.user_id
  AND user_stats.login_count = 0;
```

### TRUNCATE vs DELETE

```sql
-- DELETE: 느림, 롤백 가능, WHERE 가능
DELETE FROM users WHERE age < 18;

-- TRUNCATE: 빠름, 롤백 불가, 전체 삭제만
TRUNCATE TABLE users;

-- TRUNCATE with RESTART IDENTITY
TRUNCATE TABLE users RESTART IDENTITY;
-- SERIAL 값 1부터 다시 시작

-- CASCADE
TRUNCATE TABLE users CASCADE;
-- 참조 테이블도 함께 TRUNCATE
```

---

## 6.6 트랜잭션

### BEGIN / COMMIT / ROLLBACK

```sql
-- 트랜잭션 시작
BEGIN;

INSERT INTO users (email, name, age)
VALUES ('test@example.com', 'Test', 25);

UPDATE users SET age = 26 WHERE email = 'test@example.com';

-- 커밋 (저장)
COMMIT;

-- 또는 롤백 (취소)
ROLLBACK;
```

### 실전 예제

```sql
-- 계좌 이체
BEGIN;

-- A 계좌에서 출금
UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

-- B 계좌에 입금
UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

-- 확인
SELECT * FROM accounts WHERE id IN (1, 2);

-- 문제 없으면 COMMIT
COMMIT;

-- 문제 있으면 ROLLBACK
ROLLBACK;
```

### SAVEPOINT

```sql
BEGIN;

INSERT INTO users (email, name) VALUES ('user1@example.com', 'User1');

SAVEPOINT sp1;

INSERT INTO users (email, name) VALUES ('user2@example.com', 'User2');

-- user2 삽입 취소
ROLLBACK TO SAVEPOINT sp1;

-- user1만 삽입됨
COMMIT;
```

---

## 6.7 실전 예제

### 블로그 CRUD

```sql
-- 1. 게시글 작성
INSERT INTO posts (user_id, title, content, is_published)
VALUES (1, 'PostgreSQL 시작하기', '내용...', true)
RETURNING id, title, created_at;

-- 2. 게시글 목록 (최신순, 페이징)
SELECT
  id,
  title,
  created_at,
  view_count
FROM posts
WHERE is_published = true
ORDER BY created_at DESC
LIMIT 10 OFFSET 0;

-- 3. 게시글 상세 (조회수 증가)
BEGIN;

UPDATE posts
SET view_count = view_count + 1
WHERE id = 1;

SELECT
  id,
  title,
  content,
  view_count,
  created_at
FROM posts
WHERE id = 1;

COMMIT;

-- 4. 게시글 수정
UPDATE posts
SET
  title = 'PostgreSQL 완벽 가이드',
  content = '수정된 내용...',
  updated_at = NOW()
WHERE id = 1
RETURNING *;

-- 5. 게시글 삭제 (소프트 삭제)
UPDATE posts
SET
  is_published = false,
  deleted_at = NOW()
WHERE id = 1;

-- 또는 하드 삭제
DELETE FROM posts WHERE id = 1;
```

### 사용자 관리

```sql
-- 1. 회원가입
INSERT INTO users (email, username, password_hash)
VALUES ('new@example.com', 'newuser', '$2a$10$...')
RETURNING id, email, username, created_at;

-- 2. 로그인 (이메일로 조회)
SELECT
  id,
  email,
  username,
  password_hash
FROM users
WHERE email = 'new@example.com'
  AND is_active = true;

-- 3. 프로필 수정
UPDATE users
SET
  full_name = 'New User',
  bio = '안녕하세요',
  updated_at = NOW()
WHERE id = 10
RETURNING id, full_name, bio;

-- 4. 비밀번호 변경
UPDATE users
SET
  password_hash = '$2a$10$...new...',
  updated_at = NOW()
WHERE id = 10;

-- 5. 회원 탈퇴 (소프트 삭제)
UPDATE users
SET
  is_active = false,
  deleted_at = NOW()
WHERE id = 10;
```

### 상품 재고 관리

```sql
-- 1. 주문 생성 (재고 감소)
BEGIN;

-- 재고 확인
SELECT stock FROM products WHERE id = 100 FOR UPDATE;
-- FOR UPDATE: 다른 트랜잭션이 수정 못하게 락

-- 재고 부족 체크
DO $$
BEGIN
  IF (SELECT stock FROM products WHERE id = 100) < 5 THEN
    RAISE EXCEPTION '재고 부족';
  END IF;
END $$;

-- 재고 감소
UPDATE products
SET stock = stock - 5
WHERE id = 100;

-- 주문 생성
INSERT INTO orders (user_id, product_id, quantity)
VALUES (1, 100, 5)
RETURNING *;

COMMIT;

-- 2. 주문 취소 (재고 복구)
BEGIN;

-- 재고 복구
UPDATE products
SET stock = stock + 5
WHERE id = 100;

-- 주문 삭제
DELETE FROM orders WHERE id = 50;

COMMIT;
```

---

## 6.8 성능 팁

### INSERT 최적화

```sql
-- ❌ 느림: 여러 번 INSERT
INSERT INTO users (email, name) VALUES ('user1@example.com', 'User1');
INSERT INTO users (email, name) VALUES ('user2@example.com', 'User2');
-- ... 1000번

-- ✅ 빠름: 한 번에 INSERT
INSERT INTO users (email, name) VALUES
  ('user1@example.com', 'User1'),
  ('user2@example.com', 'User2'),
  -- ... 1000개
  ('user1000@example.com', 'User1000');

-- ✅ 더 빠름: COPY (대량 데이터)
COPY users (email, name)
FROM '/path/to/users.csv'
CSV HEADER;
```

### UPDATE 최적화

```sql
-- ❌ 느림: 불필요한 UPDATE
UPDATE users SET updated_at = NOW();
-- 모든 행 수정

-- ✅ 빠름: 필요한 행만
UPDATE users
SET age = age + 1
WHERE age < 30;

-- ❌ 느림: 반복 UPDATE
UPDATE users SET age = 25 WHERE id = 1;
UPDATE users SET age = 26 WHERE id = 2;
-- ...

-- ✅ 빠름: CASE 문
UPDATE users
SET age = CASE
  WHEN id = 1 THEN 25
  WHEN id = 2 THEN 26
  WHEN id = 3 THEN 27
END
WHERE id IN (1, 2, 3);
```

### DELETE 최적화

```sql
-- ❌ 느림: DELETE
DELETE FROM logs WHERE created_at < '2023-01-01';
-- 수백만 행 삭제

-- ✅ 빠름: TRUNCATE (전체 삭제 시)
TRUNCATE TABLE logs;

-- ✅ 빠름: 파티셔닝 + DROP
-- 오래된 파티션 삭제
DROP TABLE logs_2023;
```

---

## 핵심 요약

### INSERT
```sql
INSERT INTO table (col1, col2)
VALUES (val1, val2)
RETURNING *;

-- ON CONFLICT (UPSERT)
ON CONFLICT (col) DO UPDATE SET ...;
```

### SELECT
```sql
SELECT col1, col2
FROM table
WHERE condition
ORDER BY col
LIMIT 10 OFFSET 0;
```

### UPDATE
```sql
UPDATE table
SET col1 = val1, col2 = val2
WHERE condition
RETURNING *;
```

### DELETE
```sql
DELETE FROM table
WHERE condition
RETURNING *;

-- 전체 삭제
TRUNCATE TABLE table;
```

### 트랜잭션
```sql
BEGIN;
-- SQL 문들
COMMIT;  -- 또는 ROLLBACK;
```

---

## 다음 단계

CRUD 연산을 익혔으니, 이제 **조건과 필터링**을 배워봅시다!

👉 [Chapter 7. 조건과 필터링](07-조건과-필터링.md)

---

## 실습 과제

### 과제 1: 기본 CRUD

```sql
-- 1. users 테이블 생성
-- 2. INSERT 5명
-- 3. SELECT 전체 조회
-- 4. UPDATE 특정 사용자
-- 5. DELETE 특정 사용자
```

### 과제 2: UPSERT

```sql
-- 1. email UNIQUE 제약조건
-- 2. INSERT ... ON CONFLICT
-- 3. 중복 시 name 업데이트
-- 4. 결과 확인
```

### 과제 3: 트랜잭션

```sql
-- 1. BEGIN
-- 2. INSERT 2개
-- 3. UPDATE 1개
-- 4. SELECT 확인
-- 5. COMMIT 또는 ROLLBACK
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐ 초급
**예상 학습 시간**: 2시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!