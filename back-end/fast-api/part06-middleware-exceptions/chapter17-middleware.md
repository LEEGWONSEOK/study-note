# Chapter 17: 미들웨어 (Middleware)

## 학습 목표
- FastAPI 미들웨어의 개념과 동작 방식 이해
- 커스텀 미들웨어 작성 방법 학습
- 요청/응답 로깅, 성능 측정 구현
- Spring Boot의 Filter/Interceptor와 비교

## 1. 미들웨어란?

### 1.1 개념

미들웨어는 요청과 응답 사이에서 실행되는 함수입니다. 모든 요청을 처리하기 전후에 공통 로직을 수행할 수 있습니다.

```
Client → Middleware → Route Handler → Middleware → Client
         (before)                      (after)
```

### 1.2 Spring Boot와 비교

| 구분 | Spring Boot | FastAPI |
|------|-------------|---------|
| **필터** | Filter (javax.servlet.Filter) | Middleware |
| **인터셉터** | HandlerInterceptor | Middleware + Dependencies |
| **순서 제어** | @Order, FilterRegistrationBean | app.add_middleware() 순서 |
| **예외 처리** | @ControllerAdvice | Exception Handler + Middleware |
| **비동기 지원** | WebFilter (Reactive) | async/await 기본 지원 |

**Spring Boot Filter 예시:**
```java
@Component
@Order(1)
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        long startTime = System.currentTimeMillis();
        chain.doFilter(request, response);
        long duration = System.currentTimeMillis() - startTime;
        System.out.println("Request took " + duration + "ms");
    }
}
```

**FastAPI Middleware 예시:**
```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time
    print(f"Request took {duration:.3f}s")
    return response
```

## 2. 미들웨어 작성 방법

### 2.1 데코레이터 방식

가장 간단한 방식으로, `@app.middleware("http")` 데코레이터를 사용합니다.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.middleware("http")
async def custom_middleware(request: Request, call_next):
    # 요청 처리 전 (before)
    print(f"Incoming request: {request.method} {request.url.path}")

    # 다음 미들웨어 또는 라우트 핸들러 호출
    response = await call_next(request)

    # 응답 처리 후 (after)
    print(f"Response status: {response.status_code}")

    return response
```

**중요 포인트:**
- `call_next(request)`: 다음 미들웨어 또는 라우트 핸들러 실행
- 반드시 `response`를 반환해야 함
- 예외 처리를 추가하여 안정성 확보

### 2.2 클래스 기반 미들웨어

더 복잡한 로직이나 설정이 필요한 경우 ASGI 미들웨어 클래스를 작성합니다.

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
import time

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()

        # 요청 처리
        response = await call_next(request)

        # 처리 시간 계산
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = str(process_time)

        return response

# 미들웨어 등록
app.add_middleware(TimingMiddleware)
```

**Spring Boot HandlerInterceptor와 비교:**

```java
@Component
public class TimingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request,
                          HttpServletResponse response,
                          Object handler,
                          ModelAndView modelAndView) {
        long startTime = (Long) request.getAttribute("startTime");
        long endTime = System.currentTimeMillis();
        response.setHeader("X-Process-Time",
                          String.valueOf(endTime - startTime));
    }
}
```

### 2.3 미들웨어 실행 순서

미들웨어는 등록한 **역순**으로 요청을 처리하고, **정순**으로 응답을 처리합니다.

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def first_middleware(request: Request, call_next):
    print("1. First middleware - before")
    response = await call_next(request)
    print("6. First middleware - after")
    return response

@app.middleware("http")
async def second_middleware(request: Request, call_next):
    print("2. Second middleware - before")
    response = await call_next(request)
    print("5. Second middleware - after")
    return response

@app.get("/test")
async def test_endpoint():
    print("3. Route handler")
    return {"message": "Test"}

# 실행 순서:
# 1. First middleware - before
# 2. Second middleware - before
# 3. Route handler
# 4. Second middleware - after (5번 출력)
# 5. First middleware - after (6번 출력)
```

## 3. 요청/응답 로깅 미들웨어

### 3.1 기본 로깅 미들웨어

```python
from fastapi import FastAPI, Request
import logging
import time
from datetime import datetime

# 로거 설정
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)

app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    # 요청 정보 로깅
    start_time = time.time()

    logger.info(f"Request: {request.method} {request.url.path}")
    logger.info(f"Client: {request.client.host}")
    logger.info(f"Headers: {dict(request.headers)}")

    # 요청 처리
    response = await call_next(request)

    # 응답 정보 로깅
    duration = time.time() - start_time
    logger.info(f"Response: {response.status_code} ({duration:.3f}s)")

    return response
```

### 3.2 구조화된 로깅

JSON 형식으로 로그를 구조화하여 분석을 용이하게 합니다.

```python
from fastapi import FastAPI, Request
import logging
import json
import time
import uuid

app = FastAPI()

@app.middleware("http")
async def structured_logging(request: Request, call_next):
    # 요청 ID 생성
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id

    start_time = time.time()

    # 요청 로그
    request_log = {
        "request_id": request_id,
        "timestamp": datetime.utcnow().isoformat(),
        "method": request.method,
        "path": request.url.path,
        "query_params": dict(request.query_params),
        "client_ip": request.client.host,
        "user_agent": request.headers.get("user-agent", ""),
    }

    logger.info(f"REQUEST: {json.dumps(request_log)}")

    # 요청 처리
    response = await call_next(request)

    # 응답 로그
    duration = time.time() - start_time
    response_log = {
        "request_id": request_id,
        "status_code": response.status_code,
        "duration_seconds": round(duration, 3),
        "response_size": response.headers.get("content-length", 0),
    }

    logger.info(f"RESPONSE: {json.dumps(response_log)}")

    # 응답 헤더에 request_id 추가
    response.headers["X-Request-ID"] = request_id

    return response
```

### 3.3 요청 본문 로깅

POST/PUT 요청의 본문을 로깅하려면 주의가 필요합니다. 본문을 한 번 읽으면 다시 읽을 수 없기 때문입니다.

```python
from fastapi import FastAPI, Request
import logging

app = FastAPI()

@app.middleware("http")
async def log_request_body(request: Request, call_next):
    # GET 요청은 본문이 없으므로 스킵
    if request.method in ["POST", "PUT", "PATCH"]:
        # 본문 읽기
        body = await request.body()

        # 본문 로깅 (민감한 정보 주의!)
        logger.info(f"Request body: {body.decode('utf-8')[:500]}")  # 처음 500자만

        # 중요: 본문을 다시 읽을 수 있도록 설정
        async def receive():
            return {"type": "http.request", "body": body}

        request._receive = receive

    response = await call_next(request)
    return response
```

**주의사항:**
- 민감한 정보(비밀번호, 토큰 등)는 로깅하지 않기
- 큰 파일 업로드는 본문 로깅 제외
- 프로덕션 환경에서는 레벨 조정 (INFO → DEBUG)

**Spring Boot에서의 요청 본문 로깅:**

```java
@Component
public class RequestBodyLoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) {
        // ContentCachingRequestWrapper를 사용하여 본문 캐싱
        ContentCachingRequestWrapper wrappedRequest =
            new ContentCachingRequestWrapper(request);

        filterChain.doFilter(wrappedRequest, response);

        byte[] content = wrappedRequest.getContentAsByteArray();
        String body = new String(content, StandardCharsets.UTF_8);
        logger.info("Request body: {}", body);
    }
}
```

## 4. 성능 측정 미들웨어

### 4.1 기본 타이밍 미들웨어

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time

    # 응답 헤더에 처리 시간 추가
    response.headers["X-Process-Time"] = f"{process_time:.4f}"

    return response

@app.get("/fast")
async def fast_endpoint():
    return {"message": "Fast response"}

@app.get("/slow")
async def slow_endpoint():
    import asyncio
    await asyncio.sleep(2)  # 2초 대기
    return {"message": "Slow response"}
```

### 4.2 상세 성능 모니터링

엔드포인트별 성능 통계를 수집합니다.

```python
from fastapi import FastAPI, Request
from collections import defaultdict
import time
import statistics

app = FastAPI()

# 성능 통계 저장
performance_stats = defaultdict(list)

@app.middleware("http")
async def performance_monitoring(request: Request, call_next):
    start_time = time.time()

    # 요청 처리
    response = await call_next(request)

    # 처리 시간 기록
    duration = time.time() - start_time
    endpoint = f"{request.method} {request.url.path}"
    performance_stats[endpoint].append(duration)

    # 응답 헤더에 통계 추가
    stats = performance_stats[endpoint]
    if len(stats) >= 10:
        response.headers["X-Avg-Time"] = f"{statistics.mean(stats[-100:]):.4f}"
        response.headers["X-Min-Time"] = f"{min(stats[-100:]):.4f}"
        response.headers["X-Max-Time"] = f"{max(stats[-100:]):.4f}"

    response.headers["X-Process-Time"] = f"{duration:.4f}"

    return response

@app.get("/stats")
async def get_performance_stats():
    """성능 통계 조회"""
    result = {}
    for endpoint, times in performance_stats.items():
        if times:
            result[endpoint] = {
                "count": len(times),
                "avg": round(statistics.mean(times), 4),
                "min": round(min(times), 4),
                "max": round(max(times), 4),
                "median": round(statistics.median(times), 4),
            }
    return result
```

### 4.3 느린 요청 경고

설정한 임계값을 초과하는 요청을 로깅합니다.

```python
from fastapi import FastAPI, Request
import logging
import time

logger = logging.getLogger(__name__)
app = FastAPI()

SLOW_REQUEST_THRESHOLD = 1.0  # 1초

@app.middleware("http")
async def slow_request_warning(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    # 느린 요청 경고
    if duration > SLOW_REQUEST_THRESHOLD:
        logger.warning(
            f"Slow request detected: {request.method} {request.url.path} "
            f"took {duration:.3f}s (threshold: {SLOW_REQUEST_THRESHOLD}s)"
        )

    response.headers["X-Process-Time"] = f"{duration:.4f}"

    return response
```

## 5. CORS 미들웨어

CORS (Cross-Origin Resource Sharing)를 처리하는 내장 미들웨어입니다.

### 5.1 기본 CORS 설정

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS 미들웨어 추가
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # 허용할 오리진
    allow_credentials=True,  # 쿠키 포함 허용
    allow_methods=["*"],  # 모든 HTTP 메서드 허용
    allow_headers=["*"],  # 모든 헤더 허용
)

@app.get("/api/data")
async def get_data():
    return {"message": "CORS enabled"}
```

### 5.2 환경별 CORS 설정

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os

app = FastAPI()

# 환경 변수에서 CORS 설정 읽기
ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

if ENVIRONMENT == "production":
    allowed_origins = [
        "https://myapp.com",
        "https://www.myapp.com",
    ]
else:
    # 개발 환경에서는 모든 오리진 허용
    allowed_origins = ["*"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
    max_age=3600,  # preflight 요청 캐시 시간 (초)
)
```

**Spring Boot CORS 설정과 비교:**

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

## 6. 커스텀 헤더 추가 미들웨어

### 6.1 보안 헤더 추가

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response = await call_next(request)

    # 보안 헤더 추가
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'"

    return response
```

### 6.2 API 버전 헤더

```python
from fastapi import FastAPI, Request

app = FastAPI()

API_VERSION = "1.0.0"

@app.middleware("http")
async def add_api_version_header(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-API-Version"] = API_VERSION
    return response
```

## 7. 에러 처리 미들웨어

### 7.1 예외 캐칭 미들웨어

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

@app.middleware("http")
async def catch_exceptions_middleware(request: Request, call_next):
    try:
        response = await call_next(request)
        return response
    except Exception as exc:
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

        # 에러 응답 반환
        return JSONResponse(
            status_code=500,
            content={
                "error": "Internal Server Error",
                "message": "An unexpected error occurred",
                "path": request.url.path,
            }
        )
```

### 7.2 타임아웃 미들웨어

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import asyncio

app = FastAPI()

TIMEOUT_SECONDS = 30

@app.middleware("http")
async def timeout_middleware(request: Request, call_next):
    try:
        # 타임아웃 설정
        response = await asyncio.wait_for(
            call_next(request),
            timeout=TIMEOUT_SECONDS
        )
        return response
    except asyncio.TimeoutError:
        return JSONResponse(
            status_code=504,
            content={
                "error": "Gateway Timeout",
                "message": f"Request took longer than {TIMEOUT_SECONDS} seconds",
            }
        )
```

## 8. 인증 미들웨어

### 8.1 API Key 검증 미들웨어

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import os

app = FastAPI()

# 인증이 필요 없는 경로
PUBLIC_PATHS = ["/", "/docs", "/openapi.json", "/health"]

@app.middleware("http")
async def api_key_middleware(request: Request, call_next):
    # 공개 경로는 인증 스킵
    if request.url.path in PUBLIC_PATHS:
        return await call_next(request)

    # API Key 검증
    api_key = request.headers.get("X-API-Key")
    expected_api_key = os.getenv("API_KEY", "secret-key")

    if api_key != expected_api_key:
        return JSONResponse(
            status_code=401,
            content={"error": "Invalid or missing API key"}
        )

    response = await call_next(request)
    return response

@app.get("/public")
async def public_endpoint():
    return {"message": "This is public"}

@app.get("/protected")
async def protected_endpoint():
    return {"message": "This requires authentication"}
```

**참고:** 실제로는 Depends()를 사용한 인증이 더 권장됩니다 (Chapter 13 참고).

## 9. 실전 예제: 통합 미들웨어

여러 미들웨어를 조합한 실전 예제입니다.

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import logging
import time
import uuid
import json

# 로거 설정
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Production API")

# 1. CORS 미들웨어 (가장 먼저)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 2. 보안 헤더 미들웨어
@app.middleware("http")
async def security_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    return response

# 3. 요청 ID 및 로깅 미들웨어
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    # 요청 ID 생성
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id

    start_time = time.time()

    # 요청 로그
    request_log = {
        "request_id": request_id,
        "method": request.method,
        "path": request.url.path,
        "client_ip": request.client.host,
    }
    logger.info(f"REQUEST: {json.dumps(request_log)}")

    # 요청 처리
    try:
        response = await call_next(request)
    except Exception as exc:
        # 예외 로깅
        logger.error(f"ERROR: {request_id} - {exc}", exc_info=True)
        return JSONResponse(
            status_code=500,
            content={"error": "Internal Server Error", "request_id": request_id}
        )

    # 응답 로그
    duration = time.time() - start_time
    response_log = {
        "request_id": request_id,
        "status_code": response.status_code,
        "duration": round(duration, 3),
    }
    logger.info(f"RESPONSE: {json.dumps(response_log)}")

    # 응답 헤더 추가
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{duration:.4f}"

    return response

# 4. 성능 모니터링 미들웨어
SLOW_REQUEST_THRESHOLD = 1.0

@app.middleware("http")
async def performance_monitoring(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    # 느린 요청 경고
    if duration > SLOW_REQUEST_THRESHOLD:
        logger.warning(
            f"Slow request: {request.method} {request.url.path} "
            f"took {duration:.3f}s"
        )

    return response

# 엔드포인트
@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/slow")
async def slow_endpoint():
    import asyncio
    await asyncio.sleep(1.5)
    return {"message": "This was slow"}

@app.get("/error")
async def error_endpoint():
    raise ValueError("Something went wrong")
```

## 10. 미들웨어 모범 사례

### 10.1 미들웨어 설계 원칙

1. **단일 책임 원칙**: 각 미들웨어는 하나의 명확한 역할만 수행
2. **순서 고려**: CORS → 인증 → 로깅 → 비즈니스 로직 순서 권장
3. **성능**: 모든 요청에 영향을 주므로 가볍게 유지
4. **에러 처리**: 예외를 적절히 처리하여 서버 다운 방지
5. **테스트 가능성**: 미들웨어 로직도 단위 테스트 작성

### 10.2 미들웨어 순서 권장사항

```python
app = FastAPI()

# 1. CORS (가장 먼저)
app.add_middleware(CORSMiddleware, ...)

# 2. 보안 헤더
@app.middleware("http")
async def security_headers(...): ...

# 3. 요청 ID 생성
@app.middleware("http")
async def request_id_middleware(...): ...

# 4. 로깅
@app.middleware("http")
async def logging_middleware(...): ...

# 5. 인증 (선택적)
@app.middleware("http")
async def auth_middleware(...): ...

# 6. 성능 모니터링
@app.middleware("http")
async def performance_middleware(...): ...

# 7. 비즈니스 로직 (라우트 핸들러)
```

### 10.3 주의사항

```python
# ❌ 나쁜 예: 응답 본문 읽기 시도
@app.middleware("http")
async def bad_middleware(request: Request, call_next):
    response = await call_next(request)
    # 응답 본문은 스트리밍되므로 직접 읽을 수 없음
    # body = await response.body()  # 작동하지 않음!
    return response

# ✅ 좋은 예: 응답 본문을 읽으려면 특수 처리 필요
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import StreamingResponse

class ResponseBodyLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # StreamingResponse의 경우 본문을 캐싱
        if isinstance(response, StreamingResponse):
            response_body = b""
            async for chunk in response.body_iterator:
                response_body += chunk

            logger.info(f"Response body: {response_body.decode()}")

            # 새로운 응답 생성
            return Response(
                content=response_body,
                status_code=response.status_code,
                headers=dict(response.headers),
            )

        return response
```

## 11. Spring Boot와의 상세 비교

| 기능 | Spring Boot | FastAPI |
|------|-------------|---------|
| **필터 등록** | `@Component` 또는 `FilterRegistrationBean` | `@app.middleware("http")` 또는 `app.add_middleware()` |
| **인터셉터 등록** | `WebMvcConfigurer.addInterceptors()` | 미들웨어 + Depends로 구현 |
| **순서 제어** | `@Order` 어노테이션 | 등록 순서 (역순 실행) |
| **비동기 지원** | `WebFilter` (Spring WebFlux) | 기본 `async/await` |
| **예외 처리** | `HandlerExceptionResolver` | try-except in middleware |
| **요청 수정** | `ContentCachingRequestWrapper` | `request._receive` 재정의 |
| **응답 수정** | `ContentCachingResponseWrapper` | Response 객체 반환 전 수정 |

## 12. 연습 문제

### 12.1 기본 연습

1. 모든 요청에 고유한 Request ID를 부여하고 응답 헤더에 포함시키는 미들웨어를 작성하세요.

2. 특정 IP 주소에서 오는 요청을 차단하는 IP 필터링 미들웨어를 작성하세요.

3. API 버전 헤더를 확인하고, 지원하지 않는 버전이면 에러를 반환하는 미들웨어를 작성하세요.

### 12.2 중급 연습

4. 엔드포인트별 요청 횟수를 추적하고, `/metrics` 엔드포인트에서 조회할 수 있는 미들웨어를 작성하세요.

5. 요청 본문의 크기를 확인하고, 설정한 최대 크기를 초과하면 413 에러를 반환하는 미들웨어를 작성하세요.

6. 응답 시간이 임계값을 초과하면 자동으로 경고 이메일을 보내는 미들웨어를 작성하세요 (이메일 전송은 비동기로).

### 12.3 고급 연습

7. 요청과 응답을 데이터베이스에 기록하는 감사(Audit) 미들웨어를 작성하세요. 민감한 정보는 마스킹 처리하세요.

8. 동일한 사용자의 동시 요청 수를 제한하는 미들웨어를 작성하세요 (사용자 식별은 JWT 토큰 기반).

9. 클라이언트가 보낸 `Accept-Language` 헤더를 분석하여 `request.state`에 언어 정보를 저장하고, 다국어 응답을 제공하는 시스템을 구현하세요.

## 13. 요약

### 핵심 포인트

1. **미들웨어는 모든 요청/응답을 가로채는 함수**
   - 데코레이터 방식: `@app.middleware("http")`
   - 클래스 방식: `BaseHTTPMiddleware` 상속

2. **실행 순서**
   - 요청: 마지막 등록 → 첫 번째 등록 → 라우트
   - 응답: 라우트 → 첫 번째 등록 → 마지막 등록

3. **주요 용도**
   - 로깅 (요청/응답)
   - 성능 측정
   - 인증/인가
   - CORS 처리
   - 보안 헤더 추가
   - 에러 처리

4. **Spring Boot와 비교**
   - Filter ≈ Middleware
   - HandlerInterceptor ≈ Middleware + Dependencies
   - FastAPI는 비동기 기본 지원

5. **모범 사례**
   - 단일 책임 원칙
   - 순서 고려 (CORS → 인증 → 로깅)
   - 가벼운 로직 유지
   - 적절한 에러 처리

### 다음 챕터 예고

**Chapter 18: 예외 처리 (Exception Handling)**

다음 챕터에서는 FastAPI의 예외 처리 메커니즘을 다룹니다:
- HTTPException 사용법
- 커스텀 예외 핸들러 작성
- 전역 예외 처리
- 에러 응답 표준화
- Spring Boot의 @ExceptionHandler와 비교
- 유효성 검증 에러 커스터마이징

미들웨어와 예외 처리를 함께 활용하면 견고하고 유지보수하기 쉬운 API를 만들 수 있습니다!
