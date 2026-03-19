# Chapter 8. Client Components에서 데이터 가져오기

## 8.1 Client Components란?

**Client Component**는 브라우저에서 실행되는 컴포넌트입니다. `'use client'` 지시어로 선언합니다.

### 8.1.1 Client Component가 필요한 경우

```typescript
// ✅ Client Component가 필요한 경우
- useState, useEffect 같은 React 훅 사용
- 이벤트 핸들러 (onClick, onChange 등)
- 브라우저 API (localStorage, window 등)
- 실시간 데이터 업데이트
- 사용자 상호작용
```

### 8.1.2 Client Component 선언

```typescript
// app/components/Counter.tsx
'use client';  // 👈 이 줄이 있으면 Client Component!

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>카운트: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        증가
      </button>
    </div>
  );
}
```

## 8.2 `useEffect` + `fetch`

전통적인 React 방식으로 데이터를 가져옵니다.

### 8.2.1 기본 패턴

```typescript
'use client';

import { useState, useEffect } from 'react';

interface Post {
  id: number;
  title: string;
  body: string;
}

export default function PostsList() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('https://jsonplaceholder.typicode.com/posts')
      .then(res => res.json())
      .then(data => {
        setPosts(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>로딩 중...</div>;
  if (error) return <div>에러: {error}</div>;

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 8.2.2 async/await 사용

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function PostsList() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchPosts = async () => {
      try {
        const res = await fetch('https://api.example.com/posts');
        const data = await res.json();
        setPosts(data);
      } catch (error) {
        console.error('Failed to fetch posts:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchPosts();
  }, []);

  if (loading) return <div>로딩 중...</div>;

  return (
    <ul>
      {posts.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 8.2.3 의존성 배열 활용

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function UserPosts({ userId }: { userId: number }) {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const fetchPosts = async () => {
      const res = await fetch(`https://api.example.com/users/${userId}/posts`);
      const data = await res.json();
      setPosts(data);
    };

    fetchPosts();
  }, [userId]);  // userId가 변경되면 다시 페칭

  return (
    <ul>
      {posts.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## 8.3 SWR (Stale-While-Revalidate)

Vercel에서 만든 데이터 페칭 라이브러리입니다.

### 8.3.1 SWR 설치

```bash
npm install swr
```

### 8.3.2 기본 사용법

```typescript
'use client';

import useSWR from 'swr';

// fetcher 함수
const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function PostsList() {
  const { data, error, isLoading } = useSWR(
    'https://jsonplaceholder.typicode.com/posts',
    fetcher
  );

  if (isLoading) return <div>로딩 중...</div>;
  if (error) return <div>에러 발생</div>;

  return (
    <ul>
      {data.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 8.3.3 SWR의 장점

```typescript
'use client';

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function PostsList() {
  const { data, error, isLoading, mutate } = useSWR(
    'https://api.example.com/posts',
    fetcher,
    {
      refreshInterval: 3000,        // 3초마다 자동 새로고침
      revalidateOnFocus: true,      // 탭 포커스 시 재검증
      revalidateOnReconnect: true,  // 네트워크 재연결 시 재검증
    }
  );

  // 수동으로 데이터 새로고침
  const handleRefresh = () => {
    mutate();
  };

  if (isLoading) return <div>로딩 중...</div>;
  if (error) return <div>에러 발생</div>;

  return (
    <div>
      <button onClick={handleRefresh}>새로고침</button>
      <ul>
        {data.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 8.3.4 전역 fetcher 설정

```typescript
// app/layout.tsx (Client Component로 만들기)
'use client';

import { SWRConfig } from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <SWRConfig value={{ fetcher }}>
          {children}
        </SWRConfig>
      </body>
    </html>
  );
}
```

```typescript
// 이제 fetcher 없이 사용 가능
'use client';

import useSWR from 'swr';

export default function PostsList() {
  const { data, error, isLoading } = useSWR('https://api.example.com/posts');

  if (isLoading) return <div>로딩 중...</div>;
  if (error) return <div>에러 발생</div>;

  return <ul>{data.map((post: any) => <li key={post.id}>{post.title}</li>)}</ul>;
}
```

### 8.3.5 Mutation (데이터 수정)

```typescript
'use client';

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function TodoList() {
  const { data: todos, mutate } = useSWR('/api/todos', fetcher);

  const addTodo = async (title: string) => {
    // Optimistic Update (낙관적 업데이트)
    const newTodo = { id: Date.now(), title, completed: false };
    mutate([...todos, newTodo], false);  // 즉시 UI 업데이트

    // 서버에 전송
    await fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify(newTodo),
    });

    // 서버 데이터로 재검증
    mutate();
  };

  const deleteTodo = async (id: number) => {
    // Optimistic Update
    mutate(todos.filter((todo: any) => todo.id !== id), false);

    // 서버에 전송
    await fetch(`/api/todos/${id}`, { method: 'DELETE' });

    // 재검증
    mutate();
  };

  return (
    <div>
      <ul>
        {todos?.map((todo: any) => (
          <li key={todo.id}>
            {todo.title}
            <button onClick={() => deleteTodo(todo.id)}>삭제</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## 8.4 TanStack Query (React Query)

가장 인기 있는 데이터 페칭 라이브러리입니다.

### 8.4.1 설치

```bash
npm install @tanstack/react-query
```

### 8.4.2 Provider 설정

```typescript
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';

export default function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

```typescript
// app/layout.tsx
import Providers from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

### 8.4.3 useQuery 기본 사용

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

interface Post {
  id: number;
  title: string;
  body: string;
}

const fetchPosts = async (): Promise<Post[]> => {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts');
  return res.json();
};

export default function PostsList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  if (isLoading) return <div>로딩 중...</div>;
  if (error) return <div>에러 발생</div>;

  return (
    <ul>
      {data?.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 8.4.4 동적 쿼리

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

const fetchUser = async (userId: number) => {
  const res = await fetch(`https://api.example.com/users/${userId}`);
  return res.json();
};

export default function UserProfile({ userId }: { userId: number }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],  // userId가 변경되면 자동으로 다시 페칭
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <div>로딩 중...</div>;

  return <div>{user.name}</div>;
}
```

### 8.4.5 useMutation (데이터 수정)

```typescript
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

const createTodo = async (newTodo: Omit<Todo, 'id'>): Promise<Todo> => {
  const res = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(newTodo),
  });
  return res.json();
};

export default function TodoForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: createTodo,
    onSuccess: () => {
      // 성공 시 todos 쿼리 무효화 (자동 재페칭)
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const title = formData.get('title') as string;

    mutation.mutate({ title, completed: false });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" type="text" placeholder="할 일" />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? '추가 중...' : '추가'}
      </button>
      {mutation.isError && <div>에러 발생!</div>}
      {mutation.isSuccess && <div>추가 완료!</div>}
    </form>
  );
}
```

### 8.4.6 병렬 쿼리

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

export default function Dashboard() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(res => res.json()),
  });

  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(res => res.json()),
  });

  const { data: comments } = useQuery({
    queryKey: ['comments'],
    queryFn: () => fetch('/api/comments').then(res => res.json()),
  });

  return (
    <div>
      <p>사용자: {users?.length || 0}</p>
      <p>포스트: {posts?.length || 0}</p>
      <p>댓글: {comments?.length || 0}</p>
    </div>
  );
}
```

### 8.4.7 의존적 쿼리

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

export default function UserPosts({ userId }: { userId: number }) {
  // 1. 먼저 사용자 정보 가져오기
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(res => res.json()),
  });

  // 2. 사용자 정보가 있을 때만 포스트 가져오기
  const { data: posts } = useQuery({
    queryKey: ['posts', user?.id],
    queryFn: () => fetch(`/api/users/${user.id}/posts`).then(res => res.json()),
    enabled: !!user,  // user가 있을 때만 실행
  });

  return (
    <div>
      <h2>{user?.name}</h2>
      {posts?.map((post: any) => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}
```

## 8.5 로딩 상태 관리

### 8.5.1 기본 로딩 UI

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

export default function PostsList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(res => res.json()),
  });

  if (isLoading) {
    return (
      <div className="flex justify-center items-center min-h-screen">
        <div className="animate-spin rounded-full h-12 w-12 border-t-2 border-blue-500"></div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-red-500">
        에러가 발생했습니다: {error.message}
      </div>
    );
  }

  return (
    <ul>
      {data.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 8.5.2 스켈레톤 UI

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

function Skeleton() {
  return (
    <div className="animate-pulse">
      <div className="h-4 bg-gray-300 rounded mb-2"></div>
      <div className="h-4 bg-gray-300 rounded w-2/3"></div>
    </div>
  );
}

export default function PostsList() {
  const { data, isLoading } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(res => res.json()),
  });

  if (isLoading) {
    return (
      <div className="space-y-4">
        {[...Array(5)].map((_, i) => (
          <Skeleton key={i} />
        ))}
      </div>
    );
  }

  return (
    <ul>
      {data.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 8.5.3 에러 바운더리

```typescript
// components/ErrorBoundary.tsx
'use client';

import { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }

    return this.props.children;
  }
}
```

```typescript
// 사용
import { ErrorBoundary } from '@/components/ErrorBoundary';

export default function Page() {
  return (
    <ErrorBoundary
      fallback={<div>오류가 발생했습니다</div>}
    >
      <PostsList />
    </ErrorBoundary>
  );
}
```

## 8.6 실전 예제: 검색 기능

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

interface SearchResult {
  id: number;
  title: string;
  description: string;
}

const searchPosts = async (query: string): Promise<SearchResult[]> => {
  const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
  return res.json();
};

export default function SearchPage() {
  const [query, setQuery] = useState('');
  const [debouncedQuery, setDebouncedQuery] = useState('');

  // 디바운싱
  const handleSearch = (value: string) => {
    setQuery(value);
    const timer = setTimeout(() => {
      setDebouncedQuery(value);
    }, 500);
    return () => clearTimeout(timer);
  };

  const { data, isLoading, error } = useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn: () => searchPosts(debouncedQuery),
    enabled: debouncedQuery.length > 0,  // 쿼리가 있을 때만 실행
  });

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="검색..."
        className="border p-2 rounded w-full"
      />

      {isLoading && <div>검색 중...</div>}

      {error && <div>에러 발생</div>}

      {data && (
        <ul className="mt-4 space-y-2">
          {data.map(result => (
            <li key={result.id} className="border p-4 rounded">
              <h3 className="font-bold">{result.title}</h3>
              <p className="text-gray-600">{result.description}</p>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## 8.7 SWR vs React Query 비교

| 기능 | SWR | React Query |
|------|-----|-------------|
| **번들 크기** | 작음 (5KB) | 중간 (13KB) |
| **학습 곡선** | 쉬움 | 중간 |
| **기능** | 기본적 | 풍부함 |
| **Mutation** | 수동 | useMutation 내장 |
| **DevTools** | 없음 | 있음 |
| **캐싱** | 간단 | 강력함 |
| **추천 용도** | 간단한 앱 | 복잡한 앱 |

## 핵심 요약

1. **Client Components**
   - `'use client'` 지시어 필요
   - 브라우저에서 실행
   - 상호작용 가능

2. **데이터 페칭 방법**
   - `useEffect` + `fetch`: 기본
   - SWR: 간단하고 가벼움
   - React Query: 강력하고 기능 풍부

3. **SWR 장점**
   - 자동 재검증
   - 포커스/재연결 시 새로고침
   - Optimistic Update

4. **React Query 장점**
   - 강력한 캐싱
   - useMutation 내장
   - DevTools 지원

5. **로딩 상태**
   - 스켈레톤 UI
   - 에러 바운더리
   - 로딩 인디케이터

## 연습 문제

### 문제 1
SWR을 사용하여 포스트 목록을 가져오고, 3초마다 자동 새로고침하세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function PostsList() {
  const { data, error, isLoading } = useSWR(
    'https://jsonplaceholder.typicode.com/posts',
    fetcher,
    { refreshInterval: 3000 }
  );

  if (isLoading) return <div>로딩 중...</div>;
  if (error) return <div>에러 발생</div>;

  return (
    <ul>
      {data.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```
</details>

### 문제 2
React Query를 사용하여 TODO 추가 기능을 구현하세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';

const createTodo = async (title: string) => {
  const res = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title }),
  });
  return res.json();
};

export default function TodoForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: createTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const title = formData.get('title') as string;
    mutation.mutate(title);
    e.currentTarget.reset();
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" required />
      <button type="submit">추가</button>
    </form>
  );
}
```
</details>

### 문제 3
로딩 스켈레톤 UI를 만드세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

function Skeleton() {
  return (
    <div className="border p-4 rounded animate-pulse">
      <div className="h-4 bg-gray-300 rounded mb-2"></div>
      <div className="h-4 bg-gray-300 rounded w-3/4"></div>
    </div>
  );
}

export default function PostsList() {
  const { data, isLoading } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(res => res.json()),
  });

  if (isLoading) {
    return (
      <div className="space-y-4">
        {[...Array(5)].map((_, i) => (
          <Skeleton key={i} />
        ))}
      </div>
    );
  }

  return (
    <div className="space-y-4">
      {data.map((post: any) => (
        <div key={post.id} className="border p-4 rounded">
          <h3>{post.title}</h3>
        </div>
      ))}
    </div>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- SSR, SSG, ISR 심화
- 각 전략의 사용 시기
- 성능 최적화

[Chapter 9. SSR, SSG, ISR 이해하기 →](chapter09-rendering-strategies.md)