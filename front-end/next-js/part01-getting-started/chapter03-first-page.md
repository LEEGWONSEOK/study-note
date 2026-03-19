# Chapter 3. 첫 페이지 만들기

## 3.1 페이지 생성 기초

Next.js에서 페이지를 만드는 것은 매우 간단합니다. `app` 폴더 안에 폴더를 만들고 `page.tsx` 파일을 생성하면 됩니다.

### 3.1.1 가장 간단한 페이지

```typescript
// app/page.tsx
export default function Home() {
  return <h1>안녕하세요!</h1>;
}
```

이것만으로 `/` 경로의 페이지가 완성됩니다!

### 3.1.2 새 페이지 만들기

```bash
# 폴더 생성
mkdir app/about

# 페이지 파일 생성
touch app/about/page.tsx
```

```typescript
// app/about/page.tsx
export default function AboutPage() {
  return (
    <div>
      <h1>회사 소개</h1>
      <p>우리는 최고의 서비스를 제공합니다.</p>
    </div>
  );
}
```

브라우저에서 `http://localhost:3000/about` 접속하면 페이지가 보입니다!

## 3.2 JSX 문법 복습

JSX는 JavaScript 안에서 HTML처럼 보이는 코드를 작성할 수 있게 해주는 문법입니다.

### 3.2.1 기본 규칙

```typescript
export default function MyPage() {
  return (
    // 1. 반드시 하나의 최상위 요소로 감싸기
    <div>
      <h1>제목</h1>
      <p>내용</p>
    </div>
  );
}
```

**여러 요소를 반환할 때:**

```typescript
// ❌ 잘못된 예시 (최상위 요소가 2개)
export default function MyPage() {
  return (
    <h1>제목</h1>
    <p>내용</p>
  );
}

// ✅ Fragment 사용 (DOM 요소 추가 안 됨)
export default function MyPage() {
  return (
    <>
      <h1>제목</h1>
      <p>내용</p>
    </>
  );
}

// ✅ div로 감싸기
export default function MyPage() {
  return (
    <div>
      <h1>제목</h1>
      <p>내용</p>
    </div>
  );
}
```

### 3.2.2 JavaScript 표현식 사용

중괄호 `{}`로 JavaScript 표현식을 삽입할 수 있습니다.

```typescript
export default function ProfilePage() {
  const name = "홍길동";
  const age = 30;
  const isLoggedIn = true;

  return (
    <div>
      <h1>프로필</h1>
      <p>이름: {name}</p>
      <p>나이: {age}세</p>
      <p>계산: {10 + 20}</p>
      <p>상태: {isLoggedIn ? "로그인됨" : "로그아웃됨"}</p>
    </div>
  );
}
```

### 3.2.3 속성(Props) 사용

HTML 속성은 camelCase로 작성합니다.

```typescript
export default function StyledPage() {
  const imageUrl = "/images/logo.png";

  return (
    <div>
      {/* class → className */}
      <div className="container">

        {/* for → htmlFor */}
        <label htmlFor="email">이메일</label>
        <input id="email" type="text" />

        {/* style은 객체로 */}
        <div style={{
          color: 'blue',
          fontSize: '20px',
          marginTop: '10px'
        }}>
          스타일 적용
        </div>

        {/* 이미지 */}
        <img src={imageUrl} alt="로고" />
      </div>
    </div>
  );
}
```

### 3.2.4 조건부 렌더링

```typescript
export default function ConditionalPage() {
  const isLoggedIn = false;
  const userRole = "admin";

  return (
    <div>
      {/* 조건부 렌더링 - 삼항 연산자 */}
      {isLoggedIn ? (
        <p>환영합니다!</p>
      ) : (
        <p>로그인해주세요.</p>
      )}

      {/* 조건부 렌더링 - && 연산자 */}
      {isLoggedIn && <button>로그아웃</button>}

      {/* 다중 조건 */}
      {userRole === "admin" && (
        <div>
          <h2>관리자 패널</h2>
          <button>사용자 관리</button>
        </div>
      )}
    </div>
  );
}
```

### 3.2.5 리스트 렌더링

```typescript
export default function ListPage() {
  const fruits = ["사과", "바나나", "오렌지"];

  const users = [
    { id: 1, name: "홍길동", age: 30 },
    { id: 2, name: "김철수", age: 25 },
    { id: 3, name: "이영희", age: 28 },
  ];

  return (
    <div>
      {/* 간단한 리스트 */}
      <ul>
        {fruits.map((fruit, index) => (
          <li key={index}>{fruit}</li>
        ))}
      </ul>

      {/* 객체 배열 */}
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            {user.name} ({user.age}세)
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**⚠️ key prop 주의사항:**
- 리스트 렌더링 시 각 항목에 고유한 `key` 필수
- 가능하면 `id` 같은 고유값 사용
- `index`는 최후의 수단 (항목 순서가 변경될 수 있으면 문제)

## 3.3 컴포넌트 작성

컴포넌트는 재사용 가능한 UI 조각입니다.

### 3.3.1 컴포넌트 만들기

```typescript
// components/ui/Button.tsx
export default function Button({ children }: { children: React.ReactNode }) {
  return (
    <button className="px-4 py-2 bg-blue-500 text-white rounded">
      {children}
    </button>
  );
}
```

```typescript
// app/page.tsx
import Button from '@/components/ui/Button';

export default function Home() {
  return (
    <div>
      <h1>홈페이지</h1>
      <Button>클릭하세요</Button>
      <Button>제출</Button>
    </div>
  );
}
```

### 3.3.2 Props 전달

```typescript
// components/ui/Card.tsx
interface CardProps {
  title: string;
  description: string;
  imageUrl?: string;  // 선택적 prop
}

export default function Card({ title, description, imageUrl }: CardProps) {
  return (
    <div className="border rounded-lg p-4">
      {imageUrl && <img src={imageUrl} alt={title} />}
      <h3 className="text-xl font-bold">{title}</h3>
      <p>{description}</p>
    </div>
  );
}
```

```typescript
// app/page.tsx
import Card from '@/components/ui/Card';

export default function Home() {
  return (
    <div>
      <Card
        title="상품 1"
        description="멋진 상품입니다"
        imageUrl="/images/product1.jpg"
      />

      <Card
        title="상품 2"
        description="또 다른 멋진 상품"
      />
    </div>
  );
}
```

### 3.3.3 이벤트 핸들러

```typescript
// components/ui/InteractiveButton.tsx
'use client';  // 이벤트 핸들러는 클라이언트 컴포넌트!

export default function InteractiveButton() {
  const handleClick = () => {
    alert('버튼 클릭!');
  };

  return (
    <button onClick={handleClick}>
      클릭하세요
    </button>
  );
}
```

```typescript
// Props로 이벤트 핸들러 전달
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  disabled?: boolean;
}

export default function Button({ children, onClick, disabled }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className="px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-50"
    >
      {children}
    </button>
  );
}
```

### 3.3.4 컴포넌트 합성

```typescript
// components/layout/Header.tsx
import Link from 'next/link';

export default function Header() {
  return (
    <header className="bg-gray-800 text-white p-4">
      <nav className="flex gap-4">
        <Link href="/">홈</Link>
        <Link href="/about">소개</Link>
        <Link href="/products">상품</Link>
        <Link href="/contact">연락처</Link>
      </nav>
    </header>
  );
}
```

```typescript
// components/layout/Footer.tsx
export default function Footer() {
  return (
    <footer className="bg-gray-800 text-white p-4 text-center">
      <p>&copy; 2024 My Company. All rights reserved.</p>
    </footer>
  );
}
```

```typescript
// app/layout.tsx
import Header from '@/components/layout/Header';
import Footer from '@/components/layout/Footer';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>
        <Header />
        <main className="min-h-screen">{children}</main>
        <Footer />
      </body>
    </html>
  );
}
```

## 3.4 레이아웃 (Layout)

레이아웃은 여러 페이지에서 공통으로 사용되는 UI입니다.

### 3.4.1 루트 레이아웃 (필수)

```typescript
// app/layout.tsx
import './globals.css';
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'My Next.js App',
  description: 'Created with Next.js',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body className={inter.className}>
        <div className="flex flex-col min-h-screen">
          <header className="bg-blue-500 text-white p-4">
            <h1>내 웹사이트</h1>
          </header>

          <main className="flex-1 container mx-auto p-4">
            {children}
          </main>

          <footer className="bg-gray-800 text-white p-4 text-center">
            <p>Footer</p>
          </footer>
        </div>
      </body>
    </html>
  );
}
```

### 3.4.2 중첩 레이아웃

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      {/* 사이드바 */}
      <aside className="w-64 bg-gray-100 p-4">
        <nav>
          <ul>
            <li><a href="/dashboard">대시보드</a></li>
            <li><a href="/dashboard/analytics">분석</a></li>
            <li><a href="/dashboard/settings">설정</a></li>
          </ul>
        </nav>
      </aside>

      {/* 메인 콘텐츠 */}
      <div className="flex-1 p-4">
        {children}
      </div>
    </div>
  );
}
```

```typescript
// app/dashboard/page.tsx
export default function DashboardPage() {
  return (
    <div>
      <h1>대시보드</h1>
      <p>대시보드 내용...</p>
    </div>
  );
}
```

**렌더링 결과:**
```
RootLayout (헤더, 푸터)
  └─ DashboardLayout (사이드바)
       └─ DashboardPage (콘텐츠)
```

## 3.5 메타데이터 (SEO)

메타데이터는 검색 엔진 최적화(SEO)에 중요합니다.

### 3.5.1 정적 메타데이터

```typescript
// app/about/page.tsx
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: '회사 소개',
  description: '우리 회사는 최고의 서비스를 제공합니다',
  keywords: ['회사', '소개', 'Next.js'],

  // Open Graph (소셜 미디어 공유)
  openGraph: {
    title: '회사 소개',
    description: '우리 회사는 최고의 서비스를 제공합니다',
    images: ['/og-image.jpg'],
    url: 'https://example.com/about',
    type: 'website',
  },

  // Twitter Card
  twitter: {
    card: 'summary_large_image',
    title: '회사 소개',
    description: '우리 회사는 최고의 서비스를 제공합니다',
    images: ['/twitter-image.jpg'],
  },
};

export default function AboutPage() {
  return (
    <div>
      <h1>회사 소개</h1>
      <p>내용...</p>
    </div>
  );
}
```

### 3.5.2 동적 메타데이터

```typescript
// app/products/[id]/page.tsx
import { Metadata } from 'next';

interface Props {
  params: { id: string };
}

// 동적으로 메타데이터 생성
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      description: product.description,
      images: [product.imageUrl],
    },
  };
}

export default async function ProductPage({ params }: Props) {
  const product = await fetch(`https://api.example.com/products/${params.id}`)
    .then(res => res.json());

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p>가격: {product.price}원</p>
    </div>
  );
}
```

### 3.5.3 메타데이터 병합

```typescript
// app/layout.tsx (루트)
export const metadata = {
  title: {
    default: 'My Website',
    template: '%s | My Website',  // %s는 자식 페이지의 title로 대체됨
  },
  description: 'Default description',
};

// app/about/page.tsx
export const metadata = {
  title: 'About Us',  // 결과: "About Us | My Website"
};

// app/products/page.tsx
export const metadata = {
  title: 'Products',  // 결과: "Products | My Website"
};
```

## 3.6 실전 예제: 간단한 블로그 홈페이지

```typescript
// app/page.tsx
import { Metadata } from 'next';
import Link from 'next/link';

export const metadata: Metadata = {
  title: '블로그 홈',
  description: '개발 블로그입니다',
};

// 임시 데이터
const posts = [
  {
    id: 1,
    title: 'Next.js 시작하기',
    excerpt: 'Next.js는 React 프레임워크입니다.',
    date: '2024-01-15',
  },
  {
    id: 2,
    title: 'TypeScript 배우기',
    excerpt: 'TypeScript는 타입 안정성을 제공합니다.',
    date: '2024-01-20',
  },
  {
    id: 3,
    title: 'Tailwind CSS 활용',
    excerpt: 'Tailwind CSS로 빠르게 스타일링하기',
    date: '2024-01-25',
  },
];

export default function HomePage() {
  return (
    <div className="max-w-4xl mx-auto">
      <h1 className="text-4xl font-bold mb-8">최근 포스트</h1>

      <div className="grid gap-6">
        {posts.map((post) => (
          <article key={post.id} className="border rounded-lg p-6 hover:shadow-lg transition">
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

## 핵심 요약

1. **페이지 생성은 간단**
   - `app` 폴더에 `page.tsx` 파일 생성
   - 폴더 구조 = URL 구조

2. **JSX 기본 규칙**
   - 하나의 최상위 요소로 감싸기
   - `{}`로 JavaScript 표현식 사용
   - `className`, `htmlFor` 사용

3. **컴포넌트 재사용**
   - Props로 데이터 전달
   - TypeScript 인터페이스로 타입 정의

4. **레이아웃 활용**
   - 공통 UI는 `layout.tsx`에
   - 중첩 레이아웃 가능

5. **메타데이터 설정**
   - SEO 최적화를 위해 필수
   - 정적/동적 메타데이터 모두 가능

## 연습 문제

### 문제 1
다음 데이터를 카드 형태로 렌더링하는 페이지를 만드세요.

```typescript
const products = [
  { id: 1, name: "노트북", price: 1000000 },
  { id: 2, name: "마우스", price: 30000 },
  { id: 3, name: "키보드", price: 80000 },
];
```

<details>
<summary>정답</summary>

```typescript
// components/ProductCard.tsx
interface ProductCardProps {
  name: string;
  price: number;
}

export default function ProductCard({ name, price }: ProductCardProps) {
  return (
    <div className="border rounded-lg p-4">
      <h3 className="text-lg font-bold">{name}</h3>
      <p className="text-gray-600">{price.toLocaleString()}원</p>
    </div>
  );
}

// app/products/page.tsx
import ProductCard from '@/components/ProductCard';

const products = [
  { id: 1, name: "노트북", price: 1000000 },
  { id: 2, name: "마우스", price: 30000 },
  { id: 3, name: "키보드", price: 80000 },
];

export default function ProductsPage() {
  return (
    <div>
      <h1 className="text-2xl font-bold mb-4">상품 목록</h1>
      <div className="grid grid-cols-3 gap-4">
        {products.map((product) => (
          <ProductCard
            key={product.id}
            name={product.name}
            price={product.price}
          />
        ))}
      </div>
    </div>
  );
}
```
</details>

### 문제 2
헤더와 푸터가 있는 레이아웃을 만들고, 홈페이지와 소개 페이지를 생성하세요.

<details>
<summary>정답</summary>

```typescript
// app/layout.tsx
import Link from 'next/link';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>
        <header className="bg-blue-500 text-white p-4">
          <nav className="flex gap-4">
            <Link href="/">홈</Link>
            <Link href="/about">소개</Link>
          </nav>
        </header>

        <main className="container mx-auto p-4 min-h-screen">
          {children}
        </main>

        <footer className="bg-gray-800 text-white p-4 text-center">
          <p>&copy; 2024 My Company</p>
        </footer>
      </body>
    </html>
  );
}

// app/page.tsx
export default function HomePage() {
  return <h1>홈페이지입니다</h1>;
}

// app/about/page.tsx
export default function AboutPage() {
  return (
    <div>
      <h1>회사 소개</h1>
      <p>우리는 최고의 회사입니다.</p>
    </div>
  );
}
```
</details>

### 문제 3
동적 메타데이터를 사용하여 사용자 프로필 페이지를 만드세요.
- URL: `/users/[id]`
- 메타데이터에 사용자 이름 포함

<details>
<summary>정답</summary>

```typescript
// app/users/[id]/page.tsx
import { Metadata } from 'next';

interface Props {
  params: { id: string };
}

// 임시 데이터
const users = {
  '1': { name: '홍길동', bio: '개발자입니다' },
  '2': { name: '김철수', bio: '디자이너입니다' },
};

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const user = users[params.id as keyof typeof users];

  return {
    title: `${user.name}의 프로필`,
    description: user.bio,
  };
}

export default function UserPage({ params }: Props) {
  const user = users[params.id as keyof typeof users];

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.bio}</p>
    </div>
  );
}
```
</details>

## 다음 단계

Part 01 완료! 다음 Part에서는:
- 파일 기반 라우팅 심화
- 동적 라우트
- 네비게이션
- 중첩 라우트

[Part 02: 라우팅 (Routing) →](../part02-routing/chapter04-file-based-routing.md)
