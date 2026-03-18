# Chapter 26: 성능 최적화 (Performance Optimization)

## 학습 목표
- 데이터베이스 쿼리 최적화
- 캐싱 전략 (Redis)
- 연결 풀링과 리소스 관리
- 비동기 처리 최적화
- API 응답 시간 개선
- Spring Boot 최적화 기법과 비교

## 1. 데이터베이스 쿼리 최적화

### 1.1 N+1 문제 해결

**문제:**
```python
# ❌ N+1 쿼리 문제
@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    users = db.query(User).all()  # 1번의 쿼리
    result = []
    for user in users:
        # 각 사용자마다 추가 쿼리 (N번)
        posts = db.query(Post).filter(Post.user_id == user.id).all()
        result.append({
            "user": user,
            "posts": posts
        })
    return result
# 총 1 + N번의 쿼리
```

**해결책 1: Eager Loading**
```python
# ✅ Eager Loading으로 해결
from sqlalchemy.orm import joinedload

@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    # 한 번의 쿼리로 사용자와 포스트를 함께 조회
    users = db.query(User).options(
        joinedload(User.posts)
    ).all()

    return [
        {
            "user": user,
            "posts": user.posts  # 추가 쿼리 없음
        }
        for user in users
    ]
# 총 1번의 쿼리 (JOIN 사용)
```

**해결책 2: Subquery Load**
```python
from sqlalchemy.orm import subqueryload

@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    # 2번의 쿼리로 모든 데이터 조회
    users = db.query(User).options(
        subqueryload(User.posts)
    ).all()
    return users
# 총 2번의 쿼리 (대량 데이터에 효율적)
```

**해결책 3: Select In Load**
```python
from sqlalchemy.orm import selectinload

@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    users = db.query(User).options(
        selectinload(User.posts),
        selectinload(User.comments)
    ).all()
    return users
```

### 1.2 인덱스 최적화

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, Index

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)  # 인덱스
    email = Column(String, unique=True, index=True)     # 인덱스
    created_at = Column(DateTime, index=True)           # 정렬용 인덱스

    # 복합 인덱스
    __table_args__ = (
        Index('idx_username_email', 'username', 'email'),
        Index('idx_created_at_desc', 'created_at', postgresql_ops={'created_at': 'DESC'}),
    )

# 쿼리 성능 향상
@app.get("/users/search")
async def search_users(
    username: str = None,
    email: str = None,
    db: Session = Depends(get_db)
):
    query = db.query(User)

    # 인덱스를 활용한 검색
    if username:
        query = query.filter(User.username.like(f"%{username}%"))
    if email:
        query = query.filter(User.email == email)

    return query.all()
```

### 1.3 쿼리 최적화 기법

```python
# ❌ 비효율적: 모든 컬럼 조회
users = db.query(User).all()

# ✅ 효율적: 필요한 컬럼만 조회
users = db.query(User.id, User.username, User.email).all()

# ❌ 비효율적: COUNT(*)
total = db.query(User).count()

# ✅ 효율적: func.count() 사용
from sqlalchemy import func
total = db.query(func.count(User.id)).scalar()

# ❌ 비효율적: LIMIT 없이 전체 조회
all_users = db.query(User).all()

# ✅ 효율적: 페이지네이션
users = db.query(User).offset(skip).limit(limit).all()

# ✅ Bulk Insert (대량 삽입)
users = [User(username=f"user{i}") for i in range(1000)]
db.bulk_save_objects(users)
db.commit()
```

### 1.4 Spring Boot와 비교

**Spring Boot JPA:**
```java
// N+1 문제
@GetMapping("/users")
public List<UserDTO> getUsers() {
    List<User> users = userRepository.findAll();
    return users.stream().map(user -> {
        List<Post> posts = postRepository.findByUserId(user.getId()); // N+1
        return new UserDTO(user, posts);
    }).collect(Collectors.toList());
}

// 해결: @EntityGraph
@EntityGraph(attributePaths = {"posts"})
List<User> findAllWithPosts();

// 또는 JPQL fetch join
@Query("SELECT u FROM User u LEFT JOIN FETCH u.posts")
List<User> findAllWithPosts();
```

## 2. 캐싱 전략

### 2.1 Redis 캐싱 구현

```bash
pip install redis aioredis
```

**Redis 연결 설정:**
```python
# app/core/cache.py
import redis.asyncio as redis
from typing import Optional
from app.core.config import settings

class RedisCache:
    def __init__(self):
        self.redis: Optional[redis.Redis] = None

    async def connect(self):
        """Redis 연결"""
        self.redis = await redis.from_url(
            settings.REDIS_URL,
            encoding="utf-8",
            decode_responses=True
        )

    async def disconnect(self):
        """Redis 연결 해제"""
        if self.redis:
            await self.redis.close()

    async def get(self, key: str) -> Optional[str]:
        """캐시 조회"""
        if not self.redis:
            return None
        return await self.redis.get(key)

    async def set(self, key: str, value: str, expire: int = 3600):
        """캐시 저장"""
        if self.redis:
            await self.redis.setex(key, expire, value)

    async def delete(self, key: str):
        """캐시 삭제"""
        if self.redis:
            await self.redis.delete(key)

    async def exists(self, key: str) -> bool:
        """캐시 존재 확인"""
        if not self.redis:
            return False
        return await self.redis.exists(key) > 0

# 싱글톤 인스턴스
cache = RedisCache()

# app/main.py
@app.on_event("startup")
async def startup():
    await cache.connect()

@app.on_event("shutdown")
async def shutdown():
    await cache.disconnect()
```

### 2.2 캐싱 적용

```python
# app/api/v1/endpoints/users.py
from app.core.cache import cache
import json

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    # 캐시 확인
    cache_key = f"user:{user_id}"
    cached_user = await cache.get(cache_key)

    if cached_user:
        return json.loads(cached_user)

    # DB 조회
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404)

    # 캐시 저장 (1시간)
    user_dict = {
        "id": user.id,
        "username": user.username,
        "email": user.email
    }
    await cache.set(cache_key, json.dumps(user_dict), expire=3600)

    return user_dict

@app.put("/users/{user_id}")
async def update_user(
    user_id: int,
    user_update: UserUpdate,
    db: Session = Depends(get_db)
):
    # 사용자 업데이트
    user = db.query(User).filter(User.id == user_id).first()
    # ... 업데이트 로직 ...

    # 캐시 무효화
    await cache.delete(f"user:{user_id}")

    return user
```

### 2.3 캐싱 데코레이터

```python
# app/decorators/cache.py
from functools import wraps
import json
import hashlib
from app.core.cache import cache

def cached(expire: int = 3600, key_prefix: str = ""):
    """캐싱 데코레이터"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 캐시 키 생성
            key_data = f"{key_prefix}:{func.__name__}:{args}:{kwargs}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # 캐시 확인
            cached_value = await cache.get(cache_key)
            if cached_value:
                return json.loads(cached_value)

            # 함수 실행
            result = await func(*args, **kwargs)

            # 캐시 저장
            await cache.set(cache_key, json.dumps(result), expire)

            return result
        return wrapper
    return decorator

# 사용
@cached(expire=300, key_prefix="products")
async def get_products(category: str, page: int = 1):
    # 무거운 쿼리
    products = await db.query(Product).filter_by(category=category).all()
    return [product.dict() for product in products]
```

### 2.4 캐시 전략 패턴

**Cache-Aside (Lazy Loading):**
```python
async def get_user_cache_aside(user_id: int):
    # 1. 캐시 확인
    user = await cache.get(f"user:{user_id}")
    if user:
        return json.loads(user)

    # 2. DB 조회
    user = db.query(User).get(user_id)

    # 3. 캐시 저장
    await cache.set(f"user:{user_id}", json.dumps(user.dict()))

    return user
```

**Write-Through:**
```python
async def update_user_write_through(user_id: int, data: dict):
    # 1. DB 업데이트
    user = db.query(User).get(user_id)
    for key, value in data.items():
        setattr(user, key, value)
    db.commit()

    # 2. 캐시 업데이트 (DB와 동시)
    await cache.set(f"user:{user_id}", json.dumps(user.dict()))

    return user
```

**Write-Behind:**
```python
from fastapi import BackgroundTasks

async def update_user_write_behind(
    user_id: int,
    data: dict,
    background_tasks: BackgroundTasks
):
    # 1. 캐시 업데이트 (즉시)
    await cache.set(f"user:{user_id}", json.dumps(data))

    # 2. DB 업데이트 (백그라운드)
    background_tasks.add_task(update_db, user_id, data)

    return data

def update_db(user_id: int, data: dict):
    user = db.query(User).get(user_id)
    for key, value in data.items():
        setattr(user, key, value)
    db.commit()
```

## 3. 연결 풀링

### 3.1 데이터베이스 연결 풀

```python
# app/core/database.py
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    settings.DATABASE_URL,
    poolclass=QueuePool,
    pool_size=20,              # 기본 연결 수
    max_overflow=10,           # 추가 가능한 연결 수
    pool_timeout=30,           # 연결 대기 시간
    pool_recycle=3600,         # 연결 재활용 시간 (1시간)
    pool_pre_ping=True,        # 연결 유효성 검사
    echo_pool=False,           # 풀 로깅
)

# 연결 풀 모니터링
@app.get("/pool-status")
def get_pool_status():
    pool = engine.pool
    return {
        "size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "total": pool.size() + pool.overflow()
    }
```

### 3.2 HTTP 연결 풀

```python
# app/core/http_client.py
import httpx
from app.core.config import settings

class HTTPClient:
    def __init__(self):
        self.client = httpx.AsyncClient(
            timeout=30.0,
            limits=httpx.Limits(
                max_keepalive_connections=20,  # Keep-alive 연결 수
                max_connections=100,           # 최대 연결 수
                keepalive_expiry=30.0         # Keep-alive 만료 시간
            )
        )

    async def get(self, url: str, **kwargs):
        return await self.client.get(url, **kwargs)

    async def post(self, url: str, **kwargs):
        return await self.client.post(url, **kwargs)

    async def close(self):
        await self.client.aclose()

# 싱글톤
http_client = HTTPClient()

@app.on_event("shutdown")
async def shutdown():
    await http_client.close()
```

## 4. 비동기 최적화

### 4.1 동시 실행 최적화

```python
import asyncio

# ❌ 순차 실행 (느림)
@app.get("/dashboard")
async def get_dashboard_slow():
    user = await get_user_data()      # 1초
    posts = await get_posts_data()     # 1초
    stats = await get_stats_data()     # 1초
    return {"user": user, "posts": posts, "stats": stats}
# 총 3초

# ✅ 병렬 실행 (빠름)
@app.get("/dashboard")
async def get_dashboard_fast():
    user, posts, stats = await asyncio.gather(
        get_user_data(),    # 동시 실행
        get_posts_data(),   # 동시 실행
        get_stats_data()    # 동시 실행
    )
    return {"user": user, "posts": posts, "stats": stats}
# 총 1초 (가장 긴 작업 기준)
```

### 4.2 세마포어로 동시성 제한

```python
# 외부 API 호출 제한
external_api_semaphore = asyncio.Semaphore(5)  # 최대 5개 동시 호출

async def call_external_api(item_id: int):
    async with external_api_semaphore:
        async with httpx.AsyncClient() as client:
            response = await client.get(f"https://api.example.com/items/{item_id}")
            return response.json()

@app.get("/items/batch")
async def get_items_batch(item_ids: List[int]):
    # 최대 5개씩 동시 호출
    results = await asyncio.gather(
        *[call_external_api(item_id) for item_id in item_ids]
    )
    return results
```

### 4.3 타임아웃 설정

```python
@app.get("/slow-endpoint")
async def slow_endpoint():
    try:
        # 5초 타임아웃
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=5.0
        )
        return result
    except asyncio.TimeoutError:
        raise HTTPException(status_code=504, detail="Operation timed out")
```

## 5. 응답 압축

### 5.1 GZip 압축

```python
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# GZip 압축 미들웨어
app.add_middleware(
    GZipMiddleware,
    minimum_size=1000  # 1KB 이상만 압축
)

@app.get("/large-data")
def get_large_data():
    # 큰 JSON 데이터 반환
    return {"data": [{"id": i, "value": "x" * 100} for i in range(1000)]}
# 자동으로 압축되어 전송됨
```

### 5.2 응답 스트리밍

```python
from fastapi.responses import StreamingResponse
import asyncio

async def generate_large_file():
    """대용량 파일 스트리밍"""
    for i in range(10000):
        await asyncio.sleep(0.001)
        yield f"Line {i}\n".encode()

@app.get("/stream")
async def stream_data():
    return StreamingResponse(
        generate_large_file(),
        media_type="text/plain"
    )
```

## 6. 페이지네이션 최적화

### 6.1 Offset-based Pagination

```python
from typing import Generic, TypeVar, List
from pydantic import BaseModel

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    size: int
    pages: int

@app.get("/users", response_model=PaginatedResponse[User])
async def get_users(
    page: int = 1,
    size: int = 50,
    db: Session = Depends(get_db)
):
    # 총 개수 조회 (캐시 권장)
    total = db.query(User).count()

    # 페이지네이션
    users = db.query(User).offset((page - 1) * size).limit(size).all()

    return PaginatedResponse(
        items=users,
        total=total,
        page=page,
        size=size,
        pages=(total + size - 1) // size
    )
```

### 6.2 Cursor-based Pagination (더 효율적)

```python
@app.get("/users/cursor")
async def get_users_cursor(
    cursor: Optional[int] = None,
    size: int = 50,
    db: Session = Depends(get_db)
):
    query = db.query(User)

    if cursor:
        # 커서 이후 데이터만 조회
        query = query.filter(User.id > cursor)

    users = query.order_by(User.id).limit(size).all()

    next_cursor = users[-1].id if users else None

    return {
        "items": users,
        "next_cursor": next_cursor,
        "has_more": len(users) == size
    }
```

## 7. 프로파일링

### 7.1 실행 시간 측정

```python
import time
from functools import wraps
import logging

logger = logging.getLogger(__name__)

def profile(func):
    """실행 시간 측정 데코레이터"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        result = await func(*args, **kwargs)
        duration = time.time() - start

        logger.info(f"{func.__name__} took {duration:.3f}s")

        return result
    return wrapper

@profile
async def slow_function():
    await asyncio.sleep(1)
    return "done"
```

### 7.2 메모리 프로파일링

```bash
pip install memory-profiler
```

```python
from memory_profiler import profile

@profile
def memory_intensive_function():
    large_list = [i for i in range(1000000)]
    return sum(large_list)
```

### 7.3 Line Profiler

```bash
pip install line-profiler
```

```python
from line_profiler import LineProfiler

def profile_function():
    profiler = LineProfiler()
    profiler.add_function(target_function)
    profiler.run('target_function()')
    profiler.print_stats()
```

## 8. 모니터링과 메트릭

### 8.1 Prometheus 메트릭

```bash
pip install prometheus-fastapi-instrumentator
```

```python
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

# Prometheus 메트릭 자동 수집
Instrumentator().instrument(app).expose(app)

# /metrics 엔드포인트에서 메트릭 확인
```

### 8.2 커스텀 메트릭

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# 요청 카운터
request_count = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

# 응답 시간 히스토그램
response_time = Histogram(
    'api_response_time_seconds',
    'API response time',
    ['endpoint']
)

# 활성 사용자 게이지
active_users = Gauge('active_users', 'Number of active users')

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()

    response = await call_next(request)

    # 메트릭 기록
    duration = time.time() - start_time
    request_count.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    response_time.labels(endpoint=request.url.path).observe(duration)

    return response
```

## 9. 실전 최적화 체크리스트

### 9.1 데이터베이스

- [ ] N+1 쿼리 제거 (Eager Loading)
- [ ] 적절한 인덱스 생성
- [ ] 페이지네이션 구현
- [ ] 필요한 컬럼만 조회
- [ ] Bulk 작업 활용
- [ ] 연결 풀 설정

### 9.2 캐싱

- [ ] 자주 조회되는 데이터 캐싱
- [ ] 캐시 만료 시간 설정
- [ ] 캐시 무효화 전략
- [ ] Redis 또는 Memcached 사용

### 9.3 비동기

- [ ] I/O 작업 비동기화
- [ ] 병렬 처리 활용 (asyncio.gather)
- [ ] 동시성 제한 (Semaphore)
- [ ] 타임아웃 설정

### 9.4 응답

- [ ] GZip 압축 활성화
- [ ] 응답 크기 최소화
- [ ] 스트리밍 고려
- [ ] 적절한 HTTP 상태 코드

### 9.5 모니터링

- [ ] 메트릭 수집 (Prometheus)
- [ ] 로깅 구조화
- [ ] 에러 추적 (Sentry)
- [ ] 성능 프로파일링

## 10. 성능 측정

### 10.1 부하 테스트

```python
# locustfile.py
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 3)

    @task(3)
    def get_users(self):
        self.client.get("/users?page=1&size=50")

    @task(1)
    def get_user(self):
        self.client.get("/users/1")

    @task(2)
    def search_users(self):
        self.client.get("/users/search?q=john")
```

**실행:**
```bash
locust -f locustfile.py --host=http://localhost:8000 --users 100 --spawn-rate 10
```

### 10.2 벤치마크 비교

```python
import pytest
from fastapi.testclient import TestClient

def test_endpoint_performance(benchmark, client):
    """엔드포인트 성능 테스트"""
    result = benchmark(lambda: client.get("/users"))
    assert result.status_code == 200

# 실행 후 결과:
# Name                              Min     Max    Mean  StdDev  Median
# test_endpoint_performance      0.0125  0.0234  0.0156  0.0023  0.0152
```

## 11. 요약

### 핵심 포인트

1. **데이터베이스 최적화**
   - N+1 문제 해결 (Eager Loading)
   - 인덱스 활용
   - 쿼리 최적화

2. **캐싱**
   - Redis 활용
   - Cache-Aside 패턴
   - 적절한 만료 시간

3. **연결 풀링**
   - DB 연결 풀 설정
   - HTTP 클라이언트 재사용

4. **비동기 최적화**
   - asyncio.gather로 병렬 처리
   - 세마포어로 동시성 제한
   - 타임아웃 설정

5. **응답 최적화**
   - GZip 압축
   - 페이지네이션
   - 스트리밍

6. **모니터링**
   - Prometheus 메트릭
   - 프로파일링
   - 부하 테스트

### 다음 챕터 예고

**Chapter 27: 모니터링과 로깅 (Monitoring and Logging)**

다음 챕터에서는 프로덕션 환경의 모니터링과 로깅을 다룹니다:
- 구조화된 로깅 (JSON Logging)
- 중앙 집중식 로깅 (ELK Stack)
- APM (Application Performance Monitoring)
- 알림 시스템 (Alerting)
- 대시보드 구성 (Grafana)

성능 최적화로 빠르고 효율적인 API를 만드세요!
