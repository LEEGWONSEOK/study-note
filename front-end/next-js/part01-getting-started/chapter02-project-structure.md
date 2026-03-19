# Chapter 2. 프로젝트 구조 이해하기

## 2.1 Next.js 프로젝트 전체 구조

```
my-next-app/
├── app/                    # 🔥 핵심! App Router (페이지, 레이아웃)
│   ├── layout.tsx         # 루트 레이아웃
│   ├── page.tsx           # 홈페이지 (/)
│   ├── globals.css        # 전역 CSS
│   └── favicon.ico        # 파비콘
├── public/                # 정적 파일 (이미지, 폰트)
│   ├── images/
│   └── fonts/
├── components/            # 재사용 컴포넌트 (관례적으로 생성)
│   ├── Header.tsx
│   └── Footer.tsx
├── lib/                   # 유틸리티 함수, 헬퍼 (관례적으로 생성)
│   └── utils.ts
├── node_modules/          # 설치된 패키지들
├── .env.local            # 환경변수 (로컬)
├── .eslintrc.json        # ESLint 설정
├── .gitignore            # Git 무시 파일
├── next.config.js        # Next.js 설정
├── package.json          # 프로젝트 정보, 의존성
├── tsconfig.json         # TypeScript 설정
└── README.md             # 프로젝트 설명
```

### 중요한 폴더/파일 설명

| 경로 | 역할 | 수정 빈도 |
|------|------|-----------|
| `app/` | 라우팅과 페이지 정의 | 매우 자주 |
| `public/` | 정적 파일 (이미지 등) | 자주 |
| `components/` | 재사용 컴포넌트 | 매우 자주 |
| `lib/` | 유틸리티 함수 | 가끔 |
| `next.config.js` | Next.js 설정 | 가끔 |
| `.env.local` | 환경변수 | 가끔 |

## 2.2 `app` 디렉토리 (App Router)

Next.js 13+의 **핵심 기능**입니다. 폴더 구조가 곧 URL이 됩니다.

### 2.2.1 기본 개념

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
├── blog/
│   ├── page.tsx          → /blog
│   └── [slug]/
│       └── page.tsx      → /blog/hello-world (동적 라우트)
└── dashboard/
    ├── layout.tsx        → /dashboard의 레이아웃
    ├── page.tsx          → /dashboard
    └── settings/
        └── page.tsx      → /dashboard/settings
```

### 2.2.2 특수 파일명

Next.js는 특정 파일명을 특별하게 취급합니다:

| 파일명 | 역할 | 필수 여부 |
|--------|------|-----------|
| `layout.tsx` | 공통 레이아웃 | 루트에만 필수 |
| `page.tsx` | 페이지 컴포넌트 | 필수 (URL 생성) |
| `loading.tsx` | 로딩 UI | 선택 |
| `error.tsx` | 에러 UI | 선택 |
| `not-found.tsx` | 404 페이지 | 선택 |
| `route.ts` | API 엔드포인트 | 선택 |

### 2.2.3 `layout.tsx` - 레이아웃

**모든 페이지에 공통으로 적용되는 레이아웃**

```typescript
// app/layout.tsx (루트 레이아웃 - 필수!)
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
        <header>
          <nav>내비게이션</nav>
        </header>

        <main>{children}</main>

        <footer>푸터</footer>
      </body>
    </html>
  );
}
```

**중첩 레이아웃 예시**

```typescript
// app/dashboard/layout.tsx (대시보드 전용 레이아웃)
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard-container">
      <aside>사이드바</aside>
      <div className="content">{children}</div>
    </div>
  );
}
```

```
결과:
/                    → RootLayout
/dashboard           → RootLayout + DashboardLayout
/dashboard/settings  → RootLayout + DashboardLayout
```

### 2.2.4 `page.tsx` - 페이지

**실제 URL에 매핑되는 페이지**

```typescript
// app/page.tsx → /
export default function HomePage() {
  return <h1>홈페이지</h1>;
}
```

```typescript
// app/about/page.tsx → /about
export default function AboutPage() {
  return <h1>소개 페이지</h1>;
}
```

```typescript
// app/blog/[slug]/page.tsx → /blog/:slug
export default function BlogPostPage({
  params,
}: {
  params: { slug: string };
}) {
  return <h1>블로그 포스트: {params.slug}</h1>;
}
```

### 2.2.5 `loading.tsx` - 로딩 UI

**페이지 로딩 중 표시되는 UI**

```typescript
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div>
      <p>로딩 중...</p>
      <div className="spinner"></div>
    </div>
  );
}
```

**동작 방식:**
```
1. 사용자가 /dashboard 접속
2. loading.tsx 표시 (즉시)
3. page.tsx 데이터 로딩
4. page.tsx 표시 (loading.tsx 대체)
```

### 2.2.6 `error.tsx` - 에러 UI

**에러 발생 시 표시되는 UI**

```typescript
// app/dashboard/error.tsx
'use client';  // 에러 컴포넌트는 반드시 클라이언트 컴포넌트

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>에러 발생!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>다시 시도</button>
    </div>
  );
}
```

### 2.2.7 메타데이터 (SEO)

**페이지별 메타데이터 설정**

```typescript
// app/about/page.tsx
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: '회사 소개',
  description: '우리 회사를 소개합니다',
  openGraph: {
    title: '회사 소개',
    description: '우리 회사를 소개합니다',
    images: ['/og-image.jpg'],
  },
};

export default function AboutPage() {
  return <h1>회사 소개</h1>;
}
```

**동적 메타데이터**

```typescript
// app/blog/[slug]/page.tsx
export async function generateMetadata({
  params,
}: {
  params: { slug: string };
}): Promise<Metadata> {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());

  return {
    title: post.title,
    description: post.excerpt,
  };
}

export default function BlogPostPage({ params }: { params: { slug: string } }) {
  return <h1>블로그 포스트</h1>;
}
```

## 2.3 `public` 폴더

**정적 파일 저장소**

```
public/
├── images/
│   ├── logo.png
│   └── hero.jpg
├── fonts/
│   └── custom-font.woff2
└── favicon.ico
```

### 사용 방법

```typescript
// 이미지 사용
import Image from 'next/image';

export default function Home() {
  return (
    <div>
      {/* /public/images/logo.png */}
      <Image src="/images/logo.png" alt="로고" width={200} height={100} />

      {/* /public/hero.jpg */}
      <img src="/hero.jpg" alt="히어로" />
    </div>
  );
}
```

**주의사항:**
- `public` 폴더는 루트 `/`에 매핑됨
- `/public/images/logo.png` → `/images/logo.png`로 접근
- 빌드 시 최적화되지 않으므로 이미지는 `next/image` 사용 권장

## 2.4 `components` 폴더

**재사용 가능한 컴포넌트**

```
components/
├── ui/
│   ├── Button.tsx
│   ├── Input.tsx
│   └── Card.tsx
├── layout/
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── Sidebar.tsx
└── features/
    ├── ProductCard.tsx
    └── UserProfile.tsx
```

**예시:**

```typescript
// components/ui/Button.tsx
export default function Button({
  children,
  onClick,
}: {
  children: React.ReactNode;
  onClick?: () => void;
}) {
  return (
    <button
      onClick={onClick}
      className="px-4 py-2 bg-blue-500 text-white rounded"
    >
      {children}
    </button>
  );
}
```

```typescript
// app/page.tsx에서 사용
import Button from '@/components/ui/Button';

export default function Home() {
  return (
    <div>
      <h1>홈페이지</h1>
      <Button onClick={() => alert('클릭!')}>
        클릭하세요
      </Button>
    </div>
  );
}
```

## 2.5 `lib` 폴더

**유틸리티 함수, 헬퍼**

```
lib/
├── utils.ts          # 일반 유틸리티
├── api.ts            # API 호출 함수
├── db.ts             # 데이터베이스 클라이언트
└── constants.ts      # 상수
```

**예시:**

```typescript
// lib/utils.ts
export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('ko-KR').format(date);
}

export function cn(...classes: string[]): string {
  return classes.filter(Boolean).join(' ');
}
```

```typescript
// lib/api.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL;

export async function fetchPosts() {
  const res = await fetch(`${API_URL}/posts`);
  if (!res.ok) throw new Error('Failed to fetch posts');
  return res.json();
}
```

## 2.6 설정 파일들

### 2.6.1 `next.config.js`

**Next.js 설정 파일**

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 이미지 도메인 허용
  images: {
    domains: ['example.com', 'cdn.example.com'],
  },

  // 환경변수
  env: {
    CUSTOM_KEY: 'my-value',
  },

  // 리다이렉트
  async redirects() {
    return [
      {
        source: '/old-blog/:slug',
        destination: '/blog/:slug',
        permanent: true,
      },
    ];
  },

  // 리라이트 (프록시)
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://api.example.com/:path*',
      },
    ];
  },
};

module.exports = nextConfig;
```

### 2.6.2 환경변수 (`.env.local`)

**환경변수 파일**

```bash
# .env.local (Git에 커밋하지 않음!)

# 서버에서만 접근 가능
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
API_SECRET_KEY=super-secret-key

# 클라이언트에서도 접근 가능 (NEXT_PUBLIC_ 접두사)
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_GA_ID=G-XXXXXXXXXX
```

**사용 방법:**

```typescript
// Server Component (app/page.tsx)
export default async function HomePage() {
  // 서버 전용 환경변수 (안전!)
  const dbUrl = process.env.DATABASE_URL;
  const apiKey = process.env.API_SECRET_KEY;

  // 클라이언트 공개 환경변수
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;

  return <div>...</div>;
}
```

```typescript
// Client Component
'use client';

export default function ClientComponent() {
  // 클라이언트에서는 NEXT_PUBLIC_만 접근 가능
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;  // ✅
  const dbUrl = process.env.DATABASE_URL;          // ❌ undefined

  return <div>...</div>;
}
```

**환경변수 파일 종류:**

| 파일 | 용도 | Git 커밋 |
|------|------|----------|
| `.env` | 모든 환경 공통 기본값 | ✅ 가능 |
| `.env.local` | 로컬 개발 (우선순위 높음) | ❌ 금지 |
| `.env.development` | 개발 환경 전용 | ✅ 가능 |
| `.env.production` | 프로덕션 환경 전용 | ✅ 가능 |

### 2.6.3 `tsconfig.json`

**TypeScript 설정**

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"]  // @/로 임포트 가능
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

**`@/` 별칭 사용:**

```typescript
// ❌ 상대 경로 (복잡함)
import Button from '../../../components/ui/Button';

// ✅ 절대 경로 (깔끔함)
import Button from '@/components/ui/Button';
```

## 2.7 실전 프로젝트 구조 예시

```
my-ecommerce-app/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                    # 홈
│   ├── (auth)/                     # 그룹 라우트 (URL에 포함 안 됨)
│   │   ├── login/
│   │   │   └── page.tsx            # /login
│   │   └── register/
│   │       └── page.tsx            # /register
│   ├── products/
│   │   ├── page.tsx                # /products
│   │   └── [id]/
│   │       └── page.tsx            # /products/123
│   ├── cart/
│   │   └── page.tsx                # /cart
│   └── api/
│       └── checkout/
│           └── route.ts            # /api/checkout
├── components/
│   ├── ui/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   └── Card.tsx
│   ├── layout/
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   └── Navigation.tsx
│   └── features/
│       ├── ProductCard.tsx
│       ├── CartItem.tsx
│       └── CheckoutForm.tsx
├── lib/
│   ├── utils.ts
│   ├── api.ts
│   ├── db.ts
│   └── constants.ts
├── types/
│   ├── product.ts
│   └── user.ts
├── public/
│   ├── images/
│   └── icons/
├── .env.local
├── next.config.js
└── package.json
```

## 2.8 폴더 구조 베스트 프랙티스

### 1. **라우팅은 `app` 폴더에만**
```
✅ app/products/page.tsx
❌ components/products/page.tsx
```

### 2. **재사용 컴포넌트는 `components`에**
```
✅ components/ui/Button.tsx
❌ app/ui/Button.tsx
```

### 3. **유틸리티는 `lib`에**
```
✅ lib/utils.ts
❌ app/utils.ts
```

### 4. **타입 정의는 `types`에 (선택)**
```typescript
// types/product.ts
export interface Product {
  id: number;
  name: string;
  price: number;
}
```

### 5. **그룹 라우트로 정리**
```
app/
├── (marketing)/        # URL에 포함 안 됨
│   ├── about/
│   └── contact/
└── (shop)/             # URL에 포함 안 됨
    ├── products/
    └── cart/
```

## 핵심 요약

1. **`app` 폴더가 핵심**
   - 폴더 구조 = URL 구조
   - `page.tsx`로 페이지 생성
   - `layout.tsx`로 공통 레이아웃

2. **특수 파일명 활용**
   - `loading.tsx`: 로딩 UI
   - `error.tsx`: 에러 UI
   - `not-found.tsx`: 404 페이지

3. **정적 파일은 `public`에**
   - `/public/images/logo.png` → `/images/logo.png`

4. **환경변수 관리**
   - `.env.local`: 로컬 개발용 (Git 제외)
   - `NEXT_PUBLIC_`: 클라이언트 공개
   - 일반 변수: 서버 전용

5. **절대 경로 사용**
   - `@/`로 깔끔한 임포트

## 연습 문제

### 문제 1
다음 URL에 매핑되는 파일 경로를 작성하세요:
1. `/`
2. `/about`
3. `/blog/hello-world`
4. `/dashboard/settings`

<details>
<summary>정답</summary>

1. `app/page.tsx`
2. `app/about/page.tsx`
3. `app/blog/[slug]/page.tsx` (slug = "hello-world")
4. `app/dashboard/settings/page.tsx`
</details>

### 문제 2
환경변수를 설정하고 사용하는 코드를 작성하세요.
- API URL: `https://api.example.com` (클라이언트에서 접근 가능)
- API Key: `secret-key` (서버 전용)

<details>
<summary>정답</summary>

```bash
# .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
API_SECRET_KEY=secret-key
```

```typescript
// Server Component
export default async function ServerPage() {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;  // ✅
  const apiKey = process.env.API_SECRET_KEY;       // ✅

  return <div>...</div>;
}

// Client Component
'use client';
export default function ClientPage() {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;  // ✅
  const apiKey = process.env.API_SECRET_KEY;       // ❌ undefined

  return <div>...</div>;
}
```
</details>

### 문제 3
다음 요구사항을 만족하는 프로젝트 구조를 설계하세요:
- 홈페이지 (`/`)
- 소개 페이지 (`/about`)
- 블로그 목록 (`/blog`)
- 블로그 상세 (`/blog/[slug]`)
- 공통 헤더/푸터
- 블로그에만 사이드바

<details>
<summary>정답</summary>

```
app/
├── layout.tsx              # 공통 헤더/푸터
├── page.tsx                # /
├── about/
│   └── page.tsx            # /about
└── blog/
    ├── layout.tsx          # 블로그 사이드바
    ├── page.tsx            # /blog
    └── [slug]/
        └── page.tsx        # /blog/:slug

components/
├── layout/
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── BlogSidebar.tsx
```

```typescript
// app/layout.tsx
import Header from '@/components/layout/Header';
import Footer from '@/components/layout/Footer';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Header />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}

// app/blog/layout.tsx
import BlogSidebar from '@/components/layout/BlogSidebar';

export default function BlogLayout({ children }) {
  return (
    <div className="flex">
      <aside><BlogSidebar /></aside>
      <div>{children}</div>
    </div>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- 실제로 페이지 만들기
- JSX 문법 복습
- 컴포넌트 작성
- 스타일링 기초

[Chapter 3. 첫 페이지 만들기 →](chapter03-first-page.md)