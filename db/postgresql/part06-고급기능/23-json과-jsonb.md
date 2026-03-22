# Chapter 23. JSON과 JSONB

## 23.1 JSON과 JSONB란?

### 정의

```
JSON (JavaScript Object Notation):
텍스트 형식으로 저장, 입력 그대로 유지

JSONB (JSON Binary):
바이너리 형식으로 저장, 효율적인 처리

비유: 문서 보관
┌─────────────────────────────────┐
│ JSON:                           │
│ - 종이 문서 (텍스트)            │
│ - 빠른 입력                     │
│ - 느린 검색                     │
├─────────────────────────────────┤
│ JSONB:                          │
│ - 정리된 파일 캐비닛            │
│ - 느린 입력 (정리 필요)         │
│ - 빠른 검색 (인덱스 가능)       │
└─────────────────────────────────┘

대부분의 경우 JSONB 사용 권장!
```

### JSON vs JSONB

```sql
-- JSON: 텍스트 그대로 저장
SELECT '{"name": "Alice",  "age": 25}'::json;
-- {"name": "Alice",  "age": 25}
-- 공백 유지, 순서 유지

-- JSONB: 바이너리로 저장 (정규화)
SELECT '{"name": "Alice",  "age": 25}'::jsonb;
-- {"age": 25, "name": "Alice"}
-- 공백 제거, 키 정렬
```

### 차이점 비교

```
특징        | JSON          | JSONB
------------|---------------|------------------
저장 형식   | 텍스트        | 바이너리
입력 속도   | 빠름          | 느림 (파싱 필요)
조회 속도   | 느림          | 빠름
인덱스      | 불가          | 가능 (GIN)
중복 키     | 허용          | 마지막 값만
공백/순서   | 유지          | 제거/정렬
크기        | 큼            | 작음

권장: JSONB (대부분의 경우)
```

---

## 23.2 기본 사용

### 데이터 저장

```sql
-- 테이블 생성
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  metadata JSONB
);

-- 데이터 삽입
INSERT INTO users (name, metadata) VALUES
('Alice', '{"age": 25, "city": "Seoul", "hobbies": ["reading", "coding"]}'),
('Bob', '{"age": 30, "city": "Busan", "premium": true}');

-- 조회
SELECT * FROM users;
--  id | name  |                    metadata
-- ----+-------+-----------------------------------------------
--   1 | Alice | {"age": 25, "city": "Seoul", "hobbies": [...]}
--   2 | Bob   | {"age": 30, "city": "Busan", "premium": true}
```

### 데이터 접근

```sql
-- -> 연산자: JSON 객체 반환
SELECT metadata -> 'age' FROM users;
--  ?column?
-- ----------
--  25
--  30

-- ->> 연산자: 텍스트 반환
SELECT metadata ->> 'age' FROM users;
--  ?column?
-- ----------
--  25
--  30

-- 중첩 접근
SELECT metadata -> 'hobbies' -> 0 FROM users WHERE name = 'Alice';
--  ?column?
-- ----------
--  "reading"

-- 경로 접근 (#> 연산자)
SELECT metadata #> '{hobbies, 0}' FROM users WHERE name = 'Alice';
--  ?column?
-- ----------
--  "reading"

-- 텍스트로 반환 (#>> 연산자)
SELECT metadata #>> '{hobbies, 0}' FROM users WHERE name = 'Alice';
--  ?column?
-- ----------
--  reading
```

---

## 23.3 JSONB 연산자

### 포함 연산자 (@>, <@)

```sql
-- @> : 왼쪽이 오른쪽을 포함
SELECT * FROM users
WHERE metadata @> '{"city": "Seoul"}';
--  id | name  | metadata
-- ----+-------+----------
--   1 | Alice | {...}

-- 여러 조건
SELECT * FROM users
WHERE metadata @> '{"age": 25, "city": "Seoul"}';

-- <@ : 오른쪽이 왼쪽을 포함 (반대)
SELECT '{"age": 25}'::jsonb <@ '{"age": 25, "city": "Seoul"}'::jsonb;
-- true
```

### 존재 연산자 (?, ?|, ?&)

```sql
-- ? : 키 존재 여부
SELECT * FROM users WHERE metadata ? 'premium';
--  id | name | metadata
-- ----+------+----------
--   2 | Bob  | {...}

-- ?| : 여러 키 중 하나라도 존재
SELECT * FROM users WHERE metadata ?| array['premium', 'vip'];

-- ?& : 모든 키가 존재
SELECT * FROM users WHERE metadata ?& array['age', 'city'];
```

### 삭제 연산자 (-, #-)

```sql
-- - : 키 삭제
SELECT metadata - 'age' FROM users WHERE name = 'Alice';
-- {"city": "Seoul", "hobbies": [...]}

-- 배열 요소 삭제
SELECT metadata - 'hobbies' - 0 FROM users WHERE name = 'Alice';

-- #- : 경로로 삭제
SELECT metadata #- '{hobbies, 0}' FROM users WHERE name = 'Alice';
-- {"age": 25, "city": "Seoul", "hobbies": ["coding"]}
```

### 연결 연산자 (||)

```sql
-- || : 병합
SELECT
  metadata || '{"country": "Korea"}'::jsonb
FROM users WHERE name = 'Alice';
-- {"age": 25, "city": "Seoul", "hobbies": [...], "country": "Korea"}

-- 중복 키는 덮어쓰기
SELECT
  metadata || '{"age": 26}'::jsonb
FROM users WHERE name = 'Alice';
-- {"age": 26, "city": "Seoul", "hobbies": [...]}
```

---

## 23.4 JSONB 함수

### 객체/배열 생성

```sql
-- jsonb_build_object: 객체 생성
SELECT jsonb_build_object('name', 'Alice', 'age', 25);
-- {"age": 25, "name": "Alice"}

-- jsonb_build_array: 배열 생성
SELECT jsonb_build_array('a', 'b', 'c');
-- ["a", "b", "c"]

-- jsonb_object: 키-값 쌍에서 객체 생성
SELECT jsonb_object('{name, age}', '{Alice, 25}');
-- {"age": "25", "name": "Alice"}
```

### 객체/배열 조작

```sql
-- jsonb_set: 값 설정
UPDATE users
SET metadata = jsonb_set(
  metadata,
  '{age}',
  '26'::jsonb
)
WHERE name = 'Alice';

-- 중첩 경로
UPDATE users
SET metadata = jsonb_set(
  metadata,
  '{address, city}',
  '"Seoul"'::jsonb,
  true  -- create_missing = true
)
WHERE name = 'Alice';

-- jsonb_insert: 삽입
SELECT jsonb_insert(
  '{"a": 1, "c": 3}'::jsonb,
  '{b}',
  '2'::jsonb
);
-- {"a": 1, "b": 2, "c": 3}
```

### 배열 함수

```sql
-- jsonb_array_length: 배열 길이
SELECT jsonb_array_length(metadata -> 'hobbies')
FROM users WHERE name = 'Alice';
-- 2

-- jsonb_array_elements: 배열 요소 분해
SELECT jsonb_array_elements(metadata -> 'hobbies')
FROM users WHERE name = 'Alice';
--  jsonb_array_elements
-- ----------------------
--  "reading"
--  "coding"

-- jsonb_array_elements_text: 텍스트로
SELECT jsonb_array_elements_text(metadata -> 'hobbies')
FROM users WHERE name = 'Alice';
--  jsonb_array_elements_text
-- ---------------------------
--  reading
--  coding
```

### 키/값 분해

```sql
-- jsonb_each: 키-값 쌍 분해
SELECT * FROM jsonb_each('{"a": 1, "b": 2}'::jsonb);
--  key | value
-- -----+-------
--  a   | 1
--  b   | 2

-- jsonb_each_text: 텍스트로
SELECT * FROM jsonb_each_text('{"a": 1, "b": 2}'::jsonb);
--  key | value
-- -----+-------
--  a   | 1
--  b   | 2

-- jsonb_object_keys: 키만
SELECT jsonb_object_keys('{"a": 1, "b": 2}'::jsonb);
--  jsonb_object_keys
-- -------------------
--  a
--  b
```

---

## 23.5 인덱스

### GIN 인덱스 (기본)

```sql
-- GIN 인덱스 생성
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);

-- 검색 (인덱스 사용)
SELECT * FROM users WHERE metadata @> '{"city": "Seoul"}';
-- Index Scan using idx_users_metadata

-- 존재 검색
SELECT * FROM users WHERE metadata ? 'premium';
-- Index Scan
```

### 특정 키 인덱스

```sql
-- 특정 경로만 인덱스
CREATE INDEX idx_users_city ON users ((metadata ->> 'city'));

-- 검색
SELECT * FROM users WHERE metadata ->> 'city' = 'Seoul';
-- Index Scan using idx_users_city

-- 숫자 비교를 위한 인덱스
CREATE INDEX idx_users_age ON users (((metadata ->> 'age')::INT));

SELECT * FROM users WHERE (metadata ->> 'age')::INT > 20;
-- Index Scan using idx_users_age
```

### GIN 인덱스 연산자 클래스

```sql
-- jsonb_ops (기본): @>, ?, ?|, ?&
CREATE INDEX idx_metadata ON users USING GIN(metadata jsonb_ops);

-- jsonb_path_ops: @> 만 (더 작고 빠름)
CREATE INDEX idx_metadata_path ON users USING GIN(metadata jsonb_path_ops);

-- 크기 비교
SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE tablename = 'users';
--  indexname           | size
-- ---------------------+-------
--  idx_metadata        | 10 MB
--  idx_metadata_path   | 7 MB
```

---

## 23.6 실전 예제

### 예제 1: 사용자 프로필

```sql
-- 테이블
CREATE TABLE user_profiles (
  id SERIAL PRIMARY KEY,
  user_id INT UNIQUE NOT NULL,
  profile JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_user_profiles_profile ON user_profiles USING GIN(profile);

-- 프로필 삽입
INSERT INTO user_profiles (user_id, profile) VALUES
(1, '{
  "name": "Alice",
  "age": 25,
  "email": "alice@example.com",
  "settings": {
    "theme": "dark",
    "notifications": true,
    "language": "ko"
  },
  "interests": ["coding", "reading", "music"]
}');

-- 프로필 조회
SELECT
  user_id,
  profile ->> 'name' AS name,
  profile ->> 'email' AS email,
  profile -> 'settings' ->> 'theme' AS theme
FROM user_profiles;

-- 프로필 업데이트
UPDATE user_profiles
SET
  profile = jsonb_set(profile, '{age}', '26'::jsonb),
  updated_at = NOW()
WHERE user_id = 1;

-- 중첩 업데이트
UPDATE user_profiles
SET profile = jsonb_set(
  profile,
  '{settings, theme}',
  '"light"'::jsonb
)
WHERE user_id = 1;

-- 배열에 추가
UPDATE user_profiles
SET profile = jsonb_set(
  profile,
  '{interests}',
  (profile -> 'interests') || '"gaming"'::jsonb
)
WHERE user_id = 1;
```

### 예제 2: 제품 속성

```sql
-- 제품 테이블
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  category VARCHAR(100),
  price NUMERIC(10, 2),
  attributes JSONB  -- 제품별로 다른 속성
);

-- 인덱스
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- 데이터 삽입
INSERT INTO products (name, category, price, attributes) VALUES
('Laptop', 'Electronics', 1200000, '{
  "brand": "Dell",
  "cpu": "i7",
  "ram": "16GB",
  "storage": "512GB SSD",
  "screen": "15.6inch"
}'),
('T-Shirt', 'Clothing', 29000, '{
  "brand": "Nike",
  "size": "L",
  "color": "blue",
  "material": "cotton"
}'),
('Book', 'Books', 15000, '{
  "author": "John Doe",
  "pages": 350,
  "publisher": "ABC Press",
  "isbn": "1234567890"
}');

-- 속성으로 검색
SELECT name, price, attributes
FROM products
WHERE attributes @> '{"brand": "Dell"}';

-- 여러 속성
SELECT name, price
FROM products
WHERE attributes @> '{"cpu": "i7", "ram": "16GB"}';

-- 속성 존재 여부
SELECT name, category
FROM products
WHERE attributes ? 'isbn';

-- 범위 검색 (가격 속성이 있는 경우)
SELECT name, attributes ->> 'pages' AS pages
FROM products
WHERE (attributes ->> 'pages')::INT > 300;
```

### 예제 3: 이벤트 로그

```sql
-- 이벤트 로그 테이블
CREATE TABLE event_logs (
  id BIGSERIAL PRIMARY KEY,
  user_id INT,
  event_type VARCHAR(50) NOT NULL,
  event_data JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 파티셔닝 (날짜별)
-- CREATE TABLE event_logs (...) PARTITION BY RANGE (created_at);

-- 인덱스
CREATE INDEX idx_event_logs_type ON event_logs(event_type);
CREATE INDEX idx_event_logs_data ON event_logs USING GIN(event_data);
CREATE INDEX idx_event_logs_created ON event_logs(created_at);

-- 이벤트 삽입
INSERT INTO event_logs (user_id, event_type, event_data) VALUES
(1, 'page_view', '{
  "page": "/products/123",
  "referrer": "google.com",
  "device": "mobile",
  "browser": "Chrome"
}'),
(1, 'purchase', '{
  "product_id": 123,
  "quantity": 2,
  "total_amount": 50000,
  "payment_method": "card"
}'),
(2, 'signup', '{
  "method": "google",
  "referral_code": "FRIEND123"
}');

-- 이벤트 분석
SELECT
  event_type,
  COUNT(*) AS count,
  COUNT(DISTINCT user_id) AS unique_users
FROM event_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY event_type
ORDER BY count DESC;

-- 특정 조건 이벤트
SELECT
  user_id,
  event_data ->> 'total_amount' AS amount,
  created_at
FROM event_logs
WHERE event_type = 'purchase'
  AND (event_data ->> 'total_amount')::NUMERIC > 100000
ORDER BY created_at DESC;

-- 퍼널 분석
WITH funnel AS (
  SELECT
    user_id,
    MAX(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) AS viewed,
    MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS added,
    MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased
  FROM event_logs
  WHERE created_at >= NOW() - INTERVAL '30 days'
  GROUP BY user_id
)
SELECT
  SUM(viewed) AS total_views,
  SUM(added) AS total_adds,
  SUM(purchased) AS total_purchases,
  ROUND(SUM(added)::NUMERIC / SUM(viewed) * 100, 2) AS add_rate,
  ROUND(SUM(purchased)::NUMERIC / SUM(added) * 100, 2) AS purchase_rate
FROM funnel;
```

### 예제 4: 설정 관리

```sql
-- 애플리케이션 설정 테이블
CREATE TABLE app_settings (
  id SERIAL PRIMARY KEY,
  key VARCHAR(100) UNIQUE NOT NULL,
  value JSONB NOT NULL,
  description TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 설정 삽입
INSERT INTO app_settings (key, value, description) VALUES
('email', '{
  "smtp_host": "smtp.gmail.com",
  "smtp_port": 587,
  "from_email": "noreply@example.com",
  "use_tls": true
}', 'Email configuration'),
('payment', '{
  "providers": ["stripe", "paypal"],
  "test_mode": false,
  "currencies": ["KRW", "USD"]
}', 'Payment settings'),
('features', '{
  "new_ui": true,
  "beta_features": false,
  "maintenance_mode": false
}', 'Feature flags');

-- 설정 조회 함수
CREATE OR REPLACE FUNCTION get_setting(setting_key VARCHAR, json_path TEXT)
RETURNS TEXT AS $$
DECLARE
  result TEXT;
BEGIN
  SELECT value #>> string_to_array(json_path, '.')
  INTO result
  FROM app_settings
  WHERE key = setting_key;

  RETURN result;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT get_setting('email', 'smtp_host');
-- smtp.gmail.com

-- 설정 업데이트
UPDATE app_settings
SET
  value = jsonb_set(value, '{test_mode}', 'true'::jsonb),
  updated_at = NOW()
WHERE key = 'payment';
```

---

## 23.7 JSON vs 정규화

### 언제 JSON 사용?

```
✅ JSON 사용:
- 스키마가 자주 변경
- 제품별로 다른 속성
- 이벤트 로그 (구조가 다양)
- 외부 API 응답 저장
- 설정/메타데이터

❌ JSON 사용 피하기:
- 자주 JOIN하는 데이터
- 집계가 많은 데이터
- 정규화된 관계가 명확한 경우
```

### 하이브리드 접근

```sql
-- 핵심 데이터는 컬럼, 추가 데이터는 JSON
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,  -- 자주 검색
  name VARCHAR(100) NOT NULL,           -- 자주 사용
  created_at TIMESTAMPTZ DEFAULT NOW(), -- 자주 필터링
  metadata JSONB DEFAULT '{}'::jsonb    -- 선택적 데이터
);

-- 장점:
-- - 핵심 데이터는 빠른 검색
-- - 유연한 추가 데이터
```

---

## 23.8 성능 최적화

### 통계 확인

```sql
-- JSONB 컬럼 통계
SELECT
  tablename,
  attname,
  n_distinct,
  avg_width
FROM pg_stats
WHERE tablename = 'users' AND attname = 'metadata';
```

### 파티셔닝

```sql
-- 날짜별 파티셔닝 + JSONB
CREATE TABLE events (
  id BIGSERIAL,
  event_type VARCHAR(50),
  event_data JSONB,
  created_at TIMESTAMPTZ
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- 각 파티션에 인덱스
CREATE INDEX idx_events_2024_01_data
ON events_2024_01 USING GIN(event_data);
```

### 컬럼 추출

```sql
-- 자주 쿼리하는 JSONB 필드 → 컬럼으로
ALTER TABLE users ADD COLUMN city VARCHAR(100)
GENERATED ALWAYS AS (metadata ->> 'city') STORED;

CREATE INDEX idx_users_city ON users(city);

-- 이제 빠른 검색
SELECT * FROM users WHERE city = 'Seoul';
-- B-Tree 인덱스 사용!
```

---

## 핵심 요약

### JSON vs JSONB

```
JSON:  텍스트, 빠른 입력, 느린 검색
JSONB: 바이너리, 느린 입력, 빠른 검색, 인덱스 가능

권장: JSONB (대부분의 경우)
```

### 주요 연산자

```sql
->   : JSON 객체 반환
->>  : 텍스트 반환
#>   : 경로로 JSON 객체
#>>  : 경로로 텍스트
@>   : 포함 (왼쪽이 오른쪽 포함)
?    : 키 존재
||   : 병합
-    : 키 삭제
```

### 주요 함수

```sql
jsonb_set()              -- 값 설정
jsonb_build_object()     -- 객체 생성
jsonb_array_elements()   -- 배열 분해
jsonb_each()             -- 키-값 분해
jsonb_object_keys()      -- 키 목록
```

### 인덱스

```sql
-- GIN 인덱스 (전체)
CREATE INDEX idx_name ON table USING GIN(jsonb_column);

-- 특정 키 인덱스
CREATE INDEX idx_name ON table ((jsonb_column ->> 'key'));

-- 숫자 인덱스
CREATE INDEX idx_name ON table (((jsonb_column ->> 'key')::INT));
```

---

## 다음 단계

JSON과 JSONB를 마스터했으니, Part 6이 완료되었습니다!
이제 **EXPLAIN**을 배워봅시다!

👉 [Chapter 24. EXPLAIN](../part07-성능최적화/24-explain.md)

---

## 실습 과제

### 과제 1: 기본 JSONB

```sql
-- 1. JSONB 컬럼 테이블 생성
-- 2. 데이터 삽입
-- 3. 연산자로 조회 (->, ->>, @>)
```

### 과제 2: 인덱스

```sql
-- 1. GIN 인덱스 생성
-- 2. 검색 쿼리 작성
-- 3. EXPLAIN으로 인덱스 사용 확인
```

### 과제 3: 실전 활용

```sql
-- 1. 제품 속성 테이블
-- 2. 다양한 속성 저장
-- 3. 속성으로 검색
-- 4. 속성 업데이트
```

### 과제 4: 성능 비교

```sql
-- 1. JSON vs JSONB 성능 비교
-- 2. 인덱스 유무 성능 비교
-- 3. 정규화 vs JSONB 비교
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐⭐⭐ 필수!
