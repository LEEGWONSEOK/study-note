# Chapter 35. Streaming Replication

## 🎯 이 챕터의 목표
- **Primary 서버** 완벽 설정하기
- **Standby 서버** 구축하기 (pg_basebackup)
- **Hot Standby** 활성화하여 읽기 쿼리 처리하기
- **Replication Slot** 생성 및 관리하기
- **동기/비동기 복제** 전환하기
- **복제 지연** 모니터링 및 해결하기

---

## 1. 환경 준비

### 🖥️ 실습 환경

```
구성:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Primary 서버:
  호스트: 10.0.1.10 (primary-db)
  포트: 5432
  데이터 디렉토리: /var/lib/postgresql/14/primary

Standby 서버:
  호스트: 10.0.1.20 (standby-db)
  포트: 5432
  데이터 디렉토리: /var/lib/postgresql/14/standby

네트워크: 양방향 통신 가능
```

---

## 2. Primary 서버 설정

### 📝 1단계: postgresql.conf 설정

```bash
# Primary 서버에 로그인
ssh primary-db

# 설정 파일 수정
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```ini
# /etc/postgresql/14/main/postgresql.conf

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# WAL 설정 (Replication 필수)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# WAL 레벨 (replica로 설정)
wal_level = replica
# minimal: 최소 로깅 (복제 불가)
# replica: 물리적 복제 가능 (권장)
# logical: 논리적 복제 가능

# WAL 보관 설정
wal_keep_size = 1GB
# Standby가 일시적으로 연결 끊겨도 WAL 보관
# 또는 Replication Slot 사용 권장

max_wal_senders = 10
# 동시 연결 가능한 Standby 수 (최소 Standby 수 + 예비)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Replication 설정
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Replication Slot 사용 (권장)
max_replication_slots = 10

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 동기 복제 설정 (선택사항)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# 비동기 복제 (기본값, 빠름)
# synchronous_standby_names = ''

# 동기 복제 (데이터 손실 없음, 느림)
# synchronous_standby_names = 'standby1'

# 여러 Standby 중 1개만 동기
# synchronous_standby_names = 'FIRST 1 (standby1, standby2)'

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 기타 설정
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# 외부 연결 허용
listen_addresses = '*'

# 로깅
log_line_prefix = '%t [%p] %u@%d '
log_replication_commands = on
```

### 🔐 2단계: pg_hba.conf 설정

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

```ini
# /etc/postgresql/14/main/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# 로컬 연결
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 로컬
host    all             all             127.0.0.1/32            scram-sha-256

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Replication 연결 (중요!)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Standby 서버에서 복제 연결 허용
host    replication     replication     10.0.1.20/32            scram-sha-256

# 또는 전체 서브넷 허용 (보안 주의)
# host    replication     replication     10.0.1.0/24             scram-sha-256

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 일반 데이터베이스 연결
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

host    all             all             10.0.1.0/24             scram-sha-256
```

### 👤 3단계: Replication 사용자 생성

```bash
# PostgreSQL 접속
sudo -u postgres psql
```

```sql
-- Replication 전용 사용자 생성
CREATE ROLE replication WITH
  LOGIN
  REPLICATION
  PASSWORD 'replication_password';

-- 권한 확인
\du replication
--              List of roles
--  Role name   |  Attributes  | Member of
-- -------------+--------------+-----------
--  replication | Replication  | {}

-- Replication 슬롯 생성 (Standby용)
SELECT pg_create_physical_replication_slot('standby1_slot');

-- 슬롯 확인
SELECT slot_name, slot_type, active FROM pg_replication_slots;
--   slot_name    | slot_type | active
-- ---------------+-----------+--------
--  standby1_slot | physical  | f
-- (아직 Standby 연결 전이라 active = false)
```

### 🔄 4단계: PostgreSQL 재시작

```bash
# 설정 적용을 위해 재시작
sudo systemctl restart postgresql

# 상태 확인
sudo systemctl status postgresql

# 로그 확인
sudo tail -f /var/log/postgresql/postgresql-14-main.log
```

---

## 3. Standby 서버 구축

### 📥 1단계: pg_basebackup으로 초기 동기화

```bash
# Standby 서버에 로그인
ssh standby-db

# PostgreSQL 중지 (처음 설치 시)
sudo systemctl stop postgresql

# 기존 데이터 디렉토리 백업 (있다면)
sudo mv /var/lib/postgresql/14/main /var/lib/postgresql/14/main.old

# Primary에서 베이스 백업 받기
sudo -u postgres pg_basebackup \
  -h 10.0.1.10 \
  -p 5432 \
  -U replication \
  -D /var/lib/postgresql/14/main \
  -Fp \
  -Xs \
  -P \
  -R \
  -S standby1_slot

# 옵션 설명:
# -h: Primary 호스트
# -p: Primary 포트
# -U: Replication 사용자
# -D: Standby 데이터 디렉토리
# -Fp: Plain 포맷 (디렉토리 구조 그대로)
# -Xs: WAL을 streaming으로 받음
# -P: 진행 상황 표시
# -R: standby.signal 및 primary_conninfo 자동 생성 (PostgreSQL 12+)
# -S: Replication Slot 이름

# 실행 중 비밀번호 입력:
# Password: replication_password

# 출력 예시:
# 24563/24563 kB (100%), 1/1 tablespace
```

### 🔧 2단계: Standby 설정 확인

```bash
# standby.signal 파일 확인 (PostgreSQL 12+)
ls -l /var/lib/postgresql/14/main/standby.signal
# -rw------- 1 postgres postgres 0 Jan 15 10:00 standby.signal

# postgresql.auto.conf 확인 (-R 옵션으로 자동 생성됨)
sudo -u postgres cat /var/lib/postgresql/14/main/postgresql.auto.conf
```

```ini
# postgresql.auto.conf (자동 생성됨)
primary_conninfo = 'user=replication password=replication_password channel_binding=prefer host=10.0.1.10 port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'

primary_slot_name = 'standby1_slot'
```

### ⚙️ 3단계: Standby 추가 설정 (선택사항)

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```ini
# postgresql.conf (Standby 추가 설정)

# Hot Standby 활성화 (읽기 쿼리 허용)
hot_standby = on

# Standby에서 피드백 전송 (복제 지연 모니터링)
hot_standby_feedback = on

# 읽기 쿼리 대기 시간 (Standby에서)
max_standby_streaming_delay = 30s
# WAL 재생이 읽기 쿼리와 충돌 시 대기 시간
```

### 🚀 4단계: Standby 시작

```bash
# PostgreSQL 시작
sudo systemctl start postgresql

# 상태 확인
sudo systemctl status postgresql

# 로그 확인 (복제 시작 확인)
sudo tail -f /var/log/postgresql/postgresql-14-main.log
# 2024-01-15 10:05:00 KST [1234]: LOG:  entering standby mode
# 2024-01-15 10:05:01 KST [1234]: LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
# 2024-01-15 10:05:02 KST [1234]: LOG:  consistent recovery state reached at 0/3000100
# 2024-01-15 10:05:02 KST [1234]: LOG:  database system is ready to accept read-only connections
```

---

## 4. 복제 확인 및 테스트

### ✅ 1단계: Primary에서 확인

```bash
ssh primary-db
sudo -u postgres psql
```

```sql
-- Standby 연결 상태 확인
SELECT
  client_addr,
  application_name,
  state,
  sync_state,
  replay_lsn
FROM pg_stat_replication;

-- 결과:
--  client_addr | application_name | state     | sync_state | replay_lsn
-- -------------+------------------+-----------+------------+-------------
--  10.0.1.20   | 14/main          | streaming | async      | 0/3000500

-- state = streaming: 정상 복제 중 ✅
-- sync_state = async: 비동기 복제

-- Replication Slot 활성화 확인
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
--   slot_name    | active | restart_lsn
-- ---------------+--------+-------------
--  standby1_slot | t      | 0/3000000
-- active = true ✅
```

### ✅ 2단계: Standby에서 확인

```bash
ssh standby-db
sudo -u postgres psql
```

```sql
-- Standby 모드 확인
SELECT pg_is_in_recovery();
--  pg_is_in_recovery
-- -------------------
--  t
-- true = Standby 모드 ✅

-- 복제 지연 확인
SELECT
  pg_last_wal_receive_lsn() AS receive_lsn,
  pg_last_wal_replay_lsn() AS replay_lsn,
  pg_wal_lsn_diff(
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn()
  ) AS lag_bytes;

--  receive_lsn | replay_lsn | lag_bytes
-- -------------+------------+-----------
--  0/3000500   | 0/3000500  |         0
-- lag_bytes = 0: 지연 없음 ✅

-- 마지막 WAL 재생 시간
SELECT pg_last_xact_replay_timestamp();
--  pg_last_xact_replay_timestamp
-- --------------------------------
--  2024-01-15 10:10:00+09
```

### ✅ 3단계: 데이터 복제 테스트

```sql
-- Primary에서 테스트 데이터 삽입
-- (Primary 서버에서)
CREATE TABLE replication_test (
  id SERIAL PRIMARY KEY,
  data TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO replication_test (data) VALUES ('Test from Primary');

SELECT * FROM replication_test;
--  id |       data        |         created_at
-- ----+-------------------+----------------------------
--   1 | Test from Primary | 2024-01-15 10:15:00+09

-- Standby에서 확인 (1~2초 대기)
-- (Standby 서버에서)
SELECT * FROM replication_test;
--  id |       data        |         created_at
-- ----+-------------------+----------------------------
--   1 | Test from Primary | 2024-01-15 10:15:00+09
-- ✅ 복제 성공!

-- Standby에서 쓰기 시도
INSERT INTO replication_test (data) VALUES ('Test from Standby');
-- ERROR:  cannot execute INSERT in a read-only transaction
-- ✅ 읽기 전용 확인
```

---

## 5. Hot Standby (읽기 쿼리 처리)

### 📖 Hot Standby란?

```
Hot Standby = Standby에서 읽기 쿼리 실행 가능
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Warm Standby (예전):
  Standby에서 쿼리 불가
  오직 복제만 수행

Hot Standby (현재):
  Standby에서 읽기 쿼리 가능! ✅
  복제와 동시에 SELECT 처리
```

### 🚀 활용 예시

```javascript
// Node.js 애플리케이션
const { Pool } = require('pg');

// Primary 연결 (쓰기)
const primaryPool = new Pool({
  host: '10.0.1.10',
  database: 'mydb',
  user: 'app_user',
  password: 'app_password',
  max: 20
});

// Standby 연결 (읽기)
const standbyPool = new Pool({
  host: '10.0.1.20',
  database: 'mydb',
  user: 'app_user',
  password: 'app_password',
  max: 50  // 읽기가 많으므로 풀 크기 증가
});

// 쓰기 작업
async function createUser(name, email) {
  return await primaryPool.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id',
    [name, email]
  );
}

// 읽기 작업 (Standby 사용)
async function getUsers() {
  return await standbyPool.query('SELECT * FROM users');
}

// 사용
const newUser = await createUser('Alice', 'alice@example.com');
console.log('Created user:', newUser.rows[0].id);

// 읽기는 Standby에서
const users = await getUsers();
console.log('Total users:', users.rows.length);
```

### ⚠️ Hot Standby 제약사항

```sql
-- ✅ 허용되는 작업
SELECT * FROM users;
SELECT COUNT(*) FROM orders;
EXPLAIN SELECT * FROM products;

-- ❌ 허용되지 않는 작업
INSERT INTO users VALUES (...);
-- ERROR:  cannot execute INSERT in a read-only transaction

UPDATE users SET name = 'Bob' WHERE id = 1;
-- ERROR:  cannot execute UPDATE in a read-only transaction

DELETE FROM users WHERE id = 1;
-- ERROR:  cannot execute DELETE in a read-only transaction

CREATE TABLE test (...);
-- ERROR:  cannot execute CREATE TABLE in a read-only transaction

-- ❌ 임시 테이블도 불가
CREATE TEMP TABLE temp_data AS SELECT * FROM users;
-- ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

---

## 6. 동기 복제 설정

### 🔄 비동기 → 동기 전환

```bash
# Primary 서버에서
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```ini
# postgresql.conf

# 동기 복제 활성화
synchronous_commit = on
synchronous_standby_names = '14/main'  # Standby application_name

# 또는 특정 이름 지정 (pg_basebackup -R로 자동 설정된 이름 확인)
# SELECT application_name FROM pg_stat_replication;
```

```bash
# 설정 리로드 (재시작 불필요)
sudo systemctl reload postgresql
```

### ✅ 동기 복제 확인

```sql
-- Primary에서
SELECT
  application_name,
  state,
  sync_state,
  sync_priority
FROM pg_stat_replication;

--  application_name | state     | sync_state | sync_priority
-- ------------------+-----------+------------+---------------
--  14/main          | streaming | sync       |             1
-- sync_state = sync ✅

-- 성능 테스트
\timing on

INSERT INTO replication_test (data) VALUES ('Sync test');
-- Time: 25.123 ms (동기 복제로 인해 약간 느림)

-- 비동기였을 때: ~5ms
-- 동기일 때: ~25ms (네트워크 왕복 시간 추가)
```

### 🎛️ 고급 동기 설정

```ini
# 여러 Standby 중 1개만 동기
synchronous_standby_names = 'FIRST 1 (standby1, standby2, standby3)'
# 3개 중 1개만 확인하면 응답 (나머지는 비동기)

# 여러 Standby 모두 동기
synchronous_standby_names = 'standby1, standby2'
# 둘 다 확인해야 응답 (더 안전, 더 느림)

# N개 중 M개 동기
synchronous_standby_names = 'FIRST 2 (standby1, standby2, standby3)'
# 3개 중 2개만 확인하면 응답
```

---

## 7. 복제 지연 모니터링

### 📊 복제 지연 확인 쿼리

```sql
-- Primary에서 실행
SELECT
  application_name,
  client_addr,
  state,
  sync_state,
  -- WAL 위치 차이 (바이트)
  pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
  pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag_bytes,
  pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag_bytes,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
  -- 시간 지연 (PostgreSQL 10+)
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;

-- 결과 예시:
--  application_name | state     | replay_lag_bytes | replay_lag
-- ------------------+-----------+------------------+-------------
--  14/main          | streaming |            16384 | 00:00:02.5
-- (16KB 뒤처짐, 2.5초 지연)
```

### 🚨 지연 알림 쿼리

```sql
-- 위험 수준 판단
SELECT
  application_name,
  CASE
    WHEN replay_lag > interval '1 minute' THEN '🔴 CRITICAL'
    WHEN replay_lag > interval '10 seconds' THEN '🟡 WARNING'
    WHEN replay_lag IS NULL THEN '⚪ UNKNOWN'
    ELSE '🟢 OK'
  END AS status,
  replay_lag,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### 📈 모니터링 뷰 생성

```sql
-- 복제 상태 종합 뷰
CREATE OR REPLACE VIEW v_replication_monitor AS
SELECT
  application_name AS standby_name,
  client_addr AS standby_ip,
  state,
  sync_state,
  -- 지연 (바이트)
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag_size,
  -- 지연 (시간)
  COALESCE(replay_lag, interval '0') AS lag_time,
  -- 상태
  CASE
    WHEN state != 'streaming' THEN '🔴 NOT STREAMING'
    WHEN replay_lag > interval '1 minute' THEN '🔴 CRITICAL LAG'
    WHEN replay_lag > interval '10 seconds' THEN '🟡 WARNING LAG'
    ELSE '🟢 OK'
  END AS health_status,
  -- 마지막 메시지 시간
  backend_start,
  reply_time
FROM pg_stat_replication;

-- 사용
SELECT * FROM v_replication_monitor;
```

---

## 8. 복제 지연 해결

### 🐌 지연 발생 원인

```
복제 지연의 주요 원인
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1️⃣ Primary 부하 과다
   - 대량 INSERT/UPDATE
   - 긴 트랜잭션
   → WAL 생성 속도 > Standby 재생 속도

2️⃣ 네트워크 대역폭 부족
   - WAL 전송 지연
   → Primary와 Standby 간 느린 네트워크

3️⃣ Standby 하드웨어 부족
   - CPU, 디스크 I/O 부족
   → WAL 재생 속도 느림

4️⃣ Hot Standby 쿼리 충돌
   - 긴 읽기 쿼리
   → WAL 재생 대기

5️⃣ Standby 설정 문제
   - max_standby_streaming_delay 너무 큼
   - hot_standby_feedback = off
```

### 🛠️ 해결 방법

#### 1️⃣ hot_standby_feedback 활성화

```ini
# Standby: postgresql.conf
hot_standby_feedback = on
# Standby의 읽기 쿼리 정보를 Primary에 전달
# Primary가 VACUUM을 지연시켜 충돌 방지
```

```bash
sudo systemctl reload postgresql
```

#### 2️⃣ max_standby_streaming_delay 조정

```ini
# Standby: postgresql.conf
max_standby_streaming_delay = 30s
# WAL 재생이 읽기 쿼리와 충돌 시 대기 시간
# 0: 즉시 쿼리 취소 (복제 우선)
# -1: 무한 대기 (쿼리 우선, 지연 증가 위험)
```

#### 3️⃣ Standby 하드웨어 업그레이드

```
권장 사항:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CPU: Primary와 동일 이상
메모리: Primary와 동일 이상
디스크: SSD 권장 (WAL 재생 속도 중요)
네트워크: Primary와 저지연 네트워크 (1Gbps 이상)
```

#### 4️⃣ 읽기 쿼리 최적화

```sql
-- Standby에서 긴 쿼리 확인
SELECT
  pid,
  usename,
  application_name,
  state,
  query_start,
  now() - query_start AS duration,
  query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid != pg_backend_pid()
ORDER BY duration DESC;

-- 긴 쿼리 종료 (필요 시)
SELECT pg_terminate_backend(pid);
```

#### 5️⃣ 대량 작업 시 주의

```sql
-- ❌ 나쁜 예: 한 번에 대량 UPDATE
UPDATE users SET status = 'active';
-- 수백만 행 → 엄청난 WAL 생성 → Standby 지연 증가

-- ✅ 좋은 예: 배치로 나눠서 UPDATE
DO $$
DECLARE
  batch_size INT := 10000;
  total_updated INT := 0;
BEGIN
  LOOP
    UPDATE users
    SET status = 'active'
    WHERE id IN (
      SELECT id FROM users
      WHERE status != 'active'
      LIMIT batch_size
    );

    GET DIAGNOSTICS total_updated = ROW_COUNT;
    EXIT WHEN total_updated = 0;

    -- 각 배치 후 잠시 대기 (Standby가 따라잡을 시간)
    PERFORM pg_sleep(0.1);

    RAISE NOTICE 'Updated % rows', total_updated;
  END LOOP;
END $$;
```

---

## 9. Cascading Replication (계단식 복제)

### 🪜 Cascading Replication 구조

```
Primary → Standby1 (중간 노드) → Standby2
                                → Standby3
```

### 🔧 설정 방법

```bash
# Standby1 설정 (Primary로부터 복제 + 하위 Standby에 전달)
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```ini
# Standby1: postgresql.conf
hot_standby = on
max_wal_senders = 10  # 하위 Standby를 위해 설정
```

```bash
# Standby2, 3 구축 (Standby1에서 pg_basebackup)
sudo -u postgres pg_basebackup \
  -h standby1-ip \
  -U replication \
  -D /var/lib/postgresql/14/main \
  -Fp -Xs -P -R
```

### ✅ 확인

```sql
-- Standby1에서 (하위 Standby 연결 확인)
SELECT client_addr, state FROM pg_stat_replication;
--  client_addr | state
-- -------------+-----------
--  10.0.1.21   | streaming  (Standby2)
--  10.0.1.22   | streaming  (Standby3)
```

---

## 10. 실습 과제

### 🎯 과제 1: 로컬 Replication 구축
로컬 환경에서 Primary + Standby 구축하기:

```
1. Primary 설정 (포트 5432)
   - wal_level = replica
   - Replication 사용자 생성
   - Replication Slot 생성

2. Standby 구축 (포트 5433)
   - pg_basebackup으로 초기 동기화
   - Hot Standby 활성화

3. 테스트
   - Primary에 데이터 삽입
   - Standby에서 확인
   - 복제 지연 모니터링
```

### 🎯 과제 2: 동기 복제 성능 비교
비동기 vs 동기 복제 성능 측정:

```sql
-- 1. 비동기 모드로 10,000건 INSERT
-- 2. 동기 모드로 10,000건 INSERT
-- 3. 실행 시간 비교

-- 측정 쿼리:
\timing on

DO $$
BEGIN
  FOR i IN 1..10000 LOOP
    INSERT INTO test_table (data) VALUES ('test');
  END LOOP;
END $$;

-- 결과 기록:
-- 비동기: ___초
-- 동기: ___초
-- 차이: ___배
```

### 🎯 과제 3: 복제 지연 시뮬레이션
의도적으로 지연을 발생시키고 해결하기:

```
1. Standby에서 긴 쿼리 실행
   SELECT pg_sleep(60);

2. Primary에서 대량 INSERT

3. 복제 지연 확인
   SELECT replay_lag FROM pg_stat_replication;

4. 해결
   - 긴 쿼리 종료
   - hot_standby_feedback 활성화
   - 지연 해소 확인
```

---

## 11. 다음 단계

Streaming Replication 구축을 마스터했으니, 이제 Failover와 고가용성을 배울 차례입니다!

### 다음 챕터 미리보기
```
Chapter 36: Failover와 고가용성
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 배울 내용:
   ├─ Failover의 개념과 종류
   ├─ 수동 Failover 절차
   ├─ 자동 Failover (pg_auto_failover, Patroni)
   ├─ 로드밸런서 설정 (HAProxy, PgBouncer)
   ├─ VIP (Virtual IP) 관리
   └─ 고가용성 아키텍처 설계

💡 실습:
   ├─ 수동 Failover 시뮬레이션
   ├─ Patroni를 이용한 자동 Failover
   ├─ HAProxy 로드밸런싱
   ├─ Failover 테스트 자동화
   └─ 재해 복구 훈련
```

---

## 📖 정리

이번 챕터에서 배운 내용:

| 개념 | 핵심 내용 |
|------|----------|
| **Primary 설정** | wal_level=replica, max_wal_senders, Replication 사용자 |
| **pg_basebackup** | 초기 동기화, -R 옵션으로 자동 설정 |
| **Hot Standby** | Standby에서 읽기 쿼리 가능, 부하 분산 |
| **Replication Slot** | WAL 보관 보장, 네트워크 단절 대비 |
| **동기 복제** | synchronous_standby_names, 데이터 손실 없음 |
| **복제 지연** | pg_stat_replication 모니터링, 원인별 해결 |

### 핵심 명령어

```bash
# Primary 설정
wal_level = replica
max_wal_senders = 10

# Replication 사용자
CREATE ROLE replication WITH LOGIN REPLICATION PASSWORD 'pass';

# Replication Slot
SELECT pg_create_physical_replication_slot('slot_name');

# Standby 구축
pg_basebackup -h primary -U replication -D /data -Fp -Xs -P -R

# 복제 상태 확인
SELECT * FROM pg_stat_replication;
```

**기억하세요**: "Streaming Replication은 고가용성의 기반입니다. 하지만 구축만으로는 부족합니다. 지속적인 모니터링과 Failover 계획이 있어야 진정한 고가용성을 달성할 수 있습니다!"

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐ (고급)
**학습 시간**: 120분
**대상**: 백엔드 개발자, 데브옵스 엔지니어, DBA
**중요도**: 🔥🔥🔥🔥🔥 (매우 높음)

이전: [Chapter 34. Replication 개요](34-replication-개요.md)
다음: [Chapter 36. Failover와 고가용성](36-failover와-고가용성.md)