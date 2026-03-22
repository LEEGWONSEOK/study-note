# Chapter 22. Full Text Search

## 22.1 Full Text Search란?

### 정의

```
Full Text Search (전문 검색):
텍스트 내용을 분석하여 의미 기반으로 검색

비유: 도서관 검색
┌─────────────────────────────────┐
│ ❌ LIKE 검색:                   │
│ "PostgreSQL" → 정확히 일치만    │
│                                 │
│ ✅ 전문 검색:                   │
│ "PostgreSQL" →                  │
│ - PostgreSQL                    │
│ - postgres                      │
│ - PostgreSQL의                  │
│ - PostgreSQL를                  │
│ (어간, 형태소 분석)             │
└─────────────────────────────────┘
```

### LIKE vs Full Text Search

```sql
-- ❌ LIKE: 느리고 부정확
SELECT * FROM posts
WHERE title LIKE '%postgresql%' OR content LIKE '%postgresql%';
-- 문제:
-- - 인덱스 사용 불가
-- - 대소문자 구분
-- - 어간 처리 안 됨

-- ✅ Full Text Search: 빠르고 정확
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql');
-- 장점:
-- - GIN 인덱스 사용
-- - 어간 처리 (PostgreSQL, postgres → postgres)
-- - 랭킹 (관련도 점수)
```

---

## 22.2 기본 개념

### tsvector (텍스트 벡터)

```sql
-- 텍스트를 검색 가능한 형태로 변환
SELECT to_tsvector('english', 'PostgreSQL is a powerful database');
-- 'databas':6 'postgresql':1 'power':4

-- 특징:
-- - 소문자 변환
-- - 불용어(is, a) 제거
-- - 어간 추출 (powerful → power, database → databas)
-- - 위치 정보 포함 (:1, :4, :6)
```

### tsquery (검색 쿼리)

```sql
-- 검색 쿼리 생성
SELECT to_tsquery('english', 'postgresql & database');
-- 'postgresql' & 'databas'

-- 연산자:
-- &  : AND
-- |  : OR
-- !  : NOT
-- <-> : 인접 (바로 다음)

-- 예제
SELECT to_tsquery('english', 'postgresql & (fast | powerful)');
-- 'postgresql' & ( 'fast' | 'power' )
```

### @@ (매치 연산자)

```sql
-- 매칭 여부 확인
SELECT
  to_tsvector('english', 'PostgreSQL is powerful') @@
  to_tsquery('english', 'postgresql & powerful');
-- true

SELECT
  to_tsvector('english', 'PostgreSQL is powerful') @@
  to_tsquery('english', 'mysql');
-- false
```

---

## 22.3 기본 검색

### 단순 검색

```sql
-- 게시글 검색
SELECT id, title, content
FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql');

-- 결과:
--  id | title                    | content
-- ----+--------------------------+------------------
--   1 | PostgreSQL Tutorial      | Learn PostgreSQL...
--   2 | Database Performance     | ...postgres...
```

### 여러 조건 검색

```sql
-- AND 검색
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql & performance');

-- OR 검색
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql | mysql');

-- NOT 검색
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql & !mysql');
```

### plainto_tsquery (간단한 쿼리)

```sql
-- 사용자 입력을 자동으로 변환
SELECT plainto_tsquery('english', 'postgresql performance tuning');
-- 'postgresql' & 'performance' & 'tune'

-- 사용
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ plainto_tsquery('english', 'postgresql performance');
```

### phraseto_tsquery (구문 검색)

```sql
-- 정확한 구문 검색
SELECT phraseto_tsquery('english', 'postgresql database');
-- 'postgresql' <-> 'databas'

-- "postgresql database"가 연속으로 나오는 것만
SELECT * FROM posts
WHERE to_tsvector('english', content)
      @@ phraseto_tsquery('english', 'postgresql database');
```

---

## 22.4 랭킹 (관련도)

### ts_rank (기본 랭킹)

```sql
-- 관련도 점수 계산
SELECT
  id,
  title,
  ts_rank(
    to_tsvector('english', title || ' ' || content),
    to_tsquery('english', 'postgresql')
  ) AS rank
FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql')
ORDER BY rank DESC;

-- 결과:
--  id | title                | rank
-- ----+----------------------+--------
--   1 | PostgreSQL Tutorial  | 0.607
--   2 | Database Guide       | 0.303
```

### ts_rank_cd (커버 밀도 랭킹)

```sql
-- 검색어 밀도 기반 랭킹
SELECT
  id,
  title,
  ts_rank_cd(
    to_tsvector('english', title || ' ' || content),
    to_tsquery('english', 'postgresql & performance')
  ) AS rank
FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql & performance')
ORDER BY rank DESC;
```

### 가중치 적용

```sql
-- 제목에 더 높은 가중치
SELECT
  id,
  title,
  ts_rank(
    setweight(to_tsvector('english', title), 'A') ||
    setweight(to_tsvector('english', content), 'B'),
    to_tsquery('english', 'postgresql')
  ) AS rank
FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql')
ORDER BY rank DESC;

-- 가중치: A > B > C > D
```

---

## 22.5 검색 컬럼 추가

### tsvector 컬럼 추가

```sql
-- 컬럼 추가
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- 데이터 채우기
UPDATE posts
SET search_vector = to_tsvector('english', title || ' ' || content);

-- 검색 (빠름!)
SELECT * FROM posts
WHERE search_vector @@ to_tsquery('english', 'postgresql');
```

### 자동 업데이트 (Trigger)

```sql
-- 트리거 함수
CREATE OR REPLACE FUNCTION posts_search_trigger()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector := to_tsvector('english', NEW.title || ' ' || NEW.content);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 트리거 생성
CREATE TRIGGER tsvector_update
BEFORE INSERT OR UPDATE OF title, content ON posts
FOR EACH ROW
EXECUTE FUNCTION posts_search_trigger();

-- 이제 INSERT/UPDATE 시 자동 갱신!
INSERT INTO posts (title, content)
VALUES ('PostgreSQL Guide', 'Learn PostgreSQL...');
-- search_vector 자동 생성
```

---

## 22.6 GIN 인덱스

### 인덱스 생성

```sql
-- GIN 인덱스 생성
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);

-- 검색 (매우 빠름!)
EXPLAIN ANALYZE
SELECT * FROM posts
WHERE search_vector @@ to_tsquery('english', 'postgresql');

-- Index Scan using idx_posts_search
-- Execution Time: 0.5 ms
```

### 함수 기반 인덱스

```sql
-- search_vector 컬럼 없이도 가능
CREATE INDEX idx_posts_search_func ON posts
USING GIN(to_tsvector('english', title || ' ' || content));

-- 검색
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('english', 'postgresql');
-- 인덱스 사용!
```

---

## 22.7 다국어 지원

### 언어 설정

```sql
-- 영어
SELECT to_tsvector('english', 'running runs');
-- 'run':1,2

-- 한국어 (기본 simple)
SELECT to_tsvector('simple', '데이터베이스 관리 시스템');
-- '관리':2 '데이터베이스':1 '시스템':3

-- 지원 언어 확인
SELECT cfgname FROM pg_ts_config;
-- english, simple, ...
```

### 한국어 검색 (pg_trgm)

```sql
-- pg_trgm 확장 설치
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 유사도 검색
SELECT
  title,
  similarity(title, '데이터베이스') AS sim
FROM posts
WHERE title % '데이터베이스'  -- % 연산자: 유사도 검색
ORDER BY sim DESC;

-- GIN 인덱스
CREATE INDEX idx_posts_title_trgm ON posts USING GIN(title gin_trgm_ops);

-- 부분 매칭
SELECT * FROM posts WHERE title LIKE '%데이터%';
-- 인덱스 사용!
```

---

## 22.8 실전 예제

### 예제 1: 블로그 검색

```sql
-- 테이블 준비
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  author_id INT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  search_vector tsvector
);

-- 트리거
CREATE TRIGGER posts_search_update
BEFORE INSERT OR UPDATE OF title, content ON posts
FOR EACH ROW
EXECUTE FUNCTION tsvector_update_trigger(
  search_vector, 'pg_catalog.english', title, content
);

-- 인덱스
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);

-- 검색 함수
CREATE OR REPLACE FUNCTION search_posts(
  query_text TEXT,
  limit_count INT DEFAULT 20
)
RETURNS TABLE(
  id INT,
  title VARCHAR,
  content TEXT,
  rank REAL,
  headline TEXT
) AS $$
BEGIN
  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.content,
    ts_rank(p.search_vector, query) AS rank,
    ts_headline(
      'english',
      p.content,
      query,
      'MaxWords=50, MinWords=20'
    ) AS headline
  FROM posts p,
       to_tsquery('english', query_text) query
  WHERE p.search_vector @@ query
  ORDER BY rank DESC, p.created_at DESC
  LIMIT limit_count;
END;
$$ LANGUAGE plpgsql;

-- 사용
SELECT * FROM search_posts('postgresql & performance');
```

### 예제 2: 전자상거래 상품 검색

```sql
-- 상품 테이블
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100),
  brand VARCHAR(100),
  price NUMERIC(10, 2),
  search_vector tsvector
);

-- 가중치 적용 업데이트
UPDATE products
SET search_vector =
  setweight(to_tsvector('english', name), 'A') ||
  setweight(to_tsvector('english', COALESCE(brand, '')), 'B') ||
  setweight(to_tsvector('english', COALESCE(category, '')), 'C') ||
  setweight(to_tsvector('english', COALESCE(description, '')), 'D');

-- 트리거
CREATE OR REPLACE FUNCTION products_search_trigger()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', NEW.name), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.brand, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(NEW.category, '')), 'C') ||
    setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'D');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_update
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION products_search_trigger();

-- 인덱스
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- 검색
SELECT
  id,
  name,
  brand,
  price,
  ts_rank(search_vector, query) AS rank
FROM products,
     to_tsquery('english', 'laptop & (dell | hp)') query
WHERE search_vector @@ query
ORDER BY rank DESC, price ASC;
```

### 예제 3: 사용자 프로필 검색

```sql
-- 사용자 테이블
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  full_name VARCHAR(100),
  bio TEXT,
  skills TEXT[],
  location VARCHAR(100),
  search_vector tsvector
);

-- 배열 포함 검색 벡터
CREATE OR REPLACE FUNCTION users_search_trigger()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', NEW.full_name), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.bio, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(array_to_string(NEW.skills, ' '), '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.location, '')), 'C');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_search_update
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION users_search_trigger();

-- 인덱스
CREATE INDEX idx_users_search ON users USING GIN(search_vector);

-- 검색
SELECT
  id,
  username,
  full_name,
  skills,
  ts_rank(search_vector, query) AS rank
FROM users,
     to_tsquery('english', 'javascript & react') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

---

## 22.9 하이라이트 (ts_headline)

### 기본 사용

```sql
-- 검색어 하이라이트
SELECT
  ts_headline(
    'english',
    'PostgreSQL is a powerful open source database',
    to_tsquery('english', 'postgresql & database')
  );
-- <b>PostgreSQL</b> is a powerful open source <b>database</b>
```

### 옵션 설정

```sql
SELECT
  ts_headline(
    'english',
    content,
    to_tsquery('english', 'postgresql'),
    'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=20, ShortWord=3'
  ) AS headline
FROM posts
WHERE search_vector @@ to_tsquery('english', 'postgresql');

-- 옵션:
-- StartSel/StopSel: 하이라이트 태그
-- MaxWords/MinWords: 표시할 단어 수
-- ShortWord: 짧은 단어 제외
-- MaxFragments: 조각 수
```

---

## 22.10 성능 최적화

### 인덱스 크기 확인

```sql
SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE tablename = 'posts';

--  indexname          | size
-- --------------------+-------
--  idx_posts_search   | 15 MB
```

### 파티셔닝과 검색

```sql
-- 날짜별 파티셔닝
CREATE TABLE posts (
  id SERIAL,
  title VARCHAR(255),
  content TEXT,
  created_at TIMESTAMPTZ,
  search_vector tsvector
) PARTITION BY RANGE (created_at);

CREATE TABLE posts_2024_01 PARTITION OF posts
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- 각 파티션에 인덱스
CREATE INDEX idx_posts_2024_01_search
ON posts_2024_01 USING GIN(search_vector);

-- 검색 (자동으로 파티션 프루닝)
SELECT * FROM posts
WHERE created_at >= '2024-01-01'
  AND created_at < '2024-02-01'
  AND search_vector @@ to_tsquery('english', 'postgresql');
```

### 통계 업데이트

```sql
-- 검색 성능을 위한 통계 갱신
ANALYZE posts;

-- 자동 VACUUM 확인
SELECT
  schemaname,
  tablename,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'posts';
```

---

## 22.11 검색 개선

### 동의어 처리

```sql
-- 동의어 사전 생성
CREATE TEXT SEARCH DICTIONARY synonym_dict (
  TEMPLATE = synonym,
  SYNONYMS = my_synonyms
);

-- my_synonyms 파일 (/usr/share/postgresql/15/tsearch_data/my_synonyms.syn):
-- postgres postgresql
-- db database

-- 설정에 추가
ALTER TEXT SEARCH CONFIGURATION english
ALTER MAPPING FOR asciiword WITH synonym_dict, english_stem;

-- 이제 "postgres" 검색하면 "postgresql"도 매칭됨
```

### 불용어 처리

```sql
-- 불용어 확인
SELECT * FROM ts_debug('english', 'the quick brown fox');
--   alias   |   description   |  token  | dictionaries | dictionary | lexemes
-- ----------+-----------------+---------+--------------+------------+---------
--  article  | Article         | the     | {english_stem}|            | {}      -- 불용어
--  adjective| Adjective       | quick   | {english_stem}| english_stem| {quick}
--  adjective| Adjective       | brown   | {english_stem}| english_stem| {brown}
--  noun     | Noun            | fox     | {english_stem}| english_stem| {fox}
```

### 퍼지 검색

```sql
-- pg_trgm으로 유사 검색
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 유사도 기반 검색
SELECT
  title,
  similarity(title, 'PostgreSQL') AS sim
FROM posts
WHERE similarity(title, 'PostgreSQL') > 0.3
ORDER BY sim DESC;

-- 오타 허용
SELECT * FROM posts
WHERE title % 'postgresgl';  -- PostgreSQL과 유사
```

---

## 핵심 요약

### 기본 개념

```sql
-- tsvector: 검색 가능한 텍스트
to_tsvector('english', text)

-- tsquery: 검색 쿼리
to_tsquery('english', 'word1 & word2')
plainto_tsquery('english', 'word1 word2')
phraseto_tsquery('english', 'exact phrase')

-- 매칭
tsvector @@ tsquery
```

### 랭킹

```sql
-- 관련도 점수
ts_rank(tsvector, tsquery)
ts_rank_cd(tsvector, tsquery)

-- 가중치
setweight(to_tsvector('english', text), 'A')
```

### 검색 컬럼

```sql
-- 컬럼 추가
ALTER TABLE table ADD COLUMN search_vector tsvector;

-- 트리거
CREATE TRIGGER tsvector_update
BEFORE INSERT OR UPDATE ON table
FOR EACH ROW
EXECUTE FUNCTION tsvector_update_trigger(
  search_vector, 'pg_catalog.english', column1, column2
);

-- 인덱스
CREATE INDEX idx_search ON table USING GIN(search_vector);
```

### 검색 쿼리

```sql
SELECT * FROM table
WHERE search_vector @@ to_tsquery('english', 'query')
ORDER BY ts_rank(search_vector, to_tsquery('english', 'query')) DESC;
```

---

## 다음 단계

Full Text Search를 마스터했으니, 이제 **JSON과 JSONB**를 배워봅시다!

👉 [Chapter 23. JSON과 JSONB](23-json과-jsonb.md)

---

## 실습 과제

### 과제 1: 기본 검색

```sql
-- 1. 블로그 검색 시스템 구축
-- 2. tsvector 컬럼 추가
-- 3. GIN 인덱스 생성
-- 4. 검색 쿼리 작성
```

### 과제 2: 랭킹

```sql
-- 1. ts_rank로 관련도 정렬
-- 2. 제목/본문 가중치 적용
-- 3. 상위 10개 결과 출력
```

### 과제 3: 하이라이트

```sql
-- 1. ts_headline으로 검색어 하이라이트
-- 2. 옵션 설정 (MaxWords, MinWords)
-- 3. HTML 마크업 적용
```

### 과제 4: 실전 검색

```sql
-- 1. 전자상거래 상품 검색
-- 2. 여러 컬럼 검색 (이름, 설명, 카테고리)
-- 3. 가중치 적용
-- 4. 성능 측정
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐⭐ 매우 중요!
