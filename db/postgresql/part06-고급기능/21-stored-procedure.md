# Chapter 21. Stored Procedure

## 21.1 Stored Procedure란?

### 정의

```
Stored Procedure (저장 프로시저):
데이터베이스에 저장된 재사용 가능한 프로그램

비유: 레시피
┌─────────────────────────────────┐
│ 레시피 (Procedure):             │
│ 1. 재료 준비                    │
│ 2. 조리                         │
│ 3. 완성                         │
│                                 │
│ 장점:                           │
│ - 재사용 가능                   │
│ - 일관된 결과                   │
│ - 실수 방지                     │
└─────────────────────────────────┘

Stored Procedure = 복잡한 로직을 DB에 저장
```

### Function vs Procedure

```
Function:
- RETURN 값 필수
- SELECT에서 사용 가능
- 트랜잭션 제어 불가

Procedure (PostgreSQL 11+):
- RETURN 값 선택적
- CALL로 실행
- 트랜잭션 제어 가능 (COMMIT, ROLLBACK)
```

---

## 21.2 Function 기본

### 단순 Function

```sql
-- 함수 생성
CREATE OR REPLACE FUNCTION add_numbers(a INT, b INT)
RETURNS INT AS $$
BEGIN
  RETURN a + b;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT add_numbers(10, 20);
-- 30

-- 또는
SELECT add_numbers(a := 10, b := 20);
-- Named parameters
```

### 여러 값 반환 (OUT 파라미터)

```sql
-- OUT 파라미터
CREATE OR REPLACE FUNCTION get_user_stats(
  user_id INT,
  OUT post_count INT,
  OUT comment_count INT,
  OUT total_likes INT
) AS $$
BEGIN
  SELECT COUNT(*) INTO post_count
  FROM posts WHERE posts.user_id = get_user_stats.user_id;

  SELECT COUNT(*) INTO comment_count
  FROM comments WHERE comments.user_id = get_user_stats.user_id;

  SELECT COUNT(*) INTO total_likes
  FROM likes WHERE likes.user_id = get_user_stats.user_id;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM get_user_stats(1);
--  post_count | comment_count | total_likes
-- ------------+---------------+-------------
--     10      |      25       |     50
```

### 테이블 반환 (RETURNS TABLE)

```sql
-- 테이블 반환
CREATE OR REPLACE FUNCTION get_active_users()
RETURNS TABLE(id INT, name VARCHAR, email VARCHAR) AS $$
BEGIN
  RETURN QUERY
  SELECT u.id, u.name, u.email
  FROM users u
  WHERE u.is_active = true
  ORDER BY u.created_at DESC;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM get_active_users();
--  id | name  |      email
-- ----+-------+-----------------
--   1 | Alice | alice@ex.com
--   2 | Bob   | bob@ex.com

-- WHERE 조건 추가 가능
SELECT * FROM get_active_users() WHERE name LIKE 'A%';
```

---

## 21.3 Procedure 기본 (PostgreSQL 11+)

### 단순 Procedure

```sql
-- Procedure 생성
CREATE OR REPLACE PROCEDURE insert_user(
  p_name VARCHAR,
  p_email VARCHAR
) AS $$
BEGIN
  INSERT INTO users (name, email) VALUES (p_name, p_email);
  COMMIT;  -- Procedure에서만 가능!
END;
$$ LANGUAGE plpgsql;

-- 실행
CALL insert_user('Alice', 'alice@example.com');
```

### 트랜잭션 제어

```sql
-- Procedure: 트랜잭션 제어 가능
CREATE OR REPLACE PROCEDURE process_orders() AS $$
DECLARE
  order_record RECORD;
BEGIN
  FOR order_record IN
    SELECT * FROM orders WHERE status = 'pending'
  LOOP
    BEGIN
      -- 주문 처리
      UPDATE orders SET status = 'processing' WHERE id = order_record.id;

      -- 재고 차감
      UPDATE products SET stock = stock - 1 WHERE id = order_record.product_id;

      COMMIT;  -- 각 주문마다 커밋
    EXCEPTION
      WHEN OTHERS THEN
        ROLLBACK;  -- 에러 시 롤백
        RAISE NOTICE 'Order % failed: %', order_record.id, SQLERRM;
    END;
  END LOOP;
END;
$$ LANGUAGE plpgsql;

-- 실행
CALL process_orders();
```

---

## 21.4 PL/pgSQL 기본 문법

### 변수 선언

```sql
CREATE OR REPLACE FUNCTION variable_examples()
RETURNS VOID AS $$
DECLARE
  -- 기본 타입
  v_int INT := 10;
  v_text TEXT := 'Hello';
  v_bool BOOLEAN := true;
  v_date DATE := CURRENT_DATE;

  -- 테이블 컬럼 타입 사용
  v_name users.name%TYPE;

  -- 행 타입
  v_user users%ROWTYPE;

  -- 레코드
  v_record RECORD;
BEGIN
  -- 변수 사용
  v_name := 'Alice';

  SELECT * INTO v_user FROM users WHERE id = 1;
  RAISE NOTICE 'User: %', v_user.name;
END;
$$ LANGUAGE plpgsql;
```

### 조건문

```sql
CREATE OR REPLACE FUNCTION check_age(p_age INT)
RETURNS TEXT AS $$
BEGIN
  IF p_age < 18 THEN
    RETURN 'Minor';
  ELSIF p_age < 65 THEN
    RETURN 'Adult';
  ELSE
    RETURN 'Senior';
  END IF;
END;
$$ LANGUAGE plpgsql;

-- CASE 문
CREATE OR REPLACE FUNCTION get_grade(p_score INT)
RETURNS VARCHAR AS $$
BEGIN
  RETURN CASE
    WHEN p_score >= 90 THEN 'A'
    WHEN p_score >= 80 THEN 'B'
    WHEN p_score >= 70 THEN 'C'
    ELSE 'F'
  END;
END;
$$ LANGUAGE plpgsql;
```

### 반복문

```sql
-- LOOP
CREATE OR REPLACE FUNCTION loop_example()
RETURNS VOID AS $$
DECLARE
  i INT := 1;
BEGIN
  LOOP
    RAISE NOTICE 'i = %', i;
    i := i + 1;
    EXIT WHEN i > 10;
  END LOOP;
END;
$$ LANGUAGE plpgsql;


-- WHILE
CREATE OR REPLACE FUNCTION while_example()
RETURNS VOID AS $$
DECLARE
  i INT := 1;
BEGIN
  WHILE i <= 10 LOOP
    RAISE NOTICE 'i = %', i;
    i := i + 1;
  END LOOP;
END;
$$ LANGUAGE plpgsql;


-- FOR (정수 범위)
CREATE OR REPLACE FUNCTION for_example()
RETURNS VOID AS $$
DECLARE
  i INT;
BEGIN
  FOR i IN 1..10 LOOP
    RAISE NOTICE 'i = %', i;
  END LOOP;
END;
$$ LANGUAGE plpgsql;


-- FOR (쿼리 결과)
CREATE OR REPLACE FUNCTION for_query_example()
RETURNS VOID AS $$
DECLARE
  user_record RECORD;
BEGIN
  FOR user_record IN
    SELECT id, name FROM users WHERE is_active = true
  LOOP
    RAISE NOTICE 'User: % (%)', user_record.name, user_record.id;
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 예외 처리

```sql
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC AS $$
BEGIN
  RETURN a / b;
EXCEPTION
  WHEN division_by_zero THEN
    RAISE NOTICE 'Division by zero!';
    RETURN NULL;
  WHEN OTHERS THEN
    RAISE NOTICE 'Error: %', SQLERRM;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT safe_divide(10, 0);
-- NOTICE: Division by zero!
-- NULL
```

---

## 21.5 실전 Function 예제

### 예제 1: 계좌 이체

```sql
CREATE OR REPLACE FUNCTION transfer_money(
  from_account_id INT,
  to_account_id INT,
  amount NUMERIC
) RETURNS BOOLEAN AS $$
DECLARE
  from_balance NUMERIC;
BEGIN
  -- 잔액 확인
  SELECT balance INTO from_balance
  FROM accounts
  WHERE id = from_account_id
  FOR UPDATE;

  IF from_balance < amount THEN
    RAISE EXCEPTION 'Insufficient balance';
  END IF;

  -- 출금
  UPDATE accounts
  SET balance = balance - amount
  WHERE id = from_account_id;

  -- 입금
  UPDATE accounts
  SET balance = balance + amount
  WHERE id = to_account_id;

  -- 거래 내역
  INSERT INTO transactions (from_account_id, to_account_id, amount, created_at)
  VALUES (from_account_id, to_account_id, amount, NOW());

  RETURN true;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT transfer_money(1, 2, 10000);
```

### 예제 2: 페이지네이션

```sql
CREATE OR REPLACE FUNCTION get_posts_paginated(
  page_num INT DEFAULT 1,
  page_size INT DEFAULT 20
)
RETURNS TABLE(
  id INT,
  title VARCHAR,
  content TEXT,
  author_name VARCHAR,
  created_at TIMESTAMPTZ,
  total_count BIGINT
) AS $$
DECLARE
  offset_val INT;
BEGIN
  offset_val := (page_num - 1) * page_size;

  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.content,
    u.name AS author_name,
    p.created_at,
    COUNT(*) OVER() AS total_count
  FROM posts p
  JOIN users u ON p.user_id = u.id
  WHERE p.status = 'published'
  ORDER BY p.created_at DESC
  LIMIT page_size
  OFFSET offset_val;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM get_posts_paginated(1, 10);
--  id | title | content | author_name | created_at | total_count
-- ----+-------+---------+-------------+------------+-------------
--   1 | ...   | ...     | Alice       | ...        | 100
```

### 예제 3: 검색 (전문 검색)

```sql
CREATE OR REPLACE FUNCTION search_posts(
  search_query TEXT,
  limit_count INT DEFAULT 20
)
RETURNS TABLE(
  id INT,
  title VARCHAR,
  content TEXT,
  rank REAL,
  created_at TIMESTAMPTZ
) AS $$
BEGIN
  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.content,
    ts_rank(
      to_tsvector('english', p.title || ' ' || p.content),
      to_tsquery('english', search_query)
    ) AS rank,
    p.created_at
  FROM posts p
  WHERE to_tsvector('english', p.title || ' ' || p.content)
        @@ to_tsquery('english', search_query)
  ORDER BY rank DESC, p.created_at DESC
  LIMIT limit_count;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM search_posts('postgresql & performance');
```

### 예제 4: 통계 계산

```sql
CREATE OR REPLACE FUNCTION calculate_user_stats(p_user_id INT)
RETURNS TABLE(
  total_posts INT,
  total_comments INT,
  total_likes INT,
  avg_post_likes NUMERIC,
  first_post_date TIMESTAMPTZ,
  last_post_date TIMESTAMPTZ,
  engagement_score NUMERIC
) AS $$
BEGIN
  RETURN QUERY
  WITH user_posts AS (
    SELECT
      p.id,
      p.created_at,
      COUNT(DISTINCT l.id) AS like_count
    FROM posts p
    LEFT JOIN likes l ON p.id = l.post_id
    WHERE p.user_id = p_user_id
    GROUP BY p.id, p.created_at
  )
  SELECT
    COUNT(*)::INT AS total_posts,
    (SELECT COUNT(*)::INT FROM comments WHERE user_id = p_user_id) AS total_comments,
    (SELECT COUNT(*)::INT FROM likes WHERE user_id = p_user_id) AS total_likes,
    ROUND(AVG(up.like_count), 2) AS avg_post_likes,
    MIN(up.created_at) AS first_post_date,
    MAX(up.created_at) AS last_post_date,
    ROUND(
      (COUNT(*) * 10 +
       (SELECT COUNT(*) FROM comments WHERE user_id = p_user_id) * 5 +
       (SELECT COUNT(*) FROM likes WHERE user_id = p_user_id)) / 100.0,
      2
    ) AS engagement_score
  FROM user_posts up;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM calculate_user_stats(1);
```

---

## 21.6 실전 Procedure 예제

### 예제 1: 배치 데이터 처리

```sql
CREATE OR REPLACE PROCEDURE process_pending_orders()
AS $$
DECLARE
  order_rec RECORD;
  processed_count INT := 0;
  failed_count INT := 0;
BEGIN
  FOR order_rec IN
    SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at
  LOOP
    BEGIN
      -- 주문 처리
      UPDATE orders
      SET status = 'processing', processed_at = NOW()
      WHERE id = order_rec.id;

      -- 재고 차감
      UPDATE products
      SET stock = stock - 1
      WHERE id = order_rec.product_id;

      -- 알림 발송 (큐에 추가)
      INSERT INTO notification_queue (order_id, type)
      VALUES (order_rec.id, 'order_processing');

      processed_count := processed_count + 1;
      COMMIT;

    EXCEPTION
      WHEN OTHERS THEN
        ROLLBACK;
        UPDATE orders
        SET status = 'failed', error_message = SQLERRM
        WHERE id = order_rec.id;
        COMMIT;

        failed_count := failed_count + 1;
        RAISE NOTICE 'Order % failed: %', order_rec.id, SQLERRM;
    END;
  END LOOP;

  RAISE NOTICE 'Processed: %, Failed: %', processed_count, failed_count;
END;
$$ LANGUAGE plpgsql;

-- 실행
CALL process_pending_orders();
```

### 예제 2: 데이터 정리 (아카이빙)

```sql
CREATE OR REPLACE PROCEDURE archive_old_logs(days_to_keep INT DEFAULT 90)
AS $$
DECLARE
  archive_date DATE;
  rows_archived INT;
BEGIN
  archive_date := CURRENT_DATE - days_to_keep;

  -- 아카이브 테이블로 이동
  INSERT INTO logs_archive
  SELECT * FROM logs
  WHERE created_at < archive_date;

  GET DIAGNOSTICS rows_archived = ROW_COUNT;

  -- 원본에서 삭제
  DELETE FROM logs WHERE created_at < archive_date;

  COMMIT;

  RAISE NOTICE 'Archived % rows older than %', rows_archived, archive_date;
END;
$$ LANGUAGE plpgsql;

-- 실행
CALL archive_old_logs(90);
```

### 예제 3: 통계 테이블 갱신

```sql
CREATE OR REPLACE PROCEDURE refresh_daily_stats()
AS $$
DECLARE
  yesterday DATE := CURRENT_DATE - 1;
BEGIN
  -- 어제 통계 삭제 (재계산)
  DELETE FROM daily_stats WHERE date = yesterday;

  -- 새로 계산
  INSERT INTO daily_stats (date, new_users, new_orders, revenue)
  SELECT
    yesterday,
    (SELECT COUNT(*) FROM users WHERE DATE(created_at) = yesterday),
    (SELECT COUNT(*) FROM orders WHERE DATE(created_at) = yesterday),
    (SELECT COALESCE(SUM(total_amount), 0) FROM orders
     WHERE DATE(created_at) = yesterday AND status = 'completed');

  COMMIT;

  RAISE NOTICE 'Daily stats refreshed for %', yesterday;
END;
$$ LANGUAGE plpgsql;

-- 스케줄링 (pg_cron)
SELECT cron.schedule('refresh-daily-stats', '0 1 * * *', 'CALL refresh_daily_stats()');
```

---

## 21.7 동적 SQL

### EXECUTE 사용

```sql
CREATE OR REPLACE FUNCTION dynamic_query(table_name TEXT, column_name TEXT)
RETURNS TABLE(result TEXT) AS $$
BEGIN
  RETURN QUERY EXECUTE
    format('SELECT %I::TEXT FROM %I LIMIT 10', column_name, table_name);
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM dynamic_query('users', 'name');
```

### SQL Injection 방지

```sql
-- ❌ 위험: SQL Injection 가능
CREATE OR REPLACE FUNCTION unsafe_query(user_input TEXT)
RETURNS TABLE(id INT, name VARCHAR) AS $$
BEGIN
  RETURN QUERY EXECUTE
    'SELECT id, name FROM users WHERE name = ''' || user_input || '''';
  -- SQL Injection 위험!
END;
$$ LANGUAGE plpgsql;


-- ✅ 안전: 파라미터 바인딩
CREATE OR REPLACE FUNCTION safe_query(user_input TEXT)
RETURNS TABLE(id INT, name VARCHAR) AS $$
BEGIN
  RETURN QUERY EXECUTE
    'SELECT id, name FROM users WHERE name = $1'
  USING user_input;
END;
$$ LANGUAGE plpgsql;
```

---

## 21.8 Function 성능 최적화

### IMMUTABLE, STABLE, VOLATILE

```sql
-- IMMUTABLE: 같은 입력 → 같은 결과 (캐싱 가능)
CREATE OR REPLACE FUNCTION add(a INT, b INT)
RETURNS INT AS $$
BEGIN
  RETURN a + b;
END;
$$ LANGUAGE plpgsql IMMUTABLE;


-- STABLE: 같은 트랜잭션 내에서 같은 결과
CREATE OR REPLACE FUNCTION get_current_date()
RETURNS DATE AS $$
BEGIN
  RETURN CURRENT_DATE;
END;
$$ LANGUAGE plpgsql STABLE;


-- VOLATILE: 매번 다른 결과 가능 (기본값)
CREATE OR REPLACE FUNCTION generate_random()
RETURNS NUMERIC AS $$
BEGIN
  RETURN random();
END;
$$ LANGUAGE plpgsql VOLATILE;
```

### PARALLEL SAFE

```sql
-- 병렬 실행 가능
CREATE OR REPLACE FUNCTION expensive_calculation(n INT)
RETURNS INT AS $$
BEGIN
  RETURN n * n;
END;
$$ LANGUAGE plpgsql PARALLEL SAFE IMMUTABLE;

-- 대량 데이터 처리 시 병렬 실행
SELECT expensive_calculation(id) FROM generate_series(1, 1000000) AS id;
```

### RETURNS NULL ON NULL INPUT

```sql
-- NULL 입력 시 자동으로 NULL 반환 (함수 실행 안 함)
CREATE OR REPLACE FUNCTION multiply(a INT, b INT)
RETURNS INT AS $$
BEGIN
  RETURN a * b;
END;
$$ LANGUAGE plpgsql RETURNS NULL ON NULL INPUT;

SELECT multiply(NULL, 10);
-- NULL (함수 실행 안 됨)
```

---

## 21.9 Function/Procedure 관리

### 조회

```sql
-- psql에서
\df                    -- 모든 함수
\df+ function_name     -- 함수 상세

-- SQL로
SELECT
  proname AS function_name,
  pg_get_function_arguments(oid) AS arguments,
  pg_get_functiondef(oid) AS definition
FROM pg_proc
WHERE pronamespace = 'public'::regnamespace
  AND prokind = 'f'  -- 'f' = function, 'p' = procedure
ORDER BY proname;
```

### 수정

```sql
-- CREATE OR REPLACE 사용
CREATE OR REPLACE FUNCTION function_name(...)
RETURNS ... AS $$
...
$$ LANGUAGE plpgsql;
```

### 삭제

```sql
-- Function 삭제
DROP FUNCTION function_name(arg_types);

-- IF EXISTS
DROP FUNCTION IF EXISTS function_name(arg_types);

-- Procedure 삭제
DROP PROCEDURE procedure_name(arg_types);
```

---

## 21.10 디버깅

### RAISE 사용

```sql
CREATE OR REPLACE FUNCTION debug_example(p_id INT)
RETURNS VOID AS $$
DECLARE
  v_name VARCHAR;
BEGIN
  SELECT name INTO v_name FROM users WHERE id = p_id;

  RAISE DEBUG 'Debug: %', v_name;
  RAISE LOG 'Log: %', v_name;
  RAISE INFO 'Info: %', v_name;
  RAISE NOTICE 'Notice: %', v_name;
  RAISE WARNING 'Warning: %', v_name;
END;
$$ LANGUAGE plpgsql;

-- 로그 레벨 설정
SET client_min_messages = DEBUG;
```

### ASSERT 사용

```sql
CREATE OR REPLACE FUNCTION assert_example(p_value INT)
RETURNS VOID AS $$
BEGIN
  ASSERT p_value > 0, 'Value must be positive';
  ASSERT p_value < 100, 'Value must be less than 100';

  -- 로직
END;
$$ LANGUAGE plpgsql;

-- 에러 발생
SELECT assert_example(-1);
-- ERROR: Value must be positive
```

---

## 핵심 요약

### Function

```sql
-- 기본
CREATE OR REPLACE FUNCTION func_name(params)
RETURNS return_type AS $$
BEGIN
  RETURN value;
END;
$$ LANGUAGE plpgsql;

-- 테이블 반환
CREATE OR REPLACE FUNCTION func_name()
RETURNS TABLE(col1 type, col2 type) AS $$
BEGIN
  RETURN QUERY SELECT ...;
END;
$$ LANGUAGE plpgsql;
```

### Procedure (PostgreSQL 11+)

```sql
CREATE OR REPLACE PROCEDURE proc_name(params) AS $$
BEGIN
  -- 트랜잭션 제어 가능
  COMMIT;
  ROLLBACK;
END;
$$ LANGUAGE plpgsql;

-- 실행
CALL proc_name(args);
```

### 주요 문법

```sql
-- 변수
DECLARE
  v_name TYPE := value;

-- 조건문
IF condition THEN ... ELSIF ... ELSE ... END IF;

-- 반복문
FOR i IN 1..10 LOOP ... END LOOP;

-- 예외 처리
EXCEPTION
  WHEN error_type THEN ...
```

### 성능 옵션

```sql
IMMUTABLE       -- 캐싱 가능
STABLE          -- 트랜잭션 내 일관성
VOLATILE        -- 매번 실행 (기본)
PARALLEL SAFE   -- 병렬 실행 가능
```

---

## 다음 단계

Stored Procedure를 마스터했으니, 이제 **Full Text Search**를 배워봅시다!

👉 [Chapter 22. Full Text Search](22-full-text-search.md)

---

## 실습 과제

### 과제 1: 기본 Function

```sql
-- 1. 계산 함수 (IMMUTABLE)
-- 2. 통계 함수 (RETURNS TABLE)
-- 3. 검색 함수 (파라미터 바인딩)
```

### 과제 2: Procedure

```sql
-- 1. 배치 처리 Procedure
-- 2. 트랜잭션 제어 (COMMIT/ROLLBACK)
-- 3. 예외 처리
```

### 과제 3: 복잡한 로직

```sql
-- 1. 계좌 이체 Function
-- 2. 페이지네이션 Function
-- 3. 통계 계산 Function
```

### 과제 4: 성능 최적화

```sql
-- 1. IMMUTABLE vs STABLE 비교
-- 2. PARALLEL SAFE 테스트
-- 3. 대량 데이터 처리 성능 측정
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자, DBA
**중요도**: ⭐⭐⭐⭐ 매우 중요!
