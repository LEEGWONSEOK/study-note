# FastAPI 완전 정복 - Spring Boot 개발자를 위한 가이드

**대상 독자**: Spring Boot Java 백엔드 개발자
**목표**: FastAPI로 프로덕션 레벨 REST API 개발
**분량**: 실전 중심의 완전 가이드

> 📌 **Note**: Python 기본 문법은 `language/python` 문서를 참고하세요.
> - 이 문서: FastAPI 프레임워크에 집중
> - Python 기초: `language/python` 참고

---

## 📚 목차

### Part 01: FastAPI 시작하기

- [Chapter 1. FastAPI 소개](part01-getting-started/chapter01-introduction.md)
  - FastAPI란?
  - Spring Boot vs FastAPI 비교
  - FastAPI의 특징과 장점
  - 설치 및 환경 설정 (Python, pip, venv)
  - 첫 FastAPI 애플리케이션

- [Chapter 2. 라우팅과 HTTP 메서드](part01-getting-started/chapter02-routing.md)
  - 경로 작성 (`@app.get`, `@app.post`, 등)
  - Spring의 `@RequestMapping` vs FastAPI 데코레이터
  - 경로 매개변수 (Path Parameters)
  - 쿼리 매개변수 (Query Parameters)
  - Request Body
  - Response 모델

- [Chapter 3. 자동 문서화](part01-getting-started/chapter03-documentation.md)
  - Swagger UI (Spring의 SpringDoc vs FastAPI)
  - ReDoc
  - OpenAPI 스키마
  - 문서 커스터마이징

---

### Part 02: 요청과 응답 처리

- [Chapter 4. Pydantic 모델](part02-request-response/chapter04-pydantic.md)
  - Pydantic이란? (Spring의 DTO와 비교)
  - 모델 정의 및 타입 힌트
  - 자동 검증 (Spring Validation vs Pydantic)
  - Field 설정 (기본값, 필수값, 제약조건)
  - 중첩 모델
  - Config 클래스

- [Chapter 5. 요청 데이터 검증](part02-request-response/chapter05-validation.md)
  - Query, Path, Body, Header, Cookie 파라미터
  - 검증 규칙 (문자열, 숫자, 날짜, 이메일 등)
  - 커스텀 검증기
  - 에러 응답 포맷
  - Spring의 `@Valid` vs FastAPI 검증

- [Chapter 6. 응답 모델과 상태 코드](part02-request-response/chapter06-response.md)
  - Response 모델 정의
  - 상태 코드 설정 (Spring의 `ResponseEntity` vs FastAPI)
  - 여러 응답 모델 (Union Types)
  - 응답 제외 필드 (`response_model_exclude`)
  - JSON 직렬화
  - File Upload/Download

---

### Part 03: 의존성 주입 (Dependency Injection)

- [Chapter 7. 의존성 주입 기초](part03-dependency-injection/chapter07-di-basics.md)
  - FastAPI의 `Depends` (Spring의 DI와 비교)
  - 함수형 의존성
  - 클래스형 의존성
  - 의존성 체인
  - 싱글톤 vs 프로토타입 스코프

- [Chapter 8. 고급 의존성 패턴](part03-dependency-injection/chapter08-advanced-di.md)
  - 서브 의존성 (Sub-dependencies)
  - 전역 의존성 (Global Dependencies)
  - 의존성 오버라이드 (테스트용)
  - `yield`를 사용한 컨텍스트 관리
  - Spring의 `@Bean` vs FastAPI 의존성

---

### Part 04: 데이터베이스 연동

- [Chapter 9. SQLAlchemy 기초](part04-database/chapter09-sqlalchemy-basics.md)
  - SQLAlchemy란? (Spring JPA vs SQLAlchemy)
  - ORM 모델 정의 (Entity와 비교)
  - 데이터베이스 연결 설정
  - 세션 관리 (Spring의 EntityManager vs Session)
  - CRUD 기본 작업

- [Chapter 10. 데이터베이스 CRUD 구현](part04-database/chapter10-crud.md)
  - Repository 패턴 (Spring Data JPA vs SQLAlchemy)
  - Create, Read, Update, Delete 작업
  - 쿼리 작성 (QuerySet vs SQLAlchemy Query)
  - 페이지네이션
  - 정렬 및 필터링

- [Chapter 11. 마이그레이션 (Alembic)](part04-database/chapter11-migrations.md)
  - Alembic 설치 및 설정 (Spring의 Flyway/Liquibase와 비교)
  - 마이그레이션 파일 생성
  - 마이그레이션 실행 및 롤백
  - 자동 마이그레이션
  - 버전 관리

- [Chapter 12. 고급 데이터베이스 기능](part04-database/chapter12-advanced-db.md)
  - Relationship (OneToMany, ManyToMany)
  - Lazy Loading vs Eager Loading
  - 트랜잭션 관리 (Spring의 `@Transactional`)
  - 데이터베이스 풀링
  - 다중 데이터베이스 연결

---

### Part 05: 인증과 보안

- [Chapter 13. JWT 인증](part05-security/chapter13-jwt.md)
  - JWT란? (Spring Security JWT와 비교)
  - Access Token & Refresh Token
  - 토큰 생성 및 검증
  - 비밀번호 해싱 (bcrypt)
  - 로그인/로그아웃 구현

- [Chapter 14. OAuth2 인증](part05-security/chapter14-oauth2.md)
  - OAuth2 개념 (Spring OAuth2와 비교)
  - Password Flow
  - Authorization Code Flow
  - 소셜 로그인 (Google, GitHub)
  - Scope와 권한 관리

- [Chapter 15. 권한과 역할 관리](part05-security/chapter15-authorization.md)
  - Role-based Access Control (Spring의 `@PreAuthorize` vs FastAPI)
  - 커스텀 권한 체크
  - 의존성 기반 권한 검증
  - User, Role, Permission 모델
  - 미들웨어를 통한 인증

- [Chapter 16. 보안 Best Practices](part05-security/chapter16-security-best-practices.md)
  - CORS 설정 (Spring의 CORS Configuration)
  - CSRF 방지
  - SQL Injection 방지
  - XSS 방지
  - Rate Limiting
  - 환경 변수 관리 (`.env`, `pydantic-settings`)

---

### Part 06: 미들웨어와 예외 처리

- [Chapter 17. 미들웨어](part06-middleware-exceptions/chapter17-middleware.md)
  - 미들웨어란? (Spring의 Filter/Interceptor와 비교)
  - 커스텀 미들웨어 작성
  - 요청/응답 로깅
  - 성능 측정
  - CORS 미들웨어
  - 내장 미들웨어

- [Chapter 18. 예외 처리](part06-middleware-exceptions/chapter18-exception-handling.md)
  - HTTPException (Spring의 `@ExceptionHandler` 비교)
  - 커스텀 예외 핸들러
  - 전역 예외 처리
  - 상태 코드별 처리
  - 에러 응답 표준화

---

### Part 07: 백그라운드 작업과 비동기

- [Chapter 19. 비동기 프로그래밍](part07-async-background/chapter19-async.md)
  - async/await 기초
  - FastAPI의 비동기 지원 (Spring WebFlux와 비교)
  - 비동기 라우트 핸들러
  - 비동기 데이터베이스 (SQLAlchemy Async)
  - 비동기 HTTP 클라이언트 (httpx)

- [Chapter 20. 백그라운드 태스크](part07-async-background/chapter20-background-tasks.md)
  - BackgroundTasks
  - 이메일 전송
  - 파일 처리
  - Celery 연동 (Spring의 `@Async` vs Celery)
  - Redis Queue

---

### Part 08: 테스팅

- [Chapter 21. 테스트 기초](part08-testing/chapter21-testing-basics.md)
  - pytest 설치 및 설정 (JUnit vs pytest)
  - TestClient 사용
  - 기본 API 테스트
  - 테스트 구조 및 네이밍
  - Fixtures

- [Chapter 22. 통합 테스트](part08-testing/chapter22-integration-testing.md)
  - 데이터베이스 테스트 (Spring의 `@DataJpaTest` 비교)
  - 테스트 데이터베이스 설정
  - 의존성 오버라이드
  - Mock과 Spy (Spring의 `@MockBean` vs pytest-mock)
  - 트랜잭션 롤백

- [Chapter 23. 테스트 커버리지와 CI/CD](part08-testing/chapter23-coverage-ci.md)
  - pytest-cov
  - 커버리지 리포트
  - GitHub Actions 연동
  - Docker 테스트 환경
  - 테스트 자동화

---

### Part 09: 프로젝트 구조와 패턴

- [Chapter 24. 프로젝트 구조](part09-architecture/chapter24-project-structure.md)
  - 계층형 아키텍처 (Spring과 비교)
  - 디렉토리 구조 Best Practice
  - 모듈 분리 (routers, models, schemas, services)
  - 설정 파일 관리
  - 대규모 프로젝트 구조

- [Chapter 25. 디자인 패턴](part09-architecture/chapter25-design-patterns.md)
  - Repository 패턴
  - Service 패턴
  - Factory 패턴
  - Strategy 패턴
  - Singleton 패턴 (FastAPI 컨텍스트)

---

### Part 10: 성능과 최적화

- [Chapter 26. 성능 최적화](part10-performance/chapter26-optimization.md)
  - 프로파일링
  - 데이터베이스 쿼리 최적화 (N+1 문제)
  - 캐싱 전략 (Redis)
  - Connection Pooling
  - 비동기 처리

- [Chapter 27. 모니터링과 로깅](part10-performance/chapter27-monitoring.md)
  - 구조화된 로깅 (Spring의 Logback vs Python logging)
  - 로그 레벨 및 포맷
  - Sentry 연동
  - Prometheus & Grafana
  - Health Check 엔드포인트

---

### Part 11: 배포

- [Chapter 28. Docker 배포](part11-deployment/chapter28-docker.md)
  - Dockerfile 작성
  - Docker Compose
  - Multi-stage Build
  - 환경별 설정 (dev, staging, prod)
  - Docker 이미지 최적화

- [Chapter 29. 프로덕션 배포](part11-deployment/chapter29-production.md)
  - Uvicorn vs Gunicorn (Spring Boot 내장 Tomcat과 비교)
  - NGINX 리버스 프록시
  - HTTPS 설정 (Let's Encrypt)
  - 무중단 배포
  - AWS/GCP/Azure 배포 가이드

- [Chapter 30. CI/CD 파이프라인](part11-deployment/chapter30-cicd.md)
  - GitHub Actions
  - 자동 테스트
  - 자동 배포
  - 롤백 전략
  - 환경 변수 관리

---

### Part 12: 실전 프로젝트

- [Chapter 31. 블로그 API 만들기](part12-real-project/chapter31-blog-api.md)
  - 요구사항 정의
  - DB 설계 (User, Post, Comment)
  - CRUD API 구현
  - 인증 및 권한
  - 페이지네이션 및 검색

- [Chapter 32. E-Commerce API 만들기](part12-real-project/chapter32-ecommerce-api.md)
  - 상품 관리 (Product, Category)
  - 장바구니 (Cart)
  - 주문 처리 (Order, Payment)
  - 재고 관리
  - 결제 연동

---

### 부록

- [부록 A. FastAPI vs Spring Boot 완전 비교](appendix/appendix-a-comparison.md)
  - 아키텍처 비교
  - 성능 벤치마크
  - 생태계 비교
  - 학습 곡선
  - 언제 무엇을 선택할까?

- [부록 B. 유용한 라이브러리](appendix/appendix-b-libraries.md)
  - Pydantic
  - SQLAlchemy
  - Alembic
  - httpx
  - python-jose (JWT)
  - passlib (해싱)
  - python-multipart
  - aioredis

- [부록 C. 실무 Tips](appendix/appendix-c-tips.md)
  - 코드 스타일 (Black, Flake8)
  - 타입 체크 (mypy)
  - 문서화 자동화
  - API 버저닝
  - 에러 처리 패턴
  - 성능 튜닝 체크리스트

- [부록 D. 리소스](appendix/appendix-d-resources.md)
  - 공식 문서
  - 추천 강의
  - GitHub 예제 프로젝트
  - 커뮤니티
  - 블로그/아티클

---

## 🎯 학습 로드맵

### 1주차: FastAPI 기초
- Part 01-02: FastAPI 시작, 요청/응답 처리
- Pydantic 모델, 검증
- **목표**: 간단한 CRUD API 작성

### 2주차: 핵심 기능
- Part 03-04: 의존성 주입, 데이터베이스
- SQLAlchemy ORM, 마이그레이션
- **목표**: DB 연동 REST API 작성

### 3주차: 인증과 보안
- Part 05-06: JWT, OAuth2, 미들웨어, 예외 처리
- **목표**: 인증이 있는 보안 API 작성

### 4주차: 고급 기능
- Part 07-08: 비동기, 백그라운드 작업, 테스팅
- **목표**: 비동기 API 및 테스트 작성

### 5주차: 아키텍처와 배포
- Part 09-11: 프로젝트 구조, 최적화, 배포
- **목표**: 프로덕션 레벨 API 배포

### 6주차 이후: 실전 프로젝트
- Part 12: 실전 프로젝트
- **목표**: 완성도 있는 API 서비스 개발

---

## 🔥 Spring Boot 개발자를 위한 특별 구성

각 챕터마다 포함된 내용:
- ✅ **Spring Boot vs FastAPI 비교표**
- ✅ **코드 예제 (양쪽 모두 제공)**
- ✅ **실습 프로젝트**
- ✅ **연습 문제 + 정답**
- ✅ **핵심 요약 (한 페이지)**
- ✅ **실전 팁 및 주의사항**

---

## 📖 이 가이드의 특징

### 1. **Spring Boot 개발자 맞춤형**
   - 모든 개념을 Spring Boot와 1:1 비교
   - `@RestController` ↔ `@app.get`
   - `@Autowired` ↔ `Depends`
   - JPA ↔ SQLAlchemy
   - Spring Security ↔ FastAPI Security

### 2. **실전 중심**
   - 프로덕션 레벨 코드
   - Best Practices
   - 보안 및 성능 고려
   - 실제 프로젝트 구조

### 3. **빠른 학습**
   - Spring Boot 지식 활용
   - 차이점 중심 설명
   - 실습 위주
   - 6주 완성 로드맵

### 4. **최신 기술 스택**
   - FastAPI 최신 버전
   - Python 3.10+ (Type Hints)
   - async/await
   - Pydantic v2
   - SQLAlchemy 2.0

---

## 🚀 시작하기

**필수 선행 지식**:
- ✅ Spring Boot 기본 (REST API 개발 경험)
- ✅ Python 기초 문법 (`language/python` 참고)
- ✅ HTTP, REST API 개념
- ✅ 데이터베이스 기초 (SQL)

**환경 설정**:
```bash
# Python 3.10+ 설치 확인
python --version

# 가상환경 생성
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# FastAPI 및 의존성 설치
pip install "fastapi[all]" uvicorn sqlalchemy alembic
```

---

[Part 01: FastAPI 시작하기 →](part01-getting-started/chapter01-introduction.md)

---

**작성일**: 2024년
**FastAPI 버전**: 0.110+
**Python 버전**: 3.10+
**대상**: Spring Boot Java 백엔드 개발자
