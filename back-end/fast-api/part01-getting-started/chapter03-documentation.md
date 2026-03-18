# Chapter 3. 자동 문서화

## 3.1 FastAPI의 자동 문서화

FastAPI의 가장 강력한 기능 중 하나는 **코드에서 자동으로 API 문서를 생성**한다는 것입니다.

### Spring Boot와의 비교

```java
// Spring Boot - SpringDoc 의존성 추가 필요
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>

// 추가 설정
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("My API")
                .version("1.0.0")
                .description("API Documentation"));
    }
}
```

```python
# FastAPI - 설치만 하면 끝!
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    version="1.0.0",
    description="API Documentation"
)

# 자동으로 /docs, /redoc, /openapi.json 제공!
```

---

## 3.2 Swagger UI

### 접속 방법

```
http://localhost:8000/docs
```

### 제공되는 기능

1. **대화형 문서**: 브라우저에서 직접 API 테스트 가능
2. **자동 스키마**: Request/Response 모델 자동 표시
3. **인증 테스트**: Authorization 헤더 설정 가능
4. **타입 정보**: 모든 파라미터의 타입, 제약조건 표시

### 기본 예제

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    description: str | None = None

@app.get("/items/{item_id}")
def get_item(item_id: int, q: str | None = None):
    """
    특정 아이템을 조회합니다.

    - **item_id**: 아이템 ID (필수)
    - **q**: 검색 쿼리 (선택)
    """
    return {"item_id": item_id, "q": q}

@app.post("/items")
def create_item(item: Item):
    """
    새로운 아이템을 생성합니다.
    """
    return item
```

**Swagger UI에서 자동으로:**
- 경로 매개변수 `item_id`의 타입(int) 표시
- 쿼리 매개변수 `q`의 선택적 여부 표시
- Request Body의 JSON 스키마 자동 생성
- 함수 docstring이 설명으로 표시

---

## 3.3 ReDoc

Swagger UI의 대안으로 더 깔끔한 디자인의 문서입니다.

### 접속 방법

```
http://localhost:8000/redoc
```

### 특징

- 📖 읽기 전용 (테스트 불가)
- 🎨 더 깔끔한 디자인
- 📱 모바일 친화적
- 🔍 더 나은 검색 기능

---

## 3.4 OpenAPI 스키마

### OpenAPI JSON

```
http://localhost:8000/openapi.json
```

FastAPI는 자동으로 OpenAPI 3.0 스펙을 따르는 JSON 스키마를 생성합니다.

### 커스터마이징

```python
from fastapi import FastAPI

app = FastAPI(
    title="My Super Project",
    description="This is a very fancy project with auto docs",
    version="2.5.0",
    terms_of_service="http://example.com/terms/",
    contact={
        "name": "API Support",
        "url": "http://example.com/contact/",
        "email": "support@example.com",
    },
    license_info={
        "name": "Apache 2.0",
        "url": "https://www.apache.org/licenses/LICENSE-2.0.html",
    },
)
```

**Spring Boot와 비교:**
```java
// Spring Boot - 별도 설정 클래스 필요
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
        .info(new Info()
            .title("My Super Project")
            .version("2.5.0")
            .description("This is a very fancy project")
            .termsOfService("http://example.com/terms/")
            .contact(new Contact()
                .name("API Support")
                .email("support@example.com"))
            .license(new License()
                .name("Apache 2.0")
                .url("https://www.apache.org/licenses/LICENSE-2.0.html")));
}
```

---

## 3.5 태그를 사용한 그룹화

태그를 사용하여 관련된 엔드포인트를 그룹화할 수 있습니다.

### 기본 태그 사용

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/", tags=["users"])
def get_users():
    return [{"username": "john"}]

@app.post("/users/", tags=["users"])
def create_user(user: User):
    return user

@app.get("/items/", tags=["items"])
def get_items():
    return [{"name": "item1"}]

@app.post("/items/", tags=["items"])
def create_item(item: Item):
    return item
```

### 태그 메타데이터

```python
from fastapi import FastAPI

tags_metadata = [
    {
        "name": "users",
        "description": "사용자 관리 API. 회원가입, 로그인, 프로필 조회 등",
    },
    {
        "name": "items",
        "description": "아이템 관리 API. CRUD 작업 지원",
        "externalDocs": {
            "description": "외부 문서",
            "url": "https://example.com/items/",
        },
    },
]

app = FastAPI(openapi_tags=tags_metadata)

@app.get("/users/", tags=["users"])
def get_users():
    return [{"username": "john"}]

@app.get("/items/", tags=["items"])
def get_items():
    return [{"name": "item1"}]
```

### APIRouter와 태그

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
def get_users():
    return [{"username": "john"}]

@router.get("/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}
```

---

## 3.6 응답 예제 추가

Pydantic 모델에 예제 데이터를 추가할 수 있습니다.

### Config.schema_extra 사용

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

    class Config:
        schema_extra = {
            "example": {
                "name": "Laptop",
                "description": "A powerful laptop for developers",
                "price": 1299.99,
                "tax": 129.99
            }
        }
```

### Field의 example 사용

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(..., example="Laptop")
    description: str | None = Field(None, example="A powerful laptop")
    price: float = Field(..., example=1299.99)
    tax: float | None = Field(None, example=129.99)
```

### 여러 예제 제공

```python
from fastapi import FastAPI, Body
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.post("/items/")
def create_item(
    item: Item = Body(
        examples={
            "normal": {
                "summary": "일반 아이템",
                "description": "일반적인 아이템 예제",
                "value": {
                    "name": "Laptop",
                    "description": "A nice laptop",
                    "price": 1299.99,
                    "tax": 129.99,
                },
            },
            "cheap": {
                "summary": "저렴한 아이템",
                "value": {
                    "name": "Keyboard",
                    "price": 49.99,
                },
            },
            "invalid": {
                "summary": "잘못된 데이터",
                "value": {
                    "name": "Mouse",
                    "price": "not a number",  # 의도적으로 잘못된 타입
                },
            },
        },
    ),
):
    return item
```

---

## 3.7 응답 모델 문서화

### 다중 응답 모델

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

class Message(BaseModel):
    message: str

@app.get(
    "/items/{item_id}",
    response_model=Item,
    responses={
        200: {
            "description": "성공",
            "content": {
                "application/json": {
                    "example": {"name": "Laptop", "price": 1299.99}
                }
            },
        },
        404: {
            "description": "아이템을 찾을 수 없음",
            "model": Message,
        },
    },
)
def get_item(item_id: int):
    if item_id == 1:
        return {"name": "Laptop", "price": 1299.99}
    return {"message": "Item not found"}
```

### 상태 코드별 응답

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

class ErrorMessage(BaseModel):
    detail: str

@app.get(
    "/items/{item_id}",
    response_model=Item,
    responses={
        404: {"model": ErrorMessage, "description": "아이템이 존재하지 않음"},
        500: {"model": ErrorMessage, "description": "서버 에러"},
    },
)
def get_item(item_id: int):
    if item_id != 1:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"name": "Laptop", "price": 1299.99}
```

---

## 3.8 엔드포인트 문서화 상세 설정

### summary, description, deprecated

```python
from fastapi import FastAPI

app = FastAPI()

@app.get(
    "/items/",
    summary="아이템 목록 조회",
    description="모든 아이템의 목록을 반환합니다. 페이지네이션을 지원합니다.",
    response_description="아이템 목록",
)
def get_items():
    return [{"name": "item1"}, {"name": "item2"}]

@app.get(
    "/old-items/",
    summary="구버전 아이템 조회",
    deprecated=True,
)
def get_old_items():
    """
    이 엔드포인트는 더 이상 사용되지 않습니다.
    대신 /items/ 를 사용하세요.
    """
    return [{"name": "old-item"}]
```

### Docstring으로 문서화

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/items/")
def create_item(item: Item):
    """
    아이템을 생성합니다.

    다음 정보가 필요합니다:
    - **name**: 아이템 이름 (필수)
    - **price**: 가격 (필수, 양수)
    - **description**: 설명 (선택)
    - **tax**: 세금 (선택)

    반환값:
    - 생성된 아이템 정보
    """
    return item
```

**Markdown 지원:**
```python
@app.get("/items/")
def get_items():
    """
    ## 아이템 목록 조회

    이 엔드포인트는 모든 아이템을 반환합니다.

    ### 기능
    - 페이지네이션
    - 정렬
    - 필터링

    ### 예제
    ```
    GET /items/?skip=0&limit=10
    ```

    ### 주의사항
    > 최대 100개까지만 반환됩니다.
    """
    return []
```

---

## 3.9 문서 URL 커스터마이징

### 기본 URL 변경

```python
from fastapi import FastAPI

app = FastAPI(
    docs_url="/api/docs",      # 기본: /docs
    redoc_url="/api/redoc",    # 기본: /redoc
    openapi_url="/api/openapi.json"  # 기본: /openapi.json
)
```

### 문서 비활성화 (프로덕션)

```python
from fastapi import FastAPI

app = FastAPI(
    docs_url=None,    # Swagger UI 비활성화
    redoc_url=None,   # ReDoc 비활성화
)

# 또는 환경에 따라 조건부
import os

docs_enabled = os.getenv("ENVIRONMENT") != "production"

app = FastAPI(
    docs_url="/docs" if docs_enabled else None,
    redoc_url="/redoc" if docs_enabled else None,
)
```

---

## 3.10 Spring Boot SpringDoc과 완전 비교

### 설정 비교

| 기능 | Spring Boot (SpringDoc) | FastAPI |
|------|------------------------|---------|
| **설치** | Maven/Gradle 의존성 추가 | pip install만으로 완료 |
| **기본 설정** | 별도 Config 클래스 필요 | FastAPI() 인스턴스 생성만 |
| **문서 URL** | /swagger-ui.html | /docs (커스터마이징 가능) |
| **API 설명** | @Operation, @ApiResponse | 함수 docstring 또는 파라미터 |
| **모델 예제** | @Schema(example=...) | Config.schema_extra 또는 Field |
| **태그** | @Tag | tags 파라미터 |
| **인증** | @SecurityRequirement | dependencies 파라미터 |
| **숨기기** | @Hidden | include_in_schema=False |

### 예제 비교

#### Spring Boot
```java
@Tag(name = "users", description = "사용자 관리 API")
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Operation(
        summary = "사용자 조회",
        description = "ID로 사용자를 조회합니다"
    )
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "성공"),
        @ApiResponse(responseCode = "404", description = "사용자 없음")
    })
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}
```

#### FastAPI
```python
@app.get(
    "/api/users/{id}",
    tags=["users"],
    summary="사용자 조회",
    responses={
        200: {"description": "성공"},
        404: {"description": "사용자 없음"}
    }
)
def get_user(id: int):
    """ID로 사용자를 조회합니다"""
    return {"id": id, "name": "John"}
```

**FastAPI가 훨씬 간결합니다!**

---

## 3.11 실전 팁

### 1. 프로덕션 환경에서 문서 숨기기

```python
import os
from fastapi import FastAPI

def create_app():
    if os.getenv("ENVIRONMENT") == "production":
        return FastAPI(docs_url=None, redoc_url=None)
    else:
        return FastAPI(
            title="My API (Development)",
            version="1.0.0"
        )

app = create_app()
```

### 2. 버전별 문서

```python
from fastapi import FastAPI

app_v1 = FastAPI(
    title="My API v1",
    version="1.0.0",
    docs_url="/api/v1/docs",
    openapi_url="/api/v1/openapi.json"
)

app_v2 = FastAPI(
    title="My API v2",
    version="2.0.0",
    docs_url="/api/v2/docs",
    openapi_url="/api/v2/openapi.json"
)
```

### 3. 공통 응답 정의

```python
from fastapi import FastAPI

common_responses = {
    404: {"description": "리소스를 찾을 수 없음"},
    500: {"description": "서버 에러"},
}

app = FastAPI()

@app.get("/items/{item_id}", responses=common_responses)
def get_item(item_id: int):
    return {"item_id": item_id}

@app.get("/users/{user_id}", responses=common_responses)
def get_user(user_id: int):
    return {"user_id": user_id}
```

---

## 3.12 핵심 요약

### FastAPI 자동 문서화의 장점

1. ✅ **제로 설정**: 설치만 하면 자동으로 문서 생성
2. ✅ **실시간 테스트**: Swagger UI에서 직접 API 테스트 가능
3. ✅ **타입 기반**: 타입 힌트만으로 스키마 자동 생성
4. ✅ **OpenAPI 3.0**: 표준 스펙 준수
5. ✅ **커스터마이징**: 필요시 상세 설정 가능

### Spring Boot vs FastAPI

| 측면 | Spring Boot | FastAPI |
|------|-------------|---------|
| **설정 복잡도** | 중간~높음 | 매우 낮음 |
| **코드량** | 많음 | 적음 |
| **학습 곡선** | 높음 | 낮음 |
| **유연성** | 높음 | 높음 |
| **UI** | Swagger UI | Swagger UI + ReDoc |

---

## 3.13 실습 예제

### 실습 1: 완전한 API 문서 만들기

**요구사항:**
- 블로그 API 문서화
- 태그 사용 (posts, users, comments)
- 응답 예제 추가
- 에러 응답 정의

**정답:**
```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

tags_metadata = [
    {"name": "posts", "description": "게시글 관리"},
    {"name": "users", "description": "사용자 관리"},
    {"name": "comments", "description": "댓글 관리"},
]

app = FastAPI(
    title="Blog API",
    version="1.0.0",
    description="블로그 관리 API",
    openapi_tags=tags_metadata
)

class Post(BaseModel):
    title: str = Field(..., example="FastAPI 시작하기")
    content: str = Field(..., example="FastAPI는 빠르고 쉬운...")
    author: str = Field(..., example="John Doe")

    class Config:
        schema_extra = {
            "example": {
                "title": "FastAPI 완전 정복",
                "content": "FastAPI는 Python 웹 프레임워크...",
                "author": "Jane Smith"
            }
        }

common_responses = {
    404: {"description": "리소스를 찾을 수 없음"},
    500: {"description": "서버 에러"},
}

@app.get(
    "/posts",
    tags=["posts"],
    summary="게시글 목록 조회",
    responses=common_responses
)
def get_posts():
    """
    모든 게시글을 조회합니다.

    - **페이지네이션** 지원
    - **정렬** 지원
    """
    return []

@app.post(
    "/posts",
    tags=["posts"],
    summary="게시글 생성",
    status_code=201,
    responses=common_responses
)
def create_post(post: Post):
    """
    새로운 게시글을 생성합니다.
    """
    return post
```

---

## 3.14 연습 문제

### 문제 1
FastAPI의 자동 문서화가 Spring Boot SpringDoc보다 우수한 점 5가지를 나열하세요.

### 문제 2
다음 요구사항을 만족하는 API를 만들고 문서화하세요:
- 상품 CRUD API
- 각 엔드포인트마다 상세한 설명
- 요청/응답 예제 3개씩
- 에러 응답 정의

### 문제 3
프로덕션 환경에서 API 문서를 조건부로 비활성화하는 방법을 구현하세요.

---

## 3.15 다음 챕터 예고

다음 챕터에서는 **Pydantic 모델**을 깊이 있게 다룹니다:
- Pydantic 모델 정의
- 데이터 검증 규칙
- 중첩 모델
- Config 클래스
- Field 설정

**Spring Boot의 DTO + Bean Validation에 대응하는 Pydantic의 모든 것을 배웁니다!**

---

[← 이전: Chapter 2. 라우팅과 HTTP 메서드](chapter02-routing.md) | [목차로](../README.md) | [다음: Part 02 →](../part02-request-response/chapter04-pydantic.md)
