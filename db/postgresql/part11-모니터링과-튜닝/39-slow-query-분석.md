# Chapter 39. Slow Query 분석

## 🎯 이 챕터의 목표
- **느린 쿼리** 찾고 분석하기
- **EXPLAIN ANALYZE** 심화 활용하기
- **실행 계획** 최적화하기
- **auto_explain** 확장으로 자동 로깅하기
- **쿼리 튜닝** 실전 사례 익히기

---

## 1. 느린 쿼리 찾기

### 🔍 로깅 설정

```ini
# postgresql.conf

# 느린 쿼리 로깅
log_min_duration_statement = 1000  # 1초 이상 쿼리 로깅
log_line_prefix = '%t [%p] %u@%d '
log_statement = 'none'
log_duration = off

# 재시작
sudo systemctl reload postgresql
```

### 📊 pg_stat_statements로 찾기

```sql
-- 가장 느린 쿼리 TOP 10
SELECT
  calls,
  ROUND(total_exec_time::numeric, 2) AS total_time_ms,
  ROUND(mean_exec_time::numeric, 2) AS avg_time_ms,
  ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 2) AS percent,
  query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

## 2. EXPLAIN ANALYZE 심화

```sql
-- 기본 실행 계획
EXPLAIN
SELECT * FROM orders WHERE user_id = 100;

-- 실제 실행 + 통계
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 100;

-- 출력:
-- Index Scan using idx_orders_user_id on orders
--   (cost=0.42..8.44 rows=1 width=45)
--   (actual time=0.025..0.026 rows=1 loops=1)
--   Buffers: shared hit=4
-- Planning Time: 0.123 ms
-- Execution Time: 0.045 ms
```

### 실행 계획 읽는 법

```
Seq Scan: 전체 테이블 스캔 (느림) ❌
Index Scan: 인덱스 사용 (빠름) ✅
Index Only Scan: 인덱스만 (가장 빠름) ✅✅
Bitmap Heap Scan: 대량 행 처리
Hash Join: 해시 조인
Nested Loop: 중첩 루프 조인
```

---

## 3. 쿼리 최적화 사례

### Case 1: N+1 Problem

```sql
-- ❌ N+1 쿼리
SELECT * FROM users;  -- 1번
-- 각 user마다:
SELECT * FROM orders WHERE user_id = ?;  -- N번

-- ✅ JOIN으로 해결
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

### Case 2: 인덱스 미사용

```sql
-- ❌ 함수 사용으로 인덱스 미사용
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- ✅ 함수 인덱스 생성
CREATE INDEX idx_users_email_lower ON users (LOWER(email));

-- 또는 원본 데이터를 소문자로 저장
```

### Case 3: SELECT *

```sql
-- ❌ 불필요한 컬럼 조회
SELECT * FROM large_table;

-- ✅ 필요한 컬럼만
SELECT id, name, email FROM large_table;
```

---

## 4. auto_explain 확장

```ini
# postgresql.conf

# auto_explain 활성화
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000  # 1초 이상
auto_explain.log_analyze = true
auto_explain.log_buffers = true
auto_explain.log_timing = true
auto_explain.log_nested_statements = true

# 재시작 필요
sudo systemctl restart postgresql
```

---

## 📖 정리

| 도구 | 용도 |
|------|----------|
| **EXPLAIN** | 실행 계획 확인 |
| **EXPLAIN ANALYZE** | 실제 실행 + 통계 |
| **pg_stat_statements** | 쿼리별 성능 통계 |
| **auto_explain** | 자동 로깅 |

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐ (고급)
**학습 시간**: 90분

이전: [Chapter 38. pg_stat으로 모니터링](38-pg_stat으로-모니터링.md)
다음: [Chapter 40. VACUUM과 ANALYZE](40-vacuum과-analyze.md)