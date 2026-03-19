# Chapter 13. CSS Modules

## 13.1 CSS Modules란?

CSS Modules는 CSS 클래스를 **로컬 스코프**로 만들어주는 기능입니다. Next.js에서 기본 지원됩니다.

### 13.1.1 문제점: 전역 CSS

```css
/* styles.css */
.button {
  background: blue;
  color: white;
}
```

```typescript
// ComponentA.tsx
import './styles.css';

export default function ComponentA() {
  return <button className="button">버튼 A</button>;
}
```

```typescript
// ComponentB.tsx
import './styles.css';

export default function ComponentB() {
  // ❌ 문제: ComponentA의 .button 스타일과 충돌!
  return <button className="button">버튼 B</button>;
}
```

### 13.1.2 해결책: CSS Modules

```css
/* Button.module.css */
.button {
  background: blue;
  color: white;
}
```

```typescript
// Button.tsx
import styles from './Button.module.css';

export default function Button() {
  // ✅ 고유한 클래스명으로 변환됨: button_abc123
  return <button className={styles.button}>버튼</button>;
}
```

**결과:**
```html
<button class="Button_button__abc123">버튼</button>
```

## 13.2 기본 사용법

### 13.2.1 CSS Module 파일 생성

```css
/* components/Button.module.css */
.primary {
  background-color: #0070f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
}

.primary:hover {
  background-color: #0051cc;
}

.secondary {
  background-color: #eaeaea;
  color: #333;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
}
```

### 13.2.2 컴포넌트에서 사용

```typescript
// components/Button.tsx
import styles from './Button.module.css';

interface Props {
  variant?: 'primary' | 'secondary';
  children: React.ReactNode;
  onClick?: () => void;
}

export default function Button({ variant = 'primary', children, onClick }: Props) {
  return (
    <button
      className={styles[variant]}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

```typescript
// 사용
<Button variant="primary">주 버튼</Button>
<Button variant="secondary">보조 버튼</Button>
```

## 13.3 여러 클래스 결합하기

### 13.3.1 템플릿 리터럴 사용

```css
/* Card.module.css */
.card {
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 20px;
}

.featured {
  border-color: #0070f3;
  box-shadow: 0 4px 8px rgba(0, 112, 243, 0.2);
}

.large {
  padding: 40px;
}
```

```typescript
// components/Card.tsx
import styles from './Card.module.css';

interface Props {
  featured?: boolean;
  large?: boolean;
  children: React.ReactNode;
}

export default function Card({ featured, large, children }: Props) {
  return (
    <div
      className={`
        ${styles.card}
        ${featured ? styles.featured : ''}
        ${large ? styles.large : ''}
      `}
    >
      {children}
    </div>
  );
}
```

### 13.3.2 배열 조인 사용

```typescript
export default function Card({ featured, large, children }: Props) {
  const classes = [
    styles.card,
    featured && styles.featured,
    large && styles.large,
  ].filter(Boolean).join(' ');

  return <div className={classes}>{children}</div>;
}
```

### 13.3.3 clsx 라이브러리 사용

```bash
npm install clsx
```

```typescript
import styles from './Card.module.css';
import clsx from 'clsx';

export default function Card({ featured, large, children }: Props) {
  return (
    <div
      className={clsx(
        styles.card,
        featured && styles.featured,
        large && styles.large
      )}
    >
      {children}
    </div>
  );
}
```

## 13.4 전역 스타일

### 13.4.1 글로벌 CSS

```css
/* app/globals.css */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  line-height: 1.6;
  color: #333;
}

a {
  color: #0070f3;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}
```

```typescript
// app/layout.tsx
import './globals.css';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

### 13.4.2 CSS Modules에서 전역 스타일

```css
/* Button.module.css */

/* 로컬 스타일 */
.button {
  padding: 10px 20px;
}

/* 전역 스타일 */
:global(.no-margin) {
  margin: 0 !important;
}

/* 중첩 */
.button :global(.icon) {
  margin-right: 8px;
}
```

## 13.5 조건부 스타일

### 13.5.1 Props 기반 스타일

```css
/* Alert.module.css */
.alert {
  padding: 16px;
  border-radius: 6px;
  margin-bottom: 16px;
}

.success {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.error {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.warning {
  background-color: #fff3cd;
  color: #856404;
  border: 1px solid #ffeeba;
}

.info {
  background-color: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
}
```

```typescript
// components/Alert.tsx
import styles from './Alert.module.css';
import clsx from 'clsx';

interface Props {
  type: 'success' | 'error' | 'warning' | 'info';
  children: React.ReactNode;
}

export default function Alert({ type, children }: Props) {
  return (
    <div className={clsx(styles.alert, styles[type])}>
      {children}
    </div>
  );
}
```

```typescript
// 사용
<Alert type="success">성공!</Alert>
<Alert type="error">에러 발생!</Alert>
<Alert type="warning">주의하세요</Alert>
<Alert type="info">정보입니다</Alert>
```

### 13.5.2 상태 기반 스타일

```css
/* Toggle.module.css */
.toggle {
  width: 50px;
  height: 24px;
  background-color: #ccc;
  border-radius: 12px;
  position: relative;
  cursor: pointer;
  transition: background-color 0.3s;
}

.toggle.active {
  background-color: #0070f3;
}

.slider {
  width: 20px;
  height: 20px;
  background-color: white;
  border-radius: 50%;
  position: absolute;
  top: 2px;
  left: 2px;
  transition: transform 0.3s;
}

.toggle.active .slider {
  transform: translateX(26px);
}
```

```typescript
// components/Toggle.tsx
'use client';

import { useState } from 'react';
import styles from './Toggle.module.css';
import clsx from 'clsx';

export default function Toggle() {
  const [active, setActive] = useState(false);

  return (
    <div
      className={clsx(styles.toggle, active && styles.active)}
      onClick={() => setActive(!active)}
    >
      <div className={styles.slider} />
    </div>
  );
}
```

## 13.6 중첩 선택자

### 13.6.1 자식 선택자

```css
/* Card.module.css */
.card {
  border: 1px solid #eaeaea;
  padding: 20px;
}

.card h2 {
  margin-top: 0;
  color: #0070f3;
}

.card p {
  color: #666;
  line-height: 1.6;
}

.card .footer {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #eaeaea;
}
```

```typescript
// components/Card.tsx
import styles from './Card.module.css';

export default function Card() {
  return (
    <div className={styles.card}>
      <h2>제목</h2>
      <p>내용입니다.</p>
      <div className={styles.footer}>
        <button>더보기</button>
      </div>
    </div>
  );
}
```

## 13.7 미디어 쿼리

### 13.7.1 반응형 디자인

```css
/* Container.module.css */
.container {
  width: 100%;
  padding: 0 16px;
  margin: 0 auto;
}

/* 모바일 기본 */
.container {
  max-width: 100%;
}

/* 태블릿 */
@media (min-width: 768px) {
  .container {
    max-width: 720px;
    padding: 0 24px;
  }
}

/* 데스크톱 */
@media (min-width: 1024px) {
  .container {
    max-width: 960px;
  }
}

/* 와이드 */
@media (min-width: 1280px) {
  .container {
    max-width: 1200px;
  }
}
```

### 13.7.2 반응형 그리드

```css
/* ProductGrid.module.css */
.grid {
  display: grid;
  gap: 20px;
}

/* 모바일: 1열 */
.grid {
  grid-template-columns: 1fr;
}

/* 태블릿: 2열 */
@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* 데스크톱: 3열 */
@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
  }
}

/* 와이드: 4열 */
@media (min-width: 1440px) {
  .grid {
    grid-template-columns: repeat(4, 1fr);
  }
}
```

## 13.8 애니메이션

### 13.8.1 CSS 애니메이션

```css
/* Loading.module.css */
.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #0070f3;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.fadeIn {
  animation: fadeIn 0.5s ease-in;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

```typescript
// components/Loading.tsx
import styles from './Loading.module.css';

export default function Loading() {
  return (
    <div className={styles.fadeIn}>
      <div className={styles.spinner} />
      <p>로딩 중...</p>
    </div>
  );
}
```

### 13.8.2 Transition

```css
/* Button.module.css */
.button {
  background-color: #0070f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.button:hover {
  background-color: #0051cc;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 112, 243, 0.3);
}

.button:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(0, 112, 243, 0.3);
}
```

## 13.9 실전 예제: 블로그 카드

```css
/* BlogCard.module.css */
.card {
  border: 1px solid #eaeaea;
  border-radius: 8px;
  overflow: hidden;
  transition: all 0.3s ease;
  background: white;
}

.card:hover {
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  transform: translateY(-4px);
}

.image {
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.content {
  padding: 20px;
}

.title {
  font-size: 24px;
  font-weight: bold;
  margin: 0 0 12px 0;
  color: #333;
}

.excerpt {
  font-size: 16px;
  color: #666;
  line-height: 1.6;
  margin: 0 0 16px 0;
}

.meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 14px;
  color: #999;
}

.author {
  display: flex;
  align-items: center;
  gap: 8px;
}

.avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
}

.date {
  color: #999;
}

/* 반응형 */
@media (max-width: 768px) {
  .title {
    font-size: 20px;
  }

  .excerpt {
    font-size: 14px;
  }

  .content {
    padding: 16px;
  }
}
```

```typescript
// components/BlogCard.tsx
import styles from './BlogCard.module.css';
import Link from 'next/link';

interface Props {
  slug: string;
  title: string;
  excerpt: string;
  author: {
    name: string;
    avatar: string;
  };
  date: string;
  imageUrl: string;
}

export default function BlogCard({ slug, title, excerpt, author, date, imageUrl }: Props) {
  return (
    <Link href={`/blog/${slug}`} className={styles.card}>
      <img src={imageUrl} alt={title} className={styles.image} />

      <div className={styles.content}>
        <h2 className={styles.title}>{title}</h2>
        <p className={styles.excerpt}>{excerpt}</p>

        <div className={styles.meta}>
          <div className={styles.author}>
            <img src={author.avatar} alt={author.name} className={styles.avatar} />
            <span>{author.name}</span>
          </div>
          <time className={styles.date}>{date}</time>
        </div>
      </div>
    </Link>
  );
}
```

## 13.10 CSS Modules 모범 사례

### 13.10.1 네이밍 규칙

```css
/* ✅ 좋은 예시 */
.button { }
.buttonPrimary { }  /* camelCase */
.button-primary { } /* kebab-case */

/* ❌ 나쁜 예시 */
.Button { }         /* PascalCase는 컴포넌트명으로 혼동 */
.btn { }            /* 축약은 피하기 */
```

### 13.10.2 컴포넌트별 CSS 파일

```
components/
├── Button/
│   ├── Button.tsx
│   └── Button.module.css
├── Card/
│   ├── Card.tsx
│   └── Card.module.css
└── Header/
    ├── Header.tsx
    └── Header.module.css
```

### 13.10.3 공통 스타일 변수

```css
/* styles/variables.css */
:root {
  --color-primary: #0070f3;
  --color-secondary: #7928ca;
  --color-success: #0070f3;
  --color-error: #e00;

  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  --border-radius: 6px;
  --transition: 0.3s ease;
}
```

```css
/* Button.module.css */
.button {
  background-color: var(--color-primary);
  padding: var(--spacing-md) var(--spacing-lg);
  border-radius: var(--border-radius);
  transition: var(--transition);
}
```

## 핵심 요약

1. **CSS Modules**
   - 로컬 스코프 CSS
   - `.module.css` 확장자
   - 클래스명 충돌 방지

2. **사용법**
   - `import styles from './styles.module.css'`
   - `className={styles.className}`
   - clsx로 조건부 스타일

3. **장점**
   - 스타일 격리
   - 타입 안전성 (TypeScript)
   - 간단한 사용법

4. **조합**
   - 여러 클래스 결합
   - 조건부 스타일
   - 동적 스타일

5. **모범 사례**
   - 컴포넌트별 CSS 파일
   - 일관된 네이밍
   - CSS 변수 활용

## 연습 문제

### 문제 1
버튼 컴포넌트를 만들고 primary/secondary 스타일을 적용하세요.

<details>
<summary>정답</summary>

```css
/* Button.module.css */
.button {
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.primary {
  background: #0070f3;
  color: white;
}

.secondary {
  background: #eaeaea;
  color: #333;
}
```

```typescript
// Button.tsx
import styles from './Button.module.css';

interface Props {
  variant: 'primary' | 'secondary';
  children: React.ReactNode;
}

export default function Button({ variant, children }: Props) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  );
}
```
</details>

### 문제 2
반응형 그리드를 만드세요 (모바일 1열, 태블릿 2열, 데스크톱 3열).

<details>
<summary>정답</summary>

```css
/* Grid.module.css */
.grid {
  display: grid;
  gap: 20px;
  grid-template-columns: 1fr;
}

@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

```typescript
// Grid.tsx
import styles from './Grid.module.css';

export default function Grid({ children }: { children: React.ReactNode }) {
  return <div className={styles.grid}>{children}</div>;
}
```
</details>

## 다음 단계

다음 챕터에서는:
- Tailwind CSS 설치
- 유틸리티 클래스
- 반응형 디자인
- 다크 모드

[Chapter 14. Tailwind CSS →](chapter14-tailwind-css.md)