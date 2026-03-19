# Chapter 1. Next.js 소개

## 1.1 Next.js란?

Next.js는 **React 기반 프론트엔드 프레임워크**입니다.

### 핵심 정의
- **React 프레임워크**: React 위에 구축된 프레임워크
- **프로덕션 준비**: 실제 서비스에 필요한 기능들이 내장
- **풀스택 기능**: 백엔드 기능도 포함하지만, 주로 프론트엔드로 사용

```
React          →  라이브러리 (UI만 담당)
Next.js        →  프레임워크 (라우팅, SSR, 최적화 등 포함)
```

## 1.2 왜 Next.js를 사용하는가?

### React만으로는 부족한 이유

순수 React(CRA - Create React App)의 문제점:
1. **SEO 문제**: 검색 엔진이 콘텐츠를 제대로 인식 못함
2. **초기 로딩 느림**: JavaScript 다운로드 후에야 화면 표시
3. **라우팅 설정 필요**: React Router 별도 설치/설정
4. **성능 최적화 수동**: 코드 스플리팅, 이미지 최적화 등을 직접 구현

### Next.js가 해결하는 것

```typescript
// React (CRA)의 문제
// 1. SEO 불가능 - 검색엔진은 빈 HTML만 봄
<div id="root"></div>  // 서버에서 전달되는 HTML

// 2. JavaScript 로딩 후에야 화면 표시
// 사용자가 흰 화면을 오래 봄

// Next.js의 해결
// 1. 서버에서 완성된 HTML 전달 (SEO 가능)
<div>
  <h1>실제 콘텐츠</h1>
  <p>검색엔진이 읽을 수 있는 내용</p>
</div>

// 2. HTML이 먼저 보이고, 이후 JavaScript 실행
// 사용자가 즉시 콘텐츠를 볼 수 있음
```

## 1.3 React vs Next.js

### 비교표

| 기능 | React (CRA) | Next.js |
|------|-------------|---------|
| **렌더링** | CSR만 가능 | SSR, SSG, ISR, CSR 모두 가능 |
| **라우팅** | React Router 설치 필요 | 파일 기반 라우팅 내장 |
| **SEO** | 어려움 | 기본 지원 |
| **성능 최적화** | 수동 구현 | 자동 최적화 |
| **이미지 최적화** | 수동 | 자동 (next/image) |
| **코드 스플리팅** | 수동 | 자동 |
| **API Routes** | 없음 | 내장 |
| **배포** | 정적 호스팅 | Vercel, 서버 호스팅 |

### 언제 무엇을 사용할까?

**React (CRA)를 사용하는 경우:**
- 간단한 SPA (Single Page Application)
- SEO가 필요 없는 내부 관리자 페이지
- 모바일 앱의 웹뷰

**Next.js를 사용하는 경우:**
- SEO가 중요한 서비스 (블로그, 쇼핑몰, 마케팅 페이지)
- 초기 로딩 속도가 중요한 서비스
- 성능 최적화가 필요한 대규모 프로젝트
- 풀스택 개발이 필요한 경우

## 1.4 렌더링 전략 (SSR, SSG, CSR)

Next.js의 가장 큰 특징은 **다양한 렌더링 방식을 지원**한다는 것입니다.

### 1.4.1 CSR (Client-Side Rendering)

**React의 기본 방식**

```typescript
// React의 CSR 동작 방식
1. 서버: 빈 HTML + JavaScript 파일 전달
2. 브라우저: JavaScript 다운로드
3. 브라우저: JavaScript 실행 → 화면 렌더링
4. 사용자: 이제야 콘텐츠 볼 수 있음
```

**장점:**
- 페이지 전환이 빠름 (SPA)
- 서버 부담 적음

**단점:**
- 초기 로딩 느림
- SEO 불가능
- JavaScript 비활성화 시 동작 안 함

### 1.4.2 SSR (Server-Side Rendering)

**요청마다 서버에서 HTML 생성**

```typescript
// app/products/page.tsx - SSR 예제
export default async function ProductsPage() {
  // 매 요청마다 서버에서 실행
  const res = await fetch('https://api.example.com/products', {
    cache: 'no-store'  // 캐시 사용 안 함
  });
  const products = await res.json();

  return (
    <div>
      <h1>상품 목록</h1>
      {products.map(product => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

**동작 방식:**
```
1. 사용자 요청 → 서버
2. 서버: API 호출, 데이터 가져오기
3. 서버: HTML 생성 (React 렌더링)
4. 서버 → 브라우저: 완성된 HTML 전달
5. 사용자: 즉시 콘텐츠 볼 수 있음
6. 브라우저: JavaScript 실행 (Hydration)
```

**장점:**
- SEO 최적화
- 초기 로딩 빠름
- 항상 최신 데이터

**단점:**
- 서버 부담 증가
- 요청마다 렌더링 (느릴 수 있음)

**사용 예시:**
- 실시간 대시보드
- 사용자별 맞춤 콘텐츠
- 자주 변경되는 데이터

### 1.4.3 SSG (Static Site Generation)

**빌드 시점에 HTML 생성**

```typescript
// app/about/page.tsx - SSG 예제
export default function AboutPage() {
  return (
    <div>
      <h1>회사 소개</h1>
      <p>우리는...</p>
    </div>
  );
}
// 빌드 시 HTML 생성 → 정적 파일로 저장
```

**동작 방식:**
```
빌드 시점 (개발자가 배포할 때):
1. npm run build 실행
2. Next.js: 모든 페이지 HTML 생성
3. 생성된 HTML을 정적 파일로 저장

사용자 요청 시:
1. 사용자 요청 → 서버
2. 서버: 미리 생성된 HTML 전달 (초고속!)
3. 사용자: 즉시 콘텐츠 볼 수 있음
```

**장점:**
- 최고의 성능 (정적 파일)
- SEO 최적화
- 서버 부담 거의 없음
- CDN 활용 가능

**단점:**
- 빌드 시점의 데이터만 사용
- 자주 변경되는 데이터에 부적합

**사용 예시:**
- 블로그 포스트
- 회사 소개 페이지
- 문서 사이트
- 마케팅 랜딩 페이지

### 1.4.4 ISR (Incremental Static Regeneration)

**SSG + 주기적 업데이트**

```typescript
// app/posts/page.tsx - ISR 예제
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 }  // 60초마다 재생성
  });
  const posts = await res.json();

  return (
    <div>
      <h1>블로그 포스트</h1>
      {posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}
```

**동작 방식:**
```
1. 빌드 시: 정적 HTML 생성 (SSG처럼)
2. 사용자 요청: 정적 HTML 전달 (빠름!)
3. 60초 후 다음 요청:
   - 기존 HTML 전달 (사용자는 빠르게 봄)
   - 백그라운드에서 새 HTML 생성
   - 다음 요청부터 새 HTML 제공
```

**장점:**
- SSG의 성능 + SSR의 최신 데이터
- 최적의 균형

**단점:**
- 완전한 실시간은 아님

**사용 예시:**
- 뉴스 사이트
- 상품 목록
- 자주 업데이트되지만 실시간은 아닌 콘텐츠

### 1.4.5 렌더링 전략 선택 가이드

```typescript
// 페이지별로 다른 렌더링 전략 사용 가능!

// 1. 정적 콘텐츠 (SSG)
// app/about/page.tsx
export default function AboutPage() {
  return <div>회사 소개</div>;
}

// 2. 주기적 업데이트 (ISR)
// app/blog/page.tsx
export default async function BlogPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }  // 1시간마다
  });
  const posts = await res.json();
  return <div>{/* ... */}</div>;
}

// 3. 실시간 데이터 (SSR)
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const res = await fetch('https://api.example.com/stats', {
    cache: 'no-store'  // 캐시 없음
  });
  const stats = await res.json();
  return <div>{/* ... */}</div>;
}

// 4. 클라이언트 렌더링 (CSR)
// app/interactive/page.tsx
'use client';
import { useState, useEffect } from 'react';

export default function InteractivePage() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch('https://api.example.com/data')
      .then(res => res.json())
      .then(setData);
  }, []);

  return <div>{/* ... */}</div>;
}
```

## 1.5 Next.js 설치 및 첫 프로젝트

### 1.5.1 시스템 요구사항

```bash
# Node.js 버전 확인 (18.17 이상 필요)
node --version
# v18.17.0 이상이어야 함

# npm 버전 확인
npm --version
```

### 1.5.2 프로젝트 생성

```bash
# Next.js 프로젝트 생성
npx create-next-app@latest my-next-app

# 설치 중 질문들
✔ Would you like to use TypeScript? … Yes
✔ Would you like to use ESLint? … Yes
✔ Would you like to use Tailwind CSS? … Yes
✔ Would you like to use `src/` directory? … No
✔ Would you like to use App Router? … Yes
✔ Would you like to customize the default import alias? … No

# 프로젝트 폴더로 이동
cd my-next-app

# 개발 서버 실행
npm run dev
```

### 1.5.3 브라우저에서 확인

```
http://localhost:3000
```

첫 페이지가 보이면 성공!

### 1.5.4 생성된 파일 구조

```
my-next-app/
├── app/                    # 라우팅 폴더 (App Router)
│   ├── favicon.ico        # 파비콘
│   ├── globals.css        # 전역 CSS
│   ├── layout.tsx         # 루트 레이아웃
│   └── page.tsx           # 홈페이지
├── public/                # 정적 파일 (이미지, 폰트 등)
├── node_modules/          # 패키지들
├── .eslintrc.json        # ESLint 설정
├── .gitignore            # Git 무시 파일
├── next.config.js        # Next.js 설정
├── package.json          # 프로젝트 정보
├── tsconfig.json         # TypeScript 설정
└── README.md             # 프로젝트 설명
```

### 1.5.5 첫 페이지 수정해보기

`app/page.tsx` 파일을 열어서 수정:

```typescript
// app/page.tsx
export default function Home() {
  return (
    <main>
      <h1>안녕하세요! Next.js입니다.</h1>
      <p>이것이 첫 번째 Next.js 프로젝트입니다.</p>
    </main>
  );
}
```

저장하면 자동으로 브라우저가 새로고침됩니다! (Fast Refresh)

## 1.6 개발 서버 명령어

```bash
# 개발 서버 실행
npm run dev

# 프로덕션 빌드
npm run build

# 프로덕션 서버 실행
npm run start

# 린터 실행
npm run lint
```

## 핵심 요약

1. **Next.js = React 기반 프론트엔드 프레임워크**
   - React만으로는 부족한 부분을 해결

2. **다양한 렌더링 전략**
   - SSG: 빌드 시 생성 (최고 성능)
   - SSR: 요청마다 생성 (실시간 데이터)
   - ISR: 주기적 재생성 (균형)
   - CSR: 클라이언트 렌더링 (상호작용)

3. **자동 최적화**
   - 코드 스플리팅
   - 이미지 최적화
   - 폰트 최적화

4. **개발자 경험**
   - Fast Refresh (즉시 새로고침)
   - TypeScript 기본 지원
   - ESLint 내장

## 연습 문제

### 문제 1
React와 Next.js의 가장 큰 차이점 3가지를 설명하세요.

<details>
<summary>정답</summary>

1. **렌더링 방식**
   - React: CSR만 가능
   - Next.js: SSR, SSG, ISR, CSR 모두 지원

2. **SEO**
   - React: 어려움 (빈 HTML 전달)
   - Next.js: 기본 지원 (완성된 HTML 전달)

3. **라우팅**
   - React: React Router 별도 설치 필요
   - Next.js: 파일 기반 라우팅 내장
</details>

### 문제 2
다음 상황에 적합한 렌더링 전략을 선택하세요:
1. 회사 소개 페이지 (거의 변경 안 됨)
2. 실시간 주식 대시보드
3. 블로그 포스트 목록 (하루에 몇 개 추가됨)

<details>
<summary>정답</summary>

1. **SSG (Static Site Generation)**
   - 정적 콘텐츠, 변경 거의 없음
   - 최고 성능

2. **SSR (Server-Side Rendering)** 또는 **CSR**
   - 실시간 데이터 필요
   - 매 요청마다 최신 데이터

3. **ISR (Incremental Static Regeneration)**
   - 주기적 업데이트 (예: 10분마다)
   - SSG의 성능 + 적절한 최신성
</details>

### 문제 3
Next.js 프로젝트를 생성하고 "Hello, Next.js!" 를 표시하는 페이지를 만드세요.

<details>
<summary>정답</summary>

```bash
# 1. 프로젝트 생성
npx create-next-app@latest hello-nextjs
cd hello-nextjs

# 2. app/page.tsx 수정
export default function Home() {
  return (
    <main>
      <h1>Hello, Next.js!</h1>
    </main>
  );
}

# 3. 개발 서버 실행
npm run dev

# 4. http://localhost:3000 접속
```
</details>

## 다음 단계

다음 챕터에서는:
- Next.js 프로젝트 구조 상세 분석
- `app` 디렉토리의 역할
- 설정 파일들 이해

[Chapter 2. 프로젝트 구조 이해하기 →](chapter02-project-structure.md)