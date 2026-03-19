# Chapter 10. Server Components 심화

## 10.1 Server Components의 핵심 개념

Server Components는 Next.js 13+의 가장 혁신적인 기능입니다. **서버에서만 실행되는 React 컴포넌트**입니다.

### 10.1.1 Server Components의 특징

```typescript
// app/page.tsx - 기본적으로 Server Component
export default async function HomePage() {
  // ✅ 서버에서만 실행됨
  // ✅ async/await 사용 가능
  // ✅ 데이터베이스 직접 접근 가능
  // ✅ API 키 같은 민감정보 안전하게 사용

  const posts = await db.post.findMany();

  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
        </article>
      ))}
    </div>
  );
}
```

**장점:**
- ✅ **Zero JavaScript**: 클라이언트에 JavaScript 전송 안 됨
- ✅ **직접 데이터 액세스**: DB, 파일 시스템 직접 접근
- ✅ **보안**: 민감정보가 클라이언트에 노출 안 됨
- ✅ **성능**: 번들 크기 감소
- ✅ **SEO**: 완벽한 검색 엔진 최적화

**제약사항:**
- ❌ `useState`, `useEffect` 사용 불가
- ❌ 이벤트 핸들러 (`onClick` 등) 사용 불가
- ❌ 브라우저 API 사용 불가
- ❌ React Context 사용 불가

## 10.2 Server Components의 장점

### 10.2.1 번들 크기 감소

```typescript
// ❌ Client Component - 클라이언트에 라이브러리 전송
'use client';
import { format } from 'date-fns';  // 라이브러리가 번들에 포함됨

export default function DateDisplay({ date }: { date: Date }) {
  return <div>{format(date, 'yyyy-MM-dd')}</div>;
}
```

```typescript
// ✅ Server Component - 라이브러리가 번들에 포함 안 됨
import { format } from 'date-fns';  // 서버에서만 실행

export default function DateDisplay({ date }: { date: Date }) {
  return <div>{format(date, 'yyyy-MM-dd')}</div>;
}
```

**결과:**
```
Client Component: 번들 크기 +200KB
Server Component: 번들 크기 +0KB
```

### 10.2.2 백엔드 리소스 직접 접근

```typescript
// app/posts/page.tsx
import { db } from '@/lib/db';  // Prisma, Drizzle 등

export default async function PostsPage() {
  // 데이터베이스 직접 쿼리 (서버에서만 실행)
  const posts = await db.post.findMany({
    include: {
      author: true,
      comments: true,
    },
    orderBy: {
      createdAt: 'desc',
    },
  });

  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>작성자: {post.author.name}</p>
          <p>댓글: {post.comments.length}개</p>
        </article>
      ))}
    </div>
  );
}
```

### 10.2.3 민감정보 보호

```typescript
// app/admin/page.tsx
export default async function AdminPage() {
  // API 키가 클라이언트에 절대 노출되지 않음!
  const apiKey = process.env.ADMIN_API_KEY;
  const secretToken = process.env.SECRET_TOKEN;

  const data = await fetch('https://api.example.com/admin', {
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'X-Secret-Token': secretToken,
    },
  }).then(res => res.json());

  return (
    <div>
      <h1>관리자 대시보드</h1>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  );
}
```

**Client Component였다면:**
```typescript
// ❌ 위험! API 키가 클라이언트 코드에 포함됨
'use client';
const apiKey = process.env.ADMIN_API_KEY;  // undefined!
// 클라이언트에서는 NEXT_PUBLIC_ 접두사가 있어야만 접근 가능
```

## 10.3 데이터베이스 직접 접근

### 10.3.1 Prisma 사용 예제

```typescript
// lib/db.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const db = globalForPrisma.prisma || new PrismaClient();

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db;
```

```typescript
// app/users/page.tsx
import { db } from '@/lib/db';

export default async function UsersPage() {
  // 서버에서 직접 DB 쿼리
  const users = await db.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
      _count: {
        select: { posts: true },
      },
    },
  });

  return (
    <div>
      <h1>사용자 목록</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>
            {user.name} ({user.email}) - 포스트 {user._count.posts}개
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 10.3.2 복잡한 쿼리

```typescript
// app/dashboard/page.tsx
import { db } from '@/lib/db';

export default async function DashboardPage() {
  // 병렬로 여러 쿼리 실행
  const [
    totalUsers,
    totalPosts,
    recentPosts,
    topAuthors,
  ] = await Promise.all([
    db.user.count(),
    db.post.count(),
    db.post.findMany({
      take: 5,
      orderBy: { createdAt: 'desc' },
      include: { author: true },
    }),
    db.user.findMany({
      take: 5,
      orderBy: {
        posts: { _count: 'desc' },
      },
      include: {
        _count: { select: { posts: true } },
      },
    }),
  ]);

  return (
    <div>
      <h1>대시보드</h1>
      <div className="stats">
        <div>전체 사용자: {totalUsers}</div>
        <div>전체 포스트: {totalPosts}</div>
      </div>

      <h2>최근 포스트</h2>
      {recentPosts.map(post => (
        <div key={post.id}>
          {post.title} by {post.author.name}
        </div>
      ))}

      <h2>인기 작성자</h2>
      {topAuthors.map(author => (
        <div key={author.id}>
          {author.name} - {author._count.posts}개
        </div>
      ))}
    </div>
  );
}
```

### 10.3.3 트랜잭션

```typescript
// app/admin/actions.ts
'use server';

import { db } from '@/lib/db';

export async function transferOwnership(postId: number, newOwnerId: number) {
  // 트랜잭션으로 안전하게 처리
  await db.$transaction(async (tx) => {
    // 1. 포스트 소유자 변경
    await tx.post.update({
      where: { id: postId },
      data: { authorId: newOwnerId },
    });

    // 2. 이전 소유자 통계 업데이트
    await tx.user.update({
      where: { id: oldOwnerId },
      data: {
        postCount: { decrement: 1 },
      },
    });

    // 3. 새 소유자 통계 업데이트
    await tx.user.update({
      where: { id: newOwnerId },
      data: {
        postCount: { increment: 1 },
      },
    });
  });
}
```

## 10.4 파일 시스템 접근

### 10.4.1 파일 읽기

```typescript
// app/docs/[slug]/page.tsx
import { readFile } from 'fs/promises';
import path from 'path';
import { marked } from 'marked';

interface Props {
  params: { slug: string };
}

export default async function DocPage({ params }: Props) {
  // 파일 시스템에서 마크다운 파일 읽기
  const filePath = path.join(process.cwd(), 'docs', `${params.slug}.md`);
  const markdown = await readFile(filePath, 'utf-8');

  // 마크다운을 HTML로 변환
  const html = marked(markdown);

  return (
    <article
      dangerouslySetInnerHTML={{ __html: html }}
      className="prose"
    />
  );
}

// 빌드 시 모든 문서 페이지 생성
export async function generateStaticParams() {
  const { readdir } = await import('fs/promises');
  const docsDir = path.join(process.cwd(), 'docs');
  const files = await readdir(docsDir);

  return files
    .filter(file => file.endsWith('.md'))
    .map(file => ({
      slug: file.replace('.md', ''),
    }));
}
```

### 10.4.2 JSON 파일 읽기

```typescript
// app/config/page.tsx
import { readFile } from 'fs/promises';
import path from 'path';

export default async function ConfigPage() {
  // 설정 파일 읽기
  const configPath = path.join(process.cwd(), 'config.json');
  const configData = await readFile(configPath, 'utf-8');
  const config = JSON.parse(configData);

  return (
    <div>
      <h1>애플리케이션 설정</h1>
      <pre>{JSON.stringify(config, null, 2)}</pre>
    </div>
  );
}
```

## 10.5 외부 API 호출

### 10.5.1 기본 API 호출

```typescript
// app/weather/page.tsx
interface WeatherData {
  temperature: number;
  description: string;
  humidity: number;
}

export default async function WeatherPage() {
  // API 키를 안전하게 사용
  const apiKey = process.env.WEATHER_API_KEY;

  const res = await fetch(
    `https://api.weather.com/data?key=${apiKey}&city=Seoul`,
    { next: { revalidate: 600 } }  // 10분마다 갱신
  );

  const weather: WeatherData = await res.json();

  return (
    <div>
      <h1>서울 날씨</h1>
      <p>온도: {weather.temperature}°C</p>
      <p>날씨: {weather.description}</p>
      <p>습도: {weather.humidity}%</p>
    </div>
  );
}
```

### 10.5.2 여러 API 병렬 호출

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  // 여러 API를 병렬로 호출
  const [userData, postsData, commentsData] = await Promise.all([
    fetch('https://api.example.com/users').then(res => res.json()),
    fetch('https://api.example.com/posts').then(res => res.json()),
    fetch('https://api.example.com/comments').then(res => res.json()),
  ]);

  return (
    <div>
      <h1>통계</h1>
      <p>사용자: {userData.length}</p>
      <p>포스트: {postsData.length}</p>
      <p>댓글: {commentsData.length}</p>
    </div>
  );
}
```

### 10.5.3 에러 처리

```typescript
// app/products/page.tsx
import { notFound } from 'next/navigation';

export default async function ProductsPage() {
  const res = await fetch('https://api.example.com/products');

  if (!res.ok) {
    // 404면 not-found 페이지로
    if (res.status === 404) {
      notFound();
    }

    // 그 외 에러는 error.tsx로
    throw new Error(`Failed to fetch: ${res.status}`);
  }

  const products = await res.json();

  return (
    <div>
      {products.map((product: any) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

## 10.6 Server Components의 제약사항

### 10.6.1 사용 불가능한 것들

```typescript
// ❌ 이런 것들은 Server Component에서 불가능
export default async function ServerPage() {
  // ❌ React Hooks
  const [state, setState] = useState(0);
  useEffect(() => {}, []);

  // ❌ 이벤트 핸들러
  const handleClick = () => {};
  <button onClick={handleClick}>클릭</button>

  // ❌ 브라우저 API
  window.localStorage.getItem('key');
  document.getElementById('id');

  // ❌ React Context
  const value = useContext(MyContext);

  return <div>...</div>;
}
```

### 10.6.2 해결 방법: Client Component 사용

```typescript
// app/page.tsx (Server Component)
import ClientButton from './ClientButton';

export default async function HomePage() {
  // 서버에서 데이터 페칭
  const data = await fetch('https://api.example.com/data')
    .then(res => res.json());

  return (
    <div>
      <h1>서버 컴포넌트</h1>
      <p>데이터: {data.title}</p>

      {/* 상호작용은 Client Component에게 */}
      <ClientButton />
    </div>
  );
}
```

```typescript
// ClientButton.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function ClientButton() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      클릭 수: {count}
    </button>
  );
}
```

## 10.7 실전 예제: 블로그 시스템

### 10.7.1 블로그 목록 (Server Component)

```typescript
// app/blog/page.tsx
import { db } from '@/lib/db';
import Link from 'next/link';

export default async function BlogPage() {
  // 데이터베이스에서 직접 가져오기
  const posts = await db.post.findMany({
    include: {
      author: true,
      _count: {
        select: { comments: true },
      },
    },
    orderBy: {
      createdAt: 'desc',
    },
  });

  return (
    <div className="max-w-4xl mx-auto">
      <h1 className="text-4xl font-bold mb-8">블로그</h1>

      <div className="space-y-6">
        {posts.map(post => (
          <article key={post.id} className="border rounded-lg p-6">
            <Link href={`/blog/${post.slug}`}>
              <h2 className="text-2xl font-bold mb-2">{post.title}</h2>
            </Link>
            <p className="text-gray-600 mb-2">{post.excerpt}</p>
            <div className="flex justify-between text-sm text-gray-500">
              <span>작성자: {post.author.name}</span>
              <span>댓글 {post._count.comments}개</span>
              <time>{new Date(post.createdAt).toLocaleDateString()}</time>
            </div>
          </article>
        ))}
      </div>
    </div>
  );
}
```

### 10.7.2 블로그 상세 (Server Component + Client Component)

```typescript
// app/blog/[slug]/page.tsx (Server Component)
import { db } from '@/lib/db';
import { notFound } from 'next/navigation';
import CommentSection from './CommentSection';

interface Props {
  params: { slug: string };
}

export async function generateMetadata({ params }: Props) {
  const post = await db.post.findUnique({
    where: { slug: params.slug },
  });

  if (!post) return {};

  return {
    title: post.title,
    description: post.excerpt,
  };
}

export default async function BlogPostPage({ params }: Props) {
  // 데이터베이스에서 포스트 가져오기
  const post = await db.post.findUnique({
    where: { slug: params.slug },
    include: {
      author: true,
    },
  });

  if (!post) {
    notFound();
  }

  return (
    <article className="max-w-4xl mx-auto">
      <h1 className="text-4xl font-bold mb-4">{post.title}</h1>

      <div className="text-gray-600 mb-8">
        <span>작성자: {post.author.name}</span>
        <span className="mx-2">•</span>
        <time>{new Date(post.createdAt).toLocaleDateString()}</time>
      </div>

      <div
        className="prose prose-lg"
        dangerouslySetInnerHTML={{ __html: post.content }}
      />

      {/* 댓글 섹션 (Client Component) */}
      <CommentSection postId={post.id} />
    </article>
  );
}
```

```typescript
// app/blog/[slug]/CommentSection.tsx (Client Component)
'use client';

import { useState, useEffect } from 'react';

interface Comment {
  id: number;
  author: string;
  content: string;
  createdAt: string;
}

export default function CommentSection({ postId }: { postId: number }) {
  const [comments, setComments] = useState<Comment[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/posts/${postId}/comments`)
      .then(res => res.json())
      .then(data => {
        setComments(data);
        setLoading(false);
      });
  }, [postId]);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);

    await fetch(`/api/posts/${postId}/comments`, {
      method: 'POST',
      body: JSON.stringify({
        author: formData.get('author'),
        content: formData.get('content'),
      }),
    });

    // 댓글 목록 새로고침
    const res = await fetch(`/api/posts/${postId}/comments`);
    const data = await res.json();
    setComments(data);
    e.currentTarget.reset();
  };

  if (loading) return <div>댓글 로딩 중...</div>;

  return (
    <div className="mt-12">
      <h2 className="text-2xl font-bold mb-4">댓글 {comments.length}개</h2>

      <form onSubmit={handleSubmit} className="mb-8">
        <input
          name="author"
          placeholder="이름"
          required
          className="border p-2 rounded w-full mb-2"
        />
        <textarea
          name="content"
          placeholder="댓글 내용"
          required
          className="border p-2 rounded w-full mb-2"
          rows={4}
        />
        <button type="submit" className="px-4 py-2 bg-blue-500 text-white rounded">
          댓글 작성
        </button>
      </form>

      <div className="space-y-4">
        {comments.map(comment => (
          <div key={comment.id} className="border rounded p-4">
            <div className="font-bold">{comment.author}</div>
            <p className="text-gray-700">{comment.content}</p>
            <time className="text-sm text-gray-500">
              {new Date(comment.createdAt).toLocaleString()}
            </time>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 10.8 성능 최적화

### 10.8.1 데이터 페칭 최적화

```typescript
// ❌ 순차적 페칭 (느림)
export default async function SlowPage() {
  const users = await fetch('/api/users').then(res => res.json());
  const posts = await fetch('/api/posts').then(res => res.json());
  const comments = await fetch('/api/comments').then(res => res.json());

  return <div>...</div>;
}

// ✅ 병렬 페칭 (빠름)
export default async function FastPage() {
  const [users, posts, comments] = await Promise.all([
    fetch('/api/users').then(res => res.json()),
    fetch('/api/posts').then(res => res.json()),
    fetch('/api/comments').then(res => res.json()),
  ]);

  return <div>...</div>;
}
```

### 10.8.2 Suspense를 활용한 Streaming

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <div>
      <h1>대시보드</h1>

      {/* 빠른 데이터는 바로 표시 */}
      <QuickStats />

      {/* 느린 데이터는 Suspense로 감싸기 */}
      <Suspense fallback={<div>분석 데이터 로딩 중...</div>}>
        <SlowAnalytics />
      </Suspense>

      <Suspense fallback={<div>차트 로딩 중...</div>}>
        <SlowChart />
      </Suspense>
    </div>
  );
}

async function QuickStats() {
  const stats = await fetch('/api/quick-stats').then(res => res.json());
  return <div>{stats.visitors} 방문자</div>;
}

async function SlowAnalytics() {
  // 느린 API (5초 걸림)
  const data = await fetch('/api/slow-analytics').then(res => res.json());
  return <div>{data.content}</div>;
}

async function SlowChart() {
  // 느린 API (3초 걸림)
  const data = await fetch('/api/slow-chart').then(res => res.json());
  return <div>{data.chart}</div>;
}
```

## 핵심 요약

1. **Server Components**
   - 서버에서만 실행
   - async/await 사용 가능
   - 클라이언트에 JavaScript 전송 안 됨

2. **주요 장점**
   - 번들 크기 감소
   - 데이터베이스 직접 접근
   - 민감정보 보호
   - 완벽한 SEO

3. **사용 가능**
   - 데이터베이스 쿼리
   - 파일 시스템 접근
   - 외부 API 호출
   - 환경변수 사용

4. **사용 불가능**
   - React Hooks
   - 이벤트 핸들러
   - 브라우저 API
   - React Context

5. **최적화**
   - 병렬 페칭 (Promise.all)
   - Suspense로 Streaming
   - 캐싱 전략 활용

## 연습 문제

### 문제 1
데이터베이스에서 사용자 목록을 가져와 표시하는 Server Component를 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/users/page.tsx
import { db } from '@/lib/db';

export default async function UsersPage() {
  const users = await db.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
    },
  });

  return (
    <div>
      <h1>사용자 목록</h1>
      <ul>
        {users.map(user => (
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
여러 API를 병렬로 호출하여 대시보드를 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const [users, posts, comments] = await Promise.all([
    fetch('https://api.example.com/users').then(res => res.json()),
    fetch('https://api.example.com/posts').then(res => res.json()),
    fetch('https://api.example.com/comments').then(res => res.json()),
  ]);

  return (
    <div>
      <h1>대시보드</h1>
      <p>사용자: {users.length}</p>
      <p>포스트: {posts.length}</p>
      <p>댓글: {comments.length}</p>
    </div>
  );
}
```
</details>

### 문제 3
파일 시스템에서 마크다운 파일을 읽어 표시하세요.

<details>
<summary>정답</summary>

```typescript
// app/docs/[slug]/page.tsx
import { readFile } from 'fs/promises';
import path from 'path';

interface Props {
  params: { slug: string };
}

export default async function DocPage({ params }: Props) {
  const filePath = path.join(process.cwd(), 'docs', `${params.slug}.md`);
  const content = await readFile(filePath, 'utf-8');

  return (
    <article>
      <pre>{content}</pre>
    </article>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- Client Components 심화
- `'use client'` 지시어
- 상호작용 구현
- 브라우저 API 사용

[Chapter 11. Client Components 심화 →](chapter11-client-components-deep.md)