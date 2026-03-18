# Chapter 20: 비동기 프로그래밍 (Async Programming)

## 학습 목표
- Python의 async/await 문법 완전 이해
- 동시성(Concurrency)과 병렬성(Parallelism) 차이 이해
- 비동기 데이터베이스 쿼리 작성
- 비동기 HTTP 클라이언트 사용
- asyncio 고급 패턴과 모범 사례
- Spring WebFlux와 비교

## 1. 비동기 프로그래밍 개요

### 1.1 동기 vs 비동기

**동기 (Synchronous):**
```python
import time

def fetch_data_sync():
    print("Fetching data...")
    time.sleep(2)  # I/O 대기 (네트워크, DB 등)
    print("Data fetched!")
    return {"data": "result"}

def main_sync():
    print("Start")
    result1 = fetch_data_sync()  # 2초 대기
    result2 = fetch_data_sync()  # 2초 대기
    print("End")
    # 총 시간: 4초

main_sync()
```

**비동기 (Asynchronous):**
```python
import asyncio

async def fetch_data_async():
    print("Fetching data...")
    await asyncio.sleep(2)  # 비동기 대기
    print("Data fetched!")
    return {"data": "result"}

async def main_async():
    print("Start")
    # 두 작업을 동시에 실행
    result1, result2 = await asyncio.gather(
        fetch_data_async(),
        fetch_data_async()
    )
    print("End")
    # 총 시간: 2초 (동시 실행)

asyncio.run(main_async())
```

### 1.2 동시성 vs 병렬성

**동시성 (Concurrency):**
- 여러 작업을 번갈아가며 실행 (한 번에 하나씩)
- 싱글 스레드에서 가능
- I/O 바운드 작업에 적합 (네트워크, 파일, DB)

**병렬성 (Parallelism):**
- 여러 작업을 진짜 동시에 실행 (여러 CPU 코어 사용)
- 멀티 프로세스/스레드 필요
- CPU 바운드 작업에 적합 (계산, 데이터 처리)

```
동시성 (Concurrency):
Task A: ████░░░░████░░░░████
Task B: ░░░░████░░░░████░░░░
Time:   ────────────────────→
        (한 번에 하나씩 빠르게 전환)

병렬성 (Parallelism):
Task A: ████████████████████
Task B: ████████████████████
Time:   ────────────────────→
        (정말로 동시에 실행)
```

### 1.3 Spring Boot와 비교

| 구분 | Spring Boot | FastAPI |
|------|-------------|---------|
| **동기 방식** | Spring MVC (기본) | def 함수 |
| **비동기 방식** | Spring WebFlux (Reactor) | async def 함수 |
| **Reactive 타입** | Mono, Flux | Coroutine |
| **스레드 모델** | Thread per request | Event Loop |
| **배압 (Backpressure)** | Reactor 지원 | 제한적 |
| **학습 곡선** | 복잡 (Reactor 학습 필요) | 간단 (async/await) |

**Spring WebFlux 예시:**
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userRepository.findById(id);
    }

    @GetMapping("/users")
    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }
}
```

**FastAPI 예시:**
```python
@app.get("/users/{id}")
async def get_user(id: int):
    user = await user_repository.find_by_id(id)
    return user

@app.get("/users")
async def get_all_users():
    users = await user_repository.find_all()
    return users
```

## 2. async/await 기본

### 2.1 async def와 await

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

# 비동기 함수 정의
async def fetch_user_from_db(user_id: int):
    """데이터베이스에서 사용자 조회 (시뮬레이션)"""
    await asyncio.sleep(1)  # DB 쿼리 시뮬레이션
    return {"id": user_id, "name": f"User {user_id}"}

# 비동기 엔드포인트
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # await 키워드로 비동기 함수 호출
    user = await fetch_user_from_db(user_id)
    return user
```

**중요 규칙:**
1. `async def`로 정의한 함수는 코루틴(coroutine)을 반환
2. 코루틴은 `await` 키워드로 실행
3. `await`는 `async def` 함수 안에서만 사용 가능
4. `await` 중에는 다른 작업이 실행될 수 있음 (Non-blocking)

### 2.2 동기 함수 vs 비동기 함수

FastAPI에서 두 방식 모두 사용 가능합니다.

```python
from fastapi import FastAPI
import time
import asyncio

app = FastAPI()

# 동기 함수 (블로킹)
@app.get("/sync")
def sync_endpoint():
    time.sleep(2)  # 블로킹 - 다른 요청도 대기
    return {"message": "Sync response"}

# 비동기 함수 (논블로킹)
@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(2)  # 논블로킹 - 다른 요청 처리 가능
    return {"message": "Async response"}
```

**성능 비교:**
```bash
# 동기 엔드포인트에 10개 동시 요청
# 총 시간: ~20초 (순차 처리)

# 비동기 엔드포인트에 10개 동시 요청
# 총 시간: ~2초 (동시 처리)
```

### 2.3 언제 async를 사용할까?

**async def 사용 (권장):**
- 데이터베이스 쿼리 (`await db.execute()`)
- 외부 API 호출 (`await httpx.get()`)
- 파일 읽기/쓰기 (`await aiofiles.open()`)
- 캐시 조회 (Redis: `await redis.get()`)

**def 사용:**
- 간단한 계산
- 메모리 내 데이터 조회
- 동기 라이브러리만 사용 가능한 경우

```python
# ✅ 좋은 예: I/O 작업은 async
@app.get("/data")
async def get_data():
    data = await db.fetch("SELECT * FROM users")
    result = await external_api.get("/data")
    return {"data": data, "result": result}

# ✅ 좋은 예: 간단한 계산은 동기
@app.get("/calculate")
def calculate(a: int, b: int):
    return {"result": a + b}

# ❌ 나쁜 예: async 안에서 블로킹 호출
@app.get("/bad")
async def bad_example():
    time.sleep(2)  # 블로킹! - asyncio.sleep() 사용해야 함
    return {"message": "Bad"}
```

## 3. 여러 작업 동시 실행

### 3.1 asyncio.gather()

여러 비동기 작업을 동시에 실행하고 모든 결과를 기다립니다.

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

async def fetch_user(user_id: int):
    await asyncio.sleep(1)
    return {"id": user_id, "name": f"User {user_id}"}

async def fetch_posts(user_id: int):
    await asyncio.sleep(1)
    return [{"id": 1, "title": "Post 1"}, {"id": 2, "title": "Post 2"}]

async def fetch_comments(user_id: int):
    await asyncio.sleep(1)
    return [{"id": 1, "text": "Comment 1"}]

@app.get("/users/{user_id}/full")
async def get_user_full_profile(user_id: int):
    # 3개 작업을 동시에 실행 (총 1초)
    user, posts, comments = await asyncio.gather(
        fetch_user(user_id),
        fetch_posts(user_id),
        fetch_comments(user_id)
    )

    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }

# 순차 실행과 비교
@app.get("/users/{user_id}/full-slow")
async def get_user_full_profile_slow(user_id: int):
    # 순차 실행 (총 3초)
    user = await fetch_user(user_id)  # 1초
    posts = await fetch_posts(user_id)  # 1초
    comments = await fetch_comments(user_id)  # 1초

    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }
```

### 3.2 asyncio.create_task()

작업을 백그라운드에서 실행하고 나중에 결과를 받습니다.

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

async def long_task(task_id: int):
    await asyncio.sleep(2)
    return {"task_id": task_id, "result": "completed"}

@app.get("/parallel-tasks")
async def run_parallel_tasks():
    # 작업 생성 (즉시 시작)
    task1 = asyncio.create_task(long_task(1))
    task2 = asyncio.create_task(long_task(2))
    task3 = asyncio.create_task(long_task(3))

    # 다른 작업 수행 가능
    print("Tasks started, doing other work...")

    # 모든 작업 완료 대기
    results = await asyncio.gather(task1, task2, task3)

    return {"results": results}
```

### 3.3 에러 처리

```python
from fastapi import FastAPI, HTTPException
import asyncio

app = FastAPI()

async def task_that_might_fail(task_id: int):
    await asyncio.sleep(1)
    if task_id == 2:
        raise ValueError(f"Task {task_id} failed!")
    return {"task_id": task_id, "status": "success"}

@app.get("/tasks-with-error-handling")
async def run_tasks_with_error_handling():
    tasks = [
        task_that_might_fail(1),
        task_that_might_fail(2),  # 이 작업은 실패
        task_that_might_fail(3),
    ]

    # return_exceptions=True: 예외를 결과로 반환
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # 결과 처리
    successful_results = []
    errors = []

    for i, result in enumerate(results):
        if isinstance(result, Exception):
            errors.append({"task_id": i + 1, "error": str(result)})
        else:
            successful_results.append(result)

    return {
        "successful": successful_results,
        "errors": errors
    }
```

## 4. 비동기 데이터베이스 쿼리

### 4.1 SQLAlchemy 비동기 사용

```bash
pip install sqlalchemy[asyncio] aiosqlite
```

**database.py:**
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import declarative_base

DATABASE_URL = "sqlite+aiosqlite:///./test.db"

# 비동기 엔진 생성
engine = create_async_engine(DATABASE_URL, echo=True)

# 비동기 세션 팩토리
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

Base = declarative_base()

# 의존성
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

**models.py:**
```python
from sqlalchemy import Column, Integer, String
from database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)
```

**main.py:**
```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from database import get_db, engine, Base
from models import User

app = FastAPI()

# 테이블 생성
@app.on_event("startup")
async def startup():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

# 사용자 조회 (비동기)
@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).filter(User.id == user_id))
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return {
        "id": user.id,
        "username": user.username,
        "email": user.email
    }

# 모든 사용자 조회
@app.get("/users")
async def get_all_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    users = result.scalars().all()

    return [
        {"id": user.id, "username": user.username, "email": user.email}
        for user in users
    ]

# 사용자 생성
@app.post("/users")
async def create_user(
    username: str,
    email: str,
    db: AsyncSession = Depends(get_db)
):
    user = User(username=username, email=email)
    db.add(user)
    await db.commit()
    await db.refresh(user)

    return {"id": user.id, "username": user.username, "email": user.email}

# 사용자 업데이트
@app.put("/users/{user_id}")
async def update_user(
    user_id: int,
    username: str,
    email: str,
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(select(User).filter(User.id == user_id))
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    user.username = username
    user.email = email
    await db.commit()
    await db.refresh(user)

    return {"id": user.id, "username": user.username, "email": user.email}

# 사용자 삭제
@app.delete("/users/{user_id}")
async def delete_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).filter(User.id == user_id))
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    await db.delete(user)
    await db.commit()

    return {"message": "User deleted"}
```

### 4.2 복잡한 쿼리

```python
from sqlalchemy import select, and_, or_
from sqlalchemy.orm import selectinload

# WHERE 조건
@app.get("/users/search")
async def search_users(
    username: str = None,
    email: str = None,
    db: AsyncSession = Depends(get_db)
):
    query = select(User)

    if username:
        query = query.filter(User.username.like(f"%{username}%"))
    if email:
        query = query.filter(User.email.like(f"%{email}%"))

    result = await db.execute(query)
    users = result.scalars().all()

    return [
        {"id": user.id, "username": user.username, "email": user.email}
        for user in users
    ]

# JOIN (관계 로딩)
@app.get("/users/{user_id}/with-posts")
async def get_user_with_posts(user_id: int, db: AsyncSession = Depends(get_db)):
    # selectinload로 eager loading
    query = select(User).options(selectinload(User.posts)).filter(User.id == user_id)
    result = await db.execute(query)
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return {
        "id": user.id,
        "username": user.username,
        "posts": [{"id": post.id, "title": post.title} for post in user.posts]
    }
```

## 5. 비동기 HTTP 클라이언트

### 5.1 httpx 사용

```bash
pip install httpx
```

**동기 requests vs 비동기 httpx:**

```python
# ❌ 동기 (requests)
import requests

@app.get("/external-sync")
def get_external_data_sync():
    response = requests.get("https://api.example.com/data")
    return response.json()

# ✅ 비동기 (httpx)
import httpx

@app.get("/external-async")
async def get_external_data_async():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

### 5.2 여러 API 동시 호출

```python
from fastapi import FastAPI
import httpx
import asyncio

app = FastAPI()

@app.get("/aggregated-data")
async def get_aggregated_data():
    async with httpx.AsyncClient() as client:
        # 3개 API를 동시에 호출
        responses = await asyncio.gather(
            client.get("https://api.example.com/users"),
            client.get("https://api.example.com/posts"),
            client.get("https://api.example.com/comments")
        )

    users = responses[0].json()
    posts = responses[1].json()
    comments = responses[2].json()

    return {
        "users": users,
        "posts": posts,
        "comments": comments
    }
```

### 5.3 타임아웃과 에러 처리

```python
from fastapi import FastAPI, HTTPException
import httpx

app = FastAPI()

@app.get("/external-with-timeout")
async def get_external_with_timeout():
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get("https://api.example.com/slow-endpoint")
            response.raise_for_status()  # 4xx/5xx 에러 발생
            return response.json()

    except httpx.TimeoutException:
        raise HTTPException(status_code=504, detail="External API timeout")
    except httpx.HTTPStatusError as e:
        raise HTTPException(status_code=e.response.status_code, detail="External API error")
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Unexpected error: {e}")
```

### 5.4 재사용 가능한 클라이언트

```python
from fastapi import FastAPI
import httpx

app = FastAPI()

# 전역 클라이언트 (앱 수명 동안 유지)
http_client = None

@app.on_event("startup")
async def startup_event():
    global http_client
    http_client = httpx.AsyncClient(
        timeout=10.0,
        limits=httpx.Limits(max_keepalive_connections=5, max_connections=10)
    )

@app.on_event("shutdown")
async def shutdown_event():
    await http_client.aclose()

@app.get("/data")
async def get_data():
    response = await http_client.get("https://api.example.com/data")
    return response.json()
```

## 6. 비동기 파일 처리

### 6.1 aiofiles 사용

```bash
pip install aiofiles
```

```python
from fastapi import FastAPI, UploadFile, File
import aiofiles

app = FastAPI()

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    # 비동기 파일 쓰기
    async with aiofiles.open(f"uploads/{file.filename}", "wb") as f:
        content = await file.read()
        await f.write(content)

    return {"filename": file.filename, "size": len(content)}

@app.get("/files/{filename}")
async def read_file(filename: str):
    # 비동기 파일 읽기
    try:
        async with aiofiles.open(f"uploads/{filename}", "rb") as f:
            content = await f.read()
        return {"content": content.decode("utf-8")}
    except FileNotFoundError:
        raise HTTPException(status_code=404, detail="File not found")
```

## 7. asyncio 고급 패턴

### 7.1 asyncio.wait()

timeout과 함께 작업을 관리합니다.

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

async def slow_task(task_id: int):
    await asyncio.sleep(task_id * 2)
    return {"task_id": task_id, "status": "completed"}

@app.get("/tasks-with-timeout")
async def run_tasks_with_timeout():
    tasks = [
        asyncio.create_task(slow_task(1)),  # 2초
        asyncio.create_task(slow_task(2)),  # 4초
        asyncio.create_task(slow_task(3)),  # 6초
    ]

    # 3초 타임아웃
    done, pending = await asyncio.wait(tasks, timeout=3.0)

    # 완료된 작업
    completed = [await task for task in done]

    # 미완료 작업 취소
    for task in pending:
        task.cancel()

    return {
        "completed": completed,
        "cancelled": len(pending)
    }
```

### 7.2 asyncio.as_completed()

작업이 완료되는 대로 처리합니다.

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

async def fetch_data(source_id: int):
    await asyncio.sleep(source_id)  # 소스마다 다른 시간
    return {"source_id": source_id, "data": f"Data from source {source_id}"}

@app.get("/streaming-results")
async def get_streaming_results():
    tasks = [fetch_data(i) for i in [3, 1, 2]]  # 다른 순서로 완료됨

    results = []
    for coro in asyncio.as_completed(tasks):
        result = await coro
        results.append(result)
        print(f"Received: {result}")  # 1 → 2 → 3 순서로 받음

    return {"results": results}
```

### 7.3 세마포어 (동시 실행 제한)

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

# 최대 3개만 동시 실행
semaphore = asyncio.Semaphore(3)

async def limited_task(task_id: int):
    async with semaphore:
        print(f"Task {task_id} started")
        await asyncio.sleep(2)
        print(f"Task {task_id} completed")
        return {"task_id": task_id}

@app.get("/limited-concurrent-tasks")
async def run_limited_tasks():
    # 10개 작업 생성하지만, 동시에 3개만 실행
    tasks = [limited_task(i) for i in range(10)]
    results = await asyncio.gather(*tasks)

    return {"results": results}
```

## 8. 성능 최적화

### 8.1 동기 함수를 비동기로 실행

CPU 바운드 작업이나 블로킹 라이브러리를 별도 스레드에서 실행합니다.

```python
from fastapi import FastAPI
import asyncio
import time

app = FastAPI()

def cpu_intensive_task(n: int):
    """CPU 집약적 작업 (블로킹)"""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

@app.get("/compute/{n}")
async def compute(n: int):
    # 블로킹 함수를 별도 스레드에서 실행
    result = await asyncio.to_thread(cpu_intensive_task, n)
    return {"result": result}

# 또는 run_in_executor 사용
@app.get("/compute2/{n}")
async def compute2(n: int):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, cpu_intensive_task, n)
    return {"result": result}
```

### 8.2 연결 풀링

데이터베이스 연결을 재사용합니다.

```python
from sqlalchemy.ext.asyncio import create_async_engine

# 연결 풀 설정
engine = create_async_engine(
    DATABASE_URL,
    echo=True,
    pool_size=20,  # 연결 풀 크기
    max_overflow=0,  # 추가 연결 허용 안함
    pool_pre_ping=True,  # 연결 유효성 검사
)
```

### 8.3 캐싱

자주 조회되는 데이터를 캐싱합니다.

```python
from fastapi import FastAPI
from functools import lru_cache
import asyncio

app = FastAPI()

# 간단한 인메모리 캐시
cache = {}

async def get_expensive_data(key: str):
    if key in cache:
        print(f"Cache hit: {key}")
        return cache[key]

    print(f"Cache miss: {key}, fetching...")
    await asyncio.sleep(2)  # 시뮬레이션
    data = {"key": key, "value": f"data for {key}"}

    cache[key] = data
    return data

@app.get("/cached-data/{key}")
async def get_cached_data(key: str):
    return await get_expensive_data(key)
```

## 9. 실전 예제: 완전한 비동기 API

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy import select
from pydantic import BaseModel
import httpx
import asyncio

# ===== 설정 =====
DATABASE_URL = "sqlite+aiosqlite:///./app.db"
engine = create_async_engine(DATABASE_URL)
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

app = FastAPI(title="Async API Example")

# HTTP 클라이언트
http_client = httpx.AsyncClient(timeout=10.0)

@app.on_event("shutdown")
async def shutdown():
    await http_client.aclose()

# ===== 의존성 =====
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

# ===== 모델 =====
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    external_data: dict = None

# ===== 비즈니스 로직 =====
async def fetch_user_from_db(user_id: int, db: AsyncSession):
    """데이터베이스에서 사용자 조회"""
    result = await db.execute(select(User).filter(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

async def fetch_external_api(user_id: int):
    """외부 API 호출"""
    try:
        response = await http_client.get(f"https://api.example.com/users/{user_id}")
        response.raise_for_status()
        return response.json()
    except Exception as e:
        # 외부 API 실패해도 계속 진행
        return {"error": str(e)}

async def log_access(user_id: int):
    """접근 로그 기록"""
    await asyncio.sleep(0.1)  # 시뮬레이션
    print(f"Access logged for user {user_id}")

# ===== 엔드포인트 =====
@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user_with_external_data(
    user_id: int,
    db: AsyncSession = Depends(get_db)
):
    """사용자 정보 + 외부 API 데이터 + 로깅 (모두 비동기)"""

    # 3개 작업을 동시에 실행
    user, external_data, _ = await asyncio.gather(
        fetch_user_from_db(user_id, db),
        fetch_external_api(user_id),
        log_access(user_id),
        return_exceptions=True  # 일부 실패해도 계속
    )

    return UserResponse(
        id=user.id,
        username=user.username,
        email=user.email,
        external_data=external_data if not isinstance(external_data, Exception) else None
    )

@app.get("/users")
async def get_all_users(
    limit: int = 100,
    db: AsyncSession = Depends(get_db)
):
    """모든 사용자 조회 (비동기)"""
    result = await db.execute(select(User).limit(limit))
    users = result.scalars().all()

    return [
        {"id": user.id, "username": user.username, "email": user.email}
        for user in users
    ]

@app.post("/users/batch")
async def create_users_batch(
    usernames: list[str],
    db: AsyncSession = Depends(get_db)
):
    """여러 사용자를 동시에 생성"""
    users = [User(username=username, email=f"{username}@example.com") for username in usernames]

    for user in users:
        db.add(user)

    await db.commit()

    # 모든 사용자를 refresh (동시에)
    await asyncio.gather(*[db.refresh(user) for user in users])

    return [
        {"id": user.id, "username": user.username}
        for user in users
    ]
```

## 10. 모범 사례

### 10.1 async/await 사용 가이드

```python
# ✅ 좋은 예
@app.get("/good")
async def good_example():
    # 비동기 I/O 작업
    user = await db.get_user(1)
    posts = await db.get_posts(user.id)
    return {"user": user, "posts": posts}

# ❌ 나쁜 예: async 없이 await
@app.get("/bad1")
def bad_example1():
    user = await db.get_user(1)  # SyntaxError!
    return user

# ❌ 나쁜 예: await 없이 비동기 함수 호출
@app.get("/bad2")
async def bad_example2():
    user = db.get_user(1)  # 코루틴 객체 반환 (실행 안됨)
    return user

# ❌ 나쁜 예: async 안에서 블로킹 호출
@app.get("/bad3")
async def bad_example3():
    time.sleep(2)  # 블로킹! 다른 요청도 대기
    return {"message": "Done"}
```

### 10.2 에러 처리

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    try:
        user = await fetch_user(user_id)
        return user
    except asyncio.TimeoutError:
        raise HTTPException(status_code=504, detail="Request timeout")
    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")
```

### 10.3 리소스 정리

```python
# ✅ 좋은 예: Context manager 사용
@app.get("/data")
async def get_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()

# ❌ 나쁜 예: 수동 정리 (누락 가능)
@app.get("/data-bad")
async def get_data_bad():
    client = httpx.AsyncClient()
    response = await client.get("https://api.example.com/data")
    # client.aclose() 호출 안함! 리소스 누수
    return response.json()
```

## 11. 연습 문제

### 11.1 기본 연습

1. 2개의 외부 API를 동시에 호출하고 결과를 합쳐서 반환하는 엔드포인트를 작성하세요.

2. 데이터베이스에서 사용자를 조회하고, 조회 기록을 백그라운드에서 로깅하는 비동기 엔드포인트를 작성하세요.

3. 파일을 비동기로 읽어서 내용을 반환하는 엔드포인트를 작성하세요 (aiofiles 사용).

### 11.2 중급 연습

4. 10개의 API 엔드포인트를 동시에 호출하되, 3초 타임아웃을 설정하고, 완료된 것만 반환하세요.

5. 데이터베이스에 1000개의 레코드를 삽입하되, 100개씩 배치로 나누어 비동기로 처리하세요.

6. 세마포어를 사용하여 동시에 최대 5개의 외부 API 호출만 허용하는 시스템을 구현하세요.

### 11.3 고급 연습

7. Redis를 사용한 비동기 캐싱 시스템을 구현하세요. 캐시 미스 시 데이터베이스 조회, 캐시 히트 시 즉시 반환.

8. WebSocket을 활용하여 비동기 작업의 진행 상황을 실시간으로 클라이언트에 전송하는 시스템을 구현하세요.

9. 여러 데이터 소스(DB, API, 파일)에서 데이터를 가져와 집계하는 완전한 비동기 파이프라인을 구현하세요.

## 12. 요약

### 핵심 포인트

1. **async/await**: Python의 비동기 프로그래밍 핵심
   - `async def`: 비동기 함수 정의
   - `await`: 비동기 작업 실행
   - I/O 바운드 작업에 적합

2. **동시성**: 여러 작업을 효율적으로 처리
   - `asyncio.gather()`: 여러 작업 동시 실행
   - `asyncio.create_task()`: 백그라운드 작업
   - `asyncio.wait()`: 타임아웃 관리

3. **비동기 라이브러리**:
   - SQLAlchemy (async): 비동기 DB 쿼리
   - httpx: 비동기 HTTP 클라이언트
   - aiofiles: 비동기 파일 처리

4. **성능 최적화**:
   - 연결 풀링
   - 캐싱
   - 세마포어로 동시 실행 제한

5. **모범 사례**:
   - I/O 작업은 async
   - 블로킹 호출 피하기
   - Context manager로 리소스 관리
   - 적절한 에러 처리

6. **Spring WebFlux 비교**:
   - FastAPI: async/await (간단)
   - Spring WebFlux: Reactor (복잡)
   - 둘 다 높은 동시성 처리

### 다음 챕터 예고

**Chapter 21: 테스팅 기초 (Testing Basics)**

다음 챕터에서는 FastAPI 애플리케이션 테스팅을 다룹니다:
- pytest와 TestClient 사용
- 단위 테스트와 통합 테스트
- 비동기 테스트 작성
- Mock과 Fixture
- 테스트 커버리지
- Spring Boot의 @SpringBootTest와 비교

비동기 프로그래밍을 마스터하면 고성능 API를 구축할 수 있습니다!
