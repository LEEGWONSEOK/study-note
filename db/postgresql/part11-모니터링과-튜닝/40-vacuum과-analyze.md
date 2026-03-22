# Chapter 40. VACUUM과 ANALYZE

## 🎯 이 챕터의 목표
- **MVCC**와 Dead Tuple 이해하기
- **VACUUM**의 원리와 종류 알기
- **ANALYZE** 통계 수집하기
- **autovacuum** 설정 및 튜닝하기
- **Bloat** 모니터링 및 관리하기

---

## 1. MVCC와 Dead Tuple

### 📦 버전 관리에 비유하면

```
MVCC = 파일의 여러 버전 유지
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

원본: users (id=1, name='Alice')
UPDATE: name='Bob'

결과:
  - 구버전(Alice): Dead Tuple (삭제 대기)
  - 신버전(Bob): Live Tuple (활성)

VACUUM 역할:
  - Dead Tuple 정리
  - 공간 재사용 가능하게 표시
```

---

## 2. VACUUM 종류

### 🧹 기본 VACUUM

```sql
-- 테이블 VACUUM
VACUUM users;

-- 전체 데이터베이스
VACUUM;

-- VERBOSE 옵션 (상세 정보)
VACUUM VERBOSE users;
-- INFO:  vacuuming "public.users"
-- INFO:  "users": removed 1000 dead row versions in 10 pages
-- INFO:  "users": found 1000 removable, 5000 nonremovable row versions
```

### 💪 VACUUM FULL (전체 정리)

```sql
-- FULL VACUUM: 테이블 완전 재작성
VACUUM FULL users;

-- 주의:
-- 1. 테이블 잠금 (쓰기 차단)
-- 2. 시간 오래 걸림
-- 3. 임시 디스크 공간 필요
-- 4. 프로덕션에서 신중히 사용
```

### 🔄 VACUUM ANALYZE (통계 포함)

```sql
-- VACUUM + ANALYZE 동시 수행
VACUUM ANALYZE users;
```

---

## 3. ANALYZE (통계 수집)

```sql
-- 테이블 통계 수집
ANALYZE users;

-- 특정 컬럼만
ANALYZE users (email, created_at);

-- 전체 데이터베이스
ANALYZE;

-- 통계 확인
SELECT
  tablename,
  attname,
  n_distinct,
  most_common_vals
FROM pg_stats
WHERE tablename = 'users';
```

---

## 4. autovacuum 설정

```ini
# postgresql.conf

# autovacuum 활성화 (기본값: on)
autovacuum = on

# autovacuum 실행 간격
autovacuum_naptime = 1min

# 동시 autovacuum 워커 수
autovacuum_max_workers = 3

# VACUUM 트리거 조건
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
# 공식: threshold + (scale_factor * 테이블 행 수)
# 예: 50 + (0.2 * 1000) = 250개 dead tuple 발생 시 실행

# ANALYZE 트리거 조건
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1
```

### 테이블별 설정

```sql
-- 자주 변경되는 테이블: 더 자주 VACUUM
ALTER TABLE active_table SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.02
);

-- 거의 변경 안 되는 테이블: 덜 자주 VACUUM
ALTER TABLE static_table SET (
  autovacuum_vacuum_scale_factor = 0.5
);
```

---

## 5. Bloat 모니터링

```sql
-- 테이블 bloat 확인
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
  n_dead_tup AS dead_tuples,
  n_live_tup AS live_tuples,
  ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Bloat 해결
-- 1. VACUUM: 공간 재사용 표시
VACUUM users;

-- 2. VACUUM FULL: 완전 정리 (다운타임 필요)
VACUUM FULL users;

-- 3. pg_repack: 온라인 재구성 (확장 필요)
-- pg_repack -t users
```

---

## 6. 모니터링 쿼리

```sql
-- autovacuum 실행 이력
SELECT
  schemaname,
  relname,
  last_vacuum,
  last_autovacuum,
  vacuum_count,
  autovacuum_count
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST;

-- 현재 실행 중인 VACUUM
SELECT
  pid,
  query_start,
  now() - query_start AS duration,
  query
FROM pg_stat_activity
WHERE query LIKE '%VACUUM%'
  AND query NOT LIKE '%pg_stat_activity%';
```

---

## 📖 정리

| 명령 | 용도 | 잠금 |
|------|----------|------|
| **VACUUM** | Dead tuple 정리 | 읽기/쓰기 가능 |
| **VACUUM FULL** | 완전 재작성 | 쓰기 차단 |
| **ANALYZE** | 통계 수집 | 읽기/쓰기 가능 |
| **autovacuum** | 자동 VACUUM | 백그라운드 |

### 권장 설정

```
대용량 트랜잭션:
  autovacuum_vacuum_scale_factor = 0.05
  autovacuum_max_workers = 6

일반 시스템:
  autovacuum_vacuum_scale_factor = 0.2 (기본값)
  autovacuum_max_workers = 3
```

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐ (고급)
**학습 시간**: 90분

이전: [Chapter 39. Slow Query 분석](39-slow-query-분석.md)
다음: [Chapter 41. Node.js 연동](../part12-실전/41-nodejs-연동.md)