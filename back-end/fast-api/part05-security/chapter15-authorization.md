# Chapter 15. 권한과 역할 관리

## 15.1 Role-Based Access Control (RBAC)

RBAC는 사용자의 역할(Role)에 따라 접근 권한을 제어하는 방식입니다. Spring Security의 Role 기반 인가와 동일한 개념입니다.

### Spring Security vs FastAPI RBAC

```java
// Spring Security
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/admin/users")
public List<User> getAllUsers() {
    return userService.findAll();
}

@PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
@DeleteMapping("/posts/{id}")
public void deletePost(@PathVariable Long id) {
    postService.delete(id);
}
```

```python
# FastAPI
from app.api.deps import require_role

@app.get("/admin/users")
async def get_all_users(
    current_user: User = Depends(require_role(["admin"]))
):
    return user_service.find_all()

@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    current_user: User = Depends(require_role(["admin", "moderator"]))
):
    post_service.delete(post_id)
```

---

## 15.2 Role 모델 설계

### 데이터베이스 모델

```python
# app/models/role.py
from sqlalchemy import Column, Integer, String, Table, ForeignKey
from sqlalchemy.orm import relationship
from app.database import Base

# User-Role 중간 테이블 (Many-to-Many)
user_roles = Table(
    'user_roles',
    Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id'), primary_key=True),
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True)
)

class Role(Base):
    __tablename__ = "roles"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False)  # admin, user, moderator
    description = Column(String(255))

    # 관계
    users = relationship("User", secondary=user_roles, back_populates="roles")
    permissions = relationship("Permission", secondary="role_permissions", back_populates="roles")

# app/models/user.py 수정
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(100), unique=True, nullable=False)
    username = Column(String(50), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)

    # Role 관계 추가
    roles = relationship("Role", secondary=user_roles, back_populates="users")
```

**Spring Boot JPA와 비교:**
```java
// Spring Boot
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String username;
    private String password;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}

@Entity
@Table(name = "roles")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany(mappedBy = "roles")
    private Set<User> users = new HashSet<>();
}
```

---

## 15.3 Permission 시스템

### Permission 모델

```python
# app/models/permission.py
from sqlalchemy import Column, Integer, String, Table, ForeignKey
from sqlalchemy.orm import relationship
from app.database import Base

# Role-Permission 중간 테이블
role_permissions = Table(
    'role_permissions',
    Base.metadata,
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True),
    Column('permission_id', Integer, ForeignKey('permissions.id'), primary_key=True)
)

class Permission(Base):
    __tablename__ = "permissions"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, nullable=False)  # posts:read, posts:write
    description = Column(String(255))
    resource = Column(String(50))  # posts, users, comments
    action = Column(String(50))    # read, write, delete

    # 관계
    roles = relationship("Role", secondary=role_permissions, back_populates="permissions")
```

### 초기 데이터 생성

```python
# app/db/init_db.py
from sqlalchemy.orm import Session
from app.models.role import Role
from app.models.permission import Permission

def init_roles_and_permissions(db: Session):
    """초기 역할 및 권한 설정"""

    # Permission 생성
    permissions = [
        Permission(name="posts:read", resource="posts", action="read", description="게시글 조회"),
        Permission(name="posts:write", resource="posts", action="write", description="게시글 작성"),
        Permission(name="posts:delete", resource="posts", action="delete", description="게시글 삭제"),
        Permission(name="users:read", resource="users", action="read", description="사용자 조회"),
        Permission(name="users:write", resource="users", action="write", description="사용자 수정"),
        Permission(name="users:delete", resource="users", action="delete", description="사용자 삭제"),
    ]

    for permission in permissions:
        existing = db.query(Permission).filter(Permission.name == permission.name).first()
        if not existing:
            db.add(permission)

    db.commit()

    # Role 생성
    # 1. User Role (일반 사용자)
    user_role = db.query(Role).filter(Role.name == "user").first()
    if not user_role:
        user_role = Role(name="user", description="일반 사용자")
        user_role.permissions = [
            db.query(Permission).filter(Permission.name == "posts:read").first(),
            db.query(Permission).filter(Permission.name == "posts:write").first(),
        ]
        db.add(user_role)

    # 2. Moderator Role (운영자)
    moderator_role = db.query(Role).filter(Role.name == "moderator").first()
    if not moderator_role:
        moderator_role = Role(name="moderator", description="운영자")
        moderator_role.permissions = [
            db.query(Permission).filter(Permission.name == "posts:read").first(),
            db.query(Permission).filter(Permission.name == "posts:write").first(),
            db.query(Permission).filter(Permission.name == "posts:delete").first(),
            db.query(Permission).filter(Permission.name == "users:read").first(),
        ]
        db.add(moderator_role)

    # 3. Admin Role (관리자) - 모든 권한
    admin_role = db.query(Role).filter(Role.name == "admin").first()
    if not admin_role:
        admin_role = Role(name="admin", description="관리자")
        admin_role.permissions = db.query(Permission).all()
        db.add(admin_role)

    db.commit()
```

---

## 15.4 역할 기반 의존성

### Role 체크 의존성

```python
# app/api/deps.py
from typing import List
from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session

from app.database import get_db
from app.models.user import User
from app.api.deps import get_current_user

def require_role(allowed_roles: List[str]):
    """
    특정 역할을 가진 사용자만 허용하는 의존성

    Usage:
        @app.get("/admin/users")
        async def get_users(
            current_user: User = Depends(require_role(["admin"]))
        ):
            ...
    """
    def role_checker(current_user: User = Depends(get_current_user)):
        user_role_names = [role.name for role in current_user.roles]

        # 사용자가 허용된 역할 중 하나라도 가지고 있는지 확인
        if not any(role in allowed_roles for role in user_role_names):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"User does not have required role. Required: {allowed_roles}"
            )

        return current_user

    return role_checker

def require_admin(current_user: User = Depends(get_current_user)):
    """관리자만 허용"""
    return require_role(["admin"])(current_user)

def require_moderator_or_admin(current_user: User = Depends(get_current_user)):
    """운영자 또는 관리자만 허용"""
    return require_role(["moderator", "admin"])(current_user)
```

**Spring Security와 비교:**
```java
// Spring Security
@PreAuthorize("hasRole('ADMIN')")
public void adminOnlyMethod() { }

@PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
public void moderatorOrAdminMethod() { }

// 커스텀 메서드 보안
@PreAuthorize("@securityService.hasPermission(#userId, 'USER_READ')")
public User getUser(@Param("userId") Long userId) { }
```

---

## 15.5 Permission 기반 의존성

### Permission 체크

```python
# app/api/deps.py
def require_permission(required_permissions: List[str]):
    """
    특정 권한을 가진 사용자만 허용

    Usage:
        @app.delete("/posts/{post_id}")
        async def delete_post(
            post_id: int,
            current_user: User = Depends(require_permission(["posts:delete"]))
        ):
            ...
    """
    def permission_checker(current_user: User = Depends(get_current_user)):
        # 사용자의 모든 권한 수집
        user_permissions = []
        for role in current_user.roles:
            for permission in role.permissions:
                user_permissions.append(permission.name)

        # 필요한 권한이 모두 있는지 확인
        for required_permission in required_permissions:
            if required_permission not in user_permissions:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail=f"Missing required permission: {required_permission}"
                )

        return current_user

    return permission_checker

# 리소스별 헬퍼 함수
def can_read_posts(current_user: User = Depends(get_current_user)):
    return require_permission(["posts:read"])(current_user)

def can_write_posts(current_user: User = Depends(get_current_user)):
    return require_permission(["posts:write"])(current_user)

def can_delete_posts(current_user: User = Depends(get_current_user)):
    return require_permission(["posts:delete"])(current_user)
```

---

## 15.6 실전 예제: 게시판 권한

### 게시글 API

```python
# app/api/routers/posts.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from app.database import get_db
from app.api.deps import get_current_user, require_permission
from app.models.user import User
from app.models.post import Post
from app.schemas.post import PostCreate, PostUpdate, PostResponse

router = APIRouter(prefix="/posts", tags=["posts"])

@router.get("/", response_model=List[PostResponse])
async def list_posts(
    skip: int = 0,
    limit: int = 10,
    current_user: User = Depends(require_permission(["posts:read"])),
    db: Session = Depends(get_db)
):
    """게시글 목록 조회 - posts:read 권한 필요"""
    posts = db.query(Post).offset(skip).limit(limit).all()
    return posts

@router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: int,
    current_user: User = Depends(require_permission(["posts:read"])),
    db: Session = Depends(get_db)
):
    """게시글 조회 - posts:read 권한 필요"""
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return post

@router.post("/", response_model=PostResponse, status_code=status.HTTP_201_CREATED)
async def create_post(
    post_data: PostCreate,
    current_user: User = Depends(require_permission(["posts:write"])),
    db: Session = Depends(get_db)
):
    """게시글 작성 - posts:write 권한 필요"""
    post = Post(
        **post_data.dict(),
        author_id=current_user.id
    )
    db.add(post)
    db.commit()
    db.refresh(post)
    return post

@router.put("/{post_id}", response_model=PostResponse)
async def update_post(
    post_id: int,
    post_data: PostUpdate,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """게시글 수정 - 작성자 본인 또는 moderator/admin"""
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # 권한 체크: 작성자 본인 또는 moderator/admin
    user_roles = [role.name for role in current_user.roles]
    is_author = post.author_id == current_user.id
    is_privileged = "admin" in user_roles or "moderator" in user_roles

    if not (is_author or is_privileged):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not authorized to update this post"
        )

    # 수정
    for key, value in post_data.dict(exclude_unset=True).items():
        setattr(post, key, value)

    db.commit()
    db.refresh(post)
    return post

@router.delete("/{post_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_post(
    post_id: int,
    current_user: User = Depends(require_permission(["posts:delete"])),
    db: Session = Depends(get_db)
):
    """게시글 삭제 - posts:delete 권한 필요 (admin/moderator)"""
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    db.delete(post)
    db.commit()
```

---

## 15.7 동적 권한 체크

### 리소스 소유권 체크

```python
# app/services/authorization.py
from sqlalchemy.orm import Session
from app.models.user import User
from app.models.post import Post

class AuthorizationService:
    @staticmethod
    def can_modify_post(user: User, post: Post) -> bool:
        """게시글 수정 권한 체크"""
        # 1. 작성자 본인
        if post.author_id == user.id:
            return True

        # 2. Admin 또는 Moderator
        user_roles = [role.name for role in user.roles]
        if "admin" in user_roles or "moderator" in user_roles:
            return True

        return False

    @staticmethod
    def can_delete_post(user: User, post: Post) -> bool:
        """게시글 삭제 권한 체크"""
        user_roles = [role.name for role in user.roles]

        # Admin만 삭제 가능 또는 작성자 본인 (설정에 따라)
        return "admin" in user_roles

    @staticmethod
    def has_permission(user: User, permission_name: str) -> bool:
        """특정 권한 보유 여부 확인"""
        for role in user.roles:
            for permission in role.permissions:
                if permission.name == permission_name:
                    return True
        return False

# 사용
from app.services.authorization import AuthorizationService

@router.put("/posts/{post_id}")
async def update_post(
    post_id: int,
    post_data: PostUpdate,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # 동적 권한 체크
    if not AuthorizationService.can_modify_post(current_user, post):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not authorized to modify this post"
        )

    # 수정 로직
    # ...
```

**Spring Security와 비교:**
```java
// Spring Security - @PostAuthorize
@Service
public class PostService {
    @PostAuthorize("returnObject.author.id == authentication.principal.id or hasRole('ADMIN')")
    public Post getPost(Long postId) {
        return postRepository.findById(postId).orElseThrow();
    }

    @PreAuthorize("@authorizationService.canModifyPost(authentication.principal, #postId)")
    public Post updatePost(Long postId, PostUpdateDto updateDto) {
        // ...
    }
}

// AuthorizationService
@Service
public class AuthorizationService {
    public boolean canModifyPost(UserDetails user, Long postId) {
        Post post = postRepository.findById(postId).orElseThrow();
        return post.getAuthor().getUsername().equals(user.getUsername())
            || user.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
    }
}
```

---

## 15.8 관리자 API

### 역할 관리

```python
# app/api/routers/admin.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from app.database import get_db
from app.api.deps import require_role
from app.models.user import User
from app.models.role import Role
from app.schemas.user import UserResponse

router = APIRouter(prefix="/admin", tags=["admin"])

@router.post("/users/{user_id}/roles/{role_name}")
async def assign_role_to_user(
    user_id: int,
    role_name: str,
    current_user: User = Depends(require_role(["admin"])),
    db: Session = Depends(get_db)
):
    """사용자에게 역할 부여 - admin만 가능"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    role = db.query(Role).filter(Role.name == role_name).first()
    if not role:
        raise HTTPException(status_code=404, detail="Role not found")

    # 역할 추가
    if role not in user.roles:
        user.roles.append(role)
        db.commit()

    return {"message": f"Role '{role_name}' assigned to user {user_id}"}

@router.delete("/users/{user_id}/roles/{role_name}")
async def remove_role_from_user(
    user_id: int,
    role_name: str,
    current_user: User = Depends(require_role(["admin"])),
    db: Session = Depends(get_db)
):
    """사용자로부터 역할 제거 - admin만 가능"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    role = db.query(Role).filter(Role.name == role_name).first()
    if not role:
        raise HTTPException(status_code=404, detail="Role not found")

    # 역할 제거
    if role in user.roles:
        user.roles.remove(role)
        db.commit()

    return {"message": f"Role '{role_name}' removed from user {user_id}"}

@router.get("/users", response_model=List[UserResponse])
async def list_all_users(
    current_user: User = Depends(require_role(["admin"])),
    db: Session = Depends(get_db)
):
    """모든 사용자 조회 - admin만 가능"""
    users = db.query(User).all()
    return users
```

---

## 15.9 핵심 요약

### RBAC 구조

```
User (사용자)
  ↓ Many-to-Many
Role (역할: admin, moderator, user)
  ↓ Many-to-Many
Permission (권한: posts:read, posts:write, posts:delete)
```

### Spring Security vs FastAPI

| 기능 | Spring Security | FastAPI |
|------|----------------|---------|
| **역할 체크** | `@PreAuthorize("hasRole('ADMIN')")` | `Depends(require_role(["admin"]))` |
| **권한 체크** | `@PreAuthorize("hasAuthority('READ')")` | `Depends(require_permission(["posts:read"]))` |
| **동적 체크** | `@PreAuthorize("@service.check()")` | `AuthorizationService.can_modify()` |
| **설정** | SecurityConfig + annotations | 의존성 함수 |

### 기억할 핵심

1. ✅ **Role**: 사용자의 역할 (admin, user, moderator)
2. ✅ **Permission**: 세밀한 권한 (resource:action)
3. ✅ **Many-to-Many**: User ↔ Role ↔ Permission
4. ✅ **의존성 체크**: require_role(), require_permission()
5. ✅ **동적 체크**: 리소스 소유권 확인

---

## 15.10 실습 예제

### 실습 1: 댓글 권한 시스템

**요구사항:**
- 모든 사용자: 댓글 조회, 작성
- 작성자: 자신의 댓글 수정/삭제
- Moderator: 모든 댓글 삭제

### 실습 2: 동적 권한 체크

**요구사항:**
- 게시글 작성자는 자신의 글만 수정 가능
- Admin은 모든 글 수정 가능
- Moderator는 부적절한 글 삭제 가능

---

## 15.11 다음 챕터 예고

다음 챕터에서는 **보안 Best Practices**를 다룹니다:
- CORS 설정
- CSRF 방지
- SQL Injection 방지
- XSS 방지
- Rate Limiting
- 환경 변수 관리

---

[← 이전: Chapter 14. OAuth2 인증](chapter14-oauth2.md) | [목차로](../README.md) | [다음: Chapter 16. 보안 Best Practices →](chapter16-security-best-practices.md)
