# PostgreSQL 완전 정복 - 기초부터 실전까지

**대상 독자**: 백엔드 개발자
**목표**: PostgreSQL 기초부터 실전 운영까지 완전 마스터
**분량**: 책 한 권 분량

---

## 목차

### Part 1: PostgreSQL 기초
- [Chapter 1. PostgreSQL 소개](part01-기초/01-postgresql-소개.md)
- [Chapter 2. 설치와 환경설정](part01-기초/02-설치와-환경설정.md)
- [Chapter 3. psql 기본 사용법](part01-기초/03-psql-기본사용법.md)

### Part 2: SQL 기초
- [Chapter 4. 데이터베이스와 테이블](part02-sql기초/04-데이터베이스와-테이블.md)
- [Chapter 5. 데이터 타입](part02-sql기초/05-데이터-타입.md)
- [Chapter 6. CRUD 연산](part02-sql기초/06-CRUD-연산.md)
- [Chapter 7. 조건과 필터링](part02-sql기초/07-조건과-필터링.md)

### Part 3: SQL 심화
- [Chapter 8. JOIN](part03-sql심화/08-JOIN.md)
- [Chapter 9. 집계 함수와 GROUP BY](part03-sql심화/09-집계함수와-GROUPBY.md)
- [Chapter 10. 서브쿼리](part03-sql심화/10-서브쿼리.md)
- [Chapter 11. Window Function](part03-sql심화/11-window-function.md)
- [Chapter 12. CTE (Common Table Expression)](part03-sql심화/12-CTE.md)

### Part 4: 제약조건과 인덱스
- [Chapter 13. 제약조건 (Constraints)](part04-제약조건과-인덱스/13-제약조건.md)
- [Chapter 14. 인덱스 기초](part04-제약조건과-인덱스/14-인덱스-기초.md)
- [Chapter 15. 인덱스 최적화](part04-제약조건과-인덱스/15-인덱스-최적화.md)

### Part 5: 트랜잭션과 동시성
- [Chapter 16. 트랜잭션 기초](part05-트랜잭션과-동시성/16-트랜잭션-기초.md)
- [Chapter 17. 격리 수준 (Isolation Level)](part05-트랜잭션과-동시성/17-격리수준.md)
- [Chapter 18. 락(Lock)과 동시성 제어](part05-트랜잭션과-동시성/18-락과-동시성제어.md)

### Part 6: 고급 기능
- [Chapter 19. View와 Materialized View](part06-고급기능/19-view.md)
- [Chapter 20. Trigger](part06-고급기능/20-trigger.md)
- [Chapter 21. Stored Procedure와 Function](part06-고급기능/21-stored-procedure.md)
- [Chapter 22. Full Text Search](part06-고급기능/22-full-text-search.md)
- [Chapter 23. JSON과 JSONB](part06-고급기능/23-JSON.md)

### Part 7: 성능 최적화
- [Chapter 24. EXPLAIN과 쿼리 분석](part07-성능최적화/24-EXPLAIN.md)
- [Chapter 25. 쿼리 최적화](part07-성능최적화/25-쿼리-최적화.md)
- [Chapter 26. 파티셔닝](part07-성능최적화/26-파티셔닝.md)
- [Chapter 27. 커넥션 풀링](part07-성능최적화/27-커넥션-풀링.md)

### Part 8: 백업과 복구
- [Chapter 28. 백업 전략](part08-백업과-복구/28-백업-전략.md)
- [Chapter 29. pg_dump와 pg_restore](part08-백업과-복구/29-pg_dump.md)
- [Chapter 30. PITR (Point-in-Time Recovery)](part08-백업과-복구/30-PITR.md)

### Part 9: 보안
- [Chapter 31. 사용자와 권한 관리](part09-보안/31-사용자와-권한.md)
- [Chapter 32. 인증과 암호화](part09-보안/32-인증과-암호화.md)
- [Chapter 33. SQL Injection 방어](part09-보안/33-SQL-injection-방어.md)

### Part 10: 복제와 고가용성
- [Chapter 34. Replication 기초](part10-복제와-고가용성/34-replication-기초.md)
- [Chapter 35. Streaming Replication](part10-복제와-고가용성/35-streaming-replication.md)
- [Chapter 36. Failover와 고가용성](part10-복제와-고가용성/36-failover.md)

### Part 11: 모니터링과 튜닝
- [Chapter 37. 시스템 카탈로그](part11-모니터링과-튜닝/37-시스템-카탈로그.md)
- [Chapter 38. pg_stat 뷰](part11-모니터링과-튜닝/38-pg_stat.md)
- [Chapter 39. Slow Query 분석](part11-모니터링과-튜닝/39-slow-query.md)
- [Chapter 40. VACUUM과 ANALYZE](part11-모니터링과-튜닝/40-vacuum-analyze.md)

### Part 12: 실전 애플리케이션
- [Chapter 41. Node.js 연동 (node-postgres)](part12-실전/41-nodejs-연동.md)
- [Chapter 42. ORM (Sequelize, Prisma)](part12-실전/42-ORM.md)
- [Chapter 43. 마이그레이션 관리](part12-실전/43-마이그레이션.md)
- [Chapter 44. Docker로 PostgreSQL 운영](part12-실전/44-docker.md)
- [Chapter 45. AWS RDS PostgreSQL](part12-실전/45-aws-rds.md)

### Part 13: 부록
- [부록 A. SQL 치트시트](part13-부록/부록A-SQL-치트시트.md)
- [부록 B. psql 명령어 치트시트](part13-부록/부록B-psql-치트시트.md)
- [부록 C. 트러블슈팅 가이드](part13-부록/부록C-트러블슈팅.md)
- [부록 D. 설정 파라미터 가이드](part13-부록/부록D-설정-파라미터.md)
- [부록 E. 유용한 확장 기능](part13-부록/부록E-확장기능.md)

---

## 학습 로드맵

### 입문 (1주)
- Part 1-2: PostgreSQL 기초, SQL 기초
- 설치, psql, 기본 CRUD

### 초급 (1-2주)
- Part 3-4: SQL 심화, 제약조건과 인덱스
- JOIN, 집계, 인덱스

### 중급 (1-2주)
- Part 5-6: 트랜잭션, 고급 기능
- 트랜잭션, View, Trigger, JSON

### 고급 (2주)
- Part 7-8: 성능 최적화, 백업/복구
- EXPLAIN, 쿼리 최적화, 백업 전략

### 운영 (1-2주)
- Part 9-11: 보안, 복제, 모니터링
- 권한 관리, Replication, 모니터링

### 실전 (1주)
- Part 12: 실전 애플리케이션
- Node.js 연동, ORM, Docker, AWS RDS

---

## 권장 학습 순서

### 1단계: 필수 기초 (반드시)
```
Ch 1-7: PostgreSQL 기초 + SQL 기초
→ 설치부터 기본 CRUD까지
```

### 2단계: SQL 마스터 (중요)
```
Ch 8-12: SQL 심화
→ JOIN, 집계, Window Function, CTE
```

### 3단계: 성능 (실무 필수)
```
Ch 13-15: 인덱스
Ch 24-25: EXPLAIN, 쿼리 최적화
→ 느린 쿼리 해결 능력
```

### 4단계: 트랜잭션과 동시성
```
Ch 16-18: 트랜잭션, 격리수준, 락
→ 동시성 문제 해결
```

### 5단계: 실전 연동
```
Ch 41-45: Node.js, ORM, Docker, AWS
→ 실무 적용
```

### 6단계: 운영 (선택)
```
Ch 28-40: 백업, 보안, 복제, 모니터링
→ 운영 환경 관리
```

---

## 실습 환경

### 로컬 설치
```bash
# macOS
brew install postgresql@15
brew services start postgresql@15

# Ubuntu
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
```

### Docker 사용
```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mysecret \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15
```

### 클라우드 (AWS RDS)
```
AWS Console → RDS → Create Database
→ PostgreSQL 15.x
→ Free tier (테스트용)
```

---

## 이 문서의 특징

### ✅ 실무 중심
- 실제 사용하는 쿼리 패턴
- 성능 최적화 실전 팁
- 트러블슈팅 케이스

### ✅ 비유와 예시
- 복잡한 개념을 쉬운 비유로
- 풍부한 코드 예제
- 실행 가능한 샘플 쿼리

### ✅ 단계적 학습
- 기초부터 고급까지
- 각 챕터는 독립적
- 필요한 부분만 선택 학습 가능

### ✅ 최신 버전 기반
- PostgreSQL 15.x+
- 최신 기능 포함
- Best Practice

---

## 사전 지식

### 필수
- 기본 SQL 문법 (SELECT, INSERT 등)
- 터미널 사용법
- 기본 프로그래밍 지식

### 권장
- Linux 기초
- Docker 기초
- 백엔드 개발 경험

---

## 학습 목표

### 이 과정을 마치면...

**SQL 마스터**
- 복잡한 JOIN 쿼리 작성
- Window Function 활용
- CTE로 가독성 높은 쿼리

**성능 최적화**
- EXPLAIN으로 쿼리 분석
- 적절한 인덱스 설계
- 느린 쿼리 해결

**트랜잭션 이해**
- ACID 속성 이해
- 격리 수준 선택
- 동시성 문제 해결

**실전 운영**
- Node.js 연동
- ORM 활용
- Docker/AWS 배포

**문제 해결**
- 성능 문제 진단
- 데이터 무결성 보장
- 장애 대응

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**대상**: 백엔드 개발자
**난이도**: 입문 → 고급
**예상 학습 기간**: 6-8주
