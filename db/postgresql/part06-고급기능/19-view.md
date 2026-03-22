# Chapter 19. View

## 19.1 View란?

### 정의

```
View (뷰):
저장된 쿼리에 이름을 붙인 가상 테이블

비유: 바로가기
┌─────────────────────────────────┐
│ 실제 폴더                       │
│ /Documents/Projects/2024/Q1/... │
│                                 │
│ 바로가기 (View)                 │
│ "현재 프로젝트" → 실제 폴더     │
└─────────────────────────────────┘

View = 복잡한 쿼리를 간단하게 재사용
```

### View의 특징

```
✅ 장점:
- 복잡한 쿼리 단순화
- 보안 (특정 컬럼만 노출)
- 추상화 (내부 구조 숨김)
- 재사용성

❌ 단점:
- 성능 (매번 쿼리 실행)
- 인덱스 없음
- 제약조건 없음

실제 데이터는 테이블에 저장됨!
```

---

## 19.2 View 생성과 사용

### 기본 생성

```sql
-- 복잡한 쿼리
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(p.id) AS post_count,
  MAX(p.created_at) AS last_post_date
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name, u.email;

-- View로 저장
CREATE VIEW user_stats AS
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(p.id) AS post_count,
  MAX(p.created_at) AS last_post_date
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name, u.email;

-- 사용 (테이블처럼!)
SELECT * FROM user_stats WHERE post_count > 10;

SELECT name, post_count
FROM user_stats
ORDER BY post_count DESC
LIMIT 10;
```

### View 조회

```sql
-- psql에서
\dv

-- SQL로
SELECT
  schemaname,
  viewname,
  definition
FROM pg_views
WHERE schemaname = 'public';

--  schemaname | viewname    | definition
-- ------------+-------------+------------------
--  public     | user_stats  | SELECT u.id...
```

### View 삭제

```sql
-- View 삭제
DROP VIEW user_stats;

-- IF EXISTS 사용
DROP VIEW IF EXISTS user_stats;

-- 여러 View 삭제
DROP VIEW user_stats, post_stats;

-- CASCADE (종속 View도 삭제)
DROP VIEW user_stats CASCADE;
```

---

## 19.3 View 수정

### OR REPLACE

```sql
-- View 생성
CREATE VIEW user_summary AS
SELECT id, name, email FROM users;

-- View 수정
CREATE OR REPLACE VIEW user_summary AS
SELECT id, name, email, created_at FROM users;
-- created_at 컬럼 추가

-- 사용
SELECT * FROM user_summary;
--  id | name  |      email       |       created_at
-- ----+-------+------------------+---------------------
--   1 | Alice | alice@ex.com     | 2024-01-15 10:00:00
```

### 컬럼 추가 제약

```sql
-- ⚠️ 주의: 컬럼 제거는 불가
CREATE VIEW user_summary AS
SELECT id, name, email, created_at FROM users;

-- 에러!
CREATE OR REPLACE VIEW user_summary AS
SELECT id, name FROM users;
-- ERROR: cannot drop columns from view

-- 해결: DROP 후 재생성
DROP VIEW user_summary;
CREATE VIEW user_summary AS
SELECT id, name FROM users;
```

---

## 19.4 실전 View 예제

### 예제 1: 활성 사용자

```sql
-- 활성 사용자 View
CREATE VIEW active_users AS
SELECT
  id,
  name,
  email,
  created_at,
  last_login
FROM users
WHERE is_active = true
  AND deleted_at IS NULL;

-- 사용
SELECT * FROM active_users WHERE last_login > NOW() - INTERVAL '7 days';

SELECT COUNT(*) FROM active_users;
```

### 예제 2: 주문 통계

```sql
-- 주문 요약 View
CREATE VIEW order_summary AS
SELECT
  o.id,
  o.user_id,
  u.name AS user_name,
  u.email,
  o.status,
  o.total_amount,
  o.created_at,
  COUNT(oi.id) AS item_count
FROM orders o
JOIN users u ON o.user_id = u.id
LEFT JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, o.user_id, u.name, u.email, o.status, o.total_amount, o.created_at;

-- 사용
SELECT * FROM order_summary
WHERE status = 'completed'
  AND created_at >= '2024-01-01'
ORDER BY total_amount DESC;

-- 사용자별 총 주문액
SELECT
  user_id,
  user_name,
  COUNT(*) AS order_count,
  SUM(total_amount) AS total_spent
FROM order_summary
WHERE status = 'completed'
GROUP BY user_id, user_name
ORDER BY total_spent DESC;
```

### 예제 3: 게시글 상세

```sql
-- 게시글 상세 정보 View
CREATE VIEW post_details AS
SELECT
  p.id,
  p.title,
  p.content,
  p.status,
  p.view_count,
  p.created_at,
  u.id AS author_id,
  u.name AS author_name,
  u.email AS author_email,
  c.name AS category_name,
  COUNT(DISTINCT cm.id) AS comment_count,
  COUNT(DISTINCT l.id) AS like_count
FROM posts p
JOIN users u ON p.user_id = u.id
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN comments cm ON p.id = cm.post_id
LEFT JOIN likes l ON p.id = l.post_id
GROUP BY p.id, p.title, p.content, p.status, p.view_count, p.created_at,
         u.id, u.name, u.email, c.name;

-- 사용
SELECT * FROM post_details WHERE status = 'published' ORDER BY created_at DESC LIMIT 10;

-- 인기 게시글
SELECT * FROM post_details
WHERE status = 'published'
ORDER BY view_count DESC, like_count DESC
LIMIT 20;
```

### 예제 4: 대시보드 통계

```sql
-- 일별 통계 View
CREATE VIEW daily_stats AS
SELECT
  DATE(created_at) AS date,
  COUNT(DISTINCT CASE WHEN source = 'users' THEN id END) AS new_users,
  COUNT(DISTINCT CASE WHEN source = 'orders' THEN id END) AS new_orders,
  SUM(CASE WHEN source = 'orders' THEN total_amount ELSE 0 END) AS revenue
FROM (
  SELECT id, created_at, 'users' AS source, NULL::NUMERIC AS total_amount
  FROM users
  UNION ALL
  SELECT id, created_at, 'orders' AS source, total_amount
  FROM orders
) AS combined
GROUP BY DATE(created_at)
ORDER BY date DESC;

-- 사용
SELECT * FROM daily_stats WHERE date >= CURRENT_DATE - INTERVAL '30 days';

-- 월별 통계
SELECT
  DATE_TRUNC('month', date) AS month,
  SUM(new_users) AS total_new_users,
  SUM(new_orders) AS total_orders,
  SUM(revenue) AS total_revenue
FROM daily_stats
GROUP BY DATE_TRUNC('month', date)
ORDER BY month DESC;
```

---

## 19.5 보안과 권한

### 컬럼 숨기기

```sql
-- 민감한 정보 제외
CREATE VIEW public_users AS
SELECT
  id,
  name,
  email,
  created_at
FROM users;
-- password_hash, phone 등 민감 정보 제외

-- 일반 사용자에게 권한 부여
GRANT SELECT ON public_users TO public_user;

-- 실제 테이블 접근 차단
REVOKE ALL ON users FROM public_user;
```

### 행 필터링

```sql
-- 자신의 데이터만 보기
CREATE VIEW my_orders AS
SELECT *
FROM orders
WHERE user_id = current_user_id();

-- 함수 정의
CREATE OR REPLACE FUNCTION current_user_id()
RETURNS INT AS $$
  SELECT id FROM users WHERE username = CURRENT_USER;
$$ LANGUAGE sql STABLE;

-- 사용자는 자신의 주문만 볼 수 있음
SELECT * FROM my_orders;
```

### 보안 View

```sql
-- SECURITY BARRIER (보안 강화)
CREATE VIEW sensitive_data WITH (security_barrier = true) AS
SELECT
  id,
  name,
  email
FROM users
WHERE is_public = true;

-- 일반 View: 최적화로 WHERE 조건이 먼저 실행될 수 있음
-- SECURITY BARRIER: View의 WHERE가 항상 먼저 실행 (보안 강화)
```

---

## 19.6 Updatable View (수정 가능한 View)

### 단순 View (자동으로 수정 가능)

```sql
-- 단일 테이블 기반 View
CREATE VIEW active_users AS
SELECT id, name, email, is_active
FROM users
WHERE is_active = true;

-- INSERT
INSERT INTO active_users (name, email)
VALUES ('Alice', 'alice@example.com');
-- 실제로 users 테이블에 삽입!

-- UPDATE
UPDATE active_users SET name = 'Alice Updated' WHERE id = 1;

-- DELETE
DELETE FROM active_users WHERE id = 1;


-- 제약: WITH CHECK OPTION
CREATE OR REPLACE VIEW active_users AS
SELECT id, name, email, is_active
FROM users
WHERE is_active = true
WITH CHECK OPTION;

-- 이제 is_active = false로 UPDATE 불가
UPDATE active_users SET is_active = false WHERE id = 1;
-- ERROR: new row violates check option for view "active_users"
```

### 복잡한 View (INSTEAD OF 트리거 필요)

```sql
-- JOIN이 포함된 View
CREATE VIEW user_with_profile AS
SELECT
  u.id,
  u.name,
  u.email,
  p.bio,
  p.avatar_url
FROM users u
LEFT JOIN profiles p ON u.id = p.user_id;

-- 직접 UPDATE 불가
UPDATE user_with_profile SET bio = 'New bio' WHERE id = 1;
-- ERROR: cannot update view "user_with_profile"


-- INSTEAD OF 트리거로 해결
CREATE OR REPLACE FUNCTION update_user_with_profile()
RETURNS TRIGGER AS $$
BEGIN
  -- users 테이블 업데이트
  UPDATE users
  SET name = NEW.name, email = NEW.email
  WHERE id = NEW.id;

  -- profiles 테이블 업데이트 (또는 삽입)
  INSERT INTO profiles (user_id, bio, avatar_url)
  VALUES (NEW.id, NEW.bio, NEW.avatar_url)
  ON CONFLICT (user_id) DO UPDATE
  SET bio = EXCLUDED.bio, avatar_url = EXCLUDED.avatar_url;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_user_with_profile
INSTEAD OF UPDATE ON user_with_profile
FOR EACH ROW
EXECUTE FUNCTION update_user_with_profile();

-- 이제 UPDATE 가능!
UPDATE user_with_profile SET bio = 'New bio' WHERE id = 1;
```

---

## 19.7 Materialized View (구체화된 뷰)

### 정의

```
Materialized View:
쿼리 결과를 실제로 저장하는 View

일반 View vs Materialized View:
┌─────────────────────────────────┐
│ 일반 View:                      │
│ - 매번 쿼리 실행                │
│ - 항상 최신 데이터              │
│ - 느릴 수 있음                  │
├─────────────────────────────────┤
│ Materialized View:              │
│ - 결과 미리 저장                │
│ - 빠름 (저장된 데이터 읽기)     │
│ - 수동 REFRESH 필요             │
└─────────────────────────────────┘
```

### 생성과 사용

```sql
-- Materialized View 생성
CREATE MATERIALIZED VIEW user_stats_mv AS
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(p.id) AS post_count,
  COUNT(DISTINCT c.id) AS comment_count,
  MAX(p.created_at) AS last_post_date
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
LEFT JOIN comments c ON u.id = c.user_id
GROUP BY u.id, u.name, u.email;

-- 인덱스 추가 (일반 테이블처럼!)
CREATE INDEX idx_user_stats_mv_post_count ON user_stats_mv(post_count);

-- 사용
SELECT * FROM user_stats_mv WHERE post_count > 10;
-- 매우 빠름! (미리 계산된 결과)
```

### REFRESH

```sql
-- 데이터 갱신
REFRESH MATERIALIZED VIEW user_stats_mv;

-- CONCURRENTLY (무중단 갱신)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats_mv;
-- 주의: UNIQUE 인덱스 필요!

CREATE UNIQUE INDEX idx_user_stats_mv_id ON user_stats_mv(id);
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats_mv;
```

### 자동 갱신 (Cron)

```sql
-- pg_cron 확장 설치
CREATE EXTENSION pg_cron;

-- 매일 자정에 갱신
SELECT cron.schedule(
  'refresh-user-stats',
  '0 0 * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats_mv'
);

-- 또는 트리거로 자동 갱신
CREATE OR REPLACE FUNCTION refresh_user_stats()
RETURNS TRIGGER AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats_mv;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_refresh_user_stats
AFTER INSERT OR UPDATE OR DELETE ON posts
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_user_stats();
-- 주의: 성능 영향 고려!
```

---

## 19.8 View vs Materialized View

### 비교

```
특징          | View            | Materialized View
--------------|-----------------|--------------------
저장 방식     | 쿼리만 저장     | 결과 데이터 저장
속도          | 느림 (매번 실행)| 빠름 (저장된 데이터)
데이터 신선도 | 항상 최신       | REFRESH 필요
인덱스        | 불가            | 가능
저장 공간     | 거의 없음       | 많이 사용
갱신 방법     | 자동            | 수동 REFRESH
```

### 사용 시나리오

```
일반 View:
✅ 간단한 쿼리
✅ 실시간 데이터 필요
✅ 자주 변경되는 데이터
✅ 보안/권한 제어

Materialized View:
✅ 복잡한 집계 쿼리
✅ 실시간 불필요 (일/시간 단위 갱신)
✅ 대시보드/보고서
✅ 성능이 중요한 경우
```

---

## 19.9 실전 Materialized View

### 예제 1: 일별 매출 통계

```sql
-- 일별 매출 (복잡한 집계)
CREATE MATERIALIZED VIEW daily_revenue_mv AS
SELECT
  DATE(o.created_at) AS date,
  COUNT(DISTINCT o.id) AS order_count,
  COUNT(DISTINCT o.user_id) AS customer_count,
  SUM(o.total_amount) AS total_revenue,
  AVG(o.total_amount) AS avg_order_value,
  SUM(CASE WHEN o.status = 'completed' THEN o.total_amount ELSE 0 END) AS completed_revenue,
  SUM(CASE WHEN o.status = 'cancelled' THEN o.total_amount ELSE 0 END) AS cancelled_revenue
FROM orders o
GROUP BY DATE(o.created_at)
ORDER BY date DESC;

-- 인덱스
CREATE UNIQUE INDEX idx_daily_revenue_mv_date ON daily_revenue_mv(date);

-- 매일 자정 갱신
SELECT cron.schedule(
  'refresh-daily-revenue',
  '5 0 * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue_mv'
);

-- 사용
SELECT * FROM daily_revenue_mv WHERE date >= CURRENT_DATE - INTERVAL '30 days';
-- 매우 빠름!
```

### 예제 2: 상품 랭킹

```sql
-- 인기 상품 랭킹
CREATE MATERIALIZED VIEW product_ranking_mv AS
SELECT
  p.id,
  p.name,
  p.category_id,
  c.name AS category_name,
  COUNT(DISTINCT oi.order_id) AS order_count,
  SUM(oi.quantity) AS total_sold,
  SUM(oi.quantity * oi.price) AS total_revenue,
  AVG(r.rating) AS avg_rating,
  COUNT(r.id) AS review_count,
  ROW_NUMBER() OVER (ORDER BY SUM(oi.quantity) DESC) AS sales_rank,
  ROW_NUMBER() OVER (PARTITION BY p.category_id ORDER BY SUM(oi.quantity) DESC) AS category_rank
FROM products p
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN reviews r ON p.id = r.product_id
GROUP BY p.id, p.name, p.category_id, c.name;

-- 인덱스
CREATE UNIQUE INDEX idx_product_ranking_mv_id ON product_ranking_mv(id);
CREATE INDEX idx_product_ranking_mv_category ON product_ranking_mv(category_id);

-- 1시간마다 갱신
SELECT cron.schedule(
  'refresh-product-ranking',
  '0 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY product_ranking_mv'
);

-- 사용
SELECT * FROM product_ranking_mv ORDER BY sales_rank LIMIT 100;
SELECT * FROM product_ranking_mv WHERE category_id = 10 ORDER BY category_rank LIMIT 20;
```

### 예제 3: 사용자 세그먼트

```sql
-- 사용자 세그먼트 (RFM 분석)
CREATE MATERIALIZED VIEW user_segments_mv AS
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(DISTINCT o.id) AS order_count,
  SUM(o.total_amount) AS total_spent,
  AVG(o.total_amount) AS avg_order_value,
  MAX(o.created_at) AS last_order_date,
  CURRENT_DATE - MAX(o.created_at)::DATE AS days_since_last_order,
  CASE
    WHEN COUNT(DISTINCT o.id) >= 10 AND SUM(o.total_amount) >= 1000000 THEN 'VIP'
    WHEN COUNT(DISTINCT o.id) >= 5 AND SUM(o.total_amount) >= 500000 THEN 'Gold'
    WHEN COUNT(DISTINCT o.id) >= 2 AND SUM(o.total_amount) >= 100000 THEN 'Silver'
    WHEN COUNT(DISTINCT o.id) >= 1 THEN 'Bronze'
    ELSE 'New'
  END AS segment
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'completed'
GROUP BY u.id, u.name, u.email;

-- 인덱스
CREATE UNIQUE INDEX idx_user_segments_mv_id ON user_segments_mv(id);
CREATE INDEX idx_user_segments_mv_segment ON user_segments_mv(segment);

-- 매일 갱신
SELECT cron.schedule(
  'refresh-user-segments',
  '0 1 * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY user_segments_mv'
);

-- 사용
SELECT segment, COUNT(*) FROM user_segments_mv GROUP BY segment;
SELECT * FROM user_segments_mv WHERE segment = 'VIP' ORDER BY total_spent DESC;
```

---

## 19.10 View 성능 최적화

### 인덱스 활용

```sql
-- View는 기본 테이블의 인덱스를 사용
CREATE VIEW active_posts AS
SELECT * FROM posts WHERE status = 'published';

-- posts.status에 인덱스 있으면 View도 빠름
CREATE INDEX idx_posts_status ON posts(status);

SELECT * FROM active_posts WHERE created_at > '2024-01-01';
-- posts.created_at 인덱스 사용
```

### WITH 절 사용

```sql
-- ❌ 느림: 중복 계산
CREATE VIEW user_stats AS
SELECT
  u.id,
  u.name,
  (SELECT COUNT(*) FROM posts WHERE user_id = u.id) AS post_count,
  (SELECT COUNT(*) FROM comments WHERE user_id = u.id) AS comment_count,
  (SELECT MAX(created_at) FROM posts WHERE user_id = u.id) AS last_post
FROM users u;

-- ✅ 빠름: JOIN 사용
CREATE VIEW user_stats AS
SELECT
  u.id,
  u.name,
  COUNT(DISTINCT p.id) AS post_count,
  COUNT(DISTINCT c.id) AS comment_count,
  MAX(p.created_at) AS last_post
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
LEFT JOIN comments c ON u.id = c.user_id
GROUP BY u.id, u.name;
```

### Materialized View로 전환

```sql
-- 느린 일반 View
CREATE VIEW complex_stats AS
SELECT ... -- 복잡한 쿼리

-- Materialized View로 전환
DROP VIEW complex_stats;
CREATE MATERIALIZED VIEW complex_stats AS
SELECT ... -- 같은 쿼리

CREATE UNIQUE INDEX idx_complex_stats_id ON complex_stats(id);

-- 주기적 갱신
REFRESH MATERIALIZED VIEW CONCURRENTLY complex_stats;
```

---

## 핵심 요약

### View

```sql
-- 생성
CREATE VIEW view_name AS SELECT ...;

-- 수정
CREATE OR REPLACE VIEW view_name AS SELECT ...;

-- 삭제
DROP VIEW view_name;

-- 사용
SELECT * FROM view_name;
```

### Materialized View

```sql
-- 생성
CREATE MATERIALIZED VIEW mv_name AS SELECT ...;

-- 갱신
REFRESH MATERIALIZED VIEW mv_name;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_name;

-- 인덱스
CREATE INDEX idx_name ON mv_name(column);
```

### View vs Materialized View

```
View:
- 쿼리만 저장
- 항상 최신
- 매번 실행 (느림)

Materialized View:
- 결과 저장
- REFRESH 필요
- 저장된 데이터 읽기 (빠름)
```

### 사용 시나리오

```
일반 View:
- 단순 쿼리
- 실시간 데이터
- 보안/권한

Materialized View:
- 복잡한 집계
- 대시보드/보고서
- 성능 중요
```

---

## 다음 단계

View를 마스터했으니, 이제 **Trigger**를 배워봅시다!

👉 [Chapter 20. Trigger](20-trigger.md)

---

## 실습 과제

### 과제 1: 기본 View

```sql
-- 1. 활성 사용자 View 생성
-- 2. 사용자 통계 View 생성 (게시글 수, 댓글 수)
-- 3. View 조회 및 필터링
```

### 과제 2: Materialized View

```sql
-- 1. 일별 매출 통계 Materialized View
-- 2. 인덱스 추가
-- 3. REFRESH 실행
-- 4. 일반 View와 성능 비교
```

### 과제 3: 보안 View

```sql
-- 1. 민감한 정보 제외한 View
-- 2. WITH CHECK OPTION 사용
-- 3. 권한 설정
```

### 과제 4: 복잡한 Materialized View

```sql
-- 1. 상품 랭킹 Materialized View
-- 2. 여러 테이블 JOIN
-- 3. 윈도우 함수 사용
-- 4. 자동 갱신 스케줄 설정
```

---

**작성일**: 2024년
**PostgreSQL 버전**: 15.x+
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 2-3시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐⭐ 매우 중요!