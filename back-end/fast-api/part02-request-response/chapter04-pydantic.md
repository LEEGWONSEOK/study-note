# Chapter 4. Pydantic 모델

## 4.1 Pydantic이란?

Pydantic은 Python의 타입 힌트를 사용하여 **데이터 검증과 설정 관리**를 수행하는 라이브러리입니다. FastAPI의 핵심 기능 중 하나입니다.

### Spring Boot와의 비교

**Spring Boot에서:**
- DTO 클래스 생성
- Bean Validation 애노테이션 (`@NotNull`, `@Size`, `@Email` 등)
- Getter/Setter 필요
- `@Valid` 애노테이션으로 검증 활성화

**FastAPI/Pydantic에서:**
- Pydantic 모델 클래스 생성
- 타입 힌트로 자동 검증
- 추가 검증은 `Field()` 사용
- 자동으로 검증됨 (별도 애노테이션 불필요)

---

## 4.2 기본 Pydantic 모델

### 간단한 모델 정의

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    email: str
    age: int | None = None  # 선택적 필드
    is_active: bool = True   # 기본값
```

### Spring Boot DTO와 비교

```java
// Spring Boot
public class UserDto {
    @NotNull
    private Long id;

    @NotBlank
    private String name;

    @Email
    private String email;

    private Integer age;

    private Boolean isActive = true;

    // Getters, Setters, Constructor 필요
}
```

```python
# FastAPI - 훨씬 간결!
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    email: str
    age: int | None = None
    is_active: bool = True

# Getter/Setter 불필요, 자동 생성됨!
```

### 사용 예제

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    name: str
    email: str

@app.post("/users")
def create_user(user: User):
    # Pydantic이 자동으로 검증
    return user

# 올바른 요청
# POST /users
# {"id": 1, "name": "John", "email": "john@example.com"}
# ✅ 성공

# 잘못된 요청
# {"id": "not_a_number", "name": "John", "email": "john@example.com"}
# ❌ 422 Validation Error
```

---

## 4.3 타입 힌트와 자동 검증

Pydantic은 Python의 타입 힌트를 사용하여 자동으로 데이터를 검증합니다.

### 기본 타입

```python
from pydantic import BaseModel
from typing import List, Dict, Set, Tuple

class DataTypes(BaseModel):
    # 기본 타입
    integer: int
    floating: float
    text: str
    flag: bool

    # 컬렉션
    tags: List[str]
    metadata: Dict[str, str]
    unique_ids: Set[int]
    coordinates: Tuple[float, float]

    # 선택적 (Optional)
    description: str | None = None
    count: int | None = None
```

### 자동 타입 변환

Pydantic은 가능한 경우 타입을 자동으로 변환합니다.

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    quantity: int

# 자동 변환 예제
item = Item(
    name="Laptop",
    price="1299.99",  # str -> float 자동 변환
    quantity="5"       # str -> int 자동 변환
)

print(item.price)    # 1299.99 (float)
print(item.quantity) # 5 (int)
```

**Spring Boot에서는:**
```java
// Spring Boot는 @RequestBody로 JSON을 역직렬화할 때 자동 변환
// 하지만 타입이 맞지 않으면 400 Bad Request
```

---

## 4.4 Field를 사용한 고급 검증

`Field()`를 사용하여 추가적인 검증 규칙을 정의할 수 있습니다.

### 문자열 검증

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    password: str = Field(..., min_length=8)
    email: str = Field(..., regex=r"^[\w\.-]+@[\w\.-]+\.\w+$")
    bio: str | None = Field(None, max_length=500)

# Spring Boot 비교
# @Size(min=3, max=50)
# private String username;
#
# @Size(min=8)
# private String password;
#
# @Email
# private String email;
#
# @Size(max=500)
# private String bio;
```

### 숫자 검증

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str
    price: float = Field(..., gt=0, description="가격은 0보다 커야 함")
    discount: float = Field(0, ge=0, le=100, description="할인율 0-100%")
    stock: int = Field(..., ge=0, description="재고는 0 이상")
    rating: float = Field(None, ge=0, le=5, description="평점 0-5")

# Field 검증 옵션:
# gt (greater than): >
# ge (greater than or equal): >=
# lt (less than): <
# le (less than or equal): <=
```

**Spring Boot 비교:**
```java
@Positive
private Double price;

@Min(0) @Max(100)
private Double discount;

@PositiveOrZero
private Integer stock;

@DecimalMin("0.0") @DecimalMax("5.0")
private Double rating;
```

### 리스트 검증

```python
from pydantic import BaseModel, Field
from typing import List

class Article(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    tags: List[str] = Field(..., min_items=1, max_items=10)
    authors: List[str] = Field(default_factory=list)

# 요청 예제
# {
#   "title": "FastAPI Tutorial",
#   "tags": ["python", "fastapi", "tutorial"]
# }
```

---

## 4.5 중첩 모델 (Nested Models)

Pydantic 모델 안에 다른 Pydantic 모델을 포함할 수 있습니다.

### 기본 중첩

```python
from pydantic import BaseModel
from typing import List

class Address(BaseModel):
    street: str
    city: str
    country: str
    zip_code: str

class User(BaseModel):
    id: int
    name: str
    email: str
    address: Address  # 중첩 모델

# 사용 예제
@app.post("/users")
def create_user(user: User):
    return user

# 요청 데이터
# {
#   "id": 1,
#   "name": "John",
#   "email": "john@example.com",
#   "address": {
#     "street": "123 Main St",
#     "city": "Seoul",
#     "country": "Korea",
#     "zip_code": "12345"
#   }
# }
```

### 복잡한 중첩 구조

```python
from pydantic import BaseModel
from typing import List
from datetime import datetime

class Comment(BaseModel):
    id: int
    content: str
    created_at: datetime

class Author(BaseModel):
    id: int
    name: str
    email: str

class Post(BaseModel):
    id: int
    title: str
    content: str
    author: Author
    comments: List[Comment] = []
    tags: List[str] = []

# Spring Boot에서는 여러 DTO 클래스 필요
```

---

## 4.6 Config 클래스

`Config` 클래스를 사용하여 모델의 동작을 커스터마이징할 수 있습니다.

### 기본 설정

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    email: str

    class Config:
        # ORM 객체를 Pydantic 모델로 변환 가능
        from_attributes = True  # Pydantic v2 (구버전: orm_mode = True)

        # 예제 데이터 (Swagger UI에 표시)
        json_schema_extra = {
            "example": {
                "id": 1,
                "name": "John Doe",
                "email": "john@example.com"
            }
        }
```

### 유용한 Config 옵션

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

    class Config:
        # 추가 필드 허용 안함 (기본값)
        extra = "forbid"  # "allow", "ignore"도 가능

        # 불변 모델 (값 변경 불가)
        frozen = True

        # 문자열 자동 strip
        str_strip_whitespace = True

        # 대소문자 구분 안함
        str_to_lower = True
        str_to_upper = False

        # 최소/최대값 검증을 inclusive하게
        validate_assignment = True
```

---

## 4.7 Pydantic Validators

커스텀 검증 로직을 추가할 수 있습니다.

### field_validator (Pydantic v2)

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    username: str
    password: str
    email: str

    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('username은 영문자와 숫자만 가능합니다')
        return v

    @field_validator('password')
    @classmethod
    def password_strength(cls, v):
        if len(v) < 8:
            raise ValueError('비밀번호는 최소 8자 이상이어야 합니다')
        if not any(char.isdigit() for char in v):
            raise ValueError('비밀번호는 최소 1개의 숫자를 포함해야 합니다')
        if not any(char.isupper() for char in v):
            raise ValueError('비밀번호는 최소 1개의 대문자를 포함해야 합니다')
        return v
```

**Spring Boot에서는:**
```java
// 커스텀 Validator 클래스 생성 필요
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordValidator.class)
public @interface ValidPassword {
    // ...
}
```

### model_validator (모델 전체 검증)

```python
from pydantic import BaseModel, model_validator

class UserRegistration(BaseModel):
    password: str
    password_confirm: str
    email: str

    @model_validator(mode='after')
    def check_passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError('비밀번호가 일치하지 않습니다')
        return self
```

---

## 4.8 요청/응답 모델 분리

보안을 위해 요청과 응답 모델을 분리하는 것이 좋습니다.

### 패스워드 숨기기

```python
from pydantic import BaseModel, EmailStr

# 요청 모델 (패스워드 포함)
class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str
    full_name: str | None = None

# 응답 모델 (패스워드 제외)
class UserResponse(BaseModel):
    id: int
    username: str
    email: EmailStr
    full_name: str | None = None

    class Config:
        from_attributes = True

# 사용
@app.post("/users", response_model=UserResponse)
def create_user(user: UserCreate):
    # DB에 저장 (패스워드는 해싱해서 저장)
    # ...
    return {
        "id": 1,
        "username": user.username,
        "email": user.email,
        "full_name": user.full_name,
        # password는 응답에 포함되지 않음!
    }
```

**Spring Boot와 비교:**
```java
// Spring Boot - 별도의 Request/Response DTO 필요
public class UserCreateDto { /* ... */ }
public class UserResponseDto { /* ... */ }

@PostMapping("/users")
public UserResponseDto createUser(@RequestBody UserCreateDto user) {
    // ModelMapper 등으로 변환 필요
}
```

---

## 4.9 실전 예제: E-Commerce 모델

```python
from pydantic import BaseModel, Field, EmailStr
from typing import List
from datetime import datetime
from enum import Enum

class OrderStatus(str, Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class ProductBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=200)
    description: str | None = Field(None, max_length=1000)
    price: float = Field(..., gt=0)
    stock: int = Field(..., ge=0)

class ProductCreate(ProductBase):
    category_id: int

class ProductResponse(ProductBase):
    id: int
    category_id: int
    created_at: datetime

    class Config:
        from_attributes = True

class OrderItem(BaseModel):
    product_id: int
    quantity: int = Field(..., gt=0)
    price: float = Field(..., gt=0)

class OrderCreate(BaseModel):
    items: List[OrderItem] = Field(..., min_items=1)
    shipping_address: str

    @field_validator('items')
    @classmethod
    def validate_items(cls, v):
        if len(v) > 100:
            raise ValueError('주문 아이템은 최대 100개까지 가능합니다')
        return v

class OrderResponse(BaseModel):
    id: int
    user_id: int
    items: List[OrderItem]
    total_amount: float
    status: OrderStatus
    shipping_address: str
    created_at: datetime

    class Config:
        from_attributes = True
```

---

## 4.10 핵심 요약

### Pydantic vs Spring Boot Validation

| 기능 | Spring Boot | Pydantic |
|------|-------------|----------|
| **기본 검증** | `@NotNull`, `@NotBlank` | 타입 힌트 |
| **문자열 길이** | `@Size(min=, max=)` | `Field(min_length=, max_length=)` |
| **숫자 범위** | `@Min`, `@Max` | `Field(ge=, le=)` |
| **이메일** | `@Email` | `EmailStr` |
| **정규표현식** | `@Pattern` | `Field(regex=)` |
| **커스텀 검증** | 별도 Validator 클래스 | `@field_validator` |
| **중첩 객체** | `@Valid` 중첩 | 자동 검증 |
| **JSON 변환** | Jackson | 자동 처리 |

### 기억할 핵심

1. ✅ **타입 힌트 = 검증**: 타입만 명시하면 자동 검증
2. ✅ **Field()**: 추가 제약조건 설정
3. ✅ **중첩 모델**: 복잡한 구조도 쉽게 표현
4. ✅ **Config**: 모델 동작 커스터마이징
5. ✅ **Validator**: 커스텀 검증 로직
6. ✅ **요청/응답 분리**: 보안과 명확성

---

## 4.11 실습 예제

### 실습 1: 블로그 Post 모델

**요구사항:**
- 제목: 1-200자
- 내용: 필수
- 작성자: 이메일 형식
- 태그: 최소 1개, 최대 5개
- 공개 여부: 기본값 false

**정답:**
```python
from pydantic import BaseModel, Field, EmailStr
from typing import List

class PostCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    content: str
    author_email: EmailStr
    tags: List[str] = Field(..., min_items=1, max_items=5)
    is_published: bool = False

    class Config:
        json_schema_extra = {
            "example": {
                "title": "FastAPI 시작하기",
                "content": "FastAPI는 빠르고 쉬운...",
                "author_email": "john@example.com",
                "tags": ["python", "fastapi"],
                "is_published": False
            }
        }
```

### 실습 2: 회원가입 검증

**요구사항:**
- username: 영문자/숫자만, 3-20자
- password: 최소 8자, 숫자/대문자 포함
- password_confirm: password와 일치
- age: 14세 이상

**정답:**
```python
from pydantic import BaseModel, Field, field_validator, model_validator

class UserSignup(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    password: str = Field(..., min_length=8)
    password_confirm: str
    age: int = Field(..., ge=14)

    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('영문자와 숫자만 사용 가능합니다')
        return v

    @field_validator('password')
    @classmethod
    def password_strength(cls, v):
        if not any(char.isdigit() for char in v):
            raise ValueError('최소 1개의 숫자 필요')
        if not any(char.isupper() for char in v):
            raise ValueError('최소 1개의 대문자 필요')
        return v

    @model_validator(mode='after')
    def check_passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError('비밀번호가 일치하지 않습니다')
        return self
```

---

## 4.12 연습 문제

### 문제 1
Pydantic의 `Field()`에서 사용 가능한 검증 옵션 10가지를 나열하세요.

### 문제 2
다음 Spring Boot DTO를 Pydantic 모델로 변환하세요:

```java
public class BookDto {
    @NotBlank
    @Size(min = 1, max = 300)
    private String title;

    @NotBlank
    private String author;

    @Positive
    private Double price;

    @Min(1) @Max(5)
    private Integer rating;

    @PastOrPresent
    private LocalDate publishedDate;
}
```

### 문제 3
사용자 프로필 업데이트 API를 위한 Pydantic 모델을 작성하세요. 모든 필드는 선택적이어야 하며, 최소 1개 이상의 필드는 반드시 포함되어야 합니다.

---

## 4.13 다음 챕터 예고

다음 챕터에서는 **요청 데이터 검증**을 다룹니다:
- Query, Path, Body, Header, Cookie 파라미터
- 복잡한 검증 규칙
- 커스텀 검증기
- 에러 응답 포맷

**FastAPI의 강력한 검증 시스템을 완전히 마스터합니다!**

---

[← 이전: Chapter 3. 자동 문서화](../part01-getting-started/chapter03-documentation.md) | [목차로](../README.md) | [다음: Chapter 5. 요청 데이터 검증 →](chapter05-validation.md)