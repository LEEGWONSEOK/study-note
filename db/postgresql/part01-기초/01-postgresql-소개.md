# Chapter 1. PostgreSQL 소개

## 1.1 PostgreSQL이란?

### 정의

```
PostgreSQL:
오픈소스 관계형 데이터베이스 관리 시스템 (RDBMS)

별명: Postgres
발음: 포스트그레스큐엘
역사: 1986년 UC 버클리에서 시작
라이선스: PostgreSQL License (BSD 유사)
```

### 비유로 이해하기

```
데이터베이스 = 도서관

┌─────────────────────────────────┐
│ 도서관 (Database)               │
│                                 │
│ ├─ 책장 (Table)                │
│ │  ├─ 소설책 (Row)              │
│ │  ├─ 역사책 (Row)              │
│ │  └─ 과학책 (Row)              │
│ │                                │
│ ├─ 카탈로그 (Index)            │
│ │  → 빠른 검색                  │
│ │                                │
│ └─ 사서 (PostgreSQL)           │
│    → 데이터 관리                │
└─────────────────────────────────┘
```

---

## 1.2 왜 PostgreSQL인가?

### 주요 특징

```
1. 오픈소스
   - 무료
   - 활발한 커뮤니티
   - 상업적 사용 가능

2. ACID 보장
   - Atomicity: 원자성
   - Consistency: 일관성
   - Isolation: 격리성
   - Durability: 지속성

3. 강력한 기능
   - 복잡한 쿼리 지원
   - Full Text Search
   - JSON/JSONB 지원
   - GIS (PostGIS)
   - Array, UUID 등 풍부한 데이터 타입

4. 확장성
   - 수백 GB ~ TB 데이터
   - Replication
   - Partitioning
   - 플러그인 시스템

5. 표준 준수
   - SQL:2016 표준
   - ANSI SQL 호환
```

---

## 1.3 PostgreSQL vs MySQL

### 비교표

```
┌─────────────┬──────────────┬──────────────┐
│  항목       │ PostgreSQL   │ MySQL        │
├─────────────┼──────────────┼──────────────┤
│ 라이선스    │ PostgreSQL   │ GPL/상용     │
│ ACID        │ 완벽 지원    │ InnoDB만     │
│ 트랜잭션    │ 강력함       │ 기본         │
│ 동시성      │ MVCC 우수    │ 보통         │
│ JSON        │ JSONB (빠름) │ JSON (느림)  │
│ Full Text   │ 내장         │ 제한적       │
│ Window Func │ 완벽 지원    │ 8.0+         │
│ CTE         │ 완벽 지원    │ 8.0+         │
│ 복제        │ 논리+물리    │ 물리         │
│ GIS         │ PostGIS      │ 기본         │
│ 속도        │ 복잡한 쿼리  │ 단순 쿼리    │
│ 메모리      │ 더 많이 사용 │ 적게 사용    │
└─────────────┴──────────────┴──────────────┘
```

### 언제 PostgreSQL?

```
✅ PostgreSQL 선택
- 복잡한 쿼리 (JOIN, 서브쿼리)
- 트랜잭션 중요
- JSON 데이터 많이 사용
- Full Text Search
- GIS 데이터 (지도, 위치)
- 데이터 무결성 중요

✅ MySQL 선택
- 단순한 Read 위주
- 빠른 성능 (단순 쿼리)
- 메모리 제약
- 레거시 시스템
```

---

## 1.4 PostgreSQL 아키텍처

### 프로세스 구조

```
┌─────────────────────────────────┐
│       postmaster (main)         │
│    포트 5432 리스닝             │
└────────────┬────────────────────┘
             │
   ┌─────────┼──────────────┐
   │         │              │
   ▼         ▼              ▼
┌──────┐ ┌──────┐       ┌──────┐
│Client│ │Client│  ...  │Client│
│Backend││Backend│      │Backend│
└──────┘ └──────┘       └──────┘
   │         │              │
   └─────────┼──────────────┘
             │
   ┌─────────▼──────────────┐
   │                        │
   │  Shared Memory         │
   │  - Shared Buffers      │
   │  - WAL Buffers         │
   │  - Lock Tables         │
   │                        │
   └────────────────────────┘
             │
   ┌─────────▼──────────────┐
   │  Background Workers    │
   │  - WAL Writer          │
   │  - Checkpointer        │
   │  - Autovacuum          │
   │  - Stats Collector     │
   └────────────────────────┘
             │
   ┌─────────▼──────────────┐
   │    Disk Storage        │
   │  - Data Files          │
   │  - WAL Files           │
   │  - Config Files        │
   └────────────────────────┘
```

### 주요 구성 요소

```
1. Postmaster
   - 메인 프로세스
   - 클라이언트 연결 관리
   - 백그라운드 워커 생성

2. Backend Process
   - 클라이언트당 1개 프로세스
   - SQL 실행
   - 트랜잭션 관리

3. Shared Memory
   - Shared Buffers: 데이터 캐시
   - WAL Buffers: 로그 버퍼
   - Lock Tables: 락 정보

4. Background Workers
   - WAL Writer: 로그 쓰기
   - Checkpointer: 체크포인트
   - Autovacuum: 자동 정리
   - Stats Collector: 통계 수집

5. Storage
   - Data Files: 실제 데이터
   - WAL (Write-Ahead Log): 로그
   - Config: 설정 파일
```

---

## 1.5 데이터베이스 개념

### 계층 구조

```
PostgreSQL Cluster (설치)
│
├─ Database 1
│  ├─ Schema: public
│  │  ├─ Table: users
│  │  ├─ Table: posts
│  │  └─ Index, View, etc.
│  │
│  └─ Schema: private
│     └─ Table: logs
│
├─ Database 2
│  └─ Schema: public
│     └─ Table: products
│
└─ System Databases
   ├─ postgres (기본)
   ├─ template0
   └─ template1
```

### 용어 정리

```
Cluster:
- PostgreSQL 설치 인스턴스
- 여러 Database 포함

Database:
- 논리적 데이터 저장소
- 독립적 (DB간 쿼리 불가)

Schema:
- Database 내 네임스페이스
- 기본: public
- 여러 Schema 가능

Table:
- 실제 데이터 저장
- Row와 Column

Row (Tuple):
- 데이터 레코드

Column (Attribute):
- 데이터 필드
```

---

## 1.6 ACID 속성

### Atomicity (원자성)

```
모두 성공 또는 모두 실패

예시: 계좌 이체
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

→ 둘 다 성공 또는 둘 다 실패
→ 중간 상태 없음
```

### Consistency (일관성)

```
데이터베이스 규칙 유지

예시: 제약조건
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  age INT CHECK (age >= 0)
);

INSERT INTO users (email, age) VALUES ('test@example.com', -1);
-- ERROR: age >= 0 위반

→ 규칙을 위반하는 데이터는 저장 안 됨
```

### Isolation (격리성)

```
트랜잭션은 서로 독립적

예시:
트랜잭션 A: 사용자 1의 잔액 조회
트랜잭션 B: 사용자 1의 잔액 변경

→ A는 B의 중간 결과를 보지 못함
→ 격리 수준에 따라 다름 (후술)
```

### Durability (지속성)

```
커밋된 데이터는 영구 보존

예시:
BEGIN;
  INSERT INTO logs VALUES ('중요 로그');
COMMIT;

[서버 다운]
[재시작]

→ '중요 로그' 그대로 남아있음
→ WAL (Write-Ahead Log) 덕분
```

---

## 1.7 PostgreSQL 버전

### 버전 정책

```
PostgreSQL 15.2
           ││└─ Patch (버그 수정)
           │└── Minor (기능 추가, 하위 호환)
           └─── Major (큰 변경)

릴리스 주기:
- Major: 1년마다
- Minor: 수시 (보안/버그)
- 지원 기간: 5년
```

### 주요 버전별 특징

```
PostgreSQL 15 (2022):
- MERGE 문 추가
- Public schema 권한 변경
- 성능 향상 (정렬, JOIN)

PostgreSQL 14 (2021):
- 성능 대폭 향상
- JSON 구독
- 파이프라인 모드

PostgreSQL 13 (2020):
- B-tree 인덱스 최적화
- Incremental Sort
- Partitioning 개선

PostgreSQL 12 (2019):
- CTE 최적화
- JSON Path
- Generated Column

권장: PostgreSQL 15+
최소: PostgreSQL 12+
```

---

## 1.8 사용 사례

### 실제 사용 기업

```
대기업:
- Apple (iCloud)
- Instagram
- Spotify
- Reddit
- Twitch

스타트업:
- 대부분의 모던 스타트업
- SaaS 기업들

공공기관:
- 미국 정부
- 유럽 정부 기관
```

### 적합한 사용 사례

```
✅ 웹 애플리케이션
- 사용자 관리
- 콘텐츠 관리
- 전자상거래

✅ 분석 시스템
- 복잡한 쿼리
- Reporting
- BI 도구

✅ 지리 정보 시스템 (GIS)
- PostGIS 확장
- 지도 서비스
- 위치 기반 서비스

✅ 금융 시스템
- 트랜잭션 중요
- ACID 보장
- 감사 추적

✅ 로그 분석
- JSON 데이터
- Full Text Search
- 시계열 데이터

❌ 부적합한 경우
- 초고속 단순 쿼리만 (Redis)
- 비정형 문서 (MongoDB)
- 실시간 스트리밍 (Kafka)
```

---

## 1.9 생태계

### 확장 기능 (Extensions)

```
PostGIS:
- 지리 정보 시스템
- 지도, 위치 데이터

pg_stat_statements:
- 쿼리 통계
- 성능 분석

pgcrypto:
- 암호화 함수

uuid-ossp:
- UUID 생성

hstore:
- Key-Value 저장

pg_trgm:
- Full Text Search 향상
```

### 도구

```
관리 도구:
- pgAdmin: GUI 관리 도구
- DBeaver: 멀티 DB 클라이언트
- DataGrip: JetBrains IDE

모니터링:
- pg_stat_monitor
- pgBadger
- Prometheus + Grafana

백업:
- pg_dump/pg_restore
- pgBackRest
- Barman

연결 풀링:
- PgBouncer
- Pgpool-II

고가용성:
- Patroni
- repmgr
- Stolon
```

---

## 1.10 학습 로드맵

### 단계별 학습

```
1주차: 기초
- PostgreSQL 설치
- psql 사용법
- 기본 SQL (SELECT, INSERT)

2주차: SQL 심화
- JOIN
- 집계 함수
- 서브쿼리

3주차: 고급 기능
- Window Function
- CTE
- JSON

4주차: 성능
- 인덱스
- EXPLAIN
- 쿼리 최적화

5주차: 트랜잭션
- ACID
- 격리 수준
- 락

6주차: 실전
- Node.js 연동
- ORM
- Docker 배포
```

---

## 핵심 요약

### PostgreSQL이란?
```
오픈소스 RDBMS
강력한 기능
ACID 보장
확장 가능
```

### 주요 특징
```
SQL 표준 준수
복잡한 쿼리 지원
JSON/JSONB
Full Text Search
GIS (PostGIS)
```

### vs MySQL
```
PostgreSQL: 복잡한 쿼리, 트랜잭션
MySQL: 단순 쿼리, 빠른 속도
```

### 사용 사례
```
웹 애플리케이션
분석 시스템
GIS
금융 시스템
```

---

## 다음 단계

PostgreSQL의 개념을 이해했으니, 이제 **설치와 환경설정**을 배워봅시다!

👉 [Chapter 2. 설치와 환경설정](02-설치와-환경설정.md)

---

## 참고 자료

### 공식 문서
- https://www.postgresql.org/docs/
- https://wiki.postgresql.org/

### 커뮤니티
- PostgreSQL 메일링 리스트
- Stack Overflow: [postgresql] 태그
- Reddit: r/PostgreSQL

### 블로그
- https://www.cybertec-postgresql.com/
- https://www.2ndquadrant.com/blog/

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐ 입문
**예상 학습 시간**: 1시간
**대상**: 모든 개발자