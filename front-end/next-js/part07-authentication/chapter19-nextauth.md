# Chapter 19: NextAuth.js

## 1. NextAuth.js란?

### 1.1 개념

NextAuth.js는 Next.js를 위한 완전한 인증 솔루션입니다.

**장점:**
- OAuth 제공자 통합 (Google, GitHub, Facebook 등)
- 이메일/비밀번호 인증
- JWT와 세션 지원
- 타입스크립트 완벽 지원
- 보안 모범 사례 내장

### 1.2 설치

```bash
npm install next-auth@beta
# App Router는 beta 버전 사용
```

## 2. 기본 설정

### 2.1 환경 변수

```bash
# .env.local
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key-here

# Google OAuth (선택)
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# GitHub OAuth (선택)
GITHUB_ID=your-github-id
GITHUB_SECRET=your-github-secret
```

**SECRET 생성:**
```bash
openssl rand -base64 32
```

### 2.2 Auth.js 설정

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import GitHubProvider from 'next-auth/providers/github';
import CredentialsProvider from 'next-auth/providers/credentials';
import bcrypt from 'bcrypt';

const handler = NextAuth({
  providers: [
    // Google OAuth
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),

    // GitHub OAuth
    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),

    // 이메일/비밀번호
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: "이메일", type: "email" },
        password: { label: "비밀번호", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        const user = await db.user.findUnique({
          where: { email: credentials.email }
        });

        if (!user) {
          return null;
        }

        const isValid = await bcrypt.compare(
          credentials.password,
          user.hashedPassword
        );

        if (!isValid) {
          return null;
        }

        return {
          id: user.id,
          email: user.email,
          name: user.name,
        };
      }
    })
  ],

  session: {
    strategy: 'jwt'
  },

  pages: {
    signIn: '/login',
    signOut: '/logout',
    error: '/error',
  },

  callbacks: {
    async jwt({ token, user }) {
      // 첫 로그인 시 user 정보 토큰에 추가
      if (user) {
        token.id = user.id;
      }
      return token;
    },

    async session({ session, token }) {
      // 세션에 사용자 ID 추가
      if (token && session.user) {
        session.user.id = token.id as string;
      }
      return session;
    }
  }
});

export { handler as GET, handler as POST };
```

### 2.3 타입 확장

```typescript
// types/next-auth.d.ts
import 'next-auth';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      email: string;
      name: string;
      image?: string;
    }
  }

  interface User {
    id: string;
    email: string;
    name: string;
  }
}
```

## 3. 로그인/로그아웃 구현

### 3.1 SessionProvider 설정

```typescript
// app/providers.tsx
'use client';

import { SessionProvider } from 'next-auth/react';

export default function Providers({ children }: { children: React.ReactNode }) {
  return (
    <SessionProvider>
      {children}
    </SessionProvider>
  );
}
```

```typescript
// app/layout.tsx
import Providers from './providers';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ko">
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

### 3.2 로그인 페이지

```typescript
// app/login/page.tsx
'use client';

import { signIn } from 'next-auth/react';
import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function LoginPage() {
  const router = useRouter();
  const [error, setError] = useState('');

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);

    const result = await signIn('credentials', {
      email: formData.get('email'),
      password: formData.get('password'),
      redirect: false,
    });

    if (result?.error) {
      setError('로그인에 실패했습니다');
    } else {
      router.push('/dashboard');
      router.refresh();
    }
  }

  return (
    <div className="max-w-md mx-auto mt-10">
      <h1 className="text-2xl font-bold mb-6">로그인</h1>

      {error && (
        <div className="mb-4 p-3 bg-red-100 text-red-700 rounded">
          {error}
        </div>
      )}

      {/* 이메일/비밀번호 로그인 */}
      <form onSubmit={handleSubmit} className="space-y-4 mb-6">
        <div>
          <label htmlFor="email" className="block mb-2">
            이메일
          </label>
          <input
            type="email"
            id="email"
            name="email"
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div>
          <label htmlFor="password" className="block mb-2">
            비밀번호
          </label>
          <input
            type="password"
            id="password"
            name="password"
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <button
          type="submit"
          className="w-full bg-blue-500 text-white py-2 rounded hover:bg-blue-600"
        >
          로그인
        </button>
      </form>

      {/* 소셜 로그인 */}
      <div className="space-y-2">
        <button
          onClick={() => signIn('google', { callbackUrl: '/dashboard' })}
          className="w-full bg-white border py-2 rounded hover:bg-gray-50 flex items-center justify-center gap-2"
        >
          <img src="/google-icon.svg" alt="Google" className="w-5 h-5" />
          Google로 로그인
        </button>

        <button
          onClick={() => signIn('github', { callbackUrl: '/dashboard' })}
          className="w-full bg-gray-800 text-white py-2 rounded hover:bg-gray-900 flex items-center justify-center gap-2"
        >
          <img src="/github-icon.svg" alt="GitHub" className="w-5 h-5" />
          GitHub로 로그인
        </button>
      </div>
    </div>
  );
}
```

### 3.3 로그아웃 버튼

```typescript
// app/components/LogoutButton.tsx
'use client';

import { signOut } from 'next-auth/react';

export default function LogoutButton() {
  return (
    <button
      onClick={() => signOut({ callbackUrl: '/login' })}
      className="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600"
    >
      로그아웃
    </button>
  );
}
```

## 4. 세션 사용하기

### 4.1 Client Component에서

```typescript
// app/components/UserProfile.tsx
'use client';

import { useSession } from 'next-auth/react';

export default function UserProfile() {
  const { data: session, status } = useSession();

  if (status === 'loading') {
    return <div>로딩 중...</div>;
  }

  if (status === 'unauthenticated') {
    return <div>로그인이 필요합니다</div>;
  }

  return (
    <div>
      <h2>안녕하세요, {session?.user?.name}님!</h2>
      <p>이메일: {session?.user?.email}</p>
      {session?.user?.image && (
        <img
          src={session.user.image}
          alt="Profile"
          className="w-16 h-16 rounded-full"
        />
      )}
    </div>
  );
}
```

### 4.2 Server Component에서

```typescript
// app/dashboard/page.tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await getServerSession();

  if (!session) {
    redirect('/login');
  }

  return (
    <div>
      <h1>대시보드</h1>
      <p>환영합니다, {session.user?.name}님!</p>
    </div>
  );
}
```

### 4.3 Server Action에서

```typescript
// app/actions/post.ts
'use server';

import { getServerSession } from 'next-auth';
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const session = await getServerSession();

  if (!session) {
    throw new Error('인증이 필요합니다');
  }

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.create({
    data: {
      title,
      content,
      authorId: session.user.id
    }
  });

  revalidatePath('/posts');
}
```

## 5. 보호된 페이지

### 5.1 Middleware로 보호

```typescript
// middleware.ts
export { default } from 'next-auth/middleware';

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*']
};
```

### 5.2 커스텀 인증 체크

```typescript
// middleware.ts
import { withAuth } from 'next-auth/middleware';

export default withAuth(
  function middleware(req) {
    console.log('Token:', req.nextauth.token);
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token
    },
  }
);

export const config = {
  matcher: ['/dashboard/:path*']
};
```

### 5.3 페이지 레벨 보호

```typescript
// app/dashboard/layout.tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';

export default async function DashboardLayout({
  children
}: {
  children: React.ReactNode
}) {
  const session = await getServerSession();

  if (!session) {
    redirect('/login');
  }

  return (
    <div className="min-h-screen">
      <header className="bg-gray-800 text-white p-4">
        <div className="container mx-auto flex justify-between items-center">
          <h1>Dashboard</h1>
          <div className="flex items-center gap-4">
            <span>{session.user?.name}</span>
            <LogoutButton />
          </div>
        </div>
      </header>
      <main className="container mx-auto p-4">
        {children}
      </main>
    </div>
  );
}
```

## 6. OAuth 제공자 설정

### 6.1 Google OAuth 설정

1. [Google Cloud Console](https://console.cloud.google.com/) 접속
2. 새 프로젝트 생성
3. OAuth 동의 화면 구성
4. 사용자 인증 정보 → OAuth 클라이언트 ID 생성
5. 승인된 리디렉션 URI 추가:
   - `http://localhost:3000/api/auth/callback/google`
   - `https://yourdomain.com/api/auth/callback/google`

### 6.2 GitHub OAuth 설정

1. GitHub Settings → Developer settings → OAuth Apps
2. New OAuth App
3. Application name 입력
4. Homepage URL: `http://localhost:3000`
5. Authorization callback URL:
   - `http://localhost:3000/api/auth/callback/github`

### 6.3 추가 정보 요청

```typescript
// app/api/auth/[...nextauth]/route.ts
GoogleProvider({
  clientId: process.env.GOOGLE_CLIENT_ID!,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
  authorization: {
    params: {
      scope: 'openid email profile',
      prompt: 'consent',
      access_type: 'offline',
      response_type: 'code'
    }
  }
}),
```

## 7. 데이터베이스 어댑터

### 7.1 Prisma 어댑터 설치

```bash
npm install @auth/prisma-adapter
```

### 7.2 Prisma 스키마

```prisma
// prisma/schema.prisma
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

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

### 7.3 어댑터 설정

```typescript
// app/api/auth/[...nextauth]/route.ts
import { PrismaAdapter } from '@auth/prisma-adapter';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

const handler = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    // ...
  ],
});
```

## 8. 실습 문제

### 문제 1: 소셜 로그인 + 이메일 연동
같은 이메일로 Google과 이메일/비밀번호 로그인을 모두 사용할 수 있도록 구현하세요.

**힌트:**
- 이메일로 기존 사용자 찾기
- OAuth 로그인 시 계정 연결
- `signIn` 콜백에서 처리

### 문제 2: 역할 기반 접근 제어
사용자에게 `role` (user, admin)을 부여하고, admin만 접근 가능한 페이지를 만드세요.

### 문제 3: 로그인 기록
사용자의 로그인 시간과 IP 주소를 기록하는 기능을 구현하세요.

## 9. 실습 해답

### 문제 1 해답

```typescript
// app/api/auth/[...nextauth]/route.ts
const handler = NextAuth({
  // ...
  callbacks: {
    async signIn({ user, account, profile }) {
      if (account?.provider === 'google' || account?.provider === 'github') {
        // 1. 이메일로 기존 사용자 찾기
        const existingUser = await db.user.findUnique({
          where: { email: user.email! },
          include: { accounts: true }
        });

        if (existingUser) {
          // 2. 이미 같은 제공자로 연결되어 있는지 확인
          const linkedAccount = existingUser.accounts.find(
            acc => acc.provider === account.provider
          );

          if (!linkedAccount) {
            // 3. 계정 연결
            await db.account.create({
              data: {
                userId: existingUser.id,
                type: account.type,
                provider: account.provider,
                providerAccountId: account.providerAccountId,
                access_token: account.access_token,
                refresh_token: account.refresh_token,
                expires_at: account.expires_at,
                token_type: account.token_type,
                scope: account.scope,
                id_token: account.id_token,
              }
            });
          }

          // 기존 사용자로 로그인
          user.id = existingUser.id;
        }
      }

      return true;
    },
  }
});
```

### 문제 2 해답

**Prisma 스키마 업데이트**
```prisma
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  role          String    @default("user") // 추가
  accounts      Account[]
  sessions      Session[]
}
```

**NextAuth 설정**
```typescript
// app/api/auth/[...nextauth]/route.ts
const handler = NextAuth({
  // ...
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;

        // DB에서 role 가져오기
        const dbUser = await db.user.findUnique({
          where: { id: user.id },
          select: { role: true }
        });

        token.role = dbUser?.role;
      }
      return token;
    },

    async session({ session, token }) {
      if (token && session.user) {
        session.user.id = token.id as string;
        session.user.role = token.role as string;
      }
      return session;
    }
  }
});
```

**타입 확장**
```typescript
// types/next-auth.d.ts
declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      email: string;
      name: string;
      image?: string;
      role: string; // 추가
    }
  }
}
```

**Admin 페이지 보호**
```typescript
// middleware.ts
import { withAuth } from 'next-auth/middleware';

export default withAuth(
  function middleware(req) {
    const token = req.nextauth.token;
    const isAdmin = token?.role === 'admin';

    // admin 페이지는 admin만 접근
    if (req.nextUrl.pathname.startsWith('/admin') && !isAdmin) {
      return NextResponse.rewrite(new URL('/denied', req.url));
    }
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token
    },
  }
);

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*']
};
```

**Admin 페이지**
```typescript
// app/admin/page.tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';

export default async function AdminPage() {
  const session = await getServerSession();

  if (session?.user?.role !== 'admin') {
    redirect('/denied');
  }

  return (
    <div>
      <h1>관리자 페이지</h1>
      <p>관리자만 접근할 수 있습니다.</p>
    </div>
  );
}
```

### 문제 3 해답

**Prisma 스키마**
```prisma
model LoginHistory {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  ip        String
  userAgent String?
  createdAt DateTime @default(now())
}

model User {
  // ...
  loginHistory LoginHistory[]
}
```

**로그인 기록 저장**
```typescript
// app/api/auth/[...nextauth]/route.ts
import { headers } from 'next/headers';

const handler = NextAuth({
  // ...
  events: {
    async signIn({ user }) {
      const headersList = headers();
      const ip = headersList.get('x-forwarded-for') ||
                 headersList.get('x-real-ip') ||
                 'unknown';
      const userAgent = headersList.get('user-agent') || 'unknown';

      await db.loginHistory.create({
        data: {
          userId: user.id,
          ip,
          userAgent
        }
      });
    }
  }
});
```

**로그인 기록 조회**
```typescript
// app/dashboard/login-history/page.tsx
import { getServerSession } from 'next-auth';

export default async function LoginHistoryPage() {
  const session = await getServerSession();

  const history = await db.loginHistory.findMany({
    where: { userId: session?.user?.id },
    orderBy: { createdAt: 'desc' },
    take: 10
  });

  return (
    <div>
      <h1>로그인 기록</h1>
      <table className="w-full">
        <thead>
          <tr>
            <th>날짜</th>
            <th>IP 주소</th>
            <th>브라우저</th>
          </tr>
        </thead>
        <tbody>
          {history.map(record => (
            <tr key={record.id}>
              <td>{new Date(record.createdAt).toLocaleString()}</td>
              <td>{record.ip}</td>
              <td>{record.userAgent}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

## 핵심 요약

1. **NextAuth.js**: Next.js 전용 완전한 인증 솔루션
2. **제공자**: OAuth (Google, GitHub) + Credentials
3. **세션 사용**: `useSession()` (클라이언트), `getServerSession()` (서버)
4. **보호**: Middleware + `authorized` 콜백
5. **확장**: 어댑터로 DB 연동, 콜백으로 커스터마이징

[← Chapter 18: 인증 기초](./chapter18-authentication-basics.md) | [Chapter 20: 권한 관리 →](./chapter20-authorization.md)