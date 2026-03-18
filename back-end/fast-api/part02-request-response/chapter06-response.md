# Chapter 6. 응답 모델과 상태 코드

## 6.1 Response 모델 기초

FastAPI는 `response_model` 파라미터를 사용하여 응답 데이터의 구조를 정의하고 자동으로 직렬화합니다.

### Spring Boot와 비교

```java
// Spring Boot
public class UserResponseDto {
    private Long id;
    private String username;
    private String email;
    // password는 포함하지 않음
}

@PostMapping("/users")
public ResponseEntity<UserResponseDto> createUser(@RequestBody UserDto user) {
    User savedUser = userService.save(user);
    UserResponseDto response = modelMapper.map(savedUser, UserResponseDto.class);
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

```python
# FastAPI - 훨씬 간결
from pydantic import BaseModel

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    # password는 자동으로 제외됨!

@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate):
    # DB 저장 로직...
    return {
        "id": 1,
        "username": user.username,
        "email": user.email,
        "password": user.password  # 이 필드는 응답에 포함되지 않음!
    }
```

---

## 6.2 response_model 사용

### 기본 사용법

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    description: str | None = None

class ItemResponse(BaseModel):
    name: str
    price: float

@app.post("/items", response_model=ItemResponse)
def create_item(item: Item):
    # description은 응답에서 자동으로 제외됨
    return item
```

### 보안을 위한 필드 제외

```python
from pydantic import BaseModel, EmailStr

class UserIn(BaseModel):
    username: str
    email: EmailStr
    password: str
    full_name: str | None = None

class UserOut(BaseModel):
    username: str
    email: EmailStr
    full_name: str | None = None

@app.post("/users", response_model=UserOut)
def create_user(user: UserIn):
    # password는 UserOut에 없으므로 응답에 포함되지 않음
    return user
```

---

## 6.3 response_model_exclude

특정 필드를 응답에서 제외합니다.

### 기본 사용

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    username: str
    email: str
    password: str
    is_active: bool

@app.get("/users/{user_id}", response_model=User, response_model_exclude={"password"})
def get_user(user_id: int):
    return {
        "id": user_id,
        "username": "john",
        "email": "john@example.com",
        "password": "hashed_password",
        "is_active": True
    }
# 응답: password 필드 제외
```

### 여러 필드 제외

```python
@app.get(
    "/users/{user_id}",
    response_model=User,
    response_model_exclude={"password", "email"}
)
def get_user(user_id: int):
    return user_data
```

---

## 6.4 response_model_include

특정 필드만 응답에 포함합니다.

```python
@app.get(
    "/users/{user_id}",
    response_model=User,
    response_model_include={"id", "username"}
)
def get_user(user_id: int):
    return {
        "id": 1,
        "username": "john",
        "email": "john@example.com",
        "password": "secret"
    }
# 응답: {"id": 1, "username": "john"}만 포함
```

---

## 6.5 response_model_exclude_unset

설정되지 않은 기본값을 제외합니다.

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5
    tags: list = []

@app.post("/items", response_model=Item, response_model_exclude_unset=True)
def create_item(item: Item):
    return item

# 요청:
# {"name": "Laptop", "price": 1299.99}

# response_model_exclude_unset=False (기본값):
# {"name": "Laptop", "description": null, "price": 1299.99, "tax": 10.5, "tags": []}

# response_model_exclude_unset=True:
# {"name": "Laptop", "price": 1299.99}
```

### response_model_exclude_defaults / response_model_exclude_none

```python
# response_model_exclude_defaults=True
# 기본값과 같은 값은 제외

# response_model_exclude_none=True
# None 값은 제외

@app.get("/items/{item_id}", response_model=Item, response_model_exclude_none=True)
def get_item(item_id: int):
    return {"name": "Laptop", "price": 1299.99, "description": None}
# 응답: {"name": "Laptop", "price": 1299.99}
```

---

## 6.6 상태 코드 설정

HTTP 상태 코드를 명시적으로 설정합니다.

### 기본 상태 코드

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/items", status_code=status.HTTP_201_CREATED)
def create_item(item: Item):
    return item

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int):
    # 204는 응답 body 없음
    return None

@app.get("/items/{item_id}", status_code=status.HTTP_200_OK)
def get_item(item_id: int):
    return {"item_id": item_id}
```

### 자주 사용하는 상태 코드

```python
from fastapi import status

# 성공
status.HTTP_200_OK                  # 200
status.HTTP_201_CREATED            # 201
status.HTTP_202_ACCEPTED           # 202
status.HTTP_204_NO_CONTENT         # 204

# 리다이렉션
status.HTTP_301_MOVED_PERMANENTLY  # 301
status.HTTP_302_FOUND              # 302

# 클라이언트 에러
status.HTTP_400_BAD_REQUEST        # 400
status.HTTP_401_UNAUTHORIZED       # 401
status.HTTP_403_FORBIDDEN          # 403
status.HTTP_404_NOT_FOUND          # 404
status.HTTP_422_UNPROCESSABLE_ENTITY  # 422

# 서버 에러
status.HTTP_500_INTERNAL_SERVER_ERROR  # 500
status.HTTP_503_SERVICE_UNAVAILABLE    # 503
```

### Spring Boot와 비교

```java
// Spring Boot
@PostMapping("/items")
public ResponseEntity<Item> createItem(@RequestBody Item item) {
    return ResponseEntity.status(HttpStatus.CREATED).body(item);
}

@DeleteMapping("/items/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void deleteItem(@PathVariable Long id) {
    itemService.delete(id);
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

## 6.7 여러 응답 모델 (Union Types)

상황에 따라 다른 응답 모델을 반환할 수 있습니다.

### Union 사용

```python
from typing import Union
from pydantic import BaseModel

class UserBase(BaseModel):
    username: str
    email: str

class UserAdmin(UserBase):
    role: str = "admin"
    permissions: list[str]

class UserRegular(UserBase):
    role: str = "user"

@app.get("/users/{user_id}", response_model=Union[UserAdmin, UserRegular])
def get_user(user_id: int, is_admin: bool = False):
    if is_admin:
        return UserAdmin(
            username="admin",
            email="admin@example.com",
            permissions=["read", "write", "delete"]
        )
    return UserRegular(username="user", email="user@example.com")
```

### 리스트 응답

```python
from typing import List

@app.get("/items", response_model=List[Item])
def list_items():
    return [
        {"name": "Item 1", "price": 10.0},
        {"name": "Item 2", "price": 20.0}
    ]
```

---

## 6.8 Response 직접 사용

더 세밀한 제어가 필요한 경우 `Response` 객체를 직접 사용합니다.

### JSONResponse

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/items/{item_id}")
def get_item(item_id: int):
    content = {"item_id": item_id, "name": "Item"}
    return JSONResponse(
        content=content,
        status_code=200,
        headers={"X-Custom-Header": "value"}
    )
```

### 커스텀 헤더 추가

```python
from fastapi import Response

@app.get("/items/{item_id}")
def get_item(item_id: int, response: Response):
    response.headers["X-Custom-Header"] = "custom-value"
    response.status_code = 200
    return {"item_id": item_id}
```

---

## 6.9 파일 다운로드

파일을 응답으로 반환합니다.

### FileResponse

```python
from fastapi import FastAPI
from fastapi.responses import FileResponse

app = FastAPI()

@app.get("/download/{filename}")
def download_file(filename: str):
    file_path = f"/path/to/files/{filename}"
    return FileResponse(
        path=file_path,
        filename=filename,
        media_type="application/octet-stream"
    )
```

### StreamingResponse

대용량 파일이나 스트리밍이 필요한 경우:

```python
from fastapi.responses import StreamingResponse
import io

@app.get("/download/large-file")
def download_large_file():
    def generate():
        for i in range(1000000):
            yield f"Line {i}\n"

    return StreamingResponse(
        generate(),
        media_type="text/plain",
        headers={"Content-Disposition": "attachment; filename=large-file.txt"}
    )
```

### 이미지 응답

```python
from fastapi.responses import Response

@app.get("/image")
def get_image():
    with open("image.png", "rb") as f:
        image_data = f.read()

    return Response(
        content=image_data,
        media_type="image/png"
    )
```

---

## 6.10 RedirectResponse

다른 URL로 리다이렉트합니다.

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/old-path")
def old_path():
    return RedirectResponse(url="/new-path", status_code=301)

@app.get("/new-path")
def new_path():
    return {"message": "This is the new path"}
```

### 조건부 리다이렉트

```python
@app.get("/items/{item_id}")
def get_item(item_id: int):
    if item_id == 0:
        return RedirectResponse(url="/")
    return {"item_id": item_id}
```

---

## 6.11 HTMLResponse

HTML을 직접 반환합니다.

```python
from fastapi.responses import HTMLResponse

@app.get("/", response_class=HTMLResponse)
def read_root():
    html_content = """
    <!DOCTYPE html>
    <html>
        <head>
            <title>FastAPI</title>
        </head>
        <body>
            <h1>Hello, FastAPI!</h1>
        </body>
    </html>
    """
    return html_content
```

---

## 6.12 실전 예제: 완전한 CRUD API

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import List

app = FastAPI()

# 임시 DB
items_db = {}
item_id_counter = 1

class ItemCreate(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

class ItemUpdate(BaseModel):
    name: str | None = None
    description: str | None = None
    price: float | None = None
    tax: float | None = None

class ItemResponse(BaseModel):
    id: int
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

# CREATE
@app.post("/items", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
def create_item(item: ItemCreate):
    global item_id_counter
    item_data = item.dict()
    item_data["id"] = item_id_counter
    items_db[item_id_counter] = item_data
    item_id_counter += 1
    return item_data

# READ ALL
@app.get("/items", response_model=List[ItemResponse])
def list_items(skip: int = 0, limit: int = 10):
    items = list(items_db.values())
    return items[skip : skip + limit]

# READ ONE
@app.get("/items/{item_id}", response_model=ItemResponse)
def get_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Item {item_id} not found"
        )
    return items_db[item_id]

# UPDATE
@app.put("/items/{item_id}", response_model=ItemResponse)
def update_item(item_id: int, item: ItemUpdate):
    if item_id not in items_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Item {item_id} not found"
        )

    stored_item = items_db[item_id]
    update_data = item.dict(exclude_unset=True)
    items_db[item_id] = {**stored_item, **update_data}

    return items_db[item_id]

# DELETE
@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Item {item_id} not found"
        )

    del items_db[item_id]
    return None
```

---

## 6.13 핵심 요약

### Response 관련 기능

| 기능 | 설명 | 사용 예 |
|------|------|---------|
| **response_model** | 응답 구조 정의 | `response_model=UserOut` |
| **response_model_exclude** | 특정 필드 제외 | `response_model_exclude={"password"}` |
| **response_model_include** | 특정 필드만 포함 | `response_model_include={"id", "name"}` |
| **response_model_exclude_unset** | 미설정 필드 제외 | `response_model_exclude_unset=True` |
| **status_code** | HTTP 상태 코드 | `status_code=201` |

### 응답 타입

| 타입 | 용도 |
|------|------|
| **JSONResponse** | JSON 응답 (기본) |
| **FileResponse** | 파일 다운로드 |
| **StreamingResponse** | 스트리밍 응답 |
| **RedirectResponse** | 리다이렉트 |
| **HTMLResponse** | HTML 응답 |
| **PlainTextResponse** | 일반 텍스트 |

---

## 6.14 실습 예제

### 실습 1: 사용자 API

**요구사항:**
- POST /users: 회원가입 (비밀번호는 응답에서 제외)
- GET /users/{id}: 사용자 조회 (이메일은 본인만 볼 수 있음)
- 201, 200 상태 코드 사용

**정답:**
```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserPublic(BaseModel):
    id: int
    username: str

class UserPrivate(UserPublic):
    email: EmailStr

@app.post("/users", response_model=UserPublic, status_code=201)
def create_user(user: UserCreate):
    return {
        "id": 1,
        "username": user.username,
        "email": user.email,
        "password": user.password  # 자동 제외됨
    }

@app.get("/users/{user_id}")
def get_user(user_id: int, is_owner: bool = False):
    user_data = {
        "id": user_id,
        "username": "john",
        "email": "john@example.com"
    }

    if is_owner:
        return UserPrivate(**user_data)
    return UserPublic(**user_data)
```

---

## 6.15 연습 문제

### 문제 1
`response_model_exclude_unset`와 `response_model_exclude_none`의 차이점을 설명하세요.

### 문제 2
파일 다운로드 API를 작성하세요:
- 파일명을 받아서 다운로드
- Content-Disposition 헤더 설정
- 파일이 없으면 404 반환

### 문제 3
상품 목록 API를 작성하세요:
- 관리자: 모든 필드 반환
- 일반 사용자: 민감한 필드 제외

---

## 6.16 다음 챕터 예고

다음 챕터에서는 **의존성 주입 (Dependency Injection)**을 다룹니다:
- `Depends` 사용법
- Spring의 `@Autowired`와 비교
- 함수형 vs 클래스형 의존성
- 의존성 체인

**FastAPI의 강력한 DI 시스템을 마스터합니다!**

---

[← 이전: Chapter 5. 요청 데이터 검증](chapter05-validation.md) | [목차로](../README.md) | [다음: Part 03 →](../part03-dependency-injection/chapter07-di-basics.md)
