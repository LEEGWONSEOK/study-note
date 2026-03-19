# Chapter 7. Server Components에서 데이터 가져오기

## 7.1 Server Components란?

Next.js 13+의 가장 큰 변화는 **Server Components**입니다. 기본적으로 모든 컴포넌트는 Server Component입니다.

### 7.1.1 Server Components의 특징

```typescript
// app/page.tsx - 기본적으로 Server Component
export default async function HomePage() {
  // 서버에서만 실행됨!
  const res = await fetch('https://api.example.com/posts');
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

**장점:**
- ✅ 서버에서 직접 데이터 페칭
- ✅ API 키 같은 민감정보를 안전하게 사용
- ✅ 데이터베이스 직접 접근 가능
- ✅ JavaScript 번들 크기 감소
- ✅ SEO 최적화

**제약사항:**
- ❌ `useState`, `useEffect` 사용 불가
- ❌ 이벤트 핸들러 (`onClick` 등) 사용 불가
- ❌ 브라우저 API 사용 불가

## 7.2 `async/await` 사용

Server Components는 **비동기 함수**로 만들 수 있습니다.

### 7.2.1 기본 데이터 페칭

```typescript
// app/posts/page.tsx
export default async function PostsPage() {
  // 서버에서 실행됨
  const res = await fetch('https://jsonplaceholder.typicode.com/posts');
  const posts = await res.json();

  return (
    <div>
      <h1>포스트 목록</h1>
      <ul>
        {posts.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 7.2.2 여러 데이터 동시에 가져오기

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  // 병렬로 실행
  const [users, posts, comments] = await Promise.all([
    fetch('https://api.example.com/users').then(res => res.json()),
    fetch('https://api.example.com/posts').then(res => res.json()),
    fetch('https://api.example.com/comments').then(res => res.json()),
  ]);

  return (
    <div>
      <h1>대시보드</h1>
      <p>사용자: {users.length}명</p>
      <p>포스트: {posts.length}개</p>
      <p>댓글: {comments.length}개</p>
    </div>
  );
}
```

### 7.2.3 순차적으로 가져오기

```typescript
// app/user/[id]/page.tsx
interface Props {
  params: { id: string };
}

export default async function UserPage({ params }: Props) {
  // 1. 사용자 정보 먼저
  const user = await fetch(`https://api.example.com/users/${params.id}`)
    .then(res => res.json());

  // 2. 그 사용자의 포스트
  const posts = await fetch(`https://api.example.com/users/${params.id}/posts`)
    .then(res => res.json());

  return (
    <div>
      <h1>{user.name}</h1>
      <h2>포스트 ({posts.length})</h2>
      <ul>
        {posts.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## 7.3 `fetch` API

Next.js는 기본 `fetch` API를 확장했습니다.

### 7.3.1 기본 사용법

```typescript
// 기본 fetch
const res = await fetch('https://api.example.com/data');
const data = await res.json();
```

### 7.3.2 에러 처리

```typescript
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts');

  if (!res.ok) {
    throw new Error('데이터를 불러오는데 실패했습니다');
  }

  const posts = await res.json();

  return (
    <div>
      {posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}
```

### 7.3.3 POST 요청

```typescript
export default async function CreatePostPage() {
  const createPost = async () => {
    'use server';  // Server Action

    const res = await fetch('https://api.example.com/posts', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        title: 'New Post',
        content: 'Content here',
      }),
    });

    const data = await res.json();
    return data;
  };

  return <div>...</div>;
}
```

## 7.4 캐싱 전략

Next.js는 `fetch`를 자동으로 캐싱합니다.

### 7.4.1 기본 캐싱 (Static - SSG)

```typescript
// 빌드 시점에 한 번만 실행, 이후 캐시 사용
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return <div>...</div>;
}
```

### 7.4.2 캐싱 비활성화 (Dynamic - SSR)

```typescript
// 매 요청마다 실행
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    cache: 'no-store'  // 캐시 사용 안 함
  });
  const posts = await res.json();

  return <div>...</div>;
}
```

### 7.4.3 시간 기반 재검증 (ISR)

```typescript
// 60초마다 재검증
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 }  // 60초
  });
  const posts = await res.json();

  return <div>...</div>;
}
```

**동작 방식:**
```
1. 첫 요청: 데이터 페칭 → 캐시 저장
2. 60초 이내 요청: 캐시된 데이터 반환
3. 60초 후 요청:
   - 캐시된 데이터 반환 (사용자는 빠르게 봄)
   - 백그라운드에서 새 데이터 페칭
   - 다음 요청부터 새 데이터 제공
```

### 7.4.4 캐싱 옵션 정리

```typescript
// 1. 캐싱 (SSG) - 기본값
fetch('https://api.example.com/data');
fetch('https://api.example.com/data', { cache: 'force-cache' });

// 2. 캐싱 안 함 (SSR)
fetch('https://api.example.com/data', { cache: 'no-store' });

// 3. 시간 기반 재검증 (ISR)
fetch('https://api.example.com/data', {
  next: { revalidate: 60 }  // 60초
});

// 4. 태그 기반 재검증
fetch('https://api.example.com/data', {
  next: { tags: ['posts'] }
});
```

## 7.5 Revalidation (재검증)

캐시를 수동으로 무효화할 수 있습니다.

### 7.5.1 시간 기반 재검증

```typescript
// app/posts/page.tsx
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }  // 1시간마다
  });
  const posts = await res.json();

  return <div>...</div>;
}
```

### 7.5.2 온디맨드 재검증 (태그 기반)

```typescript
// app/posts/page.tsx
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] }  // 'posts' 태그
  });
  const posts = await res.json();

  return <div>...</div>;
}
```

```typescript
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const tag = request.nextUrl.searchParams.get('tag');

  if (tag) {
    revalidateTag(tag);  // 'posts' 태그 캐시 무효화
    return Response.json({ revalidated: true });
  }

  return Response.json({ revalidated: false });
}

// 사용: POST /api/revalidate?tag=posts
```

### 7.5.3 경로 기반 재검증

```typescript
// app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache';

export async function POST() {
  revalidatePath('/posts');  // /posts 페이지 캐시 무효화
  return Response.json({ revalidated: true });
}
```

## 7.6 데이터 페칭 패턴

### 7.6.1 병렬 페칭

```typescript
// ✅ 좋은 패턴: 병렬 실행
export default async function Page() {
  const [users, posts] = await Promise.all([
    fetch('https://api.example.com/users').then(res => res.json()),
    fetch('https://api.example.com/posts').then(res => res.json()),
  ]);

  return <div>...</div>;
}
```

### 7.6.2 순차 페칭 (의존성 있을 때)

```typescript
// ✅ 순차 실행 (두 번째가 첫 번째에 의존)
export default async function Page() {
  const user = await fetch('https://api.example.com/user/1')
    .then(res => res.json());

  const posts = await fetch(`https://api.example.com/users/${user.id}/posts`)
    .then(res => res.json());

  return <div>...</div>;
}
```

### 7.6.3 컴포넌트 분리

```typescript
// app/page.tsx
export default function HomePage() {
  return (
    <div>
      <UserInfo />
      <PostsList />
    </div>
  );
}

// 각 컴포넌트가 독립적으로 데이터 페칭
async function UserInfo() {
  const user = await fetch('https://api.example.com/user')
    .then(res => res.json());

  return <div>{user.name}</div>;
}

async function PostsList() {
  const posts = await fetch('https://api.example.com/posts')
    .then(res => res.json());

  return <ul>{posts.map(...)}</ul>;
}
```

### 7.6.4 Streaming (점진적 렌더링)

```typescript
// app/page.tsx
import { Suspense } from 'react';

export default function HomePage() {
  return (
    <div>
      <h1>대시보드</h1>

      {/* 빠른 데이터 먼저 표시 */}
      <QuickData />

      {/* 느린 데이터는 Suspense로 감싸기 */}
      <Suspense fallback={<div>로딩 중...</div>}>
        <SlowData />
      </Suspense>
    </div>
  );
}

async function QuickData() {
  const data = await fetch('https://api.example.com/quick')
    .then(res => res.json());
  return <div>{data.title}</div>;
}

async function SlowData() {
  // 느린 API
  const data = await fetch('https://api.example.com/slow')
    .then(res => res.json());
  return <div>{data.content}</div>;
}
```

## 7.7 실전 예제: 블로그 목록

```typescript
// app/blog/page.tsx
import Link from 'next/link';
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: '블로그',
  description: '블로그 포스트 목록',
};

interface Post {
  id: number;
  title: string;
  excerpt: string;
  date: string;
}

export default async function BlogPage() {
  // 데이터 페칭
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }  // 1시간마다 재검증
  });

  if (!res.ok) {
    throw new Error('포스트를 불러올 수 없습니다');
  }

  const posts: Post[] = await res.json();

  return (
    <div className="max-w-4xl mx-auto">
      <h1 className="text-4xl font-bold mb-8">블로그</h1>

      <div className="grid gap-6">
        {posts.map((post) => (
          <article
            key={post.id}
            className="border rounded-lg p-6 hover:shadow-lg transition"
          >
            <Link href={`/blog/${post.id}`}>
              <h2 className="text-2xl font-bold mb-2">{post.title}</h2>
              <p className="text-gray-600 mb-2">{post.excerpt}</p>
              <time className="text-sm text-gray-400">{post.date}</time>
            </Link>
          </article>
        ))}
      </div>
    </div>
  );
}
```

## 7.8 실전 예제: 상품 상세 페이지

```typescript
// app/products/[id]/page.tsx
import { Metadata } from 'next';
import { notFound } from 'next/navigation';

interface Props {
  params: { id: string };
}

interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  imageUrl: string;
}

// 동적 메타데이터
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return {
    title: product.name,
    description: product.description,
  };
}

export default async function ProductPage({ params }: Props) {
  const res = await fetch(`https://api.example.com/products/${params.id}`, {
    next: { revalidate: 60 }  // 1분마다
  });

  if (!res.ok) {
    notFound();
  }

  const product: Product = await res.json();

  return (
    <div className="max-w-4xl mx-auto">
      <div className="grid md:grid-cols-2 gap-8">
        <img
          src={product.imageUrl}
          alt={product.name}
          className="rounded-lg"
        />
        <div>
          <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
          <p className="text-gray-600 mb-4">{product.description}</p>
          <p className="text-2xl font-bold text-blue-500">
            {product.price.toLocaleString()}원
          </p>
          <button className="mt-4 px-6 py-2 bg-blue-500 text-white rounded">
            구매하기
          </button>
        </div>
      </div>
    </div>
  );
}
```

## 핵심 요약

1. **Server Components**
   - 기본적으로 모든 컴포넌트는 서버 컴포넌트
   - `async/await` 사용 가능
   - 서버에서만 실행

2. **`fetch` API**
   - Next.js가 확장한 fetch
   - 자동 캐싱
   - 재검증 지원

3. **캐싱 전략**
   - `cache: 'force-cache'`: SSG (기본값)
   - `cache: 'no-store'`: SSR
   - `next: { revalidate: 60 }`: ISR

4. **데이터 페칭 패턴**
   - 병렬: `Promise.all`
   - 순차: `await` 연속
   - 스트리밍: `Suspense`

5. **재검증**
   - 시간 기반: `revalidate`
   - 태그 기반: `revalidateTag`
   - 경로 기반: `revalidatePath`

## 연습 문제

### 문제 1
API에서 사용자 목록을 가져와 표시하는 페이지를 만드세요.
- API: `https://jsonplaceholder.typicode.com/users`
- 1시간마다 재검증

<details>
<summary>정답</summary>

```typescript
// app/users/page.tsx
export default async function UsersPage() {
  const res = await fetch('https://jsonplaceholder.typicode.com/users', {
    next: { revalidate: 3600 }  // 1시간
  });

  const users = await res.json();

  return (
    <div>
      <h1>사용자 목록</h1>
      <ul>
        {users.map((user: any) => (
          <li key={user.id}>
            {user.name} ({user.email})
          </li>
        ))}
      </ul>
    </div>
  );
}
```
</details>

### 문제 2
포스트 목록과 댓글 수를 병렬로 가져와 표시하세요.

<details>
<summary>정답</summary>

```typescript
// app/posts/page.tsx
export default async function PostsPage() {
  const [posts, comments] = await Promise.all([
    fetch('https://jsonplaceholder.typicode.com/posts').then(res => res.json()),
    fetch('https://jsonplaceholder.typicode.com/comments').then(res => res.json()),
  ]);

  return (
    <div>
      <h1>통계</h1>
      <p>포스트: {posts.length}개</p>
      <p>댓글: {comments.length}개</p>
    </div>
  );
}
```
</details>

### 문제 3
에러 처리가 포함된 데이터 페칭을 구현하세요.

<details>
<summary>정답</summary>

```typescript
// app/products/page.tsx
export default async function ProductsPage() {
  const res = await fetch('https://api.example.com/products');

  if (!res.ok) {
    throw new Error('상품을 불러올 수 없습니다');
  }

  const products = await res.json();

  return (
    <div>
      <h1>상품 목록</h1>
      {products.map((product: any) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}

// app/products/error.tsx
'use client';

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>오류 발생!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>다시 시도</button>
    </div>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- Client Components에서 데이터 가져오기
- `useEffect` + `fetch`
- SWR, React Query
- 로딩 상태 관리

[Chapter 8. Client Components에서 데이터 가져오기 →](chapter08-client-data-fetching.md)