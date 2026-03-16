# Kotlin 학습 노트
> Java Spring 백엔드 개발자를 위한 Kotlin 완전 정복

## 목차

### Part 1. Kotlin 시작하기
#### Chapter 1. Kotlin 소개
- 1.1 Kotlin이란?
- 1.2 왜 Kotlin인가? (Java와의 비교)
- 1.3 Kotlin의 특징과 장점
- 1.4 개발 환경 설정 (IntelliJ IDEA)
- 1.5 첫 번째 Kotlin 프로그램

#### Chapter 2. 기본 문법
- 2.1 변수 선언 (val vs var)
- 2.2 기본 타입 (Int, String, Boolean 등)
- 2.3 타입 추론
- 2.4 문자열 템플릿
- 2.5 주석
- 2.6 Java와 비교: 기본 문법 차이점

---

### Part 2. Kotlin 기초 문법
#### Chapter 3. 제어 흐름
- 3.1 if 표현식 (expression vs statement)
- 3.2 when 표현식 (switch의 강력한 대안)
- 3.3 for 루프와 범위(Range)
- 3.4 while과 do-while
- 3.5 Java와 비교: 제어 흐름

#### Chapter 4. 함수
- 4.1 함수 선언과 호출
- 4.2 파라미터와 반환 타입
- 4.3 기본 파라미터 값
- 4.4 이름있는 인자 (Named Arguments)
- 4.5 단일 표현식 함수
- 4.6 확장 함수 (Extension Functions)
- 4.7 중위 함수 (Infix Functions)
- 4.8 Java와 비교: 함수 작성법

#### Chapter 5. 클래스와 객체 기초
- 5.1 클래스 선언
- 5.2 생성자 (주 생성자와 부 생성자)
- 5.3 프로퍼티 (Properties)
- 5.4 Getter와 Setter
- 5.5 데이터 클래스 (Data Class)
- 5.6 object 키워드 (싱글톤, 동반 객체)
- 5.7 Java와 비교: 클래스 작성법

---

### Part 3. Kotlin 핵심 개념
#### Chapter 6. Null 안전성
- 6.1 Nullable 타입 (?)
- 6.2 안전 호출 연산자 (?.)
- 6.3 Elvis 연산자 (?:)
- 6.4 !! 연산자 (Not-null assertion)
- 6.5 안전한 캐스팅 (as?)
- 6.6 let, run, apply 함수와 Null 처리
- 6.7 Java와 비교: Null 처리 방식

#### Chapter 7. 컬렉션
- 7.1 List, Set, Map
- 7.2 불변 컬렉션 vs 가변 컬렉션
- 7.3 컬렉션 생성과 초기화
- 7.4 컬렉션 연산 (filter, map, reduce 등)
- 7.5 Sequence를 통한 지연 평가
- 7.6 Java와 비교: 컬렉션 처리

#### Chapter 8. 람다와 고차 함수
- 8.1 람다 표현식 기초
- 8.2 고차 함수 (Higher-order Functions)
- 8.3 it 키워드
- 8.4 함수 타입
- 8.5 인라인 함수 (inline)
- 8.6 수신 객체 지정 람다
- 8.7 Java와 비교: 람다 사용법

---

### Part 4. 객체지향 프로그래밍
#### Chapter 9. 상속과 인터페이스
- 9.1 상속 (open, override)
- 9.2 추상 클래스
- 9.3 인터페이스
- 9.4 인터페이스의 기본 구현
- 9.5 다중 상속 문제 해결
- 9.6 Java와 비교: 상속 구조

#### Chapter 10. 가시성 제어자와 접근 제어
- 10.1 가시성 제어자 (private, protected, internal, public)
- 10.2 internal의 활용
- 10.3 Java와 비교: 접근 제어 차이점

#### Chapter 11. 특별한 클래스들
- 11.1 Enum 클래스
- 11.2 Sealed 클래스 (봉인 클래스)
- 11.3 중첩 클래스와 내부 클래스
- 11.4 인라인 클래스 (Value Class)
- 11.5 Java와 비교: 특수 클래스 활용

---

### Part 5. 고급 기능
#### Chapter 12. 제네릭
- 12.1 제네릭 기초
- 12.2 제네릭 함수와 클래스
- 12.3 변성 (Variance): in, out
- 12.4 타입 소거와 reified
- 12.5 제네릭 제약
- 12.6 Java와 비교: 제네릭 처리

#### Chapter 13. 델리게이션
- 13.1 클래스 델리게이션 (by)
- 13.2 프로퍼티 델리게이션
- 13.3 lazy 델리게이션
- 13.4 observable 델리게이션
- 13.5 커스텀 델리게이션 만들기
- 13.6 Java와 비교: 위임 패턴

#### Chapter 14. 연산자 오버로딩
- 14.1 연산자 오버로딩 기초
- 14.2 산술 연산자
- 14.3 비교 연산자
- 14.4 인덱스 접근 연산자
- 14.5 invoke 연산자
- 14.6 Java와 비교: 연산자 처리

---

### Part 6. 함수형 프로그래밍
#### Chapter 15. 함수형 프로그래밍 개념
- 15.1 함수형 프로그래밍이란?
- 15.2 불변성 (Immutability)
- 15.3 순수 함수 (Pure Functions)
- 15.4 1급 함수 (First-class Functions)
- 15.5 Java와 비교: 함수형 접근법

#### Chapter 16. 스코프 함수
- 16.1 let
- 16.2 run
- 16.3 with
- 16.4 apply
- 16.5 also
- 16.6 스코프 함수 선택 가이드
- 16.7 Java와 비교: 빌더 패턴 대체

#### Chapter 17. 고급 컬렉션 연산
- 17.1 변환 연산 (map, flatMap)
- 17.2 필터링 (filter, filterNot, partition)
- 17.3 집계 연산 (reduce, fold)
- 17.4 그룹화와 분할 (groupBy, chunked)
- 17.5 정렬 (sortedBy, sortedWith)
- 17.6 Java Stream API와 비교

---

### Part 7. 코루틴과 비동기 프로그래밍
#### Chapter 18. 코루틴 기초
- 18.1 코루틴이란?
- 18.2 코루틴과 스레드의 차이
- 18.3 첫 번째 코루틴 (launch, runBlocking)
- 18.4 suspend 함수
- 18.5 코루틴 스코프
- 18.6 Java와 비교: 비동기 처리 방식

#### Chapter 19. 코루틴 심화
- 19.1 Job과 취소
- 19.2 async와 await
- 19.3 Dispatcher (IO, Default, Main)
- 19.4 예외 처리
- 19.5 Channel
- 19.6 Flow
- 19.7 Java CompletableFuture와 비교

---

### Part 8. Kotlin과 Spring Boot
#### Chapter 20. Spring Boot 프로젝트 설정
- 20.1 Spring Initializr로 Kotlin 프로젝트 생성
- 20.2 Gradle/Maven 설정
- 20.3 필수 플러그인 (kotlin-spring, kotlin-jpa)
- 20.4 프로젝트 구조
- 20.5 Java Spring과 비교: 설정 차이점

#### Chapter 21. Spring과 Kotlin 통합
- 21.1 Bean 정의와 DI
- 21.2 Controller 작성
- 21.3 Service 계층
- 21.4 Configuration 클래스
- 21.5 데이터 클래스를 활용한 DTO
- 21.6 Java Spring과 비교: 코드 간결성

#### Chapter 22. Spring Data JPA with Kotlin
- 22.1 Entity 클래스 작성
- 22.2 data class를 Entity로 사용할 때 주의점
- 22.3 Repository 인터페이스
- 22.4 @Query와 JPQL
- 22.5 지연 로딩과 프록시 문제
- 22.6 all-open, no-arg 플러그인
- 22.7 Java JPA와 비교: Entity 작성법

#### Chapter 23. REST API 개발
- 23.1 @RestController
- 23.2 요청 매핑 (@GetMapping, @PostMapping 등)
- 23.3 @RequestBody와 @ResponseBody
- 23.4 Path Variable과 Query Parameter
- 23.5 예외 처리 (@ControllerAdvice)
- 23.6 Validation
- 23.7 Java Spring과 비교: API 코드

#### Chapter 24. 코루틴과 Spring WebFlux
- 24.1 Spring WebFlux 소개
- 24.2 코루틴을 활용한 비동기 API
- 24.3 suspend 함수와 Controller
- 24.4 R2DBC를 통한 비동기 DB 접근
- 24.5 Flow를 활용한 스트리밍
- 24.6 Java Reactive와 비교

---

### Part 9. 실전 활용과 Best Practices
#### Chapter 25. Kotlin 코딩 컨벤션
- 25.1 네이밍 규칙
- 25.2 코드 구조화
- 25.3 관용적 Kotlin 코드 작성법
- 25.4 안티 패턴 피하기
- 25.5 Java에서 Kotlin으로 마이그레이션 팁

#### Chapter 26. 테스트
- 26.1 JUnit 5와 Kotlin
- 26.2 Kotest 프레임워크
- 26.3 MockK를 활용한 Mocking
- 26.4 Spring Boot Test
- 26.5 코루틴 테스트
- 26.6 Java 테스트 코드와 비교

#### Chapter 27. 실무 패턴과 구조
- 27.1 Result 타입을 활용한 예외 처리
- 27.2 sealed class를 활용한 상태 관리
- 27.3 DSL 만들기
- 27.4 확장 함수로 유틸리티 구성
- 27.5 스마트 캐스팅 활용
- 27.6 실무에서 자주 쓰는 패턴

#### Chapter 28. 성능과 최적화
- 28.1 인라인 함수의 성능 영향
- 28.2 컬렉션 vs Sequence 성능
- 28.3 불필요한 객체 생성 줄이기
- 28.4 코루틴 최적화
- 28.5 프로파일링과 디버깅

---

### Part 10. 부록
#### Appendix A. Java to Kotlin 변환 가이드
- A.1 자동 변환 도구 활용
- A.2 수동 변환 체크리스트
- A.3 주요 변환 패턴
- A.4 변환 시 주의사항

#### Appendix B. Kotlin 표준 라이브러리 참고
- B.1 자주 사용하는 함수들
- B.2 유용한 확장 함수
- B.3 컬렉션 함수 치트시트

#### Appendix C. 추가 학습 자료
- C.1 공식 문서와 레퍼런스
- C.2 추천 도서
- C.3 온라인 강의
- C.4 커뮤니티와 리소스

---

## 학습 가이드

### 추천 학습 순서
1. **1주차**: Part 1-2 (Kotlin 기본 문법)
2. **2주차**: Part 3-4 (핵심 개념과 OOP)
3. **3주차**: Part 5-6 (고급 기능과 함수형 프로그래밍)
4. **4주차**: Part 7 (코루틴)
5. **5주차**: Part 8 (Spring Boot 통합)
6. **6주차**: Part 9 (실전 활용)

### 각 챕터 구성
- **개념 설명**: 해당 주제의 핵심 개념
- **Java와 비교**: Java 개발자 관점에서의 차이점
- **코드 예제**: 실제 동작하는 예제 코드
- **실전 팁**: 실무에서 활용할 수 있는 팁
- **연습 문제**: (선택) 학습 내용 확인

---

## 작성 예정

이 노트는 순차적으로 작성될 예정입니다. 각 챕터는 다음과 같은 형식으로 작성됩니다:
- 명확한 개념 설명
- Java 코드와 Kotlin 코드의 비교
- 실제 동작하는 예제
- 실무 활용 팁

---

**시작일**: 2026-03-16
**목표**: Java Spring 백엔드 개발자가 Kotlin을 완전히 마스터하기