# Chapter 15. CSS-in-JS (Styled Components)

## 15.1 CSS-in-JS란?

CSS-in-JS는 JavaScript 파일 안에서 CSS를 작성하는 방식입니다. Styled Components가 가장 인기 있는 라이브러리입니다.

### 15.1.1 기존 방식 vs CSS-in-JS

```css
/* 기존 CSS */
.button {
  background-color: blue;
  color: white;
  padding: 10px 20px;
}
```

```typescript
// CSS-in-JS (Styled Components)
import styled from 'styled-components';

const Button = styled.button`
  background-color: blue;
  color: white;
  padding: 10px 20px;
`;
```

**장점:**
- ✅ 스타일 격리 (자동으로 고유 클래스명 생성)
- ✅ 동적 스타일링 (props 기반)
- ✅ JavaScript의 모든 기능 활용
- ✅ 타입 안전성 (TypeScript)

**단점:**
- ❌ 런타임 오버헤드
- ❌ 번들 크기 증가
- ❌ SSR 설정 복잡

## 15.2 Next.js에서 Styled Components 설정

### 15.2.1 설치

```bash
npm install styled-components
npm install -D @types/styled-components
```

### 15.2.2 Next.js 설정

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  compiler: {
    styledComponents: true,
  },
}

module.exports = nextConfig
```

### 15.2.3 레지스트리 설정 (App Router)

```typescript
// lib/registry.tsx
'use client';

import React, { useState } from 'react';
import { useServerInsertedHTML } from 'next/navigation';
import { ServerStyleSheet, StyleSheetManager } from 'styled-components';

export default function StyledComponentsRegistry({
  children,
}: {
  children: React.ReactNode;
}) {
  const [styledComponentsStyleSheet] = useState(() => new ServerStyleSheet());

  useServerInsertedHTML(() => {
    const styles = styledComponentsStyleSheet.getStyleElement();
    styledComponentsStyleSheet.instance.clearTag();
    return <>{styles}</>;
  });

  if (typeof window !== 'undefined') return <>{children}</>;

  return (
    <StyleSheetManager sheet={styledComponentsStyleSheet.instance}>
      {children}
    </StyleSheetManager>
  );
}
```

```typescript
// app/layout.tsx
import StyledComponentsRegistry from '@/lib/registry';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <StyledComponentsRegistry>{children}</StyledComponentsRegistry>
      </body>
    </html>
  );
}
```

## 15.3 기본 사용법

### 15.3.1 간단한 컴포넌트

```typescript
// components/Button.tsx
'use client';

import styled from 'styled-components';

const StyledButton = styled.button`
  background-color: #0070f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.3s;

  &:hover {
    background-color: #0051cc;
  }

  &:active {
    transform: scale(0.98);
  }
`;

export default function Button({ children }: { children: React.ReactNode }) {
  return <StyledButton>{children}</StyledButton>;
}
```

### 15.3.2 Props 기반 스타일링

```typescript
// components/Button.tsx
'use client';

import styled from 'styled-components';

interface ButtonProps {
  $variant?: 'primary' | 'secondary' | 'danger';
  $size?: 'sm' | 'md' | 'lg';
}

const StyledButton = styled.button<ButtonProps>`
  /* 기본 스타일 */
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;

  /* 변형 */
  ${props => {
    switch (props.$variant) {
      case 'secondary':
        return `
          background-color: #eaeaea;
          color: #333;
          &:hover { background-color: #d4d4d4; }
        `;
      case 'danger':
        return `
          background-color: #e00;
          color: white;
          &:hover { background-color: #c00; }
        `;
      default:
        return `
          background-color: #0070f3;
          color: white;
          &:hover { background-color: #0051cc; }
        `;
    }
  }}

  /* 크기 */
  ${props => {
    switch (props.$size) {
      case 'sm':
        return `
          padding: 8px 16px;
          font-size: 14px;
        `;
      case 'lg':
        return `
          padding: 16px 32px;
          font-size: 18px;
        `;
      default:
        return `
          padding: 12px 24px;
          font-size: 16px;
        `;
    }
  }}
`;

export default function Button({
  variant = 'primary',
  size = 'md',
  children,
  ...props
}: {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
  onClick?: () => void;
}) {
  return (
    <StyledButton $variant={variant} $size={size} {...props}>
      {children}
    </StyledButton>
  );
}
```

**주의:** Next.js 13+에서는 props 앞에 `$` 접두사를 붙여야 DOM에 전달되지 않습니다.

### 15.3.3 스타일 확장

```typescript
'use client';

import styled from 'styled-components';

const Button = styled.button`
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
`;

// Button 스타일을 상속
const PrimaryButton = styled(Button)`
  background-color: #0070f3;
  color: white;
`;

const SecondaryButton = styled(Button)`
  background-color: #eaeaea;
  color: #333;
`;

export default function Buttons() {
  return (
    <>
      <PrimaryButton>주 버튼</PrimaryButton>
      <SecondaryButton>보조 버튼</SecondaryButton>
    </>
  );
}
```

## 15.4 테마 시스템

### 15.4.1 테마 정의

```typescript
// styles/theme.ts
export const lightTheme = {
  colors: {
    primary: '#0070f3',
    secondary: '#7928ca',
    background: '#ffffff',
    text: '#333333',
    border: '#eaeaea',
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
  },
  borderRadius: '6px',
};

export const darkTheme = {
  colors: {
    primary: '#0070f3',
    secondary: '#7928ca',
    background: '#1a1a1a',
    text: '#ffffff',
    border: '#333333',
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
  },
  borderRadius: '6px',
};

export type Theme = typeof lightTheme;
```

### 15.4.2 ThemeProvider 설정

```typescript
// components/ThemeProvider.tsx
'use client';

import { useState } from 'react';
import { ThemeProvider as StyledThemeProvider } from 'styled-components';
import { lightTheme, darkTheme } from '@/styles/theme';

export default function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [isDark, setIsDark] = useState(false);

  return (
    <StyledThemeProvider theme={isDark ? darkTheme : lightTheme}>
      {children}
    </StyledThemeProvider>
  );
}
```

```typescript
// app/layout.tsx
import StyledComponentsRegistry from '@/lib/registry';
import ThemeProvider from '@/components/ThemeProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <StyledComponentsRegistry>
          <ThemeProvider>{children}</ThemeProvider>
        </StyledComponentsRegistry>
      </body>
    </html>
  );
}
```

### 15.4.3 테마 사용

```typescript
'use client';

import styled from 'styled-components';

const Container = styled.div`
  background-color: ${props => props.theme.colors.background};
  color: ${props => props.theme.colors.text};
  padding: ${props => props.theme.spacing.lg};
  border: 1px solid ${props => props.theme.colors.border};
  border-radius: ${props => props.theme.borderRadius};
`;

const Title = styled.h1`
  color: ${props => props.theme.colors.primary};
  margin-bottom: ${props => props.theme.spacing.md};
`;

export default function Card() {
  return (
    <Container>
      <Title>카드 제목</Title>
      <p>카드 내용입니다.</p>
    </Container>
  );
}
```

## 15.5 GlobalStyle

### 15.5.1 전역 스타일 정의

```typescript
// styles/GlobalStyle.tsx
'use client';

import { createGlobalStyle } from 'styled-components';

const GlobalStyle = createGlobalStyle`
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background-color: ${props => props.theme.colors.background};
    color: ${props => props.theme.colors.text};
    line-height: 1.6;
  }

  a {
    color: ${props => props.theme.colors.primary};
    text-decoration: none;

    &:hover {
      text-decoration: underline;
    }
  }

  button {
    font-family: inherit;
  }
`;

export default GlobalStyle;
```

### 15.5.2 GlobalStyle 적용

```typescript
// components/ThemeProvider.tsx
'use client';

import { ThemeProvider as StyledThemeProvider } from 'styled-components';
import GlobalStyle from '@/styles/GlobalStyle';
import { lightTheme } from '@/styles/theme';

export default function ThemeProvider({ children }: { children: React.ReactNode }) {
  return (
    <StyledThemeProvider theme={lightTheme}>
      <GlobalStyle />
      {children}
    </StyledThemeProvider>
  );
}
```

## 15.6 미디어 쿼리

### 15.6.1 기본 미디어 쿼리

```typescript
'use client';

import styled from 'styled-components';

const Container = styled.div`
  padding: 16px;

  @media (min-width: 768px) {
    padding: 24px;
  }

  @media (min-width: 1024px) {
    padding: 32px;
    max-width: 1200px;
    margin: 0 auto;
  }
`;
```

### 15.6.2 재사용 가능한 미디어 쿼리

```typescript
// styles/media.ts
export const breakpoints = {
  mobile: '640px',
  tablet: '768px',
  desktop: '1024px',
  wide: '1280px',
};

export const media = {
  mobile: `@media (min-width: ${breakpoints.mobile})`,
  tablet: `@media (min-width: ${breakpoints.tablet})`,
  desktop: `@media (min-width: ${breakpoints.desktop})`,
  wide: `@media (min-width: ${breakpoints.wide})`,
};
```

```typescript
'use client';

import styled from 'styled-components';
import { media } from '@/styles/media';

const Grid = styled.div`
  display: grid;
  grid-template-columns: 1fr;
  gap: 16px;

  ${media.tablet} {
    grid-template-columns: repeat(2, 1fr);
  }

  ${media.desktop} {
    grid-template-columns: repeat(3, 1fr);
  }

  ${media.wide} {
    grid-template-columns: repeat(4, 1fr);
  }
`;
```

## 15.7 애니메이션

### 15.7.1 키프레임 애니메이션

```typescript
'use client';

import styled, { keyframes } from 'styled-components';

const spin = keyframes`
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
`;

const Spinner = styled.div`
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #0070f3;
  border-radius: 50%;
  animation: ${spin} 1s linear infinite;
`;

const fadeIn = keyframes`
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
`;

const Card = styled.div`
  animation: ${fadeIn} 0.5s ease-in;
`;
```

### 15.7.2 Transition

```typescript
'use client';

import styled from 'styled-components';

const Button = styled.button`
  background-color: #0070f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;

  &:hover {
    background-color: #0051cc;
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 112, 243, 0.3);
  }

  &:active {
    transform: translateY(0);
  }
`;
```

## 15.8 실전 예제: 상품 카드

```typescript
// components/ProductCard.tsx
'use client';

import styled from 'styled-components';
import { media } from '@/styles/media';

const Card = styled.article`
  border: 1px solid ${props => props.theme.colors.border};
  border-radius: ${props => props.theme.borderRadius};
  overflow: hidden;
  background: ${props => props.theme.colors.background};
  transition: all 0.3s ease;

  &:hover {
    transform: translateY(-4px);
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  }
`;

const Image = styled.img`
  width: 100%;
  height: 200px;
  object-fit: cover;
`;

const Content = styled.div`
  padding: ${props => props.theme.spacing.lg};

  ${media.mobile} {
    padding: ${props => props.theme.spacing.md};
  }
`;

const Title = styled.h3`
  font-size: 20px;
  font-weight: bold;
  color: ${props => props.theme.colors.text};
  margin-bottom: ${props => props.theme.spacing.sm};

  ${media.mobile} {
    font-size: 18px;
  }
`;

const Description = styled.p`
  color: ${props => props.theme.colors.text};
  opacity: 0.7;
  line-height: 1.6;
  margin-bottom: ${props => props.theme.spacing.md};
`;

const Price = styled.div`
  font-size: 24px;
  font-weight: bold;
  color: ${props => props.theme.colors.primary};
  margin-bottom: ${props => props.theme.spacing.md};
`;

const Button = styled.button`
  width: 100%;
  background-color: ${props => props.theme.colors.primary};
  color: white;
  padding: ${props => props.theme.spacing.md};
  border: none;
  border-radius: ${props => props.theme.borderRadius};
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    opacity: 0.9;
  }

  &:active {
    transform: scale(0.98);
  }
`;

interface Props {
  title: string;
  description: string;
  price: number;
  imageUrl: string;
}

export default function ProductCard({ title, description, price, imageUrl }: Props) {
  return (
    <Card>
      <Image src={imageUrl} alt={title} />
      <Content>
        <Title>{title}</Title>
        <Description>{description}</Description>
        <Price>{price.toLocaleString()}원</Price>
        <Button>장바구니 담기</Button>
      </Content>
    </Card>
  );
}
```

## 15.9 TypeScript 타입 지원

### 15.9.1 테마 타입 정의

```typescript
// styled.d.ts
import 'styled-components';
import { Theme } from './styles/theme';

declare module 'styled-components' {
  export interface DefaultTheme extends Theme {}
}
```

이제 `props.theme`에서 자동완성이 동작합니다!

## 15.10 CSS Modules vs Tailwind vs Styled Components

| 특징 | CSS Modules | Tailwind | Styled Components |
|------|-------------|----------|-------------------|
| **학습 곡선** | 낮음 | 중간 | 높음 |
| **번들 크기** | 작음 | 중간 | 큼 |
| **성능** | 좋음 | 좋음 | 중간 |
| **동적 스타일** | 어려움 | 중간 | 쉬움 |
| **타입 안전성** | 없음 | 없음 | 있음 |
| **재사용성** | 중간 | 높음 | 높음 |

**선택 가이드:**
- **CSS Modules**: 전통적인 CSS를 선호, 간단한 프로젝트
- **Tailwind**: 빠른 개발, 일관된 디자인 시스템
- **Styled Components**: 동적 스타일링, 컴포넌트 기반 개발

## 핵심 요약

1. **Styled Components**
   - CSS-in-JS 라이브러리
   - JavaScript 안에서 CSS 작성
   - 동적 스타일링 강력

2. **설정**
   - Next.js 설정 필요
   - Registry 설정 (SSR)
   - TypeScript 타입 지원

3. **기능**
   - Props 기반 스타일링
   - 테마 시스템
   - 미디어 쿼리
   - 애니메이션

4. **장단점**
   - ✅ 동적 스타일링
   - ✅ 타입 안전성
   - ❌ 런타임 오버헤드
   - ❌ 설정 복잡

5. **언제 사용?**
   - 복잡한 동적 스타일
   - 테마 시스템 필요
   - TypeScript 프로젝트

## 연습 문제

### 문제 1
Styled Components로 버튼을 만드세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import styled from 'styled-components';

const Button = styled.button`
  background-color: #0070f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;

  &:hover {
    background-color: #0051cc;
  }
`;

export default function MyButton() {
  return <Button>클릭하세요</Button>;
}
```
</details>

### 문제 2
Props로 색상을 변경할 수 있는 버튼을 만드세요.

<details>
<summary>정답</summary>

```typescript
'use client';

import styled from 'styled-components';

interface ButtonProps {
  $color: string;
}

const Button = styled.button<ButtonProps>`
  background-color: ${props => props.$color};
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
`;

export default function ColorButton() {
  return (
    <>
      <Button $color="blue">파랑</Button>
      <Button $color="red">빨강</Button>
      <Button $color="green">초록</Button>
    </>
  );
}
```
</details>

## Part 05 완료!

Part 05에서 배운 내용:
- ✅ CSS Modules (로컬 스코프)
- ✅ Tailwind CSS (유틸리티 우선)
- ✅ Styled Components (CSS-in-JS)

다음 Part에서는:
- Server Actions
- 폼 처리
- 데이터 Mutation
- Revalidation

[Part 06: 폼과 데이터 처리 →](../part06-forms-data/chapter16-server-actions.md)