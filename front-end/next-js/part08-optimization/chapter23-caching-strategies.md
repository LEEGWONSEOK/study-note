# Chapter 23: 캐싱 전략

## 1. Next.js 캐싱 개요

### 1.1 캐싱 레이어

Next.js는 4가지 레벨의 캐싱을 제공합니다:

```
1. Request Memoization   → 동일 요청 재사용 (렌더링 중)
2. Data Cache            → fetch 결과 저장 (서버)
3. Full Route Cache      → 전체 페이지 캐싱 (빌드/런타임)
4. Router Cache          → 클라이언트 캐싱 (브라우저)
```

### 1.2 캐싱 위치

```
┌─────────────┐
│   Browser   │ ← Router Cache (클라이언트)
└─────────────┘
       ↕
┌─────────────┐
│   Server    │ ← Data Cache, Full Route Cache (서버)
└─────────────┘
```

## 2. Data Cache (fetch)

### 2.1 기본 캐싱

```tsx
// 기본적으로 fetch는 무기한 캐싱
export default async function Page() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return <div>{/* ... */}</div>;
}
```

### 2.2 캐싱 비활성화

```tsx
// 매 요청마다 새로 가져오기
const res = await fetch('https://api.example.com/posts', {
  cache: 'no-store'
});
```

### 2.3 시간 기반 재검증 (ISR)

```tsx
// 10초마다 재검증
const res = await fetch('https://api.example.com/posts', {
  next: { revalidate: 10 }
});
```

**동작 방식:**
```
요청 1 (0초):   캐시 생성 → 응답
요청 2 (5초):   캐시 사용 → 응답
요청 3 (11초):  캐시 사용 → 응답 (백그라운드 재검증 시작)
요청 4 (12초):  새 캐시 사용 → 응답
```

### 2.4 태그 기반 재검증

```tsx
// 태그 추가
const res = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] }
});
```

```typescript
// 수동 재검증
import { revalidateTag } from 'next/cache';

revalidateTag('posts'); // 'posts' 태그를 가진 모든 캐시 무효화
```

## 3. Full Route Cache

### 3.1 정적 렌더링 (캐싱됨)

```tsx
// 빌드 시 렌더링 → 캐싱
export default async function Page() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return <div>{/* ... */}</div>;
}
```

### 3.2 동적 렌더링 (캐싱 안 됨)

```tsx
// 동적 함수 사용 → 매 요청마다 렌더링
import { cookies } from 'next/headers';

export default async function Page() {
  const cookieStore = cookies(); // 동적!

  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return <div>{/* ... */}</div>;
}
```

**동적 함수:**
```typescript
- cookies()
- headers()
- searchParams (페이지 props)
- fetch(..., { cache: 'no-store' })
```

### 3.3 라우트 세그먼트 설정

```tsx
// app/posts/page.tsx
export const dynamic = 'force-dynamic'; // 항상 동적
// 또는
export const revalidate = 60; // 60초마다 재검증
```

**옵션:**
```typescript
export const dynamic = 'auto' | 'force-dynamic' | 'error' | 'force-static';
export const revalidate = false | 0 | number;
```

## 4. Router Cache (클라이언트)

### 4.1 자동 캐싱

```tsx
'use client';

import Link from 'next/link';

export default function Navigation() {
  return (
    <nav>
      <Link href="/posts">Posts</Link>    {/* 프리페치 + 캐시 */}
      <Link href="/about">About</Link>    {/* 프리페치 + 캐시 */}
    </nav>
  );
}
```

**동작:**
- `<Link>` 컴포넌트가 뷰포트에 들어오면 자동 프리페치
- 결과를 Router Cache에 저장
- 클릭 시 즉시 표시

### 4.2 프리페치 제어

```tsx
// 프리페치 비활성화
<Link href="/posts" prefetch={false}>
  Posts
</Link>

// 명시적 프리페치
<Link href="/posts" prefetch={true}>
  Posts
</Link>
```

### 4.3 캐시 무효화

```tsx
'use client';

import { useRouter } from 'next/navigation';

export default function Page() {
  const router = useRouter();

  function handleRefresh() {
    router.refresh(); // 현재 페이지 재검증
  }

  return (
    <button onClick={handleRefresh}>
      새로고침
    </button>
  );
}
```

## 5. Request Memoization

### 5.1 자동 중복 제거

```tsx
// 두 번 호출해도 실제 요청은 1번만!
async function getUser(id: string) {
  const res = await fetch(`https://api.example.com/users/${id}`);
  return res.json();
}

export default async function Page() {
  const user1 = await getUser('1'); // 요청 발생
  const user2 = await getUser('1'); // 캐시 사용 (요청 안 함)

  return <div>{user1.name}</div>;
}
```

**범위:** 단일 렌더링 패스 (서버 컴포넌트 트리)

### 5.2 POST 요청은 제외

```tsx
// POST는 메모이제이션 안 됨
await fetch('https://api.example.com/posts', {
  method: 'POST',
  body: JSON.stringify({ title: 'New Post' })
});
```

## 6. 재검증 전략

### 6.1 시간 기반 (ISR)

```tsx
// 페이지 레벨
export const revalidate = 3600; // 1시간

export default async function Page() {
  return <div>This page revalidates every hour</div>;
}
```

```tsx
// fetch 레벨
const res = await fetch('https://api.example.com/posts', {
  next: { revalidate: 60 } // 1분
});
```

### 6.2 온디맨드 (수동)

**경로 기반:**
```typescript
// app/actions/posts.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  // 게시글 생성...

  revalidatePath('/posts');           // /posts 재검증
  revalidatePath('/posts/[id]');      // 동적 경로 재검증
  revalidatePath('/posts', 'layout'); // 레이아웃 포함
}
```

**태그 기반:**
```typescript
// 데이터 가져올 때 태그 추가
const res = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts', 'blog'] }
});

// 재검증
import { revalidateTag } from 'next/cache';

revalidateTag('posts'); // 'posts' 태그 전체 재검증
```

### 6.3 혼합 전략

```tsx
// app/posts/page.tsx
export const revalidate = 3600; // 기본 1시간

export default async function PostsPage() {
  // 일반 게시글: 1시간 캐싱
  const posts = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] }
  }).then(res => res.json());

  // 인기 게시글: 5분 캐싱
  const trending = await fetch('https://api.example.com/posts/trending', {
    next: { revalidate: 300 }
  }).then(res => res.json());

  return (
    <div>
      <TrendingPosts posts={trending} />
      <AllPosts posts={posts} />
    </div>
  );
}
```

## 7. 캐싱 없는 데이터 가져오기

### 7.1 fetch 옵션

```tsx
// 옵션 1: cache: 'no-store'
const res = await fetch('https://api.example.com/user', {
  cache: 'no-store'
});

// 옵션 2: revalidate: 0
const res = await fetch('https://api.example.com/user', {
  next: { revalidate: 0 }
});
```

### 7.2 동적 함수 사용

```tsx
import { cookies } from 'next/headers';

export default async function Page() {
  // cookies() 사용으로 인해 전체 페이지가 동적 렌더링
  const cookieStore = cookies();

  const res = await fetch('https://api.example.com/posts');
  // 이 fetch도 캐싱 안 됨

  return <div>{/* ... */}</div>;
}
```

### 7.3 페이지 레벨 설정

```tsx
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic';
export const revalidate = 0;

export default async function DashboardPage() {
  // 모든 데이터가 캐싱 안 됨
  const user = await fetch('https://api.example.com/user');
  const stats = await fetch('https://api.example.com/stats');

  return <div>{/* ... */}</div>;
}
```

## 8. 실전 예시

### 8.1 블로그 포스트 (ISR)

```tsx
// app/posts/[id]/page.tsx
export const revalidate = 3600; // 1시간

export default async function PostPage({
  params
}: {
  params: { id: string }
}) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`, {
    next: { tags: [`post-${params.id}`] }
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
// app/actions/posts.ts
'use server';

import { revalidateTag } from 'next/cache';

export async function updatePost(id: string, data: any) {
  // 업데이트...

  revalidateTag(`post-${id}`); // 특정 포스트만 재검증
  revalidateTag('posts');      // 목록도 재검증
}
```

### 8.2 실시간 대시보드

```tsx
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic';
export const revalidate = 0;

export default async function DashboardPage() {
  // 항상 최신 데이터
  const stats = await fetch('https://api.example.com/stats', {
    cache: 'no-store'
  }).then(res => res.json());

  return (
    <div>
      <h1>실시간 통계</h1>
      <p>방문자: {stats.visitors}</p>
      <p>매출: {stats.revenue}</p>
    </div>
  );
}
```

### 8.3 하이브리드 캐싱

```tsx
// app/page.tsx
export default async function HomePage() {
  // 정적 콘텐츠 (빌드 시 캐싱)
  const staticContent = await fetch('https://api.example.com/content', {
    next: { revalidate: 86400 } // 24시간
  }).then(res => res.json());

  // 동적 콘텐츠 (매 요청마다)
  const dynamicContent = await fetch('https://api.example.com/trending', {
    cache: 'no-store'
  }).then(res => res.json());

  return (
    <div>
      <StaticSection content={staticContent} />
      <DynamicSection content={dynamicContent} />
    </div>
  );
}
```

## 9. 캐싱 디버깅

### 9.1 빌드 출력

```bash
npm run build
```

```
Route (app)                              Size     First Load JS
┌ ○ /                                    5 kB       85 kB
├ ○ /about                               3 kB       83 kB
├ ● /posts                               2 kB       82 kB
└ ƒ /dashboard                           4 kB       84 kB

○ (Static)   정적 렌더링
● (SSG)      Static Site Generation
ƒ (Dynamic)  동적 렌더링
```

### 9.2 캐시 헤더 확인

```bash
curl -I https://yourdomain.com/posts
```

```
HTTP/2 200
cache-control: s-maxage=3600, stale-while-revalidate
x-nextjs-cache: HIT
```

**캐시 상태:**
```
HIT:    캐시에서 제공
MISS:   캐시 미스 (새로 생성)
STALE:  오래된 캐시 (재검증 필요)
```

## 10. 실습 문제

### 문제 1: 뉴스 피드
다음 요구사항을 만족하는 뉴스 피드를 구현하세요:
- 메인 기사: 5분마다 업데이트
- 사이드 기사: 1시간마다 업데이트
- 댓글: 캐싱 안 함

### 문제 2: E-Commerce 상품 페이지
- 상품 정보: 1시간 캐싱
- 재고: 캐싱 안 함
- 리뷰: 10분 캐싱
- 관리자가 상품 수정 시 즉시 반영

### 문제 3: 캐시 워밍
빌드 시 자주 방문하는 100개 페이지를 미리 생성하세요.

## 11. 실습 해답

### 문제 1 해답

```tsx
// app/news/page.tsx
export default async function NewsPage() {
  // 메인 기사: 5분 캐싱
  const mainArticles = await fetch('https://api.example.com/news/main', {
    next: {
      revalidate: 300, // 5분
      tags: ['main-news']
    }
  }).then(res => res.json());

  // 사이드 기사: 1시간 캐싱
  const sideArticles = await fetch('https://api.example.com/news/side', {
    next: {
      revalidate: 3600, // 1시간
      tags: ['side-news']
    }
  }).then(res => res.json());

  // 댓글: 캐싱 안 함
  const comments = await fetch('https://api.example.com/comments', {
    cache: 'no-store'
  }).then(res => res.json());

  return (
    <div className="grid grid-cols-3 gap-4">
      {/* 메인 콘텐츠 */}
      <div className="col-span-2">
        <h2 className="text-2xl font-bold mb-4">주요 뉴스</h2>
        {mainArticles.map((article: any) => (
          <article key={article.id} className="mb-4">
            <h3>{article.title}</h3>
            <p>{article.summary}</p>
          </article>
        ))}
      </div>

      {/* 사이드바 */}
      <div>
        <h2 className="text-xl font-bold mb-4">더 많은 뉴스</h2>
        {sideArticles.map((article: any) => (
          <article key={article.id} className="mb-2">
            <h4 className="text-sm">{article.title}</h4>
          </article>
        ))}

        <h2 className="text-xl font-bold mb-4 mt-8">최근 댓글</h2>
        {comments.map((comment: any) => (
          <div key={comment.id} className="mb-2 text-sm">
            <strong>{comment.author}:</strong> {comment.text}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 문제 2 해답

```tsx
// app/products/[id]/page.tsx
export default async function ProductPage({
  params
}: {
  params: { id: string }
}) {
  // 상품 정보: 1시간 캐싱
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    next: {
      revalidate: 3600,
      tags: [`product-${params.id}`]
    }
  }).then(res => res.json());

  // 재고: 캐싱 안 함
  const stock = await fetch(`https://api.example.com/products/${params.id}/stock`, {
    cache: 'no-store'
  }).then(res => res.json());

  // 리뷰: 10분 캐싱
  const reviews = await fetch(`https://api.example.com/products/${params.id}/reviews`, {
    next: {
      revalidate: 600,
      tags: [`reviews-${params.id}`]
    }
  }).then(res => res.json());

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="text-2xl font-bold">${product.price}</p>

      <div className="mt-4">
        <p className={stock.available ? 'text-green-600' : 'text-red-600'}>
          {stock.available ? `재고 있음 (${stock.quantity}개)` : '품절'}
        </p>
      </div>

      <div className="mt-8">
        <h2 className="text-xl font-bold mb-4">리뷰</h2>
        {reviews.map((review: any) => (
          <div key={review.id} className="mb-4">
            <strong>{review.author}</strong>
            <p>{review.comment}</p>
            <p>⭐ {review.rating}/5</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

```typescript
// app/actions/products.ts
'use server';

import { revalidateTag } from 'next/cache';

export async function updateProduct(id: string, data: any) {
  // 상품 업데이트 로직...
  await fetch(`https://api.example.com/products/${id}`, {
    method: 'PUT',
    body: JSON.stringify(data)
  });

  // 즉시 반영
  revalidateTag(`product-${id}`);
}

export async function addReview(productId: string, review: any) {
  // 리뷰 추가 로직...
  await fetch(`https://api.example.com/products/${productId}/reviews`, {
    method: 'POST',
    body: JSON.stringify(review)
  });

  // 리뷰 캐시 갱신
  revalidateTag(`reviews-${productId}`);
}
```

### 문제 3 해답

```typescript
// scripts/cache-warming.ts
async function warmCache() {
  // 인기 상품 100개 ID 가져오기
  const response = await fetch('https://api.example.com/products/popular?limit=100');
  const products = await response.json();

  console.log(`Warming cache for ${products.length} products...`);

  // 동시 요청 제한 (5개씩)
  const batchSize = 5;

  for (let i = 0; i < products.length; i += batchSize) {
    const batch = products.slice(i, i + batchSize);

    await Promise.all(
      batch.map(async (product: any) => {
        try {
          const res = await fetch(
            `https://yourdomain.com/products/${product.id}`,
            { cache: 'force-cache' }
          );

          console.log(`✓ Cached: ${product.id} (${res.status})`);
        } catch (error) {
          console.error(`✗ Failed: ${product.id}`, error);
        }
      })
    );

    // 서버 부하 방지
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  console.log('Cache warming completed!');
}

warmCache();
```

**빌드 후 실행:**
```bash
npm run build
node scripts/cache-warming.ts
```

**또는 generateStaticParams 사용:**
```tsx
// app/products/[id]/page.tsx
export async function generateStaticParams() {
  const products = await fetch('https://api.example.com/products/popular?limit=100')
    .then(res => res.json());

  return products.map((product: any) => ({
    id: product.id.toString()
  }));
}

export default async function ProductPage({
  params
}: {
  params: { id: string }
}) {
  // 100개 페이지가 빌드 시 미리 생성됨
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return <div>{product.name}</div>;
}
```

## 핵심 요약

1. **4가지 캐싱 레이어**: Request Memoization, Data Cache, Full Route Cache, Router Cache
2. **시간 기반 재검증**: `revalidate: 60` (ISR)
3. **온디맨드 재검증**: `revalidatePath()`, `revalidateTag()`
4. **캐싱 비활성화**: `cache: 'no-store'`, 동적 함수 사용
5. **하이브리드**: 페이지 내에서 서로 다른 캐싱 전략 혼합 가능

[← Chapter 22: 번들 최적화](./chapter22-bundle-optimization.md) | [Part 09: 배포 →](../part09-deployment/README.md)