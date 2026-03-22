# Chapter 20. Trigger

## 20.1 Trigger란?

### 정의

```
Trigger (트리거):
특정 이벤트(INSERT, UPDATE, DELETE) 발생 시 자동으로 실행되는 함수

비유: 자동 알람
┌─────────────────────────────────┐
│ 이벤트:                         │
│ "문이 열림"                     │
│                                 │
│ 자동 동작 (Trigger):            │
│ 1. 불 켜기                      │
│ 2. 알림 보내기                  │
│ 3. 로그 기록                    │
└─────────────────────────────────┘

Trigger = 데이터 변경 시 자동 실행
```

### Trigger 구성 요소

```
1. 이벤트:
   - INSERT, UPDATE, DELETE

2. 타이밍:
   - BEFORE (변경 전)
   - AFTER (변경 후)
   - INSTEAD OF (View에서)

3. 단위:
   - FOR EACH ROW (행마다)
   - FOR EACH STATEMENT (문장마다)

4. 함수:
   - PL/pgSQL 함수
```

---

## 20.2 Trigger 기본 구조

### BEFORE Trigger

```sql
-- 함수 생성
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger 생성
CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- 테스트
UPDATE users SET name = 'Alice' WHERE id = 1;

SELECT id, name, updated_at FROM users WHERE id = 1;
--  id | name  |       updated_at
-- ----+-------+---------------------
--   1 | Alice | 2024-01-15 10:30:00
-- updated_at이 자동으로 갱신됨!
```

### AFTER Trigger

```sql
-- 로그 테이블
CREATE TABLE user_audit_log (
  id SERIAL PRIMARY KEY,
  user_id INT,
  action VARCHAR(10),
  old_data JSONB,
  new_data JSONB,
  changed_at TIMESTAMPTZ DEFAULT NOW()
);

-- 함수
CREATE OR REPLACE FUNCTION log_user_changes()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO user_audit_log (user_id, action, old_data, new_data)
  VALUES (
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    CASE WHEN TG_OP = 'DELETE' THEN row_to_json(OLD) ELSE NULL END,
    CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW) ELSE NULL END
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger
CREATE TRIGGER trg_users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION log_user_changes();

-- 테스트
UPDATE users SET email = 'newemail@example.com' WHERE id = 1;

SELECT * FROM user_audit_log;
--  id | user_id | action | old_data           | new_data           | changed_at
-- ----+---------+--------+--------------------+--------------------+-------------
--   1 |    1    | UPDATE | {"id":1,"email":...}| {"id":1,"email":...}| 2024-01-15...
```

---

## 20.3 Trigger 특수 변수

### OLD와 NEW

```sql
-- OLD: 변경 전 데이터
-- NEW: 변경 후 데이터

CREATE OR REPLACE FUNCTION example_trigger()
RETURNS TRIGGER AS $$
BEGIN
  -- INSERT: OLD는 NULL, NEW는 새 데이터
  -- UPDATE: OLD는 이전 데이터, NEW는 새 데이터
  -- DELETE: OLD는 이전 데이터, NEW는 NULL

  RAISE NOTICE 'Operation: %', TG_OP;
  RAISE NOTICE 'OLD: %', OLD;
  RAISE NOTICE 'NEW: %', NEW;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### TG_OP (Operation)

```sql
CREATE OR REPLACE FUNCTION multi_operation_trigger()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    RAISE NOTICE 'New row inserted: %', NEW.id;
  ELSIF TG_OP = 'UPDATE' THEN
    RAISE NOTICE 'Row updated: % -> %', OLD.name, NEW.name;
  ELSIF TG_OP = 'DELETE' THEN
    RAISE NOTICE 'Row deleted: %', OLD.id;
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

### 기타 특수 변수

```sql
CREATE OR REPLACE FUNCTION show_trigger_vars()
RETURNS TRIGGER AS $$
BEGIN
  RAISE NOTICE 'TG_NAME: %', TG_NAME;              -- 트리거 이름
  RAISE NOTICE 'TG_WHEN: %', TG_WHEN;              -- BEFORE/AFTER
  RAISE NOTICE 'TG_LEVEL: %', TG_LEVEL;            -- ROW/STATEMENT
  RAISE NOTICE 'TG_OP: %', TG_OP;                  -- INSERT/UPDATE/DELETE
  RAISE NOTICE 'TG_TABLE_NAME: %', TG_TABLE_NAME;  -- 테이블 이름
  RAISE NOTICE 'TG_TABLE_SCHEMA: %', TG_TABLE_SCHEMA;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## 20.4 실전 Trigger 예제

### 예제 1: 자동 타임스탬프

```sql
-- 함수
CREATE OR REPLACE FUNCTION set_timestamps()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    NEW.created_at = NOW();
    NEW.updated_at = NOW();
  ELSIF TG_OP = 'UPDATE' THEN
    NEW.updated_at = NOW();
    NEW.created_at = OLD.created_at;  -- 생성일은 변경 안 됨
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 여러 테이블에 적용
CREATE TRIGGER trg_users_timestamps
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_timestamps();

CREATE TRIGGER trg_posts_timestamps
BEFORE INSERT OR UPDATE ON posts
FOR EACH ROW
EXECUTE FUNCTION set_timestamps();
```

### 예제 2: 소프트 삭제

```sql
-- 실제 DELETE 대신 deleted_at 설정
CREATE OR REPLACE FUNCTION soft_delete()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    UPDATE users SET deleted_at = NOW() WHERE id = OLD.id;
    RETURN NULL;  -- 실제 삭제 취소
  END IF;
  RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_soft_delete
BEFORE DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION soft_delete();

-- 테스트
DELETE FROM users WHERE id = 1;
-- 실제로는 deleted_at만 설정됨

SELECT id, name, deleted_at FROM users WHERE id = 1;
--  id | name  |       deleted_at
-- ----+-------+---------------------
--   1 | Alice | 2024-01-15 10:30:00
```

### 예제 3: 재고 관리

```sql
-- 주문 생성 시 재고 자동 차감
CREATE OR REPLACE FUNCTION update_stock_on_order()
RETURNS TRIGGER AS $$
BEGIN
  -- 재고 차감
  UPDATE products
  SET stock = stock - NEW.quantity
  WHERE id = NEW.product_id;

  -- 재고 부족 체크
  IF (SELECT stock FROM products WHERE id = NEW.product_id) < 0 THEN
    RAISE EXCEPTION 'Insufficient stock for product %', NEW.product_id;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_items_stock
AFTER INSERT ON order_items
FOR EACH ROW
EXECUTE FUNCTION update_stock_on_order();

-- 테스트
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES (1, 10, 5, 50000);
-- products.stock이 5 감소

-- 재고 부족 시
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES (1, 10, 1000, 50000);
-- ERROR: Insufficient stock for product 10
```

### 예제 4: 자동 슬러그 생성

```sql
-- URL용 슬러그 자동 생성
CREATE OR REPLACE FUNCTION generate_slug()
RETURNS TRIGGER AS $$
BEGIN
  -- 제목에서 슬러그 생성
  NEW.slug = LOWER(
    REGEXP_REPLACE(
      REGEXP_REPLACE(NEW.title, '[^a-zA-Z0-9가-힣\s-]', '', 'g'),
      '\s+', '-', 'g'
    )
  );

  -- 중복 체크
  IF EXISTS (SELECT 1 FROM posts WHERE slug = NEW.slug AND id != COALESCE(NEW.id, 0)) THEN
    NEW.slug = NEW.slug || '-' || EXTRACT(EPOCH FROM NOW())::INT;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posts_slug
BEFORE INSERT OR UPDATE OF title ON posts
FOR EACH ROW
EXECUTE FUNCTION generate_slug();

-- 테스트
INSERT INTO posts (title, content) VALUES ('Hello World!', 'Content...');

SELECT id, title, slug FROM posts;
--  id | title        | slug
-- ----+--------------+-------------
--   1 | Hello World! | hello-world
```

### 예제 5: 캐시 무효화

```sql
-- 게시글 변경 시 캐시 무효화 플래그 설정
CREATE TABLE cache_invalidation (
  key VARCHAR(255) PRIMARY KEY,
  invalidated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION invalidate_post_cache()
RETURNS TRIGGER AS $$
BEGIN
  -- 전체 게시글 목록 캐시 무효화
  INSERT INTO cache_invalidation (key)
  VALUES ('posts:list')
  ON CONFLICT (key) DO UPDATE
  SET invalidated_at = NOW();

  -- 특정 게시글 캐시 무효화
  INSERT INTO cache_invalidation (key)
  VALUES ('posts:' || COALESCE(NEW.id, OLD.id))
  ON CONFLICT (key) DO UPDATE
  SET invalidated_at = NOW();

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posts_cache_invalidation
AFTER INSERT OR UPDATE OR DELETE ON posts
FOR EACH ROW
EXECUTE FUNCTION invalidate_post_cache();
```

---

## 20.5 INSTEAD OF Trigger (View용)

### View 업데이트

```sql
-- View 생성
CREATE VIEW user_profile AS
SELECT
  u.id,
  u.name,
  u.email,
  p.bio,
  p.avatar_url
FROM users u
LEFT JOIN profiles p ON u.id = p.user_id;

-- INSTEAD OF 트리거
CREATE OR REPLACE FUNCTION update_user_profile()
RETURNS TRIGGER AS $$
BEGIN
  -- users 테이블 업데이트
  UPDATE users
  SET name = NEW.name, email = NEW.email
  WHERE id = NEW.id;

  -- profiles 테이블 업데이트 (또는 삽입)
  INSERT INTO profiles (user_id, bio, avatar_url)
  VALUES (NEW.id, NEW.bio, NEW.avatar_url)
  ON CONFLICT (user_id) DO UPDATE
  SET bio = EXCLUDED.bio, avatar_url = EXCLUDED.avatar_url;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_user_profile_update
INSTEAD OF UPDATE ON user_profile
FOR EACH ROW
EXECUTE FUNCTION update_user_profile();

-- 테스트
UPDATE user_profile SET bio = 'New bio' WHERE id = 1;
-- users와 profiles 모두 업데이트됨!
```

---

## 20.6 조건부 Trigger (WHEN)

### 특정 조건에서만 실행

```sql
-- 가격 변경 시에만 로그
CREATE OR REPLACE FUNCTION log_price_change()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO price_history (product_id, old_price, new_price, changed_at)
  VALUES (NEW.id, OLD.price, NEW.price, NOW());
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_price_change
AFTER UPDATE OF price ON products
FOR EACH ROW
WHEN (OLD.price IS DISTINCT FROM NEW.price)
EXECUTE FUNCTION log_price_change();

-- price가 변경될 때만 실행!
```

### 특정 값 범위 체크

```sql
-- 나이가 음수면 에러
CREATE OR REPLACE FUNCTION check_valid_age()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.age < 0 THEN
    RAISE EXCEPTION 'Age cannot be negative: %', NEW.age;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_check_age
BEFORE INSERT OR UPDATE OF age ON users
FOR EACH ROW
WHEN (NEW.age IS NOT NULL)
EXECUTE FUNCTION check_valid_age();
```

---

## 20.7 Trigger 관리

### Trigger 조회

```sql
-- psql에서
\df  -- 함수 목록
\dy  -- 트리거 목록

-- SQL로
SELECT
  trigger_name,
  event_manipulation,
  event_object_table,
  action_statement,
  action_timing
FROM information_schema.triggers
WHERE trigger_schema = 'public'
ORDER BY event_object_table, trigger_name;

--  trigger_name           | event_manipulation | event_object_table | action_timing
-- ------------------------+--------------------+--------------------+---------------
--  trg_users_timestamps   | INSERT             | users              | BEFORE
--  trg_users_timestamps   | UPDATE             | users              | BEFORE
--  trg_users_audit        | INSERT             | users              | AFTER
```

### Trigger 비활성화/활성화

```sql
-- 비활성화
ALTER TABLE users DISABLE TRIGGER trg_users_timestamps;

-- 활성화
ALTER TABLE users ENABLE TRIGGER trg_users_timestamps;

-- 모든 트리거 비활성화
ALTER TABLE users DISABLE TRIGGER ALL;

-- 모든 트리거 활성화
ALTER TABLE users ENABLE TRIGGER ALL;
```

### Trigger 삭제

```sql
-- Trigger 삭제
DROP TRIGGER trg_users_timestamps ON users;

-- IF EXISTS 사용
DROP TRIGGER IF EXISTS trg_users_timestamps ON users;

-- 함수도 삭제
DROP FUNCTION IF EXISTS set_timestamps();
```

---

## 20.8 Trigger 실행 순서

### 여러 Trigger 순서

```sql
-- 알파벳 순서로 실행됨
CREATE TRIGGER trg_a_first ...
CREATE TRIGGER trg_b_second ...
CREATE TRIGGER trg_c_third ...

-- 순서 명시적 지정 (PostgreSQL 11+)
-- 함수 이름을 순서대로
CREATE TRIGGER trg_01_first ...
CREATE TRIGGER trg_02_second ...
CREATE TRIGGER trg_03_third ...
```

### BEFORE vs AFTER 순서

```
1. BEFORE STATEMENT 트리거
2. BEFORE ROW 트리거 (각 행마다)
3. 실제 작업 (INSERT/UPDATE/DELETE)
4. AFTER ROW 트리거 (각 행마다)
5. AFTER STATEMENT 트리거
```

---

## 20.9 Trigger 성능 고려사항

### FOR EACH ROW vs FOR EACH STATEMENT

```sql
-- ❌ 느림: 100만 행마다 실행
CREATE TRIGGER trg_slow
AFTER INSERT ON logs
FOR EACH ROW
EXECUTE FUNCTION expensive_function();

INSERT INTO logs SELECT ... FROM generate_series(1, 1000000);
-- expensive_function()이 100만 번 실행!


-- ✅ 빠름: 한 번만 실행
CREATE TRIGGER trg_fast
AFTER INSERT ON logs
FOR EACH STATEMENT
EXECUTE FUNCTION cheap_function();

INSERT INTO logs SELECT ... FROM generate_series(1, 1000000);
-- cheap_function()이 1번만 실행!
```

### 조건 최적화

```sql
-- ❌ 나쁜 예: 항상 실행
CREATE TRIGGER trg_bad
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION check_price_change();

CREATE OR REPLACE FUNCTION check_price_change()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.price != NEW.price THEN
    -- 가격 변경 처리
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;


-- ✅ 좋은 예: 가격 변경 시에만 실행
CREATE TRIGGER trg_good
AFTER UPDATE OF price ON products
FOR EACH ROW
WHEN (OLD.price IS DISTINCT FROM NEW.price)
EXECUTE FUNCTION log_price_change();
```

### Trigger 최소화

```sql
-- ❌ 너무 많은 Trigger
CREATE TRIGGER trg_1 AFTER INSERT ON users ...
CREATE TRIGGER trg_2 AFTER UPDATE ON users ...
CREATE TRIGGER trg_3 AFTER DELETE ON users ...

-- ✅ 하나로 통합
CREATE TRIGGER trg_users_all
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION handle_user_changes();
```

---

## 20.10 Trigger 디버깅

### RAISE 사용

```sql
CREATE OR REPLACE FUNCTION debug_trigger()
RETURNS TRIGGER AS $$
BEGIN
  RAISE NOTICE 'Trigger fired: %', TG_OP;
  RAISE NOTICE 'OLD: %', OLD;
  RAISE NOTICE 'NEW: %', NEW;

  -- 조건부 에러
  IF NEW.price < 0 THEN
    RAISE EXCEPTION 'Price cannot be negative';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 로그 레벨

```sql
CREATE OR REPLACE FUNCTION log_levels()
RETURNS TRIGGER AS $$
BEGIN
  RAISE DEBUG 'Debug level';      -- 개발 시
  RAISE LOG 'Log level';          -- 서버 로그
  RAISE INFO 'Info level';        -- 정보성
  RAISE NOTICE 'Notice level';    -- 사용자 알림
  RAISE WARNING 'Warning level';  -- 경고
  RAISE EXCEPTION 'Error level';  -- 에러 (롤백)

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 로그 레벨 설정
SET client_min_messages = DEBUG;
```

---

## 20.11 Trigger 안티패턴

### 1. 재귀 Trigger

```sql
-- ❌ 무한 루프!
CREATE OR REPLACE FUNCTION recursive_trigger()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE users SET counter = counter + 1 WHERE id = NEW.id;
  -- 이 UPDATE가 다시 Trigger 발생 → 무한 루프!
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ✅ 해결: 조건 추가
CREATE OR REPLACE FUNCTION safe_trigger()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.counter != OLD.counter + 1 THEN
    NEW.counter = OLD.counter + 1;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 2. 복잡한 비즈니스 로직

```sql
-- ❌ Trigger에 너무 많은 로직
CREATE OR REPLACE FUNCTION complex_trigger()
RETURNS TRIGGER AS $$
BEGIN
  -- 외부 API 호출
  -- 복잡한 계산
  -- 여러 테이블 업데이트
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ✅ 애플리케이션에서 처리
-- Trigger는 간단한 작업만 (타임스탬프, 로그 등)
```

### 3. 성능 무시

```sql
-- ❌ 무거운 작업
CREATE TRIGGER trg_heavy
AFTER INSERT ON logs
FOR EACH ROW
EXECUTE FUNCTION send_email();  -- 느림!

-- ✅ 큐에 넣고 백그라운드 처리
CREATE TRIGGER trg_enqueue
AFTER INSERT ON logs
FOR EACH ROW
EXECUTE FUNCTION enqueue_notification();
```

---

## 핵심 요약

### Trigger 기본 구조

```sql
-- 함수 생성
CREATE OR REPLACE FUNCTION function_name()
RETURNS TRIGGER AS $$
BEGIN
  -- 로직
  RETURN NEW;  -- 또는 OLD, NULL
END;
$$ LANGUAGE plpgsql;

-- Trigger 생성
CREATE TRIGGER trigger_name
{ BEFORE | AFTER | INSTEAD OF }
{ INSERT | UPDATE | DELETE }
ON table_name
[ FOR EACH { ROW | STATEMENT } ]
[ WHEN ( condition ) ]
EXECUTE FUNCTION function_name();
```

### 특수 변수

```
OLD    - 변경 전 데이터
NEW    - 변경 후 데이터
TG_OP  - INSERT/UPDATE/DELETE
TG_WHEN    - BEFORE/AFTER
TG_LEVEL   - ROW/STATEMENT
TG_TABLE_NAME - 테이블 이름
```

### 실행 순서

```
1. BEFORE STATEMENT
2. BEFORE ROW (각 행)
3. 실제 작업
4. AFTER ROW (각 행)
5. AFTER STATEMENT
```

### 사용 시나리오

```
✅ 적합:
- 자동 타임스탬프
- 감사 로그
- 데이터 검증
- 자동 계산

❌ 부적합:
- 복잡한 비즈니스 로직
- 외부 API 호출
- 무거운 작업
```

---

## 다음 단계

Trigger를 마스터했으니, 이제 **Stored Procedure**를 배워봅시다!

👉 [Chapter 21. Stored Procedure](21-stored-procedure.md)

---

## 실습 과제

### 과제 1: 기본 Trigger

```sql
-- 1. updated_at 자동 갱신 Trigger
-- 2. 감사 로그 Trigger
-- 3. 테스트
```

### 과제 2: 비즈니스 로직

```sql
-- 1. 재고 자동 차감 Trigger
-- 2. 재고 부족 시 에러
-- 3. 주문 취소 시 재고 복구
```

### 과제 3: 복잡한 Trigger

```sql
-- 1. 소프트 삭제 Trigger
-- 2. 슬러그 자동 생성
-- 3. 중복 처리
```

### 과제 4: 성능 최적화

```sql
-- 1. FOR EACH ROW vs STATEMENT 비교
-- 2. WHEN 조건 추가
-- 3. 성능 측정
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자, DBA
**중요도**: ⭐⭐⭐⭐ 매우 중요!
