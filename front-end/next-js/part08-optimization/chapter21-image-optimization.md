# Chapter 21: 이미지 최적화

## 1. 이미지 최적화의 중요성

### 1.1 왜 최적화가 필요한가?

**문제점:**
```
- 대용량 이미지 → 느린 로딩 속도
- 다양한 화면 크기 → 불필요한 대역폭 낭비
- 레이아웃 시프트 → 사용자 경험 저하
```

**해결책:**
```
- 자동 포맷 변환 (WebP, AVIF)
- 반응형 이미지 제공
- 지연 로딩 (Lazy Loading)
- 이미지 크기 최적화
```

## 2. Next.js Image 컴포넌트

### 2.1 기본 사용법

**기존 HTML img 태그**
```tsx
// ❌ 최적화 안 됨
<img src="/photo.jpg" alt="Photo" />
```

**Next.js Image 컴포넌트**
```tsx
// ✅ 자동 최적화
import Image from 'next/image';

export default function Page() {
  return (
    <Image
      src="/photo.jpg"
      alt="Photo"
      width={800}
      height={600}
    />
  );
}
```

### 2.2 주요 기능

**1. 자동 포맷 변환**
```tsx
// 브라우저가 지원하면 WebP/AVIF로 자동 변환
<Image src="/photo.jpg" alt="Photo" width={800} height={600} />
// → 실제 제공: photo.webp (용량 30% 감소)
```

**2. 반응형 크기**
```tsx
// 화면 크기에 맞는 이미지 자동 제공
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  sizes="(max-width: 768px) 100vw, 50vw"
/>
```

**3. 지연 로딩**
```tsx
// 뷰포트에 들어올 때만 로드
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  loading="lazy" // 기본값
/>
```

### 2.3 필수 Props

```tsx
import Image from 'next/image';

<Image
  src="/photo.jpg"        // 이미지 경로
  alt="사진 설명"          // 접근성 (필수!)
  width={800}             // 너비 (픽셀)
  height={600}            // 높이 (픽셀)
/>
```

## 3. 정적 이미지 (Static Import)

### 3.1 로컬 이미지

```tsx
import Image from 'next/image';
import profilePic from '@/public/me.jpg';

export default function Page() {
  return (
    <Image
      src={profilePic}
      alt="Profile"
      // width, height 자동 계산
    />
  );
}
```

**장점:**
- width, height 자동 설정
- 빌드 타임에 최적화
- 블러 플레이스홀더 자동 생성

### 3.2 자동 블러 플레이스홀더

```tsx
import profilePic from '@/public/me.jpg';

<Image
  src={profilePic}
  alt="Profile"
  placeholder="blur" // 자동 블러 효과
/>
```

## 4. 외부 이미지

### 4.1 외부 URL 설정

```typescript
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'example.com',
        port: '',
        pathname: '/images/**',
      },
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
      }
    ],
  },
};
```

### 4.2 외부 이미지 사용

```tsx
<Image
  src="https://example.com/images/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
/>
```

### 4.3 커스텀 블러 플레이스홀더

```tsx
<Image
  src="https://example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQ..." // Base64 인코딩
/>
```

**블러 데이터 생성:**
```typescript
// lib/blur-image.ts
import { getPlaiceholder } from 'plaiceholder';

export async function getBlurData(imageUrl: string) {
  const buffer = await fetch(imageUrl).then(res => res.arrayBuffer());

  const { base64 } = await getPlaiceholder(Buffer.from(buffer));

  return base64;
}
```

```bash
npm install plaiceholder
```

## 5. 반응형 이미지

### 5.1 fill 속성

```tsx
// 부모 요소 크기에 맞춤
<div style={{ position: 'relative', width: '100%', height: '400px' }}>
  <Image
    src="/photo.jpg"
    alt="Photo"
    fill
    style={{ objectFit: 'cover' }}
  />
</div>
```

### 5.2 sizes 속성

```tsx
<Image
  src="/photo.jpg"
  alt="Photo"
  fill
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
/>
```

**sizes 해석:**
```
- 768px 이하: 뷰포트 너비의 100%
- 768px ~ 1200px: 뷰포트 너비의 50%
- 1200px 이상: 뷰포트 너비의 33%
```

### 5.3 실전 예시

```tsx
// app/components/HeroImage.tsx
import Image from 'next/image';

export default function HeroImage() {
  return (
    <div className="relative w-full h-[400px] md:h-[600px]">
      <Image
        src="/hero.jpg"
        alt="Hero Image"
        fill
        priority // 우선 로드
        sizes="100vw"
        style={{ objectFit: 'cover' }}
      />
    </div>
  );
}
```

## 6. 최적화 옵션

### 6.1 우선순위 (Priority)

```tsx
// 최우선 로드 (LCP 이미지에 사용)
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority
/>
```

**사용 시기:**
- 메인 히어로 이미지
- Above the fold 콘텐츠
- 페이지당 1-2개만 사용

### 6.2 로딩 전략

```tsx
// Lazy loading (기본값)
<Image src="/photo.jpg" alt="Photo" width={800} height={600} />

// Eager loading (즉시 로드)
<Image
  src="/logo.png"
  alt="Logo"
  width={100}
  height={100}
  loading="eager"
/>
```

### 6.3 품질 (Quality)

```tsx
// 품질 조정 (기본값: 75)
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  quality={90} // 1-100
/>
```

## 7. 이미지 갤러리 구현

### 7.1 그리드 레이아웃

```tsx
// app/gallery/page.tsx
import Image from 'next/image';

const images = [
  { id: 1, src: '/gallery/1.jpg', alt: 'Photo 1' },
  { id: 2, src: '/gallery/2.jpg', alt: 'Photo 2' },
  { id: 3, src: '/gallery/3.jpg', alt: 'Photo 3' },
  // ...
];

export default function GalleryPage() {
  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-4 p-4">
      {images.map(image => (
        <div key={image.id} className="relative aspect-square">
          <Image
            src={image.src}
            alt={image.alt}
            fill
            sizes="(max-width: 768px) 100vw, 33vw"
            style={{ objectFit: 'cover' }}
            className="rounded-lg"
          />
        </div>
      ))}
    </div>
  );
}
```

### 7.2 모달 뷰어

```tsx
// app/components/ImageModal.tsx
'use client';

import { useState } from 'react';
import Image from 'next/image';

export default function ImageModal({
  src,
  alt
}: {
  src: string;
  alt: string;
}) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      {/* 썸네일 */}
      <div
        className="relative aspect-square cursor-pointer"
        onClick={() => setIsOpen(true)}
      >
        <Image
          src={src}
          alt={alt}
          fill
          sizes="(max-width: 768px) 100vw, 33vw"
          style={{ objectFit: 'cover' }}
        />
      </div>

      {/* 모달 */}
      {isOpen && (
        <div
          className="fixed inset-0 bg-black/80 flex items-center justify-center z-50"
          onClick={() => setIsOpen(false)}
        >
          <div className="relative w-full h-full max-w-4xl max-h-4xl p-4">
            <Image
              src={src}
              alt={alt}
              fill
              sizes="100vw"
              style={{ objectFit: 'contain' }}
            />
          </div>
        </div>
      )}
    </>
  );
}
```

## 8. 이미지 업로드 처리

### 8.1 파일 업로드

```tsx
// app/upload/page.tsx
'use client';

import { useState } from 'react';
import Image from 'next/image';

export default function UploadPage() {
  const [preview, setPreview] = useState<string | null>(null);

  function handleFileChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];

    if (file) {
      const reader = new FileReader();

      reader.onloadend = () => {
        setPreview(reader.result as string);
      };

      reader.readAsDataURL(file);
    }
  }

  return (
    <div>
      <input
        type="file"
        accept="image/*"
        onChange={handleFileChange}
      />

      {preview && (
        <div className="relative w-full h-64 mt-4">
          <Image
            src={preview}
            alt="Preview"
            fill
            style={{ objectFit: 'contain' }}
          />
        </div>
      )}
    </div>
  );
}
```

### 8.2 서버 업로드

```typescript
// app/actions/upload.ts
'use server';

import { writeFile } from 'fs/promises';
import { join } from 'path';

export async function uploadImage(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file) {
    throw new Error('파일이 없습니다');
  }

  // 파일 검증
  if (!file.type.startsWith('image/')) {
    throw new Error('이미지 파일만 업로드 가능합니다');
  }

  const maxSize = 5 * 1024 * 1024; // 5MB
  if (file.size > maxSize) {
    throw new Error('파일 크기는 5MB 이하여야 합니다');
  }

  // 파일 저장
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  const filename = `${Date.now()}-${file.name}`;
  const path = join(process.cwd(), 'public', 'uploads', filename);

  await writeFile(path, buffer);

  return `/uploads/${filename}`;
}
```

## 9. 성능 측정

### 9.1 Lighthouse 지표

**측정 항목:**
```
- LCP (Largest Contentful Paint): 가장 큰 콘텐츠 렌더링 시간
- CLS (Cumulative Layout Shift): 레이아웃 이동 누적
- FID (First Input Delay): 첫 입력 지연
```

**이미지 최적화 효과:**
```
Before: LCP 4.5s, CLS 0.25
After:  LCP 1.2s, CLS 0.01
```

### 9.2 DevTools 분석

```bash
# Chrome DevTools → Network 탭
1. Disable cache 체크
2. 페이지 새로고침
3. 이미지 요청 확인:
   - 파일 크기
   - 로딩 시간
   - 포맷 (WebP, AVIF)
```

## 10. 실습 문제

### 문제 1: 프로필 사진 업로드
사용자가 프로필 사진을 업로드하고 미리보기를 보여주는 기능을 구현하세요.

**요구사항:**
- 드래그 앤 드롭 지원
- 이미지 크롭 기능
- 서버에 업로드

### 문제 2: 무한 스크롤 갤러리
스크롤 시 자동으로 이미지를 로드하는 갤러리를 만드세요.

### 문제 3: 이미지 압축
업로드된 이미지를 서버에서 자동으로 압축하는 기능을 구현하세요.

## 11. 실습 해답

### 문제 1 해답

```bash
npm install react-image-crop
```

```tsx
// app/profile/edit/page.tsx
'use client';

import { useState, useRef } from 'react';
import Image from 'next/image';
import ReactCrop, { Crop } from 'react-image-crop';
import 'react-image-crop/dist/ReactCrop.css';
import { uploadProfileImage } from '@/app/actions/profile';

export default function EditProfilePage() {
  const [src, setSrc] = useState<string | null>(null);
  const [crop, setCrop] = useState<Crop>();
  const [preview, setPreview] = useState<string | null>(null);
  const imgRef = useRef<HTMLImageElement>(null);

  function handleFileChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];

    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => {
        setSrc(reader.result as string);
      };
      reader.readAsDataURL(file);
    }
  }

  function handleDrop(e: React.DragEvent<HTMLDivElement>) {
    e.preventDefault();
    const file = e.dataTransfer.files[0];

    if (file && file.type.startsWith('image/')) {
      const reader = new FileReader();
      reader.onloadend = () => {
        setSrc(reader.result as string);
      };
      reader.readAsDataURL(file);
    }
  }

  async function handleComplete(crop: Crop) {
    if (!imgRef.current || !crop.width || !crop.height) return;

    const canvas = document.createElement('canvas');
    const scaleX = imgRef.current.naturalWidth / imgRef.current.width;
    const scaleY = imgRef.current.naturalHeight / imgRef.current.height;

    canvas.width = crop.width;
    canvas.height = crop.height;

    const ctx = canvas.getContext('2d');

    if (ctx) {
      ctx.drawImage(
        imgRef.current,
        crop.x * scaleX,
        crop.y * scaleY,
        crop.width * scaleX,
        crop.height * scaleY,
        0,
        0,
        crop.width,
        crop.height
      );

      canvas.toBlob(blob => {
        if (blob) {
          setPreview(URL.createObjectURL(blob));
        }
      });
    }
  }

  async function handleUpload() {
    if (!preview) return;

    const blob = await fetch(preview).then(r => r.blob());
    const formData = new FormData();
    formData.append('file', blob, 'profile.jpg');

    const url = await uploadProfileImage(formData);
    console.log('Uploaded:', url);
  }

  return (
    <div className="max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">프로필 사진 변경</h1>

      {/* 드래그 앤 드롭 영역 */}
      <div
        className="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center mb-4"
        onDrop={handleDrop}
        onDragOver={e => e.preventDefault()}
      >
        <input
          type="file"
          accept="image/*"
          onChange={handleFileChange}
          className="mb-4"
        />
        <p>또는 이미지를 드래그하세요</p>
      </div>

      {/* 크롭 */}
      {src && (
        <div className="mb-4">
          <ReactCrop
            crop={crop}
            onChange={c => setCrop(c)}
            onComplete={handleComplete}
            aspect={1} // 1:1 비율
          >
            <img
              ref={imgRef}
              src={src}
              alt="Crop"
              style={{ maxWidth: '100%' }}
            />
          </ReactCrop>
        </div>
      )}

      {/* 미리보기 */}
      {preview && (
        <div className="mb-4">
          <h2 className="text-xl font-bold mb-2">미리보기</h2>
          <div className="relative w-32 h-32">
            <Image
              src={preview}
              alt="Preview"
              fill
              style={{ objectFit: 'cover' }}
              className="rounded-full"
            />
          </div>
        </div>
      )}

      {/* 업로드 버튼 */}
      {preview && (
        <button
          onClick={handleUpload}
          className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
        >
          업로드
        </button>
      )}
    </div>
  );
}
```

### 문제 2 해답

```tsx
// app/gallery/page.tsx
'use client';

import { useState, useEffect, useRef } from 'react';
import Image from 'next/image';

export default function InfiniteGalleryPage() {
  const [images, setImages] = useState<any[]>([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const observerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    loadMore();
  }, [page]);

  useEffect(() => {
    const observer = new IntersectionObserver(
      entries => {
        if (entries[0].isIntersecting && !loading && hasMore) {
          setPage(prev => prev + 1);
        }
      },
      { threshold: 0.5 }
    );

    if (observerRef.current) {
      observer.observe(observerRef.current);
    }

    return () => observer.disconnect();
  }, [loading, hasMore]);

  async function loadMore() {
    if (loading) return;

    setLoading(true);

    try {
      const res = await fetch(`/api/images?page=${page}&limit=12`);
      const newImages = await res.json();

      if (newImages.length === 0) {
        setHasMore(false);
      } else {
        setImages(prev => [...prev, ...newImages]);
      }
    } catch (error) {
      console.error('Failed to load images:', error);
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="container mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">무한 스크롤 갤러리</h1>

      <div className="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-4">
        {images.map((image, index) => (
          <div key={`${image.id}-${index}`} className="relative aspect-square">
            <Image
              src={image.url}
              alt={image.title}
              fill
              sizes="(max-width: 768px) 100vw, (max-width: 1024px) 33vw, 25vw"
              style={{ objectFit: 'cover' }}
              className="rounded-lg"
            />
          </div>
        ))}
      </div>

      {/* 로딩 인디케이터 */}
      <div ref={observerRef} className="py-8 text-center">
        {loading && <p>로딩 중...</p>}
        {!hasMore && <p>모든 이미지를 불러왔습니다</p>}
      </div>
    </div>
  );
}
```

**API Route**
```typescript
// app/api/images/route.ts
import { NextRequest, NextResponse } from 'next/server';

const mockImages = Array.from({ length: 100 }, (_, i) => ({
  id: i + 1,
  url: `https://picsum.photos/400/400?random=${i}`,
  title: `Image ${i + 1}`
}));

export async function GET(req: NextRequest) {
  const searchParams = req.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '12');

  const start = (page - 1) * limit;
  const end = start + limit;

  const images = mockImages.slice(start, end);

  // 약간의 지연 (실제 API 시뮬레이션)
  await new Promise(resolve => setTimeout(resolve, 500));

  return NextResponse.json(images);
}
```

### 문제 3 해답

```bash
npm install sharp
```

```typescript
// app/actions/upload.ts
'use server';

import sharp from 'sharp';
import { writeFile } from 'fs/promises';
import { join } from 'path';

export async function uploadAndCompressImage(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file) {
    throw new Error('파일이 없습니다');
  }

  // 파일 검증
  if (!file.type.startsWith('image/')) {
    throw new Error('이미지 파일만 업로드 가능합니다');
  }

  // 파일 읽기
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  // 이미지 정보 확인
  const metadata = await sharp(buffer).metadata();

  // 압축 및 리사이징
  let processedBuffer = buffer;

  // 큰 이미지는 리사이징
  if (metadata.width && metadata.width > 1920) {
    processedBuffer = await sharp(buffer)
      .resize(1920, null, {
        withoutEnlargement: true,
        fit: 'inside'
      })
      .jpeg({ quality: 85 })
      .toBuffer();
  } else {
    // 품질 압축만
    processedBuffer = await sharp(buffer)
      .jpeg({ quality: 85 })
      .toBuffer();
  }

  // 썸네일 생성
  const thumbnailBuffer = await sharp(buffer)
    .resize(300, 300, {
      fit: 'cover',
      position: 'center'
    })
    .jpeg({ quality: 80 })
    .toBuffer();

  // 파일 저장
  const filename = `${Date.now()}-${file.name.replace(/\.[^/.]+$/, '')}.jpg`;
  const thumbnailFilename = `${Date.now()}-${file.name.replace(/\.[^/.]+$/, '')}-thumb.jpg`;

  const uploadDir = join(process.cwd(), 'public', 'uploads');
  await writeFile(join(uploadDir, filename), processedBuffer);
  await writeFile(join(uploadDir, thumbnailFilename), thumbnailBuffer);

  return {
    url: `/uploads/${filename}`,
    thumbnail: `/uploads/${thumbnailFilename}`,
    originalSize: buffer.length,
    compressedSize: processedBuffer.length,
    compressionRatio: ((1 - processedBuffer.length / buffer.length) * 100).toFixed(2) + '%'
  };
}
```

**사용 예시**
```tsx
// app/upload/page.tsx
'use client';

import { useState } from 'react';
import { uploadAndCompressImage } from '@/app/actions/upload';

export default function UploadPage() {
  const [result, setResult] = useState<any>(null);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();

    const formData = new FormData(e.currentTarget);
    const result = await uploadAndCompressImage(formData);

    setResult(result);
  }

  return (
    <div className="max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">이미지 압축 업로드</h1>

      <form onSubmit={handleSubmit}>
        <input type="file" name="file" accept="image/*" required />
        <button
          type="submit"
          className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
        >
          업로드
        </button>
      </form>

      {result && (
        <div className="mt-4">
          <h2 className="text-xl font-bold mb-2">업로드 완료!</h2>
          <p>원본 크기: {(result.originalSize / 1024).toFixed(2)} KB</p>
          <p>압축 크기: {(result.compressedSize / 1024).toFixed(2)} KB</p>
          <p>압축률: {result.compressionRatio}</p>

          <div className="mt-4 grid grid-cols-2 gap-4">
            <div>
              <h3 className="font-bold mb-2">원본</h3>
              <img src={result.url} alt="Original" />
            </div>
            <div>
              <h3 className="font-bold mb-2">썸네일</h3>
              <img src={result.thumbnail} alt="Thumbnail" />
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
```

## 핵심 요약

1. **Next.js Image**: 자동 최적화 (포맷 변환, 반응형, 지연 로딩)
2. **정적 이미지**: Static Import로 width/height 자동 설정
3. **외부 이미지**: `remotePatterns`로 도메인 허용
4. **반응형**: `fill` + `sizes`로 화면 크기별 최적화
5. **성능**: `priority` (LCP), `loading` (lazy/eager), `quality` 조절

[← Chapter 20: 권한 관리](../part07-authentication/chapter20-authorization.md) | [Chapter 22: 번들 최적화 →](./chapter22-bundle-optimization.md)