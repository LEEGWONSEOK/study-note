# Chapter 14. Tailwind CSS

## 14.1 Tailwind CSS란?

Tailwind CSS는 **유틸리티 우선(Utility-First) CSS 프레임워크**입니다. 미리 정의된 클래스를 조합하여 스타일링합니다.

### 14.1.1 전통적인 CSS vs Tailwind

```css
/* 전통적인 CSS */
.button {
  background-color: #0070f3;
  color: white;
  padding: 12px 24px;
  border-radius: 6px;
  font-weight: 600;
}
```

```typescript
// Tailwind CSS
<button className="bg-blue-500 text-white px-6 py-3 rounded-md font-semibold">
  버튼
</button>
```

**장점:**
- ✅ CSS 파일 작성 불필요
- ✅ 빠른 프로토타이핑
- ✅ 일관된 디자인 시스템
- ✅ 자동 최적화 (미사용 CSS 제거)

**단점:**
- ❌ 클래스명이 길어질 수 있음
- ❌ 학습 곡선
- ❌ HTML이 복잡해 보일 수 있음

## 14.2 Tailwind CSS 설치

Next.js는 Tailwind CSS를 기본 지원합니다.

### 14.2.1 새 프로젝트에서 설치

```bash
# Next.js 프로젝트 생성 시 Tailwind 선택
npx create-next-app@latest my-app
# ✔ Would you like to use Tailwind CSS? › Yes
```

### 14.2.2 기존 프로젝트에 설치

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

```css
/* app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## 14.3 기본 유틸리티 클래스

### 14.3.1 색상

```typescript
// 배경색
<div className="bg-blue-500">파랑</div>
<div className="bg-red-500">빨강</div>
<div className="bg-green-500">초록</div>

// 텍스트 색상
<p className="text-blue-500">파란 텍스트</p>
<p className="text-gray-700">회색 텍스트</p>

// 테두리 색상
<div className="border border-blue-500">테두리</div>
```

**색상 스케일:**
```
50  - 가장 밝음
100, 200, 300, 400, 500 - 중간
600, 700, 800, 900 - 어두움
950 - 가장 어두움
```

### 14.3.2 간격 (Spacing)

```typescript
// Padding
<div className="p-4">padding: 1rem</div>
<div className="px-4">padding-left & right: 1rem</div>
<div className="py-4">padding-top & bottom: 1rem</div>
<div className="pt-4">padding-top: 1rem</div>
<div className="pr-4">padding-right: 1rem</div>
<div className="pb-4">padding-bottom: 1rem</div>
<div className="pl-4">padding-left: 1rem</div>

// Margin
<div className="m-4">margin: 1rem</div>
<div className="mx-auto">margin: 0 auto (중앙 정렬)</div>
<div className="mt-8">margin-top: 2rem</div>

// 간격 스케일: 0, 1, 2, 3, 4, 5, 6, 8, 10, 12, 16, 20, 24, 32...
// 4 = 1rem = 16px
```

### 14.3.3 크기

```typescript
// Width
<div className="w-full">width: 100%</div>
<div className="w-1/2">width: 50%</div>
<div className="w-64">width: 16rem</div>
<div className="w-screen">width: 100vw</div>

// Height
<div className="h-64">height: 16rem</div>
<div className="h-screen">height: 100vh</div>

// Min/Max
<div className="min-h-screen">min-height: 100vh</div>
<div className="max-w-4xl">max-width: 56rem</div>
```

### 14.3.4 Flexbox

```typescript
// Flex 컨테이너
<div className="flex">
  <div>Item 1</div>
  <div>Item 2</div>
</div>

// 방향
<div className="flex flex-row">가로</div>
<div className="flex flex-col">세로</div>

// 정렬
<div className="flex justify-center">가운데 정렬</div>
<div className="flex justify-between">양쪽 정렬</div>
<div className="flex items-center">세로 중앙</div>
<div className="flex justify-center items-center">완전 중앙</div>

// Gap
<div className="flex gap-4">간격 1rem</div>
```

### 14.3.5 Grid

```typescript
// 그리드
<div className="grid grid-cols-3 gap-4">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>

// 반응형 그리드
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {/* 모바일: 1열, 태블릿: 2열, 데스크톱: 3열 */}
</div>
```

### 14.3.6 타이포그래피

```typescript
// 폰트 크기
<p className="text-xs">12px</p>
<p className="text-sm">14px</p>
<p className="text-base">16px</p>
<p className="text-lg">18px</p>
<p className="text-xl">20px</p>
<p className="text-2xl">24px</p>
<p className="text-4xl">36px</p>

// 폰트 굵기
<p className="font-light">300</p>
<p className="font-normal">400</p>
<p className="font-medium">500</p>
<p className="font-semibold">600</p>
<p className="font-bold">700</p>

// 텍스트 정렬
<p className="text-left">왼쪽</p>
<p className="text-center">중앙</p>
<p className="text-right">오른쪽</p>
```

## 14.4 반응형 디자인

### 14.4.1 브레이크포인트

```typescript
// sm: 640px
// md: 768px
// lg: 1024px
// xl: 1280px
// 2xl: 1536px

<div className="
  text-sm          {/* 모바일: 14px */}
  md:text-base     {/* 태블릿: 16px */}
  lg:text-lg       {/* 데스크톱: 18px */}
">
  반응형 텍스트
</div>
```

### 14.4.2 반응형 레이아웃

```typescript
// 반응형 컨테이너
<div className="
  container mx-auto
  px-4
  sm:px-6
  md:px-8
  lg:px-12
">
  콘텐츠
</div>

// 반응형 그리드
<div className="
  grid
  grid-cols-1      {/* 모바일: 1열 */}
  sm:grid-cols-2   {/* 작은 화면: 2열 */}
  md:grid-cols-3   {/* 중간 화면: 3열 */}
  lg:grid-cols-4   {/* 큰 화면: 4열 */}
  gap-4
">
  {products.map(product => (
    <div key={product.id}>...</div>
  ))}
</div>
```

### 14.4.3 숨기기/보이기

```typescript
// 모바일에서만 보이기
<div className="block md:hidden">
  모바일 메뉴
</div>

// 데스크톱에서만 보이기
<div className="hidden md:block">
  데스크톱 메뉴
</div>

// 태블릿에서만 보이기
<div className="hidden md:block lg:hidden">
  태블릿 전용
</div>
```

## 14.5 상태 변형 (Hover, Focus 등)

### 14.5.1 Hover

```typescript
<button className="
  bg-blue-500
  hover:bg-blue-600    {/* 마우스 오버 시 */}
  text-white
  px-6 py-3
  rounded-md
  transition           {/* 부드러운 전환 */}
">
  호버 버튼
</button>
```

### 14.5.2 Focus

```typescript
<input
  type="text"
  className="
    border
    border-gray-300
    focus:border-blue-500     {/* 포커스 시 */}
    focus:ring-2              {/* 링 효과 */}
    focus:ring-blue-200
    px-4 py-2
    rounded-md
    outline-none
  "
  placeholder="이메일"
/>
```

### 14.5.3 Active, Disabled

```typescript
<button className="
  bg-blue-500
  active:bg-blue-700           {/* 클릭 시 */}
  disabled:opacity-50          {/* 비활성화 시 */}
  disabled:cursor-not-allowed
  text-white
  px-6 py-3
  rounded-md
">
  버튼
</button>
```

### 14.5.4 그룹 Hover

```typescript
<div className="group border rounded-lg p-4 hover:shadow-lg">
  <h3 className="group-hover:text-blue-500">제목</h3>
  <p className="text-gray-600 group-hover:text-gray-900">설명</p>
  <button className="opacity-0 group-hover:opacity-100 transition">
    더보기
  </button>
</div>
```

## 14.6 다크 모드

### 14.6.1 설정

```javascript
// tailwind.config.js
module.exports = {
  darkMode: 'class', // 또는 'media'
  // ...
}
```

### 14.6.2 클래스 기반 다크 모드

```typescript
<div className="
  bg-white
  dark:bg-gray-900        {/* 다크 모드: 어두운 배경 */}
  text-gray-900
  dark:text-white         {/* 다크 모드: 흰색 텍스트 */}
  p-8
">
  콘텐츠
</div>
```

### 14.6.3 다크 모드 토글

```typescript
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko" className="dark">  {/* 'dark' 클래스 추가/제거 */}
      <body>{children}</body>
    </html>
  );
}
```

```typescript
// components/ThemeToggle.tsx
'use client';

import { useState, useEffect } from 'react';

export default function ThemeToggle() {
  const [darkMode, setDarkMode] = useState(false);

  useEffect(() => {
    if (darkMode) {
      document.documentElement.classList.add('dark');
    } else {
      document.documentElement.classList.remove('dark');
    }
  }, [darkMode]);

  return (
    <button
      onClick={() => setDarkMode(!darkMode)}
      className="
        px-4 py-2
        bg-gray-200 dark:bg-gray-700
        text-gray-900 dark:text-white
        rounded-md
      "
    >
      {darkMode ? '🌙 다크' : '☀️ 라이트'}
    </button>
  );
}
```

## 14.7 커스터마이징

### 14.7.1 색상 커스터마이징

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
        brand: '#FF6B6B',
      },
    },
  },
}
```

```typescript
// 사용
<div className="bg-primary-500">Primary 500</div>
<div className="bg-brand">Brand Color</div>
```

### 14.7.2 간격 커스터마이징

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      spacing: {
        '128': '32rem',
        '144': '36rem',
      },
    },
  },
}
```

```typescript
<div className="w-128">width: 32rem</div>
```

### 14.7.3 폰트 커스터마이징

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: ['Pretendard', 'sans-serif'],
        mono: ['Fira Code', 'monospace'],
      },
    },
  },
}
```

```typescript
<p className="font-sans">Pretendard</p>
<code className="font-mono">Code Font</code>
```

## 14.8 재사용 가능한 컴포넌트

### 14.8.1 버튼 컴포넌트

```typescript
// components/Button.tsx
import { ButtonHTMLAttributes } from 'react';
import clsx from 'clsx';

interface Props extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
}

export default function Button({
  variant = 'primary',
  size = 'md',
  children,
  className,
  ...props
}: Props) {
  return (
    <button
      className={clsx(
        // 기본 스타일
        'rounded-md font-semibold transition',

        // 변형
        {
          'bg-blue-500 hover:bg-blue-600 text-white': variant === 'primary',
          'bg-gray-200 hover:bg-gray-300 text-gray-900': variant === 'secondary',
          'bg-red-500 hover:bg-red-600 text-white': variant === 'danger',
        },

        // 크기
        {
          'px-3 py-1.5 text-sm': size === 'sm',
          'px-4 py-2 text-base': size === 'md',
          'px-6 py-3 text-lg': size === 'lg',
        },

        // 추가 클래스
        className
      )}
      {...props}
    >
      {children}
    </button>
  );
}
```

```typescript
// 사용
<Button variant="primary" size="lg">주 버튼</Button>
<Button variant="secondary">보조 버튼</Button>
<Button variant="danger" size="sm">삭제</Button>
```

### 14.8.2 카드 컴포넌트

```typescript
// components/Card.tsx
import { ReactNode } from 'react';
import clsx from 'clsx';

interface Props {
  children: ReactNode;
  className?: string;
  hover?: boolean;
}

export default function Card({ children, className, hover = false }: Props) {
  return (
    <div
      className={clsx(
        'border border-gray-200 rounded-lg p-6 bg-white',
        hover && 'hover:shadow-lg transition cursor-pointer',
        className
      )}
    >
      {children}
    </div>
  );
}
```

```typescript
// 사용
<Card hover>
  <h3 className="text-xl font-bold mb-2">제목</h3>
  <p className="text-gray-600">내용</p>
</Card>
```

## 14.9 실전 예제: 블로그 레이아웃

```typescript
// app/blog/page.tsx
export default function BlogPage() {
  const posts = [
    { id: 1, title: 'Next.js 시작하기', excerpt: '...', date: '2024-01-15' },
    { id: 2, title: 'Tailwind CSS', excerpt: '...', date: '2024-01-20' },
    { id: 3, title: 'TypeScript', excerpt: '...', date: '2024-01-25' },
  ];

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <header className="bg-white border-b border-gray-200">
        <div className="container mx-auto px-4 py-6">
          <h1 className="text-3xl font-bold text-gray-900">블로그</h1>
        </div>
      </header>

      {/* Main Content */}
      <main className="container mx-auto px-4 py-8">
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {posts.map(post => (
            <article
              key={post.id}
              className="
                bg-white
                border border-gray-200
                rounded-lg
                p-6
                hover:shadow-lg
                transition
                cursor-pointer
              "
            >
              <h2 className="text-xl font-bold text-gray-900 mb-2">
                {post.title}
              </h2>
              <p className="text-gray-600 mb-4">
                {post.excerpt}
              </p>
              <time className="text-sm text-gray-500">
                {post.date}
              </time>
            </article>
          ))}
        </div>
      </main>

      {/* Footer */}
      <footer className="bg-white border-t border-gray-200 mt-12">
        <div className="container mx-auto px-4 py-6 text-center text-gray-600">
          © 2024 My Blog
        </div>
      </footer>
    </div>
  );
}
```

## 14.10 Tailwind 모범 사례

### 14.10.1 긴 클래스명 정리

```typescript
// ❌ 읽기 어려움
<div className="flex items-center justify-between px-4 py-3 bg-white border border-gray-200 rounded-lg shadow-sm hover:shadow-md transition">

// ✅ 줄바꿈으로 정리
<div className="
  flex items-center justify-between
  px-4 py-3
  bg-white border border-gray-200
  rounded-lg shadow-sm
  hover:shadow-md transition
">
```

### 14.10.2 컴포넌트로 추상화

```typescript
// ❌ 중복된 스타일
<button className="px-4 py-2 bg-blue-500 text-white rounded-md">버튼 1</button>
<button className="px-4 py-2 bg-blue-500 text-white rounded-md">버튼 2</button>

// ✅ 컴포넌트로 추상화
function PrimaryButton({ children }) {
  return (
    <button className="px-4 py-2 bg-blue-500 text-white rounded-md">
      {children}
    </button>
  );
}
```

### 14.10.3 @apply 사용 (선택적)

```css
/* globals.css */
@layer components {
  .btn-primary {
    @apply px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 transition;
  }
}
```

```typescript
<button className="btn-primary">버튼</button>
```

## 핵심 요약

1. **Tailwind CSS**
   - 유틸리티 우선 CSS 프레임워크
   - HTML에서 직접 스타일링
   - 빠른 개발 속도

2. **기본 클래스**
   - 색상: `bg-`, `text-`, `border-`
   - 간격: `p-`, `m-`, `gap-`
   - 레이아웃: `flex`, `grid`

3. **반응형**
   - `sm:`, `md:`, `lg:`, `xl:`
   - 모바일 우선 접근

4. **상태**
   - `hover:`, `focus:`, `active:`
   - `dark:` (다크 모드)
   - `group-hover:`

5. **모범 사례**
   - 컴포넌트로 추상화
   - 클래스명 줄바꿈
   - 커스터마이징 활용

## 연습 문제

### 문제 1
Tailwind로 버튼을 만드세요 (파랑 배경, 흰 텍스트, 호버 효과).

<details>
<summary>정답</summary>

```typescript
<button className="
  bg-blue-500
  hover:bg-blue-600
  text-white
  px-6 py-3
  rounded-md
  transition
">
  클릭하세요
</button>
```
</details>

### 문제 2
반응형 그리드를 만드세요 (모바일 1열, 태블릿 2열, 데스크톱 3열).

<details>
<summary>정답</summary>

```typescript
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  <div className="border p-4">아이템 1</div>
  <div className="border p-4">아이템 2</div>
  <div className="border p-4">아이템 3</div>
</div>
```
</details>

### 문제 3
다크 모드를 지원하는 카드를 만드세요.

<details>
<summary>정답</summary>

```typescript
<div className="
  bg-white dark:bg-gray-800
  text-gray-900 dark:text-white
  border border-gray-200 dark:border-gray-700
  rounded-lg p-6
">
  <h3 className="text-xl font-bold mb-2">제목</h3>
  <p className="text-gray-600 dark:text-gray-400">내용</p>
</div>
```
</details>

## 다음 단계

다음 챕터에서는:
- Styled Components 설치
- CSS-in-JS
- 동적 스타일링
- 테마 시스템

[Chapter 15. Styled Components →](chapter15-styled-components.md)
