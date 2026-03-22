# Chapter 33. SQL Injection 방어

## 🎯 이 챕터의 목표
- **SQL Injection** 공격 원리 완벽 이해하기
- **Parameterized Query** 마스터하기
- **ORM 안전하게** 사용하는 방법 익히기
- **입력 검증과 이스케이핑** 구현하기
- **실제 공격 시나리오** 재현하고 방어하기
- **보안 테스트** 자동화하기

---

## 1. SQL Injection이란? (비유로 이해하기)

### 🚪 출입 보안 시스템에 비유하면

```
SQL Injection = 출입증 위조
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

정상적인 상황:
  직원: "제 이름은 Alice입니다"
  시스템: "Alice 확인, 입장 허가"

SQL Injection 공격:
  해커: "제 이름은 Alice' OR '1'='1 입니다"
  시스템: "Alice' OR '1'='1 확인... 어? 모두 입장 허가!"

실제 SQL:
  -- 정상
  SELECT * FROM users WHERE name = 'Alice'

  -- 공격
  SELECT * FROM users WHERE name = 'Alice' OR '1'='1'
  -- '1'='1'은 항상 참 → 모든 사용자 조회!
```

---

## 2. SQL Injection 공격 원리

### 💥 취약한 코드 예시

#### Node.js (취약한 코드)
```javascript
// ❌ 절대 이렇게 하지 마세요!
const { Pool } = require('pg');
const pool = new Pool({ /* config */ });

// 사용자 로그인
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  // 🚨 취약점: 사용자 입력을 직접 SQL에 삽입
  const query = `
    SELECT * FROM users
    WHERE username = '${username}'
      AND password = '${password}'
  `;

  const result = await pool.query(query);

  if (result.rows.length > 0) {
    res.json({ success: true, user: result.rows[0] });
  } else {
    res.json({ success: false });
  }
});
```

### 🎭 공격 시나리오

#### 공격 1: 인증 우회
```javascript
// 공격자의 입력:
username: "admin' OR '1'='1"
password: "anything"

// 생성되는 SQL:
SELECT * FROM users
WHERE username = 'admin' OR '1'='1'
  AND password = 'anything'

// 해석:
// username = 'admin' 또는 '1'='1' (항상 참)
// → admin 계정으로 로그인 성공!

// 더 간단한 공격:
username: "admin'--"
password: "anything"

// 생성되는 SQL:
SELECT * FROM users
WHERE username = 'admin'--' AND password = 'anything'
-- '--'는 SQL 주석 → password 체크 무시됨!
```

#### 공격 2: 데이터 유출
```javascript
// 사용자 검색 기능 (취약)
app.get('/search', async (req, res) => {
  const { keyword } = req.query;

  const query = `
    SELECT id, name, email FROM users
    WHERE name LIKE '%${keyword}%'
  `;

  const result = await pool.query(query);
  res.json(result.rows);
});

// 공격자의 입력:
keyword: "%' UNION SELECT id, password, email FROM users--"

// 생성되는 SQL:
SELECT id, name, email FROM users
WHERE name LIKE '%%' UNION SELECT id, password, email FROM users--%'

// 결과: 모든 사용자의 비밀번호 유출!
```

#### 공격 3: 데이터 삭제
```javascript
// 공격자의 입력:
keyword: "'; DROP TABLE users; --"

// 생성되는 SQL:
SELECT id, name, email FROM users
WHERE name LIKE '%'; DROP TABLE users; --%'

// 결과: users 테이블 삭제! 💥
```

#### 공격 4: 권한 상승
```sql
-- 공격자의 입력:
username: "'; UPDATE users SET role = 'admin' WHERE username = 'hacker'; --"

-- 생성되는 SQL:
SELECT * FROM users
WHERE username = ''; UPDATE users SET role = 'admin' WHERE username = 'hacker'; --'

-- 결과: hacker 계정이 관리자로 승격!
```

---

## 3. Parameterized Query (안전한 방법) ⭐

### ✅ 올바른 코드 예시

#### Node.js (pg 라이브러리)
```javascript
const { Pool } = require('pg');
const pool = new Pool({ /* config */ });

// ✅ 안전한 로그인
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  // Parameterized Query 사용
  const query = `
    SELECT * FROM users
    WHERE username = $1
      AND password = $2
  `;

  // 파라미터는 별도로 전달
  const result = await pool.query(query, [username, password]);

  if (result.rows.length > 0) {
    res.json({ success: true, user: result.rows[0] });
  } else {
    res.json({ success: false });
  }
});

// 공격 시도:
// username: "admin' OR '1'='1"
// password: "anything"

// 실제 실행되는 쿼리:
// SELECT * FROM users
// WHERE username = 'admin'' OR ''1''=''1'
//   AND password = 'anything'
// → 리터럴 문자열로 처리되어 공격 실패! ✅
```

#### 왜 안전한가?

```
Parameterized Query의 작동 원리
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. SQL 문법 파싱 (Prepare)
   SELECT * FROM users WHERE username = $1 AND password = $2
   ↓
   SQL 구조 확정 (WHERE 절에 2개 파라미터)

2. 파라미터 바인딩 (Bind)
   $1 ← "admin' OR '1'='1"  (문자열로 이스케이핑)
   $2 ← "anything"

3. 실행 (Execute)
   파라미터는 데이터로만 취급, SQL 구조 변경 불가!

결과: SQL Injection 원천 차단 ✅
```

### 📚 다양한 언어/라이브러리 예시

#### Python (psycopg2)
```python
import psycopg2

conn = psycopg2.connect("dbname=mydb user=postgres")
cur = conn.cursor()

# ❌ 취약한 코드
username = request.form['username']
cur.execute(f"SELECT * FROM users WHERE username = '{username}'")

# ✅ 안전한 코드 (Parameterized Query)
username = request.form['username']
cur.execute("SELECT * FROM users WHERE username = %s", (username,))

# 또는 Named Parameters
cur.execute(
    "SELECT * FROM users WHERE username = %(username)s AND age > %(age)s",
    {'username': username, 'age': 18}
)

cur.close()
conn.close()
```

#### Java (JDBC)
```java
// ❌ 취약한 코드
String username = request.getParameter("username");
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(
    "SELECT * FROM users WHERE username = '" + username + "'"
);

// ✅ 안전한 코드 (PreparedStatement)
String username = request.getParameter("username");
String password = request.getParameter("password");

PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ? AND password = ?"
);
pstmt.setString(1, username);
pstmt.setString(2, password);

ResultSet rs = pstmt.executeQuery();
```

#### PHP (PDO)
```php
<?php
// ❌ 취약한 코드
$username = $_POST['username'];
$sql = "SELECT * FROM users WHERE username = '$username'";
$result = $pdo->query($sql);

// ✅ 안전한 코드 (Prepared Statement)
$username = $_POST['username'];
$password = $_POST['password'];

$stmt = $pdo->prepare("SELECT * FROM users WHERE username = :username AND password = :password");
$stmt->execute(['username' => $username, 'password' => $password]);
$result = $stmt->fetchAll();
?>
```

---

## 4. ORM에서의 SQL Injection

### 🛡️ ORM도 안전하지 않을 수 있다!

#### Sequelize (Node.js ORM)

```javascript
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize('postgres://user:pass@localhost:5432/mydb');

const User = sequelize.define('User', {
  username: DataTypes.STRING,
  email: DataTypes.STRING,
  role: DataTypes.STRING
});

// ✅ 안전: ORM 메서드 사용
app.get('/users', async (req, res) => {
  const { role } = req.query;

  const users = await User.findAll({
    where: { role: role }  // Parameterized Query 자동 생성
  });

  res.json(users);
});

// ⚠️ 위험: Raw Query
app.get('/search', async (req, res) => {
  const { keyword } = req.query;

  // ❌ 취약한 코드
  const users = await sequelize.query(
    `SELECT * FROM users WHERE name LIKE '%${keyword}%'`
  );

  // ✅ 안전한 코드 (Replacement)
  const users = await sequelize.query(
    'SELECT * FROM users WHERE name LIKE :keyword',
    {
      replacements: { keyword: `%${keyword}%` },
      type: Sequelize.QueryTypes.SELECT
    }
  );

  res.json(users);
});

// ⚠️ 주의: order, attributes는 검증 필요
app.get('/users', async (req, res) => {
  const { sortBy } = req.query;

  // ❌ 취약한 코드
  const users = await User.findAll({
    order: [[sortBy, 'ASC']]  // sortBy에 SQL 주입 가능!
  });

  // ✅ 안전한 코드 (화이트리스트)
  const allowedColumns = ['name', 'email', 'created_at'];
  const sortBy = allowedColumns.includes(req.query.sortBy)
    ? req.query.sortBy
    : 'created_at';

  const users = await User.findAll({
    order: [[sortBy, 'ASC']]
  });

  res.json(users);
});
```

#### TypeORM (TypeScript ORM)

```typescript
import { getRepository } from 'typeorm';
import { User } from './entity/User';

// ✅ 안전: Query Builder
app.get('/users', async (req, res) => {
  const { role } = req.query;

  const users = await getRepository(User)
    .createQueryBuilder('user')
    .where('user.role = :role', { role: role })
    .getMany();

  res.json(users);
});

// ❌ 취약: Raw SQL
app.get('/search', async (req, res) => {
  const { keyword } = req.query;

  // 취약한 코드
  const users = await getRepository(User)
    .query(`SELECT * FROM users WHERE name LIKE '%${keyword}%'`);

  // ✅ 안전한 코드
  const users = await getRepository(User)
    .query('SELECT * FROM users WHERE name LIKE $1', [`%${keyword}%`]);

  res.json(users);
});
```

### 🎯 ORM 사용 시 주의사항

```
✅ 안전한 ORM 사용법
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. ORM 메서드 사용 (find, where, update 등)
   → 자동으로 Parameterized Query 생성

2. Raw Query 사용 시:
   - 반드시 파라미터 바인딩 사용
   - 문자열 연결(+, template literal) 금지

3. 동적 컬럼명/테이블명:
   - 화이트리스트로 검증
   - 사용자 입력을 컬럼명/테이블명으로 직접 사용 금지

4. LIKE 패턴:
   - %를 애플리케이션에서 추가
   - 사용자 입력에 %가 있으면 이스케이핑

5. ORDER BY, LIMIT, OFFSET:
   - 화이트리스트 또는 타입 검증
   - 숫자는 parseInt() 후 사용
```

---

## 5. 입력 검증과 이스케이핑

### ✅ 다층 방어 전략

```
방어의 계층
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1️⃣ 입력 검증 (가장 바깥)
   ├─ 화이트리스트 (허용 목록)
   ├─ 타입 검증 (숫자는 숫자로)
   └─ 길이 제한

2️⃣ Parameterized Query (핵심 방어) ⭐
   └─ SQL 구조와 데이터 분리

3️⃣ 최소 권한 (DB 사용자)
   ├─ 애플리케이션 사용자는 DDL 불가
   └─ 필요한 테이블만 접근

4️⃣ 에러 메시지 숨기기
   └─ SQL 에러를 클라이언트에 노출 금지

5️⃣ 모니터링 및 로깅
   └─ 의심스러운 쿼리 패턴 감지
```

### 🔍 입력 검증 예시

```javascript
const express = require('express');
const { body, query, validationResult } = require('express-validator');

// 1. 타입 검증
app.get('/user/:id',
  // id는 숫자만 허용
  param('id').isInt(),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const userId = parseInt(req.params.id, 10);
    const user = await pool.query(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );

    res.json(user.rows[0]);
  }
);

// 2. 화이트리스트 검증
app.get('/users',
  query('sortBy').isIn(['name', 'email', 'created_at']),
  query('order').isIn(['ASC', 'DESC']),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { sortBy, order } = req.query;

    // sortBy와 order는 화이트리스트 검증됨
    // 하지만 여전히 직접 삽입은 위험!
    const query = `SELECT * FROM users ORDER BY ${sortBy} ${order}`;
    // ⚠️ 이것도 위험할 수 있음 (Second-Order Injection)

    // ✅ 더 안전한 방법
    const allowedColumns = {
      'name': 'name',
      'email': 'email',
      'created_at': 'created_at'
    };
    const columnMap = allowedColumns[sortBy] || 'created_at';
    const orderDir = order === 'DESC' ? 'DESC' : 'ASC';

    const result = await pool.query(
      `SELECT * FROM users ORDER BY ${columnMap} ${orderDir}`
    );

    res.json(result.rows);
  }
);

// 3. 문자열 길이 제한
app.post('/users',
  body('username').isLength({ min: 3, max: 20 }).isAlphanumeric(),
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8, max: 100 }),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { username, email, password } = req.body;

    // Parameterized Query
    const result = await pool.query(
      'INSERT INTO users (username, email, password) VALUES ($1, $2, $3) RETURNING id',
      [username, email, password]
    );

    res.json({ success: true, userId: result.rows[0].id });
  }
);

// 4. LIKE 패턴 이스케이핑
function escapeLikePattern(str) {
  return str.replace(/[%_\\]/g, '\\$&');
}

app.get('/search',
  query('keyword').isLength({ max: 50 }),
  async (req, res) => {
    const { keyword } = req.query;

    // 특수문자 이스케이핑
    const escapedKeyword = escapeLikePattern(keyword);

    const result = await pool.query(
      "SELECT * FROM products WHERE name LIKE $1 ESCAPE '\\'",
      [`%${escapedKeyword}%`]
    );

    res.json(result.rows);
  }
);
```

---

## 6. 에러 처리 및 보안

### 🚨 에러 메시지 노출 방지

```javascript
// ❌ 위험: SQL 에러를 클라이언트에 노출
app.get('/user/:id', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT * FROM users WHERE id = $1',
      [req.params.id]
    );
    res.json(result.rows[0]);
  } catch (error) {
    // SQL 에러 메시지가 그대로 노출됨!
    res.status(500).json({ error: error.message });
    // 예: "column 'pasword' does not exist"
    //     → 테이블 구조 정보 유출!
  }
});

// ✅ 안전: 일반적인 에러 메시지
app.get('/user/:id', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT * FROM users WHERE id = $1',
      [req.params.id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(result.rows[0]);
  } catch (error) {
    // 서버 로그에는 상세 에러 기록
    console.error('Database error:', error);

    // 클라이언트에는 일반적인 메시지만
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 전역 에러 핸들러
app.use((err, req, res, next) => {
  // 로그 기록
  console.error('Error:', {
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    ip: req.ip
  });

  // 프로덕션: 상세 정보 숨김
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    // 개발: 디버깅용 상세 정보
    res.status(500).json({
      error: err.message,
      stack: err.stack
    });
  }
});
```

---

## 7. Stored Procedure로 보안 강화

### 🔒 Stored Procedure 활용

```sql
-- 사용자 인증 Stored Procedure
CREATE OR REPLACE FUNCTION authenticate_user(
  p_username VARCHAR,
  p_password VARCHAR
)
RETURNS TABLE(user_id INT, username VARCHAR, role VARCHAR)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT id, username, role
  FROM users
  WHERE username = p_username
    AND password = crypt(p_password, password);  -- bcrypt 해시 비교

  -- 실패 시 빈 결과 반환
END;
$$;

-- 애플리케이션에서 호출
-- Parameterized Query로 안전하게 호출
const result = await pool.query(
  'SELECT * FROM authenticate_user($1, $2)',
  [username, password]
);

-- 장점:
-- 1. 복잡한 비즈니스 로직을 DB에 캡슐화
-- 2. 애플리케이션 코드 단순화
-- 3. 권한 제어 (사용자는 테이블 직접 접근 불가, 함수만 실행)
```

```sql
-- 안전한 검색 Stored Procedure
CREATE OR REPLACE FUNCTION search_products(p_keyword VARCHAR)
RETURNS TABLE(id INT, name VARCHAR, price NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
  -- 입력 검증
  IF LENGTH(p_keyword) > 100 THEN
    RAISE EXCEPTION 'Keyword too long';
  END IF;

  -- LIKE 패턴 이스케이핑
  RETURN QUERY
  SELECT id, name, price
  FROM products
  WHERE name ILIKE '%' || replace(replace(p_keyword, '%', '\%'), '_', '\_') || '%'
  LIMIT 100;  -- 결과 제한
END;
$$;

-- 사용자에게 함수 실행 권한만 부여
GRANT EXECUTE ON FUNCTION search_products TO app_user;
REVOKE ALL ON TABLE products FROM app_user;
```

---

## 8. 실전 공격 및 방어 시나리오

### 💀 시나리오 1: 로그인 우회

#### 취약한 코드
```javascript
// login.js (취약)
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  const query = `
    SELECT * FROM users
    WHERE username = '${username}' AND password = '${password}'
  `;

  const result = await pool.query(query);

  if (result.rows.length > 0) {
    req.session.userId = result.rows[0].id;
    res.json({ success: true });
  } else {
    res.json({ success: false, message: 'Invalid credentials' });
  }
});
```

#### 공격
```bash
# 공격 요청
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin'\''--",
    "password": "anything"
  }'

# 생성되는 SQL:
# SELECT * FROM users
# WHERE username = 'admin'--' AND password = 'anything'

# 결과: admin으로 로그인 성공!
```

#### 방어
```javascript
// login.js (안전)
const bcrypt = require('bcrypt');

app.post('/login',
  body('username').isLength({ min: 3, max: 20 }).trim(),
  body('password').isLength({ min: 8, max: 100 }),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { username, password } = req.body;

    // Parameterized Query
    const result = await pool.query(
      'SELECT id, username, password_hash FROM users WHERE username = $1',
      [username]
    );

    if (result.rows.length === 0) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }

    const user = result.rows[0];

    // bcrypt로 비밀번호 비교
    const isValid = await bcrypt.compare(password, user.password_hash);

    if (!isValid) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }

    req.session.userId = user.id;
    res.json({ success: true });
  }
);
```

### 💀 시나리오 2: UNION 기반 데이터 유출

#### 공격
```bash
# 검색 기능 악용
curl "http://localhost:3000/search?keyword=%27%20UNION%20SELECT%20id,%20password_hash,%20email%20FROM%20users--"

# URL 디코딩:
# keyword: ' UNION SELECT id, password_hash, email FROM users--

# 생성되는 SQL:
# SELECT id, name, description FROM products
# WHERE name LIKE '%' UNION SELECT id, password_hash, email FROM users--%'

# 결과: 모든 사용자의 비밀번호 해시 유출!
```

#### 방어
```javascript
app.get('/search',
  query('keyword').isLength({ max: 50 }).trim(),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { keyword } = req.query;

    // LIKE 패턴 이스케이핑
    const escapedKeyword = keyword.replace(/[%_\\]/g, '\\$&');

    // Parameterized Query
    const result = await pool.query(
      "SELECT id, name, description FROM products WHERE name LIKE $1 ESCAPE '\\'",
      [`%${escapedKeyword}%`]
    );

    res.json(result.rows);
  }
);
```

### 💀 시나리오 3: Second-Order SQL Injection

```javascript
// Second-Order Injection: 저장된 데이터를 나중에 사용 시 발생

// 1단계: 악의적인 데이터 저장
app.post('/register', async (req, res) => {
  const { username } = req.body;

  // ✅ Parameterized Query로 저장 (안전)
  await pool.query(
    'INSERT INTO users (username) VALUES ($1)',
    ["admin' OR '1'='1"]  // 악의적인 username 저장
  );

  res.json({ success: true });
});

// 2단계: 저장된 데이터를 SQL에 직접 사용 (취약)
app.get('/user-posts/:username', async (req, res) => {
  const { username } = req.params;

  // 먼저 사용자 조회 (안전)
  const userResult = await pool.query(
    'SELECT username FROM users WHERE username = $1',
    [username]
  );

  if (userResult.rows.length === 0) {
    return res.status(404).json({ error: 'User not found' });
  }

  const foundUsername = userResult.rows[0].username;  // "admin' OR '1'='1"

  // ❌ DB에서 가져온 데이터를 직접 SQL에 삽입 (위험!)
  const postsQuery = `
    SELECT * FROM posts WHERE author = '${foundUsername}'
  `;

  const posts = await pool.query(postsQuery);
  // SQL Injection 발생!

  res.json(posts.rows);
});

// ✅ 올바른 방어: DB에서 가져온 데이터도 Parameterized Query 사용
app.get('/user-posts/:username', async (req, res) => {
  const { username } = req.params;

  // 사용자 ID로 조회 (username 대신)
  const userResult = await pool.query(
    'SELECT id FROM users WHERE username = $1',
    [username]
  );

  if (userResult.rows.length === 0) {
    return res.status(404).json({ error: 'User not found' });
  }

  const userId = userResult.rows[0].id;

  // user_id로 조회 (숫자이므로 안전)
  const posts = await pool.query(
    'SELECT * FROM posts WHERE user_id = $1',
    [userId]
  );

  res.json(posts.rows);
});
```

---

## 9. 보안 테스트 자동화

### 🧪 SQLMap을 사용한 취약점 스캔

```bash
# SQLMap 설치
sudo apt install sqlmap

# 1. GET 파라미터 테스트
sqlmap -u "http://localhost:3000/search?keyword=test" \
  --batch \
  --risk=3 \
  --level=5

# 2. POST 데이터 테스트
sqlmap -u "http://localhost:3000/login" \
  --data="username=admin&password=test" \
  --method=POST \
  --batch

# 3. 쿠키 포함 테스트
sqlmap -u "http://localhost:3000/profile" \
  --cookie="session=abc123" \
  --batch

# 결과 해석:
# [CRITICAL] SQL injection vulnerability found!
# → 취약점 발견, 코드 수정 필요

# [INFO] testing if GET parameter 'keyword' is dynamic
# [INFO] GET parameter 'keyword' appears to be dynamic
# [INFO] heuristic (basic) test shows that GET parameter 'keyword' might not be injectable
# → 취약점 없음 (안전)
```

### 🔍 Jest를 사용한 보안 테스트

```javascript
// __tests__/security.test.js
const request = require('supertest');
const app = require('../app');

describe('SQL Injection 방어 테스트', () => {
  const sqlInjectionPayloads = [
    "' OR '1'='1",
    "admin'--",
    "' UNION SELECT NULL--",
    "'; DROP TABLE users; --",
    "1' AND 1=2 UNION SELECT NULL, NULL, NULL--",
  ];

  test('로그인 SQL Injection 방어', async () => {
    for (const payload of sqlInjectionPayloads) {
      const res = await request(app)
        .post('/login')
        .send({ username: payload, password: 'test' });

      // 로그인 실패해야 함 (공격 차단)
      expect(res.body.success).toBe(false);

      // 500 에러가 아니어야 함 (에러 노출 방지)
      expect(res.status).not.toBe(500);
    }
  });

  test('검색 SQL Injection 방어', async () => {
    for (const payload of sqlInjectionPayloads) {
      const res = await request(app)
        .get('/search')
        .query({ keyword: payload });

      // 에러 없이 결과 반환해야 함
      expect(res.status).toBe(200);

      // 결과는 빈 배열이거나 정상 데이터
      expect(Array.isArray(res.body)).toBe(true);
    }
  });

  test('숫자 파라미터 타입 검증', async () => {
    const res = await request(app)
      .get('/user/abc123');  // 숫자가 아닌 값

    // 400 Bad Request 또는 404 Not Found
    expect([400, 404]).toContain(res.status);
  });
});
```

---

## 10. 보안 체크리스트

### ✅ SQL Injection 방어 체크리스트

```
[ ] Parameterized Query
    [ ] 모든 SQL에 $1, $2 등 파라미터 사용
    [ ] 문자열 연결(+, template literal) 사용 안 함
    [ ] ORM Raw Query도 파라미터 바인딩 사용

[ ] 입력 검증
    [ ] 타입 검증 (숫자는 parseInt)
    [ ] 길이 제한 (최대 길이 설정)
    [ ] 화이트리스트 (ORDER BY, 컬럼명)
    [ ] LIKE 패턴 이스케이핑

[ ] ORM 안전 사용
    [ ] ORM 메서드 우선 사용 (find, where)
    [ ] Raw Query 최소화
    [ ] 동적 컬럼명은 화이트리스트 검증

[ ] 에러 처리
    [ ] SQL 에러 메시지 클라이언트 노출 금지
    [ ] 일반적인 에러 메시지만 반환
    [ ] 상세 에러는 서버 로그만

[ ] 권한 관리
    [ ] 애플리케이션 사용자는 최소 권한
    [ ] DDL 권한 없음 (DROP, ALTER 불가)
    [ ] 민감 테이블 접근 제한

[ ] 테스트
    [ ] 자동화된 보안 테스트 작성
    [ ] SQLMap 스캔 정기 실행
    [ ] 코드 리뷰 시 보안 체크

[ ] 모니터링
    [ ] 의심스러운 쿼리 패턴 감지
    [ ] 실패한 로그인 시도 추적
    [ ] SQL 에러 로그 모니터링
```

---

## 11. 실습 과제

### 🎯 과제 1: 취약점 찾기
다음 코드에서 SQL Injection 취약점을 찾고 수정하세요:

```javascript
// 취약한 코드
app.get('/products', async (req, res) => {
  const { category, minPrice, maxPrice, sortBy } = req.query;

  let query = 'SELECT * FROM products WHERE 1=1';

  if (category) {
    query += ` AND category = '${category}'`;
  }

  if (minPrice) {
    query += ` AND price >= ${minPrice}`;
  }

  if (maxPrice) {
    query += ` AND price <= ${maxPrice}`;
  }

  if (sortBy) {
    query += ` ORDER BY ${sortBy}`;
  }

  const result = await pool.query(query);
  res.json(result.rows);
});

// 여기에 안전한 버전을 작성하세요
```

### 🎯 과제 2: 보안 테스트 작성
위 코드에 대한 보안 테스트를 작성하세요:

```javascript
describe('Products API 보안 테스트', () => {
  // 여기에 테스트 케이스 작성
});
```

### 🎯 과제 3: Stored Procedure 활용
안전한 제품 검색 Stored Procedure를 작성하세요:

```sql
-- 요구사항:
-- 1. 키워드로 제품 검색
-- 2. 카테고리 필터 (선택)
-- 3. 가격 범위 (선택)
-- 4. 정렬 (name, price, created_at)
-- 5. 페이지네이션 (limit, offset)
-- 6. 모든 입력 검증 포함

CREATE OR REPLACE FUNCTION search_products_safe(
  -- 파라미터 정의
)
RETURNS TABLE(...)
LANGUAGE plpgsql
AS $$
BEGIN
  -- 여기에 작성하세요
END;
$$;
```

---

## 12. 다음 단계

SQL Injection 방어를 마스터했으니, Part 9: 보안이 완료되었습니다!

이제 Part 10: 복제와 고가용성으로 넘어갑니다!

### 다음 챕터 미리보기
```
Chapter 34: Replication 개요
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 배울 내용:
   ├─ Replication의 개념과 종류
   ├─ Primary-Standby 구조
   ├─ Synchronous vs Asynchronous
   ├─ Streaming Replication 원리
   ├─ Logical Replication vs Physical
   └─ 고가용성 아키텍처 설계

💡 실습:
   ├─ 단일 Standby 서버 구축
   ├─ 복제 지연 모니터링
   ├─ 복제 슬롯 관리
   ├─ Cascading Replication
   └─ 읽기 분산 로드밸런싱
```

---

## 📖 정리

이번 챕터에서 배운 내용:

| 개념 | 핵심 내용 |
|------|----------|
| **SQL Injection** | 사용자 입력을 SQL에 직접 삽입하여 발생하는 취약점 |
| **Parameterized Query** | SQL 구조와 데이터를 분리하는 안전한 방법 ⭐ |
| **ORM 보안** | Raw Query도 파라미터 바인딩 필수 |
| **입력 검증** | 타입, 길이, 화이트리스트 검증 |
| **에러 처리** | SQL 에러 메시지 클라이언트 노출 금지 |
| **다층 방어** | 입력 검증 + Parameterized Query + 최소 권한 |

### 핵심 원칙

```javascript
// ❌ 절대 하지 말 것
const query = `SELECT * FROM users WHERE id = ${userId}`;
const query = `SELECT * FROM users WHERE name = '${name}'`;

// ✅ 항상 이렇게
const query = 'SELECT * FROM users WHERE id = $1';
pool.query(query, [userId]);

const query = 'SELECT * FROM users WHERE name = $1';
pool.query(query, [name]);
```

**기억하세요**: "Parameterized Query는 선택이 아닌 필수입니다. 모든 동적 SQL에는 반드시 파라미터 바인딩을 사용하세요. 이것이 SQL Injection을 방어하는 가장 확실한 방법입니다!"

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐⭐ (고급)
**학습 시간**: 120분
**대상**: 백엔드 개발자, 보안 담당자, 풀스택 개발자
**중요도**: 🔥🔥🔥🔥🔥 (매우 높음)

이전: [Chapter 32. 인증과 암호화](32-인증과-암호화.md)
다음: [Chapter 34. Replication 개요](../part10-복제와-고가용성/34-replication-개요.md)