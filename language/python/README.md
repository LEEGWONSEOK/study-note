# Python 완전 정복 - Java 개발자를 위한 가이드

**대상 독자**: 기존 Java Spring Backend 개발자
**목표**: Python 언어 마스터 (코어 + 실전)
**분량**: 책 한 권 분량

> 📌 **Note**: Django/FastAPI 같은 웹 프레임워크는 별도 문서로 관리됩니다.
> - 이 문서: Python 언어 자체에 집중
> - 프레임워크: `framework/django`, `framework/fastapi` 참고

---

## 📚 목차

### Part 1: Python 시작하기
- [Chapter 1. Python 소개](part1-getting-started/chapter1-introduction.md)
  - Python이란?
  - Java vs Python 비교
  - 개발 환경 설정 (Python, pip, venv)
  - 첫 Python 프로그램

- [Chapter 2. 기본 문법](part1-getting-started/chapter2-basic-syntax.md)
  - 변수와 타입
  - 들여쓰기 (Indentation)
  - 주석과 Docstring
  - 입출력

---

### Part 2: 기본 문법

- [Chapter 3. 데이터 타입](part2-basic-syntax/chapter3-data-types.md)
  - 숫자 (int, float, complex)
  - 문자열 (str)
  - 불린 (bool)
  - None 타입
  - 타입 변환

- [Chapter 4. 컬렉션](part2-basic-syntax/chapter4-collections.md)
  - 리스트 (List)
  - 튜플 (Tuple)
  - 딕셔너리 (Dict)
  - 셋 (Set)
  - 컬렉션 연산

- [Chapter 5. 제어 흐름](part2-basic-syntax/chapter5-control-flow.md)
  - if/elif/else
  - for 루프
  - while 루프
  - break, continue, pass
  - match-case (Python 3.10+)

---

### Part 3: 함수와 모듈

- [Chapter 6. 함수](part3-functions-modules/chapter6-functions.md)
  - 함수 정의
  - 매개변수 (위치, 키워드, 기본값, 가변)
  - 반환값
  - Lambda 함수
  - 데코레이터

- [Chapter 7. 모듈과 패키지](part3-functions-modules/chapter7-modules-packages.md)
  - 모듈 import
  - 패키지 구조
  - `__init__.py`
  - 표준 라이브러리
  - pip와 패키지 관리

- [Chapter 8. 파일 처리](part3-functions-modules/chapter8-file-handling.md)
  - 파일 읽기/쓰기
  - with 문
  - CSV, JSON 처리
  - 경로 처리 (pathlib)

---

### Part 4: 객체지향 프로그래밍

- [Chapter 9. 클래스 기초](part4-oop/chapter9-classes-basics.md)
  - 클래스와 객체
  - `__init__` 생성자
  - 인스턴스 변수 vs 클래스 변수
  - 메서드 (인스턴스, 클래스, 정적)

- [Chapter 10. 상속과 다형성](part4-oop/chapter10-inheritance.md)
  - 상속 (Inheritance)
  - 메서드 오버라이딩
  - super()
  - 다중 상속 (MRO)
  - 추상 클래스 (ABC)

- [Chapter 11. 특수 메서드](part4-oop/chapter11-special-methods.md)
  - `__str__`, `__repr__`
  - `__len__`, `__getitem__`
  - `__eq__`, `__lt__` (비교 연산)
  - `__enter__`, `__exit__` (컨텍스트 매니저)
  - 연산자 오버로딩

---

### Part 5: 고급 기능

- [Chapter 12. 예외 처리](part5-advanced/chapter12-exception-handling.md)
  - try/except/finally
  - 예외 타입
  - raise
  - 커스텀 예외
  - 예외 체이닝

- [Chapter 13. 이터레이터와 제너레이터](part5-advanced/chapter13-iterators-generators.md)
  - 이터레이터 프로토콜
  - `__iter__`, `__next__`
  - 제너레이터 (yield)
  - 제너레이터 표현식
  - itertools

- [Chapter 14. 컴프리헨션](part5-advanced/chapter14-comprehensions.md)
  - 리스트 컴프리헨션
  - 딕셔너리 컴프리헨션
  - 셋 컴프리헨션
  - 제너레이터 표현식
  - Walrus 연산자 (`:=`)

---

### Part 6: 함수형 프로그래밍

- [Chapter 15. 함수형 개념](part6-functional/chapter15-functional-concepts.md)
  - 일급 함수 (First-class Function)
  - 순수 함수
  - map, filter, reduce
  - functools
  - partial, lru_cache

- [Chapter 16. 데코레이터 심화](part6-functional/chapter16-decorators.md)
  - 함수 데코레이터
  - 클래스 데코레이터
  - 매개변수가 있는 데코레이터
  - functools.wraps
  - 실전 예제 (로깅, 타이밍, 캐싱)

- [Chapter 17. 타입 힌트](part6-functional/chapter17-type-hints.md)
  - 타입 힌트 기초
  - typing 모듈
  - Generic, TypeVar
  - Protocol
  - mypy로 타입 체크

---

### Part 7: 동시성 프로그래밍

- [Chapter 18. 멀티스레딩](part7-concurrency/chapter18-multithreading.md)
  - threading 모듈
  - Thread 클래스
  - Lock, RLock
  - GIL (Global Interpreter Lock)
  - 스레드 풀

- [Chapter 19. 멀티프로세싱](part7-concurrency/chapter19-multiprocessing.md)
  - multiprocessing 모듈
  - Process 클래스
  - Pool
  - Queue, Pipe
  - 프로세스 간 통신

- [Chapter 20. 비동기 프로그래밍 (asyncio)](part7-concurrency/chapter20-asyncio.md)
  - async/await
  - 코루틴
  - asyncio.run()
  - Task, gather
  - aiohttp (비동기 HTTP)

---

### Part 8: 웹 개발 - Django

- [Chapter 21. Django 시작하기](part8-django/chapter21-django-setup.md)
  - Django란?
  - 프로젝트 생성
  - 앱 구조
  - settings.py 설정
  - 개발 서버 실행

- [Chapter 22. Models (ORM)](part8-django/chapter22-models.md)
  - Model 정의
  - Field 타입
  - Relationships (ForeignKey, ManyToMany)
  - Migration
  - QuerySet API

- [Chapter 23. Views와 URLconf](part8-django/chapter23-views-urls.md)
  - Function-based Views
  - Class-based Views
  - URL 패턴
  - URL 파라미터
  - 템플릿 렌더링

- [Chapter 24. Django REST Framework](part8-django/chapter24-drf.md)
  - DRF 설치
  - Serializers
  - ViewSet
  - Router
  - Authentication & Permissions

---

### Part 9: 웹 개발 - FastAPI

- [Chapter 25. FastAPI 시작하기](part9-fastapi/chapter25-fastapi-setup.md)
  - FastAPI란?
  - 첫 API 작성
  - 경로 매개변수
  - 쿼리 매개변수
  - Request Body

- [Chapter 26. Pydantic과 Validation](part9-fastapi/chapter26-pydantic.md)
  - Pydantic 모델
  - 자동 검증
  - Response 모델
  - Config
  - Field 설정

- [Chapter 27. 데이터베이스 연동](part9-fastapi/chapter27-database.md)
  - SQLAlchemy
  - Async SQLAlchemy
  - Migration (Alembic)
  - CRUD 구현

- [Chapter 28. 인증과 보안](part9-fastapi/chapter28-security.md)
  - JWT 인증
  - OAuth2
  - 비밀번호 해싱
  - CORS
  - 환경 변수

---

### Part 10: 실전 활용

- [Chapter 29. 테스팅](part10-practice/chapter29-testing.md)
  - unittest
  - pytest
  - Mock
  - 테스트 커버리지
  - TDD

- [Chapter 30. 로깅과 디버깅](part10-practice/chapter30-logging-debugging.md)
  - logging 모듈
  - 로그 레벨
  - 핸들러, 포매터
  - pdb 디버거
  - 프로파일링

- [Chapter 31. 패키지 배포](part10-practice/chapter31-packaging.md)
  - setup.py, pyproject.toml
  - Poetry
  - PyPI 배포
  - Docker 이미지
  - 가상 환경 관리

- [Chapter 32. 성능 최적화](part10-practice/chapter32-optimization.md)
  - 프로파일링 도구
  - 메모리 최적화
  - Cython
  - NumPy 활용
  - 병렬 처리

---

### Part 11: 부록

- [부록 A. 유용한 표준 라이브러리](part11-appendix/appendix-a-stdlib.md)
  - collections
  - itertools
  - functools
  - datetime
  - json, csv
  - pathlib
  - re (정규표현식)

- [부록 B. 필수 서드파티 라이브러리](part11-appendix/appendix-b-libraries.md)
  - requests
  - pandas
  - numpy
  - SQLAlchemy
  - celery
  - pytest
  - black, flake8

- [부록 C. 커뮤니티 리소스](part11-appendix/appendix-c-resources.md)
  - 공식 문서
  - 추천 도서
  - 온라인 강의
  - 커뮤니티
  - 블로그

---

## 🎯 학습 로드맵

### 초급 (1-2주)
- Part 1-2: Python 기초 문법
- 변수, 타입, 컬렉션, 제어문

### 중급 (2-3주)
- Part 3-5: 함수, 모듈, OOP, 고급 기능
- 제너레이터, 데코레이터, 예외 처리

### 중고급 (2-3주)
- Part 6-7: 함수형, 타입 힌트, 비동기
- asyncio, 멀티스레딩

### 실전 (4-6주)
- Part 8-10: Django/FastAPI, 테스팅, 배포
- REST API 개발, 데이터베이스 연동

---

## 🔥 Java 개발자를 위한 특별 섹션

각 챕터마다 포함된 내용:
- ✅ **Java vs Python 비교**
- ✅ **실습 예제**
- ✅ **연습 문제 + 정답**
- ✅ **핵심 요약**
- ✅ **실전 팁**

---

## 📖 이 책의 특징

1. **Java 개발자 맞춤**
   - 모든 개념을 Java와 비교
   - Spring과 Django/FastAPI 비교
   - 타입 시스템 차이 설명

2. **실전 중심**
   - REST API 개발
   - 데이터베이스 연동
   - 인증/인가
   - 배포

3. **최신 Python**
   - Python 3.10+ 기준
   - Type Hints
   - asyncio
   - FastAPI

---

## 🚀 시작하기

[Part 1: Python 시작하기 →](part1-getting-started/chapter1-introduction.md)

---

**작성일**: 2024년
**Python 버전**: 3.10+
**대상**: Java Spring Backend 개발자