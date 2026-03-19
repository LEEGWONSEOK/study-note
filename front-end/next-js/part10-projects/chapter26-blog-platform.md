# Chapter 26: 블로그 플랫폼 프로젝트

## 1. 프로젝트 개요

### 1.1 기능 요구사항

```
✅ 사용자 인증 (NextAuth.js)
✅ 마크다운 포스트 작성/수정/삭제
✅ 태그 및 카테고리
✅ 댓글 시스템
✅ 검색 기능
✅ RSS 피드
✅ SEO 최적화
```

### 1.2 기술 스택

```
Frontend:   Next.js 14 (App Router)
Styling:    Tailwind CSS
Database:   PostgreSQL + Prisma
Auth:       NextAuth.js
Editor:     react-markdown + react-simplemde-editor
Deployment: Vercel
```

## 2. 프로젝트 설정

### 2.1 초기 설정

```bash
npx create-next-app@latest blog-platform
cd blog-platform

# 의존성 설치
npm install prisma @prisma/client
npm install next-auth
npm install react-markdown react-simplemde-editor easymde
npm install @tailwindcss/typography
npm install date-fns
npm install zod
```

### 2.2 데이터베이스 스키마

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
  posts         Post[]
  comments      Comment[]
  createdAt     DateTime  @default(now())
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Post {
  id          String    @id @default(cuid())
  title       String
  slug        String    @unique
  content     String    @db.Text
  excerpt     String?
  published   Boolean   @default(false)
  authorId    String
  author      User      @relation(fields: [authorId], references: [id])
  tags        Tag[]
  comments    Comment[]
  viewCount   Int       @default(0)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}

model Comment {
  id        String   @id @default(cuid())
  content   String   @db.Text
  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now())
}
```

```bash
npx prisma migrate dev --name init
npx prisma generate
```

### 2.3 환경 변수

```bash
# .env.local
DATABASE_URL="postgresql://user:password@localhost:5432/blog"
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-secret-here"

GOOGLE_CLIENT_ID="your-google-client-id"
GOOGLE_CLIENT_SECRET="your-google-secret"
```

## 3. 인증 구현

### 3.1 NextAuth 설정

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';

const handler = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    session: async ({ session, user }) => {
      if (session?.user) {
        session.user.id = user.id;
      }
      return session;
    },
  },
  pages: {
    signIn: '/auth/signin',
  },
});

export { handler as GET, handler as POST };
```

### 3.2 Prisma 클라이언트

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: ['query'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

## 4. 포스트 CRUD

### 4.1 포스트 목록

```tsx
// app/page.tsx
import Link from 'next/link';
import { prisma } from '@/lib/prisma';
import { format } from 'date-fns';

export default async function HomePage() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    include: {
      author: {
        select: { name: true, image: true }
      },
      tags: true,
      _count: {
        select: { comments: true }
      }
    },
    orderBy: { createdAt: 'desc' },
    take: 10
  });

  return (
    <div className="max-w-4xl mx-auto p-4">
      <h1 className="text-4xl font-bold mb-8">최신 포스트</h1>

      <div className="space-y-6">
        {posts.map(post => (
          <article key={post.id} className="border-b pb-6">
            <Link href={`/posts/${post.slug}`}>
              <h2 className="text-2xl font-bold hover:text-blue-600 mb-2">
                {post.title}
              </h2>
            </Link>

            <div className="flex items-center gap-4 text-sm text-gray-600 mb-2">
              <span>{post.author.name}</span>
              <span>{format(new Date(post.createdAt), 'yyyy-MM-dd')}</span>
              <span>{post._count.comments}개 댓글</span>
            </div>

            {post.excerpt && (
              <p className="text-gray-700 mb-2">{post.excerpt}</p>
            )}

            <div className="flex gap-2">
              {post.tags.map(tag => (
                <Link
                  key={tag.id}
                  href={`/tags/${tag.name}`}
                  className="text-xs bg-gray-200 px-2 py-1 rounded"
                >
                  #{tag.name}
                </Link>
              ))}
            </div>
          </article>
        ))}
      </div>
    </div>
  );
}
```

### 4.2 포스트 상세

```tsx
// app/posts/[slug]/page.tsx
import { notFound } from 'next/navigation';
import { prisma } from '@/lib/prisma';
import ReactMarkdown from 'react-markdown';
import { format } from 'date-fns';
import Comments from '@/app/components/Comments';

export async function generateMetadata({
  params
}: {
  params: { slug: string }
}) {
  const post = await prisma.post.findUnique({
    where: { slug: params.slug }
  });

  if (!post) return {};

  return {
    title: post.title,
    description: post.excerpt,
  };
}

export default async function PostPage({
  params
}: {
  params: { slug: string }
}) {
  const post = await prisma.post.findUnique({
    where: { slug: params.slug },
    include: {
      author: true,
      tags: true,
      comments: {
        include: { author: true },
        orderBy: { createdAt: 'desc' }
      }
    }
  });

  if (!post) {
    notFound();
  }

  // 조회수 증가
  await prisma.post.update({
    where: { id: post.id },
    data: { viewCount: { increment: 1 } }
  });

  return (
    <article className="max-w-4xl mx-auto p-4">
      <header className="mb-8">
        <h1 className="text-4xl font-bold mb-4">{post.title}</h1>

        <div className="flex items-center gap-4 text-gray-600 mb-4">
          <img
            src={post.author.image || '/default-avatar.png'}
            alt={post.author.name || 'Author'}
            className="w-10 h-10 rounded-full"
          />
          <div>
            <div className="font-medium">{post.author.name}</div>
            <div className="text-sm">
              {format(new Date(post.createdAt), 'yyyy-MM-dd')} · 조회 {post.viewCount}
            </div>
          </div>
        </div>

        <div className="flex gap-2">
          {post.tags.map(tag => (
            <span
              key={tag.id}
              className="text-sm bg-gray-200 px-3 py-1 rounded-full"
            >
              #{tag.name}
            </span>
          ))}
        </div>
      </header>

      <div className="prose prose-lg max-w-none mb-12">
        <ReactMarkdown>{post.content}</ReactMarkdown>
      </div>

      <Comments postId={post.id} comments={post.comments} />
    </article>
  );
}
```

### 4.3 포스트 작성

```tsx
// app/posts/new/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import dynamic from 'next/dynamic';
import { createPost } from '@/app/actions/posts';

const SimpleMDE = dynamic(() => import('react-simplemde-editor'), {
  ssr: false
});

import 'easymde/dist/easymde.min.css';

export default function NewPostPage() {
  const router = useRouter();
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [tags, setTags] = useState('');
  const [published, setPublished] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    const formData = new FormData();
    formData.append('title', title);
    formData.append('content', content);
    formData.append('tags', tags);
    formData.append('published', published.toString());

    const result = await createPost(formData);

    if (result.success) {
      router.push(`/posts/${result.slug}`);
    }
  }

  return (
    <div className="max-w-4xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">새 포스트 작성</h1>

      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label htmlFor="title" className="block mb-2 font-medium">
            제목
          </label>
          <input
            type="text"
            id="title"
            value={title}
            onChange={e => setTitle(e.target.value)}
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div>
          <label className="block mb-2 font-medium">내용</label>
          <SimpleMDE
            value={content}
            onChange={setContent}
          />
        </div>

        <div>
          <label htmlFor="tags" className="block mb-2 font-medium">
            태그 (쉼표로 구분)
          </label>
          <input
            type="text"
            id="tags"
            value={tags}
            onChange={e => setTags(e.target.value)}
            placeholder="next.js, react, typescript"
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div className="flex items-center gap-2">
          <input
            type="checkbox"
            id="published"
            checked={published}
            onChange={e => setPublished(e.target.checked)}
          />
          <label htmlFor="published">바로 게시</label>
        </div>

        <button
          type="submit"
          className="px-6 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
        >
          게시하기
        </button>
      </form>
    </div>
  );
}
```

### 4.4 Server Actions

```typescript
// app/actions/posts.ts
'use server';

import { getServerSession } from 'next-auth';
import { prisma } from '@/lib/prisma';
import { revalidatePath } from 'next/cache';

function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/\s+/g, '-')
    .substring(0, 100);
}

export async function createPost(formData: FormData) {
  const session = await getServerSession();

  if (!session?.user?.email) {
    throw new Error('인증이 필요합니다');
  }

  const user = await prisma.user.findUnique({
    where: { email: session.user.email }
  });

  if (!user) {
    throw new Error('사용자를 찾을 수 없습니다');
  }

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  const tagsString = formData.get('tags') as string;
  const published = formData.get('published') === 'true';

  const slug = generateSlug(title);
  const excerpt = content.substring(0, 200);

  // 태그 처리
  const tagNames = tagsString
    .split(',')
    .map(tag => tag.trim())
    .filter(Boolean);

  const tags = await Promise.all(
    tagNames.map(name =>
      prisma.tag.upsert({
        where: { name },
        create: { name },
        update: {}
      })
    )
  );

  const post = await prisma.post.create({
    data: {
      title,
      slug,
      content,
      excerpt,
      published,
      authorId: user.id,
      tags: {
        connect: tags.map(tag => ({ id: tag.id }))
      }
    }
  });

  revalidatePath('/');
  revalidatePath('/posts');

  return { success: true, slug: post.slug };
}

export async function updatePost(id: string, formData: FormData) {
  const session = await getServerSession();

  if (!session?.user?.email) {
    throw new Error('인증이 필요합니다');
  }

  const post = await prisma.post.findUnique({
    where: { id },
    include: { author: true }
  });

  if (!post || post.author.email !== session.user.email) {
    throw new Error('권한이 없습니다');
  }

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  const published = formData.get('published') === 'true';

  await prisma.post.update({
    where: { id },
    data: {
      title,
      content,
      published,
      updatedAt: new Date()
    }
  });

  revalidatePath(`/posts/${post.slug}`);

  return { success: true };
}

export async function deletePost(id: string) {
  const session = await getServerSession();

  if (!session?.user?.email) {
    throw new Error('인증이 필요합니다');
  }

  const post = await prisma.post.findUnique({
    where: { id },
    include: { author: true }
  });

  if (!post || post.author.email !== session.user.email) {
    throw new Error('권한이 없습니다');
  }

  await prisma.post.delete({
    where: { id }
  });

  revalidatePath('/');
  revalidatePath('/posts');

  return { success: true };
}
```

## 5. 댓글 시스템

### 5.1 댓글 컴포넌트

```tsx
// app/components/Comments.tsx
'use client';

import { useState } from 'react';
import { useSession } from 'next-auth/react';
import { format } from 'date-fns';
import { createComment } from '@/app/actions/comments';

export default function Comments({
  postId,
  comments: initialComments
}: {
  postId: string;
  comments: any[];
}) {
  const { data: session } = useSession();
  const [comments, setComments] = useState(initialComments);
  const [content, setContent] = useState('');

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    if (!content.trim()) return;

    const formData = new FormData();
    formData.append('postId', postId);
    formData.append('content', content);

    const result = await createComment(formData);

    if (result.success) {
      setComments([result.comment, ...comments]);
      setContent('');
    }
  }

  return (
    <div className="mt-12 border-t pt-8">
      <h2 className="text-2xl font-bold mb-6">
        댓글 {comments.length}
      </h2>

      {session ? (
        <form onSubmit={handleSubmit} className="mb-8">
          <textarea
            value={content}
            onChange={e => setContent(e.target.value)}
            placeholder="댓글을 작성하세요..."
            className="w-full px-4 py-2 border rounded mb-2"
            rows={3}
          />
          <button
            type="submit"
            className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
          >
            댓글 작성
          </button>
        </form>
      ) : (
        <p className="mb-8 text-gray-600">
          댓글을 작성하려면 로그인하세요.
        </p>
      )}

      <div className="space-y-4">
        {comments.map(comment => (
          <div key={comment.id} className="border-b pb-4">
            <div className="flex items-center gap-2 mb-2">
              <img
                src={comment.author.image || '/default-avatar.png'}
                alt={comment.author.name || 'User'}
                className="w-8 h-8 rounded-full"
              />
              <div>
                <div className="font-medium">{comment.author.name}</div>
                <div className="text-xs text-gray-500">
                  {format(new Date(comment.createdAt), 'yyyy-MM-dd HH:mm')}
                </div>
              </div>
            </div>
            <p className="text-gray-700">{comment.content}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 5.2 댓글 Server Action

```typescript
// app/actions/comments.ts
'use server';

import { getServerSession } from 'next-auth';
import { prisma } from '@/lib/prisma';
import { revalidatePath } from 'next/cache';

export async function createComment(formData: FormData) {
  const session = await getServerSession();

  if (!session?.user?.email) {
    throw new Error('인증이 필요합니다');
  }

  const user = await prisma.user.findUnique({
    where: { email: session.user.email }
  });

  if (!user) {
    throw new Error('사용자를 찾을 수 없습니다');
  }

  const postId = formData.get('postId') as string;
  const content = formData.get('content') as string;

  const comment = await prisma.comment.create({
    data: {
      content,
      postId,
      authorId: user.id
    },
    include: {
      author: {
        select: {
          name: true,
          image: true
        }
      }
    }
  });

  const post = await prisma.post.findUnique({
    where: { id: postId },
    select: { slug: true }
  });

  revalidatePath(`/posts/${post?.slug}`);

  return { success: true, comment };
}
```

## 6. 검색 기능

### 6.1 검색 페이지

```tsx
// app/search/page.tsx
import { prisma } from '@/lib/prisma';
import Link from 'next/link';

export default async function SearchPage({
  searchParams
}: {
  searchParams: { q: string }
}) {
  const query = searchParams.q || '';

  const posts = query
    ? await prisma.post.findMany({
        where: {
          published: true,
          OR: [
            { title: { contains: query, mode: 'insensitive' } },
            { content: { contains: query, mode: 'insensitive' } }
          ]
        },
        include: {
          author: { select: { name: true } },
          tags: true
        },
        take: 20
      })
    : [];

  return (
    <div className="max-w-4xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">
        검색 결과: "{query}"
      </h1>

      {posts.length > 0 ? (
        <div className="space-y-4">
          {posts.map(post => (
            <article key={post.id} className="border-b pb-4">
              <Link href={`/posts/${post.slug}`}>
                <h2 className="text-xl font-bold hover:text-blue-600">
                  {post.title}
                </h2>
              </Link>
              <p className="text-sm text-gray-600">
                {post.author.name} · {post.tags.map(t => `#${t.name}`).join(' ')}
              </p>
            </article>
          ))}
        </div>
      ) : (
        <p className="text-gray-600">검색 결과가 없습니다.</p>
      )}
    </div>
  );
}
```

### 6.2 검색 바 컴포넌트

```tsx
// app/components/SearchBar.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function SearchBar() {
  const router = useRouter();
  const [query, setQuery] = useState('');

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (query.trim()) {
      router.push(`/search?q=${encodeURIComponent(query)}`);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="flex gap-2">
      <input
        type="text"
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="검색..."
        className="px-4 py-2 border rounded"
      />
      <button
        type="submit"
        className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
      >
        검색
      </button>
    </form>
  );
}
```

## 7. RSS 피드

```typescript
// app/feed.xml/route.ts
import { prisma } from '@/lib/prisma';

export async function GET() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    include: { author: true },
    orderBy: { createdAt: 'desc' },
    take: 20
  });

  const rss = `<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>My Blog</title>
    <link>https://yourdomain.com</link>
    <description>A blog about web development</description>
    <language>ko</language>
    <atom:link href="https://yourdomain.com/feed.xml" rel="self" type="application/rss+xml" />
    ${posts
      .map(
        post => `
    <item>
      <title>${escapeXml(post.title)}</title>
      <link>https://yourdomain.com/posts/${post.slug}</link>
      <description>${escapeXml(post.excerpt || '')}</description>
      <author>${escapeXml(post.author.name || '')}</author>
      <pubDate>${new Date(post.createdAt).toUTCString()}</pubDate>
      <guid>https://yourdomain.com/posts/${post.slug}</guid>
    </item>
    `
      )
      .join('')}
  </channel>
</rss>`;

  return new Response(rss, {
    headers: {
      'Content-Type': 'application/xml; charset=utf-8'
    }
  });
}

function escapeXml(unsafe: string): string {
  return unsafe.replace(/[<>&'"]/g, c => {
    switch (c) {
      case '<':
        return '&lt;';
      case '>':
        return '&gt;';
      case '&':
        return '&amp;';
      case "'":
        return '&apos;';
      case '"':
        return '&quot;';
      default:
        return c;
    }
  });
}
```

## 핵심 요약

1. **NextAuth.js**: 간편한 인증 구현
2. **Prisma**: 타입 안전 ORM
3. **Server Actions**: 서버 로직 간소화
4. **Markdown**: react-markdown으로 렌더링
5. **SEO**: generateMetadata + RSS 피드

[← Chapter 25: Docker 배포](../part09-deployment/chapter25-docker-deployment.md) | [Chapter 27: E-Commerce →](./chapter27-ecommerce.md)