# Chapter 5. 요청 데이터 검증

## 5.1 요청 파라미터 타입

FastAPI는 여러 소스에서 데이터를 받을 수 있습니다.

### 파라미터 타입 비교

| 파라미터 타입 | Spring Boot | FastAPI | 위치 |
|--------------|-------------|---------|------|
| **Path** | `@PathVariable` | `Path()` | URL 경로 |
| **Query** | `@RequestParam` | `Query()` | URL 쿼리 스트링 |
| **Body** | `@RequestBody` | `Body()` | Request Body |
| **Header** | `@RequestHeader` | `Header()` | HTTP 헤더 |
| **Cookie** | `@CookieValue` | `Cookie()` | 쿠키 |
| **Form** | `@ModelAttribute` | `Form()` | 폼 데이터 |
| **File** | `MultipartFile` | `File()` | 파일 업로드 |

---

## 5.2 Path 파라미터 검증

URL 경로에 포함된 변수를 검증합니다.

### 기본 사용

```python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/items/{item_id}")
def get_item(
    item_id: int = Path(..., title="아이템 ID", ge=1, le=1000)
):
    return {"item_id": item_id}

# 올바른 요청: /items/42 ✅
# 잘못된 요청: /items/0 ❌ (ge=1 위반)
# 잘못된 요청: /items/1001 ❌ (le=1000 위반)
```

### Spring Boot와 비교

```java
// Spring Boot
@GetMapping("/items/{itemId}")
public ResponseEntity<Item> getItem(
    @PathVariable
    @Min(1) @Max(1000)
    Integer itemId
) {
    return ResponseEntity.ok(itemService.find(itemId));
}
```

```python
# FastAPI - 더 직관적
from fastapi import Path

@app.get("/items/{item_id}")
def get_item(item_id: int = Path(..., ge=1, le=1000)):
    return {"item_id": item_id}
```

### 고급 Path 검증

```python
from fastapi import FastAPI, Path
from enum import Enum

app = FastAPI()

# 문자열 길이 검증
@app.get("/users/{username}")
def get_user(
    username: str = Path(..., min_length=3, max_length=50, regex="^[a-zA-Z0-9_]+$")
):
    return {"username": username}

# 설명 추가
@app.get("/products/{product_id}")
def get_product(
    product_id: int = Path(
        ...,
        title="상품 ID",
        description="조회할 상품의 고유 ID",
        ge=1,
        example=42
    )
):
    return {"product_id": product_id}
```

---

## 5.3 Query 파라미터 검증

URL의 쿼리 스트링 파라미터를 검증합니다.

### 기본 Query 검증

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items")
def list_items(
    skip: int = Query(0, ge=0, description="건너뛸 개수"),
    limit: int = Query(10, ge=1, le=100, description="가져올 개수"),
    search: str | None = Query(None, min_length=1, max_length=50)
):
    return {
        "skip": skip,
        "limit": limit,
        "search": search
    }

# 요청 예: /items?skip=0&limit=20&search=laptop
```

### Spring Boot와 비교

```java
// Spring Boot
@GetMapping("/items")
public List<Item> listItems(
    @RequestParam(defaultValue = "0") @Min(0) Integer skip,
    @RequestParam(defaultValue = "10") @Min(1) @Max(100) Integer limit,
    @RequestParam(required = false) @Size(min=1, max=50) String search
) {
    return itemService.list(skip, limit, search);
}
```

```python
# FastAPI - 한 줄로 간결하게
@app.get("/items")
def list_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    search: str | None = Query(None, min_length=1, max_length=50)
):
    return {"skip": skip, "limit": limit, "search": search}
```

### 필수 Query 파라미터

```python
from fastapi import Query

@app.get("/search")
def search(
    q: str = Query(..., min_length=1, description="검색어 (필수)")
):
    return {"query": q}

# /search?q=fastapi ✅
# /search ❌ (필수 파라미터 누락)
```

### 여러 값을 받는 Query 파라미터

```python
from fastapi import Query
from typing import List

@app.get("/items")
def filter_items(
    tags: List[str] = Query([], description="필터링할 태그들")
):
    return {"tags": tags}

# /items?tags=electronics&tags=computers&tags=laptop
# 결과: {"tags": ["electronics", "computers", "laptop"]}
```

### Query 별칭 (alias)

```python
from fastapi import Query

@app.get("/items")
def get_items(
    item_query: str = Query(None, alias="item-query")
):
    return {"item_query": item_query}

# /items?item-query=laptop
# Python에서는 item_query로 접근 (snake_case)
# URL에서는 item-query 사용 (kebab-case)
```

---

## 5.4 Request Body 검증

POST, PUT, PATCH 요청의 Body 데이터를 검증합니다.

### 단일 Body 파라미터

```python
from fastapi import FastAPI, Body
from pydantic import BaseModel, Field

app = FastAPI()

class Item(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    description: str | None = Field(None, max_length=500)
    price: float = Field(..., gt=0)
    tax: float | None = Field(None, ge=0, le=100)

@app.post("/items")
def create_item(item: Item):
    return item
```

### 여러 Body 파라미터

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

class User(BaseModel):
    username: str
    email: str

@app.post("/offers")
def create_offer(
    item: Item,
    user: User,
    importance: int = Body(...)
):
    return {"item": item, "user": user, "importance": importance}

# 요청 Body:
# {
#   "item": {"name": "Laptop", "price": 1299.99},
#   "user": {"username": "john", "email": "john@example.com"},
#   "importance": 5
# }
```

### Body embed

```python
from fastapi import Body

@app.post("/items")
def create_item(item: Item = Body(..., embed=True)):
    return item

# embed=False (기본값):
# {"name": "Laptop", "price": 1299.99}

# embed=True:
# {"item": {"name": "Laptop", "price": 1299.99}}
```

---

## 5.5 Header 파라미터 검증

HTTP 헤더 값을 검증합니다.

### 기본 Header

```python
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/items")
def read_items(
    user_agent: str | None = Header(None),
    x_token: str = Header(..., description="인증 토큰")
):
    return {"User-Agent": user_agent, "X-Token": x_token}

# 헤더:
# User-Agent: Mozilla/5.0
# X-Token: secret-token-123
```

### Spring Boot와 비교

```java
// Spring Boot
@GetMapping("/items")
public ResponseEntity<?> readItems(
    @RequestHeader("User-Agent") String userAgent,
    @RequestHeader("X-Token") String xToken
) {
    return ResponseEntity.ok(Map.of("userAgent", userAgent, "xToken", xToken));
}
```

```python
# FastAPI - 자동 변환 (User-Agent -> user_agent)
@app.get("/items")
def read_items(
    user_agent: str | None = Header(None),
    x_token: str = Header(...)
):
    return {"user_agent": user_agent, "x_token": x_token}
```

### 여러 값을 가진 Header

```python
from typing import List

@app.get("/items")
def read_items(
    x_token: List[str] | None = Header(None)
):
    return {"X-Token values": x_token}

# 헤더:
# X-Token: token1
# X-Token: token2
# 결과: {"X-Token values": ["token1", "token2"]}
```

---

## 5.6 Cookie 파라미터 검증

쿠키 값을 검증합니다.

### 기본 Cookie

```python
from fastapi import FastAPI, Cookie

app = FastAPI()

@app.get("/items")
def read_items(
    session_id: str | None = Cookie(None),
    ads_id: str | None = Cookie(None)
):
    return {"session_id": session_id, "ads_id": ads_id}
```

### Spring Boot와 비교

```java
// Spring Boot
@GetMapping("/items")
public ResponseEntity<?> readItems(
    @CookieValue(required = false) String sessionId,
    @CookieValue(required = false) String adsId
) {
    return ResponseEntity.ok(Map.of("sessionId", sessionId, "adsId", adsId));
}
```

---

## 5.7 Form 데이터

HTML 폼 데이터를 처리합니다.

### Form 파라미터

```python
from fastapi import FastAPI, Form

app = FastAPI()

@app.post("/login")
def login(
    username: str = Form(...),
    password: str = Form(...)
):
    return {"username": username}

# Content-Type: application/x-www-form-urlencoded
# username=john&password=secret
```

### Spring Boot와 비교

```java
// Spring Boot
@PostMapping("/login")
public ResponseEntity<?> login(
    @RequestParam String username,
    @RequestParam String password
) {
    return ResponseEntity.ok(Map.of("username", username));
}
```

### Form + File

```python
from fastapi import Form, File, UploadFile

@app.post("/signup")
def signup(
    username: str = Form(...),
    email: str = Form(...),
    profile_picture: UploadFile = File(...)
):
    return {
        "username": username,
        "email": email,
        "filename": profile_picture.filename
    }
```

---

## 5.8 파일 업로드

파일 업로드를 처리하고 검증합니다.

### 단일 파일 업로드

```python
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

# bytes로 받기 (작은 파일)
@app.post("/upload")
def upload_file(file: bytes = File(...)):
    return {"file_size": len(file)}

# UploadFile 사용 (권장)
@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents)
    }
```

### 여러 파일 업로드

```python
from typing import List

@app.post("/upload-multiple")
async def upload_multiple_files(files: List[UploadFile] = File(...)):
    return {
        "filenames": [file.filename for file in files],
        "count": len(files)
    }
```

### 파일 크기 제한

```python
from fastapi import HTTPException

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    # 최대 10MB
    max_size = 10 * 1024 * 1024
    contents = await file.read()

    if len(contents) > max_size:
        raise HTTPException(status_code=413, detail="파일이 너무 큽니다")

    return {"filename": file.filename, "size": len(contents)}
```

### Spring Boot와 비교

```java
// Spring Boot
@PostMapping("/upload")
public ResponseEntity<?> upload(@RequestParam("file") MultipartFile file) {
    if (file.getSize() > 10 * 1024 * 1024) {
        throw new FileTooLargeException();
    }
    return ResponseEntity.ok(Map.of("filename", file.getOriginalFilename()));
}
```

---

## 5.9 커스텀 검증기

복잡한 검증 로직을 구현합니다.

### Pydantic Validator 사용

```python
from pydantic import BaseModel, field_validator, model_validator

class UserCreate(BaseModel):
    username: str
    email: str
    password: str
    password_confirm: str
    age: int

    @field_validator('username')
    @classmethod
    def username_valid(cls, v):
        if len(v) < 3:
            raise ValueError('username은 3자 이상이어야 합니다')
        if not v.isalnum():
            raise ValueError('username은 영문자와 숫자만 가능합니다')
        return v.lower()  # 소문자로 변환

    @field_validator('age')
    @classmethod
    def age_valid(cls, v):
        if v < 14:
            raise ValueError('14세 이상만 가입 가능합니다')
        if v > 120:
            raise ValueError('유효하지 않은 나이입니다')
        return v

    @model_validator(mode='after')
    def passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError('비밀번호가 일치하지 않습니다')
        return self
```

### Depends를 사용한 검증

```python
from fastapi import Depends, HTTPException

async def verify_token(x_token: str = Header(...)):
    if x_token != "secret-token":
        raise HTTPException(status_code=400, detail="X-Token 헤더가 유효하지 않습니다")
    return x_token

async def verify_key(x_key: str = Header(...)):
    if x_key != "secret-key":
        raise HTTPException(status_code=400, detail="X-Key 헤더가 유효하지 않습니다")
    return x_key

@app.get("/items")
async def read_items(
    token: str = Depends(verify_token),
    key: str = Depends(verify_key)
):
    return {"token": token, "key": key}
```

---

## 5.10 에러 응답 포맷

검증 실패 시 반환되는 에러 응답을 커스터마이징합니다.

### 기본 Validation Error

FastAPI는 검증 실패 시 자동으로 422 Unprocessable Entity를 반환합니다.

```python
# 잘못된 요청:
# POST /items
# {"name": "A", "price": -10}

# 응답: 422
# {
#   "detail": [
#     {
#       "loc": ["body", "name"],
#       "msg": "ensure this value has at least 3 characters",
#       "type": "value_error.any_str.min_length"
#     },
#     {
#       "loc": ["body", "price"],
#       "msg": "ensure this value is greater than 0",
#       "type": "value_error.number.not_gt"
#     }
#   ]
# }
```

### 커스텀 Exception Handler

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": " -> ".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })

    return JSONResponse(
        status_code=422,
        content={
            "success": False,
            "message": "검증 실패",
            "errors": errors
        }
    )
```

---

## 5.11 실전 예제: 복합 검증

```python
from fastapi import FastAPI, Query, Path, Body, Header, Cookie
from pydantic import BaseModel, Field, EmailStr
from typing import List

app = FastAPI()

class Item(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0, le=1000000)
    tags: List[str] = Field(default_factory=list, max_items=10)

@app.put("/users/{user_id}/items/{item_id}")
async def update_item(
    # Path parameters
    user_id: int = Path(..., ge=1, title="사용자 ID"),
    item_id: int = Path(..., ge=1, title="아이템 ID"),

    # Query parameters
    notify: bool = Query(False, description="알림 전송 여부"),
    tags: List[str] = Query([], description="필터 태그"),

    # Request Body
    item: Item = Body(..., embed=True),

    # Headers
    x_token: str = Header(..., description="인증 토큰"),
    user_agent: str | None = Header(None, alias="User-Agent"),

    # Cookie
    session_id: str | None = Cookie(None)
):
    return {
        "user_id": user_id,
        "item_id": item_id,
        "notify": notify,
        "tags": tags,
        "item": item,
        "x_token": x_token,
        "user_agent": user_agent,
        "session_id": session_id
    }
```

---

## 5.12 핵심 요약

### 파라미터 검증 비교

| 기능 | Spring Boot | FastAPI |
|------|-------------|---------|
| **Path 검증** | `@PathVariable` + Bean Validation | `Path()` |
| **Query 검증** | `@RequestParam` + Bean Validation | `Query()` |
| **Body 검증** | `@RequestBody` + `@Valid` | Pydantic 모델 |
| **Header** | `@RequestHeader` | `Header()` |
| **Cookie** | `@CookieValue` | `Cookie()` |
| **Form** | `@ModelAttribute` | `Form()` |
| **File** | `MultipartFile` | `File()`, `UploadFile` |

### 기억할 핵심

1. ✅ **타입 + 제약조건**: `Query(0, ge=0, le=100)`
2. ✅ **자동 검증**: 타입 힌트만으로도 기본 검증
3. ✅ **명확한 에러**: 어떤 필드가 왜 실패했는지 명확
4. ✅ **여러 소스**: Path, Query, Body, Header, Cookie 모두 지원
5. ✅ **커스텀 검증**: `@field_validator`로 복잡한 로직 구현

---

## 5.13 실습 예제

### 실습 1: 상품 검색 API

**요구사항:**
- Path: category_id (1-100)
- Query: keyword (1-50자), min_price (0 이상), max_price (0 이상)
- Header: X-API-Key (필수)

**정답:**
```python
from fastapi import FastAPI, Path, Query, Header, HTTPException

app = FastAPI()

@app.get("/categories/{category_id}/products")
async def search_products(
    category_id: int = Path(..., ge=1, le=100),
    keyword: str = Query(..., min_length=1, max_length=50),
    min_price: float = Query(0, ge=0),
    max_price: float = Query(1000000, ge=0),
    x_api_key: str = Header(..., alias="X-API-Key")
):
    if min_price > max_price:
        raise HTTPException(
            status_code=400,
            detail="min_price는 max_price보다 작아야 합니다"
        )

    return {
        "category_id": category_id,
        "keyword": keyword,
        "price_range": [min_price, max_price]
    }
```

---

## 5.14 연습 문제

### 문제 1
다음 Spring Boot 코드를 FastAPI로 변환하세요:

```java
@PostMapping("/users/{userId}/posts")
public ResponseEntity<?> createPost(
    @PathVariable @Min(1) Long userId,
    @RequestParam(defaultValue = "false") Boolean publish,
    @RequestBody @Valid PostDto post,
    @RequestHeader("Authorization") String auth
) {
    // ...
}
```

### 문제 2
파일 업로드 API를 만드세요:
- 이미지만 허용 (jpg, png, gif)
- 최대 5MB
- 파일명 검증

---

## 5.15 다음 챕터 예고

다음 챕터에서는 **응답 모델과 상태 코드**를 다룹니다:
- Response 모델 정의
- 여러 응답 모델 (Union Types)
- 상태 코드 설정
- 파일 다운로드

---

[← 이전: Chapter 4. Pydantic 모델](chapter04-pydantic.md) | [목차로](../README.md) | [다음: Chapter 6. 응답 모델과 상태 코드 →](chapter06-response.md)
