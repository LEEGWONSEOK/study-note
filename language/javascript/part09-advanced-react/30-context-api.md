# 30. Context API

## 학습 목표
- Context API의 개념과 필요성 이해하기
- Context 생성과 사용 방법 학습하기
- useContext Hook 마스터하기
- Context 최적화 패턴 익히기

## Context API란?

Context는 **컴포넌트 트리 전체에 데이터를 공유**하는 방법입니다.

### 비유로 이해하기
Context는 **방송국**과 같습니다:
- 방송국(Provider)이 신호를 송출
- 모든 수신기(Consumer)가 같은 신호를 받음
- 중간 중계기를 거치지 않아도 됨
- 채널(Context)별로 다른 내용 전송

### Props Drilling 문제

```javascript
// ❌ Props Drilling: 여러 레벨을 거쳐 props 전달
function App() {
    const [user, setUser] = useState({ name: '김철수' });

    return <Page user={user} />;
}

function Page({ user }) {
    return <Header user={user} />;
}

function Header({ user }) {
    return <UserMenu user={user} />;
}

function UserMenu({ user }) {
    return <UserProfile user={user} />;
}

function UserProfile({ user }) {
    return <div>{user.name}</div>;
}

// 중간 컴포넌트들은 user를 사용하지 않지만
// 전달만을 위해 props를 받아야 함!
```

### Context로 해결

```javascript
// ✅ Context: 직접 접근
import { createContext, useContext, useState } from 'react';

// 1. Context 생성
const UserContext = createContext(null);

// 2. Provider로 값 제공
function App() {
    const [user, setUser] = useState({ name: '김철수' });

    return (
        <UserContext.Provider value={user}>
            <Page />
        </UserContext.Provider>
    );
}

// 3. 중간 컴포넌트는 props 없이
function Page() {
    return <Header />;
}

function Header() {
    return <UserMenu />;
}

function UserMenu() {
    return <UserProfile />;
}

// 4. 필요한 곳에서 직접 사용
function UserProfile() {
    const user = useContext(UserContext);
    return <div>{user.name}</div>;
}
```

### Java와 비교

```java
// Java: 전역 변수나 싱글톤 패턴
public class UserManager {
    private static UserManager instance;
    private User currentUser;

    public static UserManager getInstance() {
        if (instance == null) {
            instance = new UserManager();
        }
        return instance;
    }

    public User getCurrentUser() {
        return currentUser;
    }
}

// 어디서든 접근
User user = UserManager.getInstance().getCurrentUser();
```

```javascript
// React: Context API
const UserContext = createContext();

// Provider로 제공
<UserContext.Provider value={user}>
    <App />
</UserContext.Provider>

// 어디서든 접근
const user = useContext(UserContext);

// 차이점:
// Java: 전역 상태 (테스트 어려움)
// React: 컴포넌트 트리 범위 (더 유연함)
```

## Context 생성과 사용

### 기본 패턴

```javascript
// 1. Context 생성
const ThemeContext = createContext('light'); // 기본값

// 2. Provider 컴포넌트
function App() {
    const [theme, setTheme] = useState('light');

    return (
        <ThemeContext.Provider value={theme}>
            <Toolbar />
        </ThemeContext.Provider>
    );
}

// 3. Consumer 컴포넌트
function ThemedButton() {
    const theme = useContext(ThemeContext);

    return (
        <button className={`btn-${theme}`}>
            버튼
        </button>
    );
}
```

### 여러 값 제공하기

```javascript
const AuthContext = createContext(null);

function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    const login = async (email, password) => {
        setLoading(true);
        try {
            const response = await fetch('/api/login', {
                method: 'POST',
                body: JSON.stringify({ email, password })
            });
            const data = await response.json();
            setUser(data.user);
        } catch (error) {
            console.error(error);
        } finally {
            setLoading(false);
        }
    };

    const logout = () => {
        setUser(null);
    };

    // 객체로 여러 값 제공
    const value = {
        user,
        loading,
        login,
        logout
    };

    return (
        <AuthContext.Provider value={value}>
            {children}
        </AuthContext.Provider>
    );
}

// 사용
function LoginButton() {
    const { user, login, logout } = useContext(AuthContext);

    return user ? (
        <button onClick={logout}>로그아웃</button>
    ) : (
        <button onClick={() => login('email@example.com', 'password')}>
            로그인
        </button>
    );
}
```

### 커스텀 Hook으로 캡슐화

```javascript
// Provider와 Hook을 함께 제공
const ThemeContext = createContext(null);

export function ThemeProvider({ children }) {
    const [theme, setTheme] = useState('light');

    const toggleTheme = () => {
        setTheme(prev => prev === 'light' ? 'dark' : 'light');
    };

    const value = {
        theme,
        toggleTheme
    };

    return (
        <ThemeContext.Provider value={value}>
            {children}
        </ThemeContext.Provider>
    );
}

// 커스텀 Hook
export function useTheme() {
    const context = useContext(ThemeContext);

    if (context === null) {
        throw new Error('useTheme must be used within ThemeProvider');
    }

    return context;
}

// 사용 - 더 간단하고 안전
function ThemedButton() {
    const { theme, toggleTheme } = useTheme();

    return (
        <button onClick={toggleTheme}>
            현재 테마: {theme}
        </button>
    );
}
```

## 실전 예제

### 1. 인증 시스템

```javascript
// AuthContext.jsx
import { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    // 초기화: 저장된 토큰으로 사용자 정보 로드
    useEffect(() => {
        const token = localStorage.getItem('token');
        if (token) {
            fetchUser(token);
        } else {
            setLoading(false);
        }
    }, []);

    const fetchUser = async (token) => {
        try {
            const response = await fetch('/api/me', {
                headers: { Authorization: `Bearer ${token}` }
            });
            const data = await response.json();
            setUser(data);
        } catch (error) {
            localStorage.removeItem('token');
        } finally {
            setLoading(false);
        }
    };

    const login = async (email, password) => {
        const response = await fetch('/api/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password })
        });

        if (!response.ok) {
            throw new Error('로그인 실패');
        }

        const data = await response.json();
        localStorage.setItem('token', data.token);
        setUser(data.user);
    };

    const logout = () => {
        localStorage.removeItem('token');
        setUser(null);
    };

    const value = {
        user,
        loading,
        login,
        logout,
        isAuthenticated: !!user
    };

    return (
        <AuthContext.Provider value={value}>
            {children}
        </AuthContext.Provider>
    );
}

export function useAuth() {
    const context = useContext(AuthContext);
    if (!context) {
        throw new Error('useAuth must be used within AuthProvider');
    }
    return context;
}

// 사용
function App() {
    return (
        <AuthProvider>
            <Router>
                <Routes>
                    <Route path="/login" element={<LoginPage />} />
                    <Route path="/dashboard" element={
                        <ProtectedRoute>
                            <Dashboard />
                        </ProtectedRoute>
                    } />
                </Routes>
            </Router>
        </AuthProvider>
    );
}

function ProtectedRoute({ children }) {
    const { isAuthenticated, loading } = useAuth();

    if (loading) {
        return <div>로딩 중...</div>;
    }

    if (!isAuthenticated) {
        return <Navigate to="/login" />;
    }

    return children;
}

function LoginPage() {
    const { login } = useAuth();
    const [email, setEmail] = useState('');
    const [password, setPassword] = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        try {
            await login(email, password);
        } catch (error) {
            alert('로그인 실패');
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
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

### 2. 테마 시스템

```javascript
// ThemeContext.jsx
const ThemeContext = createContext(null);

const themes = {
    light: {
        background: '#ffffff',
        text: '#000000',
        primary: '#007bff',
        secondary: '#6c757d'
    },
    dark: {
        background: '#1a1a1a',
        text: '#ffffff',
        primary: '#0d6efd',
        secondary: '#6c757d'
    }
};

export function ThemeProvider({ children }) {
    const [themeName, setThemeName] = useState(() => {
        return localStorage.getItem('theme') || 'light';
    });

    const theme = themes[themeName];

    useEffect(() => {
        localStorage.setItem('theme', themeName);
        document.body.style.backgroundColor = theme.background;
        document.body.style.color = theme.text;
    }, [themeName, theme]);

    const toggleTheme = () => {
        setThemeName(prev => prev === 'light' ? 'dark' : 'light');
    };

    const value = {
        theme,
        themeName,
        toggleTheme
    };

    return (
        <ThemeContext.Provider value={value}>
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

// 사용
function App() {
    return (
        <ThemeProvider>
            <Header />
            <Main />
        </ThemeProvider>
    );
}

function Header() {
    const { theme, themeName, toggleTheme } = useTheme();

    return (
        <header style={{ backgroundColor: theme.primary, color: theme.text }}>
            <h1>My App</h1>
            <button onClick={toggleTheme}>
                {themeName === 'light' ? '🌙' : '☀️'}
            </button>
        </header>
    );
}

function Button({ children, ...props }) {
    const { theme } = useTheme();

    return (
        <button
            {...props}
            style={{
                backgroundColor: theme.primary,
                color: theme.text,
                ...props.style
            }}
        >
            {children}
        </button>
    );
}
```

### 3. 다국어 지원 (i18n)

```javascript
// LanguageContext.jsx
const LanguageContext = createContext(null);

const translations = {
    ko: {
        welcome: '환영합니다',
        login: '로그인',
        logout: '로그아웃',
        settings: '설정'
    },
    en: {
        welcome: 'Welcome',
        login: 'Login',
        logout: 'Logout',
        settings: 'Settings'
    },
    ja: {
        welcome: 'ようこそ',
        login: 'ログイン',
        logout: 'ログアウト',
        settings: '設定'
    }
};

export function LanguageProvider({ children }) {
    const [language, setLanguage] = useState(() => {
        return localStorage.getItem('language') || 'ko';
    });

    useEffect(() => {
        localStorage.setItem('language', language);
    }, [language]);

    const t = (key) => {
        return translations[language][key] || key;
    };

    const value = {
        language,
        setLanguage,
        t
    };

    return (
        <LanguageContext.Provider value={value}>
            {children}
        </LanguageContext.Provider>
    );
}

export function useLanguage() {
    const context = useContext(LanguageContext);
    if (!context) {
        throw new Error('useLanguage must be used within LanguageProvider');
    }
    return context;
}

// 사용
function Header() {
    const { t, language, setLanguage } = useLanguage();

    return (
        <header>
            <h1>{t('welcome')}</h1>
            <select
                value={language}
                onChange={(e) => setLanguage(e.target.value)}
            >
                <option value="ko">한국어</option>
                <option value="en">English</option>
                <option value="ja">日本語</option>
            </select>
        </header>
    );
}

function LoginButton() {
    const { t } = useLanguage();
    const { user, logout } = useAuth();

    return user ? (
        <button onClick={logout}>{t('logout')}</button>
    ) : (
        <button>{t('login')}</button>
    );
}
```

### 4. 장바구니 시스템

```javascript
// CartContext.jsx
const CartContext = createContext(null);

export function CartProvider({ children }) {
    const [items, setItems] = useState(() => {
        const saved = localStorage.getItem('cart');
        return saved ? JSON.parse(saved) : [];
    });

    useEffect(() => {
        localStorage.setItem('cart', JSON.stringify(items));
    }, [items]);

    const addItem = (product) => {
        setItems(prev => {
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

    const removeItem = (productId) => {
        setItems(prev => prev.filter(item => item.id !== productId));
    };

    const updateQuantity = (productId, quantity) => {
        if (quantity <= 0) {
            removeItem(productId);
            return;
        }

        setItems(prev =>
            prev.map(item =>
                item.id === productId
                    ? { ...item, quantity }
                    : item
            )
        );
    };

    const clearCart = () => {
        setItems([]);
    };

    const total = items.reduce(
        (sum, item) => sum + item.price * item.quantity,
        0
    );

    const itemCount = items.reduce((sum, item) => sum + item.quantity, 0);

    const value = {
        items,
        addItem,
        removeItem,
        updateQuantity,
        clearCart,
        total,
        itemCount
    };

    return (
        <CartContext.Provider value={value}>
            {children}
        </CartContext.Provider>
    );
}

export function useCart() {
    const context = useContext(CartContext);
    if (!context) {
        throw new Error('useCart must be used within CartProvider');
    }
    return context;
}

// 사용
function ProductCard({ product }) {
    const { addItem } = useCart();

    return (
        <div className="product-card">
            <img src={product.image} alt={product.name} />
            <h3>{product.name}</h3>
            <p>{product.price.toLocaleString()}원</p>
            <button onClick={() => addItem(product)}>
                장바구니 담기
            </button>
        </div>
    );
}

function CartIcon() {
    const { itemCount } = useCart();

    return (
        <div className="cart-icon">
            🛒
            {itemCount > 0 && (
                <span className="badge">{itemCount}</span>
            )}
        </div>
    );
}

function CartPage() {
    const { items, updateQuantity, removeItem, total, clearCart } = useCart();

    if (items.length === 0) {
        return <div>장바구니가 비어있습니다</div>;
    }

    return (
        <div className="cart-page">
            <h2>장바구니</h2>

            {items.map(item => (
                <div key={item.id} className="cart-item">
                    <img src={item.image} alt={item.name} />
                    <div>
                        <h3>{item.name}</h3>
                        <p>{item.price.toLocaleString()}원</p>
                    </div>
                    <input
                        type="number"
                        value={item.quantity}
                        onChange={(e) =>
                            updateQuantity(item.id, Number(e.target.value))
                        }
                        min="0"
                    />
                    <p>{(item.price * item.quantity).toLocaleString()}원</p>
                    <button onClick={() => removeItem(item.id)}>
                        삭제
                    </button>
                </div>
            ))}

            <div className="cart-total">
                <h3>총액: {total.toLocaleString()}원</h3>
                <button onClick={clearCart}>전체 삭제</button>
                <button>결제하기</button>
            </div>
        </div>
    );
}
```

## Context 최적화

### 1. Context 분리

```javascript
// ❌ 하나의 큰 Context (비효율적)
const AppContext = createContext();

function AppProvider({ children }) {
    const [user, setUser] = useState(null);
    const [theme, setTheme] = useState('light');
    const [language, setLanguage] = useState('ko');

    // user만 변경되어도 모든 consumer가 리렌더링됨!

    return (
        <AppContext.Provider value={{ user, theme, language }}>
            {children}
        </AppContext.Provider>
    );
}

// ✅ Context 분리 (효율적)
const UserContext = createContext();
const ThemeContext = createContext();
const LanguageContext = createContext();

function App() {
    return (
        <UserProvider>
            <ThemeProvider>
                <LanguageProvider>
                    <Main />
                </LanguageProvider>
            </ThemeProvider>
        </UserProvider>
    );
}
```

### 2. 메모이제이션

```javascript
function AuthProvider({ children }) {
    const [user, setUser] = useState(null);

    // ❌ 매 렌더링마다 새 객체 생성
    const value = {
        user,
        login: (data) => setUser(data),
        logout: () => setUser(null)
    };

    // ✅ 함수 메모이제이션
    const login = useCallback((data) => {
        setUser(data);
    }, []);

    const logout = useCallback(() => {
        setUser(null);
    }, []);

    // ✅ 값 메모이제이션
    const memoizedValue = useMemo(() => ({
        user,
        login,
        logout
    }), [user, login, logout]);

    return (
        <AuthContext.Provider value={memoizedValue}>
            {children}
        </AuthContext.Provider>
    );
}
```

### 3. Consumer 최적화

```javascript
// React.memo로 불필요한 리렌더링 방지
const ExpensiveComponent = React.memo(function ExpensiveComponent() {
    const { theme } = useTheme();

    console.log('렌더링됨');

    return <div>테마: {theme}</div>;
});

// 필요한 값만 구조 분해
function Component() {
    // ❌ 전체 context 객체 사용
    const context = useAuth();

    // ✅ 필요한 값만 추출
    const { user } = useAuth();

    return <div>{user.name}</div>;
}
```

## 여러 Provider 조합

```javascript
// 모든 Provider를 조합하는 컴포넌트
function AppProviders({ children }) {
    return (
        <AuthProvider>
            <ThemeProvider>
                <LanguageProvider>
                    <CartProvider>
                        {children}
                    </CartProvider>
                </LanguageProvider>
            </ThemeProvider>
        </AuthProvider>
    );
}

// 깔끔한 사용
function App() {
    return (
        <AppProviders>
            <Router>
                <Routes>
                    <Route path="/" element={<Home />} />
                    <Route path="/cart" element={<Cart />} />
                </Routes>
            </Router>
        </AppProviders>
    );
}
```

## 핵심 요약

### 1. Context는 전역 상태 공유
- Props Drilling 해결
- 컴포넌트 트리 전체에서 접근
- Provider로 값 제공
- useContext로 값 소비

### 2. 패턴
```javascript
// 1. Context 생성
const MyContext = createContext(defaultValue);

// 2. Provider 제공
<MyContext.Provider value={value}>
    {children}
</MyContext.Provider>

// 3. Hook으로 소비
const value = useContext(MyContext);
```

### 3. 최적화
- Context 분리
- 값 메모이제이션
- React.memo 사용
- 필요한 값만 추출

### 4. 실전 활용
- 인증 시스템
- 테마 관리
- 다국어 지원
- 장바구니

## 연습 문제

### 문제 1: 알림 시스템
토스트 알림을 관리하는 Context를 만들어보세요.

```javascript
// 요구사항:
// - 알림 추가/제거
// - 자동 사라짐 (3초)
// - 여러 알림 동시 표시
// - 알림 타입 (success, error, info)

function NotificationProvider({ children }) {
    // 여기에 구현
}

// 사용 예시
function Component() {
    const { addNotification } = useNotification();

    return (
        <button onClick={() =>
            addNotification('성공!', 'success')
        }>
            알림 표시
        </button>
    );
}
```

### 문제 2: 모달 관리
여러 모달을 관리하는 Context를 만들어보세요.

```javascript
// 요구사항:
// - 모달 열기/닫기
// - 여러 모달 동시 관리
// - 모달 데이터 전달
// - ESC 키로 닫기

function ModalProvider({ children }) {
    // 여기에 구현
}

// 사용 예시
function Component() {
    const { openModal, closeModal } = useModal();

    return (
        <button onClick={() =>
            openModal('confirm', { title: '확인', message: '정말?' })
        }>
            모달 열기
        </button>
    );
}
```

### 문제 3: 폼 상태 관리
복잡한 폼의 상태를 관리하는 Context를 만들어보세요.

```javascript
// 요구사항:
// - 여러 스텝의 폼
// - 각 스텝의 유효성 검사
// - 이전/다음 버튼
// - 진행 상태 표시

function FormProvider({ children }) {
    // 여기에 구현
}

// 사용 예시
function MultiStepForm() {
    const { currentStep, nextStep, prevStep, formData } = useForm();

    return (
        <div>
            <ProgressBar step={currentStep} />
            {currentStep === 1 && <Step1 />}
            {currentStep === 2 && <Step2 />}
        </div>
    );
}
```

### 문제 4: 실행 취소/다시 실행
작업의 히스토리를 관리하는 Context를 만들어보세요.

```javascript
// 요구사항:
// - 작업 실행 시 히스토리에 추가
// - 실행 취소 (Ctrl+Z)
// - 다시 실행 (Ctrl+Shift+Z)
// - 히스토리 제한 (최대 50개)

function HistoryProvider({ children }) {
    // 여기에 구현
}

// 사용 예시
function Editor() {
    const { state, setState, undo, redo, canUndo, canRedo } = useHistory();

    return (
        <div>
            <button onClick={undo} disabled={!canUndo}>
                실행 취소
            </button>
            <button onClick={redo} disabled={!canRedo}>
                다시 실행
            </button>
        </div>
    );
}
```

## 다음 단계
다음 장에서는 **성능 최적화**를 학습합니다:
- React.memo와 useMemo
- 코드 분할과 지연 로딩
- 가상화와 윈도잉
- 프로파일링과 디버깅