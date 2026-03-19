# Chapter 12. Server와 Client 조합하기

## 12.1 컴포넌트 조합 기본 원칙

Server와 Client Components를 효과적으로 조합하는 것이 Next.js의 핵심입니다.

### 12.1.1 기본 규칙

```typescript
✅ Server Component는 기본값
✅ Client Component는 'use client'로 명시
✅ Server Component는 Client Component를 import 가능
❌ Client Component는 Server Component를 직접 import 불가
✅ 하지만 children props로 전달은 가능!
```

### 12.1.2 기본 패턴

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

      {/* Client Component 사용 */}
      <ClientButton initialCount={data.count} />
    </div>
  );
}
```

```typescript
// ClientButton.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function ClientButton({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount(count + 1)}>
      클릭 수: {count}
    </button>
  );
}
```

## 12.2 Props 전달 규칙

### 12.2.1 직렬화 가능한 Props만

```typescript
// ✅ 가능한 Props
- 문자열, 숫자, 불리언
- 배열, 객체 (JSON)
- Date (문자열로 변환됨)

// ❌ 불가능한 Props
- 함수
- 클래스 인스턴스
- Symbol
```

### 12.2.2 올바른 Props 전달

```typescript
// app/page.tsx (Server Component)
export default async function HomePage() {
  const posts = await fetch('https://api.example.com/posts')
    .then(res => res.json());

  return (
    <div>
      {/* ✅ JSON 직렬화 가능한 데이터 */}
      <PostsList posts={posts} />
    </div>
  );
}
```

```typescript
// components/PostsList.tsx (Client Component)
'use client';

interface Post {
  id: number;
  title: string;
}

export default function PostsList({ posts }: { posts: Post[] }) {
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 12.2.3 잘못된 Props 전달

```typescript
// ❌ 잘못된 예시
'use client';

export default function ClientComponent({
  onClick,  // 함수는 전달 불가!
  data,     // 클래스 인스턴스는 전달 불가!
}: {
  onClick: () => void;
  data: CustomClass;
}) {
  return <button onClick={onClick}>클릭</button>;
}
```

## 12.3 Server에서 Client로 데이터 전달

### 12.3.1 기본 패턴

```typescript
// app/products/[id]/page.tsx (Server Component)
import AddToCartButton from './AddToCartButton';

export default async function ProductPage({ params }: { params: { id: string } }) {
  // 서버에서 데이터 페칭
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.price}원</p>
      <img src={product.imageUrl} alt={product.name} />

      {/* Client Component에 데이터 전달 */}
      <AddToCartButton
        productId={product.id}
        productName={product.name}
        price={product.price}
      />
    </div>
  );
}
```

```typescript
// AddToCartButton.tsx (Client Component)
'use client';

import { useState } from 'react';

interface Props {
  productId: number;
  productName: string;
  price: number;
}

export default function AddToCartButton({ productId, productName, price }: Props) {
  const [quantity, setQuantity] = useState(1);
  const [added, setAdded] = useState(false);

  const handleAddToCart = async () => {
    await fetch('/api/cart', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ productId, quantity }),
    });

    setAdded(true);
    setTimeout(() => setAdded(false), 2000);
  };

  return (
    <div>
      <div>
        <button onClick={() => setQuantity(Math.max(1, quantity - 1))}>-</button>
        <span>{quantity}</span>
        <button onClick={() => setQuantity(quantity + 1)}>+</button>
      </div>

      <button onClick={handleAddToCart}>
        {added ? '장바구니에 추가됨!' : '장바구니 담기'}
      </button>

      <p>총 가격: {(price * quantity).toLocaleString()}원</p>
    </div>
  );
}
```

### 12.3.2 복잡한 데이터 전달

```typescript
// app/dashboard/page.tsx (Server Component)
import StatsChart from './StatsChart';

export default async function DashboardPage() {
  const stats = await fetch('https://api.example.com/stats')
    .then(res => res.json());

  const chartData = {
    labels: stats.map((s: any) => s.date),
    values: stats.map((s: any) => s.value),
  };

  return (
    <div>
      <h1>대시보드</h1>
      <StatsChart data={chartData} />
    </div>
  );
}
```

```typescript
// StatsChart.tsx (Client Component)
'use client';

interface ChartData {
  labels: string[];
  values: number[];
}

export default function StatsChart({ data }: { data: ChartData }) {
  // 차트 라이브러리 사용 (예: Chart.js)
  return (
    <div>
      {/* 차트 렌더링 */}
      <canvas id="chart" />
    </div>
  );
}
```

## 12.4 Client에서 Server Component 사용하기

### 12.4.1 ❌ 직접 import 불가능

```typescript
// ❌ 작동 안 함!
'use client';

import ServerComponent from './ServerComponent';

export default function ClientComponent() {
  return (
    <div>
      <ServerComponent />  {/* 에러! */}
    </div>
  );
}
```

### 12.4.2 ✅ children Props로 전달

```typescript
// app/page.tsx (Server Component)
import ClientWrapper from './ClientWrapper';
import ServerContent from './ServerContent';

export default function HomePage() {
  return (
    <ClientWrapper>
      {/* Server Component를 children으로 전달 */}
      <ServerContent />
    </ClientWrapper>
  );
}
```

```typescript
// ClientWrapper.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [show, setShow] = useState(true);

  return (
    <div>
      <button onClick={() => setShow(!show)}>토글</button>
      {show && children}
    </div>
  );
}
```

```typescript
// ServerContent.tsx (Server Component)
export default async function ServerContent() {
  const data = await fetch('https://api.example.com/data')
    .then(res => res.json());

  return <div>{data.content}</div>;
}
```

## 12.5 Context API 사용하기

### 12.5.1 Context Provider (Client Component)

```typescript
// providers/ThemeProvider.tsx
'use client';

import { createContext, useContext, useState } from 'react';

interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}
```

### 12.5.2 Layout에서 Provider 사용

```typescript
// app/layout.tsx (Server Component)
import { ThemeProvider } from '@/providers/ThemeProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

### 12.5.3 Client Component에서 Context 사용

```typescript
// components/ThemeToggle.tsx
'use client';

import { useTheme } from '@/providers/ThemeProvider';

export default function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();

  return (
    <button onClick={toggleTheme}>
      현재 테마: {theme === 'light' ? '☀️ 라이트' : '🌙 다크'}
    </button>
  );
}
```

## 12.6 실전 패턴

### 12.6.1 패턴 1: 데이터는 Server, 상호작용은 Client

```typescript
// app/posts/page.tsx (Server Component)
import PostCard from './PostCard';

export default async function PostsPage() {
  // 서버에서 데이터 페칭
  const posts = await fetch('https://api.example.com/posts')
    .then(res => res.json());

  return (
    <div className="grid grid-cols-3 gap-4">
      {posts.map((post: any) => (
        // 각 카드는 Client Component
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}
```

```typescript
// PostCard.tsx (Client Component)
'use client';

import { useState } from 'react';

interface Post {
  id: number;
  title: string;
  excerpt: string;
}

export default function PostCard({ post }: { post: Post }) {
  const [liked, setLiked] = useState(false);

  return (
    <div className="border rounded p-4">
      <h3>{post.title}</h3>
      <p>{post.excerpt}</p>
      <button
        onClick={() => setLiked(!liked)}
        style={{ color: liked ? 'red' : 'gray' }}
      >
        ❤️ {liked ? '좋아요' : '좋아요 하기'}
      </button>
    </div>
  );
}
```

### 12.6.2 패턴 2: Wrapper 패턴

```typescript
// app/page.tsx (Server Component)
import ClientSidebar from './ClientSidebar';
import ServerContent from './ServerContent';

export default function HomePage() {
  return (
    <div className="flex">
      {/* Client Component (사이드바) */}
      <ClientSidebar>
        {/* Server Component (메뉴 항목들) */}
        <ServerMenuItems />
      </ClientSidebar>

      {/* Server Component (메인 콘텐츠) */}
      <main className="flex-1">
        <ServerContent />
      </main>
    </div>
  );
}
```

```typescript
// ClientSidebar.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function ClientSidebar({ children }: { children: React.ReactNode }) {
  const [collapsed, setCollapsed] = useState(false);

  return (
    <aside className={collapsed ? 'w-16' : 'w-64'}>
      <button onClick={() => setCollapsed(!collapsed)}>
        {collapsed ? '→' : '←'}
      </button>
      {!collapsed && children}
    </aside>
  );
}
```

```typescript
// ServerMenuItems.tsx (Server Component)
export default async function ServerMenuItems() {
  const menu = await fetch('https://api.example.com/menu')
    .then(res => res.json());

  return (
    <ul>
      {menu.map((item: any) => (
        <li key={item.id}>{item.label}</li>
      ))}
    </ul>
  );
}
```

### 12.6.3 패턴 3: 레이어드 아키텍처

```typescript
// app/shop/[category]/page.tsx (Server Component - 데이터 레이어)
import ProductGrid from './ProductGrid';

export default async function CategoryPage({ params }: { params: { category: string } }) {
  // 1. 서버에서 데이터 페칭
  const products = await fetch(`https://api.example.com/products?category=${params.category}`)
    .then(res => res.json());

  // 2. Client Component에 전달
  return (
    <div>
      <h1>{params.category} 카테고리</h1>
      <ProductGrid products={products} />
    </div>
  );
}
```

```typescript
// ProductGrid.tsx (Client Component - 프레젠테이션 레이어)
'use client';

import { useState } from 'react';
import ProductCard from './ProductCard';

interface Product {
  id: number;
  name: string;
  price: number;
}

export default function ProductGrid({ products }: { products: Product[] }) {
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  const sortedProducts = [...products].sort((a, b) => {
    if (sortBy === 'name') return a.name.localeCompare(b.name);
    return a.price - b.price;
  });

  return (
    <div>
      <div>
        <button onClick={() => setSortBy('name')}>이름순</button>
        <button onClick={() => setSortBy('price')}>가격순</button>
      </div>

      <div className="grid grid-cols-3 gap-4">
        {sortedProducts.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

```typescript
// ProductCard.tsx (Client Component - UI 컴포넌트)
'use client';

import { useState } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
}

export default function ProductCard({ product }: { product: Product }) {
  const [quantity, setQuantity] = useState(1);

  const addToCart = () => {
    fetch('/api/cart', {
      method: 'POST',
      body: JSON.stringify({ productId: product.id, quantity }),
    });
  };

  return (
    <div className="border rounded p-4">
      <h3>{product.name}</h3>
      <p>{product.price.toLocaleString()}원</p>

      <input
        type="number"
        value={quantity}
        onChange={(e) => setQuantity(Number(e.target.value))}
        min="1"
      />

      <button onClick={addToCart}>장바구니 담기</button>
    </div>
  );
}
```

## 12.7 성능 최적화

### 12.7.1 불필요한 Client Component 줄이기

```typescript
// ❌ 나쁜 패턴: 전체가 Client Component
'use client';

export default function PostsPage() {
  const [liked, setLiked] = useState(false);

  // 데이터 페칭도 클라이언트에서
  const [posts, setPosts] = useState([]);
  useEffect(() => {
    fetch('/api/posts').then(res => res.json()).then(setPosts);
  }, []);

  return (
    <div>
      {posts.map(post => (
        <div key={post.id}>
          <h2>{post.title}</h2>
          <button onClick={() => setLiked(!liked)}>❤️</button>
        </div>
      ))}
    </div>
  );
}
```

```typescript
// ✅ 좋은 패턴: Server에서 데이터, Client는 최소한으로
// page.tsx (Server Component)
export default async function PostsPage() {
  // 서버에서 데이터 페칭
  const posts = await fetch('https://api.example.com/posts')
    .then(res => res.json());

  return (
    <div>
      {posts.map((post: any) => (
        <div key={post.id}>
          <h2>{post.title}</h2>
          {/* Client Component는 필요한 부분만 */}
          <LikeButton postId={post.id} />
        </div>
      ))}
    </div>
  );
}

// LikeButton.tsx (Client Component)
'use client';
import { useState } from 'react';

export default function LikeButton({ postId }: { postId: number }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>❤️</button>;
}
```

### 12.7.2 Suspense 활용

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <div>
      <h1>대시보드</h1>

      {/* 빠른 컴포넌트 */}
      <QuickStats />

      {/* 느린 컴포넌트는 Suspense로 감싸기 */}
      <Suspense fallback={<div>차트 로딩 중...</div>}>
        <SlowChart />
      </Suspense>

      <Suspense fallback={<div>활동 로딩 중...</div>}>
        <RecentActivity />
      </Suspense>
    </div>
  );
}

async function QuickStats() {
  const stats = await fetch('/api/quick-stats').then(res => res.json());
  return <div>방문자: {stats.visitors}</div>;
}

async function SlowChart() {
  await new Promise(resolve => setTimeout(resolve, 3000));
  return <div>차트 컴포넌트</div>;
}

async function RecentActivity() {
  await new Promise(resolve => setTimeout(resolve, 2000));
  return <div>최근 활동</div>;
}
```

## 12.8 실전 예제: 블로그 댓글 시스템

```typescript
// app/blog/[slug]/page.tsx (Server Component)
import { db } from '@/lib/db';
import CommentSection from './CommentSection';

export default async function BlogPostPage({ params }: { params: { slug: string } }) {
  // 서버에서 포스트와 댓글 가져오기
  const post = await db.post.findUnique({
    where: { slug: params.slug },
    include: {
      comments: {
        orderBy: { createdAt: 'desc' },
      },
    },
  });

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />

      {/* Client Component에 댓글 전달 */}
      <CommentSection
        postId={post.id}
        initialComments={post.comments}
      />
    </article>
  );
}
```

```typescript
// CommentSection.tsx (Client Component)
'use client';

import { useState } from 'react';

interface Comment {
  id: number;
  author: string;
  content: string;
  createdAt: string;
}

interface Props {
  postId: number;
  initialComments: Comment[];
}

export default function CommentSection({ postId, initialComments }: Props) {
  const [comments, setComments] = useState(initialComments);
  const [author, setAuthor] = useState('');
  const [content, setContent] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);

    try {
      const res = await fetch('/api/comments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ postId, author, content }),
      });

      const newComment = await res.json();

      setComments(prev => [newComment, ...prev]);
      setAuthor('');
      setContent('');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="mt-8">
      <h2 className="text-2xl font-bold mb-4">댓글 {comments.length}개</h2>

      <form onSubmit={handleSubmit} className="mb-8">
        <input
          type="text"
          value={author}
          onChange={(e) => setAuthor(e.target.value)}
          placeholder="이름"
          required
          className="border p-2 rounded w-full mb-2"
        />
        <textarea
          value={content}
          onChange={(e) => setContent(e.target.value)}
          placeholder="댓글 내용"
          required
          rows={4}
          className="border p-2 rounded w-full mb-2"
        />
        <button
          type="submit"
          disabled={submitting}
          className="px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-50"
        >
          {submitting ? '작성 중...' : '댓글 작성'}
        </button>
      </form>

      <div className="space-y-4">
        {comments.map(comment => (
          <div key={comment.id} className="border rounded p-4">
            <div className="font-bold">{comment.author}</div>
            <p className="text-gray-700 my-2">{comment.content}</p>
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

## 핵심 요약

1. **조합 규칙**
   - Server → Client: ✅ 가능
   - Client → Server: ❌ 직접 불가, children으로 ✅

2. **Props 전달**
   - JSON 직렬화 가능한 데이터만
   - 함수, 클래스 인스턴스 불가

3. **패턴**
   - 데이터는 Server, 상호작용은 Client
   - Wrapper 패턴 (children props)
   - 레이어드 아키텍처

4. **Context**
   - Provider는 Client Component
   - Layout에서 Provider 사용

5. **최적화**
   - Client Component 최소화
   - Suspense로 스트리밍
   - 필요한 부분만 클라이언트로

## 연습 문제

### 문제 1
Server Component에서 데이터를 가져와 Client Component에 전달하세요.

<details>
<summary>정답</summary>

```typescript
// page.tsx (Server)
export default async function Page() {
  const data = await fetch('https://api.example.com/data')
    .then(res => res.json());

  return <ClientComponent data={data} />;
}

// ClientComponent.tsx (Client)
'use client';
export default function ClientComponent({ data }: { data: any }) {
  return <div>{data.title}</div>;
}
```
</details>

### 문제 2
Client Component에서 Server Component를 children으로 사용하세요.

<details>
<summary>정답</summary>

```typescript
// page.tsx (Server)
export default function Page() {
  return (
    <ClientWrapper>
      <ServerContent />
    </ClientWrapper>
  );
}

// ClientWrapper.tsx (Client)
'use client';
import { useState } from 'react';

export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [show, setShow] = useState(true);
  return (
    <div>
      <button onClick={() => setShow(!show)}>토글</button>
      {show && children}
    </div>
  );
}

// ServerContent.tsx (Server)
export default async function ServerContent() {
  const data = await fetch('https://api.example.com/data')
    .then(res => res.json());
  return <div>{data.content}</div>;
}
```
</details>

## 다음 단계

Part 04 완료! 다음 Part에서는:
- CSS Modules
- Tailwind CSS
- Styled Components
- 스타일링 모범 사례

[Part 05: 스타일링 →](../part05-styling/chapter13-css-modules.md)