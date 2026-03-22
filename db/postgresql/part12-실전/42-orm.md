# Chapter 42. ORM (Sequelize, TypeORM)

## 🎯 이 챕터의 목표
- **Sequelize** 모델 정의 및 사용하기
- **TypeORM** 엔티티 및 데코레이터 활용하기
- **관계 매핑** (1:N, N:M) 구현하기
- **마이그레이션** 자동 생성하기
- **ORM vs Raw SQL** 비교하기

---

## 1. Sequelize (JavaScript ORM)

### 설치 및 초기화

```bash
npm install sequelize pg pg-hstore
npm install --save-dev sequelize-cli

# 초기화
npx sequelize-cli init
```

### 모델 정의

```javascript
// models/user.js
const { Model, DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  class User extends Model {}

  User.init({
    id: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    name: {
      type: DataTypes.STRING(100),
      allowNull: false,
    },
    email: {
      type: DataTypes.STRING(100),
      allowNull: false,
      unique: true,
    },
  }, {
    sequelize,
    modelName: 'User',
    tableName: 'users',
    timestamps: true,
  });

  return User;
};
```

### CRUD 연산

```javascript
// app.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize('mydb', 'postgres', 'password', {
  host: 'localhost',
  dialect: 'postgres',
});

const User = require('./models/user')(sequelize);

// CREATE
const user = await User.create({
  name: 'Alice',
  email: 'alice@example.com',
});

// READ
const users = await User.findAll();
const user = await User.findByPk(1);
const alice = await User.findOne({ where: { email: 'alice@example.com' } });

// UPDATE
await User.update(
  { name: 'Bob' },
  { where: { id: 1 } }
);

// DELETE
await User.destroy({ where: { id: 1 } });
```

### 관계 정의

```javascript
// models/post.js
class Post extends Model {}

Post.init({
  title: DataTypes.STRING,
  content: DataTypes.TEXT,
  userId: DataTypes.INTEGER,
}, { sequelize });

// 관계 설정
User.hasMany(Post, { foreignKey: 'userId' });
Post.belongsTo(User, { foreignKey: 'userId' });

// 사용
const user = await User.findByPk(1, {
  include: [Post],
});

console.log(user.Posts);  // 연관된 게시물
```

---

## 2. TypeORM (TypeScript ORM)

### 설치 및 초기화

```bash
npm install typeorm pg reflect-metadata
npm install --save-dev typescript @types/node

# tsconfig.json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "commonjs",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

### 엔티티 정의

```typescript
// entities/User.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Post } from './Post';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 100 })
  name: string;

  @Column({ length: 100, unique: true })
  email: string;

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date;

  @OneToMany(() => Post, post => post.user)
  posts: Post[];
}
```

### 데이터소스 설정

```typescript
// data-source.ts
import { DataSource } from 'typeorm';
import { User } from './entities/User';
import { Post } from './entities/Post';

export const AppDataSource = new DataSource({
  type: 'postgres',
  host: 'localhost',
  port: 5432,
  username: 'postgres',
  password: 'password',
  database: 'mydb',
  synchronize: false,  // 프로덕션에서는 false
  logging: true,
  entities: [User, Post],
  migrations: ['src/migrations/*.ts'],
});
```

### CRUD 연산

```typescript
// app.ts
import { AppDataSource } from './data-source';
import { User } from './entities/User';

await AppDataSource.initialize();

const userRepository = AppDataSource.getRepository(User);

// CREATE
const user = userRepository.create({
  name: 'Alice',
  email: 'alice@example.com',
});
await userRepository.save(user);

// READ
const users = await userRepository.find();
const user = await userRepository.findOne({ where: { id: 1 } });
const alice = await userRepository.findOne({ where: { email: 'alice@example.com' } });

// UPDATE
await userRepository.update({ id: 1 }, { name: 'Bob' });

// DELETE
await userRepository.delete({ id: 1 });
```

### Query Builder

```typescript
// 복잡한 쿼리
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .where('user.email LIKE :email', { email: '%@example.com' })
  .orderBy('user.createdAt', 'DESC')
  .take(10)
  .getMany();
```

---

## 3. 마이그레이션

### Sequelize 마이그레이션

```bash
# 마이그레이션 생성
npx sequelize-cli migration:generate --name create-users

# 실행
npx sequelize-cli db:migrate

# 롤백
npx sequelize-cli db:migrate:undo
```

```javascript
// migrations/20240115-create-users.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('users', {
      id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true,
      },
      name: {
        type: Sequelize.STRING(100),
        allowNull: false,
      },
      email: {
        type: Sequelize.STRING(100),
        allowNull: false,
        unique: true,
      },
      createdAt: {
        type: Sequelize.DATE,
        allowNull: false,
      },
      updatedAt: {
        type: Sequelize.DATE,
        allowNull: false,
      },
    });
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('users');
  },
};
```

### TypeORM 마이그레이션

```bash
# 마이그레이션 생성
npx typeorm migration:create src/migrations/CreateUsers

# 실행
npx typeorm migration:run -d src/data-source.ts

# 롤백
npx typeorm migration:revert -d src/data-source.ts
```

---

## 4. ORM vs Raw SQL

```javascript
// ORM (Sequelize)
const users = await User.findAll({
  where: {
    email: {
      [Op.like]: '%@example.com'
    }
  },
  include: [Post],
  order: [['createdAt', 'DESC']],
  limit: 10,
});

// Raw SQL
const { pool } = require('pg');
const result = await pool.query(`
  SELECT u.*, json_agg(p.*) AS posts
  FROM users u
  LEFT JOIN posts p ON u.id = p.user_id
  WHERE u.email LIKE $1
  GROUP BY u.id
  ORDER BY u.created_at DESC
  LIMIT 10
`, ['%@example.com']);
```

### 장단점

```
ORM:
✅ 타입 안전성
✅ 코드 재사용성
✅ 자동 마이그레이션
❌ 복잡한 쿼리 어려움
❌ 성능 오버헤드

Raw SQL:
✅ 완전한 제어
✅ 최적 성능
✅ 복잡한 쿼리 가능
❌ SQL Injection 위험
❌ 타입 안전성 부족
```

---

## 📖 정리

| ORM | 특징 |
|------|----------|
| **Sequelize** | JavaScript, Promise 기반, 성숙함 |
| **TypeORM** | TypeScript, 데코레이터, Active Record |
| **Prisma** | 최신, 타입 안전, 마이그레이션 우수 |

---

**작성일**: 2024-01-15
**PostgreSQL 버전**: 14+
**난이도**: ⭐⭐⭐ (중급)
**학습 시간**: 90분

이전: [Chapter 41. Node.js 연동](41-nodejs-연동.md)
다음: [Chapter 43. 마이그레이션 도구](43-마이그레이션-도구.md)