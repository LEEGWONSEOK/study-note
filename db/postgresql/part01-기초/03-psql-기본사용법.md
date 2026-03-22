# Chapter 3. psql 기본 사용법

## 3.1 psql이란?

### 정의

```
psql:
PostgreSQL 대화형 터미널

비유: 데이터베이스와의 대화
┌──────────────────────────────┐
│ 사용자 → psql → PostgreSQL   │
│                              │
│ 명령 입력 → 결과 출력        │
│ SQL 실행 → 데이터 조회       │
└──────────────────────────────┘
```

### 특징

```
✅ 대화형 인터페이스
✅ SQL 실행
✅ 메타 명령어 (\로 시작)
✅ 자동 완성 (Tab)
✅ 히스토리 (↑↓)
✅ 결과 포맷팅
```

---

## 3.2 psql 시작과 종료

### 기본 접속

```bash
# 1. 기본 접속 (현재 사용자, postgres DB)
psql

# 2. 사용자 지정
psql -U myuser

# 3. 데이터베이스 지정
psql -d mydb

# 4. 호스트 지정
psql -h localhost

# 5. 포트 지정
psql -p 5432

# 6. 전체 옵션
psql -U myuser -d mydb -h localhost -p 5432
```

### 연결 정보 확인

```sql
-- psql 안에서

-- 1. 연결 정보
\conninfo
-- You are connected to database "mydb" as user "myuser" on host "localhost" at port "5432".

-- 2. 현재 데이터베이스
SELECT current_database();

-- 3. 현재 사용자
SELECT current_user;

-- 4. PostgreSQL 버전
SELECT version();
```

### 종료

```sql
-- 방법 1: \q
\q

-- 방법 2: \quit
\quit

-- 방법 3: Ctrl+D
```

---

## 3.3 메타 명령어 (\명령)

### 데이터베이스 관련

```sql
-- 데이터베이스 목록
\l
-- 또는
\list

-- 데이터베이스 연결 (전환)
\c mydb
-- 또는
\connect mydb

-- 특정 사용자로 연결
\c mydb myuser

-- 데이터베이스 생성 (psql 밖에서)
-- createdb mydb
```

### 테이블 관련

```sql
-- 테이블 목록
\dt

-- 모든 스키마의 테이블
\dt *.*

-- 특정 스키마의 테이블
\dt public.*

-- 테이블 구조 (describe)
\d users
\d+ users  -- 더 자세히

-- 인덱스 포함
\di

-- 뷰 목록
\dv

-- 시퀀스 목록
\ds

-- 모든 객체
\d
```

### 스키마 관련

```sql
-- 스키마 목록
\dn

-- 현재 스키마
SHOW search_path;

-- 스키마 변경
SET search_path TO myschema, public;
```

### 사용자와 권한

```sql
-- 사용자(Role) 목록
\du

-- 권한 확인
\dp
-- 또는
\z
```

### 함수와 프로시저

```sql
-- 함수 목록
\df

-- 특정 함수 정의
\df+ function_name

-- 프로시저 목록
\dx
```

---

## 3.4 SQL 실행

### 단일 쿼리

```sql
-- 간단한 SELECT
SELECT * FROM users;

-- 세미콜론(;) 필수
SELECT * FROM users
-- 엔터 눌러도 계속 입력 모드
-- WHERE id = 1;  -- 이제 실행됨
```

### 여러 줄 쿼리

```sql
-- 가독성 있게 작성
SELECT
  id,
  email,
  name,
  created_at
FROM users
WHERE created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 10;
```

### 마지막 쿼리 재실행

```sql
-- 방법 1: ↑ 키로 히스토리

-- 방법 2: Ctrl+R (검색)

-- 방법 3: \g (마지막 쿼리)
SELECT * FROM users;
\g
```

---

## 3.5 출력 형식

### 기본 출력

```sql
SELECT * FROM users;
--  id |      email       |  name  |     created_at
-- ----+------------------+--------+---------------------
--   1 | alice@test.com   | Alice  | 2024-01-15 10:30:00
--   2 | bob@test.com     | Bob    | 2024-01-16 11:00:00
```

### 확장 출력 (\x)

```sql
-- 확장 모드 켜기
\x on

SELECT * FROM users WHERE id = 1;
-- -[ RECORD 1 ]---+---------------------
-- id              | 1
-- email           | alice@test.com
-- name            | Alice
-- created_at      | 2024-01-15 10:30:00

-- 확장 모드 끄기
\x off

-- 자동 (결과가 넓으면 확장 모드)
\x auto
```

### 출력 형식 변경

```sql
-- HTML 형식
\H
SELECT * FROM users LIMIT 2;
-- <table border="1">
--   <tr><th>id</th><th>email</th></tr>
--   <tr><td>1</td><td>alice@test.com</td></tr>
-- </table>

-- CSV 형식
\pset format csv
SELECT * FROM users;
-- id,email,name,created_at
-- 1,alice@test.com,Alice,2024-01-15 10:30:00

-- 기본 형식으로 돌아오기
\pset format aligned
```

### NULL 표시

```sql
-- NULL을 [NULL]로 표시
\pset null '[NULL]'

SELECT id, deleted_at FROM users;
--  id | deleted_at
-- ----+------------
--   1 | [NULL]
--   2 | 2024-01-20 10:00:00
```

---

## 3.6 파일 입출력

### 파일에서 SQL 실행

```sql
-- 방법 1: \i
\i /path/to/script.sql

-- 방법 2: \include
\include /path/to/script.sql

-- 방법 3: psql 밖에서
psql -U myuser -d mydb -f script.sql
```

### 결과를 파일로 저장

```sql
-- 출력을 파일로 리다이렉트
\o output.txt
SELECT * FROM users;
\o  -- 파일 닫기 (다시 화면 출력)

-- CSV로 저장
\copy users TO 'users.csv' CSV HEADER

-- 조건부 저장
\copy (SELECT * FROM users WHERE created_at > '2024-01-01') TO 'recent_users.csv' CSV HEADER

-- CSV에서 불러오기
\copy users FROM 'users.csv' CSV HEADER
```

---

## 3.7 히스토리

### 명령어 히스토리

```bash
# 히스토리 파일 위치
# ~/.psql_history

# 히스토리 보기
# psql 안에서 ↑↓ 키

# 검색
# Ctrl+R 누르고 검색어 입력

# 히스토리 지우기
# \! rm ~/.psql_history
```

### 히스토리 설정

```bash
# ~/.psqlrc 파일 생성
# 히스토리 크기 설정
\set HISTSIZE 10000

# 중복 제거
\set HISTCONTROL ignoredups
```

---

## 3.8 변수 사용

### 변수 설정

```sql
-- 변수 설정
\set myvar 'value'

-- 변수 사용
SELECT :'myvar';
--  ?column?
-- ----------
--  value

-- 숫자 변수
\set limit 10
SELECT * FROM users LIMIT :limit;
```

### 특수 변수

```sql
-- 마지막 쿼리 결과 행 수
\set LAST_ERROR_MESSAGE
\set LAST_ERROR_SQLSTATE

-- 프롬프트 커스터마이징
\set PROMPT1 '%n@%/%R%# '
-- myuser@mydb=#

\set PROMPT2 '%n@%/%R%# '
```

---

## 3.9 .psqlrc 설정 파일

### 파일 생성

```bash
# ~/.psqlrc 파일 생성
touch ~/.psqlrc
```

### 권장 설정

```sql
-- ~/.psqlrc

-- 자동 완성 대소문자 구분 없음
\set COMP_KEYWORD_CASE upper

-- 에러 발생 시 롤백
\set ON_ERROR_ROLLBACK interactive

-- 에러 발생 시 중지
\set ON_ERROR_STOP on

-- NULL 표시
\pset null '[NULL]'

-- 확장 출력 자동
\x auto

-- 히스토리 설정
\set HISTSIZE 10000
\set HISTCONTROL ignoredups

-- 타이밍 표시
\timing

-- 프롬프트 커스터마이징
\set PROMPT1 '%n@%/%R%# '
\set PROMPT2 '> '

-- 페이저 설정 (결과가 많을 때)
\pset pager always

-- 환영 메시지
\echo '\n\033[1;33mWelcome to PostgreSQL!\033[0m\n'
```

---

## 3.10 유용한 팁

### 자동 완성

```sql
-- Tab 키로 자동 완성
SELECT * FROM u<Tab>
-- users로 자동 완성

-- 컬럼 자동 완성
SELECT id, em<Tab>
-- email로 자동 완성
```

### 쿼리 타이밍

```sql
-- 타이밍 켜기
\timing on

SELECT COUNT(*) FROM users;
--  count
-- -------
--   1000
-- (1 row)
--
-- Time: 15.234 ms

-- 타이밍 끄기
\timing off
```

### 쿼리 편집

```sql
-- 에디터 열기
\e

-- 마지막 쿼리 편집
\e

-- 특정 파일 편집
\e /path/to/file.sql

-- 에디터 설정 (환경변수)
export EDITOR=vim  # ~/.bashrc나 ~/.zshrc에 추가
```

### 외부 명령 실행

```sql
-- \! 사용
\! ls -la

-- 현재 디렉토리
\! pwd

-- 파일 내용 확인
\! cat script.sql
```

### 조건부 실행

```sql
-- \gexec: 각 행을 SQL로 실행
SELECT 'DROP TABLE IF EXISTS ' || tablename || ';'
FROM pg_tables
WHERE schemaname = 'public'
\gexec

-- 모든 public 테이블 삭제됨
```

---

## 3.11 실전 예제

### 테이블 생성 및 데이터 삽입

```sql
-- 1. 데이터베이스 생성 및 연결
CREATE DATABASE testdb;
\c testdb

-- 2. 테이블 생성
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 3. 데이터 삽입
INSERT INTO users (email, name) VALUES
  ('alice@example.com', 'Alice'),
  ('bob@example.com', 'Bob'),
  ('charlie@example.com', 'Charlie');

-- 4. 확인
SELECT * FROM users;

-- 5. 테이블 구조 확인
\d users
```

### 데이터 조회

```sql
-- 전체 조회
SELECT * FROM users;

-- 특정 컬럼
SELECT id, name FROM users;

-- 조건
SELECT * FROM users WHERE id = 1;

-- 정렬
SELECT * FROM users ORDER BY created_at DESC;

-- 개수
SELECT COUNT(*) FROM users;
```

### 결과 저장

```sql
-- CSV로 저장
\copy (SELECT * FROM users ORDER BY created_at) TO 'users_export.csv' CSV HEADER

-- 확인
\! cat users_export.csv
```

---

## 3.12 트러블슈팅

### 응답 없음

```
현상: 쿼리 입력 후 아무 반응 없음

원인: 세미콜론(;) 빠짐

해결:
mydb=# SELECT * FROM users
mydb-# ;  -- 세미콜론 추가하고 엔터
```

### 종료 안 됨

```
현상: \q 해도 종료 안 됨

원인: 쿼리 입력 중

해결:
mydb=# SELECT * FROM users
mydb-# \q  -- 작동 안 함
mydb-# ;   -- 먼저 쿼리 종료
mydb=# \q  -- 이제 종료됨

또는 Ctrl+C로 쿼리 취소
```

### Permission denied

```sql
-- 권한 부족
ERROR:  permission denied for table users

-- 해결: 권한 부여
-- postgres 사용자로 실행
GRANT ALL PRIVILEGES ON TABLE users TO myuser;
```

### 연결 거부

```bash
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: No such file or directory

# 해결 1: PostgreSQL 시작
brew services start postgresql@15

# 해결 2: 호스트 명시
psql -h localhost
```

---

## 3.13 자주 쓰는 메타 명령 요약

```sql
-- === 연결 ===
\c dbname              -- 데이터베이스 연결
\conninfo              -- 연결 정보
\q                     -- 종료

-- === 조회 ===
\l                     -- 데이터베이스 목록
\dt                    -- 테이블 목록
\d table_name          -- 테이블 구조
\du                    -- 사용자 목록
\dn                    -- 스키마 목록

-- === 출력 ===
\x                     -- 확장 출력 토글
\timing                -- 타이밍 토글
\pset null '[NULL]'    -- NULL 표시

-- === 파일 ===
\i file.sql            -- 파일 실행
\o file.txt            -- 출력 저장
\copy ...              -- CSV 입출력

-- === 편집 ===
\e                     -- 에디터 열기
\g                     -- 마지막 쿼리 재실행

-- === 기타 ===
\?                     -- 메타 명령 도움말
\h SQL_COMMAND         -- SQL 명령 도움말
\! shell_command       -- 외부 명령 실행
```

---

## 핵심 요약

### psql 시작
```bash
psql -U user -d database -h host -p port
```

### 주요 메타 명령
```sql
\l      -- DB 목록
\c db   -- DB 연결
\dt     -- 테이블 목록
\d tbl  -- 테이블 구조
\q      -- 종료
```

### 출력 형식
```sql
\x      -- 확장 출력
\timing -- 실행 시간
\pset   -- 출력 설정
```

### 파일 입출력
```sql
\i file.sql          -- 실행
\copy ... TO 'file'  -- 저장
```

---

## 다음 단계

psql 사용법을 익혔으니, 이제 본격적으로 **데이터베이스와 테이블**을 다뤄봅시다!

👉 [Chapter 4. 데이터베이스와 테이블](../part02-sql기초/04-데이터베이스와-테이블.md)

---

## 실습 과제

### 과제 1: psql 탐색

```sql
-- 1. psql 접속
-- 2. 데이터베이스 목록 확인
\l
-- 3. 사용자 목록 확인
\du
-- 4. 종료
\q
```

### 과제 2: .psqlrc 설정

```bash
# 1. ~/.psqlrc 파일 생성
# 2. 타이밍, NULL 표시 설정
# 3. psql 재시작하여 확인
```

### 과제 3: 데이터 내보내기

```sql
-- 1. 테이블 생성 및 데이터 삽입
-- 2. CSV로 내보내기
\copy mytable TO 'export.csv' CSV HEADER
-- 3. 파일 확인
\! cat export.csv
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐ 입문
**예상 학습 시간**: 1시간
**대상**: 모든 개발자
**중요도**: ⭐⭐⭐ 매우 중요!
