# Chapter 44. Docker로 PostgreSQL

## 🎯 이 챕터의 목표
- **Docker**로 PostgreSQL 실행하기
- **docker-compose**로 환경 구성하기
- **볼륨 마운트**로 데이터 영속화하기
- **환경 변수** 설정하기
- **멀티 컨테이너** 애플리케이션 구축하기

---

## 1. 기본 PostgreSQL 컨테이너

```bash
# PostgreSQL 이미지 다운로드
docker pull postgres:14

# 컨테이너 실행
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -d postgres:14

# 연결 확인
docker exec -it my-postgres psql -U postgres -d mydb

# 컨테이너 중지/시작/삭제
docker stop my-postgres
docker start my-postgres
docker rm my-postgres
```

---

## 2. 볼륨 마운트 (데이터 영속화)

```bash
# 볼륨 생성
docker volume create postgres-data

# 볼륨 사용
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  -d postgres:14

# 컨테이너 삭제해도 데이터 유지!
docker rm -f my-postgres
docker run --name new-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  -d postgres:14
# 기존 데이터 그대로 있음!
```

---

## 3. docker-compose (권장)

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14
    container_name: my-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: mysecretpassword
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: .
    container_name: my-app
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://postgres:mysecretpassword@db:5432/mydb
    ports:
      - "3000:3000"

volumes:
  postgres-data:
```

```bash
# 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f db

# 중지 및 삭제
docker-compose down

# 볼륨까지 삭제
docker-compose down -v
```

---

## 4. 초기 데이터 설정

```sql
-- init.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com');

CREATE INDEX idx_users_email ON users(email);
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:14
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```

---

## 5. 커스텀 설정

```ini
# custom-postgresql.conf
max_connections = 200
shared_buffers = 256MB
effective_cache_size = 1GB
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:14
    volumes:
      - ./custom-postgresql.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
```

---

## 6. 실전 예제: Node.js + PostgreSQL

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/mydb
    ports:
      - "3000:3000"

volumes:
  postgres-data:
```

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

```javascript
// app.js
const express = require('express');
const { Pool } = require('pg');

const app = express();
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

app.get('/users', async (req, res) => {
  const result = await pool.query('SELECT * FROM users');
  res.json(result.rows);
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

---

## 7. 백업 및 복원

```bash
# 백업
docker exec my-postgres pg_dump -U postgres mydb > backup.sql

# 복원
docker exec -i my-postgres psql -U postgres mydb < backup.sql

# docker-compose에서
docker-compose exec db pg_dump -U postgres mydb > backup.sql
docker-compose exec -T db psql -U postgres mydb < backup.sql
```

---

## 📖 정리

| 명령 | 용도 |
|------|----------|
| **docker run** | 단일 컨테이너 실행 |
| **docker-compose** | 멀티 컨테이너 관리 |
| **volume** | 데이터 영속화 |
| **init.sql** | 초기 스키마 생성 |

### 베스트 프랙티스

```
✅ docker-compose 사용
✅ 볼륨으로 데이터 영속화
✅ healthcheck 설정
✅ 환경 변수로 설정 관리
✅ .env 파일로 비밀정보 분리
```

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐ (초급)
**학습 시간**: 60분

이전: [Chapter 43. 마이그레이션 도구](43-마이그레이션-도구.md)
다음: [Chapter 45. AWS RDS PostgreSQL](45-aws-rds-postgresql.md)