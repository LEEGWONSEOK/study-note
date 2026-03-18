# Chapter 31. 블로그 API 만들기

**목표**: 실전 프로젝트로 인증, CRUD, 검색, 페이지네이션이 포함된 완전한 블로그 API 구축

---

## 31.1 프로젝트 개요

### 31.1.1 요구사항 정의

**기능 요구사항**:
- 사용자 인증 (JWT)
- 게시글 CRUD
- 댓글 CRUD
- 게시글 검색 및 필터링
- 페이지네이션
- 좋아요 기능
- 태그 기능

**Spring Boot와 비교**:
```
Spring Boot                 FastAPI
─────────────────────────────────────────
@SpringBootApplication      app = FastAPI()
@RestController             @app.get/post/put/delete
@Service                    Service 클래스 + Depends
@Repository                 SQLAlchemy Session
@Entity                     SQLAlchemy Model
@Valid                      Pydantic 자동 검증
```

---

### 31.1.2 프로젝트 구조

```
blog-api/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI 앱 진입점
│   ├── config.py            # 설정
│   ├── database.py          # DB 연결
│   │
│   ├── models/              # SQLAlchemy 모델 (Entity)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── post.py
│   │   ├── comment.py
│   │   └── tag.py
│   │
│   ├── schemas/             # Pydantic 스키마 (DTO)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── post.py
│   │   ├── comment.py
│   │   └── common.py
│   │
│   ├── routers/             # API 라우터 (Controller)
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── posts.py
│   │   ├── comments.py
│   │   └── users.py
│   │
│   ├── services/            # 비즈니스 로직 (Service)
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── post_service.py
│   │   └── comment_service.py
│   │
│   ├── repositories/        # DB 접근 (Repository)
│   │   ├── __init__.py
│   │   ├── user_repo.py
│   │   ├── post_repo.py
│   │   └── comment_repo.py
│   │
│   └── utils/               # 유틸리티
│       ├── __init__.py
│       ├── security.py      # JWT, 해싱
│       └── dependencies.py  # 공통 의존성
│
├── alembic/                 # 마이그레이션
├── tests/                   # 테스트
├── .env                     # 환경 변수
├── requirements.txt
└── README.md
```

---

## 31.2 데이터베이스 설계

### 31.2.1 ERD

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   User      │         │    Post     │         │   Comment   │
├─────────────┤         ├─────────────┤         ├─────────────┤
│ id (PK)     │────┬───<│ author_id   │────┬───<│ author_id   │
│ email       │    │    │ id (PK)     │    │    │ id (PK)     │
│ username    │    │    │ title       │<───┼───<│ post_id     │
│ password    │    │    │ content     │    │    │ content     │
│ created_at  │    │    │ published   │    │    │ created_at  │
└─────────────┘    │    │ created_at  │    │    └─────────────┘
                   │    └─────────────┘    │
                   │            │          │
                   │            │          │
                   │    ┌───────┴──────┐   │
                   │    │              │   │
                   │    v              v   │
                   │ ┌─────────────┐      │
                   │ │ post_tags   │      │
                   │ ├─────────────┤      │
                   │ │ post_id (FK)│      │
                   │ │ tag_id (FK) │      │
                   │ └─────────────┘      │
                   │         │            │
                   │         v            │
                   │ ┌─────────────┐      │
                   │ │    Tag      │      │
                   │ ├─────────────┤      │
                   │ │ id (PK)     │      │
                   │ │ name        │      │
                   │ └─────────────┘      │
                   │                      │
                   │ ┌─────────────┐      │
                   │ │ post_likes  │      │
                   │ ├─────────────┤      │
                   └>│ user_id (FK)│      │
                     │ post_id (FK)│<─────┘
                     │ created_at  │
                     └─────────────┘
```

---

### 31.2.2 모델 정의

**models/user.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, nullable=False, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    hashed_password = Column(String(255), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    comments = relationship("Comment", back_populates="author", cascade="all, delete-orphan")
    liked_posts = relationship("Post", secondary="post_likes", back_populates="liked_by")
```

**Spring Boot 비교**:
```java
// Spring Boot
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(unique = true, nullable = false)
    private String username;

    private String password;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Post> posts;

    @ManyToMany
    @JoinTable(name = "post_likes")
    private Set<Post> likedPosts;
}
```

**models/post.py**:
```python
from sqlalchemy import Column, Integer, String, Text, Boolean, DateTime, ForeignKey, Table
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

# Many-to-Many 연관 테이블
post_tags = Table(
    'post_tags',
    Base.metadata,
    Column('post_id', Integer, ForeignKey('posts.id', ondelete='CASCADE')),
    Column('tag_id', Integer, ForeignKey('tags.id', ondelete='CASCADE'))
)

post_likes = Table(
    'post_likes',
    Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id', ondelete='CASCADE')),
    Column('post_id', Integer, ForeignKey('posts.id', ondelete='CASCADE')),
    Column('created_at', DateTime, default=datetime.utcnow)
)

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False, index=True)
    content = Column(Text, nullable=False)
    published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    author_id = Column(Integer, ForeignKey("users.id", ondelete='CASCADE'), nullable=False)

    # Relationships
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
    tags = relationship("Tag", secondary=post_tags, back_populates="posts")
    liked_by = relationship("User", secondary=post_likes, back_populates="liked_posts")
```

**models/comment.py**:
```python
from sqlalchemy import Column, Integer, Text, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class Comment(Base):
    __tablename__ = "comments"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    author_id = Column(Integer, ForeignKey("users.id", ondelete='CASCADE'), nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id", ondelete='CASCADE'), nullable=False)

    # Relationships
    author = relationship("User", back_populates="comments")
    post = relationship("Post", back_populates="comments")
```

**models/tag.py**:
```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import relationship
from app.database import Base

class Tag(Base):
    __tablename__ = "tags"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False, index=True)

    # Relationships
    posts = relationship("Post", secondary="post_tags", back_populates="tags")
```

---

## 31.3 스키마 (DTO) 정의

### 31.3.1 공통 스키마

**schemas/common.py**:
```python
from pydantic import BaseModel
from typing import Generic, TypeVar, List

T = TypeVar('T')

class PaginationParams(BaseModel):
    """페이지네이션 파라미터"""
    page: int = 1
    size: int = 10

    @property
    def skip(self) -> int:
        return (self.page - 1) * self.size

    @property
    def limit(self) -> int:
        return self.size

class PageResponse(BaseModel, Generic[T]):
    """페이지네이션 응답"""
    items: List[T]
    total: int
    page: int
    size: int
    pages: int

    class Config:
        from_attributes = True
```

**Spring Boot 비교**:
```java
// Spring Boot - Page 객체
Page<Post> posts = postRepository.findAll(PageRequest.of(page, size));
```

---

### 31.3.2 사용자 스키마

**schemas/user.py**:
```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime
from typing import Optional

# 요청 스키마
class UserCreate(BaseModel):
    """회원가입 요청"""
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)
    password: str = Field(..., min_length=8)

class UserLogin(BaseModel):
    """로그인 요청"""
    email: EmailStr
    password: str

# 응답 스키마
class UserResponse(BaseModel):
    """사용자 응답"""
    id: int
    email: str
    username: str
    created_at: datetime

    class Config:
        from_attributes = True

class UserProfile(UserResponse):
    """사용자 프로필 (추가 정보 포함)"""
    posts_count: int = 0
    comments_count: int = 0
```

**Spring Boot 비교**:
```java
// Spring Boot DTO
@Data
public class UserCreateRequest {
    @Email
    private String email;

    @Size(min = 3, max = 50)
    private String username;

    @Size(min = 8)
    private String password;
}

@Data
public class UserResponse {
    private Long id;
    private String email;
    private String username;
    private LocalDateTime createdAt;
}
```

---

### 31.3.3 게시글 스키마

**schemas/post.py**:
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import List, Optional
from .user import UserResponse

class TagBase(BaseModel):
    """태그 기본"""
    name: str = Field(..., min_length=1, max_length=50)

class TagResponse(TagBase):
    """태그 응답"""
    id: int

    class Config:
        from_attributes = True

# 요청 스키마
class PostCreate(BaseModel):
    """게시글 생성 요청"""
    title: str = Field(..., min_length=1, max_length=200)
    content: str = Field(..., min_length=1)
    published: bool = False
    tags: List[str] = []

class PostUpdate(BaseModel):
    """게시글 수정 요청"""
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    content: Optional[str] = Field(None, min_length=1)
    published: Optional[bool] = None
    tags: Optional[List[str]] = None

# 응답 스키마
class PostResponse(BaseModel):
    """게시글 응답 (기본)"""
    id: int
    title: str
    content: str
    published: bool
    created_at: datetime
    updated_at: datetime
    author_id: int

    class Config:
        from_attributes = True

class PostDetail(PostResponse):
    """게시글 상세 (author, tags, 좋아요 수 포함)"""
    author: UserResponse
    tags: List[TagResponse] = []
    likes_count: int = 0
    comments_count: int = 0

class PostSummary(BaseModel):
    """게시글 요약 (목록용)"""
    id: int
    title: str
    content: str = Field(..., max_length=200)  # 요약본
    published: bool
    created_at: datetime
    author: UserResponse
    tags: List[TagResponse] = []
    likes_count: int = 0
    comments_count: int = 0

    class Config:
        from_attributes = True
```

---

### 31.3.4 댓글 스키마

**schemas/comment.py**:
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from .user import UserResponse

class CommentCreate(BaseModel):
    """댓글 생성 요청"""
    content: str = Field(..., min_length=1)

class CommentUpdate(BaseModel):
    """댓글 수정 요청"""
    content: str = Field(..., min_length=1)

class CommentResponse(BaseModel):
    """댓글 응답"""
    id: int
    content: str
    created_at: datetime
    updated_at: datetime
    author_id: int
    post_id: int
    author: UserResponse

    class Config:
        from_attributes = True
```

---

## 31.4 Repository 계층

### 31.4.1 게시글 Repository

**repositories/post_repo.py**:
```python
from sqlalchemy.orm import Session, joinedload
from sqlalchemy import func, or_, desc
from typing import List, Optional
from app.models.post import Post, post_likes
from app.models.tag import Tag
from app.schemas.common import PaginationParams

class PostRepository:
    """게시글 Repository (Spring Data JPA와 유사)"""

    def __init__(self, db: Session):
        self.db = db

    def create(self, post: Post) -> Post:
        """게시글 생성"""
        self.db.add(post)
        self.db.commit()
        self.db.refresh(post)
        return post

    def find_by_id(self, post_id: int) -> Optional[Post]:
        """ID로 게시글 조회 (author, tags eager loading)"""
        return (
            self.db.query(Post)
            .options(
                joinedload(Post.author),
                joinedload(Post.tags)
            )
            .filter(Post.id == post_id)
            .first()
        )

    def find_all(
        self,
        pagination: PaginationParams,
        published_only: bool = True
    ) -> tuple[List[Post], int]:
        """게시글 목록 조회 (페이지네이션)"""
        query = self.db.query(Post)

        if published_only:
            query = query.filter(Post.published == True)

        # 총 개수
        total = query.count()

        # 페이지네이션 적용
        posts = (
            query
            .options(
                joinedload(Post.author),
                joinedload(Post.tags)
            )
            .order_by(desc(Post.created_at))
            .offset(pagination.skip)
            .limit(pagination.limit)
            .all()
        )

        return posts, total

    def search(
        self,
        keyword: str,
        pagination: PaginationParams
    ) -> tuple[List[Post], int]:
        """게시글 검색 (제목, 내용)"""
        query = (
            self.db.query(Post)
            .filter(
                Post.published == True,
                or_(
                    Post.title.contains(keyword),
                    Post.content.contains(keyword)
                )
            )
        )

        total = query.count()

        posts = (
            query
            .options(
                joinedload(Post.author),
                joinedload(Post.tags)
            )
            .order_by(desc(Post.created_at))
            .offset(pagination.skip)
            .limit(pagination.limit)
            .all()
        )

        return posts, total

    def find_by_author(
        self,
        author_id: int,
        pagination: PaginationParams
    ) -> tuple[List[Post], int]:
        """작성자별 게시글 조회"""
        query = self.db.query(Post).filter(Post.author_id == author_id)

        total = query.count()

        posts = (
            query
            .options(
                joinedload(Post.author),
                joinedload(Post.tags)
            )
            .order_by(desc(Post.created_at))
            .offset(pagination.skip)
            .limit(pagination.limit)
            .all()
        )

        return posts, total

    def find_by_tag(
        self,
        tag_name: str,
        pagination: PaginationParams
    ) -> tuple[List[Post], int]:
        """태그별 게시글 조회"""
        query = (
            self.db.query(Post)
            .join(Post.tags)
            .filter(Tag.name == tag_name, Post.published == True)
        )

        total = query.count()

        posts = (
            query
            .options(
                joinedload(Post.author),
                joinedload(Post.tags)
            )
            .order_by(desc(Post.created_at))
            .offset(pagination.skip)
            .limit(pagination.limit)
            .all()
        )

        return posts, total

    def update(self, post: Post) -> Post:
        """게시글 수정"""
        self.db.commit()
        self.db.refresh(post)
        return post

    def delete(self, post: Post) -> None:
        """게시글 삭제"""
        self.db.delete(post)
        self.db.commit()

    def get_likes_count(self, post_id: int) -> int:
        """좋아요 수 조회"""
        return (
            self.db.query(func.count(post_likes.c.user_id))
            .filter(post_likes.c.post_id == post_id)
            .scalar()
        )

    def is_liked_by_user(self, post_id: int, user_id: int) -> bool:
        """사용자가 좋아요 했는지 확인"""
        return (
            self.db.query(post_likes)
            .filter(
                post_likes.c.post_id == post_id,
                post_likes.c.user_id == user_id
            )
            .first()
        ) is not None

    def add_like(self, post_id: int, user_id: int) -> None:
        """좋아요 추가"""
        self.db.execute(
            post_likes.insert().values(post_id=post_id, user_id=user_id)
        )
        self.db.commit()

    def remove_like(self, post_id: int, user_id: int) -> None:
        """좋아요 제거"""
        self.db.execute(
            post_likes.delete().where(
                post_likes.c.post_id == post_id,
                post_likes.c.user_id == user_id
            )
        )
        self.db.commit()
```

**Spring Boot 비교**:
```java
// Spring Data JPA
public interface PostRepository extends JpaRepository<Post, Long> {
    Page<Post> findByPublishedTrue(Pageable pageable);

    Page<Post> findByTitleContainingOrContentContaining(
        String title, String content, Pageable pageable
    );

    @Query("SELECT p FROM Post p JOIN p.tags t WHERE t.name = :tagName")
    Page<Post> findByTagName(@Param("tagName") String tagName, Pageable pageable);

    @Query("SELECT COUNT(l) FROM PostLike l WHERE l.post.id = :postId")
    int countLikes(@Param("postId") Long postId);
}
```

---

## 31.5 Service 계층

### 31.5.1 게시글 Service

**services/post_service.py**:
```python
from typing import List, Optional
from sqlalchemy.orm import Session
from fastapi import HTTPException, status
from app.repositories.post_repo import PostRepository
from app.models.post import Post
from app.models.tag import Tag
from app.schemas.post import PostCreate, PostUpdate, PostDetail, PostSummary
from app.schemas.common import PaginationParams, PageResponse

class PostService:
    """게시글 비즈니스 로직 (Spring @Service와 유사)"""

    def __init__(self, db: Session):
        self.db = db
        self.repo = PostRepository(db)

    def create_post(self, post_data: PostCreate, author_id: int) -> Post:
        """게시글 생성"""
        # Post 엔티티 생성
        post = Post(
            title=post_data.title,
            content=post_data.content,
            published=post_data.published,
            author_id=author_id
        )

        # 태그 처리
        if post_data.tags:
            tags = self._get_or_create_tags(post_data.tags)
            post.tags = tags

        return self.repo.create(post)

    def get_post(self, post_id: int) -> PostDetail:
        """게시글 상세 조회"""
        post = self.repo.find_by_id(post_id)
        if not post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Post {post_id} not found"
            )

        # 추가 정보 포함
        return PostDetail(
            **post.__dict__,
            likes_count=self.repo.get_likes_count(post_id),
            comments_count=len(post.comments)
        )

    def get_posts(
        self,
        pagination: PaginationParams,
        published_only: bool = True
    ) -> PageResponse[PostSummary]:
        """게시글 목록 조회"""
        posts, total = self.repo.find_all(pagination, published_only)

        # PostSummary로 변환
        items = [
            PostSummary(
                **post.__dict__,
                content=post.content[:200] + "..." if len(post.content) > 200 else post.content,
                likes_count=self.repo.get_likes_count(post.id),
                comments_count=len(post.comments)
            )
            for post in posts
        ]

        return PageResponse(
            items=items,
            total=total,
            page=pagination.page,
            size=pagination.size,
            pages=(total + pagination.size - 1) // pagination.size
        )

    def search_posts(
        self,
        keyword: str,
        pagination: PaginationParams
    ) -> PageResponse[PostSummary]:
        """게시글 검색"""
        posts, total = self.repo.search(keyword, pagination)

        items = [
            PostSummary(
                **post.__dict__,
                content=post.content[:200] + "...",
                likes_count=self.repo.get_likes_count(post.id),
                comments_count=len(post.comments)
            )
            for post in posts
        ]

        return PageResponse(
            items=items,
            total=total,
            page=pagination.page,
            size=pagination.size,
            pages=(total + pagination.size - 1) // pagination.size
        )

    def get_posts_by_tag(
        self,
        tag_name: str,
        pagination: PaginationParams
    ) -> PageResponse[PostSummary]:
        """태그별 게시글 조회"""
        posts, total = self.repo.find_by_tag(tag_name, pagination)

        items = [
            PostSummary(
                **post.__dict__,
                content=post.content[:200] + "...",
                likes_count=self.repo.get_likes_count(post.id),
                comments_count=len(post.comments)
            )
            for post in posts
        ]

        return PageResponse(
            items=items,
            total=total,
            page=pagination.page,
            size=pagination.size,
            pages=(total + pagination.size - 1) // pagination.size
        )

    def update_post(
        self,
        post_id: int,
        post_data: PostUpdate,
        user_id: int
    ) -> Post:
        """게시글 수정"""
        post = self.repo.find_by_id(post_id)
        if not post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Post {post_id} not found"
            )

        # 작성자 확인
        if post.author_id != user_id:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not authorized to update this post"
            )

        # 필드 업데이트
        if post_data.title is not None:
            post.title = post_data.title
        if post_data.content is not None:
            post.content = post_data.content
        if post_data.published is not None:
            post.published = post_data.published
        if post_data.tags is not None:
            tags = self._get_or_create_tags(post_data.tags)
            post.tags = tags

        return self.repo.update(post)

    def delete_post(self, post_id: int, user_id: int) -> None:
        """게시글 삭제"""
        post = self.repo.find_by_id(post_id)
        if not post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Post {post_id} not found"
            )

        # 작성자 확인
        if post.author_id != user_id:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not authorized to delete this post"
            )

        self.repo.delete(post)

    def toggle_like(self, post_id: int, user_id: int) -> dict:
        """좋아요 토글"""
        post = self.repo.find_by_id(post_id)
        if not post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Post {post_id} not found"
            )

        is_liked = self.repo.is_liked_by_user(post_id, user_id)

        if is_liked:
            self.repo.remove_like(post_id, user_id)
            action = "unliked"
        else:
            self.repo.add_like(post_id, user_id)
            action = "liked"

        return {
            "action": action,
            "likes_count": self.repo.get_likes_count(post_id)
        }

    def _get_or_create_tags(self, tag_names: List[str]) -> List[Tag]:
        """태그 가져오기 또는 생성"""
        tags = []
        for name in tag_names:
            tag = self.db.query(Tag).filter(Tag.name == name).first()
            if not tag:
                tag = Tag(name=name)
                self.db.add(tag)
            tags.append(tag)
        self.db.commit()
        return tags
```

**Spring Boot 비교**:
```java
// Spring Boot Service
@Service
@Transactional
public class PostService {
    @Autowired
    private PostRepository postRepository;

    public Post createPost(PostCreateRequest request, Long authorId) {
        Post post = new Post();
        post.setTitle(request.getTitle());
        post.setContent(request.getContent());
        post.setAuthorId(authorId);

        return postRepository.save(post);
    }

    public PostDetail getPost(Long postId) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));

        return convertToDetail(post);
    }

    public Page<PostSummary> getPosts(Pageable pageable) {
        Page<Post> posts = postRepository.findAll(pageable);
        return posts.map(this::convertToSummary);
    }
}
```

---

## 31.6 Router (Controller) 계층

### 31.6.1 게시글 Router

**routers/posts.py**:
```python
from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.orm import Session
from typing import Optional
from app.database import get_db
from app.services.post_service import PostService
from app.schemas.post import PostCreate, PostUpdate, PostDetail, PostSummary
from app.schemas.common import PaginationParams, PageResponse
from app.utils.dependencies import get_current_user
from app.models.user import User

router = APIRouter(prefix="/posts", tags=["posts"])

# 의존성 함수
def get_post_service(db: Session = Depends(get_db)) -> PostService:
    return PostService(db)

@router.post(
    "/",
    response_model=PostDetail,
    status_code=status.HTTP_201_CREATED,
    summary="게시글 생성"
)
def create_post(
    post_data: PostCreate,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
):
    """
    게시글 생성

    - **title**: 게시글 제목 (필수)
    - **content**: 게시글 내용 (필수)
    - **published**: 공개 여부 (기본값: false)
    - **tags**: 태그 목록 (선택)
    """
    post = service.create_post(post_data, current_user.id)
    return service.get_post(post.id)

@router.get(
    "/",
    response_model=PageResponse[PostSummary],
    summary="게시글 목록 조회"
)
def get_posts(
    page: int = Query(1, ge=1, description="페이지 번호"),
    size: int = Query(10, ge=1, le=100, description="페이지 크기"),
    published_only: bool = Query(True, description="공개된 게시글만"),
    service: PostService = Depends(get_post_service)
):
    """
    게시글 목록 조회 (페이지네이션)

    - 최신순 정렬
    - 작성자, 태그, 좋아요 수, 댓글 수 포함
    """
    pagination = PaginationParams(page=page, size=size)
    return service.get_posts(pagination, published_only)

@router.get(
    "/search",
    response_model=PageResponse[PostSummary],
    summary="게시글 검색"
)
def search_posts(
    q: str = Query(..., min_length=1, description="검색 키워드"),
    page: int = Query(1, ge=1),
    size: int = Query(10, ge=1, le=100),
    service: PostService = Depends(get_post_service)
):
    """
    게시글 검색 (제목, 내용)

    - 제목 또는 내용에 키워드 포함된 게시글 검색
    """
    pagination = PaginationParams(page=page, size=size)
    return service.search_posts(q, pagination)

@router.get(
    "/tags/{tag_name}",
    response_model=PageResponse[PostSummary],
    summary="태그별 게시글 조회"
)
def get_posts_by_tag(
    tag_name: str,
    page: int = Query(1, ge=1),
    size: int = Query(10, ge=1, le=100),
    service: PostService = Depends(get_post_service)
):
    """
    특정 태그의 게시글 목록 조회
    """
    pagination = PaginationParams(page=page, size=size)
    return service.get_posts_by_tag(tag_name, pagination)

@router.get(
    "/{post_id}",
    response_model=PostDetail,
    summary="게시글 상세 조회"
)
def get_post(
    post_id: int,
    service: PostService = Depends(get_post_service)
):
    """
    게시글 상세 조회

    - 작성자, 태그, 좋아요 수, 댓글 수 포함
    """
    return service.get_post(post_id)

@router.put(
    "/{post_id}",
    response_model=PostDetail,
    summary="게시글 수정"
)
def update_post(
    post_id: int,
    post_data: PostUpdate,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
):
    """
    게시글 수정

    - 작성자만 수정 가능
    """
    post = service.update_post(post_id, post_data, current_user.id)
    return service.get_post(post.id)

@router.delete(
    "/{post_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="게시글 삭제"
)
def delete_post(
    post_id: int,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
):
    """
    게시글 삭제

    - 작성자만 삭제 가능
    """
    service.delete_post(post_id, current_user.id)

@router.post(
    "/{post_id}/like",
    summary="게시글 좋아요 토글"
)
def toggle_like(
    post_id: int,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
):
    """
    게시글 좋아요 토글

    - 좋아요가 없으면 추가, 있으면 제거
    """
    return service.toggle_like(post_id, current_user.id)
```

**Spring Boot 비교**:
```java
// Spring Boot Controller
@RestController
@RequestMapping("/posts")
public class PostController {
    @Autowired
    private PostService postService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public PostDetail createPost(
        @RequestBody @Valid PostCreateRequest request,
        @AuthenticationPrincipal User currentUser
    ) {
        return postService.createPost(request, currentUser.getId());
    }

    @GetMapping
    public Page<PostSummary> getPosts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size
    ) {
        Pageable pageable = PageRequest.of(page, size);
        return postService.getPosts(pageable);
    }

    @GetMapping("/{postId}")
    public PostDetail getPost(@PathVariable Long postId) {
        return postService.getPost(postId);
    }

    @PutMapping("/{postId}")
    public PostDetail updatePost(
        @PathVariable Long postId,
        @RequestBody @Valid PostUpdateRequest request,
        @AuthenticationPrincipal User currentUser
    ) {
        return postService.updatePost(postId, request, currentUser.getId());
    }

    @DeleteMapping("/{postId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deletePost(
        @PathVariable Long postId,
        @AuthenticationPrincipal User currentUser
    ) {
        postService.deletePost(postId, currentUser.getId());
    }
}
```

---

## 31.7 인증 구현

### 31.7.1 JWT 유틸리티

**utils/security.py**:
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import HTTPException, status

SECRET_KEY = "your-secret-key-here"  # 환경 변수로 관리
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """비밀번호 검증"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """비밀번호 해싱"""
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """액세스 토큰 생성"""
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def decode_access_token(token: str) -> dict:
    """액세스 토큰 디코딩"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

**utils/dependencies.py**:
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from app.database import get_db
from app.models.user import User
from app.utils.security import decode_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """현재 로그인한 사용자 가져오기"""
    payload = decode_access_token(token)
    user_id: int = payload.get("sub")

    if user_id is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials"
        )

    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found"
        )

    return user
```

**routers/auth.py**:
```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from app.database import get_db
from app.models.user import User
from app.schemas.user import UserCreate, UserResponse, UserLogin
from app.utils.security import verify_password, get_password_hash, create_access_token

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def register(user_data: UserCreate, db: Session = Depends(get_db)):
    """회원가입"""
    # 이메일 중복 확인
    if db.query(User).filter(User.email == user_data.email).first():
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Email already registered"
        )

    # 사용자명 중복 확인
    if db.query(User).filter(User.username == user_data.username).first():
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Username already taken"
        )

    # 사용자 생성
    user = User(
        email=user_data.email,
        username=user_data.username,
        hashed_password=get_password_hash(user_data.password)
    )
    db.add(user)
    db.commit()
    db.refresh(user)

    return user

@router.post("/login")
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """로그인"""
    # 사용자 조회 (email을 username 필드에 입력)
    user = db.query(User).filter(User.email == form_data.username).first()

    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # 액세스 토큰 생성
    access_token = create_access_token(data={"sub": str(user.id)})

    return {
        "access_token": access_token,
        "token_type": "bearer"
    }
```

---

## 31.8 메인 애플리케이션

**main.py**:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routers import auth, posts, comments, users
from app.database import engine, Base

# 데이터베이스 테이블 생성 (프로덕션에서는 Alembic 사용)
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="Blog API",
    description="FastAPI로 만든 블로그 REST API",
    version="1.0.0"
)

# CORS 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 라우터 등록
app.include_router(auth.router)
app.include_router(posts.router)
app.include_router(comments.router)
app.include_router(users.router)

@app.get("/")
def root():
    return {"message": "Blog API", "docs": "/docs"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

---

## 31.9 테스트

**tests/test_posts.py**:
```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_post():
    """게시글 생성 테스트"""
    # 회원가입
    register_response = client.post("/auth/register", json={
        "email": "test@example.com",
        "username": "testuser",
        "password": "password123"
    })
    assert register_response.status_code == 201

    # 로그인
    login_response = client.post("/auth/login", data={
        "username": "test@example.com",
        "password": "password123"
    })
    assert login_response.status_code == 200
    token = login_response.json()["access_token"]

    # 게시글 생성
    response = client.post(
        "/posts/",
        json={
            "title": "Test Post",
            "content": "This is a test post",
            "published": True,
            "tags": ["test", "python"]
        },
        headers={"Authorization": f"Bearer {token}"}
    )

    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Test Post"
    assert len(data["tags"]) == 2

def test_get_posts():
    """게시글 목록 조회 테스트"""
    response = client.get("/posts/")
    assert response.status_code == 200
    data = response.json()
    assert "items" in data
    assert "total" in data
    assert "page" in data
```

---

## 31.10 정리

### Spring Boot vs FastAPI 완전 비교

| 개념 | Spring Boot | FastAPI |
|------|-------------|---------|
| **엔티티** | `@Entity` 클래스 | SQLAlchemy `Base` 클래스 |
| **DTO** | Java 클래스 + `@Valid` | Pydantic 모델 (자동 검증) |
| **Controller** | `@RestController` | `APIRouter` + 데코레이터 |
| **Service** | `@Service` 클래스 | Service 클래스 |
| **Repository** | `JpaRepository` 인터페이스 | Repository 클래스 + Session |
| **DI** | `@Autowired` | `Depends()` |
| **인증** | Spring Security + `@AuthenticationPrincipal` | JWT + `Depends(get_current_user)` |
| **페이지네이션** | `Page<T>` + `Pageable` | 커스텀 `PageResponse<T>` |
| **검증** | `@Valid` + Hibernate Validator | Pydantic 자동 검증 |
| **예외 처리** | `@ExceptionHandler` | `HTTPException` + 핸들러 |

### 주요 학습 포인트

1. **계층형 아키텍처**: Model → Repository → Service → Router
2. **의존성 주입**: `Depends()`를 활용한 깔끔한 DI
3. **Pydantic 검증**: 자동 타입 검증 및 직렬화
4. **SQLAlchemy ORM**: 관계 설정, 쿼리 최적화
5. **JWT 인증**: 토큰 기반 인증 구현
6. **페이지네이션**: 커스텀 페이지네이션 구현
7. **검색 및 필터링**: 동적 쿼리 작성

---

## 다음 단계

- [Chapter 32. E-Commerce API 만들기 →](chapter32-ecommerce-api.md)
- 더 복잡한 비즈니스 로직 (주문, 결제)
- 트랜잭션 관리
- 재고 관리 및 동시성 제어
