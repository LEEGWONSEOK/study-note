# Chapter 9. SSR, SSG, ISR 이해하기

## 9.1 렌더링 전략 개요

Next.js의 가장 큰 강점은 **페이지마다 다른 렌더링 전략**을 선택할 수 있다는 것입니다.

### 9.1.1 렌더링 전략 비교

| 전략 | 실행 시점 | 데이터 최신성 | 속도 | 사용 예시 |
|------|-----------|---------------|------|-----------|
| **SSG** | 빌드 시 | 정적 | 최고 | 블로그, 문서 |
| **SSR** | 요청마다 | 실시간 | 중간 | 대시보드, 개인화 |
| **ISR** | 빌드 + 주기적 | 거의 최신 | 빠름 | 뉴스, 상품 |
| **CSR** | 클라이언트 | 실시간 | 느림 | 상호작용 많은 UI |

## 9.2 SSG (Static Site Generation)

**빌드 시점에 HTML을 생성**하고, 모든 요청에 동일한 HTML을 제공합니다.

### 9.2.1 기본 SSG

```typescript
// app/about/page.tsx
export default function AboutPage() {
  return (
    <div>
      <h1>회사 소개</h1>
      <p>우리는 최고의 회사입니다.</p>
    </div>
  );
}

// 빌드 시:
// npm run build → /about.html 생성
// 모든 요청에 같은 HTML 제공 (초고속!)
```

**특징:**
- ✅ 최고의 성능
- ✅ SEO 완벽
- ✅ CDN 캐싱 가능
- ❌ 빌드 시점의 데이터만 사용
- ❌ 자주 변경되는 데이터에 부적합

### 9.2.2 데이터 페칭이 있는 SSG

```typescript
// app/blog/page.tsx
export default async function BlogPage() {
  // 빌드 시점에 한 번만 실행
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return (
    <div>
      <h1>블로그</h1>
      {posts.map((post: any) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  );
}
```

**동작 방식:**
```
npm run build 실행 시:
1. fetch 실행 → API 호출
2. HTML 생성 → /blog.html 저장
3. 배포

사용자 요청 시:
1. /blog.html 즉시 제공 (초고속!)
```

### 9.2.3 동적 라우트의 SSG

```typescript
// app/blog/[slug]/page.tsx
interface Props {
  params: { slug: string };
}

// 1. 어떤 slug들을 미리 생성할지 정의
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());

  return posts.map((post: any) => ({
    slug: post.slug,
  }));
}

// 2. 각 slug에 대한 페이지 생성
export default async function BlogPostPage({ params }: Props) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

**동작 방식:**
```
npm run build 실행 시:
1. generateStaticParams() 실행
   → ['nextjs-intro', 'react-hooks', 'typescript'] 반환
2. 각 slug에 대해 페이지 생성:
   - /blog/nextjs-intro.html
   - /blog/react-hooks.html
   - /blog/typescript.html
3. 배포

사용자가 /blog/nextjs-intro 요청:
→ 미리 생성된 HTML 즉시 제공!
```

### 9.2.4 SSG 사용 시나리오

```typescript
// ✅ SSG가 적합한 경우
- 블로그 포스트
- 마케팅 랜딩 페이지
- 제품 소개 페이지
- 도움말 문서
- 회사 소개 페이지
```

## 9.3 SSR (Server-Side Rendering)

**매 요청마다 서버에서 HTML을 생성**합니다.

### 9.3.1 기본 SSR

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  // 매 요청마다 실행!
  const res = await fetch('https://api.example.com/stats', {
    cache: 'no-store'  // 캐시 사용 안 함
  });
  const stats = await res.json();

  return (
    <div>
      <h1>대시보드</h1>
      <p>오늘 방문자: {stats.visitors}</p>
      <p>매출: {stats.revenue}원</p>
    </div>
  );
}
```

**동작 방식:**
```
사용자가 /dashboard 요청:
1. 서버에서 fetch 실행 → API 호출
2. 데이터로 HTML 생성
3. HTML을 브라우저로 전송
4. 브라우저에서 HTML 표시
5. JavaScript 실행 (Hydration)
```

### 9.3.2 쿠키/헤더 기반 SSR

```typescript
// app/profile/page.tsx
import { cookies } from 'next/headers';

export default async function ProfilePage() {
  // 쿠키에서 사용자 ID 가져오기
  const cookieStore = cookies();
  const userId = cookieStore.get('userId')?.value;

  if (!userId) {
    return <div>로그인이 필요합니다</div>;
  }

  // 사용자별 데이터 페칭
  const user = await fetch(`https://api.example.com/users/${userId}`, {
    cache: 'no-store'
  }).then(res => res.json());

  return (
    <div>
      <h1>{user.name}님의 프로필</h1>
      <p>이메일: {user.email}</p>
    </div>
  );
}
```

### 9.3.3 검색 파라미터 기반 SSR

```typescript
// app/products/page.tsx
interface Props {
  searchParams: {
    category?: string;
    sort?: string;
    page?: string;
  };
}

export default async function ProductsPage({ searchParams }: Props) {
  // 쿼리 파라미터로 API 호출
  const params = new URLSearchParams({
    category: searchParams.category || 'all',
    sort: searchParams.sort || 'name',
    page: searchParams.page || '1',
  });

  const products = await fetch(
    `https://api.example.com/products?${params}`,
    { cache: 'no-store' }
  ).then(res => res.json());

  return (
    <div>
      <h1>상품 목록</h1>
      <p>카테고리: {searchParams.category || '전체'}</p>
      {products.map((product: any) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

### 9.3.4 SSR 사용 시나리오

```typescript
// ✅ SSR이 적합한 경우
- 실시간 대시보드
- 사용자별 맞춤 콘텐츠
- 검색 결과 페이지
- 장바구니, 주문 내역
- 쿠키/세션 기반 페이지
```

## 9.4 ISR (Incremental Static Regeneration)

**SSG의 성능 + SSR의 최신 데이터**를 결합한 전략입니다.

### 9.4.1 기본 ISR

```typescript
// app/posts/page.tsx
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 }  // 60초마다 재생성
  });
  const posts = await res.json();

  return (
    <div>
      <h1>최신 포스트</h1>
      {posts.map((post: any) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
        </article>
      ))}
    </div>
  );
}
```

**동작 방식:**
```
빌드 시:
1. 정적 HTML 생성 (SSG처럼)
2. /posts.html 저장

첫 번째 요청 (0초):
→ 정적 HTML 제공 (빠름!)

60초 이내 요청들:
→ 계속 정적 HTML 제공 (빠름!)

60초 후 첫 요청 (61초):
→ 정적 HTML 제공 (빠름!)
→ 백그라운드에서 새 HTML 생성 시작

62초 요청:
→ 새로 생성된 HTML 제공 (최신 데이터!)
```

### 9.4.2 페이지별 다른 Revalidate 시간

```typescript
// app/news/page.tsx - 5분마다
export default async function NewsPage() {
  const res = await fetch('https://api.example.com/news', {
    next: { revalidate: 300 }  // 5분
  });
  const news = await res.json();
  return <div>...</div>;
}

// app/products/page.tsx - 1시간마다
export default async function ProductsPage() {
  const res = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 }  // 1시간
  });
  const products = await res.json();
  return <div>...</div>;
}

// app/about/page.tsx - 1일마다
export default async function AboutPage() {
  const res = await fetch('https://api.example.com/about', {
    next: { revalidate: 86400 }  // 24시간
  });
  const data = await res.json();
  return <div>...</div>;
}
```

### 9.4.3 온디맨드 재검증

```typescript
// app/posts/[id]/page.tsx
export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`, {
    next: { tags: ['post', `post-${params.id}`] }
  }).then(res => res.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

```typescript
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const postId = request.nextUrl.searchParams.get('id');

  if (postId) {
    // 특정 포스트만 재검증
    revalidateTag(`post-${postId}`);
    return Response.json({ revalidated: true });
  }

  // 모든 포스트 재검증
  revalidateTag('post');
  return Response.json({ revalidated: true });
}

// 사용:
// POST /api/revalidate?id=123
// → /posts/123 페이지만 즉시 재생성
```

### 9.4.4 ISR 사용 시나리오

```typescript
// ✅ ISR이 적합한 경우
- 뉴스 사이트 (5-10분마다)
- 상품 목록 (30분-1시간마다)
- 블로그 (1시간-1일마다)
- 통계 페이지 (10분-1시간마다)
- 자주 변경되지만 실시간은 아닌 콘텐츠
```

## 9.5 CSR (Client-Side Rendering)

**브라우저에서 JavaScript로 렌더링**합니다.

### 9.5.1 기본 CSR

```typescript
// app/dashboard/page.tsx
'use client';

import { useState, useEffect } from 'react';

export default function DashboardPage() {
  const [stats, setStats] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('https://api.example.com/stats')
      .then(res => res.json())
      .then(data => {
        setStats(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>로딩 중...</div>;

  return (
    <div>
      <h1>대시보드</h1>
      <p>방문자: {stats.visitors}</p>
    </div>
  );
}
```

**동작 방식:**
```
1. 서버에서 빈 HTML 전송
2. 브라우저에서 JavaScript 다운로드
3. JavaScript 실행
4. API 호출
5. 데이터로 UI 렌더링
```

### 9.5.2 CSR 사용 시나리오

```typescript
// ✅ CSR이 적합한 경우
- 로그인 후 페이지 (SEO 불필요)
- 관리자 대시보드
- 실시간 채팅
- 복잡한 상호작용 (드래그앤드롭, 그래프)
```

## 9.6 렌더링 전략 선택 가이드

### 9.6.1 의사결정 트리

```
SEO가 중요한가?
├─ NO → CSR (Client Component)
└─ YES
    └─ 데이터가 자주 변경되는가?
        ├─ 거의 안 변함 → SSG
        ├─ 가끔 변함 (시간 단위) → ISR
        └─ 실시간 → SSR
```

### 9.6.2 실전 예시

```typescript
// 1. 홈페이지 - SSG (거의 안 변함)
// app/page.tsx
export default function HomePage() {
  return <div>환영합니다!</div>;
}

// 2. 블로그 목록 - ISR (하루에 몇 개 추가)
// app/blog/page.tsx
export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }  // 1시간
  }).then(res => res.json());
  return <div>...</div>;
}

// 3. 실시간 대시보드 - SSR
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const stats = await fetch('https://api.example.com/stats', {
    cache: 'no-store'
  }).then(res => res.json());
  return <div>...</div>;
}

// 4. 상호작용 많은 UI - CSR
// app/editor/page.tsx
'use client';
import { useState } from 'react';

export default function EditorPage() {
  const [content, setContent] = useState('');
  return <textarea value={content} onChange={e => setContent(e.target.value)} />;
}
```

## 9.7 하이브리드 전략

**하나의 페이지에 여러 전략 혼합 가능!**

### 9.7.1 Server + Client 컴포넌트

```typescript
// app/products/[id]/page.tsx (Server Component - SSR)
import AddToCartButton from './AddToCartButton';

export default async function ProductPage({ params }: { params: { id: string } }) {
  // 서버에서 실행
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    cache: 'no-store'
  }).then(res => res.json());

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.price}원</p>

      {/* Client Component (상호작용) */}
      <AddToCartButton productId={product.id} />
    </div>
  );
}
```

```typescript
// app/products/[id]/AddToCartButton.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function AddToCartButton({ productId }: { productId: number }) {
  const [count, setCount] = useState(1);

  const handleAddToCart = () => {
    fetch('/api/cart', {
      method: 'POST',
      body: JSON.stringify({ productId, count }),
    });
  };

  return (
    <div>
      <input
        type="number"
        value={count}
        onChange={(e) => setCount(Number(e.target.value))}
      />
      <button onClick={handleAddToCart}>장바구니 추가</button>
    </div>
  );
}
```

### 9.7.2 SSG + CSR

```typescript
// app/posts/[slug]/page.tsx (SSG)
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());
  return posts.map((post: any) => ({ slug: post.slug }));
}

export default async function PostPage({ params }: { params: { slug: string } }) {
  // 빌드 시 실행 (SSG)
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(res => res.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>

      {/* CSR로 실시간 댓글 */}
      <Comments postId={post.id} />
    </article>
  );
}
```

```typescript
// Comments.tsx (CSR)
'use client';

import { useEffect, useState } from 'react';

export default function Comments({ postId }: { postId: number }) {
  const [comments, setComments] = useState([]);

  useEffect(() => {
    fetch(`/api/posts/${postId}/comments`)
      .then(res => res.json())
      .then(setComments);
  }, [postId]);

  return (
    <div>
      <h2>댓글</h2>
      {comments.map((comment: any) => (
        <div key={comment.id}>{comment.text}</div>
      ))}
    </div>
  );
}
```

## 9.8 성능 비교

```
페이지 로드 시간 (상대적):

SSG (캐시):     ■ (가장 빠름)
ISR (캐시):     ■■
SSR:            ■■■■
CSR:            ■■■■■■ (가장 느림)

SEO:
SSG:  ⭐⭐⭐⭐⭐
ISR:  ⭐⭐⭐⭐⭐
SSR:  ⭐⭐⭐⭐⭐
CSR:  ⭐ (거의 불가능)

데이터 최신성:
SSG:  빌드 시점
ISR:  설정한 시간마다
SSR:  실시간
CSR:  실시간
```

## 핵심 요약

1. **SSG (Static Site Generation)**
   - 빌드 시 HTML 생성
   - 최고 성능, SEO 완벽
   - 정적 콘텐츠에 적합

2. **SSR (Server-Side Rendering)**
   - 매 요청마다 HTML 생성
   - 실시간 데이터
   - `cache: 'no-store'`

3. **ISR (Incremental Static Regeneration)**
   - SSG + 주기적 업데이트
   - `next: { revalidate: 60 }`
   - 최적의 균형

4. **CSR (Client-Side Rendering)**
   - 브라우저에서 렌더링
   - `'use client'`
   - 상호작용 많은 UI

5. **선택 기준**
   - SEO 필요 여부
   - 데이터 변경 빈도
   - 실시간성 요구사항

## 연습 문제

### 문제 1
다음 페이지에 적합한 렌더링 전략을 선택하세요:
1. 회사 소개 페이지 (거의 변경 안 됨)
2. 뉴스 목록 (5분마다 새 기사)
3. 사용자 대시보드 (실시간 통계)
4. 텍스트 에디터

<details>
<summary>정답</summary>

1. **SSG** - 정적, 거의 안 변함
2. **ISR** - `revalidate: 300` (5분)
3. **SSR** - `cache: 'no-store'`
4. **CSR** - `'use client'` + useState
</details>

### 문제 2
ISR을 사용하여 10분마다 재검증되는 블로그 목록을 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/blog/page.tsx
export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 600 }  // 10분 = 600초
  }).then(res => res.json());

  return (
    <div>
      <h1>블로그</h1>
      {posts.map((post: any) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
        </article>
      ))}
    </div>
  );
}
```
</details>

### 문제 3
Server Component (SSR)와 Client Component (CSR)를 조합한 상품 페이지를 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/products/[id]/page.tsx (Server - SSR)
import BuyButton from './BuyButton';

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    cache: 'no-store'
  }).then(res => res.json());

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.price}원</p>
      <BuyButton productId={product.id} />
    </div>
  );
}

// BuyButton.tsx (Client - CSR)
'use client';

import { useState } from 'react';

export default function BuyButton({ productId }: { productId: number }) {
  const [loading, setLoading] = useState(false);

  const handleBuy = async () => {
    setLoading(true);
    await fetch('/api/buy', {
      method: 'POST',
      body: JSON.stringify({ productId }),
    });
    setLoading(false);
  };

  return (
    <button onClick={handleBuy} disabled={loading}>
      {loading ? '구매 중...' : '구매하기'}
    </button>
  );
}
```
</details>

## 다음 단계

Part 03 완료! 다음 Part에서는:
- Server Components 심화
- Client Components 심화
- Server와 Client 조합 패턴

[Part 04: Server와 Client Components →](../part04-server-components/chapter10-server-components-deep.md)