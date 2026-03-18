# Chapter 2. 라우팅과 HTTP 메서드

## 2.1 라우팅 기초

라우팅은 클라이언트 요청을 적절한 핸들러 함수로 연결하는 것입니다. FastAPI는 데코레이터를 사용하여 라우팅을 정의합니다.

### Spring Boot vs FastAPI 라우팅

```java
// Spring Boot
@RestController
@RequestMapping("/api")
public class ProductController {

    @GetMapping("/products")
    public List<Product> getProducts() { }

    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) { }

    @PostMapping("/products")
    public Product createProduct(@RequestBody Product product) { }
}
```

```python
# FastAPI
from fastapi import FastAPI

app = FastAPI()

@app.get("/api/products")
def get_products():
    pass

@app.get("/api/products/{id}")
def get_product(id: int):
    pass

@app.post("/api/products")
def create_product(product: Product):
    pass
```

---

## 2.2 HTTP 메서드

FastAPI는 모든 표준 HTTP 메서드를 지원합니다.

### 메서드 데코레이터

| HTTP 메서드 | Spring Boot | FastAPI | 용도 |
|------------|-------------|---------|------|
| GET | `@GetMapping` | `@app.get()` | 조회 |
| POST | `@PostMapping` | `@app.post()` | 생성 |
| PUT | `@PutMapping` | `@app.put()` | 전체 수정 |
| PATCH | `@PatchMapping` | `@app.patch()` | 부분 수정 |
| DELETE | `@DeleteMapping` | `@app.delete()` | 삭제 |
| OPTIONS | - | `@app.options()` | CORS |
| HEAD | - | `@app.head()` | 헤더만 조회 |

### 예제: CRUD API

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

app = FastAPI()

# 임시 데이터베이스
products_db = {}
product_id_counter = 1

class Product(BaseModel):
    name: str
    price: float
    description: str | None = None

class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    description: str | None = None

# CREATE
@app.post("/products", response_model=ProductResponse, status_code=201)
def create_product(product: Product):
    global product_id_counter
    product_id = product_id_counter
    product_id_counter += 1

    product_data = product.dict()
    product_data["id"] = product_id
    products_db[product_id] = product_data

    return product_data

# READ (전체 조회)
@app.get("/products", response_model=List[ProductResponse])
def get_products():
    return list(products_db.values())

# READ (단일 조회)
@app.get("/products/{product_id}", response_model=ProductResponse)
def get_product(product_id: int):
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="Product not found")
    return products_db[product_id]

# UPDATE (전체)
@app.put("/products/{product_id}", response_model=ProductResponse)
def update_product(product_id: int, product: Product):
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="Product not found")

    product_data = product.dict()
    product_data["id"] = product_id
    products_db[product_id] = product_data

    return product_data

# UPDATE (부분)
@app.patch("/products/{product_id}", response_model=ProductResponse)
def partial_update_product(product_id: int, product: dict):
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="Product not found")

    stored_product = products_db[product_id]
    stored_product.update(product)

    return stored_product

# DELETE
@app.delete("/products/{product_id}", status_code=204)
def delete_product(product_id: int):
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="Product not found")

    del products_db[product_id]
    return None
```

---

## 2.3 경로 매개변수 (Path Parameters)

URL 경로에 포함된 동적 값을 캡처합니다.

### Spring Boot와 비교

```java
// Spring Boot
@GetMapping("/users/{userId}/posts/{postId}")
public Post getPost(
    @PathVariable Long userId,
    @PathVariable Long postId
) {
    return postService.findPost(userId, postId);
}
```

```python
# FastAPI
@app.get("/users/{user_id}/posts/{post_id}")
def get_post(user_id: int, post_id: int):
    return {"user_id": user_id, "post_id": post_id}
```

### 타입 검증

FastAPI는 타입 힌트를 통해 자동으로 검증합니다.

```python
# 올바른 요청: /items/42
# 잘못된 요청: /items/foo (자동으로 422 에러 반환)
@app.get("/items/{item_id}")
def get_item(item_id: int):
    return {"item_id": item_id}
```

### Enum을 사용한 경로 제한

```python
from enum import Enum

class ModelName(str, Enum):
    ALEXNET = "alexnet"
    RESNET = "resnet"
    LENET = "lenet"

@app.get("/models/{model_name}")
def get_model(model_name: ModelName):
    if model_name == ModelName.ALEXNET:
        return {"model": "alexnet", "message": "Deep Learning FTW!"}

    if model_name.value == "lenet":
        return {"model": "lenet", "message": "LeCNN all the images"}

    return {"model": model_name, "message": "Have some residuals"}

# 허용: /models/alexnet, /models/resnet, /models/lenet
# 거부: /models/vgg (422 에러)
```

### 경로 포함 매개변수

```python
# 파일 경로 같은 경우
@app.get("/files/{file_path:path}")
def read_file(file_path: str):
    return {"file_path": file_path}

# 요청 예: /files/home/user/documents/test.txt
# file_path = "home/user/documents/test.txt"
```

---

## 2.4 쿼리 매개변수 (Query Parameters)

URL의 `?` 뒤에 오는 key-value 쌍입니다.

### Spring Boot와 비교

```java
// Spring Boot
@GetMapping("/items")
public List<Item> getItems(
    @RequestParam(required = false, defaultValue = "0") int skip,
    @RequestParam(required = false, defaultValue = "10") int limit,
    @RequestParam(required = false) String search
) {
    return itemService.findItems(skip, limit, search);
}
```

```python
# FastAPI
@app.get("/items")
def get_items(skip: int = 0, limit: int = 10, search: str | None = None):
    return {"skip": skip, "limit": limit, "search": search}

# 요청 예:
# /items                           -> skip=0, limit=10, search=None
# /items?skip=20                   -> skip=20, limit=10, search=None
# /items?skip=20&limit=5           -> skip=20, limit=5, search=None
# /items?search=phone              -> skip=0, limit=10, search="phone"
```

### 필수 쿼리 매개변수

```python
# 기본값이 없으면 필수
@app.get("/search")
def search(q: str):  # q는 필수
    return {"query": q}

# 요청: /search?q=fastapi  ✅
# 요청: /search            ❌ 422 에러
```

### 선택적 쿼리 매개변수

```python
from typing import Optional

# Python 3.10+
@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None, short: bool = False):
    item = {"item_id": item_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update({"description": "This is an amazing item"})
    return item

# Python 3.9 이하
@app.get("/items/{item_id}")
def read_item(item_id: int, q: Optional[str] = None, short: bool = False):
    # 동일한 로직
    pass
```

### 여러 값을 가진 쿼리 매개변수

```python
from typing import List

@app.get("/items")
def read_items(tags: List[str] | None = None):
    return {"tags": tags}

# 요청: /items?tags=electronics&tags=computers&tags=laptop
# 결과: {"tags": ["electronics", "computers", "laptop"]}
```

---

## 2.5 Request Body

POST, PUT, PATCH 요청에서 데이터를 받을 때 사용합니다.

### Pydantic 모델 정의

```python
from pydantic import BaseModel, Field
from typing import Optional

class Item(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    description: str | None = Field(None, max_length=500)
    price: float = Field(..., gt=0)
    tax: float | None = None
    tags: List[str] = []

    class Config:
        schema_extra = {
            "example": {
                "name": "Laptop",
                "description": "A powerful laptop",
                "price": 1299.99,
                "tax": 129.99,
                "tags": ["electronics", "computers"]
            }
        }
```

### Spring Boot DTO와 비교

```java
// Spring Boot
public class ItemDto {
    @NotBlank
    @Size(min = 1, max = 100)
    private String name;

    @Size(max = 500)
    private String description;

    @Positive
    private Double price;

    private Double tax;

    // Getters, Setters
}

@PostMapping("/items")
public ResponseEntity<Item> createItem(@Valid @RequestBody ItemDto itemDto) {
    // ...
}
```

```python
# FastAPI - 훨씬 간결!
@app.post("/items")
def create_item(item: Item):
    # Pydantic이 자동으로 검증
    return item
```

### Request Body + Path + Query

```python
@app.put("/items/{item_id}")
def update_item(
    item_id: int,              # Path parameter
    item: Item,                # Request body
    q: str | None = None       # Query parameter
):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result.update({"q": q})
    return result

# PUT /items/42?q=test
# Body: {"name": "Laptop", "price": 1299.99}
```

---

## 2.6 Response 모델

응답 데이터의 구조를 정의하고 자동으로 직렬화합니다.

### response_model 사용

```python
from pydantic import BaseModel

class UserIn(BaseModel):
    username: str
    email: str
    password: str
    full_name: str | None = None

class UserOut(BaseModel):
    username: str
    email: str
    full_name: str | None = None
    # password는 응답에 포함되지 않음!

@app.post("/users", response_model=UserOut)
def create_user(user: UserIn):
    # 실제로는 DB에 저장
    return user  # FastAPI가 자동으로 UserOut 형식으로 변환
```

**Spring Boot와 비교:**
```java
// Spring Boot - 별도의 DTO 필요
@PostMapping("/users")
public UserResponseDto createUser(@RequestBody UserRequestDto user) {
    User savedUser = userService.save(user);
    return modelMapper.map(savedUser, UserResponseDto.class);
}
```

### response_model_exclude

```python
@app.get("/users/{user_id}", response_model=User, response_model_exclude={"password"})
def get_user(user_id: int):
    return users_db[user_id]
```

### response_model_include

```python
@app.get("/users/{user_id}", response_model=User, response_model_include={"username", "email"})
def get_user(user_id: int):
    return users_db[user_id]
```

---

## 2.7 상태 코드

HTTP 상태 코드를 명시적으로 설정할 수 있습니다.

### 기본 상태 코드

```python
from fastapi import status

@app.post("/items", status_code=status.HTTP_201_CREATED)
def create_item(item: Item):
    return item

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int):
    return None

@app.get("/items/{item_id}", status_code=status.HTTP_200_OK)
def get_item(item_id: int):
    return {"item_id": item_id}
```

### Spring Boot와 비교

```java
// Spring Boot
@PostMapping("/items")
public ResponseEntity<Item> createItem(@RequestBody Item item) {
    return ResponseEntity.status(HttpStatus.CREATED).body(item);
}

@DeleteMapping("/items/{id}")
public ResponseEntity<Void> deleteItem(@PathVariable Long id) {
    itemService.delete(id);
    return ResponseEntity.noContent().build();
}
```

```python
# FastAPI - 더 간결
@app.post("/items", status_code=201)
def create_item(item: Item):
    return item

@app.delete("/items/{item_id}", status_code=204)
def delete_item(item_id: int):
    return None
```

---

## 2.8 라우터 분리 (APIRouter)

대규모 애플리케이션에서는 라우터를 분리하는 것이 좋습니다.

### Spring Boot의 Controller와 유사

**routers/users.py**
```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["users"]
)

@router.get("/")
def get_users():
    return [{"username": "john"}, {"username": "jane"}]

@router.get("/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}

@router.post("/")
def create_user(user: User):
    return user
```

**routers/items.py**
```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/items",
    tags=["items"]
)

@router.get("/")
def get_items():
    return [{"name": "item1"}, {"name": "item2"}]
```

**main.py**
```python
from fastapi import FastAPI
from routers import users, items

app = FastAPI()

app.include_router(users.router)
app.include_router(items.router)

# /users/ 와 /items/ 엔드포인트가 자동으로 등록됨
```

### 프로젝트 구조

```
app/
├── main.py
├── routers/
│   ├── __init__.py
│   ├── users.py
│   ├── items.py
│   └── products.py
└── models/
    ├── __init__.py
    ├── user.py
    └── item.py
```

---

## 2.9 핵심 요약

### Spring Boot vs FastAPI 라우팅 비교

| 기능 | Spring Boot | FastAPI |
|------|-------------|---------|
| **GET 요청** | `@GetMapping` | `@app.get()` |
| **POST 요청** | `@PostMapping` | `@app.post()` |
| **경로 변수** | `@PathVariable` | 함수 매개변수 |
| **쿼리 파라미터** | `@RequestParam` | 함수 매개변수 (기본값 지정) |
| **Request Body** | `@RequestBody` + DTO | Pydantic 모델 |
| **검증** | `@Valid` + Bean Validation | Pydantic 자동 검증 |
| **상태 코드** | `ResponseEntity.status()` | `status_code` 파라미터 |
| **라우터 분리** | `@RestController` 클래스 | `APIRouter` |

### 기억할 핵심 사항

1. ✅ **타입 힌트가 곧 검증**: 타입을 명시하면 자동으로 검증
2. ✅ **Pydantic 모델**: DTO + Validation이 통합됨
3. ✅ **간결한 코드**: Spring Boot보다 훨씬 적은 코드
4. ✅ **자동 문서화**: 모든 라우트가 Swagger UI에 자동 등록
5. ✅ **함수형 스타일**: 클래스 없이 함수만으로 가능

---

## 2.10 실습 예제

### 실습 1: 블로그 Post API

**요구사항:**
- GET `/posts` - 전체 게시글 조회
- GET `/posts/{post_id}` - 특정 게시글 조회
- POST `/posts` - 게시글 생성
- PUT `/posts/{post_id}` - 게시글 수정
- DELETE `/posts/{post_id}` - 게시글 삭제

**모델:**
```python
class Post(BaseModel):
    title: str
    content: str
    author: str
    published: bool = False
```

**정답:**
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

app = FastAPI()

posts_db = {}
post_id_counter = 1

class Post(BaseModel):
    title: str
    content: str
    author: str
    published: bool = False

class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    author: str
    published: bool

@app.get("/posts", response_model=List[PostResponse])
def get_posts():
    return list(posts_db.values())

@app.get("/posts/{post_id}", response_model=PostResponse)
def get_post(post_id: int):
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="Post not found")
    return posts_db[post_id]

@app.post("/posts", response_model=PostResponse, status_code=201)
def create_post(post: Post):
    global post_id_counter
    post_data = post.dict()
    post_data["id"] = post_id_counter
    posts_db[post_id_counter] = post_data
    post_id_counter += 1
    return post_data

@app.put("/posts/{post_id}", response_model=PostResponse)
def update_post(post_id: int, post: Post):
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="Post not found")
    post_data = post.dict()
    post_data["id"] = post_id
    posts_db[post_id] = post_data
    return post_data

@app.delete("/posts/{post_id}", status_code=204)
def delete_post(post_id: int):
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="Post not found")
    del posts_db[post_id]
```

### 실습 2: 검색 및 페이지네이션

**요구사항:**
- GET `/posts?skip=0&limit=10&search=fastapi&published=true`
- 페이지네이션, 검색, 필터링 구현

**정답:**
```python
@app.get("/posts", response_model=List[PostResponse])
def get_posts(
    skip: int = 0,
    limit: int = 10,
    search: str | None = None,
    published: bool | None = None
):
    posts = list(posts_db.values())

    # 필터링
    if search:
        posts = [p for p in posts if search.lower() in p["title"].lower()]

    if published is not None:
        posts = [p for p in posts if p["published"] == published]

    # 페이지네이션
    return posts[skip : skip + limit]
```

---

## 2.11 연습 문제

### 문제 1
FastAPI에서 경로 매개변수와 쿼리 매개변수의 차이점을 설명하고, 각각 언제 사용해야 하는지 설명하세요.

### 문제 2
다음 Spring Boot 코드를 FastAPI로 변환하세요:

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping
    public List<Book> getBooks(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(required = false) String category
    ) {
        return bookService.findBooks(page, size, category);
    }

    @PostMapping
    public ResponseEntity<Book> createBook(@Valid @RequestBody BookDto bookDto) {
        Book book = bookService.save(bookDto);
        return ResponseEntity.status(HttpStatus.CREATED).body(book);
    }
}
```

### 문제 3
Pydantic의 `response_model`, `response_model_exclude`, `response_model_include`의 차이점과 사용 사례를 설명하세요.

---

## 2.12 다음 챕터 예고

다음 챕터에서는 **자동 문서화**를 다룹니다:
- Swagger UI 커스터마이징
- ReDoc
- OpenAPI 스키마
- 태그, 메타데이터
- 예제 데이터 추가

**FastAPI의 강력한 자동 문서화 기능을 마스터합니다!**

---

[← 이전: Chapter 1. FastAPI 소개](chapter01-introduction.md) | [목차로](../README.md) | [다음: Chapter 3. 자동 문서화 →](chapter03-documentation.md)