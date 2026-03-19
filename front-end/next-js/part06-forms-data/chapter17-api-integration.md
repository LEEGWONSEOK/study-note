# Chapter 17. 외부 API 연동

## 17.1 REST API 기본

### 17.1.1 fetch API 사용

```typescript
// Server Component에서 API 호출
export default async function PostsPage() {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts');

  if (!res.ok) {
    throw new Error('Failed to fetch posts');
  }

  const posts = await res.json();

  return (
    <ul>
      {posts.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 17.1.2 HTTP 메서드

```typescript
// GET
const res = await fetch('https://api.example.com/posts');

// POST
const res = await fetch('https://api.example.com/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ title: 'New Post' }),
});

// PUT
const res = await fetch(`https://api.example.com/posts/${id}`, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ title: 'Updated Post' }),
});

// DELETE
const res = await fetch(`https://api.example.com/posts/${id}`, {
  method: 'DELETE',
});
```

## 17.2 환경변수 관리

### 17.2.1 환경변수 설정

```bash
# .env.local
# 서버 전용 (안전)
API_KEY=your-secret-api-key
DATABASE_URL=postgresql://...

# 클라이언트에서도 접근 가능 (NEXT_PUBLIC_ 접두사)
NEXT_PUBLIC_API_URL=https://api.example.com
```

### 17.2.2 환경변수 사용

```typescript
// Server Component (서버에서만 실행)
export default async function Page() {
  // ✅ 서버 전용 환경변수 (안전!)
  const apiKey = process.env.API_KEY;
  const dbUrl = process.env.DATABASE_URL;

  // ✅ 공개 환경변수
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;

  const res = await fetch(`${apiUrl}/data`, {
    headers: {
      'Authorization': `Bearer ${apiKey}`,
    },
  });

  const data = await res.json();
  return <div>{data.title}</div>;
}
```

```typescript
// Client Component
'use client';

export default function ClientComponent() {
  // ✅ 클라이언트에서는 NEXT_PUBLIC_만 접근 가능
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;

  // ❌ undefined! 클라이언트에서 접근 불가
  const apiKey = process.env.API_KEY;

  return <div>...</div>;
}
```

## 17.3 에러 핸들링

### 17.3.1 기본 에러 처리

```typescript
export default async function PostsPage() {
  try {
    const res = await fetch('https://api.example.com/posts');

    if (!res.ok) {
      throw new Error(`HTTP error! status: ${res.status}`);
    }

    const posts = await res.json();

    return (
      <ul>
        {posts.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    );
  } catch (error) {
    console.error('Failed to fetch posts:', error);
    return <div>데이터를 불러오는데 실패했습니다.</div>;
  }
}
```

### 17.3.2 상태 코드별 처리

```typescript
export default async function PostPage({ params }: { params: { id: string } }) {
  const res = await fetch(`https://api.example.com/posts/${params.id}`);

  if (res.status === 404) {
    notFound(); // 404 페이지로
  }

  if (res.status === 401) {
    redirect('/login'); // 로그인 페이지로
  }

  if (!res.ok) {
    throw new Error('Failed to fetch post');
  }

  const post = await res.json();
  return <article>{post.title}</article>;
}
```

### 17.3.3 재시도 로직

```typescript
async function fetchWithRetry(url: string, retries = 3): Promise<Response> {
  for (let i = 0; i < retries; i++) {
    try {
      const res = await fetch(url);

      if (res.ok) {
        return res;
      }

      if (res.status === 404) {
        throw new Error('Not found');
      }

      // 서버 에러면 재시도
      if (res.status >= 500) {
        console.log(`Retry ${i + 1}/${retries}`);
        await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
        continue;
      }

      throw new Error(`HTTP error! status: ${res.status}`);
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }

  throw new Error('Max retries reached');
}
```

## 17.4 API 프록시 패턴

### 17.4.1 Route Handler로 프록시

```typescript
// app/api/posts/route.ts
export async function GET() {
  const apiKey = process.env.API_KEY; // 서버에서만 접근

  const res = await fetch('https://api.example.com/posts', {
    headers: {
      'Authorization': `Bearer ${apiKey}`,
    },
  });

  const data = await res.json();

  return Response.json(data);
}
```

```typescript
// Client Component에서 호출
'use client';

import { useEffect, useState } from 'react';

export default function PostsList() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    // 내부 API 호출 (API 키 노출 안 됨!)
    fetch('/api/posts')
      .then(res => res.json())
      .then(setPosts);
  }, []);

  return (
    <ul>
      {posts.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 17.4.2 동적 프록시

```typescript
// app/api/proxy/[...path]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { path: string[] } }
) {
  const apiKey = process.env.API_KEY;
  const path = params.path.join('/');

  const res = await fetch(`https://api.example.com/${path}`, {
    headers: {
      'Authorization': `Bearer ${apiKey}`,
    },
  });

  const data = await res.json();
  return Response.json(data);
}

// 사용: /api/proxy/posts/123 → https://api.example.com/posts/123
```

## 17.5 캐싱 전략

### 17.5.1 기본 캐싱 (SSG)

```typescript
// 기본적으로 캐시됨 (빌드 시 한 번만 호출)
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return <ul>{/* ... */}</ul>;
}
```

### 17.5.2 캐시 비활성화 (SSR)

```typescript
// 매 요청마다 호출
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    cache: 'no-store', // 캐시 사용 안 함
  });

  const posts = await res.json();
  return <ul>{/* ... */}</ul>;
}
```

### 17.5.3 시간 기반 재검증 (ISR)

```typescript
// 60초마다 재검증
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 }, // 60초
  });

  const posts = await res.json();
  return <ul>{/* ... */}</ul>;
}
```

### 17.5.4 태그 기반 캐싱

```typescript
// 태그 붙이기
const res = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});
```

```typescript
// Server Action에서 재검증
'use server';

import { revalidateTag } from 'next/cache';

export async function createPost(formData: FormData) {
  // 포스트 생성...

  revalidateTag('posts'); // 'posts' 태그 캐시 무효화
}
```

## 17.6 실전 예제: GitHub API 연동

### 17.6.1 사용자 정보 가져오기

```typescript
// app/github/[username]/page.tsx
interface GitHubUser {
  login: string;
  name: string;
  avatar_url: string;
  bio: string;
  public_repos: number;
  followers: number;
}

export default async function GitHubUserPage({
  params,
}: {
  params: { username: string };
}) {
  const res = await fetch(`https://api.github.com/users/${params.username}`, {
    next: { revalidate: 3600 }, // 1시간마다 재검증
  });

  if (!res.ok) {
    return <div>사용자를 찾을 수 없습니다.</div>;
  }

  const user: GitHubUser = await res.json();

  return (
    <div className="max-w-2xl mx-auto p-8">
      <div className="flex items-center gap-4 mb-6">
        <img
          src={user.avatar_url}
          alt={user.name}
          className="w-24 h-24 rounded-full"
        />
        <div>
          <h1 className="text-3xl font-bold">{user.name}</h1>
          <p className="text-gray-600">@{user.login}</p>
        </div>
      </div>

      <p className="mb-4">{user.bio}</p>

      <div className="flex gap-4">
        <div>
          <span className="font-bold">{user.public_repos}</span> 저장소
        </div>
        <div>
          <span className="font-bold">{user.followers}</span> 팔로워
        </div>
      </div>
    </div>
  );
}
```

### 17.6.2 저장소 목록 가져오기

```typescript
// app/github/[username]/repos/page.tsx
interface Repository {
  id: number;
  name: string;
  description: string;
  stargazers_count: number;
  language: string;
}

export default async function ReposPage({
  params,
}: {
  params: { username: string };
}) {
  const res = await fetch(
    `https://api.github.com/users/${params.username}/repos?sort=updated`,
    {
      next: { revalidate: 3600 },
    }
  );

  const repos: Repository[] = await res.json();

  return (
    <div className="max-w-4xl mx-auto p-8">
      <h1 className="text-3xl font-bold mb-8">
        {params.username}의 저장소
      </h1>

      <div className="space-y-4">
        {repos.map(repo => (
          <div key={repo.id} className="border rounded-lg p-4">
            <h2 className="text-xl font-bold mb-2">{repo.name}</h2>
            <p className="text-gray-600 mb-2">{repo.description}</p>
            <div className="flex gap-4 text-sm text-gray-500">
              <span>⭐ {repo.stargazers_count}</span>
              {repo.language && <span>{repo.language}</span>}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 17.7 API 클라이언트 패턴

### 17.7.1 재사용 가능한 API 클라이언트

```typescript
// lib/api-client.ts
class APIClient {
  private baseURL: string;
  private apiKey: string;

  constructor() {
    this.baseURL = process.env.NEXT_PUBLIC_API_URL || '';
    this.apiKey = process.env.API_KEY || '';
  }

  private async request(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<Response> {
    const url = `${this.baseURL}${endpoint}`;

    const headers = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${this.apiKey}`,
      ...options.headers,
    };

    const res = await fetch(url, {
      ...options,
      headers,
    });

    if (!res.ok) {
      throw new Error(`API Error: ${res.status}`);
    }

    return res;
  }

  async get<T>(endpoint: string, options?: RequestInit): Promise<T> {
    const res = await this.request(endpoint, { ...options, method: 'GET' });
    return res.json();
  }

  async post<T>(endpoint: string, data: any, options?: RequestInit): Promise<T> {
    const res = await this.request(endpoint, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data),
    });
    return res.json();
  }

  async put<T>(endpoint: string, data: any, options?: RequestInit): Promise<T> {
    const res = await this.request(endpoint, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data),
    });
    return res.json();
  }

  async delete(endpoint: string, options?: RequestInit): Promise<void> {
    await this.request(endpoint, { ...options, method: 'DELETE' });
  }
}

export const apiClient = new APIClient();
```

### 17.7.2 API 클라이언트 사용

```typescript
// app/posts/page.tsx
import { apiClient } from '@/lib/api-client';

interface Post {
  id: number;
  title: string;
  content: string;
}

export default async function PostsPage() {
  const posts = await apiClient.get<Post[]>('/posts');

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## 17.8 실전 예제: 날씨 API

```typescript
// app/weather/[city]/page.tsx
interface WeatherData {
  name: string;
  main: {
    temp: number;
    feels_like: number;
    humidity: number;
  };
  weather: Array<{
    description: string;
    icon: string;
  }>;
}

export default async function WeatherPage({
  params,
}: {
  params: { city: string };
}) {
  const apiKey = process.env.OPENWEATHER_API_KEY;

  const res = await fetch(
    `https://api.openweathermap.org/data/2.5/weather?q=${params.city}&appid=${apiKey}&units=metric&lang=kr`,
    {
      next: { revalidate: 600 }, // 10분마다
    }
  );

  if (!res.ok) {
    return <div>날씨 정보를 불러올 수 없습니다.</div>;
  }

  const data: WeatherData = await res.json();

  return (
    <div className="max-w-md mx-auto p-8">
      <h1 className="text-3xl font-bold mb-4">{data.name} 날씨</h1>

      <div className="bg-blue-100 rounded-lg p-6">
        <div className="flex items-center justify-between mb-4">
          <div className="text-5xl font-bold">{Math.round(data.main.temp)}°C</div>
          <img
            src={`https://openweathermap.org/img/wn/${data.weather[0].icon}@2x.png`}
            alt="날씨 아이콘"
          />
        </div>

        <p className="text-xl mb-4">{data.weather[0].description}</p>

        <div className="space-y-2">
          <p>체감온도: {Math.round(data.main.feels_like)}°C</p>
          <p>습도: {data.main.humidity}%</p>
        </div>
      </div>
    </div>
  );
}
```

## 17.9 모범 사례

### 17.9.1 타입 정의

```typescript
// types/api.ts
export interface Post {
  id: number;
  title: string;
  content: string;
  createdAt: string;
}

export interface User {
  id: number;
  name: string;
  email: string;
}
```

### 17.9.2 에러 경계

```typescript
// app/posts/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>데이터를 불러오는데 실패했습니다</h2>
      <button onClick={reset}>다시 시도</button>
    </div>
  );
}
```

### 17.9.3 로딩 상태

```typescript
// app/posts/loading.tsx
export default function Loading() {
  return (
    <div className="space-y-4">
      {[...Array(5)].map((_, i) => (
        <div key={i} className="animate-pulse">
          <div className="h-4 bg-gray-300 rounded w-3/4 mb-2"></div>
          <div className="h-4 bg-gray-300 rounded w-1/2"></div>
        </div>
      ))}
    </div>
  );
}
```

## 핵심 요약

1. **API 호출**
   - Server Component에서 fetch
   - 환경변수로 API 키 관리
   - 타입 안전성

2. **에러 처리**
   - try-catch
   - 상태 코드별 처리
   - 재시도 로직

3. **프록시 패턴**
   - Route Handler로 프록시
   - API 키 숨기기
   - CORS 우회

4. **캐싱**
   - 기본 캐싱 (SSG)
   - `cache: 'no-store'` (SSR)
   - `revalidate` (ISR)

5. **모범 사례**
   - API 클라이언트 패턴
   - 타입 정의
   - 에러 경계
   - 로딩 상태

## 연습 문제

### 문제 1
GitHub API로 사용자 정보를 가져오세요.

<details>
<summary>정답</summary>

```typescript
export default async function Page({ params }: { params: { username: string } }) {
  const res = await fetch(`https://api.github.com/users/${params.username}`);
  const user = await res.json();

  return (
    <div>
      <h1>{user.name}</h1>
      <p>@{user.login}</p>
    </div>
  );
}
```
</details>

### 문제 2
API 키를 숨기기 위한 프록시를 만드세요.

<details>
<summary>정답</summary>

```typescript
// app/api/weather/route.ts
export async function GET() {
  const apiKey = process.env.API_KEY;

  const res = await fetch(`https://api.weather.com/data?key=${apiKey}`);
  const data = await res.json();

  return Response.json(data);
}

// Client에서 호출
fetch('/api/weather');
```
</details>

## Part 06 완료!

Part 06에서 배운 내용:
- ✅ Server Actions (폼 처리)
- ✅ 외부 API 연동

다음 Part에서는:
- 인증 기초
- NextAuth.js
- 권한 관리

[Part 07: 인증과 인가 →](../part07-authentication/chapter18-auth-basics.md)