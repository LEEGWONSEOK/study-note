# Chapter 30. PITR (Point-In-Time Recovery)

## 🎯 이 챕터의 목표
- **PITR**의 개념과 작동 원리 이해하기
- **WAL (Write-Ahead Log)** 메커니즘 완벽 정복
- **베이스 백업 + WAL 아카이빙** 구축하기
- **특정 시점으로 복구**하는 방법 익히기
- **연속 아카이빙** 시스템 구축하기
- **실전 재해 복구** 시나리오 대응하기

---

## 1. PITR이란? (비유로 이해하기)

### 🎬 영화 제작에 비유하면

```
PITR = 영화의 특정 장면으로 되돌리기
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🎥 베이스 백업 (Base Backup)
   = 영화의 시작 프레임
   "모든 것이 정상이었던 초기 상태"

📹 WAL (Write-Ahead Log)
   = 이후의 모든 변경 사항을 녹화
   "프레임 1 → 프레임 2 → 프레임 3 → ..."

⏮️ PITR (Point-In-Time Recovery)
   = 원하는 장면으로 되돌리기
   "14:35:22 시점의 DB 상태로 복구"

예시:
  일요일 00:00 - 베이스 백업 📸
  월요일 10:00 - 데이터 계속 변경 중 📝
  월요일 14:35 - 실수로 중요 데이터 삭제! 💥

  복구:
  베이스 백업 복원 + WAL을 14:34까지 재생
  → 삭제하기 1분 전 상태로 복구! ✅
```

### 🎯 PITR의 장점

```
전통적인 백업 vs PITR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📅 전통적인 백업 (하루 1회):
   일요일 00:00 ─────┐
   월요일 00:00 ─────┤  ← 백업 주기
   화요일 00:00 ─────┤
   수요일 14:35 💥 장애

   복원: 화요일 00:00 백업 사용
   손실: 화요일 00:00 ~ 수요일 14:35 (38시간 35분) ❌
   RPO = 최대 24시간

⏱️ PITR (연속 아카이빙):
   일요일 00:00 ─── 베이스 백업 📸
   월요일 ~ 수요일 ─── WAL 계속 저장 📝
   수요일 14:35 💥 장애

   복원: 베이스 + WAL을 14:34까지 재생
   손실: 거의 없음 (수 초 ~ 수 분) ✅
   RPO = 거의 0

결론: PITR이 훨씬 안전!
```

---

## 2. WAL (Write-Ahead Log) 이해하기

### 📝 WAL의 작동 원리

```
WAL = 모든 변경 사항을 먼저 로그에 기록
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 클라이언트가 UPDATE 요청
   UPDATE users SET name = 'Alice' WHERE id = 1;

2. PostgreSQL의 처리 순서:

   ① WAL에 먼저 기록 📝
      "id=1 레코드의 name을 'Alice'로 변경"
      → /var/lib/postgresql/14/main/pg_wal/000000010000000000000001

   ② 메모리(버퍼)에 변경 💾
      Shared Buffer에 수정된 데이터 저장

   ③ 나중에 디스크에 기록 💿
      Checkpoint 시점에 실제 데이터 파일 갱신

3. 장애 발생 시 복구:
   디스크에 기록 전 서버가 죽어도
   WAL을 재생하면 데이터 복구 가능! ✅
```

### 🗂️ WAL 파일 구조

```bash
# WAL 파일 위치 확인
psql -U postgres -c "SHOW data_directory;"
#  /var/lib/postgresql/14/main

# WAL 디렉토리
ls -lh /var/lib/postgresql/14/main/pg_wal/
# total 512M
# -rw------- 1 postgres postgres 16M Jan 15 10:00 000000010000000000000001
# -rw------- 1 postgres postgres 16M Jan 15 10:16 000000010000000000000002
# -rw------- 1 postgres postgres 16M Jan 15 10:32 000000010000000000000003
# ...

# WAL 파일명 구조:
# 000000010000000000000001
# ├─────┤├───────┤├───────┤
# │      │        └─ 파일 번호 (16진수)
# │      └─ 로그 세그먼트
# └─ 타임라인 ID

# 각 WAL 파일 크기: 16MB (기본값)
```

### WAL 관련 설정

```sql
-- 현재 WAL 설정 확인
SHOW wal_level;          -- replica (기본값)
SHOW max_wal_size;       -- 1GB
SHOW min_wal_size;       -- 80MB
SHOW wal_segment_size;   -- 16777216 (16MB)

-- postgresql.conf 주요 설정
```

```ini
# postgresql.conf

# WAL 레벨 (PITR에 필수)
wal_level = replica
# minimal: 최소 로깅 (PITR 불가)
# replica: 복제 가능 (기본값, PITR 가능)
# logical: 논리적 복제 가능

# WAL 버퍼 크기
wal_buffers = 16MB

# Checkpoint 주기
checkpoint_timeout = 5min
checkpoint_completion_target = 0.9

# WAL 보관
archive_mode = on
archive_command = 'cp %p /backup/wal_archive/%f'
# %p: WAL 파일 전체 경로
# %f: WAL 파일명
```

---

## 3. 베이스 백업 + WAL 아카이빙 구축

### 🔧 1단계: WAL 아카이빙 설정

#### postgresql.conf 수정
```bash
# 1. 설정 파일 열기
sudo nano /etc/postgresql/14/main/postgresql.conf

# 2. 다음 내용 추가/수정
```

```ini
# WAL 아카이빙 설정
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'
archive_timeout = 300  # 5분마다 강제로 WAL 전환 (선택사항)

# Checkpoint 설정
max_wal_size = 2GB
min_wal_size = 1GB
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9
```

```bash
# 3. WAL 아카이브 디렉토리 생성
sudo mkdir -p /backup/wal_archive
sudo chown postgres:postgres /backup/wal_archive
sudo chmod 700 /backup/wal_archive

# 4. PostgreSQL 재시작
sudo systemctl restart postgresql

# 5. 설정 확인
psql -U postgres -c "SHOW archive_mode;"
#  on

psql -U postgres -c "SHOW archive_command;"
#  test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f
```

#### archive_command 상세 설명
```bash
# archive_command의 구조:
test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f
├─────────────────────────────┤    ├──────────────────────────┤
│                               │    │
│                               │    └─ 파일 복사
│                               └─ AND 연산자
└─ 파일이 존재하지 않으면 (중복 방지)

# %p: WAL 파일 전체 경로
# 예: /var/lib/postgresql/14/main/pg_wal/000000010000000000000001

# %f: WAL 파일명만
# 예: 000000010000000000000001

# 동작:
# 1. /backup/wal_archive/000000010000000000000001 파일이 없으면
# 2. 복사 실행
# 3. 이미 있으면 건너뛰기 (멱등성 보장)
```

### 🗄️ 2단계: 베이스 백업 생성

#### pg_basebackup 사용
```bash
# 1. 베이스 백업 디렉토리 생성
sudo mkdir -p /backup/base
sudo chown postgres:postgres /backup/base

# 2. 베이스 백업 실행
pg_basebackup -h localhost -U postgres \
  -D /backup/base/$(date +%Y%m%d_%H%M%S) \
  -Fp \
  -Xs \
  -P \
  -v

# 옵션 설명:
# -D: 백업 저장 디렉토리
# -Fp: Plain 포맷 (디렉토리 구조 그대로)
# -Xs: WAL을 베이스 백업에 포함 (stream)
# -P: 진행 상황 표시
# -v: 상세 출력

# 실행 결과:
# pg_basebackup: initiating base backup, waiting for checkpoint to complete
# pg_basebackup: checkpoint completed
# pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
# 24563/24563 kB (100%), 1/1 tablespace
# pg_basebackup: write-ahead log end point: 0/2000138
# pg_basebackup: syncing data to disk ...
# pg_basebackup: base backup completed
```

#### 베이스 백업 내용 확인
```bash
# 백업 디렉토리 구조
ls -lh /backup/base/20240115_140000/
# total 125M
# drwx------ 2 postgres postgres 4.0K Jan 15 14:00 base/
# drwx------ 2 postgres postgres 4.0K Jan 15 14:00 global/
# drwx------ 2 postgres postgres 4.0K Jan 15 14:00 pg_wal/
# -rw------- 1 postgres postgres  227 Jan 15 14:00 backup_label
# -rw------- 1 postgres postgres   88 Jan 15 14:00 backup_manifest
# -rw------- 1 postgres postgres  29K Jan 15 14:00 postgresql.conf
# ...

# backup_label 파일 (중요!)
cat /backup/base/20240115_140000/backup_label
# START WAL LOCATION: 0/2000028 (file 000000010000000000000002)
# CHECKPOINT LOCATION: 0/2000060
# BACKUP METHOD: streamed
# BACKUP FROM: primary
# START TIME: 2024-01-15 14:00:00 KST
# LABEL: pg_basebackup base backup
# START TIMELINE: 1
```

### 📋 3단계: 백업 검증

```bash
# 1. WAL 아카이빙 작동 확인
ls -lh /backup/wal_archive/
# -rw------- 1 postgres postgres 16M Jan 15 14:00 000000010000000000000001
# -rw------- 1 postgres postgres 16M Jan 15 14:05 000000010000000000000002
# -rw------- 1 postgres postgres 16M Jan 15 14:10 000000010000000000000003

# 2. 데이터 변경 후 WAL 생성 확인
psql -U postgres -c "
  CREATE TABLE test_pitr (id SERIAL, created_at TIMESTAMPTZ DEFAULT now());
  INSERT INTO test_pitr DEFAULT VALUES;
  SELECT pg_switch_wal();  -- 강제로 WAL 전환
"

# 3. 새 WAL 파일 확인
ls -lh /backup/wal_archive/ | tail -1
# -rw------- 1 postgres postgres 16M Jan 15 14:15 000000010000000000000004

# ✅ WAL 아카이빙 정상 작동 중!
```

---

## 4. PITR로 복구하기

### 🎯 시나리오: 실수로 데이터 삭제

```sql
-- 현재 시각: 2024-01-15 14:30:00

-- 1. 정상적인 데이터 확인
SELECT * FROM users;
-- id | name  | email               | created_at
-- ---+-------+---------------------+-------------------------
--  1 | Alice | alice@example.com   | 2024-01-15 10:00:00+09
--  2 | Bob   | bob@example.com     | 2024-01-15 11:00:00+09
--  3 | Carol | carol@example.com   | 2024-01-15 12:00:00+09

-- 2. 실수로 모든 데이터 삭제! (14:30:45)
DELETE FROM users;
-- DELETE 3

SELECT * FROM users;
-- (0 rows)

-- 😱 큰일났다! 하지만 PITR이 있으니 걱정 없음!
```

### 🔄 복구 프로세스

#### 1단계: 서비스 중단 및 준비
```bash
# 1. PostgreSQL 중지
sudo systemctl stop postgresql

# 2. 기존 데이터 디렉토리 백업 (안전을 위해)
sudo mv /var/lib/postgresql/14/main /var/lib/postgresql/14/main.old

# 3. 새 데이터 디렉토리 생성
sudo mkdir /var/lib/postgresql/14/main
sudo chown postgres:postgres /var/lib/postgresql/14/main
sudo chmod 700 /var/lib/postgresql/14/main
```

#### 2단계: 베이스 백업 복원
```bash
# 베이스 백업 복사
sudo cp -r /backup/base/20240115_140000/* /var/lib/postgresql/14/main/

# 권한 설정
sudo chown -R postgres:postgres /var/lib/postgresql/14/main
sudo chmod -R 700 /var/lib/postgresql/14/main
```

#### 3단계: 복구 설정 (PostgreSQL 12 이상)
```bash
# recovery.signal 파일 생성
sudo -u postgres touch /var/lib/postgresql/14/main/recovery.signal

# postgresql.conf에 복구 설정 추가
sudo -u postgres tee -a /var/lib/postgresql/14/main/postgresql.conf > /dev/null <<EOF

# PITR 복구 설정
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:40'  # 삭제 5초 전
recovery_target_action = 'promote'
EOF
```

#### 4단계: PostgreSQL 시작 및 복구 실행
```bash
# PostgreSQL 시작
sudo systemctl start postgresql

# 로그 모니터링
sudo tail -f /var/log/postgresql/postgresql-14-main.log
# 2024-01-15 14:35:00 KST [1234]: LOG:  starting point-in-time recovery to 2024-01-15 14:30:40+09
# 2024-01-15 14:35:00 KST [1234]: LOG:  restored log file "000000010000000000000001" from archive
# 2024-01-15 14:35:01 KST [1234]: LOG:  redo starts at 0/2000028
# 2024-01-15 14:35:02 KST [1234]: LOG:  restored log file "000000010000000000000002" from archive
# 2024-01-15 14:35:03 KST [1234]: LOG:  restored log file "000000010000000000000003" from archive
# 2024-01-15 14:35:04 KST [1234]: LOG:  recovery stopping before commit of transaction 1234, time 2024-01-15 14:30:45
# 2024-01-15 14:35:04 KST [1234]: LOG:  recovery has paused
# 2024-01-15 14:35:04 KST [1234]: LOG:  redo done at 0/3000500
# 2024-01-15 14:35:05 KST [1234]: LOG:  selected new timeline ID: 2
# 2024-01-15 14:35:05 KST [1234]: LOG:  archive recovery complete
# 2024-01-15 14:35:06 KST [1234]: LOG:  database system is ready to accept connections
```

#### 5단계: 데이터 확인
```sql
-- PostgreSQL 접속
psql -U postgres

-- 데이터 확인
SELECT * FROM users;
-- id | name  | email               | created_at
-- ---+-------+---------------------+-------------------------
--  1 | Alice | alice@example.com   | 2024-01-15 10:00:00+09
--  2 | Bob   | bob@example.com     | 2024-01-15 11:00:00+09
--  3 | Carol | carol@example.com   | 2024-01-15 12:00:00+09

-- ✅ 복구 성공! DELETE 하기 전 상태로 돌아옴!
```

---

## 5. 복구 목표 설정 방법

### 🎯 다양한 복구 목표

PostgreSQL은 여러 방식으로 복구 지점을 지정할 수 있습니다:

#### 1️⃣ 시간 기반 복구 (가장 일반적)
```ini
# postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
```

```sql
-- 활용 예시:
-- "오후 2시 30분에 무언가 잘못됐어요"
-- → recovery_target_time = '2024-01-15 14:30:00'
```

#### 2️⃣ 트랜잭션 ID 기반 복구
```ini
# postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_xid = '1234567'
```

```sql
-- 트랜잭션 ID 확인 방법:
SELECT txid_current();  -- 현재 트랜잭션 ID
--  1234567

-- 활용 예시:
-- "트랜잭션 1234567 직전으로 돌아가야 해요"
-- → recovery_target_xid = '1234566'
```

#### 3️⃣ LSN (Log Sequence Number) 기반 복구
```ini
# postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_lsn = '0/3000500'
```

```sql
-- LSN 확인:
SELECT pg_current_wal_lsn();
--  0/3000500

-- 활용 예시: 고급 사용자, 정밀한 복구 지점
```

#### 4️⃣ 특정 명칭 기반 복구 (Named Restore Point)
```sql
-- 복구 지점 생성 (중요한 작업 전)
SELECT pg_create_restore_point('before_major_update');
--  0/3000500
```

```ini
# postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_name = 'before_major_update'
```

#### 5️⃣ 즉시 복구 (Immediate)
```ini
# postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target = 'immediate'
# 베이스 백업 직후 상태로 복구 (WAL 재생 안 함)
```

### 🎛️ 복구 후 동작 설정

```ini
# recovery_target_action: 복구 목표 도달 후 동작

# 1. promote (기본값): 즉시 운영 모드로 전환
recovery_target_action = 'promote'

# 2. pause: 일시 중지 (데이터 확인 후 수동 promote)
recovery_target_action = 'pause'

# 3. shutdown: PostgreSQL 종료
recovery_target_action = 'shutdown'
```

#### pause를 사용한 안전한 복구
```ini
# postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'pause'
```

```bash
# PostgreSQL 시작
sudo systemctl start postgresql

# 복구 완료 후 일시 중지됨
```

```sql
-- 읽기 전용 모드로 데이터 확인
psql -U postgres

SELECT * FROM users;
-- 데이터가 정확히 복구되었는지 확인

-- 확인 후 promote (운영 모드로 전환)
SELECT pg_wal_replay_resume();

-- 또는 취소하고 다시 시도
-- sudo systemctl stop postgresql
-- (다른 시간으로 recovery_target_time 변경 후 재시도)
```

---

## 6. 타임라인 관리

### 📈 타임라인이란?

```
타임라인 = PITR 복구 후 새로운 역사 시작
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Timeline 1 (원본):
  일요일 ────┬──── 월요일 ────┬──── 화요일 💥 (14:30 DELETE)
             │                 │
             베이스 백업       │
                               │
                               └── PITR 복구 (14:25로)

Timeline 2 (복구 후):
  일요일 ────┴──── 월요일 ────┴──── 화요일 (14:25) ───→ 새로운 미래

왜 필요한가?
- 복구 후에도 원래 WAL과 충돌하지 않음
- 여러 번 복구 가능
- 복구 이력 추적
```

### 타임라인 파일 확인

```bash
# 타임라인 히스토리 파일
ls /var/lib/postgresql/14/main/pg_wal/
# 00000001.history  <- 없음 (Timeline 1은 원본)
# 00000002.history  <- Timeline 2로 복구한 이력
# 000000010000000000000001  <- Timeline 1의 WAL
# 000000020000000000000004  <- Timeline 2의 WAL

# 타임라인 히스토리 내용
cat /var/lib/postgresql/14/main/pg_wal/00000002.history
# 1	0/3000500	before 2024-01-15 14:30:00+09
# └─ Timeline 1에서 LSN 0/3000500 지점에서 분기
```

### 특정 타임라인으로 복구

```ini
# postgresql.conf

# 기본: 최신 타임라인 따라가기
recovery_target_timeline = 'latest'

# 특정 타임라인 지정
recovery_target_timeline = '1'

# 예: Timeline 2로 복구했다가 잘못된 경우
# Timeline 1의 다른 시점으로 다시 복구 가능
```

---

## 7. 연속 아카이빙 자동화

### 🤖 프로덕션급 백업 시스템

#### 완전 자동화 스크립트
```bash
#!/bin/bash
# pitr-backup-system.sh

set -e

# 설정
BACKUP_ROOT="/backup"
BASE_DIR="$BACKUP_ROOT/base"
WAL_DIR="$BACKUP_ROOT/wal_archive"
LOG_FILE="/var/log/pitr_backup.log"
S3_BUCKET="s3://company-backups/postgres"
RETENTION_DAYS=7

# 로그 함수
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

error_exit() {
    log "❌ ERROR: $1"
    # Slack/PagerDuty 알림
    curl -X POST $SLACK_WEBHOOK_URL \
      -d "{\"text\":\"⚠️ PITR 백업 실패: $1\"}"
    exit 1
}

# 1. 베이스 백업 실행 (매주 일요일)
if [ "$(date +\%u)" -eq 7 ]; then
    log "🔄 베이스 백업 시작"

    BACKUP_NAME="base_$(date +%Y%m%d_%H%M%S)"
    BACKUP_PATH="$BASE_DIR/$BACKUP_NAME"

    pg_basebackup -h localhost -U postgres \
      -D "$BACKUP_PATH" \
      -Fp -Xs -P -v 2>&1 | tee -a "$LOG_FILE" || \
      error_exit "pg_basebackup 실패"

    # 압축
    log "🗜️  베이스 백업 압축 중"
    tar czf "$BACKUP_PATH.tar.gz" -C "$BASE_DIR" "$BACKUP_NAME"
    rm -rf "$BACKUP_PATH"

    # S3 업로드
    log "☁️  S3 업로드 중"
    aws s3 cp "$BACKUP_PATH.tar.gz" \
      "$S3_BUCKET/base/$BACKUP_NAME.tar.gz" || \
      error_exit "S3 업로드 실패"

    log "✅ 베이스 백업 완료: $BACKUP_NAME"
fi

# 2. WAL 파일 S3 동기화 (매시간)
log "🔄 WAL 파일 동기화 중"
aws s3 sync "$WAL_DIR" "$S3_BUCKET/wal/" \
  --exclude "*" --include "*.backup" --include "0*" || \
  error_exit "WAL 동기화 실패"

# 3. 오래된 로컬 백업 삭제
log "🗑️  오래된 백업 삭제 중"
find "$BASE_DIR" -name "base_*.tar.gz" -mtime +$RETENTION_DAYS -delete
find "$WAL_DIR" -name "0*" -mtime +$RETENTION_DAYS -delete

# 4. 백업 검증
log "🔍 백업 무결성 검증 중"

# WAL 파일 개수 확인
WAL_COUNT=$(ls -1 "$WAL_DIR" | wc -l)
if [ "$WAL_COUNT" -lt 10 ]; then
    error_exit "WAL 파일 수가 너무 적음: $WAL_COUNT"
fi

# 베이스 백업 개수 확인
BASE_COUNT=$(ls -1 "$BASE_DIR" | wc -l)
if [ "$BASE_COUNT" -lt 1 ]; then
    error_exit "베이스 백업 없음"
fi

log "✅ 백업 시스템 정상: WAL $WAL_COUNT개, 베이스 $BASE_COUNT개"

# 5. 메트릭 전송
OLDEST_WAL=$(ls -1 "$WAL_DIR" | head -1)
NEWEST_WAL=$(ls -1 "$WAL_DIR" | tail -1)

curl -X POST $MONITORING_URL \
  -d "{
    \"service\": \"postgresql-pitr\",
    \"wal_count\": $WAL_COUNT,
    \"base_count\": $BASE_COUNT,
    \"oldest_wal\": \"$OLDEST_WAL\",
    \"newest_wal\": \"$NEWEST_WAL\"
  }"

log "🎉 백업 시스템 점검 완료"
```

#### Crontab 설정
```bash
# crontab -e

# 매주 일요일 02:00 - 베이스 백업
0 2 * * 0 /usr/local/bin/pitr-backup-system.sh

# 매시간 - WAL 동기화
0 * * * * /usr/local/bin/pitr-backup-system.sh

# 매일 03:00 - 백업 검증
0 3 * * * /usr/local/bin/verify-pitr-backup.sh
```

---

## 8. 실전 재해 복구 시나리오

### 🔥 시나리오 1: 랜섬웨어 공격

```
상황:
  - 2024-01-15 15:30 - 랜섬웨어가 DB 암호화
  - 마지막 베이스 백업: 2024-01-14 02:00
  - WAL 아카이빙: 정상 작동 중
  - 목표: 15:29까지 복구 (암호화 직전)
```

#### 복구 절차
```bash
#!/bin/bash
# disaster-recovery-ransomware.sh

log "🚨 랜섬웨어 복구 시작"

# 1. 감염된 서버 격리
log "🔒 감염된 서버 네트워크 차단"
sudo iptables -A INPUT -j DROP
sudo iptables -A OUTPUT -j DROP

# 2. 새 서버 준비
NEW_SERVER="recovery-server-01"
ssh $NEW_SERVER "
  sudo apt update
  sudo apt install -y postgresql-14
  sudo systemctl stop postgresql
"

# 3. 베이스 백업 다운로드 (S3에서)
log "📥 베이스 백업 다운로드"
ssh $NEW_SERVER "
  aws s3 cp s3://company-backups/postgres/base/base_20240114_020000.tar.gz /tmp/
  sudo tar xzf /tmp/base_20240114_020000.tar.gz -C /var/lib/postgresql/14/
  sudo mv /var/lib/postgresql/14/base_20240114_020000 /var/lib/postgresql/14/main
"

# 4. WAL 파일 다운로드
log "📥 WAL 파일 다운로드"
ssh $NEW_SERVER "
  sudo mkdir -p /backup/wal_archive
  aws s3 sync s3://company-backups/postgres/wal/ /backup/wal_archive/
"

# 5. 복구 설정
ssh $NEW_SERVER "
  sudo -u postgres touch /var/lib/postgresql/14/main/recovery.signal
  sudo tee -a /var/lib/postgresql/14/main/postgresql.conf > /dev/null <<EOF

restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 15:29:00'
recovery_target_action = 'promote'
EOF
"

# 6. PostgreSQL 시작
log "🔄 PostgreSQL 시작 및 복구"
ssh $NEW_SERVER "
  sudo chown -R postgres:postgres /var/lib/postgresql/14/main
  sudo systemctl start postgresql
"

# 7. 복구 모니터링
log "👀 복구 진행 상황 모니터링"
ssh $NEW_SERVER "tail -f /var/log/postgresql/postgresql-14-main.log"

# 8. 복구 완료 확인
log "✅ 복구 완료, 데이터 검증 중"
ssh $NEW_SERVER "
  psql -U postgres -c 'SELECT COUNT(*) FROM users;'
  psql -U postgres -c 'SELECT MAX(created_at) FROM orders;'
"

log "🎉 재해 복구 성공! 새 서버: $NEW_SERVER"
```

### 🗄️ 시나리오 2: 디스크 장애

```
상황:
  - Primary DB 서버의 디스크 완전 고장
  - 데이터 완전 손실
  - Stand-by 서버 있음 (Streaming Replication)
  - 목표: 최소 다운타임으로 복구
```

```bash
#!/bin/bash
# disaster-recovery-disk-failure.sh

log "💥 디스크 장애 복구 시작"

# 1. Stand-by 서버를 Primary로 승격
log "🔄 Stand-by → Primary 승격"
ssh standby-server "
  sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main
"

# 2. 애플리케이션 연결 전환
log "🔀 애플리케이션 연결 전환"
# DNS 변경 또는 로드밸런서 설정 변경
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "STANDBY_SERVER_IP"}]
      }
    }]
  }'

# 3. 고장난 서버 복구 (디스크 교체 후)
log "🛠️  고장난 서버 재구축"
ssh failed-server "
  # 디스크 교체 완료 가정
  sudo mkfs.ext4 /dev/sdb
  sudo mount /dev/sdb /var/lib/postgresql

  # Stand-by로 재구성
  pg_basebackup -h standby-server -U replication \
    -D /var/lib/postgresql/14/main \
    -Fp -Xs -P

  # Streaming Replication 설정
  sudo tee /var/lib/postgresql/14/main/postgresql.auto.conf > /dev/null <<EOF
primary_conninfo = 'host=standby-server port=5432 user=replication'
EOF

  sudo -u postgres touch /var/lib/postgresql/14/main/standby.signal
  sudo systemctl start postgresql
"

log "✅ 복구 완료! 새 Primary: standby-server, 새 Standby: failed-server"

# 총 다운타임: 약 2~5분 (Stand-by 승격 시간)
```

### 📊 시나리오 3: 잘못된 배포 롤백

```
상황:
  - 2024-01-15 16:00 - 새 배포 (스키마 변경 포함)
  - 16:15 - 치명적 버그 발견
  - 목표: 배포 직전(15:59) 상태로 롤백
```

```bash
#!/bin/bash
# rollback-bad-deployment.sh

log "🔙 배포 롤백 시작"

# 1. 복구 지점 생성 (배포 전 습관화!)
# 배포 전에 실행했어야 할 명령:
# psql -U postgres -c "SELECT pg_create_restore_point('before_v2.5_deploy');"

# 2. 현재 상태 백업 (혹시 모르니)
pg_dump -Fc mydb > /tmp/before_rollback_$(date +%Y%m%d_%H%M%S).dump

# 3. PostgreSQL 중지
sudo systemctl stop postgresql

# 4. 데이터 디렉토리 백업
sudo mv /var/lib/postgresql/14/main /var/lib/postgresql/14/main.before_rollback

# 5. 베이스 백업 복원
sudo mkdir /var/lib/postgresql/14/main
sudo tar xzf /backup/base/base_20240114_020000.tar.gz \
  -C /var/lib/postgresql/14/main --strip-components=1

# 6. 복구 설정 (Named Restore Point 사용)
sudo -u postgres touch /var/lib/postgresql/14/main/recovery.signal
sudo tee -a /var/lib/postgresql/14/main/postgresql.conf > /dev/null <<EOF

restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_name = 'before_v2.5_deploy'
recovery_target_action = 'promote'
EOF

# 7. PostgreSQL 시작
sudo systemctl start postgresql

# 8. 복구 완료 확인
psql -U postgres -c "\d users"  # 스키마 확인
psql -U postgres -c "SELECT COUNT(*) FROM users;"

log "✅ 롤백 완료! 배포 전 상태로 복구됨"

# 교훈: 중요한 배포 전에는 항상 Restore Point 생성!
```

---

## 9. 모니터링과 알림

### 📊 WAL 아카이빙 모니터링

```sql
-- 1. WAL 아카이빙 상태 확인
SELECT
  archived_count,           -- 아카이빙 성공 수
  failed_count,             -- 아카이빙 실패 수
  last_archived_wal,        -- 마지막 아카이빙된 WAL
  last_archived_time        -- 마지막 아카이빙 시간
FROM pg_stat_archiver;

-- 예시 결과:
--  archived_count | failed_count | last_archived_wal         | last_archived_time
-- ----------------+--------------+---------------------------+------------------------
--            1234 |            0 | 000000010000000000000042  | 2024-01-15 16:30:15+09

-- 2. 현재 WAL 위치
SELECT pg_current_wal_lsn();
--  0/42000500

-- 3. WAL 생성 속도 (1분간)
SELECT
  pg_current_wal_lsn() AS current_lsn,
  pg_sleep(60),
  pg_current_wal_lsn() AS lsn_after_1min,
  pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/42000500')
  ) AS wal_generated;

-- 4. 아카이빙 지연 확인
SELECT
  CASE
    WHEN last_archived_time < now() - interval '5 minutes' THEN
      '⚠️  WARNING: WAL 아카이빙 5분 이상 지연!'
    ELSE
      '✅ WAL 아카이빙 정상'
  END AS status
FROM pg_stat_archiver;
```

### 🔔 알림 스크립트

```bash
#!/bin/bash
# monitor-pitr.sh

# 1. WAL 아카이빙 지연 확인
LAST_ARCHIVE_TIME=$(psql -U postgres -t -c "
  SELECT EXTRACT(EPOCH FROM (now() - last_archived_time))
  FROM pg_stat_archiver;
")

if (( $(echo "$LAST_ARCHIVE_TIME > 300" | bc -l) )); then
    # 5분 이상 지연
    curl -X POST $SLACK_WEBHOOK_URL \
      -d "{\"text\":\"⚠️ WAL 아카이빙 ${LAST_ARCHIVE_TIME}초 지연!\"}"
fi

# 2. 아카이빙 실패 확인
FAILED_COUNT=$(psql -U postgres -t -c "
  SELECT failed_count FROM pg_stat_archiver;
")

if [ "$FAILED_COUNT" -gt 0 ]; then
    curl -X POST $PAGERDUTY_URL \
      -d "{\"event\":\"wal_archive_failed\", \"count\":$FAILED_COUNT}"
fi

# 3. 디스크 공간 확인
WAL_DISK_USAGE=$(df -h /backup/wal_archive | awk 'NR==2 {print $5}' | sed 's/%//')

if [ "$WAL_DISK_USAGE" -gt 80 ]; then
    curl -X POST $SLACK_WEBHOOK_URL \
      -d "{\"text\":\"⚠️ WAL 아카이브 디스크 사용률 ${WAL_DISK_USAGE}%!\"}"
fi

# Crontab: 5분마다 실행
# */5 * * * * /usr/local/bin/monitor-pitr.sh
```

---

## 10. 성능 최적화

### ⚡ WAL 아카이빙 성능 향상

#### 1️⃣ archive_command 최적화

```ini
# ❌ 느린 방법: 파일 존재 확인 후 복사
archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'

# ✅ 빠른 방법: rsync 사용 (네트워크 백업 시)
archive_command = 'rsync -a %p backup-server:/backup/wal_archive/%f'

# ✅ 더 빠른 방법: pigz로 압축 + 병렬 전송
archive_command = 'pigz < %p > /backup/wal_archive/%f.gz'

# ✅ 클라우드: aws s3 cp (직접 S3로)
archive_command = 'aws s3 cp %p s3://bucket/wal/%f'
```

#### 2️⃣ WAL 압축 활용 (PostgreSQL 15+)

```ini
# postgresql.conf (PostgreSQL 15 이상)
wal_compression = on  # WAL 파일 압축 (약 50% 절감)
```

#### 3️⃣ archive_timeout 조정

```ini
# archive_timeout: WAL 강제 전환 주기

# ❌ 너무 짧음: 작은 WAL 파일 많이 생성
archive_timeout = 60  # 1분

# ✅ 균형: RPO와 성능 고려
archive_timeout = 300  # 5분 (일반적)

# ⚠️  너무 길면: RPO 증가
archive_timeout = 3600  # 1시간
```

### 📈 복구 성능 향상

```ini
# postgresql.conf (복구 시)

# 병렬 WAL 재생 (PostgreSQL 13+)
recovery_init_sync_method = 'syncfs'  # 빠른 초기화

# 체크포인트 설정
max_wal_size = 10GB  # 복구 시 체크포인트 간격 증가
checkpoint_timeout = 30min

# 메모리 증가
shared_buffers = 8GB
maintenance_work_mem = 2GB
```

---

## 11. 실습 과제

### 🎯 과제 1: PITR 시스템 구축
로컬 환경에 완전한 PITR 시스템을 구축하세요:

```
1. WAL 아카이빙 설정
   - archive_mode = on
   - archive_command 설정
   - /backup/wal_archive 디렉토리 생성

2. 베이스 백업 생성
   - pg_basebackup 실행
   - backup_label 파일 확인

3. 데이터 변경
   - 테스트 테이블 생성
   - 데이터 삽입
   - 특정 시점 기록 (현재 시각)

4. 데이터 손상 시뮬레이션
   - 테이블 삭제 또는 데이터 변경

5. PITR 복구 실행
   - recovery.signal 생성
   - recovery_target_time 설정
   - 복구 성공 확인
```

### 🎯 과제 2: 복구 스크립트 작성
다음 요구사항을 만족하는 복구 스크립트를 작성하세요:

```bash
#!/bin/bash
# auto-recovery.sh

# 요구사항:
# 1. 인자: 복구 목표 시간 (YYYY-MM-DD HH:MM:SS)
# 2. 베이스 백업 자동 선택 (목표 시간 이전의 가장 최근 백업)
# 3. PostgreSQL 중지
# 4. 데이터 디렉토리 백업
# 5. 베이스 백업 복원
# 6. recovery.signal 및 설정 생성
# 7. PostgreSQL 시작
# 8. 복구 완료 대기
# 9. 결과 검증 및 리포트

# 사용 예:
# ./auto-recovery.sh "2024-01-15 14:30:00"

# 여기에 작성하세요
```

### 🎯 과제 3: 모니터링 대시보드
WAL 아카이빙 상태를 모니터링하는 SQL 뷰를 작성하세요:

```sql
-- 요구사항:
-- 1. 마지막 아카이빙 시간
-- 2. 아카이빙 지연 시간 (초)
-- 3. 아카이빙 실패 횟수
-- 4. 현재 WAL 파일명
-- 5. WAL 생성 속도 (MB/hour)
-- 6. 예상 RPO (초)
-- 7. 상태 (정상/경고/위험)

CREATE OR REPLACE VIEW pitr_status AS
-- 여기에 작성하세요
;

-- 사용:
SELECT * FROM pitr_status;
```

---

## 12. 다음 단계

PITR을 마스터했으니, 이제 보안 영역으로 넘어갑니다!

### 다음 챕터 미리보기
```
Chapter 31: 사용자와 권한
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 배울 내용:
   ├─ PostgreSQL 역할(Role) 시스템
   ├─ 사용자 생성과 관리
   ├─ 권한 부여 (GRANT/REVOKE)
   ├─ 스키마별 권한 관리
   ├─ Row Level Security (RLS)
   └─ 최소 권한 원칙 적용

💡 실습:
   ├─ 애플리케이션 전용 사용자 생성
   ├─ 읽기 전용 사용자 만들기
   ├─ 백업 전용 사용자 설정
   ├─ 멀티 테넌트 권한 설계
   └─ 감사 로그 구현

🎯 보안을 배우면:
   - 최소 권한 원칙 적용
   - 데이터 유출 방지
   - 컴플라이언스 준수
```

---

## 📖 정리

이번 챕터에서 배운 내용:

| 개념 | 핵심 내용 |
|------|----------|
| **PITR** | Point-In-Time Recovery, 특정 시점 복구 |
| **WAL** | Write-Ahead Log, 모든 변경 사항 기록 |
| **베이스 백업** | pg_basebackup, 특정 시점의 전체 스냅샷 |
| **아카이빙** | WAL 파일을 안전한 위치에 보관 |
| **복구 목표** | 시간, 트랜잭션 ID, LSN, Named Restore Point |
| **타임라인** | 복구 후 새로운 역사, 분기 추적 |
| **RPO ≈ 0** | 연속 아카이빙으로 데이터 손실 최소화 |

### 핵심 명령어 및 설정

```ini
# postgresql.conf (WAL 아카이빙)
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/wal_archive/%f'
archive_timeout = 300
```

```bash
# 베이스 백업
pg_basebackup -D /backup/base -Fp -Xs -P

# 복구 설정
touch /var/lib/postgresql/14/main/recovery.signal
# postgresql.conf에 추가:
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
```

```sql
-- WAL 관리
SELECT pg_switch_wal();  -- WAL 강제 전환
SELECT pg_current_wal_lsn();  -- 현재 WAL 위치
SELECT pg_create_restore_point('my_restore_point');  -- 복구 지점 생성
```

### PITR 체크리스트

```
[ ] WAL 아카이빙 설정 완료
[ ] archive_command 테스트
[ ] 베이스 백업 정기 실행 (주 1회)
[ ] WAL 파일 오프사이트 백업 (S3 등)
[ ] 복구 절차 문서화
[ ] 정기적 복구 훈련 (분기 1회)
[ ] 모니터링 및 알림 설정
[ ] 디스크 공간 관리
```

**기억하세요**: "PITR은 프로덕션 데이터베이스의 필수 기능입니다. RPO를 거의 0에 가깝게 만들 수 있는 유일한 방법이며, 재해 복구의 핵심입니다!"

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐ (고급)
**학습 시간**: 120분
**대상**: 백엔드 개발자, 데브옵스 엔지니어, DBA
**중요도**: 🔥🔥🔥🔥🔥 (매우 높음)

이전: [Chapter 29. pg_dump와 pg_restore](29-pg_dump와-pg_restore.md)
다음: [Chapter 31. 사용자와 권한](../part09-보안/31-사용자와-권한.md)
