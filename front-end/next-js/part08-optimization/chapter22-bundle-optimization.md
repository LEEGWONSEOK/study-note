# Chapter 22: 번들 최적화

## 1. 번들 크기의 중요성

### 1.1 왜 번들 크기가 중요한가?

```
큰 번들 크기 = 느린 로딩 = 나쁜 사용자 경험
```

**문제점:**
- 초기 로딩 시간 증가
- 모바일 데이터 소비
- TTI (Time to Interactive) 지연

**목표:**
- First Load JS: < 80KB (권장)
- 전체 페이지: < 200KB (권장)

## 2. 번들 분석

### 2.1 Next.js Bundle Analyzer

```bash
npm install @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // 기존 설정...
});
```

**사용법:**
```bash
ANALYZE=true npm run build
```

브라우저에서 자동으로 번들 분석 결과가 열립니다.

### 2.2 분석 결과 읽기

```
📦 client.js (120KB)
  ├── react (45KB)           ← 큰 라이브러리
  ├── lodash (70KB)          ← ⚠️ 불필요한 전체 import
  └── my-code (5KB)

📦 page.js (80KB)
  ├── chart.js (75KB)        ← ⚠️ 페이지별로 분리 필요
  └── my-code (5KB)
```

## 3. 동적 Import (Code Splitting)

### 3.1 기본 사용법

**Before (모든 코드가 한 번에 로드):**
```tsx
import HeavyComponent from '@/components/HeavyComponent';

export default function Page() {
  return <HeavyComponent />;
}
```

**After (필요할 때만 로드):**
```tsx
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('@/components/HeavyComponent'));

export default function Page() {
  return <HeavyComponent />;
}
```

### 3.2 로딩 상태

```tsx
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(
  () => import('@/components/HeavyComponent'),
  {
    loading: () => <div>로딩 중...</div>
  }
);
```

### 3.3 SSR 비활성화

```tsx
// 클라이언트에서만 로드 (window, document 사용 시)
const ClientOnlyComponent = dynamic(
  () => import('@/components/ClientOnlyComponent'),
  { ssr: false }
);
```

### 3.4 Named Export

```tsx
// components/Charts.tsx
export function LineChart() { /* ... */ }
export function BarChart() { /* ... */ }

// page.tsx
const LineChart = dynamic(
  () => import('@/components/Charts').then(mod => mod.LineChart)
);
```

## 4. 조건부 로딩

### 4.1 사용자 인터랙션 후 로드

```tsx
'use client';

import { useState } from 'react';
import dynamic from 'next/dynamic';

const HeavyModal = dynamic(() => import('@/components/HeavyModal'));

export default function Page() {
  const [showModal, setShowModal] = useState(false);

  return (
    <>
      <button onClick={() => setShowModal(true)}>
        모달 열기
      </button>

      {showModal && <HeavyModal />}
    </>
  );
}
```

### 4.2 뷰포트 진입 시 로드

```tsx
'use client';

import { useEffect, useState, useRef } from 'react';
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('@/components/HeavyChart'));

export default function Page() {
  const [shouldLoad, setShouldLoad] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setShouldLoad(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref}>
      {shouldLoad ? <HeavyChart /> : <div>스크롤하여 차트 로드...</div>}
    </div>
  );
}
```

## 5. 라이브러리 최적화

### 5.1 Tree Shaking

**❌ 나쁜 예 (전체 import):**
```tsx
import _ from 'lodash'; // 70KB

export function sum(a: number, b: number) {
  return _.add(a, b);
}
```

**✅ 좋은 예 (필요한 것만):**
```tsx
import add from 'lodash/add'; // 1KB

export function sum(a: number, b: number) {
  return add(a, b);
}
```

### 5.2 대체 라이브러리 사용

**lodash → lodash-es**
```bash
npm uninstall lodash
npm install lodash-es
```

```tsx
// Tree shaking 지원
import { add, map } from 'lodash-es';
```

**moment.js → date-fns**
```bash
npm uninstall moment
npm install date-fns
```

```tsx
// Before (moment.js: 288KB)
import moment from 'moment';
const formatted = moment().format('YYYY-MM-DD');

// After (date-fns: 13KB)
import { format } from 'date-fns';
const formatted = format(new Date(), 'yyyy-MM-dd');
```

### 5.3 불필요한 라이브러리 제거

```bash
# 번들 크기 확인
npm install -g bundle-phobia-cli
bundle-phobia [package-name]
```

**예시:**
```bash
$ bundle-phobia axios
📦 axios: 13.1 KB (minified), 4.5 KB (gzipped)

$ bundle-phobia ky
📦 ky: 7.2 KB (minified), 2.8 KB (gzipped)
```

## 6. 폰트 최적화

### 6.1 next/font 사용

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // FOUT 방지
});

export default function RootLayout({
  children
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ko" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

**장점:**
- 자동 self-hosting
- 레이아웃 시프트 제거
- 캐싱 최적화

### 6.2 Variable Fonts

```tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter', // CSS 변수
  display: 'swap',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko" className={inter.variable}>
      <body>{children}</body>
    </html>
  );
}
```

```css
/* globals.css */
body {
  font-family: var(--font-inter);
}
```

### 6.3 로컬 폰트

```tsx
import localFont from 'next/font/local';

const myFont = localFont({
  src: './my-font.woff2',
  display: 'swap',
});
```

## 7. Import 최적화

### 7.1 배럴 Export 문제

**❌ 나쁜 예:**
```tsx
// components/index.ts (배럴 파일)
export { Header } from './Header';
export { Footer } from './Footer';
export { Sidebar } from './Sidebar'; // 50KB
export { Chart } from './Chart';     // 100KB

// page.tsx
import { Header } from '@/components'; // 전체 로드됨! (150KB+)
```

**✅ 좋은 예:**
```tsx
// 직접 import
import { Header } from '@/components/Header'; // Header만 로드
```

### 7.2 Modularize Imports

```javascript
// next.config.js
module.exports = {
  modularizeImports: {
    '@mui/material': {
      transform: '@mui/material/{{member}}',
    },
    '@mui/icons-material': {
      transform: '@mui/icons-material/{{member}}',
    },
  },
};
```

**효과:**
```tsx
// 이렇게 작성해도
import { Button, TextField } from '@mui/material';

// 자동으로 변환됨
import Button from '@mui/material/Button';
import TextField from '@mui/material/TextField';
```

## 8. 빌드 최적화

### 8.1 Production 빌드

```json
// package.json
{
  "scripts": {
    "build": "next build",
    "start": "next start"
  }
}
```

**최적화 내용:**
- 코드 압축 (Minification)
- Tree Shaking
- Dead Code Elimination

### 8.2 출력 모드

```javascript
// next.config.js
module.exports = {
  output: 'standalone', // Docker 배포 시
};
```

### 8.3 SWC 사용

Next.js 12+ 는 기본적으로 SWC (Rust 기반 컴파일러) 사용

```
Babel vs SWC:
- 빌드 속도: 3배 빠름
- 번들 크기: 동일
```

## 9. 실전 예시

### 9.1 차트 라이브러리 동적 로딩

```tsx
// app/dashboard/page.tsx
'use client';

import { useState } from 'react';
import dynamic from 'next/dynamic';

// Chart.js는 큰 라이브러리 (200KB+)
const LineChart = dynamic(() => import('@/components/charts/LineChart'), {
  loading: () => <div className="h-64 bg-gray-200 animate-pulse" />,
  ssr: false
});

const BarChart = dynamic(() => import('@/components/charts/BarChart'), {
  loading: () => <div className="h-64 bg-gray-200 animate-pulse" />,
  ssr: false
});

export default function DashboardPage() {
  const [activeTab, setActiveTab] = useState<'line' | 'bar'>('line');

  return (
    <div>
      <div className="flex gap-2 mb-4">
        <button onClick={() => setActiveTab('line')}>Line Chart</button>
        <button onClick={() => setActiveTab('bar')}>Bar Chart</button>
      </div>

      {activeTab === 'line' && <LineChart />}
      {activeTab === 'bar' && <BarChart />}
    </div>
  );
}
```

### 9.2 에디터 동적 로딩

```tsx
// app/editor/page.tsx
'use client';

import dynamic from 'next/dynamic';

// Monaco Editor는 매우 큰 라이브러리 (2MB+)
const MonacoEditor = dynamic(
  () => import('@monaco-editor/react'),
  {
    loading: () => (
      <div className="h-screen flex items-center justify-center">
        <div className="text-lg">에디터 로딩 중...</div>
      </div>
    ),
    ssr: false
  }
);

export default function EditorPage() {
  return (
    <MonacoEditor
      height="90vh"
      defaultLanguage="typescript"
      defaultValue="// TypeScript 코드 작성"
    />
  );
}
```

## 10. 실습 문제

### 문제 1: 번들 크기 줄이기
현재 프로젝트의 번들 크기를 분석하고, 50% 이상 줄여보세요.

**힌트:**
- Bundle Analyzer 사용
- 큰 라이브러리 찾기
- 동적 import 적용

### 문제 2: 조건부 폴리필
구형 브라우저에서만 폴리필을 로드하도록 구현하세요.

### 문제 3: PDF 뷰어 최적화
PDF 뷰어를 클릭 시에만 로드하도록 최적화하세요.

## 11. 실습 해답

### 문제 1 해답

**1단계: 번들 분석**
```bash
ANALYZE=true npm run build
```

**2단계: 큰 라이브러리 찾기**
```
발견한 문제:
- lodash (전체 import): 70KB
- moment.js: 288KB
- chart.js (모든 페이지 로드): 200KB
```

**3단계: 최적화 적용**

```tsx
// Before
import _ from 'lodash';
import moment from 'moment';
import { Chart } from 'react-chartjs-2';

// After
import pick from 'lodash/pick'; // 70KB → 2KB
import { format } from 'date-fns'; // 288KB → 13KB
const Chart = dynamic(() => import('react-chartjs-2').then(mod => mod.Chart)); // 지연 로딩
```

**결과:**
```
Before: 620KB
After:  180KB
감소율: 71%
```

### 문제 2 해답

```tsx
// app/components/PolyfillLoader.tsx
'use client';

import { useEffect } from 'react';

export default function PolyfillLoader() {
  useEffect(() => {
    // IntersectionObserver 지원 확인
    if (typeof window !== 'undefined' && !('IntersectionObserver' in window)) {
      import('intersection-observer');
    }

    // Intl.ListFormat 지원 확인
    if (typeof Intl === 'undefined' || !Intl.ListFormat) {
      import('@formatjs/intl-listformat/polyfill');
    }

    // ResizeObserver 지원 확인
    if (typeof window !== 'undefined' && !('ResizeObserver' in window)) {
      import('resize-observer-polyfill');
    }
  }, []);

  return null;
}
```

```tsx
// app/layout.tsx
import PolyfillLoader from '@/components/PolyfillLoader';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <PolyfillLoader />
        {children}
      </body>
    </html>
  );
}
```

**고급 버전 (동적 import 사용):**
```tsx
'use client';

import { useEffect } from 'react';

export default function PolyfillLoader() {
  useEffect(() => {
    async function loadPolyfills() {
      const polyfills = [];

      if (!('IntersectionObserver' in window)) {
        polyfills.push(import('intersection-observer'));
      }

      if (!window.fetch) {
        polyfills.push(import('whatwg-fetch'));
      }

      if (!Promise.allSettled) {
        polyfills.push(
          import('core-js/features/promise/all-settled')
        );
      }

      await Promise.all(polyfills);
      console.log('Polyfills loaded:', polyfills.length);
    }

    loadPolyfills();
  }, []);

  return null;
}
```

### 문제 3 해답

```bash
npm install react-pdf
```

```tsx
// app/documents/page.tsx
'use client';

import { useState } from 'react';
import dynamic from 'next/dynamic';

// PDF 뷰어 동적 로딩 (1.5MB+)
const PDFViewer = dynamic(
  () => import('@/components/PDFViewer'),
  {
    loading: () => (
      <div className="flex items-center justify-center h-96">
        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900" />
        <span className="ml-4">PDF 뷰어 로딩 중...</span>
      </div>
    ),
    ssr: false
  }
);

export default function DocumentsPage() {
  const [showPDF, setShowPDF] = useState(false);
  const [selectedFile, setSelectedFile] = useState<string | null>(null);

  const documents = [
    { id: 1, name: 'Document 1.pdf', url: '/docs/doc1.pdf' },
    { id: 2, name: 'Document 2.pdf', url: '/docs/doc2.pdf' },
    { id: 3, name: 'Document 3.pdf', url: '/docs/doc3.pdf' },
  ];

  function handleViewClick(url: string) {
    setSelectedFile(url);
    setShowPDF(true);
  }

  return (
    <div className="p-4">
      <h1 className="text-2xl font-bold mb-4">문서 목록</h1>

      <div className="space-y-2 mb-8">
        {documents.map(doc => (
          <div
            key={doc.id}
            className="flex items-center justify-between p-4 border rounded"
          >
            <span>{doc.name}</span>
            <button
              onClick={() => handleViewClick(doc.url)}
              className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
            >
              보기
            </button>
          </div>
        ))}
      </div>

      {showPDF && selectedFile && (
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-4 max-w-4xl w-full max-h-[90vh] overflow-auto">
            <div className="flex justify-between items-center mb-4">
              <h2 className="text-xl font-bold">PDF 뷰어</h2>
              <button
                onClick={() => setShowPDF(false)}
                className="px-4 py-2 bg-gray-500 text-white rounded hover:bg-gray-600"
              >
                닫기
              </button>
            </div>

            <PDFViewer fileUrl={selectedFile} />
          </div>
        </div>
      )}
    </div>
  );
}
```

```tsx
// components/PDFViewer.tsx
'use client';

import { useState } from 'react';
import { Document, Page, pdfjs } from 'react-pdf';
import 'react-pdf/dist/esm/Page/AnnotationLayer.css';
import 'react-pdf/dist/esm/Page/TextLayer.css';

// PDF.js worker 설정
pdfjs.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjs.version}/pdf.worker.min.js`;

export default function PDFViewer({ fileUrl }: { fileUrl: string }) {
  const [numPages, setNumPages] = useState<number | null>(null);
  const [pageNumber, setPageNumber] = useState(1);

  return (
    <div className="flex flex-col items-center">
      <Document
        file={fileUrl}
        onLoadSuccess={({ numPages }) => setNumPages(numPages)}
        loading={
          <div className="flex items-center justify-center h-96">
            PDF 로딩 중...
          </div>
        }
      >
        <Page pageNumber={pageNumber} />
      </Document>

      {numPages && (
        <div className="mt-4 flex items-center gap-4">
          <button
            onClick={() => setPageNumber(prev => Math.max(1, prev - 1))}
            disabled={pageNumber === 1}
            className="px-4 py-2 bg-blue-500 text-white rounded disabled:bg-gray-300"
          >
            이전
          </button>

          <span>
            {pageNumber} / {numPages}
          </span>

          <button
            onClick={() => setPageNumber(prev => Math.min(numPages, prev + 1))}
            disabled={pageNumber === numPages}
            className="px-4 py-2 bg-blue-500 text-white rounded disabled:bg-gray-300"
          >
            다음
          </button>
        </div>
      )}
    </div>
  );
}
```

**최적화 결과:**
```
Before (PDF 라이브러리가 모든 페이지에 포함):
- 첫 로드: 1.8MB
- 모든 페이지 무거움

After (클릭 시에만 로드):
- 첫 로드: 200KB
- 문서 페이지만 1.8MB
- 개선: 90% 감소
```

## 핵심 요약

1. **Bundle Analyzer**: 번들 크기 분석 도구
2. **Dynamic Import**: 필요할 때만 로드 (Code Splitting)
3. **라이브러리 최적화**: Tree Shaking, 경량 대체
4. **폰트 최적화**: next/font로 자동 최적화
5. **조건부 로딩**: 인터랙션/뷰포트 기반 로딩

[← Chapter 21: 이미지 최적화](./chapter21-image-optimization.md) | [Chapter 23: 캐싱 전략 →](./chapter23-caching-strategies.md)