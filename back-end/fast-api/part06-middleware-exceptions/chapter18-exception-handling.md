# Chapter 18: 예외 처리 (Exception Handling)

## 학습 목표
- FastAPI의 예외 처리 메커니즘 이해
- HTTPException 사용법 학습
- 커스텀 예외 핸들러 작성
- 전역 예외 처리 구현
- Spring Boot의 @ExceptionHandler와 비교

## 1. FastAPI 예외 처리 개요

### 1.1 예외 처리 흐름

```
Client Request
    ↓
Middleware (before)
    ↓
Route Handler → Exception 발생
    ↓
Exception Handler
    ↓
Middleware (after)
    ↓
Client Response (Error)
```

### 1.2 Spring Boot와 비교

| 구분 | Spring Boot | FastAPI |
|------|-------------|---------|
| **기본 예외** | `ResponseStatusException` | `HTTPException` |
| **예외 핸들러** | `@ExceptionHandler` | `@app.exception_handler()` |
| **전역 핸들러** | `@ControllerAdvice` | `@app.exception_handler(Exception)` |
| **유효성 검증** | `@Valid` + `MethodArgumentNotValidException` | Pydantic + `RequestValidationError` |
| **응답 형식** | `ResponseEntity<ErrorResponse>` | `JSONResponse` |
| **커스텀 예외** | Custom Exception 클래스 | Custom Exception 클래스 |

## 2. HTTPException 기본 사용

### 2.1 HTTPException 사용법

FastAPI의 기본 예외 클래스입니다.

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

users_db = {
    1: {"name": "Alice", "email": "alice@example.com"},
    2: {"name": "Bob", "email": "bob@example.com"},
}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(
            status_code=404,
            detail="User not found"
        )
    return users_db[user_id]

@app.post("/users")
async def create_user(name: str, email: str):
    if not email or "@" not in email:
        raise HTTPException(
            status_code=400,
            detail="Invalid email address"
        )

    new_id = max(users_db.keys()) + 1
    users_db[new_id] = {"name": name, "email": email}
    return {"id": new_id, "name": name, "email": email}
```

**응답 예시:**
```json
{
  "detail": "User not found"
}
```

**Spring Boot 비교:**

```java
@RestController
@RequestMapping("/users")
public class UserController {
    @GetMapping("/{userId}")
    public UserDTO getUser(@PathVariable Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND, "User not found"
            ));
        return UserDTO.from(user);
    }
}
```

### 2.2 추가 정보 포함

`detail`에 더 많은 정보를 포함할 수 있습니다.

```python
from fastapi import FastAPI, HTTPException
from datetime import datetime

app = FastAPI()

@app.delete("/users/{user_id}")
async def delete_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(
            status_code=404,
            detail={
                "error": "User not found",
                "user_id": user_id,
                "timestamp": datetime.utcnow().isoformat(),
                "suggestion": "Check the user ID and try again"
            }
        )

    del users_db[user_id]
    return {"message": "User deleted"}
```

**응답:**
```json
{
  "detail": {
    "error": "User not found",
    "user_id": 999,
    "timestamp": "2024-01-15T10:30:00",
    "suggestion": "Check the user ID and try again"
  }
}
```

### 2.3 커스텀 헤더 추가

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/admin")
async def admin_only():
    raise HTTPException(
        status_code=401,
        detail="Authentication required",
        headers={"WWW-Authenticate": "Bearer"},
    )
```

## 3. 커스텀 예외 클래스

### 3.1 기본 커스텀 예외

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# 커스텀 예외 정의
class UserNotFoundException(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id

class DuplicateEmailException(Exception):
    def __init__(self, email: str):
        self.email = email

class InsufficientPermissionException(Exception):
    def __init__(self, required_role: str):
        self.required_role = required_role

# 예외 핸들러 등록
@app.exception_handler(UserNotFoundException)
async def user_not_found_handler(request: Request, exc: UserNotFoundException):
    return JSONResponse(
        status_code=404,
        content={
            "error": "User not found",
            "user_id": exc.user_id,
            "path": request.url.path
        }
    )

@app.exception_handler(DuplicateEmailException)
async def duplicate_email_handler(request: Request, exc: DuplicateEmailException):
    return JSONResponse(
        status_code=409,
        content={
            "error": "Email already exists",
            "email": exc.email,
        }
    )

@app.exception_handler(InsufficientPermissionException)
async def insufficient_permission_handler(request: Request, exc: InsufficientPermissionException):
    return JSONResponse(
        status_code=403,
        content={
            "error": "Insufficient permissions",
            "required_role": exc.required_role,
        }
    )

# 사용 예시
users_db = {1: {"name": "Alice", "email": "alice@example.com"}}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in users_db:
        raise UserNotFoundException(user_id)
    return users_db[user_id]

@app.post("/users")
async def create_user(name: str, email: str):
    # 이메일 중복 체크
    if any(user["email"] == email for user in users_db.values()):
        raise DuplicateEmailException(email)

    new_id = max(users_db.keys()) + 1
    users_db[new_id] = {"name": name, "email": email}
    return {"id": new_id}

@app.delete("/users/{user_id}")
async def delete_user(user_id: int, current_user_role: str = "user"):
    if current_user_role != "admin":
        raise InsufficientPermissionException("admin")

    if user_id not in users_db:
        raise UserNotFoundException(user_id)

    del users_db[user_id]
    return {"message": "User deleted"}
```

**Spring Boot 비교:**

```java
// 커스텀 예외
public class UserNotFoundException extends RuntimeException {
    private final Long userId;

    public UserNotFoundException(Long userId) {
        super("User not found: " + userId);
        this.userId = userId;
    }

    public Long getUserId() {
        return userId;
    }
}

// 전역 예외 핸들러
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex,
            HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            "User not found",
            ex.getUserId(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(DuplicateEmailException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateEmail(
            DuplicateEmailException ex) {
        ErrorResponse error = new ErrorResponse(
            "Email already exists",
            ex.getEmail()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }
}
```

### 3.2 예외 기본 클래스 상속

더 구조화된 예외 처리를 위해 기본 클래스를 만들 수 있습니다.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from typing import Optional

app = FastAPI()

# 기본 예외 클래스
class APIException(Exception):
    """모든 API 예외의 기본 클래스"""
    def __init__(
        self,
        status_code: int,
        error: str,
        message: str,
        details: Optional[dict] = None
    ):
        self.status_code = status_code
        self.error = error
        self.message = message
        self.details = details or {}

# 구체적인 예외들
class NotFoundException(APIException):
    def __init__(self, resource: str, resource_id: any):
        super().__init__(
            status_code=404,
            error="NOT_FOUND",
            message=f"{resource} not found",
            details={"resource": resource, "id": resource_id}
        )

class ValidationException(APIException):
    def __init__(self, message: str, field: str = None):
        details = {"field": field} if field else {}
        super().__init__(
            status_code=400,
            error="VALIDATION_ERROR",
            message=message,
            details=details
        )

class AuthorizationException(APIException):
    def __init__(self, message: str = "Insufficient permissions"):
        super().__init__(
            status_code=403,
            error="FORBIDDEN",
            message=message,
        )

class ConflictException(APIException):
    def __init__(self, message: str, conflicting_field: str = None):
        details = {"field": conflicting_field} if conflicting_field else {}
        super().__init__(
            status_code=409,
            error="CONFLICT",
            message=message,
            details=details
        )

# 통합 예외 핸들러
@app.exception_handler(APIException)
async def api_exception_handler(request: Request, exc: APIException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error,
            "message": exc.message,
            "details": exc.details,
            "path": request.url.path,
        }
    )

# 사용 예시
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in users_db:
        raise NotFoundException("User", user_id)
    return users_db[user_id]

@app.post("/users")
async def create_user(name: str, email: str):
    if not email or "@" not in email:
        raise ValidationException("Invalid email format", field="email")

    if any(user["email"] == email for user in users_db.values()):
        raise ConflictException(
            "Email already registered",
            conflicting_field="email"
        )

    new_id = max(users_db.keys()) + 1
    users_db[new_id] = {"name": name, "email": email}
    return {"id": new_id}
```

**응답 예시:**
```json
{
  "error": "NOT_FOUND",
  "message": "User not found",
  "details": {
    "resource": "User",
    "id": 999
  },
  "path": "/users/999"
}
```

## 4. 전역 예외 처리

### 4.1 모든 예외 캐치

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import logging
import traceback

logger = logging.getLogger(__name__)
app = FastAPI()

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # 예외 로깅
    logger.error(
        f"Unhandled exception: {exc}",
        exc_info=True,
        extra={
            "method": request.method,
            "path": request.url.path,
            "client": request.client.host,
        }
    )

    # 스택 트레이스 (개발 환경에서만)
    stack_trace = traceback.format_exc()
    logger.debug(f"Stack trace: {stack_trace}")

    return JSONResponse(
        status_code=500,
        content={
            "error": "INTERNAL_SERVER_ERROR",
            "message": "An unexpected error occurred",
            "path": request.url.path,
        }
    )

@app.get("/error")
async def trigger_error():
    # 의도적인 에러
    result = 10 / 0  # ZeroDivisionError
    return {"result": result}
```

### 4.2 환경별 예외 응답

개발 환경과 프로덕션 환경에서 다른 응답을 제공합니다.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import os
import traceback
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

@app.exception_handler(Exception)
async def environment_aware_exception_handler(request: Request, exc: Exception):
    logger.error(f"Exception: {exc}", exc_info=True)

    error_response = {
        "error": "INTERNAL_SERVER_ERROR",
        "message": "An unexpected error occurred",
        "path": request.url.path,
    }

    # 개발 환경에서는 상세 정보 포함
    if ENVIRONMENT == "development":
        error_response.update({
            "exception_type": type(exc).__name__,
            "exception_message": str(exc),
            "stack_trace": traceback.format_exc().split("\n"),
        })

    return JSONResponse(
        status_code=500,
        content=error_response
    )
```

**개발 환경 응답:**
```json
{
  "error": "INTERNAL_SERVER_ERROR",
  "message": "An unexpected error occurred",
  "path": "/error",
  "exception_type": "ZeroDivisionError",
  "exception_message": "division by zero",
  "stack_trace": [
    "Traceback (most recent call last):",
    "  File ...",
    "ZeroDivisionError: division by zero"
  ]
}
```

**프로덕션 환경 응답:**
```json
{
  "error": "INTERNAL_SERVER_ERROR",
  "message": "An unexpected error occurred",
  "path": "/error"
}
```

## 5. Pydantic 유효성 검증 에러 처리

### 5.1 기본 유효성 검증 에러

Pydantic 모델 유효성 검증 실패 시 `RequestValidationError`가 발생합니다.

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel, EmailStr, Field

app = FastAPI()

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)

@app.post("/users")
async def create_user(user: UserCreate):
    return {"username": user.username}

# 기본 에러 응답은 자동으로 처리됨
# POST /users {"username": "ab", "email": "invalid", "age": 200}
# → 422 Unprocessable Entity
```

**기본 응답:**
```json
{
  "detail": [
    {
      "loc": ["body", "username"],
      "msg": "ensure this value has at least 3 characters",
      "type": "value_error.any_str.min_length",
      "ctx": {"limit_value": 3}
    },
    {
      "loc": ["body", "email"],
      "msg": "value is not a valid email address",
      "type": "value_error.email"
    },
    {
      "loc": ["body", "age"],
      "msg": "ensure this value is less than or equal to 150",
      "type": "value_error.number.not_le",
      "ctx": {"limit_value": 150}
    }
  ]
}
```

### 5.2 커스텀 유효성 검증 에러 핸들러

더 읽기 쉬운 형식으로 변환합니다.

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel, EmailStr, Field

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    # 에러를 더 읽기 쉬운 형식으로 변환
    errors = []
    for error in exc.errors():
        error_detail = {
            "field": ".".join(str(loc) for loc in error["loc"][1:]),  # 'body' 제외
            "message": error["msg"],
            "type": error["type"],
        }

        # 추가 컨텍스트 정보
        if "ctx" in error:
            error_detail["context"] = error["ctx"]

        errors.append(error_detail)

    return JSONResponse(
        status_code=422,
        content={
            "error": "VALIDATION_ERROR",
            "message": "Request validation failed",
            "errors": errors,
            "path": request.url.path,
        }
    )

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)

@app.post("/users")
async def create_user(user: UserCreate):
    return {"username": user.username}
```

**커스텀 응답:**
```json
{
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "errors": [
    {
      "field": "username",
      "message": "ensure this value has at least 3 characters",
      "type": "value_error.any_str.min_length",
      "context": {"limit_value": 3}
    },
    {
      "field": "email",
      "message": "value is not a valid email address",
      "type": "value_error.email"
    },
    {
      "field": "age",
      "message": "ensure this value is less than or equal to 150",
      "type": "value_error.number.not_le",
      "context": {"limit_value": 150}
    }
  ],
  "path": "/users"
}
```

**Spring Boot의 @Valid 에러 처리와 비교:**

```java
@ControllerAdvice
public class ValidationExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new FieldError(
                error.getField(),
                error.getDefaultMessage()
            ))
            .collect(Collectors.toList());

        ErrorResponse response = new ErrorResponse(
            "VALIDATION_ERROR",
            "Request validation failed",
            errors
        );

        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(response);
    }
}
```

### 5.3 한국어 메시지로 변환

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

app = FastAPI()

# 에러 메시지 번역 매핑
ERROR_MESSAGES = {
    "value_error.any_str.min_length": "{field}는 최소 {limit_value}자 이상이어야 합니다",
    "value_error.any_str.max_length": "{field}는 최대 {limit_value}자 이하여야 합니다",
    "value_error.email": "{field}는 유효한 이메일 주소여야 합니다",
    "value_error.number.not_ge": "{field}는 {limit_value} 이상이어야 합니다",
    "value_error.number.not_le": "{field}는 {limit_value} 이하여야 합니다",
    "value_error.missing": "{field}는 필수 항목입니다",
}

# 필드명 한글 매핑
FIELD_NAMES = {
    "username": "사용자명",
    "email": "이메일",
    "age": "나이",
    "password": "비밀번호",
}

@app.exception_handler(RequestValidationError)
async def korean_validation_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        field = error["loc"][-1]
        field_name = FIELD_NAMES.get(field, field)
        error_type = error["type"]

        # 메시지 템플릿 가져오기
        message_template = ERROR_MESSAGES.get(
            error_type,
            "{field}의 값이 올바르지 않습니다"
        )

        # 컨텍스트 정보 포함
        context = error.get("ctx", {})
        context["field"] = field_name

        # 메시지 포맷팅
        message = message_template.format(**context)

        errors.append({
            "field": field,
            "message": message,
        })

    return JSONResponse(
        status_code=422,
        content={
            "error": "유효성 검증 실패",
            "message": "입력 데이터를 확인해주세요",
            "errors": errors,
        }
    )
```

**한국어 응답:**
```json
{
  "error": "유효성 검증 실패",
  "message": "입력 데이터를 확인해주세요",
  "errors": [
    {
      "field": "username",
      "message": "사용자명은 최소 3자 이상이어야 합니다"
    },
    {
      "field": "email",
      "message": "이메일은 유효한 이메일 주소여야 합니다"
    },
    {
      "field": "age",
      "message": "나이는 150 이하여야 합니다"
    }
  ]
}
```

## 6. 표준 에러 응답 형식

### 6.1 일관된 에러 응답 구조

모든 에러에 대해 일관된 형식을 사용합니다.

```python
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel
from datetime import datetime
from typing import Optional, List, Any
import uuid

app = FastAPI()

# 에러 응답 모델
class ErrorDetail(BaseModel):
    field: Optional[str] = None
    message: str
    code: Optional[str] = None

class StandardErrorResponse(BaseModel):
    error_id: str  # 고유 에러 ID
    timestamp: str
    status_code: int
    error: str
    message: str
    path: str
    details: Optional[List[ErrorDetail]] = None

def create_error_response(
    request: Request,
    status_code: int,
    error: str,
    message: str,
    details: Optional[List[ErrorDetail]] = None
) -> JSONResponse:
    """표준 에러 응답 생성"""
    error_id = str(uuid.uuid4())

    error_response = StandardErrorResponse(
        error_id=error_id,
        timestamp=datetime.utcnow().isoformat(),
        status_code=status_code,
        error=error,
        message=message,
        path=request.url.path,
        details=details,
    )

    return JSONResponse(
        status_code=status_code,
        content=error_response.dict(exclude_none=True)
    )

# RequestValidationError 핸들러
@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    details = [
        ErrorDetail(
            field=".".join(str(loc) for loc in error["loc"][1:]),
            message=error["msg"],
            code=error["type"]
        )
        for error in exc.errors()
    ]

    return create_error_response(
        request=request,
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        error="VALIDATION_ERROR",
        message="Request validation failed",
        details=details
    )

# 전역 예외 핸들러
@app.exception_handler(Exception)
async def global_error_handler(request: Request, exc: Exception):
    return create_error_response(
        request=request,
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        error="INTERNAL_SERVER_ERROR",
        message="An unexpected error occurred"
    )

# HTTPException 핸들러
from fastapi import HTTPException

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return create_error_response(
        request=request,
        status_code=exc.status_code,
        error=exc.detail if isinstance(exc.detail, str) else "HTTP_EXCEPTION",
        message=exc.detail if isinstance(exc.detail, str) else str(exc.detail)
    )
```

**표준 에러 응답:**
```json
{
  "error_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "status_code": 422,
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "path": "/users",
  "details": [
    {
      "field": "email",
      "message": "value is not a valid email address",
      "code": "value_error.email"
    }
  ]
}
```

### 6.2 에러 로깅

에러가 발생할 때 자동으로 로깅합니다.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import logging
import json

logger = logging.getLogger(__name__)
app = FastAPI()

@app.exception_handler(Exception)
async def logging_exception_handler(request: Request, exc: Exception):
    error_id = str(uuid.uuid4())

    # 구조화된 로그
    log_data = {
        "error_id": error_id,
        "error_type": type(exc).__name__,
        "error_message": str(exc),
        "method": request.method,
        "path": request.url.path,
        "query_params": dict(request.query_params),
        "client_ip": request.client.host,
    }

    logger.error(f"Exception: {json.dumps(log_data)}", exc_info=True)

    return JSONResponse(
        status_code=500,
        content={
            "error_id": error_id,
            "error": "INTERNAL_SERVER_ERROR",
            "message": "An unexpected error occurred",
            "support_message": f"Please contact support with error ID: {error_id}"
        }
    )
```

## 7. 실전 예제: 완전한 예외 처리 시스템

모든 개념을 통합한 실전 예제입니다.

```python
from fastapi import FastAPI, Request, status, Depends
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List
from datetime import datetime
import logging
import uuid
import os

# 로깅 설정
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Complete Exception Handling Example")

ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

# ===== 1. 에러 응답 모델 =====
class ErrorDetail(BaseModel):
    field: Optional[str] = None
    message: str
    code: Optional[str] = None

class ErrorResponse(BaseModel):
    error_id: str
    timestamp: str
    status_code: int
    error: str
    message: str
    path: str
    details: Optional[List[ErrorDetail]] = None

# ===== 2. 커스텀 예외 클래스 =====
class BaseAPIException(Exception):
    """모든 API 예외의 기본 클래스"""
    def __init__(
        self,
        status_code: int,
        error: str,
        message: str,
        details: Optional[List[ErrorDetail]] = None
    ):
        self.status_code = status_code
        self.error = error
        self.message = message
        self.details = details or []

class NotFoundException(BaseAPIException):
    def __init__(self, resource: str, resource_id: any):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            error="NOT_FOUND",
            message=f"{resource} not found",
            details=[ErrorDetail(message=f"ID: {resource_id}")]
        )

class DuplicateException(BaseAPIException):
    def __init__(self, resource: str, field: str, value: any):
        super().__init__(
            status_code=status.HTTP_409_CONFLICT,
            error="DUPLICATE",
            message=f"{resource} already exists",
            details=[ErrorDetail(field=field, message=f"Value '{value}' already exists")]
        )

class AuthorizationException(BaseAPIException):
    def __init__(self, message: str = "Insufficient permissions"):
        super().__init__(
            status_code=status.HTTP_403_FORBIDDEN,
            error="FORBIDDEN",
            message=message
        )

# ===== 3. 에러 응답 생성 함수 =====
def create_error_response(
    request: Request,
    status_code: int,
    error: str,
    message: str,
    details: Optional[List[ErrorDetail]] = None,
    exc: Optional[Exception] = None
) -> JSONResponse:
    """표준 에러 응답 생성 및 로깅"""
    error_id = str(uuid.uuid4())

    # 로그 기록
    log_data = {
        "error_id": error_id,
        "status_code": status_code,
        "error": error,
        "message": message,
        "path": request.url.path,
        "method": request.method,
        "client_ip": request.client.host,
    }

    if exc:
        log_data["exception_type"] = type(exc).__name__
        log_data["exception_message"] = str(exc)

    logger.error(f"Error: {log_data}", exc_info=exc is not None)

    # 응답 생성
    error_response = ErrorResponse(
        error_id=error_id,
        timestamp=datetime.utcnow().isoformat(),
        status_code=status_code,
        error=error,
        message=message,
        path=request.url.path,
        details=details
    )

    response_content = error_response.dict(exclude_none=True)

    # 개발 환경에서는 스택 트레이스 포함
    if ENVIRONMENT == "development" and exc:
        import traceback
        response_content["stack_trace"] = traceback.format_exc().split("\n")

    return JSONResponse(
        status_code=status_code,
        content=response_content
    )

# ===== 4. 예외 핸들러 등록 =====

# 커스텀 API 예외
@app.exception_handler(BaseAPIException)
async def api_exception_handler(request: Request, exc: BaseAPIException):
    return create_error_response(
        request=request,
        status_code=exc.status_code,
        error=exc.error,
        message=exc.message,
        details=exc.details,
        exc=exc
    )

# 유효성 검증 예외
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    details = [
        ErrorDetail(
            field=".".join(str(loc) for loc in error["loc"][1:]),
            message=error["msg"],
            code=error["type"]
        )
        for error in exc.errors()
    ]

    return create_error_response(
        request=request,
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        error="VALIDATION_ERROR",
        message="Request validation failed",
        details=details,
        exc=exc
    )

# 전역 예외 핸들러
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return create_error_response(
        request=request,
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        error="INTERNAL_SERVER_ERROR",
        message="An unexpected error occurred",
        exc=exc
    )

# ===== 5. 비즈니스 로직 =====

# Pydantic 모델
class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)

# 데이터베이스 (메모리)
users_db = {
    1: {"username": "alice", "email": "alice@example.com", "age": 30}
}

# 엔드포인트
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """사용자 조회"""
    if user_id not in users_db:
        raise NotFoundException("User", user_id)
    return users_db[user_id]

@app.post("/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    """사용자 생성"""
    # 이메일 중복 체크
    if any(u["email"] == user.email for u in users_db.values()):
        raise DuplicateException("User", "email", user.email)

    # 사용자명 중복 체크
    if any(u["username"] == user.username for u in users_db.values()):
        raise DuplicateException("User", "username", user.username)

    # 생성
    new_id = max(users_db.keys()) + 1
    user_data = user.dict()
    users_db[new_id] = user_data

    return {"id": new_id, **user_data}

@app.delete("/users/{user_id}")
async def delete_user(user_id: int, is_admin: bool = False):
    """사용자 삭제 (관리자만)"""
    if not is_admin:
        raise AuthorizationException("Admin access required")

    if user_id not in users_db:
        raise NotFoundException("User", user_id)

    del users_db[user_id]
    return {"message": "User deleted"}

@app.get("/error")
async def trigger_error():
    """의도적 에러 (테스트용)"""
    result = 10 / 0  # ZeroDivisionError
    return {"result": result}

@app.get("/")
async def root():
    return {"message": "Exception Handling Example API"}
```

## 8. 모범 사례

### 8.1 예외 처리 원칙

1. **명확한 에러 메시지**: 사용자가 무엇을 잘못했는지 명확히 전달
2. **일관된 응답 형식**: 모든 에러에 동일한 구조 사용
3. **적절한 HTTP 상태 코드**: 에러 종류에 맞는 코드 사용
4. **로깅**: 모든 에러를 로깅하여 추적 가능하게
5. **보안**: 민감한 정보(스택 트레이스, 내부 경로 등)는 프로덕션에서 숨김

### 8.2 HTTP 상태 코드 가이드

| 코드 | 이름 | 사용 시점 |
|------|------|-----------|
| 400 | Bad Request | 잘못된 요청 형식, 파라미터 오류 |
| 401 | Unauthorized | 인증 필요 (로그인 안됨) |
| 403 | Forbidden | 권한 없음 (로그인했지만 접근 불가) |
| 404 | Not Found | 리소스를 찾을 수 없음 |
| 409 | Conflict | 리소스 충돌 (중복 등록 등) |
| 422 | Unprocessable Entity | 유효성 검증 실패 |
| 500 | Internal Server Error | 서버 내부 오류 |
| 503 | Service Unavailable | 서비스 일시적 중단 |

### 8.3 에러 코드 네이밍

```python
# ✅ 좋은 예: 명확하고 구체적
"USER_NOT_FOUND"
"DUPLICATE_EMAIL"
"INVALID_PASSWORD_FORMAT"
"INSUFFICIENT_PERMISSIONS"

# ❌ 나쁜 예: 모호함
"ERROR"
"FAILED"
"BAD_REQUEST"
```

## 9. 연습 문제

### 9.1 기본 연습

1. 404, 409, 403 상태 코드를 반환하는 커스텀 예외를 각각 만들고, 예외 핸들러를 등록하세요.

2. 사용자가 없는 리소스에 접근할 때 "Resource not found" 메시지를 반환하는 예외를 작성하세요.

3. Pydantic 유효성 검증 에러를 한글로 변환하는 핸들러를 작성하세요.

### 9.2 중급 연습

4. 에러가 발생할 때마다 고유한 에러 ID를 생성하고, 로그에 기록한 뒤 응답에 포함시키세요.

5. 개발 환경에서는 스택 트레이스를 포함하고, 프로덕션에서는 제외하는 예외 핸들러를 작성하세요.

6. Rate Limit 초과 시 429 상태 코드와 함께 재시도 가능 시간을 알려주는 예외를 구현하세요.

### 9.3 고급 연습

7. 모든 에러를 데이터베이스에 기록하는 예외 핸들러를 작성하세요 (비동기로).

8. 특정 예외가 자주 발생하면 자동으로 알림(이메일/Slack)을 보내는 시스템을 구현하세요.

9. Spring Boot의 `@ControllerAdvice`와 동일한 기능을 하는 모듈화된 예외 처리 시스템을 설계하고 구현하세요.

## 10. 요약

### 핵심 포인트

1. **HTTPException**: FastAPI의 기본 예외 클래스
   - `status_code`, `detail`, `headers` 파라미터

2. **커스텀 예외**: 도메인 특화 예외 정의
   - `@app.exception_handler()` 데코레이터로 핸들러 등록

3. **전역 예외 처리**: 모든 예외를 캐치
   - `@app.exception_handler(Exception)`

4. **유효성 검증 에러**: Pydantic 모델 검증 실패
   - `RequestValidationError` 핸들러로 커스터마이징

5. **표준 에러 응답**: 일관된 형식 유지
   - 에러 ID, 타임스탬프, 상태 코드, 메시지, 상세 정보

6. **환경별 처리**: 개발/프로덕션 환경에 따라 다른 응답
   - 개발: 스택 트레이스 포함
   - 프로덕션: 최소 정보만

7. **Spring Boot 비교**
   - `@ExceptionHandler` ≈ `@app.exception_handler()`
   - `@ControllerAdvice` ≈ 전역 예외 핸들러
   - `@Valid` + `MethodArgumentNotValidException` ≈ Pydantic + `RequestValidationError`

### 다음 챕터 예고

**Chapter 19: 백그라운드 작업 (Background Tasks)**

다음 챕터에서는 백그라운드 작업 처리를 다룹니다:
- BackgroundTasks 사용법
- 이메일 발송, 파일 처리 등 비동기 작업
- Celery와의 통합
- 작업 큐와 워커 패턴
- Spring Boot의 @Async와 비교

예외 처리와 백그라운드 작업을 함께 활용하면 안정적이고 확장 가능한 API를 만들 수 있습니다!