# Chapter 11. Client Components 심화

## 11.1 Client Components란?

Client Components는 **브라우저에서 실행되는 컴포넌트**입니다. `'use client'` 지시어로 선언합니다.

### 11.1.1 Client Component 선언

```typescript
// components/Counter.tsx
'use client';  // 👈 이 한 줄!

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

**특징:**
- ✅ React Hooks 사용 가능
- ✅ 이벤트 핸들러 사용 가능
- ✅ 브라우저 API 사용 가능
- ✅ 상호작용 구현
- ❌ 서버 전용 기능 사용 불가 (DB, 파일 시스템 등)

## 11.2 `'use client'` 지시어

### 11.2.1 언제 필요한가?

```typescript
// ✅ Client Component가 필요한 경우
'use client';

// 1. React Hooks
import { useState, useEffect, useContext } from 'react';

// 2. 이벤트 핸들러
<button onClick={handleClick}>클릭</button>

// 3. 브라우저 API
window.localStorage
document.getElementById()
navigator.geolocation

// 4. 클래스 컴포넌트
class MyComponent extends React.Component {}
```

### 11.2.2 위치

```typescript
// ✅ 파일 최상단
'use client';

import { useState } from 'react';

export default function MyComponent() {
  // ...
}
```

```typescript
// ❌ 잘못된 위치
import { useState } from 'react';

'use client';  // 너무 늦음!

export default function MyComponent() {
  // ...
}
```

### 11.2.3 경계 (Boundary)

```typescript
// Parent.tsx (Server Component)
import ClientChild from './ClientChild';

export default async function Parent() {
  const data = await fetch('https://api.example.com/data')
    .then(res => res.json());

  return (
    <div>
      <h1>서버 컴포넌트</h1>
      {/* Client Component 경계 시작 */}
      <ClientChild data={data} />
    </div>
  );
}
```

```typescript
// ClientChild.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function ClientChild({ data }: { data: any }) {
  const [show, setShow] = useState(false);

  return (
    <div>
      <button onClick={() => setShow(!show)}>토글</button>
      {show && <div>{data.title}</div>}
    </div>
  );
}
```

## 11.3 useState로 상태 관리

### 11.3.1 기본 사용

```typescript
'use client';

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>카운트: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(count - 1)}>-1</button>
      <button onClick={() => setCount(0)}>리셋</button>
    </div>
  );
}
```

### 11.3.2 객체 상태

```typescript
'use client';

import { useState } from 'react';

interface User {
  name: string;
  email: string;
  age: number;
}

export default function UserForm() {
  const [user, setUser] = useState<User>({
    name: '',
    email: '',
    age: 0,
  });

  const handleChange = (field: keyof User, value: string | number) => {
    setUser(prev => ({
      ...prev,
      [field]: value,
    }));
  };

  return (
    <form>
      <input
        type="text"
        value={user.name}
        onChange={(e) => handleChange('name', e.target.value)}
        placeholder="이름"
      />
      <input
        type="email"
        value={user.email}
        onChange={(e) => handleChange('email', e.target.value)}
        placeholder="이메일"
      />
      <input
        type="number"
        value={user.age}
        onChange={(e) => handleChange('age', Number(e.target.value))}
        placeholder="나이"
      />
      <pre>{JSON.stringify(user, null, 2)}</pre>
    </form>
  );
}
```

### 11.3.3 배열 상태

```typescript
'use client';

import { useState } from 'react';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

export default function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [input, setInput] = useState('');

  const addTodo = () => {
    if (!input.trim()) return;

    setTodos(prev => [
      ...prev,
      { id: Date.now(), text: input, completed: false },
    ]);
    setInput('');
  };

  const toggleTodo = (id: number) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  };

  const deleteTodo = (id: number) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  return (
    <div>
      <div>
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && addTodo()}
          placeholder="할 일 입력"
        />
        <button onClick={addTodo}>추가</button>
      </div>

      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>삭제</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## 11.4 useEffect로 부수 효과 처리

### 11.4.1 기본 사용

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);

    // 클린업 함수
    return () => clearInterval(interval);
  }, []);

  return <div>경과 시간: {seconds}초</div>;
}
```

### 11.4.2 데이터 페칭

```typescript
'use client';

import { useState, useEffect } from 'react';

interface Post {
  id: number;
  title: string;
  body: string;
}

export default function Posts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchPosts = async () => {
      try {
        const res = await fetch('https://jsonplaceholder.typicode.com/posts');
        const data = await res.json();
        setPosts(data.slice(0, 5));
      } catch (err) {
        setError('데이터를 불러오는데 실패했습니다');
      } finally {
        setLoading(false);
      }
    };

    fetchPosts();
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

### 11.4.3 의존성 배열

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function UserPosts({ userId }: { userId: number }) {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    console.log('useEffect 실행: userId =', userId);

    fetch(`https://api.example.com/users/${userId}/posts`)
      .then(res => res.json())
      .then(setPosts);
  }, [userId]);  // userId가 변경될 때만 실행

  return (
    <ul>
      {posts.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## 11.5 이벤트 핸들러

### 11.5.1 클릭 이벤트

```typescript
'use client';

import { useState } from 'react';

export default function LikeButton() {
  const [likes, setLikes] = useState(0);
  const [liked, setLiked] = useState(false);

  const handleClick = () => {
    if (liked) {
      setLikes(prev => prev - 1);
    } else {
      setLikes(prev => prev + 1);
    }
    setLiked(!liked);
  };

  return (
    <button
      onClick={handleClick}
      style={{
        backgroundColor: liked ? 'red' : 'gray',
        color: 'white',
      }}
    >
      ❤️ {likes}
    </button>
  );
}
```

### 11.5.2 폼 이벤트

```typescript
'use client';

import { useState } from 'react';

export default function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: '',
  });
  const [submitted, setSubmitted] = useState(false);

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();

    console.log('제출된 데이터:', formData);

    // API 호출
    fetch('/api/contact', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(formData),
    });

    setSubmitted(true);
    setFormData({ name: '', email: '', message: '' });
  };

  const handleChange = (
    e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>
  ) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  };

  if (submitted) {
    return <div>문의가 제출되었습니다!</div>;
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        name="name"
        value={formData.name}
        onChange={handleChange}
        placeholder="이름"
        required
      />
      <input
        type="email"
        name="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="이메일"
        required
      />
      <textarea
        name="message"
        value={formData.message}
        onChange={handleChange}
        placeholder="메시지"
        required
        rows={4}
      />
      <button type="submit">제출</button>
    </form>
  );
}
```

### 11.5.3 키보드 이벤트

```typescript
'use client';

import { useState } from 'react';

export default function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      // 검색 실행
      console.log('검색:', query);
      setResults(['결과 1', '결과 2', '결과 3']);
    } else if (e.key === 'Escape') {
      // 초기화
      setQuery('');
      setResults([]);
    }
  };

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="검색... (Enter: 검색, Esc: 초기화)"
      />
      {results.length > 0 && (
        <ul>
          {results.map((result, i) => (
            <li key={i}>{result}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## 11.6 브라우저 API 사용

### 11.6.1 localStorage

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function ThemeSwitcher() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  // localStorage에서 테마 불러오기
  useEffect(() => {
    const saved = localStorage.getItem('theme');
    if (saved === 'dark' || saved === 'light') {
      setTheme(saved);
    }
  }, []);

  // 테마 변경 시 localStorage에 저장
  const toggleTheme = () => {
    const newTheme = theme === 'light' ? 'dark' : 'light';
    setTheme(newTheme);
    localStorage.setItem('theme', newTheme);
  };

  return (
    <button
      onClick={toggleTheme}
      style={{
        backgroundColor: theme === 'dark' ? '#333' : '#fff',
        color: theme === 'dark' ? '#fff' : '#333',
      }}
    >
      현재 테마: {theme === 'dark' ? '🌙 다크' : '☀️ 라이트'}
    </button>
  );
}
```

### 11.6.2 Geolocation

```typescript
'use client';

import { useState } from 'react';

interface Location {
  latitude: number;
  longitude: number;
}

export default function LocationFinder() {
  const [location, setLocation] = useState<Location | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const getLocation = () => {
    setLoading(true);
    setError(null);

    if (!navigator.geolocation) {
      setError('Geolocation이 지원되지 않습니다');
      setLoading(false);
      return;
    }

    navigator.geolocation.getCurrentPosition(
      (position) => {
        setLocation({
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
        });
        setLoading(false);
      },
      (err) => {
        setError(err.message);
        setLoading(false);
      }
    );
  };

  return (
    <div>
      <button onClick={getLocation} disabled={loading}>
        {loading ? '위치 확인 중...' : '내 위치 확인'}
      </button>

      {error && <div>에러: {error}</div>}

      {location && (
        <div>
          <p>위도: {location.latitude}</p>
          <p>경도: {location.longitude}</p>
        </div>
      )}
    </div>
  );
}
```

### 11.6.3 window & document

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function WindowInfo() {
  const [windowSize, setWindowSize] = useState({
    width: 0,
    height: 0,
  });
  const [scrollY, setScrollY] = useState(0);

  useEffect(() => {
    const handleResize = () => {
      setWindowSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };

    const handleScroll = () => {
      setScrollY(window.scrollY);
    };

    // 초기값 설정
    handleResize();

    window.addEventListener('resize', handleResize);
    window.addEventListener('scroll', handleScroll);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('scroll', handleScroll);
    };
  }, []);

  return (
    <div style={{ position: 'fixed', top: 10, right: 10, background: 'white', padding: 10 }}>
      <p>창 크기: {windowSize.width} x {windowSize.height}</p>
      <p>스크롤: {scrollY}px</p>
    </div>
  );
}
```

## 11.7 커스텀 훅

### 11.7.1 useLocalStorage

```typescript
// hooks/useLocalStorage.ts
'use client';

import { useState, useEffect } from 'react';

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue;

    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}
```

```typescript
// components/Counter.tsx
'use client';

import { useLocalStorage } from '@/hooks/useLocalStorage';

export default function Counter() {
  const [count, setCount] = useLocalStorage('count', 0);

  return (
    <div>
      <p>카운트: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(0)}>리셋</button>
    </div>
  );
}
```

### 11.7.2 useDebounce

```typescript
// hooks/useDebounce.ts
'use client';

import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

```typescript
// components/SearchBox.tsx
'use client';

import { useState, useEffect } from 'react';
import { useDebounce } from '@/hooks/useDebounce';

export default function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (debouncedQuery) {
      // 500ms 후에 검색 실행
      fetch(`/api/search?q=${debouncedQuery}`)
        .then(res => res.json())
        .then(setResults);
    }
  }, [debouncedQuery]);

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="검색..."
      />
      <ul>
        {results.map((result: any) => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 11.7.3 useWindowSize

```typescript
// hooks/useWindowSize.ts
'use client';

import { useState, useEffect } from 'react';

export function useWindowSize() {
  const [size, setSize] = useState({
    width: 0,
    height: 0,
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };

    handleResize();
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}
```

```typescript
// components/ResponsiveComponent.tsx
'use client';

import { useWindowSize } from '@/hooks/useWindowSize';

export default function ResponsiveComponent() {
  const { width } = useWindowSize();

  return (
    <div>
      {width < 768 ? (
        <div>모바일 뷰</div>
      ) : (
        <div>데스크톱 뷰</div>
      )}
      <p>현재 너비: {width}px</p>
    </div>
  );
}
```

## 11.8 실전 예제: 쇼핑 카트

```typescript
// components/ShoppingCart.tsx
'use client';

import { useState } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
}

interface CartItem extends Product {
  quantity: number;
}

export default function ShoppingCart() {
  const [cart, setCart] = useState<CartItem[]>([]);

  const products: Product[] = [
    { id: 1, name: '노트북', price: 1000000 },
    { id: 2, name: '마우스', price: 30000 },
    { id: 3, name: '키보드', price: 80000 },
  ];

  const addToCart = (product: Product) => {
    setCart(prev => {
      const existing = prev.find(item => item.id === product.id);

      if (existing) {
        return prev.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }

      return [...prev, { ...product, quantity: 1 }];
    });
  };

  const removeFromCart = (id: number) => {
    setCart(prev => prev.filter(item => item.id !== id));
  };

  const updateQuantity = (id: number, quantity: number) => {
    if (quantity <= 0) {
      removeFromCart(id);
      return;
    }

    setCart(prev =>
      prev.map(item =>
        item.id === id ? { ...item, quantity } : item
      )
    );
  };

  const total = cart.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  return (
    <div className="grid md:grid-cols-2 gap-8">
      {/* 상품 목록 */}
      <div>
        <h2 className="text-2xl font-bold mb-4">상품</h2>
        <div className="space-y-2">
          {products.map(product => (
            <div key={product.id} className="flex justify-between items-center border p-4">
              <div>
                <h3 className="font-bold">{product.name}</h3>
                <p className="text-gray-600">{product.price.toLocaleString()}원</p>
              </div>
              <button
                onClick={() => addToCart(product)}
                className="px-4 py-2 bg-blue-500 text-white rounded"
              >
                담기
              </button>
            </div>
          ))}
        </div>
      </div>

      {/* 장바구니 */}
      <div>
        <h2 className="text-2xl font-bold mb-4">장바구니</h2>

        {cart.length === 0 ? (
          <p className="text-gray-500">장바구니가 비어있습니다</p>
        ) : (
          <>
            <div className="space-y-2 mb-4">
              {cart.map(item => (
                <div key={item.id} className="border p-4">
                  <div className="flex justify-between items-center mb-2">
                    <h3 className="font-bold">{item.name}</h3>
                    <button
                      onClick={() => removeFromCart(item.id)}
                      className="text-red-500"
                    >
                      삭제
                    </button>
                  </div>
                  <div className="flex justify-between items-center">
                    <div className="flex items-center gap-2">
                      <button
                        onClick={() => updateQuantity(item.id, item.quantity - 1)}
                        className="px-2 py-1 bg-gray-200 rounded"
                      >
                        -
                      </button>
                      <span>{item.quantity}</span>
                      <button
                        onClick={() => updateQuantity(item.id, item.quantity + 1)}
                        className="px-2 py-1 bg-gray-200 rounded"
                      >
                        +
                      </button>
                    </div>
                    <p className="font-bold">
                      {(item.price * item.quantity).toLocaleString()}원
                    </p>
                  </div>
                </div>
              ))}
            </div>

            <div className="border-t pt-4">
              <div className="flex justify-between text-xl font-bold">
                <span>총합:</span>
                <span>{total.toLocaleString()}원</span>
              </div>
              <button className="w-full mt-4 px-4 py-2 bg-green-500 text-white rounded">
                결제하기
              </button>
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

## 핵심 요약

1. **Client Components**
   - `'use client'` 지시어 필요
   - 브라우저에서 실행
   - 상호작용 구현

2. **React Hooks**
   - `useState`: 상태 관리
   - `useEffect`: 부수 효과
   - 커스텀 훅으로 재사용

3. **이벤트 핸들러**
   - `onClick`, `onChange`, `onSubmit`
   - 폼 처리
   - 키보드 이벤트

4. **브라우저 API**
   - `localStorage`
   - `window`, `document`
   - Geolocation

5. **사용 시나리오**
   - 폼 입력
   - 상호작용 UI
   - 실시간 업데이트
   - 클라이언트 전용 기능

## 연습 문제

### 문제 1
카운터 컴포넌트를 만들고 localStorage에 저장하세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import { useState, useEffect } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const saved = localStorage.getItem('count');
    if (saved) setCount(Number(saved));
  }, []);

  useEffect(() => {
    localStorage.setItem('count', String(count));
  }, [count]);

  return (
    <div>
      <p>카운트: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(0)}>리셋</button>
    </div>
  );
}
```
</details>

### 문제 2
TODO 리스트를 만드세요 (추가, 토글, 삭제 기능).

<details>
<summary>정답</summary>

```typescript
'use client';

import { useState } from 'react';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

export default function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [input, setInput] = useState('');

  const addTodo = () => {
    if (!input.trim()) return;
    setTodos(prev => [...prev, { id: Date.now(), text: input, completed: false }]);
    setInput('');
  };

  const toggleTodo = (id: number) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  };

  const deleteTodo = (id: number) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  return (
    <div>
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && addTodo()}
      />
      <button onClick={addTodo}>추가</button>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>삭제</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- Server와 Client 컴포넌트 조합
- Props 전달 규칙
- 최적화 패턴
- 실전 예제

[Chapter 12. Server와 Client 조합하기 →](chapter12-combining-components.md)