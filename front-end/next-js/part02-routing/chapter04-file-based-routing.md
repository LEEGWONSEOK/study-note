# Chapter 4. File-based Routing

## 4.1 파일 기반 라우팅 개념

Next.js의 가장 큰 특징 중 하나는 **파일 기반 라우팅**입니다. 코드로 라우팅을 정의하는 것이 아니라, **폴더와 파일 구조가 곧 URL**이 됩니다.

### 4.1.1 전통적인 라우팅 vs Next.js 라우팅

**React Router (전통적인 방식):**

```typescript
// 라우터 설정 파일이 별도로 필요
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/blog/:slug" element={<BlogPost />} />
      </Routes>
    </BrowserRouter>
  );
}
```

**Next.js (파일 기반):**

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
└── blog/
    └── [slug]/
        └── page.tsx      → /blog/:slug
```

설정 파일 없이 **폴더와 파일만 생성**하면 라우팅이 자동으로 설정됩니다!

### 4.1.2 기본 원리

```
폴더 이름 = URL 경로
page.tsx = 해당 경로의 페이지
```

## 4.2 기본 라우팅

### 4.2.1 정적 라우트

**가장 간단한 형태:**

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
├── products/
│   └── page.tsx          → /products
└── contact/
    └── page.tsx          → /contact
```

**예제:**

```typescript
// app/page.tsx
export default function HomePage() {
  return <h1>홈페이지</h1>;
}
```

```typescript
// app/about/page.tsx
export default function AboutPage() {
  return <h1>회사 소개</h1>;
}
```

```typescript
// app/products/page.tsx
export default function ProductsPage() {
  return <h1>상품 목록</h1>;
}
```

### 4.2.2 중첩 라우트

**폴더 안에 폴더를 만들면 경로가 중첩됩니다:**

```
app/
└── blog/
    ├── page.tsx                    → /blog
    ├── categories/
    │   └── page.tsx                → /blog/categories
    └── archive/
        └── page.tsx                → /blog/archive
```

**예제:**

```typescript
// app/blog/page.tsx
export default function BlogPage() {
  return <h1>블로그</h1>;
}
```

```typescript
// app/blog/categories/page.tsx
export default function CategoriesPage() {
  return <h1>블로그 카테고리</h1>;
}
```

### 4.2.3 더 복잡한 중첩

```
app/
└── dashboard/
    ├── page.tsx                    → /dashboard
    ├── analytics/
    │   └── page.tsx                → /dashboard/analytics
    ├── users/
    │   ├── page.tsx                → /dashboard/users
    │   └── [id]/
    │       └── page.tsx            → /dashboard/users/:id
    └── settings/
        ├── page.tsx                → /dashboard/settings
        └── profile/
            └── page.tsx            → /dashboard/settings/profile
```

## 4.3 동적 라우트

실제 애플리케이션에서는 동적인 URL이 필요합니다.
- `/blog/hello-world`
- `/products/123`
- `/users/john`

### 4.3.1 단일 동적 세그먼트 `[param]`

**폴더 이름을 `[]`로 감싸면 동적 라우트가 됩니다:**

```
app/
└── blog/
    └── [slug]/
        └── page.tsx      → /blog/:slug
```

**예제:**

```typescript
// app/blog/[slug]/page.tsx
interface Props {
  params: {
    slug: string;
  };
}

export default function BlogPostPage({ params }: Props) {
  return (
    <div>
      <h1>블로그 포스트</h1>
      <p>슬러그: {params.slug}</p>
    </div>
  );
}
```

**URL 예시:**
- `/blog/hello-world` → `params.slug = "hello-world"`
- `/blog/nextjs-guide` → `params.slug = "nextjs-guide"`
- `/blog/123` → `params.slug = "123"`

### 4.3.2 실전 예제: 상품 상세 페이지

```typescript
// app/products/[id]/page.tsx
interface Props {
  params: {
    id: string;
  };
}

// 임시 데이터
const products = {
  '1': { name: '노트북', price: 1000000 },
  '2': { name: '마우스', price: 30000 },
  '3': { name: '키보드', price: 80000 },
};

export default function ProductDetailPage({ params }: Props) {
  const product = products[params.id as keyof typeof products];

  if (!product) {
    return <div>상품을 찾을 수 없습니다.</div>;
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>가격: {product.price.toLocaleString()}원</p>
    </div>
  );
}
```

### 4.3.3 여러 동적 세그먼트

```
app/
└── shop/
    └── [category]/
        └── [productId]/
            └── page.tsx      → /shop/:category/:productId
```

```typescript
// app/shop/[category]/[productId]/page.tsx
interface Props {
  params: {
    category: string;
    productId: string;
  };
}

export default function ProductPage({ params }: Props) {
  return (
    <div>
      <h1>상품 상세</h1>
      <p>카테고리: {params.category}</p>
      <p>상품 ID: {params.productId}</p>
    </div>
  );
}
```

**URL 예시:**
- `/shop/electronics/laptop-123`
  - `params.category = "electronics"`
  - `params.productId = "laptop-123"`

### 4.3.4 Catch-all 세그먼트 `[...param]`

**모든 하위 경로를 캐치합니다:**

```
app/
└── docs/
    └── [...slug]/
        └── page.tsx      → /docs/*
```

```typescript
// app/docs/[...slug]/page.tsx
interface Props {
  params: {
    slug: string[];  // 배열로 받음!
  };
}

export default function DocsPage({ params }: Props) {
  return (
    <div>
      <h1>문서</h1>
      <p>경로: {params.slug.join(' / ')}</p>
    </div>
  );
}
```

**URL 예시:**
- `/docs/getting-started`
  - `params.slug = ["getting-started"]`
- `/docs/api/users/create`
  - `params.slug = ["api", "users", "create"]`
- `/docs/a/b/c/d/e`
  - `params.slug = ["a", "b", "c", "d", "e"]`

### 4.3.5 Optional Catch-all 세그먼트 `[[...param]]`

**루트 경로도 포함합니다:**

```
app/
└── shop/
    └── [[...slug]]/
        └── page.tsx
        // /shop
        // /shop/electronics
        // /shop/electronics/laptops
```

```typescript
// app/shop/[[...slug]]/page.tsx
interface Props {
  params: {
    slug?: string[];  // optional!
  };
}

export default function ShopPage({ params }: Props) {
  if (!params.slug) {
    return <h1>전체 상품</h1>;
  }

  return (
    <div>
      <h1>상품 카테고리</h1>
      <p>경로: {params.slug.join(' > ')}</p>
    </div>
  );
}
```

**URL 예시:**
- `/shop` → `params.slug = undefined`
- `/shop/electronics` → `params.slug = ["electronics"]`
- `/shop/electronics/laptops` → `params.slug = ["electronics", "laptops"]`

## 4.4 라우트 그룹 `(folder)`

**URL에 포함되지 않는 폴더를 만들 수 있습니다.** 코드 구조 정리용!

```
app/
├── (marketing)/          # URL에 포함 안 됨
│   ├── about/
│   │   └── page.tsx      → /about
│   └── contact/
│       └── page.tsx      → /contact
├── (shop)/               # URL에 포함 안 됨
│   ├── products/
│   │   └── page.tsx      → /products
│   └── cart/
│       └── page.tsx      → /cart
└── page.tsx              → /
```

### 4.4.1 라우트 그룹의 용도

**1. 코드 구조 정리:**

```
app/
├── (auth)/
│   ├── login/page.tsx       → /login
│   └── register/page.tsx    → /register
└── (dashboard)/
    ├── analytics/page.tsx   → /analytics
    └── settings/page.tsx    → /settings
```

**2. 다른 레이아웃 적용:**

```typescript
// app/(marketing)/layout.tsx (마케팅 페이지용 레이아웃)
export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <nav>마케팅 네비게이션</nav>
      {children}
    </div>
  );
}

// app/(dashboard)/layout.tsx (대시보드용 레이아웃)
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <aside>대시보드 사이드바</aside>
      <main>{children}</main>
    </div>
  );
}
```

## 4.5 `page.tsx` 파일의 역할

### 4.5.1 페이지 컴포넌트가 되는 조건

**`page.tsx` 파일만 실제 페이지가 됩니다:**

```
app/
├── blog/
│   ├── page.tsx          ✅ /blog (페이지)
│   ├── Header.tsx        ❌ 라우트 아님 (일반 컴포넌트)
│   └── [slug]/
│       └── page.tsx      ✅ /blog/:slug (페이지)
```

### 4.5.2 페이지 Props

페이지 컴포넌트는 특별한 Props를 받습니다:

```typescript
interface PageProps {
  params: { [key: string]: string };        // 동적 라우트 파라미터
  searchParams: { [key: string]: string };  // 쿼리 파라미터
}

// app/products/[id]/page.tsx
export default function ProductPage({
  params,
  searchParams,
}: PageProps) {
  // URL: /products/123?sort=price&order=asc

  console.log(params.id);           // "123"
  console.log(searchParams.sort);   // "price"
  console.log(searchParams.order);  // "asc"

  return <div>상품 페이지</div>;
}
```

### 4.5.3 searchParams 활용 예제

```typescript
// app/products/page.tsx
interface Props {
  searchParams: {
    category?: string;
    sort?: string;
    page?: string;
  };
}

const products = [
  { id: 1, name: '노트북', category: 'electronics' },
  { id: 2, name: '마우스', category: 'electronics' },
  { id: 3, name: '의자', category: 'furniture' },
];

export default function ProductsPage({ searchParams }: Props) {
  // URL: /products?category=electronics&sort=name&page=1

  let filteredProducts = products;

  // 카테고리 필터링
  if (searchParams.category) {
    filteredProducts = filteredProducts.filter(
      p => p.category === searchParams.category
    );
  }

  // 정렬
  if (searchParams.sort === 'name') {
    filteredProducts.sort((a, b) => a.name.localeCompare(b.name));
  }

  return (
    <div>
      <h1>상품 목록</h1>
      <p>카테고리: {searchParams.category || '전체'}</p>
      <p>페이지: {searchParams.page || '1'}</p>

      <ul>
        {filteredProducts.map(product => (
          <li key={product.id}>{product.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

## 4.6 라우팅 우선순위

여러 라우트가 같은 URL과 매칭될 때 우선순위가 있습니다:

```
1. 정적 라우트
2. 동적 라우트
3. Catch-all 라우트
```

**예시:**

```
app/
├── blog/
│   ├── page.tsx                → /blog (우선순위 1)
│   ├── featured/
│   │   └── page.tsx            → /blog/featured (우선순위 1)
│   ├── [slug]/
│   │   └── page.tsx            → /blog/:slug (우선순위 2)
│   └── [...slug]/
│       └── page.tsx            → /blog/* (우선순위 3)
```

**매칭 예시:**
- `/blog` → `blog/page.tsx`
- `/blog/featured` → `blog/featured/page.tsx` (정적이 우선)
- `/blog/hello` → `blog/[slug]/page.tsx`
- `/blog/a/b/c` → `blog/[...slug]/page.tsx`

## 4.7 실전 예제: 블로그 라우팅

```
app/
└── blog/
    ├── page.tsx                    → /blog (블로그 홈)
    ├── [slug]/
    │   └── page.tsx                → /blog/:slug (포스트 상세)
    └── category/
        └── [name]/
            └── page.tsx            → /blog/category/:name (카테고리)
```

### 블로그 홈

```typescript
// app/blog/page.tsx
export const metadata = {
  title: '블로그',
};

const posts = [
  { id: 1, slug: 'nextjs-intro', title: 'Next.js 소개', category: 'tutorial' },
  { id: 2, slug: 'react-hooks', title: 'React Hooks', category: 'react' },
  { id: 3, slug: 'typescript-basics', title: 'TypeScript 기초', category: 'typescript' },
];

export default function BlogPage() {
  return (
    <div>
      <h1>블로그</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>
            <a href={`/blog/${post.slug}`}>{post.title}</a>
            <span> - {post.category}</span>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 포스트 상세

```typescript
// app/blog/[slug]/page.tsx
interface Props {
  params: { slug: string };
}

const posts = {
  'nextjs-intro': {
    title: 'Next.js 소개',
    content: 'Next.js는...',
    date: '2024-01-15'
  },
  'react-hooks': {
    title: 'React Hooks',
    content: 'React Hooks는...',
    date: '2024-01-20'
  },
};

export async function generateMetadata({ params }: Props) {
  const post = posts[params.slug as keyof typeof posts];
  return {
    title: post.title,
  };
}

export default function BlogPostPage({ params }: Props) {
  const post = posts[params.slug as keyof typeof posts];

  if (!post) {
    return <div>포스트를 찾을 수 없습니다.</div>;
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <time>{post.date}</time>
      <p>{post.content}</p>
    </article>
  );
}
```

### 카테고리별 포스트

```typescript
// app/blog/category/[name]/page.tsx
interface Props {
  params: { name: string };
}

const posts = [
  { id: 1, slug: 'nextjs-intro', title: 'Next.js 소개', category: 'tutorial' },
  { id: 2, slug: 'react-hooks', title: 'React Hooks', category: 'react' },
];

export default function CategoryPage({ params }: Props) {
  const categoryPosts = posts.filter(
    post => post.category === params.name
  );

  return (
    <div>
      <h1>{params.name} 카테고리</h1>
      <ul>
        {categoryPosts.map(post => (
          <li key={post.id}>
            <a href={`/blog/${post.slug}`}>{post.title}</a>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## 4.8 라우팅 모범 사례

### 1. 의미 있는 URL 사용

```
✅ /products/electronics/laptop-123
❌ /p/e/l123
```

### 2. 복수형 사용

```
✅ /products/:id
✅ /users/:id
❌ /product/:id
❌ /user/:id
```

### 3. 소문자와 하이픈 사용

```
✅ /blog/nextjs-tutorial
❌ /blog/NextJS_Tutorial
```

### 4. 깊이 제한 (3단계 이하 권장)

```
✅ /products/electronics/laptops
⚠️ /products/electronics/laptops/gaming/high-end
```

### 5. 라우트 그룹으로 정리

```
app/
├── (public)/
│   ├── page.tsx
│   └── about/
└── (private)/
    ├── dashboard/
    └── settings/
```

## 핵심 요약

1. **파일 = 라우트**
   - 폴더 구조가 곧 URL
   - `page.tsx`만 실제 페이지가 됨

2. **동적 라우트**
   - `[param]`: 단일 동적 세그먼트
   - `[...param]`: Catch-all
   - `[[...param]]`: Optional catch-all

3. **라우트 그룹**
   - `(folder)`: URL에 포함 안 됨
   - 코드 정리, 레이아웃 분리용

4. **Props**
   - `params`: 동적 파라미터
   - `searchParams`: 쿼리 파라미터

5. **우선순위**
   - 정적 > 동적 > Catch-all

## 연습 문제

### 문제 1
다음 URL에 매핑되는 파일 구조를 작성하세요:
- `/`
- `/products`
- `/products/123`
- `/categories/electronics`
- `/docs/getting-started/installation`

<details>
<summary>정답</summary>

```
app/
├── page.tsx                              → /
├── products/
│   ├── page.tsx                          → /products
│   └── [id]/
│       └── page.tsx                      → /products/:id
├── categories/
│   └── [name]/
│       └── page.tsx                      → /categories/:name
└── docs/
    └── [...slug]/
        └── page.tsx                      → /docs/*
```
</details>

### 문제 2
동적 라우트로 사용자 프로필 페이지를 만드세요.
- URL: `/users/[username]`
- username을 화면에 표시

<details>
<summary>정답</summary>

```typescript
// app/users/[username]/page.tsx
interface Props {
  params: {
    username: string;
  };
}

export default function UserProfilePage({ params }: Props) {
  return (
    <div>
      <h1>{params.username}님의 프로필</h1>
      <p>사용자 이름: {params.username}</p>
    </div>
  );
}
```
</details>

### 문제 3
쿼리 파라미터로 검색 기능을 구현하세요.
- URL: `/search?q=nextjs&category=tutorial`
- 검색어와 카테고리를 화면에 표시

<details>
<summary>정답</summary>

```typescript
// app/search/page.tsx
interface Props {
  searchParams: {
    q?: string;
    category?: string;
  };
}

export default function SearchPage({ searchParams }: Props) {
  return (
    <div>
      <h1>검색 결과</h1>
      {searchParams.q && <p>검색어: {searchParams.q}</p>}
      {searchParams.category && <p>카테고리: {searchParams.category}</p>}

      {!searchParams.q && !searchParams.category && (
        <p>검색어를 입력해주세요.</p>
      )}
    </div>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- `<Link>` 컴포넌트로 페이지 이동
- `useRouter` 훅
- 프로그래밍 방식 네비게이션
- Redirect

[Chapter 5. 네비게이션 →](chapter05-navigation.md)