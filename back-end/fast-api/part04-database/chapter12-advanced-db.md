# Chapter 12. 고급 데이터베이스 기능

## 12.1 Relationship (관계)

SQLAlchemy의 Relationship은 Spring JPA의 `@OneToMany`, `@ManyToOne`, `@ManyToMany`와 동일한 개념입니다.

### One-to-Many (일대다)

**Spring Boot JPA:**
```java
// User 엔티티
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Post> posts = new ArrayList<>();
}

// Post 엔티티
@Entity
public class Post {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private User author;
}
```

**FastAPI SQLAlchemy:**
```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from database import Base

# User 모델
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String(50), nullable=False)

    # 관계 정의 (One-to-Many)
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")

# Post 모델
class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(String, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"))

    # 관계 정의 (Many-to-One)
    author = relationship("User", back_populates="posts")
```

**사용 예제:**
```python
from sqlalchemy.orm import Session

# 사용자와 게시글 함께 조회
user = db.query(User).filter(User.id == 1).first()
print(user.posts)  # 사용자의 모든 게시글

# 게시글에서 작성자 조회
post = db.query(Post).filter(Post.id == 1).first()
print(post.author.username)  # 작성자 이름

# 새 게시글 생성
user = db.query(User).filter(User.id == 1).first()
new_post = Post(title="New Post", content="Content")
user.posts.append(new_post)
db.commit()
```

---

## 12.2 Many-to-Many (다대다)

### 중간 테이블 사용

**Spring Boot JPA:**
```java
@Entity
public class Student {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

**FastAPI SQLAlchemy:**
```python
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

# 중간 테이블 정의
student_course = Table(
    'student_course',
    Base.metadata,
    Column('student_id', Integer, ForeignKey('students.id'), primary_key=True),
    Column('course_id', Integer, ForeignKey('courses.id'), primary_key=True)
)

class Student(Base):
    __tablename__ = "students"

    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)

    # Many-to-Many 관계
    courses = relationship("Course", secondary=student_course, back_populates="students")

class Course(Base):
    __tablename__ = "courses"

    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)

    # Many-to-Many 관계
    students = relationship("Student", secondary=student_course, back_populates="courses")
```

**사용 예제:**
```python
# 학생에게 수업 추가
student = db.query(Student).filter(Student.id == 1).first()
course = db.query(Course).filter(Course.id == 1).first()
student.courses.append(course)
db.commit()

# 학생의 모든 수업 조회
student = db.query(Student).filter(Student.id == 1).first()
for course in student.courses:
    print(course.title)

# 수업의 모든 학생 조회
course = db.query(Course).filter(Course.id == 1).first()
for student in course.students:
    print(student.name)
```

### 추가 필드가 있는 Many-to-Many

```python
from sqlalchemy import DateTime
from datetime import datetime

# 중간 테이블을 모델로 정의
class Enrollment(Base):
    __tablename__ = "enrollments"

    student_id = Column(Integer, ForeignKey("students.id"), primary_key=True)
    course_id = Column(Integer, ForeignKey("courses.id"), primary_key=True)
    enrolled_at = Column(DateTime, default=datetime.utcnow)
    grade = Column(String(2))  # 추가 필드

    # 관계
    student = relationship("Student", back_populates="enrollments")
    course = relationship("Course", back_populates="enrollments")

class Student(Base):
    __tablename__ = "students"

    id = Column(Integer, primary_key=True)
    name = Column(String(100))

    enrollments = relationship("Enrollment", back_populates="student")

class Course(Base):
    __tablename__ = "courses"

    id = Column(Integer, primary_key=True)
    title = Column(String(200))

    enrollments = relationship("Enrollment", back_populates="course")
```

---

## 12.3 Lazy Loading vs Eager Loading

### Lazy Loading (지연 로딩)

```python
# 기본값은 Lazy Loading
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", lazy="select")  # 기본값

# 사용
user = db.query(User).filter(User.id == 1).first()
# 여기서는 posts를 조회하지 않음

print(user.posts)  # 이 시점에 posts를 조회 (추가 쿼리 발생)
```

**Spring Boot와 비교:**
```java
// Spring Boot - 기본값은 Lazy
@OneToMany(fetch = FetchType.LAZY)  // 기본값
private List<Post> posts;
```

### Eager Loading (즉시 로딩)

```python
from sqlalchemy.orm import joinedload

# 방법 1: 쿼리 시 명시
user = db.query(User).options(joinedload(User.posts)).filter(User.id == 1).first()
# posts를 JOIN으로 함께 조회 (1개 쿼리)

# 방법 2: 관계 정의 시 설정
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", lazy="joined")  # Eager Loading
```

**Spring Boot와 비교:**
```java
// Spring Boot
@OneToMany(fetch = FetchType.EAGER)
private List<Post> posts;

// 또는 JPQL
@Query("SELECT u FROM User u JOIN FETCH u.posts WHERE u.id = :id")
User findByIdWithPosts(@Param("id") Long id);
```

### Lazy Loading 옵션

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)

    # lazy 옵션
    posts1 = relationship("Post", lazy="select")      # 기본값, 접근 시 SELECT
    posts2 = relationship("Post", lazy="joined")      # JOIN으로 즉시 로딩
    posts3 = relationship("Post", lazy="subquery")    # 서브쿼리로 로딩
    posts4 = relationship("Post", lazy="dynamic")     # Query 객체 반환
```

### N+1 문제 해결

```python
from sqlalchemy.orm import joinedload, selectinload

# N+1 문제 발생
users = db.query(User).all()
for user in users:  # N명의 사용자
    print(user.posts)  # 각 사용자마다 쿼리 발생 (N번)

# 해결 방법 1: joinedload (JOIN 사용)
users = db.query(User).options(joinedload(User.posts)).all()
for user in users:
    print(user.posts)  # 추가 쿼리 없음

# 해결 방법 2: selectinload (IN 절 사용)
users = db.query(User).options(selectinload(User.posts)).all()
for user in users:
    print(user.posts)  # 추가 쿼리 없음
```

**Spring Boot와 비교:**
```java
// N+1 문제 발생
List<User> users = userRepository.findAll();
for (User user : users) {
    user.getPosts();  // 각 사용자마다 쿼리 발생
}

// 해결: JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.posts")
List<User> findAllWithPosts();
```

---

## 12.4 Cascade (영속성 전이)

### Cascade 옵션

```python
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)

    # cascade 옵션
    posts = relationship(
        "Post",
        back_populates="author",
        cascade="all, delete-orphan"
    )
```

**Cascade 옵션:**
- `all`: 모든 작업 전파 (save, merge, delete 등)
- `delete`: 부모 삭제 시 자식도 삭제
- `delete-orphan`: 관계가 끊어진 자식 삭제
- `save-update`: 부모 저장 시 자식도 저장
- `merge`: 부모 병합 시 자식도 병합

**Spring Boot와 비교:**
```java
// Spring Boot
@OneToMany(
    mappedBy = "author",
    cascade = CascadeType.ALL,
    orphanRemoval = true
)
private List<Post> posts;
```

**사용 예제:**
```python
# cascade="all, delete-orphan" 예제
user = User(username="john")
post1 = Post(title="Post 1", content="Content 1")
post2 = Post(title="Post 2", content="Content 2")

user.posts.append(post1)
user.posts.append(post2)

db.add(user)
db.commit()  # user와 post1, post2가 모두 저장됨

# 사용자 삭제 시 게시글도 함께 삭제
db.delete(user)
db.commit()  # user와 관련 posts가 모두 삭제됨
```

---

## 12.5 트랜잭션 관리

### 기본 트랜잭션

```python
from sqlalchemy.orm import Session

def transfer_money(db: Session, from_id: int, to_id: int, amount: float):
    try:
        # 출금
        from_account = db.query(Account).filter(Account.id == from_id).with_for_update().first()
        if from_account.balance < amount:
            raise ValueError("Insufficient balance")
        from_account.balance -= amount

        # 입금
        to_account = db.query(Account).filter(Account.id == to_id).with_for_update().first()
        to_account.balance += amount

        # 커밋
        db.commit()

    except Exception as e:
        # 롤백
        db.rollback()
        raise e
```

### Savepoint (중첩 트랜잭션)

```python
from sqlalchemy.orm import Session

def complex_operation(db: Session):
    # 메인 작업
    user = User(username="john")
    db.add(user)

    # Savepoint 생성
    savepoint = db.begin_nested()

    try:
        # 위험한 작업
        risky_operation(db)
        savepoint.commit()
    except Exception:
        # Savepoint만 롤백
        savepoint.rollback()

    # 메인 작업 커밋
    db.commit()
```

### 락 (Lock)

```python
from sqlalchemy import select

# 비관적 락 (Pessimistic Lock)
user = db.query(User).filter(User.id == 1).with_for_update().first()
user.balance -= 100
db.commit()

# 공유 락 (Shared Lock)
user = db.query(User).filter(User.id == 1).with_for_update(read=True).first()
```

**Spring Boot와 비교:**
```java
// Spring Boot
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT u FROM User u WHERE u.id = :id")
User findByIdForUpdate(@Param("id") Long id);

@Lock(LockModeType.PESSIMISTIC_READ)
@Query("SELECT u FROM User u WHERE u.id = :id")
User findByIdForRead(@Param("id") Long id);
```

---

## 12.6 데이터베이스 풀링

### Connection Pool 설정

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "postgresql://user:password@localhost/dbname",
    poolclass=QueuePool,
    pool_size=10,              # 기본 연결 수
    max_overflow=20,           # 추가 가능한 최대 연결 수
    pool_timeout=30,           # 연결 대기 시간 (초)
    pool_recycle=3600,         # 연결 재사용 시간 (초)
    pool_pre_ping=True,        # 연결 유효성 체크
    echo_pool=True             # 풀 디버그 로그
)
```

**Spring Boot와 비교:**
```yaml
# Spring Boot - HikariCP (기본)
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

---

## 12.7 다중 데이터베이스 연결

### 여러 데이터베이스 설정

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

# 메인 데이터베이스
engine_main = create_engine("postgresql://user:password@localhost/main_db")
SessionMain = sessionmaker(bind=engine_main)
BaseMain = declarative_base()

# 읽기 전용 데이터베이스
engine_read = create_engine("postgresql://user:password@localhost/read_replica")
SessionRead = sessionmaker(bind=engine_read)
BaseRead = declarative_base()

# 세션 의존성
def get_db_main():
    db = SessionMain()
    try:
        yield db
    finally:
        db.close()

def get_db_read():
    db = SessionRead()
    try:
        yield db
    finally:
        db.close()

# 사용
@app.get("/users")
def list_users(db: Session = Depends(get_db_read)):  # 읽기용 DB
    return db.query(User).all()

@app.post("/users")
def create_user(user: UserCreate, db: Session = Depends(get_db_main)):  # 쓰기용 DB
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    return db_user
```

---

## 12.8 실전 예제: 블로그 시스템

```python
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Table
from sqlalchemy.orm import relationship
from datetime import datetime
from database import Base

# Many-to-Many 중간 테이블
post_tags = Table(
    'post_tags',
    Base.metadata,
    Column('post_id', Integer, ForeignKey('posts.id'), primary_key=True),
    Column('tag_id', Integer, ForeignKey('tags.id'), primary_key=True)
)

# User 모델
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(100), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)

    # 관계
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    comments = relationship("Comment", back_populates="author", cascade="all, delete-orphan")

# Post 모델
class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # 관계
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
    tags = relationship("Tag", secondary=post_tags, back_populates="posts")

# Comment 모델
class Comment(Base):
    __tablename__ = "comments"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"))
    author_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)

    # 관계
    post = relationship("Post", back_populates="comments")
    author = relationship("User", back_populates="comments")

# Tag 모델
class Tag(Base):
    __tablename__ = "tags"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False)

    # 관계
    posts = relationship("Post", secondary=post_tags, back_populates="tags")

# 사용 예제
from sqlalchemy.orm import Session, joinedload

def get_post_with_all_relations(db: Session, post_id: int):
    """게시글과 모든 관계 데이터를 한 번에 조회 (N+1 방지)"""
    post = (
        db.query(Post)
        .options(
            joinedload(Post.author),
            joinedload(Post.comments).joinedload(Comment.author),
            joinedload(Post.tags)
        )
        .filter(Post.id == post_id)
        .first()
    )

    return post

def create_post_with_tags(db: Session, post_data: dict, tag_names: list):
    """게시글 생성 및 태그 연결"""
    # 태그 조회 또는 생성
    tags = []
    for tag_name in tag_names:
        tag = db.query(Tag).filter(Tag.name == tag_name).first()
        if not tag:
            tag = Tag(name=tag_name)
            db.add(tag)
        tags.append(tag)

    # 게시글 생성
    post = Post(**post_data)
    post.tags = tags

    db.add(post)
    db.commit()
    db.refresh(post)

    return post
```

---

## 12.9 핵심 요약

### Relationship 비교

| 관계 | Spring JPA | SQLAlchemy |
|------|-----------|------------|
| **One-to-Many** | `@OneToMany` | `relationship()` |
| **Many-to-One** | `@ManyToOne` | `ForeignKey` + `relationship()` |
| **Many-to-Many** | `@ManyToMany` | `secondary` + `relationship()` |
| **Cascade** | `cascade=CascadeType.ALL` | `cascade="all"` |
| **Lazy Loading** | `fetch=FetchType.LAZY` | `lazy="select"` |
| **Eager Loading** | `fetch=FetchType.EAGER` | `lazy="joined"` |

### 기억할 핵심

1. ✅ **relationship()**: 관계 정의
2. ✅ **back_populates**: 양방향 관계
3. ✅ **cascade**: 영속성 전이
4. ✅ **lazy**: 로딩 전략
5. ✅ **joinedload**: N+1 문제 해결
6. ✅ **with_for_update**: 비관적 락

---

## 12.10 실습 예제

### 실습 1: E-Commerce 관계 설정

**요구사항:**
- User - Order (One-to-Many)
- Order - OrderItem (One-to-Many)
- Product - OrderItem (One-to-Many)
- N+1 문제 해결

---

## 12.11 Part 04 완료!

Part 04에서 배운 내용:
- ✅ **Chapter 9**: SQLAlchemy 기초 - 모델 정의, CRUD
- ✅ **Chapter 10**: CRUD 구현 - Repository 패턴, 복잡한 쿼리
- ✅ **Chapter 11**: 마이그레이션 - Alembic 사용법
- ✅ **Chapter 12**: 고급 기능 - Relationship, 트랜잭션, 풀링

---

## 12.12 다음 챕터 예고

다음 Part에서는 **인증과 보안**을 다룹니다:
- JWT 인증
- OAuth2
- 권한 관리
- 보안 Best Practices

**Spring Security와 비교하며 FastAPI의 보안을 완전히 마스터합니다!**

---

[← 이전: Chapter 11. 마이그레이션 (Alembic)](chapter11-migrations.md) | [목차로](../README.md) | [다음: Part 05 →](../part05-security/chapter13-jwt.md)
