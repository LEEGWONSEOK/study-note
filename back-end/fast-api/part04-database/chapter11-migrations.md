# Chapter 11. 마이그레이션 (Alembic)

## 11.1 Alembic이란?

Alembic은 SQLAlchemy를 위한 데이터베이스 마이그레이션 도구입니다. Spring Boot의 Flyway나 Liquibase와 유사한 역할을 합니다.

### Spring Boot vs Alembic

| 기능 | Spring Boot (Flyway) | FastAPI (Alembic) |
|------|---------------------|-------------------|
| **마이그레이션 파일** | SQL 파일 (V1__create_table.sql) | Python 파일 (revision_id_*.py) |
| **자동 생성** | 수동 작성 | `alembic revision --autogenerate` |
| **실행** | 자동 (앱 시작 시) | `alembic upgrade head` |
| **롤백** | 파일 삭제 후 재실행 | `alembic downgrade -1` |
| **히스토리** | flyway_schema_history 테이블 | alembic_version 테이블 |

---

## 11.2 설치 및 초기 설정

### Alembic 설치

```bash
pip install alembic
```

### Alembic 초기화

```bash
# 프로젝트 루트에서 실행
alembic init alembic

# 또는 특정 디렉토리에
alembic init migrations
```

**생성되는 파일 구조:**
```
project/
├── alembic/
│   ├── versions/          # 마이그레이션 파일들
│   ├── env.py            # 환경 설정
│   ├── README
│   └── script.py.mako    # 템플릿
├── alembic.ini           # Alembic 설정 파일
└── app/
    ├── models/
    └── database.py
```

---

## 11.3 Alembic 설정

### alembic.ini 수정

```ini
# alembic.ini
[alembic]
script_location = alembic
prepend_sys_path = .

# 데이터베이스 URL
sqlalchemy.url = postgresql://user:password@localhost/dbname

# 또는 환경 변수 사용 (권장)
# sqlalchemy.url =

# 로그 설정
[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

### env.py 수정

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
import os
import sys

# 프로젝트 루트를 Python 경로에 추가
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

# 모델 import
from app.database import Base
from app.models.user import User  # 모든 모델 import 필요
from app.models.product import Product

# Alembic Config 객체
config = context.config

# 환경 변수에서 DB URL 가져오기
database_url = os.getenv("DATABASE_URL", "postgresql://user:password@localhost/dbname")
config.set_main_option("sqlalchemy.url", database_url)

# 로깅 설정
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# 메타데이터 설정
target_metadata = Base.metadata

def run_migrations_offline() -> None:
    """오프라인 모드에서 마이그레이션 실행"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    """온라인 모드에서 마이그레이션 실행"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

---

## 11.4 마이그레이션 파일 생성

### 자동 생성 (autogenerate)

```bash
# 모델 변경사항을 자동으로 감지하여 마이그레이션 파일 생성
alembic revision --autogenerate -m "create users table"

# 생성된 파일: alembic/versions/xxxx_create_users_table.py
```

### 수동 생성

```bash
# 빈 마이그레이션 파일 생성
alembic revision -m "add custom index"
```

### 생성된 마이그레이션 파일

```python
# alembic/versions/xxxx_create_users_table.py
"""create users table

Revision ID: xxxx
Revises:
Create Date: 2024-01-01 00:00:00.000000

"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision: str = 'xxxx'
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def upgrade() -> None:
    """테이블 생성 (마이그레이션 적용)"""
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('username', sa.String(length=50), nullable=False),
        sa.Column('email', sa.String(length=100), nullable=False),
        sa.Column('hashed_password', sa.String(length=255), nullable=False),
        sa.Column('is_active', sa.Boolean(), nullable=True),
        sa.Column('created_at', sa.DateTime(), nullable=True),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_users_email'), 'users', ['email'], unique=True)
    op.create_index(op.f('ix_users_username'), 'users', ['username'], unique=True)

def downgrade() -> None:
    """테이블 삭제 (마이그레이션 롤백)"""
    op.drop_index(op.f('ix_users_username'), table_name='users')
    op.drop_index(op.f('ix_users_email'), table_name='users')
    op.drop_table('users')
```

**Spring Boot Flyway와 비교:**
```sql
-- Spring Boot Flyway
-- V1__create_users_table.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_username ON users(username);
```

---

## 11.5 마이그레이션 실행

### 최신 버전으로 업그레이드

```bash
# 최신 버전으로 업그레이드
alembic upgrade head

# 특정 버전으로 업그레이드
alembic upgrade xxxx

# 상대적 업그레이드 (+1 버전)
alembic upgrade +1

# 특정 개수만큼 업그레이드
alembic upgrade +2
```

### 다운그레이드 (롤백)

```bash
# 한 단계 다운그레이드
alembic downgrade -1

# 두 단계 다운그레이드
alembic downgrade -2

# 특정 버전으로 다운그레이드
alembic downgrade xxxx

# 베이스로 완전 롤백
alembic downgrade base
```

### 현재 버전 확인

```bash
# 현재 데이터베이스 버전 확인
alembic current

# 마이그레이션 히스토리 확인
alembic history

# 상세 히스토리
alembic history --verbose
```

---

## 11.6 마이그레이션 작업 예제

### 컬럼 추가

```python
def upgrade() -> None:
    op.add_column('users', sa.Column('phone', sa.String(length=20), nullable=True))

def downgrade() -> None:
    op.drop_column('users', 'phone')
```

### 컬럼 수정

```python
def upgrade() -> None:
    # 컬럼 타입 변경
    op.alter_column('users', 'phone',
        existing_type=sa.String(length=20),
        type_=sa.String(length=30),
        nullable=False
    )

def downgrade() -> None:
    op.alter_column('users', 'phone',
        existing_type=sa.String(length=30),
        type_=sa.String(length=20),
        nullable=True
    )
```

### 컬럼 삭제

```python
def upgrade() -> None:
    op.drop_column('users', 'old_field')

def downgrade() -> None:
    op.add_column('users', sa.Column('old_field', sa.String(), nullable=True))
```

### 인덱스 추가/삭제

```python
def upgrade() -> None:
    op.create_index('idx_users_created_at', 'users', ['created_at'])

def downgrade() -> None:
    op.drop_index('idx_users_created_at', table_name='users')
```

### 외래키 추가

```python
def upgrade() -> None:
    op.add_column('posts', sa.Column('author_id', sa.Integer(), nullable=True))
    op.create_foreign_key(
        'fk_posts_author',
        'posts', 'users',
        ['author_id'], ['id']
    )

def downgrade() -> None:
    op.drop_constraint('fk_posts_author', 'posts', type_='foreignkey')
    op.drop_column('posts', 'author_id')
```

### 데이터 마이그레이션

```python
from alembic import op
from sqlalchemy.sql import table, column
from sqlalchemy import String, Integer

def upgrade() -> None:
    # 테이블 생성
    op.create_table(
        'roles',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(length=50), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )

    # 초기 데이터 삽입
    roles_table = table('roles',
        column('id', Integer),
        column('name', String)
    )

    op.bulk_insert(roles_table, [
        {'id': 1, 'name': 'admin'},
        {'id': 2, 'name': 'user'},
        {'id': 3, 'name': 'guest'}
    ])

def downgrade() -> None:
    op.drop_table('roles')
```

---

## 11.7 실전 활용

### 개발 워크플로우

```bash
# 1. 모델 수정
# app/models/user.py에서 User 모델에 phone 필드 추가

# 2. 마이그레이션 파일 생성
alembic revision --autogenerate -m "add phone to users"

# 3. 생성된 파일 확인 및 수정
# alembic/versions/xxxx_add_phone_to_users.py 확인

# 4. 마이그레이션 적용
alembic upgrade head

# 5. 문제 발생 시 롤백
alembic downgrade -1
```

### 팀 협업 시나리오

```bash
# 개발자 A: 새 마이그레이션 생성 및 커밋
alembic revision --autogenerate -m "add users table"
git add alembic/versions/*.py
git commit -m "Add users table migration"
git push

# 개발자 B: 변경사항 pull 후 마이그레이션 적용
git pull
alembic upgrade head
```

### 프로덕션 배포

```bash
# 1. 현재 버전 백업
alembic current > /backup/current_version.txt

# 2. 데이터베이스 백업
pg_dump dbname > /backup/db_backup.sql

# 3. 마이그레이션 적용
alembic upgrade head

# 4. 문제 발생 시 롤백
alembic downgrade -1
# 또는 데이터베이스 복구
psql dbname < /backup/db_backup.sql
```

---

## 11.8 환경별 설정

### config.py

```python
# app/config.py
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str = "postgresql://user:password@localhost/dbname"

    class Config:
        env_file = ".env"

settings = Settings()
```

### .env 파일

```bash
# 개발 환경
DATABASE_URL=postgresql://user:password@localhost/dev_db

# 프로덕션 환경
# DATABASE_URL=postgresql://user:password@prod-host/prod_db
```

### env.py에서 설정 사용

```python
# alembic/env.py
from app.config import settings

# 환경별 DB URL 사용
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
```

---

## 11.9 자주 사용하는 명령어

```bash
# 마이그레이션 생성
alembic revision --autogenerate -m "message"
alembic revision -m "message"  # 빈 파일

# 마이그레이션 적용
alembic upgrade head        # 최신 버전
alembic upgrade +1          # 한 단계
alembic upgrade xxxx        # 특정 버전

# 마이그레이션 롤백
alembic downgrade -1        # 한 단계
alembic downgrade base      # 전체 롤백
alembic downgrade xxxx      # 특정 버전

# 정보 확인
alembic current             # 현재 버전
alembic history             # 히스토리
alembic history --verbose   # 상세 히스토리
alembic show xxxx          # 특정 마이그레이션 내용

# 마이그레이션 병합 (브랜치 충돌 시)
alembic merge -m "merge message" head1 head2
```

---

## 11.10 Best Practices

### 1. 자동 생성 후 반드시 검토

```python
# autogenerate가 감지 못하는 경우:
# - 테이블 이름 변경
# - 컬럼 이름 변경
# - 제약조건 일부 변경

# 자동 생성 후 항상 파일을 열어서 확인!
```

### 2. 다운그레이드 함수 작성

```python
# 업그레이드만 작성하지 말고 다운그레이드도 작성
def upgrade() -> None:
    op.create_table('users', ...)

def downgrade() -> None:
    op.drop_table('users')  # 반드시 작성!
```

### 3. 데이터 마이그레이션은 신중하게

```python
# 대량 데이터 마이그레이션 시 배치 처리
def upgrade() -> None:
    connection = op.get_bind()

    # 배치로 처리
    batch_size = 1000
    offset = 0

    while True:
        users = connection.execute(
            f"SELECT * FROM users LIMIT {batch_size} OFFSET {offset}"
        ).fetchall()

        if not users:
            break

        # 데이터 처리
        for user in users:
            # ...
            pass

        offset += batch_size
```

### 4. 트랜잭션 사용

```python
def upgrade() -> None:
    # 트랜잭션 안에서 실행됨
    # 에러 발생 시 자동 롤백
    op.create_table('users', ...)
    op.create_index('idx_users_email', ...)
```

---

## 11.11 핵심 요약

### Alembic vs Flyway 비교

| 기능 | Flyway | Alembic |
|------|--------|---------|
| **파일 형식** | SQL | Python |
| **자동 생성** | ❌ | ✅ (autogenerate) |
| **롤백** | 어려움 | 쉬움 (downgrade) |
| **데이터 마이그레이션** | SQL로 작성 | Python으로 작성 |
| **버전 관리** | 파일명 (V1__, V2__) | revision ID |
| **실행 시점** | 앱 시작 시 자동 | 수동 실행 |

### 기억할 핵심

1. ✅ **autogenerate**: 모델 변경 자동 감지
2. ✅ **upgrade/downgrade**: 양방향 마이그레이션
3. ✅ **revision**: 각 마이그레이션의 고유 ID
4. ✅ **head**: 최신 마이그레이션
5. ✅ **base**: 초기 상태

---

## 11.12 실습 예제

### 실습 1: 첫 마이그레이션

```bash
# 1. Alembic 초기화
alembic init alembic

# 2. env.py 수정 (모델 import)

# 3. User 모델 생성

# 4. 마이그레이션 파일 생성
alembic revision --autogenerate -m "create users table"

# 5. 마이그레이션 적용
alembic upgrade head

# 6. 확인
alembic current
```

---

## 11.13 다음 챕터 예고

다음 챕터에서는 **고급 데이터베이스 기능**을 다룹니다:
- Relationship (OneToMany, ManyToMany)
- Lazy Loading vs Eager Loading
- 트랜잭션 관리
- 데이터베이스 풀링

---

[← 이전: Chapter 10. 데이터베이스 CRUD 구현](chapter10-crud.md) | [목차로](../README.md) | [다음: Chapter 12. 고급 데이터베이스 기능 →](chapter12-advanced-db.md)