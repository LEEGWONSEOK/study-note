# Chapter 5. 네비게이션

## 5.1 `<Link>` 컴포넌트

Next.js에서 페이지 간 이동은 `<Link>` 컴포넌트를 사용합니다. 일반 `<a>` 태그와 달리 **페이지 새로고침 없이** 이동합니다 (SPA 방식).

### 5.1.1 기본 사용법

```typescript
import Link from 'next/link';

export default function Navigation() {
  return (
    <nav>
      <Link href="/">홈</Link>
      <Link href="/about">소개</Link>
      <Link href="/products">상품</Link>
      <Link href="/contact">연락처</Link>
    </nav>
  );
}
```

### 5.1.2 `<a>` 태그 vs `<Link>` 컴포넌트

```typescript
// ❌ 일반 <a> 태그 (페이지 새로고침 발생)
<a href="/about">소개</a>

// ✅ <Link> 컴포넌트 (SPA 방식, 빠름!)
<Link href="/about">소개</Link>
```

**차이점:**

| 특징 | `<a>` | `<Link>` |
|------|-------|----------|
| 페이지 새로고침 | ✅ 발생 | ❌ 없음 |
| JavaScript 번들 | 다시 다운로드 | 캐시 사용 |
| 상태 유지 | ❌ 초기화 | ✅ 유지 |
| 속도 | 느림 | 빠름 |
| Prefetch | ❌ | ✅ (자동) |

### 5.1.3 동적 라우트 링크

```typescript
// 단일 동적 파라미터
<Link href="/blog/hello-world">블로그 포스트</Link>
<Link href={`/blog/${slug}`}>블로그 포스트</Link>

// 여러 동적 파라미터
<Link href="/shop/electronics/laptop-123">상품 보기</Link>
<Link href={`/shop/${category}/${productId}`}>상품 보기</Link>

// 템플릿 리터럴 활용
const postId = 123;
<Link href={`/posts/${postId}`}>포스트 #{postId}</Link>
```

### 5.1.4 쿼리 파라미터

```typescript
// 방법 1: 문자열로 직접 작성
<Link href="/search?q=nextjs&category=tutorial">검색</Link>

// 방법 2: 객체 형태
<Link
  href={{
    pathname: '/search',
    query: { q: 'nextjs', category: 'tutorial' },
  }}
>
  검색
</Link>

// 방법 3: URLSearchParams 사용
const params = new URLSearchParams({
  q: 'nextjs',
  category: 'tutorial',
  page: '1',
});
<Link href={`/search?${params.toString()}`}>검색</Link>
```

### 5.1.5 스타일링

```typescript
// 방법 1: className 사용
<Link href="/about" className="text-blue-500 hover:underline">
  소개
</Link>

// 방법 2: 자식 요소에 스타일 적용
<Link href="/about">
  <span className="text-blue-500">소개</span>
</Link>

// 방법 3: 버튼으로 스타일링
<Link
  href="/products"
  className="px-4 py-2 bg-blue-500 text-white rounded"
>
  상품 보기
</Link>
```

### 5.1.6 활성 링크 표시

현재 페이지를 표시하려면 `usePathname` 훅을 사용합니다:

```typescript
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

export default function Navigation() {
  const pathname = usePathname();

  const links = [
    { href: '/', label: '홈' },
    { href: '/about', label: '소개' },
    { href: '/products', label: '상품' },
    { href: '/contact', label: '연락처' },
  ];

  return (
    <nav className="flex gap-4">
      {links.map((link) => {
        const isActive = pathname === link.href;

        return (
          <Link
            key={link.href}
            href={link.href}
            className={isActive ? 'font-bold text-blue-500' : 'text-gray-600'}
          >
            {link.label}
          </Link>
        );
      })}
    </nav>
  );
}
```

### 5.1.7 Prefetch (사전 로딩)

`<Link>`는 기본적으로 **뷰포트에 보이는 링크를 자동으로 사전 로딩**합니다.

```typescript
// 자동 prefetch (기본값)
<Link href="/about">소개</Link>

// prefetch 비활성화
<Link href="/about" prefetch={false}>
  소개
</Link>
```

**Prefetch 동작:**
```
1. 링크가 뷰포트에 보임
2. Next.js가 자동으로 해당 페이지 데이터 로딩 시작
3. 사용자가 클릭하면 즉시 표시 (이미 로딩됨!)
```

### 5.1.8 외부 링크

외부 사이트는 일반 `<a>` 태그를 사용합니다:

```typescript
// 외부 링크 - <a> 태그 사용
<a
  href="https://github.com"
  target="_blank"
  rel="noopener noreferrer"
>
  GitHub
</a>

// 내부 링크 - <Link> 사용
<Link href="/about">소개</Link>
```

## 5.2 `useRouter` 훅

프로그래밍 방식으로 페이지를 이동할 때 사용합니다.

### 5.2.1 기본 사용법

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function MyComponent() {
  const router = useRouter();

  const handleClick = () => {
    router.push('/dashboard');
  };

  return (
    <button onClick={handleClick}>
      대시보드로 이동
    </button>
  );
}
```

### 5.2.2 `router.push()` - 히스토리 추가

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function LoginButton() {
  const router = useRouter();

  const handleLogin = async () => {
    // 로그인 로직...
    const success = true;

    if (success) {
      // 히스토리에 추가 (뒤로가기 가능)
      router.push('/dashboard');
    }
  };

  return <button onClick={handleLogin}>로그인</button>;
}
```

### 5.2.3 `router.replace()` - 히스토리 교체

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function LogoutButton() {
  const router = useRouter();

  const handleLogout = () => {
    // 로그아웃 로직...

    // 현재 페이지를 교체 (뒤로가기 불가)
    router.replace('/login');
  };

  return <button onClick={handleLogout}>로그아웃</button>;
}
```

**`push` vs `replace`:**

```typescript
// push - 히스토리 추가
router.push('/page2');  // [/page1, /page2] - 뒤로가기 가능

// replace - 히스토리 교체
router.replace('/page2');  // [/page2] - 뒤로가기 불가
```

### 5.2.4 `router.back()` / `router.forward()`

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function NavigationButtons() {
  const router = useRouter();

  return (
    <div>
      <button onClick={() => router.back()}>뒤로가기</button>
      <button onClick={() => router.forward()}>앞으로가기</button>
    </div>
  );
}
```

### 5.2.5 `router.refresh()`

현재 페이지를 새로고침합니다 (서버 컴포넌트 재실행):

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function RefreshButton() {
  const router = useRouter();

  const handleRefresh = () => {
    // 현재 페이지 새로고침
    router.refresh();
  };

  return <button onClick={handleRefresh}>새로고침</button>;
}
```

### 5.2.6 동적 라우트로 이동

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function ProductList() {
  const router = useRouter();

  const products = [
    { id: 1, name: '노트북' },
    { id: 2, name: '마우스' },
  ];

  const handleProductClick = (productId: number) => {
    router.push(`/products/${productId}`);
  };

  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          <button onClick={() => handleProductClick(product.id)}>
            {product.name}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

### 5.2.7 쿼리 파라미터와 함께 이동

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function SearchForm() {
  const router = useRouter();

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();

    const formData = new FormData(e.currentTarget);
    const query = formData.get('query') as string;
    const category = formData.get('category') as string;

    // 쿼리 파라미터와 함께 이동
    const params = new URLSearchParams({
      q: query,
      category: category,
    });

    router.push(`/search?${params.toString()}`);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="query" type="text" placeholder="검색어" />
      <select name="category">
        <option value="all">전체</option>
        <option value="tutorial">튜토리얼</option>
        <option value="news">뉴스</option>
      </select>
      <button type="submit">검색</button>
    </form>
  );
}
```

## 5.3 `usePathname` 훅

현재 경로를 가져옵니다.

```typescript
'use client';

import { usePathname } from 'next/navigation';

export default function CurrentPath() {
  const pathname = usePathname();
  // /products → "/products"
  // /blog/hello → "/blog/hello"

  return <div>현재 경로: {pathname}</div>;
}
```

### 활용 예제: 브레드크럼

```typescript
'use client';

import { usePathname } from 'next/navigation';
import Link from 'next/link';

export default function Breadcrumb() {
  const pathname = usePathname();
  const paths = pathname.split('/').filter(Boolean);

  return (
    <nav>
      <Link href="/">홈</Link>
      {paths.map((path, index) => {
        const href = `/${paths.slice(0, index + 1).join('/')}`;
        return (
          <span key={href}>
            {' > '}
            <Link href={href}>{path}</Link>
          </span>
        );
      })}
    </nav>
  );
}

// URL: /products/electronics/laptops
// 결과: 홈 > products > electronics > laptops
```

## 5.4 `useSearchParams` 훅

쿼리 파라미터를 가져옵니다.

```typescript
'use client';

import { useSearchParams } from 'next/navigation';

export default function SearchResults() {
  const searchParams = useSearchParams();

  const query = searchParams.get('q');
  const category = searchParams.get('category');
  const page = searchParams.get('page');

  // URL: /search?q=nextjs&category=tutorial&page=1
  // query = "nextjs"
  // category = "tutorial"
  // page = "1"

  return (
    <div>
      <p>검색어: {query}</p>
      <p>카테고리: {category}</p>
      <p>페이지: {page}</p>
    </div>
  );
}
```

### 쿼리 파라미터 업데이트

```typescript
'use client';

import { useSearchParams, useRouter } from 'next/navigation';

export default function FilterButtons() {
  const router = useRouter();
  const searchParams = useSearchParams();

  const updateFilter = (category: string) => {
    const params = new URLSearchParams(searchParams.toString());
    params.set('category', category);
    router.push(`/products?${params.toString()}`);
  };

  return (
    <div>
      <button onClick={() => updateFilter('electronics')}>
        전자제품
      </button>
      <button onClick={() => updateFilter('furniture')}>
        가구
      </button>
    </div>
  );
}
```

## 5.5 Redirect

서버 컴포넌트에서 리다이렉트할 때 사용합니다.

### 5.5.1 `redirect()` 함수

```typescript
import { redirect } from 'next/navigation';

export default async function ProfilePage() {
  const user = await getUser();

  // 로그인하지 않았으면 리다이렉트
  if (!user) {
    redirect('/login');
  }

  return <div>환영합니다, {user.name}님!</div>;
}
```

### 5.5.2 조건부 리다이렉트

```typescript
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const user = await getUser();

  // 권한 체크
  if (!user) {
    redirect('/login');
  }

  if (user.role !== 'admin') {
    redirect('/unauthorized');
  }

  return <div>관리자 대시보드</div>;
}
```

### 5.5.3 `notFound()` - 404 페이지

```typescript
import { notFound } from 'next/navigation';

interface Props {
  params: { id: string };
}

export default async function ProductPage({ params }: Props) {
  const product = await getProduct(params.id);

  if (!product) {
    notFound();  // 404 페이지로
  }

  return <div>{product.name}</div>;
}
```

## 5.6 실전 예제: 네비게이션 바

```typescript
// components/Navigation.tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

export default function Navigation() {
  const pathname = usePathname();

  const links = [
    { href: '/', label: '홈' },
    { href: '/products', label: '상품' },
    { href: '/about', label: '소개' },
    { href: '/contact', label: '연락처' },
  ];

  return (
    <nav className="bg-gray-800 text-white p-4">
      <div className="container mx-auto flex gap-6">
        {links.map((link) => {
          const isActive = pathname === link.href;

          return (
            <Link
              key={link.href}
              href={link.href}
              className={`
                px-3 py-2 rounded transition
                ${isActive
                  ? 'bg-blue-500 font-bold'
                  : 'hover:bg-gray-700'
                }
              `}
            >
              {link.label}
            </Link>
          );
        })}
      </div>
    </nav>
  );
}
```

## 5.7 실전 예제: 로그인 후 리다이렉트

```typescript
// app/login/page.tsx
'use client';

import { useRouter, useSearchParams } from 'next/navigation';
import { useState } from 'react';

export default function LoginPage() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  // 로그인 후 돌아갈 페이지
  const callbackUrl = searchParams.get('callbackUrl') || '/dashboard';

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    // 로그인 로직...
    const success = true;

    if (success) {
      // 로그인 성공 → 원래 가려던 페이지로
      router.push(callbackUrl);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <h1>로그인</h1>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="이메일"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="비밀번호"
      />
      <button type="submit">로그인</button>
    </form>
  );
}
```

## 5.8 네비게이션 모범 사례

### 1. 내부 링크는 `<Link>` 사용

```typescript
✅ <Link href="/about">소개</Link>
❌ <a href="/about">소개</a>
```

### 2. 외부 링크는 `<a>` 사용

```typescript
✅ <a href="https://example.com" target="_blank" rel="noopener noreferrer">
❌ <Link href="https://example.com">
```

### 3. 버튼 클릭 시 이동은 `useRouter`

```typescript
✅ const router = useRouter();
   <button onClick={() => router.push('/dashboard')}>

❌ <Link href="/dashboard">
     <button>  // Link 안에 button은 안티패턴
```

### 4. 인증 체크는 서버에서

```typescript
// ✅ 서버 컴포넌트에서
export default async function ProtectedPage() {
  const user = await getUser();
  if (!user) redirect('/login');
  return <div>...</div>;
}

// ❌ 클라이언트에서 (보안 취약)
'use client';
export default function ProtectedPage() {
  const router = useRouter();
  if (!user) router.push('/login');  // 늦음!
}
```

### 5. 로딩 상태 표시

```typescript
'use client';

import { useRouter } from 'next/navigation';
import { useState } from 'react';

export default function SubmitButton() {
  const router = useRouter();
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async () => {
    setIsLoading(true);
    // API 호출...
    await saveData();
    router.push('/success');
  };

  return (
    <button onClick={handleSubmit} disabled={isLoading}>
      {isLoading ? '처리 중...' : '제출'}
    </button>
  );
}
```

## 핵심 요약

1. **`<Link>` 컴포넌트**
   - 내부 페이지 이동에 사용
   - 자동 prefetch
   - 페이지 새로고침 없음

2. **`useRouter` 훅**
   - 프로그래밍 방식 네비게이션
   - `push()`: 히스토리 추가
   - `replace()`: 히스토리 교체
   - `back()`, `refresh()`

3. **현재 경로 정보**
   - `usePathname()`: 현재 경로
   - `useSearchParams()`: 쿼리 파라미터

4. **Redirect**
   - `redirect()`: 서버 컴포넌트에서
   - `notFound()`: 404 페이지

5. **모범 사례**
   - 내부 링크: `<Link>`
   - 외부 링크: `<a>`
   - 버튼 이동: `useRouter`

## 연습 문제

### 문제 1
홈, 소개, 상품 페이지로 이동하는 네비게이션 바를 만들고, 현재 페이지를 강조 표시하세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

export default function Navigation() {
  const pathname = usePathname();

  const links = [
    { href: '/', label: '홈' },
    { href: '/about', label: '소개' },
    { href: '/products', label: '상품' },
  ];

  return (
    <nav className="flex gap-4">
      {links.map((link) => (
        <Link
          key={link.href}
          href={link.href}
          className={pathname === link.href ? 'font-bold' : ''}
        >
          {link.label}
        </Link>
      ))}
    </nav>
  );
}
```
</details>

### 문제 2
로그인 버튼을 만들고, 로그인 성공 시 대시보드로 이동하세요 (히스토리 교체).

<details>
<summary>정답</summary>

```typescript
'use client';

import { useRouter } from 'next/navigation';

export default function LoginButton() {
  const router = useRouter();

  const handleLogin = async () => {
    // 로그인 로직 (가정)
    const success = true;

    if (success) {
      // 히스토리 교체 (뒤로가기 방지)
      router.replace('/dashboard');
    }
  };

  return <button onClick={handleLogin}>로그인</button>;
}
```
</details>

### 문제 3
검색 폼을 만들고, 제출 시 `/search?q=검색어` 페이지로 이동하세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import { useRouter } from 'next/navigation';
import { useState } from 'react';

export default function SearchForm() {
  const router = useRouter();
  const [query, setQuery] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    if (query.trim()) {
      router.push(`/search?q=${encodeURIComponent(query)}`);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="검색어 입력"
      />
      <button type="submit">검색</button>
    </form>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- 레이아웃과 템플릿
- 중첩 레이아웃
- 로딩 UI
- 에러 처리

[Chapter 6. 레이아웃과 템플릿 →](chapter06-layouts-templates.md)