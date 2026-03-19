# Chapter 6. 레이아웃과 템플릿

## 6.1 레이아웃 (Layout) 이해하기

레이아웃은 여러 페이지에서 **공통으로 사용되는 UI**입니다. 헤더, 푸터, 사이드바 등이 레이아웃에 포함됩니다.

### 6.1.1 레이아웃의 핵심 개념

```typescript
// app/layout.tsx
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <header>공통 헤더</header>
      <main>{children}</main>  {/* 페이지 내용이 여기에 */}
      <footer>공통 푸터</footer>
    </div>
  );
}
```

**특징:**
- 페이지 전환 시 **레이아웃은 유지**됨
- 상태(state)가 보존됨
- 불필요한 리렌더링 방지
- 성능 최적화

## 6.2 루트 레이아웃 (Root Layout)

모든 Next.js 프로젝트는 **반드시 루트 레이아웃**이 있어야 합니다.

### 6.2.1 기본 루트 레이아웃

```typescript
// app/layout.tsx
import './globals.css';

export const metadata = {
  title: 'My Next.js App',
  description: 'Created with Next.js',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>
        {children}
      </body>
    </html>
  );
}
```

**주의사항:**
- `<html>`, `<body>` 태그를 **반드시** 포함해야 함
- 전역 CSS 임포트
- 전역 메타데이터 설정

### 6.2.2 헤더와 푸터 추가

```typescript
// app/layout.tsx
import './globals.css';
import Header from '@/components/layout/Header';
import Footer from '@/components/layout/Footer';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>
        <Header />
        <main className="min-h-screen">
          {children}
        </main>
        <Footer />
      </body>
    </html>
  );
}
```

```typescript
// components/layout/Header.tsx
import Link from 'next/link';

export default function Header() {
  return (
    <header className="bg-gray-800 text-white p-4">
      <nav className="container mx-auto flex justify-between">
        <Link href="/" className="text-xl font-bold">
          My Site
        </Link>
        <div className="flex gap-4">
          <Link href="/products">상품</Link>
          <Link href="/about">소개</Link>
          <Link href="/contact">연락처</Link>
        </div>
      </nav>
    </header>
  );
}
```

```typescript
// components/layout/Footer.tsx
export default function Footer() {
  return (
    <footer className="bg-gray-800 text-white p-4 text-center">
      <p>&copy; 2024 My Company. All rights reserved.</p>
    </footer>
  );
}
```

## 6.3 중첩 레이아웃 (Nested Layouts)

폴더마다 `layout.tsx`를 만들면 중첩된 레이아웃을 적용할 수 있습니다.

### 6.3.1 기본 중첩 레이아웃

```
app/
├── layout.tsx              # 루트 레이아웃
├── page.tsx                # /
└── dashboard/
    ├── layout.tsx          # 대시보드 레이아웃
    ├── page.tsx            # /dashboard
    └── settings/
        └── page.tsx        # /dashboard/settings
```

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      {/* 사이드바 */}
      <aside className="w-64 bg-gray-100 p-4">
        <nav>
          <ul className="space-y-2">
            <li><a href="/dashboard">대시보드</a></li>
            <li><a href="/dashboard/analytics">분석</a></li>
            <li><a href="/dashboard/settings">설정</a></li>
          </ul>
        </nav>
      </aside>

      {/* 메인 콘텐츠 */}
      <div className="flex-1 p-4">
        {children}
      </div>
    </div>
  );
}
```

**렌더링 구조:**
```
/dashboard 접속 시:

RootLayout (헤더, 푸터)
  └─ DashboardLayout (사이드바)
       └─ DashboardPage (콘텐츠)
```

### 6.3.2 3단계 중첩 레이아웃

```
app/
├── layout.tsx                    # 1단계: 루트
└── shop/
    ├── layout.tsx                # 2단계: 쇼핑몰
    └── products/
        ├── layout.tsx            # 3단계: 상품
        └── [id]/
            └── page.tsx
```

```typescript
// app/shop/layout.tsx (쇼핑몰 레이아웃)
export default function ShopLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <div className="bg-blue-500 text-white p-2">
        🛒 쇼핑몰 배너
      </div>
      {children}
    </div>
  );
}
```

```typescript
// app/shop/products/layout.tsx (상품 레이아웃)
export default function ProductsLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <aside className="w-48">
        <h3>필터</h3>
        <ul>
          <li>전자제품</li>
          <li>가구</li>
          <li>의류</li>
        </ul>
      </aside>
      <main className="flex-1">
        {children}
      </main>
    </div>
  );
}
```

**렌더링 구조:**
```
/shop/products/123 접속 시:

RootLayout (헤더, 푸터)
  └─ ShopLayout (쇼핑몰 배너)
       └─ ProductsLayout (필터 사이드바)
            └─ ProductPage (상품 상세)
```

## 6.4 레이아웃과 메타데이터

### 6.4.1 레이아웃에서 메타데이터 설정

```typescript
// app/layout.tsx
export const metadata = {
  title: {
    default: 'My Website',
    template: '%s | My Website',  // 자식 페이지에서 사용
  },
  description: 'The best website',
};

export default function RootLayout({ children }) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

```typescript
// app/about/page.tsx
export const metadata = {
  title: 'About Us',  // 결과: "About Us | My Website"
};

export default function AboutPage() {
  return <h1>회사 소개</h1>;
}
```

### 6.4.2 섹션별 메타데이터

```typescript
// app/blog/layout.tsx
export const metadata = {
  title: {
    default: '블로그',
    template: '%s | 블로그',
  },
};

export default function BlogLayout({ children }) {
  return <div>{children}</div>;
}
```

```typescript
// app/blog/[slug]/page.tsx
export const metadata = {
  title: 'Next.js 소개',  // 결과: "Next.js 소개 | 블로그 | My Website"
};
```

## 6.5 로딩 UI (`loading.tsx`)

페이지 로딩 중 표시할 UI를 정의합니다.

### 6.5.1 기본 로딩 UI

```typescript
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="flex items-center justify-center min-h-screen">
      <div className="animate-spin rounded-full h-32 w-32 border-t-2 border-b-2 border-blue-500"></div>
    </div>
  );
}
```

**동작 방식:**
```
1. 사용자가 /dashboard 클릭
2. loading.tsx 즉시 표시
3. 백그라운드에서 page.tsx 데이터 로딩
4. 로딩 완료 후 page.tsx 표시
```

### 6.5.2 중첩 로딩 UI

```
app/
└── dashboard/
    ├── loading.tsx           # /dashboard 로딩
    ├── page.tsx
    └── analytics/
        ├── loading.tsx       # /dashboard/analytics 로딩
        └── page.tsx
```

### 6.5.3 스켈레톤 UI

```typescript
// app/products/loading.tsx
export default function Loading() {
  return (
    <div className="grid grid-cols-3 gap-4">
      {[...Array(6)].map((_, i) => (
        <div key={i} className="border rounded-lg p-4 animate-pulse">
          <div className="bg-gray-300 h-48 rounded mb-4"></div>
          <div className="bg-gray-300 h-4 rounded mb-2"></div>
          <div className="bg-gray-300 h-4 rounded w-2/3"></div>
        </div>
      ))}
    </div>
  );
}
```

## 6.6 에러 처리 (`error.tsx`)

에러 발생 시 표시할 UI를 정의합니다.

### 6.6.1 기본 에러 UI

```typescript
// app/dashboard/error.tsx
'use client';  // 반드시 클라이언트 컴포넌트!

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h2 className="text-2xl font-bold mb-4">오류가 발생했습니다!</h2>
      <p className="text-gray-600 mb-4">{error.message}</p>
      <button
        onClick={reset}
        className="px-4 py-2 bg-blue-500 text-white rounded"
      >
        다시 시도
      </button>
    </div>
  );
}
```

### 6.6.2 에러 발생시키기

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const data = await fetch('https://api.example.com/data');

  if (!data.ok) {
    throw new Error('데이터를 불러올 수 없습니다');
  }

  const result = await data.json();

  return <div>{result.title}</div>;
}
```

### 6.6.3 전역 에러 핸들러

```typescript
// app/global-error.tsx
'use client';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <html>
      <body>
        <h2>앱 전체 에러!</h2>
        <button onClick={reset}>다시 시도</button>
      </body>
    </html>
  );
}
```

## 6.7 404 페이지 (`not-found.tsx`)

페이지를 찾을 수 없을 때 표시합니다.

### 6.7.1 기본 404 페이지

```typescript
// app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h2 className="text-4xl font-bold mb-4">404</h2>
      <p className="text-xl mb-4">페이지를 찾을 수 없습니다</p>
      <Link
        href="/"
        className="px-4 py-2 bg-blue-500 text-white rounded"
      >
        홈으로 돌아가기
      </Link>
    </div>
  );
}
```

### 6.7.2 `notFound()` 함수 사용

```typescript
// app/products/[id]/page.tsx
import { notFound } from 'next/navigation';

interface Props {
  params: { id: string };
}

const products = {
  '1': { name: '노트북' },
  '2': { name: '마우스' },
};

export default function ProductPage({ params }: Props) {
  const product = products[params.id as keyof typeof products];

  if (!product) {
    notFound();  // not-found.tsx 표시
  }

  return <h1>{product.name}</h1>;
}
```

### 6.7.3 중첩 404 페이지

```typescript
// app/blog/not-found.tsx (블로그 전용 404)
export default function BlogNotFound() {
  return (
    <div>
      <h2>블로그 포스트를 찾을 수 없습니다</h2>
      <a href="/blog">블로그 홈으로</a>
    </div>
  );
}
```

## 6.8 템플릿 (Template)

템플릿은 레이아웃과 비슷하지만, **매번 새로 마운트**됩니다.

### 6.8.1 레이아웃 vs 템플릿

**레이아웃:**
- 페이지 전환 시 **유지**됨
- 상태 보존
- 리렌더링 안 됨

**템플릿:**
- 페이지 전환 시 **새로 마운트**
- 상태 초기화
- 매번 리렌더링

### 6.8.2 템플릿 사용 예시

```typescript
// app/dashboard/template.tsx
'use client';

import { useEffect } from 'react';

export default function Template({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    console.log('페이지 전환마다 실행됨!');
  }, []);

  return (
    <div className="animate-fade-in">
      {children}
    </div>
  );
}
```

**사용 시나리오:**
- 페이지 전환 애니메이션
- 페이지별 분석 추적
- 입력 폼 초기화

## 6.9 실전 예제: 완전한 레이아웃 구조

```
app/
├── layout.tsx                    # 루트 레이아웃
├── page.tsx                      # 홈
├── (marketing)/
│   ├── layout.tsx               # 마케팅 레이아웃
│   ├── about/
│   │   └── page.tsx
│   └── contact/
│       └── page.tsx
└── (dashboard)/
    ├── layout.tsx               # 대시보드 레이아웃
    ├── loading.tsx              # 대시보드 로딩
    ├── error.tsx                # 대시보드 에러
    └── dashboard/
        ├── page.tsx
        └── settings/
            └── page.tsx
```

### 루트 레이아웃

```typescript
// app/layout.tsx
import './globals.css';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: {
    default: 'My App',
    template: '%s | My App',
  },
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

### 마케팅 레이아웃

```typescript
// app/(marketing)/layout.tsx
import Link from 'next/link';

export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <header className="bg-white border-b">
        <nav className="container mx-auto p-4 flex justify-between">
          <Link href="/" className="font-bold">Logo</Link>
          <div className="flex gap-4">
            <Link href="/about">소개</Link>
            <Link href="/contact">연락처</Link>
          </div>
        </nav>
      </header>
      <main className="container mx-auto p-4">
        {children}
      </main>
      <footer className="bg-gray-100 p-4 text-center">
        <p>&copy; 2024 My Company</p>
      </footer>
    </div>
  );
}
```

### 대시보드 레이아웃

```typescript
// app/(dashboard)/layout.tsx
import Link from 'next/link';

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex h-screen">
      <aside className="w-64 bg-gray-800 text-white p-4">
        <h2 className="text-xl font-bold mb-4">대시보드</h2>
        <nav>
          <ul className="space-y-2">
            <li>
              <Link href="/dashboard" className="block p-2 hover:bg-gray-700 rounded">
                홈
              </Link>
            </li>
            <li>
              <Link href="/dashboard/analytics" className="block p-2 hover:bg-gray-700 rounded">
                분석
              </Link>
            </li>
            <li>
              <Link href="/dashboard/settings" className="block p-2 hover:bg-gray-700 rounded">
                설정
              </Link>
            </li>
          </ul>
        </nav>
      </aside>
      <main className="flex-1 p-8 overflow-auto">
        {children}
      </main>
    </div>
  );
}
```

## 핵심 요약

1. **레이아웃**
   - 공통 UI 정의
   - 페이지 전환 시 유지
   - 상태 보존

2. **중첩 레이아웃**
   - 폴더마다 `layout.tsx`
   - 부모 레이아웃 안에 중첩

3. **특수 파일**
   - `loading.tsx`: 로딩 UI
   - `error.tsx`: 에러 UI
   - `not-found.tsx`: 404 페이지

4. **템플릿**
   - 매번 새로 마운트
   - 상태 초기화
   - 애니메이션에 유용

5. **라우트 그룹**
   - `(folder)`: URL에 포함 안 됨
   - 레이아웃 분리에 유용

## 연습 문제

### 문제 1
헤더와 푸터가 있는 루트 레이아웃을 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>
        <header className="bg-gray-800 text-white p-4">
          <nav>
            <a href="/">홈</a>
            <a href="/about">소개</a>
          </nav>
        </header>
        <main className="min-h-screen">
          {children}
        </main>
        <footer className="bg-gray-100 p-4 text-center">
          <p>&copy; 2024</p>
        </footer>
      </body>
    </html>
  );
}
```
</details>

### 문제 2
대시보드 페이지에 사이드바가 있는 레이아웃을 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <aside className="w-64 bg-gray-100 p-4">
        <nav>
          <ul>
            <li><a href="/dashboard">홈</a></li>
            <li><a href="/dashboard/analytics">분석</a></li>
            <li><a href="/dashboard/settings">설정</a></li>
          </ul>
        </nav>
      </aside>
      <main className="flex-1 p-4">
        {children}
      </main>
    </div>
  );
}
```
</details>

### 문제 3
로딩 스피너와 에러 페이지를 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/products/loading.tsx
export default function Loading() {
  return (
    <div className="flex justify-center items-center min-h-screen">
      <div className="animate-spin rounded-full h-12 w-12 border-t-2 border-blue-500"></div>
    </div>
  );
}

// app/products/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div className="text-center p-8">
      <h2 className="text-2xl font-bold mb-4">오류 발생!</h2>
      <p className="mb-4">{error.message}</p>
      <button
        onClick={reset}
        className="px-4 py-2 bg-blue-500 text-white rounded"
      >
        다시 시도
      </button>
    </div>
  );
}
```
</details>

## 다음 단계

Part 02 완료! 다음 Part에서는:
- Server Components에서 데이터 가져오기
- Client Components에서 데이터 가져오기
- SSR, SSG, ISR 심화

[Part 03: 데이터 페칭 →](../part03-data-fetching/chapter07-server-data-fetching.md)