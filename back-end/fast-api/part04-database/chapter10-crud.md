# Chapter 10. 데이터베이스 CRUD 구현

## 10.1 Repository 패턴

Repository 패턴은 데이터 접근 로직을 캡슐화하는 디자인 패턴입니다. Spring Data JPA의 Repository와 동일한 개념입니다.

### Spring Boot vs FastAPI Repository

```java
// Spring Boot
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByIsActiveTrue();
    Page<User> findAll(Pageable pageable);
}
```

```python
# FastAPI
from sqlalchemy.orm import Session
from typing import Optional, List
from models.user import User

class UserRepository:
    def __init__(self, db: Session):
        self.db = db

    def find_by_id(self, user_id: int) -> Optional[User]:
        return self.db.query(User).filter(User.id == user_id).first()

    def find_by_email(self, email: str) -> Optional[User]:
        return self.db.query(User).filter(User.email == email).first()

    def find_active_users(self) -> List[User]:
        return self.db.query(User).filter(User.is_active == True).all()

    def find_all(self, skip: int = 0, limit: int = 10) -> List[User]:
        return self.db.query(User).offset(skip).limit(limit).all()
```

---

## 10.2 Generic Repository 구현

재사용 가능한 제네릭 Repository를 만들 수 있습니다.

### 기본 Generic Repository

```python
from typing import TypeVar, Generic, Type, Optional, List
from sqlalchemy.orm import Session
from pydantic import BaseModel
from database import Base

ModelType = TypeVar("ModelType", bound=Base)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)

class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    def __init__(self, model: Type[ModelType]):
        self.model = model

    def get(self, db: Session, id: int) -> Optional[ModelType]:
        return db.query(self.model).filter(self.model.id == id).first()

    def get_multi(
        self, db: Session, *, skip: int = 0, limit: int = 100
    ) -> List[ModelType]:
        return db.query(self.model).offset(skip).limit(limit).all()

    def create(self, db: Session, *, obj_in: CreateSchemaType) -> ModelType:
        obj_in_data = obj_in.dict()
        db_obj = self.model(**obj_in_data)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def update(
        self, db: Session, *, db_obj: ModelType, obj_in: UpdateSchemaType
    ) -> ModelType:
        obj_data = obj_in.dict(exclude_unset=True)
        for field, value in obj_data.items():
            setattr(db_obj, field, value)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def delete(self, db: Session, *, id: int) -> Optional[ModelType]:
        obj = db.query(self.model).get(id)
        if obj:
            db.delete(obj)
            db.commit()
        return obj
```

### 사용 예제

```python
from models.user import User
from schemas.user import UserCreate, UserUpdate

class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
    def get_by_email(self, db: Session, *, email: str) -> Optional[User]:
        return db.query(User).filter(User.email == email).first()

    def get_by_username(self, db: Session, *, username: str) -> Optional[User]:
        return db.query(User).filter(User.username == username).first()

    def get_active_users(self, db: Session) -> List[User]:
        return db.query(User).filter(User.is_active == True).all()

# 인스턴스 생성
user_crud = CRUDUser(User)

# 사용
@app.post("/users", response_model=UserResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # 이메일 중복 체크
    if user_crud.get_by_email(db, email=user.email):
        raise HTTPException(status_code=400, detail="Email already exists")

    return user_crud.create(db, obj_in=user)
```

**Spring Boot와 비교:**
```java
// Spring Boot - JpaRepository가 기본 CRUD 제공
public interface UserRepository extends JpaRepository<User, Long> {
    // 기본 메서드: save(), findById(), findAll(), delete() 등 자동 제공
    Optional<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}
```

---

## 10.3 복잡한 쿼리 작성

### JOIN 쿼리

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

# 모델 정의
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50))

    # 관계 정의
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    content = Column(String)
    author_id = Column(Integer, ForeignKey("users.id"))

    # 관계 정의
    author = relationship("User", back_populates="posts")

# JOIN 쿼리
def get_posts_with_authors(db: Session):
    # INNER JOIN
    posts = db.query(Post).join(User).all()

    # LEFT JOIN
    posts = db.query(Post).outerjoin(User).all()

    # 특정 컬럼만 선택
    results = db.query(Post.title, User.username).join(User).all()

    return posts
```

**Spring Boot JPQL과 비교:**
```java
// Spring Boot
@Query("SELECT p FROM Post p JOIN p.author")
List<Post> findPostsWithAuthors();

@Query("SELECT p FROM Post p LEFT JOIN p.author")
List<Post> findPostsWithAuthorsLeftJoin();

@Query("SELECT p.title, u.username FROM Post p JOIN p.author u")
List<Object[]> findPostTitlesWithUsernames();
```

### 서브쿼리

```python
from sqlalchemy import func

# 가장 많은 게시글을 작성한 사용자
def get_most_active_users(db: Session, limit: int = 10):
    subquery = (
        db.query(
            Post.author_id,
            func.count(Post.id).label("post_count")
        )
        .group_by(Post.author_id)
        .subquery()
    )

    users = (
        db.query(User, subquery.c.post_count)
        .join(subquery, User.id == subquery.c.author_id)
        .order_by(subquery.c.post_count.desc())
        .limit(limit)
        .all()
    )

    return users
```

### EXISTS 쿼리

```python
from sqlalchemy import exists

# 게시글이 있는 사용자만 조회
def get_users_with_posts(db: Session):
    stmt = exists().where(Post.author_id == User.id)
    users = db.query(User).filter(stmt).all()
    return users
```

---

## 10.4 필터링과 검색

### 동적 필터링

```python
from typing import Optional

def search_users(
    db: Session,
    username: Optional[str] = None,
    email: Optional[str] = None,
    is_active: Optional[bool] = None,
    skip: int = 0,
    limit: int = 10
):
    query = db.query(User)

    # 동적으로 필터 추가
    if username:
        query = query.filter(User.username.ilike(f"%{username}%"))

    if email:
        query = query.filter(User.email.ilike(f"%{email}%"))

    if is_active is not None:
        query = query.filter(User.is_active == is_active)

    # 페이지네이션
    users = query.offset(skip).limit(limit).all()
    total = query.count()

    return {"items": users, "total": total}
```

**Spring Boot Specification과 비교:**
```java
// Spring Boot
public class UserSpecification {
    public static Specification<User> hasUsername(String username) {
        return (root, query, cb) ->
            username == null ? null :
            cb.like(cb.lower(root.get("username")), "%" + username.toLowerCase() + "%");
    }

    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) ->
            email == null ? null :
            cb.like(cb.lower(root.get("email")), "%" + email.toLowerCase() + "%");
    }

    public static Specification<User> isActive(Boolean isActive) {
        return (root, query, cb) ->
            isActive == null ? null :
            cb.equal(root.get("isActive"), isActive);
    }
}

// 사용
Specification<User> spec = Specification
    .where(UserSpecification.hasUsername(username))
    .and(UserSpecification.hasEmail(email))
    .and(UserSpecification.isActive(isActive));

Page<User> users = userRepository.findAll(spec, pageable);
```

### 전체 텍스트 검색

```python
from sqlalchemy import or_

def full_text_search(db: Session, search_term: str):
    # PostgreSQL의 경우 tsvector 사용 가능
    users = db.query(User).filter(
        or_(
            User.username.ilike(f"%{search_term}%"),
            User.email.ilike(f"%{search_term}%"),
            User.full_name.ilike(f"%{search_term}%")
        )
    ).all()

    return users
```

---

## 10.5 정렬과 페이지네이션

### 고급 정렬

```python
from enum import Enum

class SortOrder(str, Enum):
    ASC = "asc"
    DESC = "desc"

def get_users_sorted(
    db: Session,
    sort_by: str = "created_at",
    order: SortOrder = SortOrder.DESC,
    skip: int = 0,
    limit: int = 10
):
    query = db.query(User)

    # 동적 정렬
    if hasattr(User, sort_by):
        column = getattr(User, sort_by)
        if order == SortOrder.DESC:
            query = query.order_by(column.desc())
        else:
            query = query.order_by(column.asc())

    users = query.offset(skip).limit(limit).all()
    return users
```

### 커서 기반 페이지네이션

```python
from typing import Optional

def get_users_cursor_based(
    db: Session,
    cursor: Optional[int] = None,
    limit: int = 10
):
    query = db.query(User).order_by(User.id)

    if cursor:
        query = query.filter(User.id > cursor)

    users = query.limit(limit).all()

    next_cursor = users[-1].id if users else None

    return {
        "items": users,
        "next_cursor": next_cursor,
        "has_more": len(users) == limit
    }
```

---

## 10.6 트랜잭션 관리

### 명시적 트랜잭션

```python
from sqlalchemy.orm import Session

def transfer_money(
    db: Session,
    from_account_id: int,
    to_account_id: int,
    amount: float
):
    try:
        # 출금
        from_account = db.query(Account).filter(Account.id == from_account_id).first()
        if not from_account or from_account.balance < amount:
            raise ValueError("Insufficient balance")

        from_account.balance -= amount

        # 입금
        to_account = db.query(Account).filter(Account.id == to_account_id).first()
        if not to_account:
            raise ValueError("Account not found")

        to_account.balance += amount

        # 커밋
        db.commit()

        return {"status": "success", "amount": amount}

    except Exception as e:
        # 롤백
        db.rollback()
        raise e
```

**Spring Boot @Transactional과 비교:**
```java
// Spring Boot
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    Account fromAccount = accountRepository.findById(fromId)
        .orElseThrow();

    if (fromAccount.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException();
    }

    fromAccount.setBalance(fromAccount.getBalance().subtract(amount));

    Account toAccount = accountRepository.findById(toId)
        .orElseThrow();

    toAccount.setBalance(toAccount.getBalance().add(amount));

    accountRepository.save(fromAccount);
    accountRepository.save(toAccount);
    // 자동으로 커밋 (에러 발생 시 자동 롤백)
}
```

### 중첩 트랜잭션 (Savepoint)

```python
def nested_transaction_example(db: Session):
    try:
        # 메인 작업
        user = User(username="john")
        db.add(user)

        # 중첩 트랜잭션 (Savepoint)
        savepoint = db.begin_nested()

        try:
            # 위험한 작업
            risky_operation()
            savepoint.commit()
        except Exception:
            # 중첩 트랜잭션만 롤백
            savepoint.rollback()

        # 메인 작업 커밋
        db.commit()

    except Exception:
        db.rollback()
        raise
```

---

## 10.7 벌크 작업

### 벌크 INSERT

```python
def bulk_create_users(db: Session, users_data: List[dict]):
    # 방법 1: add_all
    users = [User(**data) for data in users_data]
    db.add_all(users)
    db.commit()

    return users

# 방법 2: bulk_insert_mappings (더 빠름)
def bulk_create_users_fast(db: Session, users_data: List[dict]):
    db.bulk_insert_mappings(User, users_data)
    db.commit()
```

### 벌크 UPDATE

```python
def bulk_update_users(db: Session, updates: List[dict]):
    # 방법 1: update query
    db.query(User).filter(User.is_active == False).update(
        {"is_active": True},
        synchronize_session=False
    )
    db.commit()

# 방법 2: bulk_update_mappings
def bulk_update_users_fast(db: Session, updates: List[dict]):
    # updates = [{"id": 1, "username": "new_name"}, ...]
    db.bulk_update_mappings(User, updates)
    db.commit()
```

### 벌크 DELETE

```python
def bulk_delete_inactive_users(db: Session, days: int = 30):
    cutoff_date = datetime.utcnow() - timedelta(days=days)

    deleted_count = db.query(User).filter(
        User.is_active == False,
        User.last_login < cutoff_date
    ).delete(synchronize_session=False)

    db.commit()

    return deleted_count
```

---

## 10.8 에러 처리

### 유니크 제약 위반

```python
from sqlalchemy.exc import IntegrityError
from fastapi import HTTPException

def create_user_safe(db: Session, user: UserCreate):
    try:
        db_user = User(**user.dict())
        db.add(db_user)
        db.commit()
        db.refresh(db_user)
        return db_user

    except IntegrityError as e:
        db.rollback()

        if "unique constraint" in str(e).lower():
            if "email" in str(e).lower():
                raise HTTPException(
                    status_code=400,
                    detail="Email already registered"
                )
            elif "username" in str(e).lower():
                raise HTTPException(
                    status_code=400,
                    detail="Username already taken"
                )

        raise HTTPException(status_code=500, detail="Database error")
```

### 데드락 처리

```python
from sqlalchemy.exc import OperationalError
import time

def retry_on_deadlock(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except OperationalError as e:
            if "deadlock" in str(e).lower() and attempt < max_retries - 1:
                time.sleep(0.1 * (attempt + 1))  # Exponential backoff
                continue
            raise
```

---

## 10.9 실전 예제: 완전한 Product CRUD

```python
# models/product.py
from sqlalchemy import Column, Integer, String, Numeric, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from database import Base

class Category(Base):
    __tablename__ = "categories"

    id = Column(Integer, primary_key=True)
    name = Column(String(100), unique=True, nullable=False)

    products = relationship("Product", back_populates="category")

class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(200), nullable=False, index=True)
    description = Column(String)
    price = Column(Numeric(10, 2), nullable=False)
    stock = Column(Integer, default=0)
    is_available = Column(Boolean, default=True)
    category_id = Column(Integer, ForeignKey("categories.id"))
    created_at = Column(DateTime, server_default=func.now())

    category = relationship("Category", back_populates="products")

# crud/product.py
from sqlalchemy.orm import Session
from typing import Optional, List
from models.product import Product, Category

class ProductRepository:
    def find_by_id(self, db: Session, product_id: int) -> Optional[Product]:
        return db.query(Product).filter(Product.id == product_id).first()

    def find_all(
        self,
        db: Session,
        category_id: Optional[int] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None,
        in_stock: Optional[bool] = None,
        search: Optional[str] = None,
        skip: int = 0,
        limit: int = 10
    ) -> List[Product]:
        query = db.query(Product)

        # 필터 적용
        if category_id:
            query = query.filter(Product.category_id == category_id)

        if min_price:
            query = query.filter(Product.price >= min_price)

        if max_price:
            query = query.filter(Product.price <= max_price)

        if in_stock:
            query = query.filter(Product.stock > 0)

        if search:
            query = query.filter(
                or_(
                    Product.name.ilike(f"%{search}%"),
                    Product.description.ilike(f"%{search}%")
                )
            )

        # 정렬 및 페이지네이션
        products = query.order_by(Product.created_at.desc())\
                       .offset(skip)\
                       .limit(limit)\
                       .all()

        return products

    def create(self, db: Session, product_data: dict) -> Product:
        product = Product(**product_data)
        db.add(product)
        db.commit()
        db.refresh(product)
        return product

    def update(self, db: Session, product_id: int, product_data: dict) -> Optional[Product]:
        product = self.find_by_id(db, product_id)
        if not product:
            return None

        for key, value in product_data.items():
            setattr(product, key, value)

        db.commit()
        db.refresh(product)
        return product

    def delete(self, db: Session, product_id: int) -> bool:
        product = self.find_by_id(db, product_id)
        if not product:
            return False

        db.delete(product)
        db.commit()
        return True

    def update_stock(self, db: Session, product_id: int, quantity: int) -> Optional[Product]:
        product = self.find_by_id(db, product_id)
        if not product:
            return None

        product.stock += quantity
        db.commit()
        db.refresh(product)
        return product

# routers/products.py
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from database import get_db

router = APIRouter(prefix="/products", tags=["products"])
product_repo = ProductRepository()

@router.get("/")
def list_products(
    category_id: Optional[int] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    in_stock: Optional[bool] = None,
    search: Optional[str] = None,
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    db: Session = Depends(get_db)
):
    products = product_repo.find_all(
        db,
        category_id=category_id,
        min_price=min_price,
        max_price=max_price,
        in_stock=in_stock,
        search=search,
        skip=skip,
        limit=limit
    )
    return products

@router.get("/{product_id}")
def get_product(product_id: int, db: Session = Depends(get_db)):
    product = product_repo.find_by_id(db, product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product

@router.post("/", status_code=201)
def create_product(product: ProductCreate, db: Session = Depends(get_db)):
    return product_repo.create(db, product.dict())

@router.put("/{product_id}")
def update_product(
    product_id: int,
    product: ProductUpdate,
    db: Session = Depends(get_db)
):
    updated = product_repo.update(db, product_id, product.dict(exclude_unset=True))
    if not updated:
        raise HTTPException(status_code=404, detail="Product not found")
    return updated

@router.delete("/{product_id}", status_code=204)
def delete_product(product_id: int, db: Session = Depends(get_db)):
    if not product_repo.delete(db, product_id):
        raise HTTPException(status_code=404, detail="Product not found")
```

---

## 10.10 핵심 요약

### CRUD 패턴 비교

| 작업 | Spring Boot | FastAPI/SQLAlchemy |
|------|-------------|-------------------|
| **조회** | `findById()`, `findAll()` | `query().filter().first()`, `.all()` |
| **생성** | `save()` | `add()` + `commit()` |
| **수정** | `save()` (변경 감지) | 속성 변경 + `commit()` |
| **삭제** | `delete()`, `deleteById()` | `delete()` + `commit()` |
| **필터** | Specification | `filter()` |
| **정렬** | `Sort` | `order_by()` |
| **페이징** | `Pageable` | `offset()` + `limit()` |

### 기억할 핵심

1. ✅ **Repository 패턴**: 데이터 접근 로직 분리
2. ✅ **Generic Repository**: 공통 CRUD 로직 재사용
3. ✅ **동적 쿼리**: 조건에 따라 쿼리 구성
4. ✅ **트랜잭션**: try-except-commit-rollback
5. ✅ **벌크 작업**: 대량 데이터 처리 최적화

---

## 10.11 다음 챕터 예고

다음 챕터에서는 **마이그레이션 (Alembic)**을 다룹니다:
- Alembic 설치 및 설정
- 마이그레이션 파일 생성
- 마이그레이션 실행 및 롤백
- Spring의 Flyway/Liquibase와 비교

---

[← 이전: Chapter 9. SQLAlchemy 기초](chapter09-sqlalchemy-basics.md) | [목차로](../README.md) | [다음: Chapter 11. 마이그레이션 (Alembic) →](chapter11-migrations.md)