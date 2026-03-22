# Chapter 29. pg_dump와 pg_restore

## 🎯 이 챕터의 목표
- **pg_dump**로 논리적 백업 생성하기
- **백업 포맷** 비교하고 적절히 선택하기
- **pg_restore**로 백업 복원하기
- **선택적 백업/복원** (특정 테이블, 스키마)
- **대용량 DB 백업** 최적화 기법
- **실전 시나리오** 완벽 대응하기

---

## 1. pg_dump란? (비유로 이해하기)

### 📚 도서관을 복사하는 방법에 비유하면

```
pg_dump = 도서관의 모든 책을 글로 옮겨 적기
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📖 원본 데이터베이스 (도서관)
   ├─ users 테이블 (소설 코너)
   ├─ orders 테이블 (역사 코너)
   └─ products 테이블 (과학 코너)

   ↓ pg_dump

📝 백업 파일 (필사본)
   ├─ CREATE TABLE users (...);
   ├─ INSERT INTO users VALUES (...);
   ├─ CREATE INDEX idx_users_email ...;
   └─ ... (모든 내용을 SQL 문으로)

장점:
✅ 사람이 읽을 수 있음 (SQL)
✅ 다른 버전에서도 사용 가능
✅ 특정 부분만 뽑아낼 수 있음

단점:
❌ 대용량은 시간이 오래 걸림
❌ 백업 중에도 DB가 변경될 수 있음 (하지만 일관성은 보장!)
```

---

## 2. pg_dump 기본 사용법

### 🔧 가장 간단한 백업

#### 전체 데이터베이스 백업
```bash
# 기본 형식 (plain text)
pg_dump mydb > mydb_backup.sql

# 또는
pg_dump -U postgres mydb > mydb_backup.sql

# 결과 확인
ls -lh mydb_backup.sql
# -rw-r--r-- 1 user user 123M Jan 15 10:30 mydb_backup.sql

# 백업 내용 확인 (처음 20줄)
head -20 mydb_backup.sql
```

#### 백업 파일 내용 예시
```sql
--
-- PostgreSQL database dump
--

-- Dumped from database version 14.5
-- Dumped by pg_dump version 14.5

SET statement_timeout = 0;
SET lock_timeout = 0;
SET client_encoding = 'UTF8';

--
-- Name: users; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.users (
    id integer NOT NULL,
    name character varying(100),
    email character varying(100),
    created_at timestamp with time zone DEFAULT now()
);

--
-- Data for Name: users; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.users (id, name, email, created_at) FROM stdin;
1	Alice	alice@example.com	2024-01-01 10:00:00+00
2	Bob	bob@example.com	2024-01-02 11:00:00+00
\.

--
-- Name: users_id_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.users_id_seq', 2, true);

--
-- Name: users users_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_pkey PRIMARY KEY (id);
```

### 📊 주요 옵션

```bash
# 사용자 지정
pg_dump -U postgres mydb > backup.sql

# 호스트 지정 (원격 서버)
pg_dump -h db.example.com -U postgres mydb > backup.sql

# 포트 지정
pg_dump -h localhost -p 5433 -U postgres mydb > backup.sql

# 비밀번호 입력 없이 (환경변수 사용)
export PGPASSWORD='mypassword'
pg_dump -U postgres mydb > backup.sql

# 또는 .pgpass 파일 사용
echo "localhost:5432:mydb:postgres:mypassword" > ~/.pgpass
chmod 600 ~/.pgpass
pg_dump -U postgres mydb > backup.sql
```

---

## 3. 백업 포맷 비교

### 📦 4가지 백업 포맷

| 포맷 | 옵션 | 확장자 | 특징 | 용도 |
|------|------|--------|------|------|
| **Plain** | `-Fp` (기본) | `.sql` | SQL 텍스트, 사람이 읽을 수 있음 | 작은 DB, 수동 편집 필요 시 |
| **Custom** | `-Fc` | `.dump` | 압축, 선택적 복원 가능 | **권장**, 대부분의 경우 |
| **Directory** | `-Fd` | `/dir` | 병렬 백업/복원, 가장 빠름 | 대용량 DB (>100GB) |
| **Tar** | `-Ft` | `.tar` | 압축, 선택적 복원, 이식성 | 아카이빙 용도 |

### 실습: 각 포맷으로 백업하기

#### 1️⃣ Plain Format (텍스트)
```bash
# 백업
pg_dump -Fp mydb > mydb_plain.sql

# 장점: 사람이 읽을 수 있음
head -50 mydb_plain.sql
# CREATE TABLE users (...);
# INSERT INTO users VALUES (...);

# 복원
psql mydb < mydb_plain.sql

# 또는 특정 테이블만 추출 가능
grep -A 20 "CREATE TABLE users" mydb_plain.sql > users_table.sql

# 단점: 압축 안 됨, 큰 파일
ls -lh mydb_plain.sql
# -rw-r--r-- 1 user user 1.2G Jan 15 10:30 mydb_plain.sql
```

#### 2️⃣ Custom Format (추천!) ⭐
```bash
# 백업 (압축 자동)
pg_dump -Fc mydb > mydb_custom.dump

# 또는 압축 레벨 지정 (0~9, 기본 -1)
pg_dump -Fc -Z 9 mydb > mydb_custom_max.dump

# 크기 비교
ls -lh mydb_*.dump
# -rw-r--r-- 1 user user 234M Jan 15 10:30 mydb_custom.dump
# -rw-r--r-- 1 user user 189M Jan 15 10:31 mydb_custom_max.dump

# 복원
pg_restore -d mydb mydb_custom.dump

# 특정 테이블만 복원
pg_restore -d mydb -t users mydb_custom.dump

# 백업 내용 확인 (목록만)
pg_restore -l mydb_custom.dump
#
# Archive created at 2024-01-15 10:30:00 KST
#     dbname: mydb
#     TOC Entries: 234
#     Compression: -1
#     Dump Version: 1.14-0
#
# 3; 2615 2200 SCHEMA - public postgres
# 215; 1259 16384 TABLE public users postgres
# 3456; 0 16384 TABLE DATA public users postgres
# 216; 1259 16385 TABLE public posts postgres
# ...
```

#### 3️⃣ Directory Format (대용량용)
```bash
# 백업 (4개 병렬 작업)
pg_dump -Fd -j 4 mydb -f mydb_backup_dir

# 디렉토리 구조 확인
ls -lh mydb_backup_dir/
# total 234M
# -rw-r--r-- 1 user user 1.2K Jan 15 10:30 toc.dat
# -rw-r--r-- 1 user user  56M Jan 15 10:30 3456.dat.gz
# -rw-r--r-- 1 user user  78M Jan 15 10:30 3457.dat.gz
# -rw-r--r-- 1 user user  45M Jan 15 10:30 3458.dat.gz
# -rw-r--r-- 1 user user  55M Jan 15 10:30 3459.dat.gz

# 복원 (4개 병렬 작업)
pg_restore -d mydb -j 4 mydb_backup_dir

# 장점: 매우 빠름! (병렬 처리)
# 10GB DB 백업: 단일 작업 30분 → 4개 병렬 8분
```

#### 4️⃣ Tar Format
```bash
# 백업
pg_dump -Ft mydb > mydb_tar.tar

# 압축 (외부 도구 사용)
pg_dump -Ft mydb | gzip > mydb_tar.tar.gz

# 복원
pg_restore -d mydb mydb_tar.tar

# 크기 비교
ls -lh
# -rw-r--r-- 1 user user 567M Jan 15 10:30 mydb_tar.tar
# -rw-r--r-- 1 user user 198M Jan 15 10:31 mydb_tar.tar.gz
```

### 포맷 선택 가이드

```
언제 어떤 포맷을 사용할까?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📝 Plain (-Fp):
   ✅ DB 크기 < 1GB
   ✅ SQL 직접 수정 필요
   ✅ 버전 관리 (Git)에 저장

   예: 개발용 샘플 데이터, 스키마 공유

🗜️ Custom (-Fc): ⭐ 기본 선택
   ✅ DB 크기 < 100GB
   ✅ 선택적 복원 필요
   ✅ 압축 필요

   예: 대부분의 프로덕션 백업

📂 Directory (-Fd):
   ✅ DB 크기 > 100GB
   ✅ 빠른 백업/복원 필수
   ✅ 병렬 처리 가능한 환경

   예: 대용량 프로덕션 DB

📦 Tar (-Ft):
   ✅ 이식성 중요
   ✅ 외부 압축 도구 사용

   예: 장기 아카이빙, 다른 시스템으로 전송
```

---

## 4. 선택적 백업

### 🎯 특정 테이블만 백업

```bash
# 단일 테이블
pg_dump -U postgres -t users mydb > users_backup.sql

# 여러 테이블
pg_dump -U postgres -t users -t posts -t comments mydb > tables_backup.sql

# 패턴 매칭 (와일드카드)
pg_dump -U postgres -t 'log_*' mydb > logs_backup.sql
# log_2024_01, log_2024_02, ... 모두 백업

# 특정 테이블 제외
pg_dump -U postgres -T 'temp_*' mydb > backup_no_temp.sql
# temp_로 시작하는 테이블 제외
```

### 📋 스키마별 백업

```bash
# 특정 스키마만
pg_dump -U postgres -n public mydb > public_schema.sql

# 여러 스키마
pg_dump -U postgres -n public -n analytics mydb > schemas_backup.sql

# 스키마 제외
pg_dump -U postgres -N temp_schema mydb > backup_no_temp_schema.sql
```

### 🏗️ 스키마와 데이터 분리

```bash
# 1. 스키마만 백업 (테이블 구조, 인덱스, 제약조건)
pg_dump -U postgres --schema-only mydb > schema_only.sql

# schema_only.sql 내용:
# CREATE TABLE users (...);
# CREATE INDEX idx_users_email ON users(email);
# ALTER TABLE users ADD CONSTRAINT users_pkey PRIMARY KEY (id);
# (INSERT 문 없음)

# 2. 데이터만 백업
pg_dump -U postgres --data-only mydb > data_only.sql

# data_only.sql 내용:
# COPY users (id, name, email) FROM stdin;
# 1	Alice	alice@example.com
# 2	Bob	bob@example.com
# \.
# (CREATE 문 없음)

# 활용: 테스트 환경 구축
# 1단계: 운영 스키마 복사
pg_dump -U postgres --schema-only prod_db > schema.sql
psql -U postgres test_db < schema.sql

# 2단계: 샘플 데이터 삽입 (직접 작성 또는 일부만)
pg_dump -U postgres --data-only -t users prod_db | head -1000 > sample_data.sql
psql -U postgres test_db < sample_data.sql
```

### 🔐 권한과 소유자 정보

```bash
# 권한 포함 (GRANT, REVOKE)
pg_dump -U postgres --no-acl mydb > backup_no_acl.sql
# ACL (Access Control List) 제외

# 소유자 정보 제외
pg_dump -U postgres --no-owner mydb > backup_no_owner.sql
# CREATE TABLE 문에 OWNER 지정 안 함

# 둘 다 제외 (다른 서버로 이전 시 유용)
pg_dump -U postgres --no-acl --no-owner mydb > backup_portable.sql
```

---

## 5. pg_restore 완벽 가이드

### 🔄 기본 복원

```bash
# Custom 포맷 복원
pg_restore -d mydb mydb_backup.dump

# 또는 새 DB에 복원
createdb new_db
pg_restore -d new_db mydb_backup.dump

# 에러 무시하고 계속 (이미 존재하는 객체 건너뛰기)
pg_restore -d mydb --if-exists mydb_backup.dump

# 깨끗하게 복원 (기존 객체 삭제 후)
pg_restore -d mydb --clean --if-exists mydb_backup.dump
```

### 🎛️ 고급 복원 옵션

#### 특정 테이블만 복원
```bash
# 1. 백업 내용 확인
pg_restore -l mydb_backup.dump | grep "TABLE DATA"
# 3456; 0 16384 TABLE DATA public users postgres
# 3457; 0 16385 TABLE DATA public posts postgres
# 3458; 0 16386 TABLE DATA public comments postgres

# 2. users 테이블만 복원
pg_restore -d mydb -t users mydb_backup.dump

# 3. 여러 테이블 복원
pg_restore -d mydb -t users -t posts mydb_backup.dump
```

#### 스키마만 복원 (데이터 제외)
```bash
# 테이블 구조, 인덱스, 제약조건만
pg_restore -d mydb --schema-only mydb_backup.dump

# 사용 예: 새 환경에 테이블 구조만 복사
createdb dev_db
pg_restore -d dev_db --schema-only prod_backup.dump
```

#### 데이터만 복원 (스키마 제외)
```bash
# 데이터만 (INSERT 또는 COPY)
pg_restore -d mydb --data-only mydb_backup.dump

# 사용 예: 기존 테이블에 데이터 추가
# (주의: 기본 키 충돌 가능성!)
```

#### 병렬 복원 (성능 향상)
```bash
# 4개 작업 병렬 실행
pg_restore -d mydb -j 4 mydb_backup.dump

# 디렉토리 포맷에서 병렬 복원
pg_restore -d mydb -j 8 mydb_backup_dir/

# 성능 비교:
# 10GB DB 복원:
#   단일 작업: 25분
#   -j 4:      8분
#   -j 8:      5분
```

#### 선택적 복원 (목록 파일 사용)
```bash
# 1. 전체 목록 생성
pg_restore -l mydb_backup.dump > backup_toc.list

# 2. 목록 편집 (불필요한 줄 주석 처리)
nano backup_toc.list

# Before:
# 3456; 0 16384 TABLE DATA public users postgres
# 3457; 0 16385 TABLE DATA public temp_data postgres
# 3458; 0 16386 TABLE DATA public posts postgres

# After:
# 3456; 0 16384 TABLE DATA public users postgres
; 3457; 0 16385 TABLE DATA public temp_data postgres  <- 주석 처리
# 3458; 0 16386 TABLE DATA public posts postgres

# 3. 편집된 목록으로 복원
pg_restore -d mydb -L backup_toc.list mydb_backup.dump
# temp_data 테이블은 복원되지 않음
```

---

## 6. 대용량 DB 백업 최적화

### ⚡ 성능 향상 기법

#### 1️⃣ 압축 레벨 조정
```bash
# 압축 없음 (가장 빠름, 큰 파일)
pg_dump -Fc -Z 0 mydb > backup_no_compress.dump

# 기본 압축 (균형)
pg_dump -Fc mydb > backup_default.dump

# 최대 압축 (느림, 작은 파일)
pg_dump -Fc -Z 9 mydb > backup_max_compress.dump

# 성능 비교 (100GB DB 기준):
# -Z 0: 15분, 100GB
# -Z 6: 30분, 25GB (기본값)
# -Z 9: 45분, 22GB
```

#### 2️⃣ 병렬 백업 (Directory 포맷)
```bash
# CPU 코어 수에 맞춰 병렬 작업
pg_dump -Fd -j 8 mydb -f mydb_backup_dir

# 모니터링 (별도 터미널)
watch -n 1 'ls -lh mydb_backup_dir/'

# 성능 비교 (500GB DB 기준):
# 단일: 4시간
# -j 4:  1.5시간
# -j 8:  50분
```

#### 3️⃣ 네트워크 백업 최적화
```bash
# 원격 서버에서 백업 후 로컬로 압축 전송
ssh user@db-server "pg_dump -Fc mydb" | gzip > backup.dump.gz

# 또는 파이프로 직접 저장
pg_dump -h remote-db -Fc mydb | \
  ssh backup-server "cat > /backups/mydb_$(date +%Y%m%d).dump"
```

#### 4️⃣ 백업 중 부하 감소
```bash
# nice로 우선순위 낮춤 (CPU 양보)
nice -n 19 pg_dump -Fc mydb > backup.dump

# ionice로 I/O 우선순위 낮춤 (디스크 양보)
ionice -c 3 pg_dump -Fc mydb > backup.dump

# 둘 다 적용 (프로덕션 영향 최소화)
nice -n 19 ionice -c 3 pg_dump -Fc mydb > backup.dump
```

---

## 7. 실전 시나리오

### 📝 시나리오 1: 프로덕션 DB를 개발 환경으로 복사

```bash
#!/bin/bash
# copy-prod-to-dev.sh

# 1. 운영 DB 백업 (Stand-by에서 백업하여 운영 부하 감소)
echo "🔄 운영 DB 백업 중..."
pg_dump -h standby-db.example.com -U postgres -Fc prod_db > prod_backup.dump

# 2. 민감한 데이터 제거 스크립트 준비
cat > sanitize.sql <<EOF
-- 개인정보 마스킹
UPDATE users SET
  email = 'user' || id || '@example.com',
  phone = '010-0000-' || LPAD(id::text, 4, '0'),
  address = '서울시 강남구';

-- 결제 정보 삭제
DELETE FROM payment_methods;

-- 실제 주문 내역 삭제 (최근 100건만 유지)
DELETE FROM orders WHERE id NOT IN (
  SELECT id FROM orders ORDER BY created_at DESC LIMIT 100
);
EOF

# 3. 개발 DB 초기화 및 복원
echo "🗑️  개발 DB 초기화 중..."
dropdb --if-exists dev_db
createdb dev_db

echo "📥 백업 복원 중..."
pg_restore -d dev_db prod_backup.dump

# 4. 데이터 마스킹
echo "🔒 민감 데이터 마스킹 중..."
psql -d dev_db -f sanitize.sql

# 5. 정리
rm prod_backup.dump sanitize.sql

echo "✅ 완료! 개발 환경 준비됨"
```

### 🔄 시나리오 2: 특정 시점으로 부분 복원

```bash
#!/bin/bash
# restore-partial-data.sh

# 상황: users 테이블을 실수로 삭제함
# 해결: 어제 백업에서 users 테이블만 복원

# 1. 백업 파일에서 users 테이블만 추출
pg_restore -d temp_restore \
  --schema-only -t users \
  yesterday_backup.dump

# 2. 데이터 복원 (기존 데이터와 충돌 방지)
pg_restore -d mydb \
  --data-only -t users \
  --disable-triggers \
  yesterday_backup.dump

# --disable-triggers: 외래 키 체크 일시 중지
```

### 🌍 시나리오 3: 다른 PostgreSQL 버전으로 마이그레이션

```bash
#!/bin/bash
# migrate-pg-version.sh

# PostgreSQL 12 → 14 마이그레이션

# 1. 구버전에서 백업 (논리적 백업 사용!)
pg_dump -h old-server -U postgres -Fc old_db > migration_backup.dump

# 2. 신버전 서버에 새 DB 생성
createdb -h new-server -U postgres new_db

# 3. 복원
pg_restore -h new-server -U postgres -d new_db migration_backup.dump

# 4. 통계 업데이트
psql -h new-server -U postgres -d new_db -c "ANALYZE;"

# 5. 인덱스 재구축 (선택사항, 성능 향상)
psql -h new-server -U postgres -d new_db -c "REINDEX DATABASE new_db;"

# 주의사항:
# - 물리적 백업(pg_basebackup)은 버전 간 호환 안 됨
# - 논리적 백업(pg_dump)만 사용!
# - 확장 기능 버전 호환성 확인 필요
```

### 🔀 시나리오 4: 여러 DB 통합

```bash
#!/bin/bash
# merge-databases.sh

# 상황: 3개의 지역별 DB를 1개로 통합

# 1. 각 DB 백업
pg_dump -h asia-db -U postgres asia_db > asia.dump
pg_dump -h europe-db -U postgres europe_db > europe.dump
pg_dump -h america-db -U postgres america_db > america.dump

# 2. 통합 DB 생성
createdb merged_db

# 3. 스키마 생성 (첫 번째 DB 것만)
pg_restore -d merged_db --schema-only asia.dump

# 4. 각 DB 데이터 복원 (ID 충돌 방지)
# asia: ID 1,000,000,000 부터
psql -d merged_db -c "
  ALTER SEQUENCE users_id_seq RESTART WITH 1000000000;
"
pg_restore -d merged_db --data-only asia.dump

# europe: ID 2,000,000,000 부터
psql -d merged_db -c "
  ALTER SEQUENCE users_id_seq RESTART WITH 2000000000;
"
pg_restore -d merged_db --data-only europe.dump

# america: ID 3,000,000,000 부터
psql -d merged_db -c "
  ALTER SEQUENCE users_id_seq RESTART WITH 3000000000;
"
pg_restore -d merged_db --data-only america.dump

echo "✅ DB 통합 완료"
```

### 📦 시나리오 5: 클라우드로 백업 자동화

```bash
#!/bin/bash
# auto-backup-to-cloud.sh

# 환경변수 설정
DB_NAME="prod_db"
BACKUP_DIR="/tmp/pg_backups"
S3_BUCKET="s3://mycompany-db-backups"
RETENTION_DAYS=30

# 날짜 기반 파일명
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.dump"

mkdir -p "$BACKUP_DIR"

# 1. 백업 생성 (압축 최대)
echo "🔄 백업 시작: $(date)"
pg_dump -U postgres -Fc -Z 9 "$DB_NAME" > "$BACKUP_FILE"

if [ $? -ne 0 ]; then
    echo "❌ 백업 실패!" >&2
    curl -X POST $SLACK_WEBHOOK_URL \
      -d '{"text":"⚠️ PostgreSQL 백업 실패!"}'
    exit 1
fi

BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
echo "✅ 백업 완료: $BACKUP_SIZE"

# 2. 백업 검증
echo "🔍 백업 검증 중..."
pg_restore -l "$BACKUP_FILE" > /dev/null 2>&1

if [ $? -ne 0 ]; then
    echo "❌ 백업 파일 손상!" >&2
    rm "$BACKUP_FILE"
    exit 1
fi

# 3. 암호화 (선택사항)
echo "🔐 암호화 중..."
openssl enc -aes-256-cbc -salt \
  -in "$BACKUP_FILE" \
  -out "$BACKUP_FILE.enc" \
  -pass pass:$BACKUP_ENCRYPTION_KEY

# 4. S3 업로드
echo "☁️  S3 업로드 중..."
aws s3 cp "$BACKUP_FILE.enc" \
  "$S3_BUCKET/daily/$(basename $BACKUP_FILE.enc)" \
  --storage-class STANDARD_IA

if [ $? -ne 0 ]; then
    echo "❌ S3 업로드 실패!" >&2
    exit 1
fi

# 5. 로컬 파일 삭제
rm "$BACKUP_FILE" "$BACKUP_FILE.enc"

# 6. S3에서 오래된 백업 삭제
echo "🗑️  오래된 백업 삭제 중..."
aws s3 ls "$S3_BUCKET/daily/" | \
  awk '{print $4}' | \
  head -n -$RETENTION_DAYS | \
  xargs -I {} aws s3 rm "$S3_BUCKET/daily/{}"

# 7. 성공 알림
echo "✅ 백업 완료: $(date)"
curl -X POST $SLACK_WEBHOOK_URL \
  -d "{\"text\":\"✅ PostgreSQL 백업 성공\n크기: $BACKUP_SIZE\n파일: $(basename $BACKUP_FILE)\"}"

# Crontab 설정:
# 0 2 * * * /usr/local/bin/auto-backup-to-cloud.sh >> /var/log/pg_backup.log 2>&1
```

---

## 8. pg_dumpall: 전체 클러스터 백업

### 🌐 언제 pg_dumpall을 사용하나?

```
pg_dump vs pg_dumpall
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

pg_dump:
  ├─ 단일 데이터베이스 백업
  ├─ 테이블, 인덱스, 데이터
  └─ 가장 일반적으로 사용

pg_dumpall:
  ├─ 전체 PostgreSQL 클러스터 백업
  ├─ 모든 데이터베이스
  ├─ 전역 객체 (역할, 테이블스페이스)
  └─ 서버 이전 시 사용
```

### 실습: pg_dumpall 사용하기

```bash
# 1. 전체 클러스터 백업
pg_dumpall -U postgres > cluster_backup.sql

# 2. 전역 객체만 백업 (역할, 권한)
pg_dumpall -U postgres --globals-only > globals.sql

# globals.sql 내용:
# CREATE ROLE alice LOGIN PASSWORD 'md5...';
# CREATE ROLE bob LOGIN PASSWORD 'md5...';
# GRANT admin TO alice;

# 3. 스키마만 백업 (모든 DB)
pg_dumpall -U postgres --schema-only > all_schemas.sql

# 4. 복원 (전체 클러스터)
psql -U postgres -f cluster_backup.sql
```

### 활용 사례: 서버 마이그레이션

```bash
#!/bin/bash
# migrate-server.sh

# 구 서버 → 신 서버 완전 이전

# 1. 구 서버에서 전체 백업
ssh old-server "pg_dumpall -U postgres" > full_cluster.sql

# 2. 신 서버에 PostgreSQL 설치
ssh new-server "sudo apt install postgresql-14"

# 3. 복원
cat full_cluster.sql | ssh new-server "psql -U postgres"

# 4. 검증
ssh new-server "psql -U postgres -c '\l'"
# 모든 DB가 복원되었는지 확인

echo "✅ 서버 마이그레이션 완료"
```

---

## 9. 트러블슈팅

### ❌ 자주 발생하는 오류와 해결책

#### 오류 1: 권한 부족
```bash
# 에러:
pg_dump: error: query failed: ERROR:  permission denied for table users

# 원인: 백업 사용자가 테이블 읽기 권한 없음

# 해결:
psql -U postgres -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO backup_user;"
psql -U postgres -c "GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO backup_user;"
```

#### 오류 2: 디스크 공간 부족
```bash
# 에러:
pg_dump: error: could not write to output file: No space left on device

# 원인: 백업 디스크 공간 부족

# 해결 1: 다른 디스크로 백업
df -h  # 디스크 공간 확인
pg_dump -Fc mydb > /mnt/external/backup.dump

# 해결 2: 압축률 높이기
pg_dump -Fc -Z 9 mydb > backup.dump

# 해결 3: 직접 클라우드로 스트리밍
pg_dump -Fc mydb | aws s3 cp - s3://bucket/backup.dump
```

#### 오류 3: 복원 시 외래 키 충돌
```bash
# 에러:
ERROR:  insert or update on table "posts" violates foreign key constraint
DETAIL:  Key (user_id)=(123) is not present in table "users".

# 원인: 복원 순서 문제 (posts → users 순서로 복원)

# 해결 1: --disable-triggers 사용 (주의!)
pg_restore -d mydb --disable-triggers backup.dump

# 해결 2: 제약조건 나중에 생성
# 백업 시:
pg_dump -Fc --section=pre-data mydb > schema.dump
pg_dump -Fc --section=data mydb > data.dump
pg_dump -Fc --section=post-data mydb > constraints.dump

# 복원 시:
pg_restore -d mydb schema.dump      # 1. 테이블 생성
pg_restore -d mydb data.dump        # 2. 데이터 삽입
pg_restore -d mydb constraints.dump # 3. 제약조건/인덱스 생성
```

#### 오류 4: 백업 파일 손상
```bash
# 에러:
pg_restore: error: input file appears to be a text format dump.
Please use psql.

# 원인: 백업 포맷 불일치

# 해결: 포맷 확인 후 적절한 도구 사용
file backup.dump
# backup.dump: PostgreSQL custom database dump - v1.14-0

# Custom 포맷 → pg_restore
pg_restore -d mydb backup.dump

# Plain 포맷 → psql
psql -d mydb < backup.sql
```

---

## 10. 성능 비교 및 권장사항

### 📊 백업 방법 벤치마크

```
테스트 환경:
  - DB 크기: 100GB
  - 테이블 수: 500개
  - CPU: 8코어
  - 디스크: SSD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

방법                         백업 시간    파일 크기    복원 시간
────────────────────────────────────────────────────────────────
pg_dump -Fp                  45분        100GB        35분
pg_dump -Fc                  38분        23GB         28분
pg_dump -Fc -Z9              52분        21GB         28분
pg_dump -Fd -j4              18분        24GB         12분
pg_dump -Fd -j8              12분        24GB         8분
pg_basebackup (물리적)       8분         100GB        3분
```

### 🎯 권장사항

```
DB 크기별 권장 방법
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 소규모 (< 10GB):
   방법: pg_dump -Fc
   이유: 간단, 압축, 선택적 복원 가능
   예시: pg_dump -Fc -Z6 mydb > backup.dump

📦 중규모 (10GB ~ 100GB):
   방법: pg_dump -Fc 또는 -Fd -j4
   이유: 압축과 속도 균형
   예시: pg_dump -Fc mydb > backup.dump

📦 대규모 (100GB ~ 1TB):
   방법: pg_dump -Fd -j8
   이유: 병렬 처리로 속도 향상
   예시: pg_dump -Fd -j8 mydb -f backup_dir

📦 초대규모 (> 1TB):
   방법: pg_basebackup (물리적 백업)
   이유: 가장 빠름, PITR 가능
   예시: pg_basebackup -D /backups/base -Fp -Xs -P
```

---

## 11. 실습 과제

### 🎯 과제 1: 백업 스크립트 작성
다음 요구사항을 만족하는 백업 스크립트를 작성하세요:

```bash
#!/bin/bash
# advanced-backup.sh

# 요구사항:
# 1. mydb 데이터베이스를 Custom 포맷으로 백업
# 2. 파일명: mydb_YYYYMMDD_HHMMSS.dump
# 3. /backups 디렉토리에 저장
# 4. 백업 후 검증 (pg_restore -l로 목록 확인)
# 5. 검증 성공 시 S3 업로드
# 6. 로컬은 3일치만 보관
# 7. 모든 작업 로그를 /var/log/pg_backup.log에 기록
# 8. 실패 시 이메일 알림

# 여기에 작성하세요
```

<details>
<summary>💡 해답 보기</summary>

```bash
#!/bin/bash
# advanced-backup.sh

set -e  # 에러 시 즉시 종료

# 설정
DB_NAME="mydb"
BACKUP_DIR="/backups"
LOG_FILE="/var/log/pg_backup.log"
S3_BUCKET="s3://mycompany-backups/postgres"
ALERT_EMAIL="admin@example.com"
RETENTION_DAYS=3

# 로그 함수
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# 에러 핸들러
error_exit() {
    log "❌ ERROR: $1"
    echo "PostgreSQL 백업 실패: $1" | mail -s "백업 실패 알림" "$ALERT_EMAIL"
    exit 1
}

# 1. 백업 파일명 생성
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump"

log "🔄 백업 시작: $BACKUP_FILE"

# 2. 백업 실행
pg_dump -U postgres -Fc "$DB_NAME" > "$BACKUP_FILE" || \
    error_exit "pg_dump 실패"

log "✅ 백업 파일 생성 완료: $(du -h $BACKUP_FILE | cut -f1)"

# 3. 백업 검증
log "🔍 백업 파일 검증 중..."
pg_restore -l "$BACKUP_FILE" > /dev/null 2>&1 || \
    error_exit "백업 파일 손상"

TABLES=$(pg_restore -l "$BACKUP_FILE" | grep "TABLE DATA" | wc -l)
log "✅ 검증 완료: $TABLES 개 테이블"

# 4. S3 업로드
log "☁️  S3 업로드 중..."
aws s3 cp "$BACKUP_FILE" "$S3_BUCKET/$(basename $BACKUP_FILE)" || \
    error_exit "S3 업로드 실패"

log "✅ S3 업로드 완료"

# 5. 오래된 로컬 백업 삭제
log "🗑️  오래된 백업 삭제 중..."
find "$BACKUP_DIR" -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
log "✅ 정리 완료"

log "🎉 모든 작업 완료"

# Crontab:
# 0 2 * * * /usr/local/bin/advanced-backup.sh
```
</details>

### 🎯 과제 2: 선택적 복원 연습
다음 시나리오를 해결하세요:

```
상황:
  - 프로덕션 DB에서 users 테이블을 잘못 수정함
  - 어제 02:00 백업이 있음: prod_20240114_020000.dump
  - users 테이블만 복원해야 함 (다른 테이블은 그대로)

작업:
  1. 백업 파일에서 users 테이블 정보 확인
  2. 임시 DB에 users 테이블만 복원
  3. 데이터 비교 (어제 vs 오늘)
  4. 잘못된 데이터만 수정하는 SQL 작성
```

### 🎯 과제 3: 성능 최적화
500GB DB를 최대한 빠르게 백업하는 방법을 설계하세요:

```
제약 조건:
  - CPU: 16코어
  - 메모리: 64GB
  - 디스크: NVMe SSD
  - 네트워크: 10Gbps
  - 백업 윈도우: 1시간 이내

고려사항:
  1. 백업 포맷 선택
  2. 병렬 작업 수
  3. 압축 레벨
  4. I/O 우선순위
  5. 네트워크 활용

명령어 작성:
pg_dump ??? mydb ???
```

---

## 12. 다음 단계

pg_dump와 pg_restore를 마스터했으니, 이제 더 고급 복구 기법을 배울 차례입니다!

### 다음 챕터 미리보기
```
Chapter 30: PITR (Point-In-Time Recovery)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 배울 내용:
   ├─ WAL (Write-Ahead Log) 이해하기
   ├─ WAL 아카이빙 설정
   ├─ pg_basebackup으로 베이스 백업
   ├─ 특정 시점으로 복구하기
   ├─ 연속 아카이빙 구축
   └─ 재해 복구 시나리오

💡 실습:
   ├─ 베이스 백업 + WAL 아카이빙
   ├─ "10분 전"으로 DB 되돌리기
   ├─ 트랜잭션 ID로 복구
   ├─ 타임라인 관리
   └─ 자동 복구 시스템 구축

🎯 PITR을 배우면:
   - RPO를 거의 0에 가깝게!
   - 실수한 시점 직전으로 복구
   - 금융권 수준의 백업 시스템
```

---

## 📖 정리

이번 챕터에서 배운 내용:

| 개념 | 핵심 내용 |
|------|----------|
| **pg_dump** | 논리적 백업 도구, SQL 문으로 저장 |
| **백업 포맷** | Plain, Custom (권장), Directory (대용량), Tar |
| **pg_restore** | Custom/Directory/Tar 포맷 복원 |
| **선택적 백업** | -t (테이블), -n (스키마), --schema-only |
| **병렬 처리** | -j 옵션으로 속도 향상 (Directory 포맷) |
| **pg_dumpall** | 전체 클러스터 백업 (모든 DB + 전역 객체) |

### 핵심 명령어
```bash
# 기본 백업 (Custom 포맷)
pg_dump -Fc mydb > backup.dump

# 병렬 백업 (대용량)
pg_dump -Fd -j 8 mydb -f backup_dir

# 특정 테이블만
pg_dump -t users mydb > users.sql

# 복원
pg_restore -d mydb backup.dump

# 선택적 복원
pg_restore -d mydb -t users backup.dump

# 병렬 복원
pg_restore -d mydb -j 4 backup_dir
```

### 선택 가이드
```
💡 어떤 포맷을 선택할까?

일반적인 경우 → -Fc (Custom)
대용량 (>100GB) → -Fd -j8 (Directory + 병렬)
텍스트 편집 필요 → -Fp (Plain)
장기 보관 → -Ft (Tar)
```

**기억하세요**: "pg_dump는 논리적 백업의 완벽한 도구입니다. 하지만 대용량 DB나 RPO가 중요한 경우, 다음 챕터의 PITR을 고려하세요!"

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐ (중급)
**학습 시간**: 90분
**대상**: 백엔드 개발자, 데브옵스 엔지니어, DBA
**중요도**: 🔥🔥🔥🔥🔥 (매우 높음)

이전: [Chapter 28. 백업 전략](28-백업-전략.md)
다음: [Chapter 30. PITR (Point-In-Time Recovery)](30-pitr.md)
