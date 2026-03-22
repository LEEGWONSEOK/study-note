# Chapter 34. Replication 개요

## 🎯 이 챕터의 목표
- **Replication**의 개념과 필요성 이해하기
- **복제 방식** 비교하기 (Physical vs Logical)
- **Primary-Standby** 아키텍처 이해하기
- **Synchronous vs Asynchronous** 복제 차이 알기
- **고가용성 시스템** 설계 원칙 익히기
- **복제 시나리오** 선택하기

---

## 1. Replication이란? (비유로 이해하기)

### 📚 도서관 분관 시스템에 비유하면

```
Replication = 본관의 책을 분관에 복사
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🏛️ 본관 (Primary)
   ├─ 원본 데이터 보유
   ├─ 모든 변경 사항 기록
   └─ 쓰기 작업 처리

📖 분관 1, 2, 3 (Standby/Replica)
   ├─ 본관의 복사본
   ├─ 계속 동기화 (새 책 추가, 수정 반영)
   └─ 읽기 작업 분산 처리

장점:
✅ 본관이 고장나도 분관에서 서비스 가능 (고가용성)
✅ 독자들이 가까운 분관 이용 (읽기 성능 향상)
✅ 본관 부담 감소 (부하 분산)
```

---

## 2. 왜 Replication이 필요한가?

### 🎯 3가지 핵심 이유

#### 1️⃣ 고가용성 (High Availability)

```
단일 서버 vs Replication
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

❌ 단일 서버:
   [Primary DB] 💥
        ↓
   서비스 중단! (다운타임)

✅ Replication:
   [Primary DB] 💥
        ↓
   [Standby DB] → 자동으로 Primary로 승격
        ↓
   서비스 계속 운영! (다운타임 최소화)

목표 가용성:
- 99.9% (Three Nines): 연간 8.76시간 다운타임
- 99.99% (Four Nines): 연간 52.56분 다운타임
- 99.999% (Five Nines): 연간 5.26분 다운타임
```

#### 2️⃣ 읽기 성능 향상

```
읽기 부하 분산
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

❌ 단일 서버:
   모든 읽기/쓰기 → [Primary DB]
                      ↓
                   과부하! 🔥

✅ Replication:
   쓰기 → [Primary DB]
          │
          ├─ 복제 → [Standby 1] ← 읽기 요청
          ├─ 복제 → [Standby 2] ← 읽기 요청
          └─ 복제 → [Standby 3] ← 읽기 요청

읽기 요청 3배 분산! 성능 향상! 🚀
```

#### 3️⃣ 재해 복구 (Disaster Recovery)

```
지리적 분산
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

서울 IDC                         부산 IDC
[Primary DB] ────────복제────────> [Standby DB]
     💥 화재/지진 발생!

부산 Standby가 Primary로 승격
→ 서비스 계속 운영

RPO (Recovery Point Objective): 몇 초
RTO (Recovery Time Objective): 몇 분
```

---

## 3. Physical Replication vs Logical Replication

### 🔧 Physical Replication (물리적 복제)

```
Physical Replication = 디스크 블록을 그대로 복사
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Primary:
  [데이터 파일] [WAL]
       ↓         ↓
  바이트 단위로 복사
       ↓         ↓
  [데이터 파일] [WAL]
Standby:

특징:
✅ 전체 클러스터 복제 (모든 DB, 테이블, 인덱스)
✅ 매우 빠름 (바이트 복사)
✅ 설정 간단
✅ WAL 기반 (Chapter 30 PITR에서 배운 기술)

❌ 동일한 PostgreSQL 버전 필요
❌ 선택적 복제 불가 (전체 or 없음)
❌ 플랫폼 종속 (같은 OS, 아키텍처)

사용 사례:
- 고가용성 (HA)
- 읽기 부하 분산
- 재해 복구 (DR)
```

### 🧩 Logical Replication (논리적 복제)

```
Logical Replication = SQL 문 기반 복제
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Primary:
  INSERT INTO users VALUES (1, 'Alice');
       ↓
  SQL 문 전송
       ↓
  INSERT INTO users VALUES (1, 'Alice');
Standby:

특징:
✅ 선택적 복제 (특정 테이블만)
✅ 다른 PostgreSQL 버전 가능
✅ 양방향 복제 가능
✅ 필터링 가능 (특정 행만)

❌ Physical보다 느림
❌ DDL 자동 복제 안 됨
❌ 시퀀스 자동 복제 안 됨
❌ 설정 복잡

사용 사례:
- 데이터 통합 (여러 DB → 1개)
- 마이그레이션 (버전 업그레이드)
- 일부 테이블만 복제
- 멀티 마스터 (양방향 복제)
```

### 📊 비교표

| 항목 | Physical Replication | Logical Replication |
|------|----------------------|---------------------|
| **복제 단위** | 바이트 (블록) | SQL 문 (행) |
| **복제 범위** | 전체 클러스터 | 특정 테이블 선택 가능 |
| **속도** | 매우 빠름 | 느림 |
| **버전** | 동일 버전 필수 | 다른 버전 가능 |
| **플랫폼** | 동일 플랫폼 필요 | 무관 |
| **DDL 복제** | 자동 | 수동 |
| **설정** | 간단 | 복잡 |
| **사용 사례** | HA, DR, 읽기 분산 | 데이터 통합, 마이그레이션 |

---

## 4. Synchronous vs Asynchronous Replication

### ⚡ Asynchronous Replication (비동기 복제)

```
비동기 복제 = 확인 안 기다리고 바로 응답
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

클라이언트 → Primary: INSERT
              ↓
         WAL 기록 ✅
              ↓
         클라이언트 ← "완료!" (즉시 응답)
              ↓
    (백그라운드) Standby로 복제 📤

타임라인:
0ms ──┬── INSERT 받음
      │
5ms ──┼── WAL 기록
      │
6ms ──┼── 클라이언트 응답 ✅
      │
50ms ─┴── Standby 복제 완료

장점:
✅ 매우 빠름 (지연 거의 없음)
✅ Standby 장애 시에도 Primary 정상 작동
✅ 네트워크 지연에 영향 안 받음

단점:
❌ Primary 장애 시 일부 데이터 손실 가능 (복제 전 손실)
❌ 복제 지연 (Replication Lag) 발생 가능

사용 사례:
- 읽기 분산 (약간의 지연 허용)
- 재해 복구 (몇 초 손실 허용)
- 네트워크가 불안정한 환경
```

### 🔒 Synchronous Replication (동기 복제)

```
동기 복제 = Standby 확인 후 응답
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

클라이언트 → Primary: INSERT
              ↓
         WAL 기록
              ↓
         Standby로 전송 📤
              ↓
         Standby WAL 기록 대기... ⏳
              ↓
         Standby ✅ "받았어요!"
              ↓
         클라이언트 ← "완료!" (확인 후 응답)

타임라인:
0ms ──┬── INSERT 받음
      │
5ms ──┼── WAL 기록
      │
6ms ──┼── Standby로 전송
      │
50ms ─┼── Standby 확인 대기...
      │
55ms ─┴── 클라이언트 응답 ✅

장점:
✅ 데이터 손실 없음 (Zero Data Loss)
✅ Primary와 Standby 항상 동기화

단점:
❌ 느림 (네트워크 왕복 시간 추가)
❌ Standby 장애 시 Primary도 대기 (또는 설정에 따라)
❌ 네트워크 지연에 민감

사용 사례:
- 금융 시스템 (데이터 손실 절대 불가)
- 중요한 트랜잭션
- 컴플라이언스 요구사항
```

### 🎛️ 설정 비교

```ini
# postgresql.conf

# 비동기 복제 (기본값)
synchronous_commit = off

# 동기 복제
synchronous_commit = on
synchronous_standby_names = 'standby1'

# 부분 동기 (로컬 WAL만 기록 후 응답)
synchronous_commit = local

# 원격 쓰기 (Standby OS까지만, 디스크 fsync 대기 안 함)
synchronous_commit = remote_write

# 원격 적용 (Standby 디스크 fsync까지 대기)
synchronous_commit = remote_apply
```

---

## 5. Streaming Replication 아키텍처

### 🏗️ 기본 구조

```
Primary-Standby 구조
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────┐
│          Primary (읽기/쓰기)            │
│  ┌──────────┐         ┌──────────┐     │
│  │   WAL    │────────>│ WAL Sender│     │
│  │ Generator│         │  Process  │     │
│  └──────────┘         └──────────┘     │
└─────────────┬───────────────────────────┘
              │ WAL Stream 📡
              ↓
┌─────────────┴───────────────────────────┐
│        Standby (읽기 전용)              │
│  ┌──────────┐         ┌──────────┐     │
│  │   WAL    │<────────│ WAL Receiver│   │
│  │  Replay  │         │   Process   │   │
│  └──────────┘         └──────────┘     │
└─────────────────────────────────────────┘

WAL Sender:
- Primary에서 실행
- WAL 레코드를 Standby로 전송
- 각 Standby당 1개 프로세스

WAL Receiver:
- Standby에서 실행
- WAL 레코드 수신 및 저장
- Standby당 1개 프로세스

WAL Replay (Startup Process):
- 수신한 WAL을 데이터 파일에 적용
- 읽기 쿼리와 동시 실행 가능 (Hot Standby)
```

### 🔄 복제 흐름

```
1. 클라이언트 → Primary: INSERT INTO users VALUES (1, 'Alice');

2. Primary:
   ├─ 트랜잭션 실행
   ├─ WAL 기록: "INSERT users 1, Alice"
   ├─ 데이터 파일 수정 (버퍼)
   └─ WAL Sender: WAL을 Standby로 전송 📤

3. Standby:
   ├─ WAL Receiver: WAL 수신 📥
   ├─ pg_wal 디렉토리에 저장
   ├─ WAL Replay: WAL 재생
   └─ 데이터 파일 수정 (Primary와 동일)

4. 결과:
   Primary와 Standby가 동일한 상태 유지! ✅
```

---

## 6. Replication Slot (복제 슬롯)

### 🎰 Replication Slot이란?

```
Replication Slot = Standby가 받아야 할 WAL 보관
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

슬롯 없음:
  Primary: WAL 생성 → 일정 시간 후 삭제
  Standby: 네트워크 끊김 💔
           ↓
  필요한 WAL이 이미 삭제됨 ❌
  → 전체 재동기화 필요 (pg_basebackup)

슬롯 있음:
  Primary: WAL 생성 → Standby가 받을 때까지 보관 ✅
  Standby: 네트워크 끊김 💔
           ↓
  다시 연결
           ↓
  필요한 WAL 여전히 있음 ✅
  → 이어서 복제 가능
```

### 📋 Replication Slot 관리

```sql
-- Standby를 위한 슬롯 생성
SELECT pg_create_physical_replication_slot('standby1_slot');

-- 슬롯 목록 확인
SELECT
  slot_name,
  slot_type,
  active,
  restart_lsn,
  confirmed_flush_lsn
FROM pg_replication_slots;

-- 결과:
--  slot_name      | slot_type | active | restart_lsn | confirmed_flush_lsn
-- ----------------+-----------+--------+-------------+---------------------
--  standby1_slot  | physical  | t      | 0/3000000   | NULL

-- 슬롯 삭제 (Standby 제거 시)
SELECT pg_drop_replication_slot('standby1_slot');

-- 주의: 슬롯이 있으면 WAL이 계속 쌓임!
-- Standby가 오랫동안 연결 안 되면 디스크 가득 참 위험 🚨
```

---

## 7. 고가용성 시스템 설계

### 🏛️ 아키텍처 패턴

#### 패턴 1: 단일 Standby (기본)

```
[Primary] ──복제──> [Standby]
    ↑                  ↑
    │                  │
[애플리케이션]  [읽기 전용 애플리케이션]

장점:
✅ 간단한 구성
✅ Primary 장애 시 Standby로 Failover

단점:
❌ Standby도 장애 시 백업 없음
❌ 읽기 부하 분산 제한적
```

#### 패턴 2: 다중 Standby (부하 분산)

```
           [Primary] (쓰기)
              ↓
      ┌───────┼───────┐
      ↓       ↓       ↓
 [Standby1][Standby2][Standby3] (읽기)
      ↑       ↑       ↑
      └───────┴───────┘
      [읽기 로드밸런서]

장점:
✅ 읽기 부하 3배 분산
✅ Standby 장애 시 다른 Standby 사용
✅ 지리적 분산 가능

설정:
synchronous_standby_names = 'FIRST 1 (standby1, standby2, standby3)'
# 3개 중 1개만 동기화 대기 (나머지는 비동기)
```

#### 패턴 3: Cascading Replication (계단식)

```
[Primary]
    ↓
[Standby1] (중간 노드)
    ↓
    ├─> [Standby2]
    ├─> [Standby3]
    └─> [Standby4]

장점:
✅ Primary 부하 감소 (1개 Standby에만 전송)
✅ 대규모 확장 가능

단점:
❌ 복제 지연 증가 (Primary → Standby1 → Standby2)
❌ Standby1 장애 시 하위 Standby 영향

사용 사례:
- 많은 수의 읽기 복제본 필요
- 네트워크 대역폭 제한
```

#### 패턴 4: 지리적 분산 (Multi-Region)

```
서울 IDC                      부산 IDC
[Primary] ──비동기 복제──> [Standby1]
    ↓                            ↓
[Standby2]                  [Standby3]
(동기 복제)                 (로컬 읽기)

설정:
synchronous_standby_names = 'standby2'
# 서울 내 Standby2만 동기 (빠름)
# 부산 Standby1은 비동기 (DR 용도)

장점:
✅ 로컬 성능 + 원격 DR
✅ 재해 복구 가능
```

---

## 8. Replication 모니터링

### 📊 주요 모니터링 지표

```sql
-- 1. Standby 연결 상태 확인 (Primary에서)
SELECT
  client_addr AS standby_ip,
  application_name,
  state,
  sync_state,
  pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
  pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag_bytes,
  pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag_bytes,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;

-- 결과 예시:
--  standby_ip   | application_name | state     | sync_state | send_lag_bytes | replay_lag_bytes
-- --------------+------------------+-----------+------------+----------------+------------------
--  10.0.1.100   | standby1         | streaming | async      |              0 |             4096
--  10.0.1.101   | standby2         | streaming | sync       |              0 |                0

-- state:
--   startup: 초기 동기화 중
--   catchup: 따라잡는 중
--   streaming: 정상 복제 중
--   backup: 베이스 백업 중

-- sync_state:
--   sync: 동기 복제
--   async: 비동기 복제
--   potential: 동기 후보

-- 2. 복제 지연 시간 확인 (PostgreSQL 10+)
SELECT
  application_name,
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;

--  application_name | write_lag  | flush_lag  | replay_lag
-- ------------------+------------+------------+-------------
--  standby1         | 00:00:00.5 | 00:00:01.2 | 00:00:02.1

-- 3. Standby에서 지연 확인
SELECT
  pg_last_wal_receive_lsn() AS receive_lsn,
  pg_last_wal_replay_lsn() AS replay_lsn,
  pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS lag_bytes,
  EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;

--  receive_lsn | replay_lsn | lag_bytes | lag_seconds
-- -------------+------------+-----------+-------------
--  0/5000000   | 0/4FFF000  |      4096 |         2.5
-- (2.5초 지연, 4KB 뒤처짐)
```

### 🚨 알림 기준

```sql
-- 위험 수준 판단
SELECT
  application_name,
  CASE
    WHEN replay_lag > interval '1 minute' THEN '🔴 CRITICAL'
    WHEN replay_lag > interval '10 seconds' THEN '🟡 WARNING'
    ELSE '🟢 OK'
  END AS status,
  replay_lag
FROM pg_stat_replication;

-- 권장 알림 기준:
-- 🟢 OK: replay_lag < 10초
-- 🟡 WARNING: 10초 ≤ replay_lag < 1분
-- 🔴 CRITICAL: replay_lag ≥ 1분
```

---

## 9. Replication 선택 가이드

### 🎯 시나리오별 추천

```
┌─────────────────────────────────────────────────────────┐
│ 사용 사례별 Replication 선택                            │
└─────────────────────────────────────────────────────────┘

1️⃣ 고가용성 (HA) 우선
   → Physical + Synchronous
   예: 금융, 의료, 전자상거래
   설정: synchronous_standby_names = 'standby1'

2️⃣ 읽기 성능 향상
   → Physical + Asynchronous (다중 Standby)
   예: SNS 피드, 검색, 분석
   설정: Hot Standby + 로드밸런서

3️⃣ 재해 복구 (DR)
   → Physical + Asynchronous (원격지)
   예: 모든 프로덕션 시스템
   설정: 지리적 분산 (서울 ↔ 부산)

4️⃣ 데이터 통합
   → Logical Replication
   예: 여러 지점 DB → 본사 통합
   설정: 특정 테이블만 선택적 복제

5️⃣ 버전 업그레이드
   → Logical Replication
   예: PostgreSQL 12 → 14 마이그레이션
   설정: 다른 버전 간 복제

6️⃣ 멀티 테넌트 분리
   → Logical Replication
   예: tenant_a 테이블만 별도 DB로
   설정: 테이블/행 필터링
```

---

## 10. 실습 과제

### 🎯 과제 1: 아키텍처 설계
다음 요구사항에 맞는 Replication 아키텍처를 설계하세요:

```
요구사항:
- 서비스: 전자상거래 (24/7 운영)
- 사용자: 100만명
- 읽기:쓰기 비율 = 9:1
- 가용성: 99.99% 이상
- RPO: 0초 (데이터 손실 불가)
- RTO: 5분 이내
- 예산: 중간 수준

설계 항목:
1. Primary 위치
2. Standby 개수 및 위치
3. 동기/비동기 선택
4. Replication Slot 사용 여부
5. 모니터링 항목
```

### 🎯 과제 2: 모니터링 쿼리 작성
Replication 상태를 한눈에 볼 수 있는 뷰를 작성하세요:

```sql
-- 요구사항:
-- 1. Standby 이름, IP
-- 2. 연결 상태 (streaming, catchup 등)
-- 3. 복제 지연 (초, 바이트)
-- 4. 동기/비동기 구분
-- 5. 상태 (OK/WARNING/CRITICAL)

CREATE OR REPLACE VIEW v_replication_status AS
-- 여기에 작성하세요
;

SELECT * FROM v_replication_status;
```

### 🎯 과제 3: 장애 시나리오 분석
다음 각 장애 상황에서 어떻게 대응할지 작성하세요:

```
1. Primary 서버 디스크 고장
   - 즉시 조치:
   - 복구 절차:
   - 데이터 손실 여부:

2. Standby 네트워크 단절 (1시간)
   - Primary 영향:
   - 데이터 동기화:
   - 재연결 후 조치:

3. Primary-Standby 간 네트워크 지연 (1초)
   - 동기 복제 영향:
   - 비동기 복제 영향:
   - 해결 방안:
```

---

## 11. 다음 단계

Replication 개요를 마스터했으니, 이제 실제 구축을 배울 차례입니다!

### 다음 챕터 미리보기
```
Chapter 35: Streaming Replication
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 배울 내용:
   ├─ Primary 서버 설정 (postgresql.conf, pg_hba.conf)
   ├─ Standby 서버 구축 (pg_basebackup)
   ├─ Hot Standby 활성화 (읽기 전용 쿼리)
   ├─ Replication Slot 생성
   ├─ 동기/비동기 복제 전환
   └─ 복제 지연 해결

💡 실습:
   ├─ 로컬에서 Primary + Standby 구축
   ├─ WAL Sender/Receiver 프로세스 확인
   ├─ 복제 지연 모니터링
   ├─ Standby에서 읽기 쿼리 실행
   └─ Failover 시뮬레이션 (수동)
```

---

## 📖 정리

이번 챕터에서 배운 내용:

| 개념 | 핵심 내용 |
|------|----------|
| **Replication** | 데이터를 여러 서버에 복사하여 고가용성/성능 향상 |
| **Physical** | 바이트 단위 복사, 빠르고 간단, HA/DR 용도 |
| **Logical** | SQL 문 기반, 선택적 복제, 데이터 통합 용도 |
| **Synchronous** | 확인 후 응답, 데이터 손실 없음, 느림 |
| **Asynchronous** | 즉시 응답, 빠름, 일부 손실 가능 |
| **Replication Slot** | WAL 보관 보장, 네트워크 단절 대비 |

### 핵심 아키텍처

```
고가용성 시스템 구성 요소
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Primary (1대): 모든 쓰기 처리
2. Standby (N대): 읽기 분산 + Failover 대기
3. Replication: WAL 기반 실시간 복제
4. Monitoring: 복제 지연 추적
5. Failover: 자동/수동 Primary 전환
```

**기억하세요**: "Replication은 고가용성의 핵심입니다. 하지만 복제만으로는 충분하지 않습니다. 모니터링, Failover 계획, 정기 훈련이 함께 있어야 진정한 고가용성 시스템입니다!"

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐ (고급)
**학습 시간**: 90분
**대상**: 백엔드 개발자, 데브옵스 엔지니어, DBA
**중요도**: 🔥🔥🔥🔥🔥 (매우 높음)

이전: [Chapter 33. SQL Injection 방어](../part09-보안/33-sql-injection-방어.md)
다음: [Chapter 35. Streaming Replication](35-streaming-replication.md)