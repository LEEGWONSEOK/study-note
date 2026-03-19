# Chapter 24: Vercel 배포

## 1. Vercel이란?

### 1.1 개념

Vercel은 Next.js를 만든 회사에서 제공하는 클라우드 플랫폼입니다.

**장점:**
- Next.js 최적화 (제로 설정)
- 자동 HTTPS
- 글로벌 CDN
- 무료 티어
- Git 통합 (자동 배포)

### 1.2 가격

```
Hobby (무료):
- 대역폭: 100GB/월
- 빌드: 6000분/월
- 개인 프로젝트용

Pro ($20/월):
- 대역폭: 1TB/월
- 팀 협업
- 비즈니스용
```

## 2. Vercel 배포하기

### 2.1 준비사항

**1. 프로젝트 준비**
```json
// package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  }
}
```

**2. Git 저장소**
```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/username/repo.git
git push -u origin main
```

### 2.2 Vercel CLI로 배포

```bash
# Vercel CLI 설치
npm install -g vercel

# 로그인
vercel login

# 배포
vercel
```

**대화형 설정:**
```
? Set up and deploy "~/my-project"? [Y/n] y
? Which scope do you want to deploy to? My Account
? Link to existing project? [y/N] n
? What's your project's name? my-nextjs-app
? In which directory is your code located? ./
```

**결과:**
```
✓ Production: https://my-nextjs-app.vercel.app
```

### 2.3 GitHub 연동으로 배포

**1. Vercel 대시보드**
- https://vercel.com 접속
- "Add New Project" 클릭

**2. GitHub 저장소 선택**
- "Import Git Repository"
- 권한 승인
- 저장소 선택

**3. 프로젝트 설정**
```
Project Name: my-nextjs-app
Framework Preset: Next.js (자동 감지)
Root Directory: ./
Build Command: next build (자동)
Output Directory: .next (자동)
```

**4. 환경 변수 설정**
```
DATABASE_URL=postgresql://...
NEXT_PUBLIC_API_URL=https://api.example.com
JWT_SECRET=your-secret
```

**5. 배포**
- "Deploy" 클릭
- 빌드 진행 상황 확인
- 완료 후 URL 자동 생성

## 3. 자동 배포

### 3.1 브랜치별 배포

```
main 브랜치     → Production (my-app.vercel.app)
develop 브랜치  → Preview (my-app-git-develop.vercel.app)
feature 브랜치  → Preview (my-app-git-feature.vercel.app)
```

**동작:**
```bash
git checkout -b feature/new-ui
git add .
git commit -m "Add new UI"
git push origin feature/new-ui
```

→ Vercel이 자동으로 Preview 배포 생성

### 3.2 Pull Request 프리뷰

```
1. PR 생성
2. Vercel bot이 Preview URL 댓글 추가
3. 코드 변경 시 자동 재배포
4. PR 머지 → Production 배포
```

## 4. 환경 변수

### 4.1 Vercel 대시보드에서 설정

```
Settings → Environment Variables

Name:         DATABASE_URL
Value:        postgresql://...
Environments: ☑ Production ☑ Preview ☑ Development
```

### 4.2 로컬에서 사용

```bash
# Vercel 환경 변수 다운로드
vercel env pull .env.local
```

```bash
# .env.local
DATABASE_URL="postgresql://..."
NEXT_PUBLIC_API_URL="https://api.example.com"
```

### 4.3 환경별 변수

```
Production:
  DATABASE_URL=postgresql://prod-db.com
  NEXT_PUBLIC_API_URL=https://api.example.com

Preview:
  DATABASE_URL=postgresql://staging-db.com
  NEXT_PUBLIC_API_URL=https://staging-api.example.com

Development:
  DATABASE_URL=postgresql://localhost:5432
  NEXT_PUBLIC_API_URL=http://localhost:3001
```

## 5. 커스텀 도메인

### 5.1 도메인 연결

**Vercel 대시보드:**
```
Settings → Domains → Add Domain

1. 도메인 입력: example.com
2. DNS 설정:
   Type: A
   Name: @
   Value: 76.76.21.21

   Type: CNAME
   Name: www
   Value: cname.vercel-dns.com
```

**확인:**
```bash
nslookup example.com
```

### 5.2 서브도메인

```
blog.example.com → Vercel Project A
api.example.com  → Vercel Project B
```

### 5.3 자동 HTTPS

Vercel은 자동으로 SSL 인증서를 발급하고 갱신합니다.

```
http://example.com  → https://example.com (자동 리다이렉트)
```

## 6. 빌드 설정

### 6.1 vercel.json

```json
// vercel.json
{
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "framework": "nextjs",
  "outputDirectory": ".next"
}
```

### 6.2 리다이렉트

```json
// vercel.json
{
  "redirects": [
    {
      "source": "/old-path",
      "destination": "/new-path",
      "permanent": true
    },
    {
      "source": "/blog/:slug",
      "destination": "/posts/:slug",
      "permanent": false
    }
  ]
}
```

### 6.3 헤더 설정

```json
// vercel.json
{
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "Access-Control-Allow-Origin",
          "value": "*"
        },
        {
          "key": "Access-Control-Allow-Methods",
          "value": "GET, POST, PUT, DELETE, OPTIONS"
        }
      ]
    }
  ]
}
```

## 7. 성능 최적화

### 7.1 Analytics

**설정:**
```bash
npm install @vercel/analytics
```

```tsx
// app/layout.tsx
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

**확인:**
- Vercel 대시보드 → Analytics
- 페이지뷰, 성능 지표 확인

### 7.2 Speed Insights

```bash
npm install @vercel/speed-insights
```

```tsx
// app/layout.tsx
import { SpeedInsights } from '@vercel/speed-insights/next';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        {children}
        <SpeedInsights />
      </body>
    </html>
  );
}
```

**지표:**
- FCP (First Contentful Paint)
- LCP (Largest Contentful Paint)
- CLS (Cumulative Layout Shift)
- FID (First Input Delay)

### 7.3 Edge Functions

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Edge Runtime에서 실행 (빠름!)
  const country = request.geo?.country || 'US';

  return NextResponse.rewrite(
    new URL(`/${country}${request.nextUrl.pathname}`, request.url)
  );
}

export const config = {
  matcher: '/products/:path*'
};
```

## 8. 데이터베이스 연동

### 8.1 Vercel Postgres

```bash
# Vercel 대시보드에서 생성 후
npm install @vercel/postgres
```

```typescript
// app/api/posts/route.ts
import { sql } from '@vercel/postgres';

export async function GET() {
  const { rows } = await sql`SELECT * FROM posts`;
  return Response.json(rows);
}
```

### 8.2 외부 데이터베이스

**Supabase, PlanetScale, Railway 등:**
```bash
# .env.local
DATABASE_URL="postgresql://..."
```

```typescript
// lib/db.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export default prisma;
```

## 9. 모니터링

### 9.1 로그 확인

**Vercel CLI:**
```bash
vercel logs
vercel logs --follow # 실시간
vercel logs production # Production만
```

**대시보드:**
- Deployments → 특정 배포 클릭 → Logs

### 9.2 에러 추적

```bash
npm install @sentry/nextjs
```

```javascript
// sentry.client.config.js
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
});
```

## 10. 실습 문제

### 문제 1: 다단계 배포
Development → Staging → Production 워크플로우를 구성하세요.

**요구사항:**
- develop 브랜치 → Staging
- main 브랜치 → Production
- PR 머지 승인 필요

### 문제 2: A/B 테스팅
Vercel Edge Middleware를 사용하여 두 가지 버전의 페이지를 랜덤으로 보여주세요.

### 문제 3: 글로벌 배포
지역별로 다른 콘텐츠를 보여주는 사이트를 만드세요.

## 11. 실습 해답

### 문제 1 해답

**GitHub Workflow:**
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - develop
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
          vercel-args: ${{ github.ref == 'refs/heads/main' && '--prod' || '' }}
```

**브랜치 보호 규칙 (GitHub):**
```
Settings → Branches → Branch protection rules

Branch name pattern: main

☑ Require a pull request before merging
  ☑ Require approvals: 1
☑ Require status checks to pass before merging
  ☑ vercel (deployment)
```

**워크플로우:**
```bash
# 1. Feature 개발
git checkout -b feature/new-feature
git commit -m "Add feature"
git push origin feature/new-feature

# 2. PR to develop (Staging)
# → Vercel이 Preview 생성
# → 테스트 후 머지

# 3. PR to main (Production)
# → 승인 필요
# → 머지 후 자동 배포
```

### 문제 2 해답

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  // 쿠키 확인 (이미 할당된 경우)
  const variant = request.cookies.get('ab-test-variant')?.value;

  if (variant) {
    // 기존 variant 사용
    return NextResponse.rewrite(
      new URL(`/experiments/${variant}${request.nextUrl.pathname}`, request.url)
    );
  }

  // 랜덤 할당 (50/50)
  const newVariant = Math.random() < 0.5 ? 'a' : 'b';

  const response = NextResponse.rewrite(
    new URL(`/experiments/${newVariant}${request.nextUrl.pathname}`, request.url)
  );

  // 쿠키 설정 (variant 고정)
  response.cookies.set('ab-test-variant', newVariant, {
    maxAge: 60 * 60 * 24 * 7 // 7일
  });

  return response;
}

export const config = {
  matcher: '/'
};
```

**디렉토리 구조:**
```
app/
├── page.tsx                    # 원본 (사용 안 함)
└── experiments/
    ├── a/
    │   └── page.tsx           # Variant A
    └── b/
        └── page.tsx           # Variant B
```

**Variant A:**
```tsx
// app/experiments/a/page.tsx
'use client';

import { useEffect } from 'react';

export default function VariantA() {
  useEffect(() => {
    // Analytics 이벤트
    if (typeof window !== 'undefined') {
      window.gtag?.('event', 'ab_test_view', {
        variant: 'a'
      });
    }
  }, []);

  return (
    <div className="bg-blue-500 text-white p-8">
      <h1 className="text-4xl font-bold">Variant A</h1>
      <p>파란색 배경 버전</p>
      <button className="mt-4 px-6 py-3 bg-white text-blue-500 rounded">
        시작하기
      </button>
    </div>
  );
}
```

**Variant B:**
```tsx
// app/experiments/b/page.tsx
'use client';

import { useEffect } from 'react';

export default function VariantB() {
  useEffect(() => {
    // Analytics 이벤트
    if (typeof window !== 'undefined') {
      window.gtag?.('event', 'ab_test_view', {
        variant: 'b'
      });
    }
  }, []);

  return (
    <div className="bg-green-500 text-white p-8">
      <h1 className="text-4xl font-bold">Variant B</h1>
      <p>초록색 배경 버전</p>
      <button className="mt-4 px-6 py-3 bg-white text-green-500 rounded">
        지금 시작
      </button>
    </div>
  );
}
```

### 문제 3 해답

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

const SUPPORTED_COUNTRIES = ['US', 'KR', 'JP', 'CN'];

export function middleware(request: NextRequest) {
  // Vercel의 Geo 정보 사용
  const country = request.geo?.country || 'US';

  // 지원하는 국가인지 확인
  const targetCountry = SUPPORTED_COUNTRIES.includes(country) ? country : 'US';

  // 이미 올바른 경로에 있는지 확인
  if (request.nextUrl.pathname.startsWith(`/${targetCountry.toLowerCase()}`)) {
    return NextResponse.next();
  }

  // 국가별 경로로 리다이렉트
  return NextResponse.rewrite(
    new URL(`/${targetCountry.toLowerCase()}${request.nextUrl.pathname}`, request.url)
  );
}

export const config = {
  matcher: '/:path*'
};
```

**디렉토리 구조:**
```
app/
├── us/
│   ├── page.tsx
│   └── products/
│       └── page.tsx
├── kr/
│   ├── page.tsx
│   └── products/
│       └── page.tsx
├── jp/
│   ├── page.tsx
│   └── products/
│       └── page.tsx
└── cn/
    ├── page.tsx
    └── products/
        └── page.tsx
```

**미국 버전:**
```tsx
// app/us/page.tsx
export default function USHomePage() {
  return (
    <div>
      <h1>Welcome to Our Store</h1>
      <p>Free shipping on orders over $50</p>
      <p>Currency: USD</p>
    </div>
  );
}
```

**한국 버전:**
```tsx
// app/kr/page.tsx
export default function KRHomePage() {
  return (
    <div>
      <h1>스토어에 오신 것을 환영합니다</h1>
      <p>5만원 이상 무료배송</p>
      <p>통화: KRW</p>
    </div>
  );
}
```

**환경 변수 (지역별):**
```bash
# US
NEXT_PUBLIC_CURRENCY=USD
NEXT_PUBLIC_SHIPPING_THRESHOLD=50

# KR
NEXT_PUBLIC_CURRENCY=KRW
NEXT_PUBLIC_SHIPPING_THRESHOLD=50000
```

**고급: 다국어 + 통화:**
```tsx
// app/[country]/layout.tsx
import { notFound } from 'next/navigation';

const COUNTRIES = {
  us: { locale: 'en-US', currency: 'USD', name: 'United States' },
  kr: { locale: 'ko-KR', currency: 'KRW', name: '대한민국' },
  jp: { locale: 'ja-JP', currency: 'JPY', name: '日本' },
  cn: { locale: 'zh-CN', currency: 'CNY', name: '中国' },
};

export default function CountryLayout({
  children,
  params
}: {
  children: React.ReactNode;
  params: { country: string };
}) {
  const config = COUNTRIES[params.country as keyof typeof COUNTRIES];

  if (!config) {
    notFound();
  }

  return (
    <html lang={config.locale}>
      <body>
        <header className="bg-gray-800 text-white p-4">
          <div className="container mx-auto">
            <span>{config.name}</span>
            <span className="ml-4">Currency: {config.currency}</span>
          </div>
        </header>
        {children}
      </body>
    </html>
  );
}
```

## 핵심 요약

1. **Vercel**: Next.js 최적화 클라우드 플랫폼
2. **자동 배포**: Git push → 자동 빌드 및 배포
3. **환경별 배포**: Production, Preview, Development
4. **커스텀 도메인**: 자동 HTTPS 포함
5. **Edge Functions**: 글로벌 저지연 실행

[← Chapter 23: 캐싱 전략](../part08-optimization/chapter23-caching-strategies.md) | [Chapter 25: Docker 배포 →](./chapter25-docker-deployment.md)