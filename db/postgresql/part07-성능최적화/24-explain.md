# Chapter 24. EXPLAIN

## 24.1 EXPLAIN이란?

### 정의

```
EXPLAIN:
쿼리 실행 계획(Execution Plan)을 보여주는 명령

비유: 내비게이션 경로
┌─────────────────────────────────┐
│ 목적지: 서울역                  │
│                                 │
│ 경로 1: 고속도로 (빠름)         │
│ 경로 2: 국도 (느림)             │
│ 경로 3: 막힌 길 (매우 느림)     │
│                                 │
│ EXPLAIN = 어떤 경로 선택했는지  │
└─────────────────────────────────┘

쿼리 최적화의 시작점!
```

### EXPLAIN의 종류

```sql
-- EXPLAIN: 계획만 확인 (실행 X)
EXPLAIN SELECT * FROM users;

-- EXPLAIN ANALYZE: 실제 실행 + 통계
EXPLAIN ANALYZE SELECT * FROM users;

-- EXPLAIN (FORMAT JSON): JSON 형식
EXPLAIN (FORMAT JSON) SELECT * FROM users;

-- EXPLAIN (BUFFERS, ANALYZE): 버퍼 통계 포함
EXPLAIN (BUFFERS, ANALYZE) SELECT * FROM users;
```

---

## 24.2 기본 EXPLAIN

### Seq Scan (순차 스캔)

```sql
-- 인덱스 없는 테이블
EXPLAIN SELECT * FROM users WHERE age > 20;

-- Seq Scan on users (cost=0.00..25833.00 rows=500000 width=45)
--   Filter: (age > 20)

-- 해석:
-- - Seq Scan: 테이블 전체 읽기
-- - cost: 0.00 (시작 비용) .. 25833.00 (총 비용)
-- - rows: 예상 반환 행 수 (500,000)
-- - width: 평균 행 크기 (45 bytes)
-- - Filter: WHERE 조건
```

### Index Scan (인덱스 스캔)

```sql
-- 인덱스 있는 테이블
CREATE INDEX idx_users_age ON users(age);

EXPLAIN SELECT * FROM users WHERE age = 25;

-- Index Scan using idx_users_age on users (cost=0.42..8.44 rows=1 width=45)
--   Index Cond: (age = 25)

-- 해석:
-- - Index Scan: 인덱스 사용
-- - cost: 0.42 .. 8.44 (매우 낮음!)
-- - rows: 1
-- - Index Cond: 인덱스 조건
```

---

## 24.3 EXPLAIN ANALYZE

### 실제 실행 통계

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE age = 25;

-- Index Scan using idx_users_age on users
--   (cost=0.42..8.44 rows=1 width=45)
--   (actual time=0.025..0.026 rows=1 loops=1)
--   Index Cond: (age = 25)
-- Planning Time: 0.123 ms
-- Execution Time: 0.045 ms

-- actual time: 실제 실행 시간 (ms)
-- rows: 실제 반환 행 수
-- loops: 반복 횟수
-- Planning Time: 계획 수립 시간
-- Execution Time: 실제 실행 시간
```

### 예상 vs 실제

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE name LIKE 'A%';

-- Seq Scan on users
--   (cost=0.00..25833.00 rows=5000 width=45)    -- 예상: 5,000행
--   (actual time=0.1..150.5 rows=50000 loops=1) -- 실제: 50,000행
-- Planning Time: 0.2 ms
-- Execution Time: 155.3 ms

-- 예상과 실제가 크게 다름!
-- → 통계 업데이트 필요: ANALYZE users;
```

---

## 24.4 주요 스캔 방식

### Seq Scan (순차 스캔)

```sql
EXPLAIN ANALYZE SELECT * FROM users;

-- Seq Scan on users (cost=0.00..18334.00 rows=1000000 width=45)
--   (actual time=0.01..120.5 rows=1000000 loops=1)
-- Execution Time: 150.2 ms

-- 언제 사용?
-- - 인덱스 없음
-- - 테이블이 작음
-- - 대부분의 행 반환 (> 5-10%)
```

### Index Scan

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE id = 1;

-- Index Scan using users_pkey on users (cost=0.42..8.44 rows=1 width=45)
--   (actual time=0.02..0.03 rows=1 loops=1)
--   Index Cond: (id = 1)
-- Execution Time: 0.05 ms

-- 언제 사용?
-- - PRIMARY KEY, UNIQUE 검색
-- - 소수의 행 반환 (< 5%)
```

### Index Only Scan

```sql
-- 커버링 인덱스
CREATE INDEX idx_users_email_name ON users(email, name);

EXPLAIN ANALYZE
SELECT email, name FROM users WHERE email = 'alice@example.com';

-- Index Only Scan using idx_users_email_name
--   (cost=0.42..4.44 rows=1 width=25)
--   (actual time=0.01..0.02 rows=1 loops=1)
--   Index Cond: (email = 'alice@example.com')
--   Heap Fetches: 0   -- 테이블 접근 안 함!
-- Execution Time: 0.03 ms

-- 가장 빠름!
```

### Bitmap Index Scan

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE age > 20 AND age < 40;

-- Bitmap Heap Scan on users (cost=50.00..1250.00 rows=10000 width=45)
--   (actual time=5.2..25.3 rows=10000 loops=1)
--   Recheck Cond: ((age > 20) AND (age < 40))
--   Heap Blocks: exact=100
--   -> Bitmap Index Scan on idx_users_age
--        (cost=0.00..47.50 rows=10000 width=0)
--        (actual time=2.1..2.1 rows=10000 loops=1)
--        Index Cond: ((age > 20) AND (age < 40))
-- Execution Time: 28.5 ms

-- 언제 사용?
-- - 중간 정도 행 반환 (5-50%)
-- - 여러 인덱스 조합
```

---

## 24.5 JOIN 방식

### Nested Loop Join

```sql
EXPLAIN ANALYZE
SELECT u.name, p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id
WHERE u.id = 1;

-- Nested Loop (cost=0.85..16.88 rows=5 width=50)
--   (actual time=0.05..0.1 rows=5 loops=1)
--   -> Index Scan using users_pkey on users u
--        (cost=0.42..8.44 rows=1 width=25)
--        (actual time=0.02..0.03 rows=1 loops=1)
--        Index Cond: (id = 1)
--   -> Index Scan using idx_posts_user_id on posts p
--        (cost=0.42..8.36 rows=5 width=25)
--        (actual time=0.02..0.05 rows=5 loops=1)
--        Index Cond: (user_id = 1)
-- Execution Time: 0.15 ms

-- 언제 사용?
-- - 작은 테이블 JOIN
-- - 외부 테이블이 매우 작음
-- - 인덱스 있음
```

### Hash Join

```sql
EXPLAIN ANALYZE
SELECT u.name, p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id;

-- Hash Join (cost=35000.00..125000.00 rows=1000000 width=50)
--   (actual time=150.2..850.5 rows=1000000 loops=1)
--   Hash Cond: (p.user_id = u.id)
--   -> Seq Scan on posts p
--        (cost=0.00..50000.00 rows=1000000 width=25)
--        (actual time=0.01..250.3 rows=1000000 loops=1)
--   -> Hash (cost=20000.00..20000.00 rows=1000000 width=25)
--        (actual time=145.1..145.1 rows=1000000 loops=1)
--        Buckets: 131072  Batches: 16  Memory Usage: 5000kB
--        -> Seq Scan on users u
--             (cost=0.00..20000.00 rows=1000000 width=25)
--             (actual time=0.01..80.5 rows=1000000 loops=1)
-- Execution Time: 900.2 ms

-- 언제 사용?
-- - 큰 테이블 JOIN
-- - 인덱스 없음
-- - 등호 조인
```

### Merge Join

```sql
EXPLAIN ANALYZE
SELECT u.name, p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id
ORDER BY u.id;

-- Merge Join (cost=0.85..75000.00 rows=1000000 width=50)
--   (actual time=0.05..650.5 rows=1000000 loops=1)
--   Merge Cond: (u.id = p.user_id)
--   -> Index Scan using users_pkey on users u
--        (cost=0.42..30000.00 rows=1000000 width=25)
--        (actual time=0.02..250.3 rows=1000000 loops=1)
--   -> Index Scan using idx_posts_user_id on posts p
--        (cost=0.42..40000.00 rows=1000000 width=25)
--        (actual time=0.02..350.2 rows=1000000 loops=1)
-- Execution Time: 700.3 ms

-- 언제 사용?
-- - 정렬된 데이터
-- - 인덱스 있음
-- - ORDER BY와 함께
```

---

## 24.6 집계 방식

### HashAggregate

```sql
EXPLAIN ANALYZE
SELECT user_id, COUNT(*) FROM posts GROUP BY user_id;

-- HashAggregate (cost=50000.00..52000.00 rows=100000 width=12)
--   (actual time=250.5..270.3 rows=100000 loops=1)
--   Group Key: user_id
--   Batches: 1  Memory Usage: 12000kB
--   -> Seq Scan on posts
--        (cost=0.00..40000.00 rows=1000000 width=4)
--        (actual time=0.01..150.2 rows=1000000 loops=1)
-- Execution Time: 280.5 ms

-- 언제 사용?
-- - 메모리에 해시 테이블 생성 가능
-- - work_mem 충분
```

### GroupAggregate

```sql
EXPLAIN ANALYZE
SELECT user_id, COUNT(*) FROM posts GROUP BY user_id ORDER BY user_id;

-- GroupAggregate (cost=0.42..60000.00 rows=100000 width=12)
--   (actual time=0.05..350.3 rows=100000 loops=1)
--   Group Key: user_id
--   -> Index Scan using idx_posts_user_id on posts
--        (cost=0.42..50000.00 rows=1000000 width=4)
--        (actual time=0.02..250.2 rows=1000000 loops=1)
-- Execution Time: 360.5 ms

-- 언제 사용?
-- - 정렬된 데이터 (인덱스)
-- - work_mem 부족
```

---

## 24.7 EXPLAIN 옵션

### VERBOSE

```sql
EXPLAIN (VERBOSE) SELECT name, email FROM users WHERE id = 1;

-- Index Scan using users_pkey on public.users
--   (cost=0.42..8.44 rows=1 width=50)
--   Output: name, email   -- 출력 컬럼 표시
--   Index Cond: (users.id = 1)
```

### BUFFERS

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE age = 25;

-- Index Scan using idx_users_age on users
--   (cost=0.42..8.44 rows=1 width=45)
--   (actual time=0.02..0.03 rows=1 loops=1)
--   Index Cond: (age = 25)
--   Buffers: shared hit=4   -- 캐시 히트!
-- Planning Time: 0.1 ms
-- Execution Time: 0.05 ms

-- Buffers 해석:
-- - shared hit: 캐시에서 읽음 (빠름)
-- - shared read: 디스크에서 읽음 (느림)
-- - shared written: 디스크에 씀
-- - temp read/written: 임시 파일 사용
```

### COSTS OFF

```sql
EXPLAIN (COSTS OFF) SELECT * FROM users WHERE id = 1;

-- Index Scan using users_pkey on users
--   Index Cond: (id = 1)

-- 비용 정보 제거 (가독성 향상)
```

### FORMAT JSON

```sql
EXPLAIN (FORMAT JSON, ANALYZE) SELECT * FROM users WHERE id = 1;

-- [
--   {
--     "Plan": {
--       "Node Type": "Index Scan",
--       "Relation Name": "users",
--       "Index Name": "users_pkey",
--       "Actual Total Time": 0.05,
--       "Actual Rows": 1
--     },
--     "Planning Time": 0.1,
--     "Execution Time": 0.05
--   }
-- ]

-- 프로그래밍 방식 분석에 유용
```

---

## 24.8 실전 최적화 예제

### 예제 1: 느린 쿼리 분석

```sql
-- 느린 쿼리
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 1000
  AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;

-- Limit (cost=50000.00..50000.10 rows=10 width=100)
--   (actual time=250.5..250.7 rows=10 loops=1)
--   -> Sort (cost=50000.00..51000.00 rows=100000 width=100)
--        (actual time=250.5..250.6 rows=10 loops=1)
--        Sort Key: created_at DESC
--        Sort Method: top-N heapsort  Memory: 25kB
--        -> Seq Scan on orders   -- 문제: 순차 스캔!
--             (cost=0.00..45000.00 rows=100000 width=100)
--             (actual time=0.1..180.5 rows=100000 loops=1)
--             Filter: ((user_id = 1000) AND (status = 'completed'))
--             Rows Removed by Filter: 900000
-- Planning Time: 0.2 ms
-- Execution Time: 250.8 ms

-- 문제점:
-- 1. Seq Scan (인덱스 없음)
-- 2. 100만 건 읽고 10만 건만 필터링
-- 3. 정렬 필요

-- 해결: 복합 인덱스
CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at DESC);

-- 다시 실행
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 1000
  AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;

-- Limit (cost=0.42..1.50 rows=10 width=100)
--   (actual time=0.05..0.1 rows=10 loops=1)
--   -> Index Scan using idx_orders_user_status_created
--        (cost=0.42..10800.00 rows=100000 width=100)
--        (actual time=0.05..0.08 rows=10 loops=1)
--        Index Cond: ((user_id = 1000) AND (status = 'completed'))
-- Planning Time: 0.3 ms
-- Execution Time: 0.15 ms

-- 1600배 빠름! (250.8ms → 0.15ms)
```

### 예제 2: JOIN 최적화

```sql
-- 느린 JOIN
EXPLAIN ANALYZE
SELECT u.name, COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.country = 'Korea'
GROUP BY u.id, u.name
ORDER BY post_count DESC
LIMIT 10;

-- Limit (cost=125000.00..125000.05 rows=10 width=50)
--   (actual time=850.5..850.7 rows=10 loops=1)
--   -> Sort (cost=125000.00..127500.00 rows=100000 width=50)
--        (actual time=850.5..850.6 rows=10 loops=1)
--        Sort Key: (count(p.id)) DESC
--        Sort Method: top-N heapsort  Memory: 25kB
--        -> HashAggregate (cost=100000.00..120000.00 rows=100000 width=50)
--             (actual time=750.3..820.5 rows=100000 loops=1)
--             Group Key: u.id
--             -> Hash Left Join (cost=35000.00..90000.00 rows=1000000 width=12)
--                  (actual time=150.2..650.5 rows=1000000 loops=1)
--                  Hash Cond: (p.user_id = u.id)
--                  -> Seq Scan on posts p   -- 문제!
--                       (cost=0.00..40000.00 rows=1000000 width=8)
--                       (actual time=0.01..250.3 rows=1000000 loops=1)
--                  -> Hash (cost=30000.00..30000.00 rows=100000 width=12)
--                       (actual time=145.1..145.1 rows=100000 loops=1)
--                       Buckets: 16384  Batches: 8  Memory Usage: 5000kB
--                       -> Seq Scan on users u   -- 문제!
--                            (cost=0.00..30000.00 rows=100000 width=12)
--                            (actual time=0.01..120.5 rows=100000 loops=1)
--                            Filter: (country = 'Korea')
--                            Rows Removed by Filter: 900000
-- Planning Time: 0.5 ms
-- Execution Time: 850.9 ms

-- 해결: 인덱스 추가
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_posts_user_id ON posts(user_id);

-- 다시 실행
EXPLAIN ANALYZE
SELECT u.name, COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.country = 'Korea'
GROUP BY u.id, u.name
ORDER BY post_count DESC
LIMIT 10;

-- Limit (cost=15000.00..15000.05 rows=10 width=50)
--   (actual time=50.5..50.7 rows=10 loops=1)
--   -> Sort (cost=15000.00..15250.00 rows=10000 width=50)
--        (actual time=50.5..50.6 rows=10 loops=1)
--        Sort Key: (count(p.id)) DESC
--        Sort Method: top-N heapsort  Memory: 25kB
--        -> HashAggregate (cost=12000.00..14000.00 rows=10000 width=50)
--             (actual time=45.3..48.5 rows=10000 loops=1)
--             Group Key: u.id
--             -> Nested Loop Left Join (cost=0.85..10000.00 rows=100000 width=12)
--                  (actual time=0.05..35.5 rows=100000 loops=1)
--                  -> Index Scan using idx_users_country on users u
--                       (cost=0.42..2000.00 rows=10000 width=12)
--                       (actual time=0.02..5.3 rows=10000 loops=1)
--                       Index Cond: (country = 'Korea')
--                  -> Index Scan using idx_posts_user_id on posts p
--                       (cost=0.42..0.75 rows=10 width=8)
--                       (actual time=0.002..0.003 rows=10 loops=10000)
--                       Index Cond: (user_id = u.id)
-- Planning Time: 0.3 ms
-- Execution Time: 50.9 ms

-- 17배 빠름! (850.9ms → 50.9ms)
```

---

## 24.9 통계와 ANALYZE

### 통계 중요성

```sql
-- 통계 오래됨
EXPLAIN SELECT * FROM users WHERE age = 25;
-- Seq Scan (cost=0.00..25833.00 rows=5000 width=45)
-- 예상: 5,000행

-- 실제 실행
EXPLAIN ANALYZE SELECT * FROM users WHERE age = 25;
-- Seq Scan (cost=0.00..25833.00 rows=5000 width=45)
--   (actual time=0.1..150.5 rows=50000 loops=1)
-- 실제: 50,000행 (10배 차이!)

-- 통계 업데이트
ANALYZE users;

-- 다시 확인
EXPLAIN SELECT * FROM users WHERE age = 25;
-- Bitmap Heap Scan (cost=1000.00..15000.00 rows=50000 width=45)
-- 이제 정확한 예상!
```

### 자동 ANALYZE 확인

```sql
-- 자동 ANALYZE 통계
SELECT
  schemaname,
  tablename,
  last_analyze,
  last_autoanalyze,
  n_live_tup,
  n_dead_tup
FROM pg_stat_user_tables
WHERE tablename = 'users';

--  tablename | last_analyze        | n_live_tup | n_dead_tup
-- -----------+---------------------+------------+------------
--  users     | 2024-01-15 10:00:00 | 1000000    | 5000
```

---

## 24.10 일반적인 문제와 해결

### 문제 1: Seq Scan

```sql
-- 문제
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
-- Seq Scan (느림)

-- 해결
CREATE INDEX idx_users_email ON users(email);
-- Index Scan (빠름)
```

### 문제 2: 잘못된 예상

```sql
-- 문제
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';
-- (cost=... rows=1000 width=...)   -- 예상: 1,000
-- (actual ... rows=500000 loops=1) -- 실제: 500,000

-- 해결
ANALYZE users;
-- 통계 업데이트
```

### 문제 3: 비효율적인 JOIN

```sql
-- 문제
-- Hash Join (느림, 메모리 많이 사용)

-- 해결
-- 1. 인덱스 추가 → Nested Loop Join
-- 2. work_mem 증가 → Hash Join 개선
SET work_mem = '256MB';
```

### 문제 4: 정렬 오버헤드

```sql
-- 문제
EXPLAIN ANALYZE SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;
-- Sort (cost=50000.00..52500.00 rows=1000000 width=100)
--   Sort Method: external merge  Disk: 40000kB   -- 디스크 사용!

-- 해결
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);
-- Index Scan (정렬 불필요)
```

---

## 핵심 요약

### EXPLAIN 기본

```sql
-- 계획만
EXPLAIN SELECT ...;

-- 실제 실행
EXPLAIN ANALYZE SELECT ...;

-- 버퍼 통계
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- JSON 형식
EXPLAIN (FORMAT JSON, ANALYZE) SELECT ...;
```

### 주요 스캔 방식

```
Seq Scan          - 전체 읽기 (느림)
Index Scan        - 인덱스 사용 (빠름)
Index Only Scan   - 인덱스만 (가장 빠름)
Bitmap Index Scan - 중간 (5-50% 행)
```

### JOIN 방식

```
Nested Loop - 작은 테이블, 인덱스 있음
Hash Join   - 큰 테이블, 등호 조인
Merge Join  - 정렬된 데이터
```

### 최적화 체크리스트

```
✅ 인덱스 있는가?
✅ 통계 최신인가? (ANALYZE)
✅ 예상 vs 실제 비슷한가?
✅ Seq Scan → Index Scan?
✅ 정렬 최소화?
```

---

## 다음 단계

EXPLAIN을 마스터했으니, 이제 **쿼리 최적화**를 배워봅시다!

👉 [Chapter 25. 쿼리 최적화](25-쿼리-최적화.md)

---

## 실습 과제

### 과제 1: 기본 EXPLAIN

```sql
-- 1. Seq Scan 쿼리 작성
-- 2. EXPLAIN 확인
-- 3. 인덱스 추가
-- 4. Index Scan 확인
```

### 과제 2: EXPLAIN ANALYZE

```sql
-- 1. 복잡한 쿼리 작성
-- 2. EXPLAIN ANALYZE 실행
-- 3. 예상 vs 실제 비교
-- 4. 병목 지점 찾기
```

### 과제 3: JOIN 최적화

```sql
-- 1. 느린 JOIN 쿼리 작성
-- 2. EXPLAIN ANALYZE로 분석
-- 3. 인덱스 추가
-- 4. 성능 개선 확인
```

### 과제 4: 실전 최적화

```sql
-- 1. 실제 프로젝트 느린 쿼리 선택
-- 2. EXPLAIN ANALYZE 분석
-- 3. 최적화 적용
-- 4. 개선 전후 비교
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자, DBA
**중요도**: ⭐⭐⭐⭐⭐ 필수!
