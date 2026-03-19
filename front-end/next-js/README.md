# Next.js 완전 정복 - 프론트엔드 프레임워크 가이드

**목표**: Next.js로 프로덕션 레벨 웹 애플리케이션 개발
**분량**: 실전 중심의 완전 가이드
**특징**: 프론트엔드 프레임워크로서의 Next.js 학습

> 📌 **Note**: JavaScript, TypeScript, React 기초는 `language/javascript` 문서를 먼저 공부하세요.
> - 이 문서: Next.js 프레임워크에 집중
> - 선수 지식: `language/javascript` 참고 (Part 01-09)
>
> 💡 **Next.js란?**: React 기반 **프론트엔드 프레임워크**
> - SSR/SSG를 통한 SEO 최적화와 성능 향상
> - 풀스택 기능(API Routes)도 포함하지만, 주로 프론트엔드로 사용됨
> - 대부분의 프로젝트는 Next.js(프론트) + 별도 백엔드 서버 구조

---

## 🎯 학습 전 체크리스트

### 필수 선수 지식 (반드시 먼저 공부!)
- ✅ JavaScript 기초 (`language/javascript/part01-02`)
- ✅ 비동기 프로그래밍 (`language/javascript/part03`)
- ✅ TypeScript 기초 (`language/javascript/part07`)
- ✅ React 기초 (`language/javascript/part08-09`)

### 이 문서에서 배울 것
- ✅ Next.js App Router (Next.js 13+)
- ✅ Server Components vs Client Components
- ✅ File-based Routing
- ✅ 렌더링 전략 (SSR, SSG, ISR)
- ✅ 스타일링과 UI 구성
- ✅ 인증/인가
- ✅ 데이터 페칭 패턴
- ✅ 배포 (Vercel, Docker)

---

## 📚 목차

### Part 01: Next.js 시작하기

- [Chapter 1. Next.js 소개](part01-getting-started/chapter01-introduction.md)
  - Next.js란?
  - React vs Next.js 차이점
  - SSR, SSG, CSR 개념
  - 왜 Next.js를 사용하는가?
  - 설치 및 첫 프로젝트

- [Chapter 2. 프로젝트 구조 이해하기](part01-getting-started/chapter02-project-structure.md)
  - Next.js 프로젝트 구조
  - `app` 디렉토리 (App Router)
  - `public` 폴더 (정적 파일)
  - `next.config.js` 설정
  - 환경변수 (.env)

- [Chapter 3. 첫 페이지 만들기](part01-getting-started/chapter03-first-page.md)
  - 페이지 생성 기초
  - JSX 문법 복습
  - 컴포넌트 작성
  - 레이아웃 (Layout)
  - 메타데이터 (SEO)

---

### Part 02: 라우팅 (Routing)

- [Chapter 4. File-based Routing](part02-routing/chapter04-file-based-routing.md)
  - 폴더 기반 라우팅 개념
  - `page.tsx` 파일의 역할
  - 동적 라우트 (`[id]`, `[slug]`)
  - 중첩 라우트
  - 라우팅 모범 사례

- [Chapter 5. 네비게이션](part02-routing/chapter05-navigation.md)
  - `<Link>` 컴포넌트
  - `useRouter` 훅
  - 프로그래밍 방식 네비게이션
  - 쿼리 파라미터
  - Redirect

- [Chapter 6. 레이아웃과 템플릿](part02-routing/chapter06-layouts-templates.md)
  - 공통 레이아웃 (헤더, 푸터)
  - 중첩 레이아웃
  - 템플릿
  - 로딩 UI (`loading.tsx`)
  - 에러 처리 (`error.tsx`)

---

### Part 03: 데이터 페칭 (Data Fetching)

- [Chapter 7. Server Components에서 데이터 가져오기](part03-data-fetching/chapter07-server-data-fetching.md)
  - Server Components란?
  - `async/await` 사용
  - `fetch` API
  - 캐싱 전략
  - Revalidation

- [Chapter 8. Client Components에서 데이터 가져오기](part03-data-fetching/chapter08-client-data-fetching.md)
  - Client Components란?
  - `useEffect` + `fetch`
  - SWR (Stale-While-Revalidate)
  - TanStack Query (React Query)
  - 로딩 상태 관리

- [Chapter 9. SSR, SSG, ISR 이해하기](part03-data-fetching/chapter09-rendering-strategies.md)
  - SSR (Server-Side Rendering)
  - SSG (Static Site Generation)
  - ISR (Incremental Static Regeneration)
  - CSR (Client-Side Rendering)
  - 각 전략의 사용 시기

---

### Part 04: Server와 Client Components

- [Chapter 10. Server Components 심화](part04-server-components/chapter10-server-components-deep.md)
  - Server Components의 장점
  - 서버에서 실행되는 코드
  - 데이터베이스 직접 접근
  - 보안 (API 키 숨기기)
  - 제약사항 이해하기

- [Chapter 11. Client Components 심화](part04-server-components/chapter11-client-components-deep.md)
  - `'use client'` 지시어
  - 상호작용 (버튼 클릭, 폼 입력)
  - useState, useEffect 사용
  - 브라우저 API 사용
  - 이벤트 핸들러

- [Chapter 12. Server와 Client 조합하기](part04-server-components/chapter12-combining-components.md)
  - 컴포넌트 구성 패턴
  - Props 전달 규칙
  - Context API 사용
  - 실전 예제
  - 성능 최적화

---

### Part 05: 스타일링 (Styling)

- [Chapter 13. CSS Modules](part05-styling/chapter13-css-modules.md)
  - CSS Modules 기초
  - 컴포넌트별 스타일
  - 클래스 네이밍
  - Global CSS
  - 조건부 스타일

- [Chapter 14. Tailwind CSS](part05-styling/chapter14-tailwind-css.md)
  - Tailwind CSS 설치
  - 유틸리티 클래스 사용법
  - 반응형 디자인
  - 다크 모드
  - 커스터마이징

- [Chapter 15. CSS-in-JS (Styled Components)](part05-styling/chapter15-styled-components.md)
  - Styled Components 설치
  - 스타일 컴포넌트 작성
  - Props 기반 스타일
  - 테마 적용
  - Server Components와 함께 사용

---

### Part 06: 폼과 데이터 처리

- [Chapter 16. Server Actions](part06-forms-data/chapter16-server-actions.md)
  - Server Actions란?
  - 폼 제출 처리
  - 데이터 Mutation
  - Revalidation
  - 에러 처리

- [Chapter 17. 외부 API 연동](part06-forms-data/chapter17-api-integration.md)
  - REST API 호출
  - API 프록시 패턴
  - 에러 핸들링
  - 캐싱 전략
  - 환경변수 관리

---

### Part 07: 인증과 인가 (Authentication & Authorization)

- [Chapter 18. 인증 기초](part07-authentication/chapter18-auth-basics.md)
  - 인증 vs 인가
  - 세션 기반 vs 토큰 기반
  - JWT 이해하기
  - 쿠키 vs LocalStorage
  - 프론트엔드 인증 패턴

- [Chapter 19. NextAuth.js](part07-authentication/chapter19-nextauth.md)
  - NextAuth.js 설치
  - 소셜 로그인 (Google, GitHub)
  - 이메일/비밀번호 로그인
  - 세션 관리
  - 보호된 페이지

- [Chapter 20. 권한 관리 (Authorization)](part07-authentication/chapter20-authorization.md)
  - Role 기반 권한
  - 미들웨어로 보호
  - Server/Client에서 권한 체크
  - 조건부 렌더링
  - Protected Routes 패턴

---

### Part 08: 성능 최적화

- [Chapter 21. 이미지 최적화](part08-optimization/chapter21-image-optimization.md)
  - next/image 컴포넌트
  - 자동 최적화 (WebP, AVIF)
  - 반응형 이미지
  - Lazy Loading
  - 외부 이미지 처리

- [Chapter 22. 번들 최적화](part08-optimization/chapter22-bundle-optimization.md)
  - 번들 분석 (Bundle Analyzer)
  - Dynamic Import (코드 스플리팅)
  - 라이브러리 최적화
  - Tree Shaking
  - 폰트 최적화

- [Chapter 23. 캐싱 전략](part08-optimization/chapter23-caching-strategies.md)
  - Next.js 캐싱 레이어
  - Data Cache (fetch)
  - Full Route Cache
  - ISR (Incremental Static Regeneration)
  - On-demand Revalidation

---

### Part 09: 배포 (Deployment)

- [Chapter 24. Vercel 배포](part09-deployment/chapter24-vercel-deployment.md)
  - Vercel이란?
  - Git 연동 자동 배포
  - 환경변수 설정
  - 커스텀 도메인
  - Edge Functions

- [Chapter 25. Docker 배포](part09-deployment/chapter25-docker-deployment.md)
  - Docker 기초
  - Dockerfile 작성
  - Docker Compose
  - 프로덕션 배포 (AWS ECS)
  - CI/CD 파이프라인

---

### Part 10: 실전 프로젝트

- [Chapter 26. 블로그 플랫폼](part10-projects/chapter26-blog-platform.md)
  - NextAuth.js 인증
  - 마크다운 에디터
  - 댓글 시스템
  - 검색 기능
  - RSS 피드

- [Chapter 27. E-Commerce](part10-projects/chapter27-ecommerce.md)
  - 상품 목록/상세
  - 장바구니 (Zustand)
  - 결제 연동 (Stripe)
  - Webhook 처리
  - 주문 관리

---

## 🎯 학습 로드맵

### 1단계: 선수 지식 (2-4주)
- `language/javascript/part01-06`: JavaScript 기초
- `language/javascript/part07`: TypeScript
- `language/javascript/part08-09`: React

### 2단계: Next.js 기초 (1-2주)
- Part 01-02: 프로젝트 생성, 라우팅

### 3단계: 데이터 처리 (2-3주)
- Part 03-04: 데이터 페칭, Server/Client Components

### 4단계: 실전 기능 (2-3주)
- Part 05-08: 스타일링, 폼 처리, 인증, 성능 최적화

### 5단계: 배포 및 실전 (1-2주)
- Part 09-10: 배포, 실전 프로젝트

**총 학습 기간: 8-14주**

---

## 🔥 각 챕터 구성

각 챕터마다 포함된 내용:
- ✅ **핵심 개념 설명**
- ✅ **실습 예제 (따라하기)**
- ✅ **연습 문제 + 정답**
- ✅ **핵심 요약**
- ✅ **실전 팁과 베스트 프랙티스**

---

## 📖 이 가이드의 특징

1. **프론트엔드 프레임워크 중심**
   - SSR/SSG를 통한 SEO 최적화
   - 성능 최적화 기법
   - 모던 프론트엔드 개발 패턴

2. **단계별 학습**
   - 기초부터 심화까지
   - 풍부한 예제
   - 실습 중심

3. **실전 중심**
   - 실무 패턴
   - 프로덕션 레벨 코드
   - 베스트 프랙티스

4. **최신 기술 스택**
   - Next.js 14+ (App Router)
   - React 18+
   - TypeScript
   - Server/Client Components

---

## 🚀 시작하기

### 선수 지식 체크
1. JavaScript 기초를 아시나요? → [language/javascript](../language/javascript)
2. TypeScript를 아시나요? → [language/javascript/part07](../language/javascript/part07-typescript)
3. React를 아시나요? → [language/javascript/part08-09](../language/javascript/part08-react-fundamentals)

**모두 YES라면:** [Part 1: Next.js 시작하기 →](part01-getting-started/chapter01-introduction.md)

**NO가 하나라도 있다면:** `language/javascript`부터 먼저 공부하세요!

---

## 💡 학습 Tips

### Next.js 핵심 개념
1. Next.js는 **React 기반 프론트엔드 프레임워크**
2. SSR/SSG로 **SEO와 성능을 동시에 해결**
3. Server Components는 **서버에서 실행되는 React 컴포넌트**
4. 파일 기반 라우팅으로 **직관적인 페이지 구조**

### 효과적인 학습 방법
1. 각 챕터의 예제를 **직접 타이핑**하세요 (복붙 금지!)
2. 에러가 나면 **에러 메시지를 읽어보세요**
3. 작은 프로젝트부터 시작하세요
4. 공식 문서를 활용하세요
5. Server/Client Components 구분을 명확히 이해하세요

---

**작성일**: 2024년
**Next.js 버전**: 14+
**대상**: Spring Boot 백엔드 개발자 (프론트엔드 처음)