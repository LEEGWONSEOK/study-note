# Chapter 1. FastAPI 소개

## 1.1 FastAPI란?

FastAPI는 Python 3.7+의 타입 힌트를 기반으로 만들어진 **현대적이고 빠른 웹 프레임워크**입니다. API 개발에 특화되어 있으며, 자동 문서화, 데이터 검증, 직렬화 등을 기본으로 제공합니다.

### 핵심 특징

1. **빠른 성능**: NodeJS, Go와 비슷한 수준의 고성능 (Starlette 기반)
2. **빠른 개발**: 개발 속도를 약 200~300% 향상
3. **적은 버그**: 개발자 실수로 인한 버그를 약 40% 감소
4. **직관적**: 뛰어난 에디터 지원 (자동완성)
5. **짧은 코드**: 코드 중복 최소화
6. **표준 기반**: OpenAPI, JSON Schema 기반

---

## 1.2 Spring Boot vs FastAPI 비교

Spring Boot 개발자 관점에서 FastAPI의 특징을 비교해보겠습니다.

### 비교표

| 특징 | Spring Boot | FastAPI |
|------|-------------|---------|
| **언어** | Java | Python |
| **타입 시스템** | 정적 타입 (컴파일 타임) | 동적 타입 + 타입 힌트 (런타임) |
| **성능** | 높음 (JVM) | 매우 높음 (비동기 I/O) |
| **학습 곡선** | 중간~높음 | 낮음~중간 |
| **개발 속도** | 중간 | 빠름 |
| **자동 문서화** | SpringDoc (별도 설정) | 기본 내장 (Swagger UI) |
| **데이터 검증** | Bean Validation (애노테이션) | Pydantic (타입 힌트) |
| **의존성 주입** | Spring Container (@Autowired) | Depends 함수 |
| **ORM** | JPA/Hibernate | SQLAlchemy |
| **비동기 지원** | WebFlux (별도 스택) | 기본 지원 (async/await) |
| **배포** | JAR/WAR | Docker/Uvicorn |

### 주요 차이점

#### 1. **개발 철학**
```java
// Spring Boot: 애노테이션 기반, 설정보다 관습(Convention over Configuration)
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}
```

```python
# FastAPI: 타입 힌트 기반, 함수형 스타일
from fastapi import FastAPI, Depends

app = FastAPI()

@app.get("/api/users/{id}")
async def get_user(id: int, service: UserService = Depends()):
    return await service.find_by_id(id)
```

#### 2. **타입 시스템**
- **Spring Boot**: 컴파일 타임에 타입 체크 (강력한 타입 안전성)
- **FastAPI**: 런타임에 Pydantic을 통해 검증 (유연성 + 자동 검증)

#### 3. **비동기 처리**
- **Spring Boot**: WebFlux는 별도 스택 (Reactor 기반, 러닝커브 높음)
- **FastAPI**: `async/await`로 간단하게 비동기 처리

---

## 1.3 FastAPI를 선택해야 하는 경우

### FastAPI가 적합한 상황

✅ **빠른 프로토타이핑이 필요할 때**
- MVP 개발
- 스타트업 초기 제품
- 간단한 마이크로서비스

✅ **비동기 I/O가 중요할 때**
- 실시간 데이터 처리
- WebSocket 통신
- 대량의 외부 API 호출

✅ **AI/ML 모델 서빙**
- Python 생태계 활용 (TensorFlow, PyTorch)
- 데이터 분석 파이프라인 연동

✅ **작은 팀, 빠른 개발 사이클**
- 코드량 적음
- 학습 곡선 낮음

### Spring Boot가 더 적합한 상황

✅ **대규모 엔터프라이즈 애플리케이션**
- 복잡한 비즈니스 로직
- 트랜잭션 처리 중심

✅ **강력한 타입 안전성이 필요할 때**
- 컴파일 타임 에러 검출
- 대규모 팀 협업

✅ **Java 생태계 의존성**
- 기존 Java 라이브러리 활용
- 레거시 시스템 통합

---

## 1.4 설치 및 환경 설정

### Python 설치 확인

```bash
# Python 3.10 이상 필요
python --version
# 또는
python3 --version
```

### 가상환경 생성 (권장)

**Spring Boot의 Gradle/Maven과 유사한 개념**

```bash
# 가상환경 생성
python -m venv venv

# 활성화
# macOS/Linux
source venv/bin/activate

# Windows
venv\Scripts\activate

# 비활성화
deactivate
```

### FastAPI 설치

```bash
# FastAPI + 모든 의존성 설치
pip install "fastapi[all]"

# 또는 최소 설치
pip install fastapi uvicorn

# 필수 패키지
pip install pydantic python-multipart
```

**설치된 패키지 확인**
```bash
pip list
```

**의존성 고정 (requirements.txt)**
```bash
# Spring Boot의 build.gradle과 유사
pip freeze > requirements.txt

# 설치
pip install -r requirements.txt
```

---

## 1.5 첫 FastAPI 애플리케이션

### Hello World

**main.py 생성**
```python
from fastapi import FastAPI

# FastAPI 인스턴스 생성 (Spring Boot의 @SpringBootApplication과 유사)
app = FastAPI()

# 라우트 정의 (Spring의 @GetMapping과 유사)
@app.get("/")
def read_root():
    return {"message": "Hello World"}

@app.get("/hello/{name}")
def hello(name: str):
    return {"message": f"Hello, {name}!"}
```

### 애플리케이션 실행

```bash
# Uvicorn 서버 실행 (Spring Boot의 내장 Tomcat과 유사)
uvicorn main:app --reload

# 옵션 설명:
# main: 파일명 (main.py)
# app: FastAPI 인스턴스 변수명
# --reload: 코드 변경 시 자동 재시작 (개발용)
```

**Spring Boot와 비교**
```java
// Spring Boot
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
// 실행: ./gradlew bootRun 또는 java -jar app.jar
```

```python
# FastAPI
# main.py에 코드 작성
# 실행: uvicorn main:app --reload
```

### 서버 접속

```bash
# API 엔드포인트
http://localhost:8000/

# 자동 생성된 Swagger UI (대박 기능!)
http://localhost:8000/docs

# ReDoc (대안 문서)
http://localhost:8000/redoc

# OpenAPI 스키마 (JSON)
http://localhost:8000/openapi.json
```

---

## 1.6 Spring Boot vs FastAPI 코드 비교

### 예제: 간단한 User API

#### Spring Boot 방식

```java
// User.java
public class User {
    private Long id;
    private String name;
    private String email;

    // Getters, Setters, Constructor
}

// UserController.java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = new User(id, "John", "john@example.com");
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        // 저장 로직
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}
```

#### FastAPI 방식

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# Pydantic 모델 (Spring의 DTO와 유사)
class User(BaseModel):
    id: int
    name: str
    email: str

@app.get("/api/users/{id}")
async def get_user(id: int) -> User:
    return User(id=id, name="John", email="john@example.com")

@app.post("/api/users", status_code=201)
async def create_user(user: User) -> User:
    # 저장 로직
    return user
```

### 주요 차이점 분석

1. **코드량**: FastAPI가 훨씬 간결
2. **타입 힌트**: FastAPI는 타입 힌트로 자동 검증
3. **문서화**: FastAPI는 별도 설정 없이 Swagger UI 제공
4. **비동기**: `async/await` 키워드로 간단하게 비동기 처리

---

## 1.7 자동 문서화의 위력

FastAPI의 가장 강력한 기능 중 하나는 **자동 생성되는 대화형 API 문서**입니다.

### Swagger UI 접속

```
http://localhost:8000/docs
```

**제공되는 기능:**
- 📖 모든 엔드포인트 자동 문서화
- 🧪 브라우저에서 직접 API 테스트 가능
- 📝 Request/Response 스키마 확인
- 🔍 타입 정보 자동 표시

**Spring Boot에서는:**
```xml
<!-- pom.xml에 의존성 추가 필요 -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
</dependency>
```

**FastAPI에서는:**
- 설치만 하면 끝! (추가 설정 불필요)

---

## 1.8 개발 환경 설정 (VS Code 권장)

### VS Code 확장 프로그램

1. **Python** (Microsoft)
2. **Pylance** (타입 체킹)
3. **autoDocstring** (docstring 자동 생성)
4. **REST Client** (HTTP 테스트)

### settings.json 설정

```json
{
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "python.analysis.typeCheckingMode": "basic"
}
```

### 프로젝트 구조 (초기)

```
fastapi-project/
├── venv/                 # 가상환경
├── main.py              # 애플리케이션 진입점
├── requirements.txt     # 의존성 목록
└── .gitignore
```

**.gitignore**
```
venv/
__pycache__/
*.pyc
.env
.pytest_cache/
```

---

## 1.9 핵심 요약

### FastAPI의 핵심 개념

| 개념 | Spring Boot 비교 | 설명 |
|------|------------------|------|
| `FastAPI()` | `@SpringBootApplication` | 앱 인스턴스 생성 |
| `@app.get()` | `@GetMapping` | GET 엔드포인트 |
| `Pydantic Model` | DTO + `@Valid` | 데이터 검증 모델 |
| `uvicorn` | 내장 Tomcat | ASGI 서버 |
| `/docs` | SpringDoc UI | 자동 Swagger 문서 |

### Spring Boot 개발자가 기억할 것

1. ✅ **타입 힌트**: Python의 타입 힌트 = Java의 타입 시스템 (런타임 검증)
2. ✅ **함수형**: 클래스보다 함수 중심 (더 간결)
3. ✅ **비동기**: `async/await`로 간단하게 비동기 처리
4. ✅ **Pydantic**: DTO + Validation이 하나로 합쳐짐
5. ✅ **문서 자동화**: 추가 설정 없이 Swagger UI 제공

---

## 1.10 실습 예제

### 실습 1: 첫 FastAPI 앱 만들기

**요구사항:**
- `/` : "Welcome to FastAPI" 반환
- `/api/health` : `{"status": "ok"}` 반환
- `/api/info` : 앱 이름, 버전 반환

**정답:**
```python
from fastapi import FastAPI

app = FastAPI(title="My First FastAPI", version="1.0.0")

@app.get("/")
def root():
    return {"message": "Welcome to FastAPI"}

@app.get("/api/health")
def health_check():
    return {"status": "ok"}

@app.get("/api/info")
def info():
    return {
        "name": "My First FastAPI",
        "version": "1.0.0"
    }
```

### 실습 2: 경로 매개변수

**요구사항:**
- `/greet/{name}` : "Hello, {name}!" 반환
- `/square/{number}` : 숫자의 제곱 반환

**정답:**
```python
@app.get("/greet/{name}")
def greet(name: str):
    return {"message": f"Hello, {name}!"}

@app.get("/square/{number}")
def square(number: int):
    return {"result": number ** 2}
```

---

## 1.11 연습 문제

### 문제 1
Spring Boot의 `@RestController`와 FastAPI의 `@app.get()`의 차이점을 3가지 이상 설명하세요.

### 문제 2
다음 요구사항을 만족하는 API를 작성하세요:
- 엔드포인트: `/api/calculate/{operation}/{a}/{b}`
- operation: add, subtract, multiply, divide
- a, b: 정수
- 결과 반환

**힌트:**
```python
@app.get("/api/calculate/{operation}/{a}/{b}")
def calculate(operation: str, a: int, b: int):
    # 여기에 코드 작성
    pass
```

### 문제 3
FastAPI의 자동 문서화가 Spring Boot보다 유리한 점을 5가지 이상 나열하세요.

---

## 1.12 다음 챕터 예고

다음 챕터에서는 **라우팅과 HTTP 메서드**를 다룹니다:
- 경로 매개변수 (Path Parameters)
- 쿼리 매개변수 (Query Parameters)
- Request Body
- Response 모델
- HTTP 메서드 (GET, POST, PUT, DELETE, PATCH)

**Spring의 `@RequestMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody`에 대응하는 FastAPI 기능들을 배웁니다!**

---

[← 목차로](../README.md) | [다음: Chapter 2. 라우팅과 HTTP 메서드 →](chapter02-routing.md)