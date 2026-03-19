# 부록 B. 상태 관리

## 상태 관리란?

애플리케이션의 **데이터를 효율적으로 관리하고 공유**하는 방법입니다.

### 상태 관리가 필요한 이유

```javascript
// ❌ Props Drilling 문제
function App() {
    const [user, setUser] = useState(null);

    return (
        <Layout user={user}>
            <Header user={user}>
                <UserMenu user={user}>
                    <UserProfile user={user} />
                </UserMenu>
            </Header>
        </Layout>
    );
}

// ✅ 상태 관리 라이브러리 사용
function App() {
    return (
        <Layout>
            <Header>
                <UserMenu>
                    <UserProfile /> {/* 직접 스토어에서 가져옴 */}
                </UserMenu>
            </Header>
        </Layout>
    );
}
```

## Redux Toolkit (RTK)

Redux의 공식 권장 방식입니다.

### 설치

```bash
npm install @reduxjs/toolkit react-redux
```

### 기본 설정

```javascript
// store.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';
import userReducer from './userSlice';

export const store = configureStore({
    reducer: {
        counter: counterReducer,
        user: userReducer
    }
});

// App.jsx
import { Provider } from 'react-redux';
import { store } from './store';

function App() {
    return (
        <Provider store={store}>
            <YourApp />
        </Provider>
    );
}
```

### Slice 생성

```javascript
// counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
    name: 'counter',
    initialState: {
        value: 0
    },
    reducers: {
        increment: (state) => {
            state.value += 1; // Immer 덕분에 직접 수정 가능
        },
        decrement: (state) => {
            state.value -= 1;
        },
        incrementByAmount: (state, action) => {
            state.value += action.payload;
        }
    }
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;
```

### 컴포넌트에서 사용

```javascript
import { useSelector, useDispatch } from 'react-redux';
import { increment, decrement, incrementByAmount } from './counterSlice';

function Counter() {
    // 상태 읽기
    const count = useSelector((state) => state.counter.value);

    // 액션 디스패치
    const dispatch = useDispatch();

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => dispatch(increment())}>+</button>
            <button onClick={() => dispatch(decrement())}>-</button>
            <button onClick={() => dispatch(incrementByAmount(5))}>+5</button>
        </div>
    );
}
```

### 비동기 작업 (Thunk)

```javascript
// userSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// 비동기 액션
export const fetchUser = createAsyncThunk(
    'user/fetchUser',
    async (userId) => {
        const response = await fetch(`/api/users/${userId}`);
        return response.json();
    }
);

const userSlice = createSlice({
    name: 'user',
    initialState: {
        data: null,
        loading: false,
        error: null
    },
    reducers: {
        clearUser: (state) => {
            state.data = null;
        }
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUser.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            .addCase(fetchUser.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload;
            })
            .addCase(fetchUser.rejected, (state, action) => {
                state.loading = false;
                state.error = action.error.message;
            });
    }
});

export const { clearUser } = userSlice.actions;
export default userSlice.reducer;

// 컴포넌트에서 사용
function UserProfile({ userId }) {
    const dispatch = useDispatch();
    const { data, loading, error } = useSelector((state) => state.user);

    useEffect(() => {
        dispatch(fetchUser(userId));
    }, [userId, dispatch]);

    if (loading) return <div>로딩 중...</div>;
    if (error) return <div>에러: {error}</div>;
    return <div>{data?.name}</div>;
}
```

## Zustand

**간단하고 가벼운** 상태 관리 라이브러리입니다.

### 설치

```bash
npm install zustand
```

### 기본 사용법

```javascript
// store.js
import { create } from 'zustand';

export const useCounterStore = create((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
    decrement: () => set((state) => ({ count: state.count - 1 })),
    reset: () => set({ count: 0 })
}));

// 컴포넌트에서 사용
function Counter() {
    const { count, increment, decrement, reset } = useCounterStore();

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={increment}>+</button>
            <button onClick={decrement}>-</button>
            <button onClick={reset}>리셋</button>
        </div>
    );
}

// 부분 선택 (최적화)
function CountDisplay() {
    const count = useCounterStore((state) => state.count);
    return <div>{count}</div>;
}
```

### 비동기 작업

```javascript
export const useUserStore = create((set) => ({
    user: null,
    loading: false,
    error: null,

    fetchUser: async (userId) => {
        set({ loading: true, error: null });

        try {
            const response = await fetch(`/api/users/${userId}`);
            const data = await response.json();
            set({ user: data, loading: false });
        } catch (error) {
            set({ error: error.message, loading: false });
        }
    },

    clearUser: () => set({ user: null })
}));

// 사용
function UserProfile() {
    const { user, loading, fetchUser } = useUserStore();

    useEffect(() => {
        fetchUser(1);
    }, [fetchUser]);

    return <div>{user?.name}</div>;
}
```

### 미들웨어

```javascript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

// LocalStorage 저장
export const useAuthStore = create(
    persist(
        (set) => ({
            user: null,
            token: null,
            login: (user, token) => set({ user, token }),
            logout: () => set({ user: null, token: null })
        }),
        {
            name: 'auth-storage' // localStorage 키
        }
    )
);
```

## Jotai

**Atomic** 상태 관리 방식입니다.

### 설치

```bash
npm install jotai
```

### 기본 사용법

```javascript
// atoms.js
import { atom } from 'jotai';

export const countAtom = atom(0);
export const userAtom = atom(null);

// 파생 atom
export const doubleCountAtom = atom((get) => get(countAtom) * 2);

// 쓰기 가능 파생 atom
export const incrementAtom = atom(
    (get) => get(countAtom),
    (get, set) => set(countAtom, get(countAtom) + 1)
);

// 컴포넌트에서 사용
import { useAtom, useAtomValue, useSetAtom } from 'jotai';
import { countAtom, doubleCountAtom, incrementAtom } from './atoms';

function Counter() {
    const [count, setCount] = useAtom(countAtom);
    const doubleCount = useAtomValue(doubleCountAtom);
    const increment = useSetAtom(incrementAtom);

    return (
        <div>
            <p>Count: {count}</p>
            <p>Double: {doubleCount}</p>
            <button onClick={() => setCount(count + 1)}>+</button>
            <button onClick={increment}>Increment</button>
        </div>
    );
}
```

### 비동기 Atom

```javascript
import { atom } from 'jotai';

// 비동기 atom
export const userAtom = atom(async () => {
    const response = await fetch('/api/user');
    return response.json();
});

// 사용
function UserProfile() {
    const [user] = useAtom(userAtom);
    // Suspense와 함께 사용
    return <div>{user.name}</div>;
}

// Suspense로 감싸기
function App() {
    return (
        <Suspense fallback={<div>로딩 중...</div>}>
            <UserProfile />
        </Suspense>
    );
}
```

## TanStack Query (React Query)

**서버 상태 관리**에 특화된 라이브러리입니다.

### 설치

```bash
npm install @tanstack/react-query
```

### 기본 설정

```javascript
// App.jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

function App() {
    return (
        <QueryClientProvider client={queryClient}>
            <YourApp />
        </QueryClientProvider>
    );
}
```

### 데이터 가져오기

```javascript
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }) {
    const { data, isLoading, error } = useQuery({
        queryKey: ['user', userId],
        queryFn: async () => {
            const response = await fetch(`/api/users/${userId}`);
            return response.json();
        }
    });

    if (isLoading) return <div>로딩 중...</div>;
    if (error) return <div>에러: {error.message}</div>;

    return <div>{data.name}</div>;
}
```

### 데이터 변경 (Mutation)

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUserForm() {
    const queryClient = useQueryClient();

    const mutation = useMutation({
        mutationFn: async (newUser) => {
            const response = await fetch('/api/users', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(newUser)
            });
            return response.json();
        },
        onSuccess: () => {
            // 캐시 무효화 (자동 새로고침)
            queryClient.invalidateQueries(['users']);
        }
    });

    const handleSubmit = (e) => {
        e.preventDefault();
        mutation.mutate({ name: 'John', email: 'john@example.com' });
    };

    return (
        <form onSubmit={handleSubmit}>
            <button type="submit" disabled={mutation.isLoading}>
                {mutation.isLoading ? '생성 중...' : '사용자 생성'}
            </button>
            {mutation.isError && <div>에러: {mutation.error.message}</div>}
        </form>
    );
}
```

### 캐싱과 자동 새로고침

```javascript
const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    staleTime: 5 * 60 * 1000,    // 5분간 fresh
    cacheTime: 10 * 60 * 1000,   // 10분간 캐시 유지
    refetchOnWindowFocus: true,  // 윈도우 포커스 시 새로고침
    refetchInterval: 30000       // 30초마다 자동 새로고침
});
```

## 상태 관리 라이브러리 비교

### Redux Toolkit

**장점:**
- 강력한 DevTools
- 미들웨어 생태계
- Time-travel 디버깅
- 예측 가능한 상태 변경

**단점:**
- 보일러플레이트 많음
- 학습 곡선 높음

**사용 시기:**
- 대규모 애플리케이션
- 복잡한 상태 로직
- 시간 여행 디버깅 필요

### Zustand

**장점:**
- 매우 간단한 API
- 보일러플레이트 거의 없음
- 작은 번들 크기
- Provider 불필요

**단점:**
- DevTools 제한적
- Redux만큼 강력하지 않음

**사용 시기:**
- 중소규모 프로젝트
- 빠른 개발 필요
- 단순한 상태 관리

### Jotai

**장점:**
- Atomic 접근 방식
- TypeScript 친화적
- Suspense 완벽 지원
- 작은 번들 크기

**단점:**
- 새로운 개념 (atom)
- 커뮤니티 작음

**사용 시기:**
- 모던 React 기능 활용
- 세밀한 상태 제어
- Suspense 사용

### TanStack Query

**장점:**
- 서버 상태 관리 특화
- 자동 캐싱
- 자동 새로고침
- 낙관적 업데이트

**단점:**
- 서버 상태에만 적합
- 클라이언트 상태는 다른 도구 필요

**사용 시기:**
- API 데이터 많음
- 실시간 데이터
- 오프라인 지원 필요

## 실전 예제: Todo 앱

### Zustand로 구현

```javascript
// store.js
import { create } from 'zustand';

export const useTodoStore = create((set) => ({
    todos: [],
    filter: 'all',

    addTodo: (text) => set((state) => ({
        todos: [...state.todos, {
            id: Date.now(),
            text,
            completed: false
        }]
    })),

    toggleTodo: (id) => set((state) => ({
        todos: state.todos.map(todo =>
            todo.id === id
                ? { ...todo, completed: !todo.completed }
                : todo
        )
    })),

    deleteTodo: (id) => set((state) => ({
        todos: state.todos.filter(todo => todo.id !== id)
    })),

    setFilter: (filter) => set({ filter }),

    getFilteredTodos: (state) => {
        const { todos, filter } = state;
        if (filter === 'active') return todos.filter(t => !t.completed);
        if (filter === 'completed') return todos.filter(t => t.completed);
        return todos;
    }
}));

// Components
function TodoApp() {
    return (
        <div>
            <TodoInput />
            <TodoList />
            <TodoFilters />
        </div>
    );
}

function TodoInput() {
    const [text, setText] = useState('');
    const addTodo = useTodoStore((state) => state.addTodo);

    const handleSubmit = (e) => {
        e.preventDefault();
        if (text.trim()) {
            addTodo(text);
            setText('');
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                value={text}
                onChange={(e) => setText(e.target.value)}
                placeholder="할 일 입력..."
            />
            <button type="submit">추가</button>
        </form>
    );
}

function TodoList() {
    const getFilteredTodos = useTodoStore((state) => state.getFilteredTodos);
    const toggleTodo = useTodoStore((state) => state.toggleTodo);
    const deleteTodo = useTodoStore((state) => state.deleteTodo);

    const todos = useTodoStore(getFilteredTodos);

    return (
        <ul>
            {todos.map(todo => (
                <li key={todo.id}>
                    <input
                        type="checkbox"
                        checked={todo.completed}
                        onChange={() => toggleTodo(todo.id)}
                    />
                    <span>{todo.text}</span>
                    <button onClick={() => deleteTodo(todo.id)}>삭제</button>
                </li>
            ))}
        </ul>
    );
}

function TodoFilters() {
    const [filter, setFilter] = useTodoStore((state) => [
        state.filter,
        state.setFilter
    ]);

    return (
        <div>
            <button onClick={() => setFilter('all')}>전체</button>
            <button onClick={() => setFilter('active')}>활성</button>
            <button onClick={() => setFilter('completed')}>완료</button>
        </div>
    );
}
```

## 핵심 요약

### 1. Redux Toolkit (복잡한 앱)
```javascript
const slice = createSlice({
    name: 'counter',
    initialState: { value: 0 },
    reducers: {
        increment: (state) => { state.value += 1 }
    }
})
```

### 2. Zustand (간단한 앱)
```javascript
const useStore = create((set) => ({
    count: 0,
    increment: () => set(state => ({ count: state.count + 1 }))
}))
```

### 3. TanStack Query (서버 상태)
```javascript
const { data } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers
})
```

### 4. 선택 가이드
- 대규모 + 복잡함 → Redux Toolkit
- 중소규모 + 단순함 → Zustand
- 서버 데이터 → TanStack Query
- 모던 + Atomic → Jotai

## 다음 단계
부록 C에서 테스팅에 대해 학습합니다.