# Chapter 41. Node.js 연동

## 🎯 이 챕터의 목표
- **node-postgres (pg)** 라이브러리 활용하기
- **커넥션 풀** 관리하기
- **트랜잭션** 처리하기
- **Prepared Statement** 사용하기
- **실전 CRUD API** 구현하기

---

## 1. 환경 설정

```bash
# 프로젝트 초기화
mkdir my-app && cd my-app
npm init -y

# pg 라이브러리 설치
npm install pg

# TypeScript 사용 시
npm install --save-dev @types/node @types/pg typescript
```

---

## 2. 기본 연결

```javascript
// db.js
const { Client } = require('pg');

// 단일 연결 (테스트용)
const client = new Client({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'postgres',
  password: 'password',
});

async function main() {
  await client.connect();

  const result = await client.query('SELECT NOW()');
  console.log(result.rows[0]);

  await client.end();
}

main();
```

---

## 3. 커넥션 풀 (권장)

```javascript
// db.js
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'postgres',
  password: 'password',
  max: 20,                    // 최대 연결 수
  idleTimeoutMillis: 30000,   // 유휴 연결 타임아웃
  connectionTimeoutMillis: 2000,
});

// 쿼리 실행
async function getUsers() {
  const result = await pool.query('SELECT * FROM users');
  return result.rows;
}

// 연결 해제 (앱 종료 시)
process.on('SIGINT', async () => {
  await pool.end();
  process.exit(0);
});

module.exports = { pool };
```

---

## 4. CRUD 연산

```javascript
// users.js
const { pool } = require('./db');

// CREATE
async function createUser(name, email) {
  const result = await pool.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
    [name, email]
  );
  return result.rows[0];
}

// READ
async function getUser(id) {
  const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [id]
  );
  return result.rows[0];
}

// UPDATE
async function updateUser(id, name, email) {
  const result = await pool.query(
    'UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *',
    [name, email, id]
  );
  return result.rows[0];
}

// DELETE
async function deleteUser(id) {
  await pool.query('DELETE FROM users WHERE id = $1', [id]);
}

module.exports = { createUser, getUser, updateUser, deleteUser };
```

---

## 5. 트랜잭션

```javascript
// transaction.js
const { pool } = require('./db');

async function transferMoney(fromUserId, toUserId, amount) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // 출금
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE user_id = $2',
      [amount, fromUserId]
    );

    // 입금
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE user_id = $2',
      [amount, toUserId]
    );

    await client.query('COMMIT');
    console.log('Transfer successful');
  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Transfer failed:', error);
    throw error;
  } finally {
    client.release();
  }
}
```

---

## 6. Express API 예제

```javascript
// app.js
const express = require('express');
const { pool } = require('./db');

const app = express();
app.use(express.json());

// GET /users
app.get('/users', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM users');
    res.json(result.rows);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET /users/:id
app.get('/users/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// POST /users
app.post('/users', async (req, res) => {
  try {
    const { name, email } = req.body;
    const result = await pool.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [name, email]
    );
    res.status(201).json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// PUT /users/:id
app.put('/users/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { name, email } = req.body;
    const result = await pool.query(
      'UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *',
      [name, email, id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// DELETE /users/:id
app.delete('/users/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(
      'DELETE FROM users WHERE id = $1 RETURNING *',
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## 📖 정리

| 개념 | 핵심 내용 |
|------|----------|
| **Pool** | 커넥션 재사용, 성능 향상 |
| **Parameterized Query** | SQL Injection 방어 ($1, $2) |
| **Transaction** | BEGIN, COMMIT, ROLLBACK |
| **client.release()** | 반드시 연결 반환! |

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐ (초급)
**학습 시간**: 60분

이전: [Chapter 40. VACUUM과 ANALYZE](../part11-모니터링과-튜닝/40-vacuum과-analyze.md)
다음: [Chapter 42. ORM](42-orm.md)
