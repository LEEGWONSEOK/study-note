# Chapter 38. pg_stat으로 모니터링

## 🎯 이 챕터의 목표
- **pg_stat_activity**로 현재 세션 모니터링하기
- **pg_stat_database**로 데이터베이스 통계 확인하기
- **pg_stat_user_tables**로 테이블 사용 분석하기
- **pg_stat_statements** 확장으로 쿼리 추적하기
- **실시간 모니터링 뷰** 만들기

---

## 1. pg_stat_activity (현재 활동)

```sql
-- 현재 실행 중인 모든 쿼리
SELECT
  pid,
  usename AS username,
  application_name,
  client_addr,
  state,
  query_start,
  now() - query_start AS duration,
  query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- 오래 실행되는 쿼리 찾기 (5분 이상)
SELECT
  pid,
  now() - query_start AS duration,
  query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 minutes';

-- 해당 쿼리 종료
SELECT pg_terminate_backend(pid);
```

---

## 2. pg_stat_database (DB 통계)

```sql
-- 데이터베이스별 통계
SELECT
  datname,
  numbackends AS connections,
  xact_commit AS commits,
  xact_rollback AS rollbacks,
  blks_read AS disk_reads,
  blks_hit AS cache_hits,
  ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio,
  tup_returned AS rows_returned,
  tup_fetched AS rows_fetched,
  tup_inserted AS rows_inserted,
  tup_updated AS rows_updated,
  tup_deleted AS rows_deleted
FROM pg_stat_database
WHERE datname = current_database();

-- 캐시 히트율 모니터링
SELECT
  datname,
  ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio
FROM pg_stat_database
ORDER BY cache_hit_ratio;
-- 목표: 95% 이상
```

---

## 3. pg_stat_user_tables (테이블 통계)

```sql
-- 테이블별 사용 통계
SELECT
  schemaname,
  relname AS table_name,
  seq_scan AS sequential_scans,
  seq_tup_read AS rows_seq_read,
  idx_scan AS index_scans,
  idx_tup_fetch AS rows_idx_fetched,
  n_tup_ins AS rows_inserted,
  n_tup_upd AS rows_updated,
  n_tup_del AS rows_deleted,
  n_live_tup AS live_rows,
  n_dead_tup AS dead_rows,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- 인덱스가 필요한 테이블 찾기
SELECT
  schemaname,
  relname,
  seq_scan,
  seq_tup_read,
  idx_scan,
  seq_tup_read / NULLIF(seq_scan, 0) AS avg_rows_per_seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND idx_scan < seq_scan
ORDER BY seq_tup_read DESC;
```

---

## 4. pg_stat_statements (쿼리 통계)

```sql
-- 확장 설치
CREATE EXTENSION pg_stat_statements;

-- 가장 많이 실행된 쿼리
SELECT
  calls,
  total_exec_time,
  mean_exec_time,
  query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- 가장 느린 쿼리
SELECT
  calls,
  total_exec_time,
  mean_exec_time,
  query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- I/O 많이 사용하는 쿼리
SELECT
  calls,
  shared_blks_read + shared_blks_written AS total_io,
  query
FROM pg_stat_statements
ORDER BY total_io DESC
LIMIT 10;
```

---

## 5. 실시간 모니터링 뷰

```sql
-- 종합 모니터링 뷰
CREATE VIEW v_monitor_summary AS
SELECT
  (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_queries,
  (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction') AS idle_in_transaction,
  (SELECT max(now() - query_start) FROM pg_stat_activity WHERE state = 'active') AS longest_query,
  (SELECT ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)
   FROM pg_stat_database WHERE datname = current_database()) AS cache_hit_ratio,
  pg_database_size(current_database()) AS db_size;

-- 사용
SELECT * FROM v_monitor_summary;
```

---

## 📖 정리

| 뷰 | 용도 |
|------|----------|
| **pg_stat_activity** | 현재 세션, 실행 중인 쿼리 |
| **pg_stat_database** | DB 통계, 캐시 히트율 |
| **pg_stat_user_tables** | 테이블 사용 통계 |
| **pg_stat_statements** | 쿼리별 성능 통계 |

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐ (중급)
**학습 시간**: 60분

이전: [Chapter 37. 시스템 카탈로그](37-시스템-카탈로그.md)
다음: [Chapter 39. Slow Query 분석](39-slow-query-분석.md)
