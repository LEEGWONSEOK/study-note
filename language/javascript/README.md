# JavaScript 완전 정복 - Next.js를 위한 학습 가이드

**대상 독자**: Java Spring Backend 개발자 → Next.js 개발자로 전환
**목표**: JavaScript, TypeScript, React 마스터
**분량**: 책 한 권 분량

> 📌 **Note**: Next.js 프레임워크는 별도 문서로 관리됩니다.
> - 이 문서: JavaScript/TypeScript/React 언어 자체에 집중
> - 프레임워크: `framework/nextjs` 참고

---

## 📚 목차

### Part 1: JavaScript 기초
- [Chapter 1. JavaScript 소개](part01-fundamentals/01-javascript-소개.md)
  - JavaScript란?
  - Java vs JavaScript 비교
  - 개발 환경 설정 (Node.js, npm, VSCode)
  - 첫 JavaScript 프로그램
  - 실행 환경 (브라우저 vs Node.js)

- [Chapter 2. 기본 문법](part01-fundamentals/02-기본-문법.md)
  - 변수 (var, let, const)
  - 데이터 타입 (Primitive vs Reference)
  - 타입 변환 (명시적, 암묵적)
  - 연산자
  - 주석

- [Chapter 3. 제어문](part01-fundamentals/03-제어문.md)
  - if/else
  - switch
  - for, while, do-while
  - break, continue
  - label

- [Chapter 4. 함수 기초](part01-fundamentals/04-함수-기초.md)
  - 함수 선언
  - 함수 표현식
  - 매개변수와 인수
  - 반환값
  - 스코프
  - 호이스팅

---

### Part 2: 모던 JavaScript (ES6+)

- [Chapter 5. ES6+ 핵심 문법](part02-modern-js/05-es6-핵심.md)
  - let, const
  - 템플릿 리터럴
  - 화살표 함수
  - 단축 속성명
  - 계산된 속성명
  - 옵셔널 체이닝 (?.)
  - Nullish 병합 연산자 (??)

- [Chapter 6. 배열과 객체](part02-modern-js/06-배열과-객체.md)
  - 배열 메서드 (map, filter, reduce, forEach)
  - 구조 분해 할당
  - 스프레드 연산자
  - Rest 파라미터
  - 배열/객체 불변성

- [Chapter 7. 모듈 시스템](part02-modern-js/07-모듈-시스템.md)
  - ES6 모듈 (import/export)
  - CommonJS (require/module.exports)
  - Named vs Default Export
  - 동적 import
  - npm 패키지 관리

---

### Part 3: 비동기 프로그래밍

- [Chapter 8. 비동기 기초](part03-async/08-비동기-기초.md)
  - 동기 vs 비동기
  - 콜백 함수
  - 콜백 헬 (Callback Hell)
  - 이벤트 루프
  - 태스크 큐

- [Chapter 9. Promise](part03-async/09-promise.md)
  - Promise 생성
  - then, catch, finally
  - Promise 체이닝
  - Promise.all, Promise.race
  - Promise.allSettled, Promise.any

- [Chapter 10. async/await](part03-async/10-async-await.md)
  - async 함수
  - await 키워드
  - 에러 처리 (try/catch)
  - 병렬 처리
  - for await...of

---

### Part 4: 객체지향 JavaScript

- [Chapter 11. 객체와 프로토타입](part04-oop/11-객체와-프로토타입.md)
  - 객체 생성 방법
  - 프로토타입 체인
  - 상속
  - Object.create()
  - Property Descriptor

- [Chapter 12. 클래스](part04-oop/12-클래스.md)
  - class 문법
  - constructor
  - 메서드 (인스턴스, 정적, private)
  - getter/setter
  - 상속 (extends, super)
  - Java와의 비교

- [Chapter 13. this와 실행 컨텍스트](part04-oop/13-this와-실행-컨텍스트.md)
  - this 바인딩
  - call, apply, bind
  - 화살표 함수의 this
  - 렉시컬 환경
  - 클로저

---

### Part 5: 함수형 프로그래밍

- [Chapter 14. 함수형 개념](part05-functional/14-함수형-개념.md)
  - 일급 함수
  - 순수 함수
  - 불변성
  - 고차 함수
  - 커링 (Currying)

- [Chapter 15. 함수형 배열 메서드](part05-functional/15-함수형-배열-메서드.md)
  - map, filter, reduce 심화
  - flatMap
  - every, some
  - find, findIndex
  - 메서드 체이닝

- [Chapter 16. 고급 함수형 기법](part05-functional/16-고급-함수형-기법.md)
  - 함수 합성 (Composition)
  - 파이프 (Pipe)
  - 메모이제이션
  - 지연 평가
  - Ramda, Lodash 소개

---

### Part 6: DOM & Browser APIs

- [Chapter 17. DOM 조작](part06-dom-browser/17-dom-조작.md)
  - DOM이란?
  - 요소 선택 (querySelector, getElementById)
  - 요소 생성/삭제/수정
  - 스타일 조작
  - 클래스 조작

- [Chapter 18. 이벤트](part06-dom-browser/18-이벤트.md)
  - 이벤트 핸들러
  - 이벤트 객체
  - 이벤트 전파 (캡처링, 버블링)
  - 이벤트 위임
  - preventDefault, stopPropagation

- [Chapter 19. Browser APIs](part06-dom-browser/19-browser-apis.md)
  - Fetch API
  - LocalStorage, SessionStorage
  - Web Storage
  - Geolocation
  - History API

---

### Part 7: TypeScript

- [Chapter 20. TypeScript 시작하기](part07-typescript/20-typescript-시작하기.md)
  - TypeScript란?
  - 설치 및 설정
  - tsconfig.json
  - 컴파일
  - Java 개발자에게 TypeScript가 친숙한 이유

- [Chapter 21. 타입 시스템](part07-typescript/21-타입-시스템.md)
  - 기본 타입 (string, number, boolean)
  - 배열, 튜플
  - 객체 타입
  - any, unknown, never
  - 타입 추론
  - 타입 단언

- [Chapter 22. 고급 타입](part07-typescript/22-고급-타입.md)
  - Union, Intersection
  - Literal 타입
  - 타입 가드
  - 타입 별칭 (Type Alias)
  - 인터페이스 (Interface)
  - 제네릭 (Generics)

- [Chapter 23. TypeScript 실전](part07-typescript/23-typescript-실전.md)
  - 클래스 타입
  - 모듈 타입
  - 유틸리티 타입 (Partial, Pick, Omit, Record)
  - 데코레이터
  - 네임스페이스
  - React에서 TypeScript 사용

---

### Part 8: React 기초

- [Chapter 24. React 시작하기](part08-react-fundamentals/24-react-시작하기.md)
  - React란?
  - Virtual DOM
  - JSX
  - 프로젝트 생성 (Vite)
  - 컴포넌트 기초

- [Chapter 25. 컴포넌트와 Props](part08-react-fundamentals/25-컴포넌트와-props.md)
  - 함수형 컴포넌트
  - Props
  - Props 타입 검증
  - children
  - 컴포넌트 합성

- [Chapter 26. State와 생명주기](part08-react-fundamentals/26-state와-생명주기.md)
  - useState
  - 상태 업데이트
  - 불변성
  - useEffect
  - 생명주기
  - 정리 (Cleanup)

- [Chapter 27. 이벤트 처리](part08-react-fundamentals/27-이벤트-처리.md)
  - React 이벤트
  - 이벤트 핸들러
  - 합성 이벤트
  - 폼 다루기
  - 제어 컴포넌트

- [Chapter 28. 리스트와 Key](part08-react-fundamentals/28-리스트와-key.md)
  - 리스트 렌더링
  - Key의 중요성
  - map()으로 컴포넌트 생성
  - 조건부 렌더링

---

### Part 9: Advanced React

- [Chapter 29. Hooks 심화](part09-advanced-react/29-hooks-심화.md)
  - useReducer
  - useCallback
  - useMemo
  - useRef
  - 커스텀 Hook

- [Chapter 30. Context API](part09-advanced-react/30-context-api.md)
  - Context 생성
  - Provider, Consumer
  - useContext
  - Context 최적화
  - 전역 상태 관리

- [Chapter 31. 성능 최적화](part09-advanced-react/31-성능-최적화.md)
  - React.memo
  - 불필요한 렌더링 방지
  - Code Splitting
  - Lazy Loading
  - React DevTools

- [Chapter 32. 고급 패턴](part09-advanced-react/32-고급-패턴.md)
  - Compound Components
  - Render Props
  - HOC (Higher-Order Components)
  - Custom Hooks 패턴
  - Composition vs Inheritance

---

### Part 10: 부록

- [부록 A. 필수 도구](part10-appendix/부록A-필수-도구.md)
  - Node.js, npm, yarn, pnpm
  - Vite
  - Babel, Webpack
  - ESLint, Prettier
  - Git Hooks (Husky)

- [부록 B. 상태 관리 라이브러리](part10-appendix/부록B-상태-관리.md)
  - Redux Toolkit
  - Zustand
  - Recoil
  - Jotai
  - TanStack Query (React Query)

- [부록 C. 테스팅](part10-appendix/부록C-테스팅.md)
  - Jest
  - React Testing Library
  - Vitest
  - E2E 테스트 (Playwright, Cypress)

- [부록 D. 커뮤니티 리소스](part10-appendix/부록D-리소스.md)
  - 공식 문서
  - 추천 강의
  - 유용한 사이트
  - 블로그
  - GitHub 저장소

---

## 🎯 학습 로드맵

### 초급 (1-2주)
- Part 1-2: JavaScript 기초 + ES6+
- 변수, 함수, 배열, 객체, 모듈

### 중급 (2-3주)
- Part 3-5: 비동기, OOP, 함수형
- Promise, async/await, 클래스, 고차 함수

### TypeScript (1-2주)
- Part 7: TypeScript
- 타입 시스템, 제네릭, 유틸리티 타입

### React 입문 (2-3주)
- Part 8: React 기초
- 컴포넌트, Props, State, Hooks

### React 심화 (2-3주)
- Part 9: Advanced React
- 최적화, Context, 고급 패턴

### Next.js 준비 완료! (총 8-13주)
- `framework/nextjs`로 이동

---

## 🔥 Java 개발자를 위한 특별 섹션

각 챕터마다 포함된 내용:
- ✅ **Java vs JavaScript 비교**
- ✅ **실습 예제**
- ✅ **연습 문제 + 정답**
- ✅ **핵심 요약**
- ✅ **실전 팁**

---

## 📖 이 가이드의 특징

1. **Java 개발자 맞춤**
   - 모든 개념을 Java와 비교
   - 타입 시스템 차이 명확히 설명
   - Spring vs React 비교

2. **Next.js를 위한 최적화**
   - JavaScript → TypeScript → React 순서
   - 서버 사이드 개념 포함
   - 최신 문법 중심 (ES6+)

3. **실전 중심**
   - 실무에서 자주 쓰는 패턴
   - 모던 개발 도구
   - 테스팅, 성능 최적화

4. **최신 기술 스택**
   - ES2015+ (ES6+)
   - TypeScript 5.0+
   - React 18+
   - Node.js 20+

---

## 🚀 시작하기

[Part 1: JavaScript 기초 →](part01-fundamentals/01-javascript-소개.md)

---

**작성일**: 2024년
**JavaScript 버전**: ES2015+ (ES6+)
**TypeScript 버전**: 5.0+
**React 버전**: 18+
**대상**: Java Spring Backend 개발자