# Chapter 18: 인증 기초

## 1. 인증(Authentication)이란?

### 1.1 개념

인증은 사용자가 누구인지 확인하는 프로세스입니다.

```
사용자 → 로그인 (이메일/비밀번호) → 서버 검증 → 토큰 발급 → 인증 완료
```

### 1.2 웹 인증 방식

**세션 기반 인증**
```typescript
// 서버에서 세션 저장
// 클라이언트는 세션 ID만 쿠키로 보관
{
  sessionId: "abc123",
  userId: "user-1",
  createdAt: "2024-03-19"
}
```

**토큰 기반 인증 (JWT)**
```typescript
// 클라이언트가 토큰 보관
// 서버는 상태를 저장하지 않음 (Stateless)
const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

// 디코딩하면
{
  userId: "user-1",
  email: "user@example.com",
  exp: 1710864000
}
```

## 2. Next.js에서의 인증 구현

### 2.1 기본 로그인 구현

**로그인 Server Action**
```typescript
// app/actions/auth.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

export async function login(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  // 1. 사용자 찾기 (DB에서)
  const user = await db.user.findUnique({
    where: { email }
  });

  if (!user) {
    return { error: '사용자를 찾을 수 없습니다' };
  }

  // 2. 비밀번호 검증
  const isValid = await bcrypt.compare(password, user.hashedPassword);

  if (!isValid) {
    return { error: '비밀번호가 일치하지 않습니다' };
  }

  // 3. JWT 토큰 생성
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn: '7d' }
  );

  // 4. 쿠키에 저장
  cookies().set('auth-token', token, {
    httpOnly: true,    // JavaScript로 접근 불가
    secure: true,      // HTTPS만
    sameSite: 'lax',   // CSRF 방지
    maxAge: 60 * 60 * 24 * 7 // 7일
  });

  redirect('/dashboard');
}
```

**로그인 폼 컴포넌트**
```typescript
// app/login/page.tsx
import { login } from '@/app/actions/auth';

export default function LoginPage() {
  return (
    <div className="max-w-md mx-auto mt-10">
      <h1 className="text-2xl font-bold mb-6">로그인</h1>

      <form action={login} className="space-y-4">
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
    </div>
  );
}
```

### 2.2 회원가입 구현

```typescript
// app/actions/auth.ts
'use server';

import bcrypt from 'bcrypt';

export async function signup(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const name = formData.get('name') as string;

  // 1. 이메일 중복 확인
  const existing = await db.user.findUnique({
    where: { email }
  });

  if (existing) {
    return { error: '이미 사용 중인 이메일입니다' };
  }

  // 2. 비밀번호 해싱
  const hashedPassword = await bcrypt.hash(password, 10);

  // 3. 사용자 생성
  const user = await db.user.create({
    data: {
      email,
      name,
      hashedPassword
    }
  });

  // 4. 자동 로그인 (토큰 발급)
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn: '7d' }
  );

  cookies().set('auth-token', token, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7
  });

  redirect('/dashboard');
}
```

### 2.3 로그아웃 구현

```typescript
// app/actions/auth.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function logout() {
  // 쿠키 삭제
  cookies().delete('auth-token');
  redirect('/login');
}
```

**로그아웃 버튼**
```typescript
// app/components/LogoutButton.tsx
'use client';

import { logout } from '@/app/actions/auth';

export default function LogoutButton() {
  return (
    <form action={logout}>
      <button
        type="submit"
        className="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600"
      >
        로그아웃
      </button>
    </form>
  );
}
```

## 3. 인증 상태 확인

### 3.1 현재 사용자 가져오기

```typescript
// app/lib/auth.ts
import { cookies } from 'next/headers';
import jwt from 'jsonwebtoken';

export async function getCurrentUser() {
  const token = cookies().get('auth-token')?.value;

  if (!token) {
    return null;
  }

  try {
    // 토큰 검증
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string;
      email: string;
    };

    // 사용자 정보 조회
    const user = await db.user.findUnique({
      where: { id: payload.userId },
      select: {
        id: true,
        email: true,
        name: true,
        // 비밀번호는 제외
      }
    });

    return user;
  } catch (error) {
    return null;
  }
}
```

### 3.2 Server Component에서 사용

```typescript
// app/dashboard/page.tsx
import { getCurrentUser } from '@/app/lib/auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const user = await getCurrentUser();

  if (!user) {
    redirect('/login');
  }

  return (
    <div>
      <h1>환영합니다, {user.name}님!</h1>
      <p>이메일: {user.email}</p>
    </div>
  );
}
```

### 3.3 Middleware로 보호하기

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import jwt from 'jsonwebtoken';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value;

  // 보호된 경로
  if (request.nextUrl.pathname.startsWith('/dashboard')) {
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url));
    }

    try {
      jwt.verify(token, process.env.JWT_SECRET!);
    } catch (error) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // 로그인한 사용자는 로그인 페이지 접근 불가
  if (request.nextUrl.pathname === '/login') {
    if (token) {
      try {
        jwt.verify(token, process.env.JWT_SECRET!);
        return NextResponse.redirect(new URL('/dashboard', request.url));
      } catch (error) {
        // 토큰 만료 시 로그인 페이지 허용
      }
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/login']
};
```

## 4. 비밀번호 보안

### 4.1 비밀번호 해싱

```typescript
import bcrypt from 'bcrypt';

// 회원가입 시
const hashedPassword = await bcrypt.hash(password, 10);
// 10은 salt rounds (높을수록 안전하지만 느림)

// 로그인 시 검증
const isValid = await bcrypt.compare(password, hashedPassword);
```

### 4.2 비밀번호 강도 검증

```typescript
// app/lib/validation.ts
export function validatePassword(password: string) {
  const errors: string[] = [];

  if (password.length < 8) {
    errors.push('비밀번호는 최소 8자 이상이어야 합니다');
  }

  if (!/[A-Z]/.test(password)) {
    errors.push('대문자를 최소 1개 포함해야 합니다');
  }

  if (!/[a-z]/.test(password)) {
    errors.push('소문자를 최소 1개 포함해야 합니다');
  }

  if (!/[0-9]/.test(password)) {
    errors.push('숫자를 최소 1개 포함해야 합니다');
  }

  if (!/[!@#$%^&*]/.test(password)) {
    errors.push('특수문자를 최소 1개 포함해야 합니다');
  }

  return {
    isValid: errors.length === 0,
    errors
  };
}
```

## 5. 실습 문제

### 문제 1: 비밀번호 재설정 기능
비밀번호를 잊었을 때 이메일로 재설정 링크를 보내는 기능을 구현하세요.

**힌트:**
- 재설정 토큰 생성 (랜덤 문자열 + 만료시간)
- DB에 토큰 저장
- 이메일 발송
- 토큰 검증 후 비밀번호 변경

### 문제 2: Remember Me 기능
"로그인 상태 유지" 체크박스를 추가하고, 체크 시 토큰 만료 기간을 30일로 연장하세요.

### 문제 3: 이메일 인증
회원가입 시 이메일 인증을 거쳐야만 로그인할 수 있도록 구현하세요.

**요구사항:**
- 회원가입 시 `emailVerified: false` 상태로 저장
- 인증 이메일 발송
- 인증 링크 클릭 시 `emailVerified: true`로 변경
- 로그인 시 이메일 인증 여부 확인

## 6. 실습 해답

### 문제 1 해답

```typescript
// app/actions/auth.ts
'use server';

import crypto from 'crypto';
import { sendEmail } from '@/app/lib/email';

export async function requestPasswordReset(formData: FormData) {
  const email = formData.get('email') as string;

  const user = await db.user.findUnique({
    where: { email }
  });

  if (!user) {
    // 보안상 이메일 존재 여부를 노출하지 않음
    return { success: true };
  }

  // 1. 재설정 토큰 생성
  const resetToken = crypto.randomBytes(32).toString('hex');
  const resetTokenExpiry = new Date(Date.now() + 3600000); // 1시간

  // 2. DB에 저장
  await db.user.update({
    where: { id: user.id },
    data: {
      resetToken,
      resetTokenExpiry
    }
  });

  // 3. 이메일 발송
  const resetUrl = `${process.env.NEXT_PUBLIC_URL}/reset-password?token=${resetToken}`;

  await sendEmail({
    to: email,
    subject: '비밀번호 재설정',
    html: `
      <p>비밀번호 재설정을 요청하셨습니다.</p>
      <p>아래 링크를 클릭하여 비밀번호를 재설정하세요:</p>
      <a href="${resetUrl}">비밀번호 재설정하기</a>
      <p>이 링크는 1시간 동안 유효합니다.</p>
    `
  });

  return { success: true };
}

export async function resetPassword(formData: FormData) {
  const token = formData.get('token') as string;
  const newPassword = formData.get('password') as string;

  // 1. 토큰 검증
  const user = await db.user.findFirst({
    where: {
      resetToken: token,
      resetTokenExpiry: {
        gt: new Date() // 만료되지 않음
      }
    }
  });

  if (!user) {
    return { error: '유효하지 않거나 만료된 토큰입니다' };
  }

  // 2. 비밀번호 변경
  const hashedPassword = await bcrypt.hash(newPassword, 10);

  await db.user.update({
    where: { id: user.id },
    data: {
      hashedPassword,
      resetToken: null,
      resetTokenExpiry: null
    }
  });

  return { success: true };
}
```

**재설정 요청 페이지**
```typescript
// app/forgot-password/page.tsx
import { requestPasswordReset } from '@/app/actions/auth';

export default function ForgotPasswordPage() {
  return (
    <div className="max-w-md mx-auto mt-10">
      <h1 className="text-2xl font-bold mb-6">비밀번호 찾기</h1>

      <form action={requestPasswordReset} className="space-y-4">
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

        <button
          type="submit"
          className="w-full bg-blue-500 text-white py-2 rounded hover:bg-blue-600"
        >
          재설정 링크 보내기
        </button>
      </form>
    </div>
  );
}
```

**재설정 페이지**
```typescript
// app/reset-password/page.tsx
import { resetPassword } from '@/app/actions/auth';

export default function ResetPasswordPage({
  searchParams
}: {
  searchParams: { token: string }
}) {
  return (
    <div className="max-w-md mx-auto mt-10">
      <h1 className="text-2xl font-bold mb-6">비밀번호 재설정</h1>

      <form action={resetPassword} className="space-y-4">
        <input type="hidden" name="token" value={searchParams.token} />

        <div>
          <label htmlFor="password" className="block mb-2">
            새 비밀번호
          </label>
          <input
            type="password"
            id="password"
            name="password"
            required
            minLength={8}
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div>
          <label htmlFor="confirmPassword" className="block mb-2">
            비밀번호 확인
          </label>
          <input
            type="password"
            id="confirmPassword"
            name="confirmPassword"
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <button
          type="submit"
          className="w-full bg-blue-500 text-white py-2 rounded hover:bg-blue-600"
        >
          비밀번호 변경
        </button>
      </form>
    </div>
  );
}
```

### 문제 2 해답

```typescript
// app/actions/auth.ts
'use server';

export async function login(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const rememberMe = formData.get('rememberMe') === 'on';

  // ... 사용자 검증 로직 ...

  // 토큰 만료 기간 설정
  const expiresIn = rememberMe ? '30d' : '7d';
  const maxAge = rememberMe
    ? 60 * 60 * 24 * 30  // 30일
    : 60 * 60 * 24 * 7;  // 7일

  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET!,
    { expiresIn }
  );

  cookies().set('auth-token', token, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge
  });

  redirect('/dashboard');
}
```

**로그인 폼**
```typescript
// app/login/page.tsx
export default function LoginPage() {
  return (
    <form action={login} className="space-y-4">
      <div>
        <label htmlFor="email">이메일</label>
        <input type="email" id="email" name="email" required />
      </div>

      <div>
        <label htmlFor="password">비밀번호</label>
        <input type="password" id="password" name="password" required />
      </div>

      <div className="flex items-center">
        <input
          type="checkbox"
          id="rememberMe"
          name="rememberMe"
          className="mr-2"
        />
        <label htmlFor="rememberMe">로그인 상태 유지</label>
      </div>

      <button type="submit">로그인</button>
    </form>
  );
}
```

### 문제 3 해답

```typescript
// app/actions/auth.ts
'use server';

import crypto from 'crypto';
import { sendEmail } from '@/app/lib/email';

export async function signup(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const name = formData.get('name') as string;

  // ... 중복 확인 및 해싱 ...

  // 1. 인증 토큰 생성
  const verificationToken = crypto.randomBytes(32).toString('hex');

  // 2. 사용자 생성 (미인증 상태)
  const user = await db.user.create({
    data: {
      email,
      name,
      hashedPassword,
      emailVerified: false,
      verificationToken
    }
  });

  // 3. 인증 이메일 발송
  const verifyUrl = `${process.env.NEXT_PUBLIC_URL}/verify-email?token=${verificationToken}`;

  await sendEmail({
    to: email,
    subject: '이메일 인증',
    html: `
      <p>회원가입을 환영합니다!</p>
      <p>아래 링크를 클릭하여 이메일을 인증하세요:</p>
      <a href="${verifyUrl}">이메일 인증하기</a>
    `
  });

  redirect('/verify-email-sent');
}

export async function verifyEmail(token: string) {
  const user = await db.user.findFirst({
    where: { verificationToken: token }
  });

  if (!user) {
    return { error: '유효하지 않은 토큰입니다' };
  }

  await db.user.update({
    where: { id: user.id },
    data: {
      emailVerified: true,
      verificationToken: null
    }
  });

  return { success: true };
}

export async function login(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  const user = await db.user.findUnique({
    where: { email }
  });

  if (!user) {
    return { error: '사용자를 찾을 수 없습니다' };
  }

  // 이메일 인증 확인
  if (!user.emailVerified) {
    return { error: '이메일 인증이 필요합니다' };
  }

  // ... 나머지 로그인 로직 ...
}
```

**인증 페이지**
```typescript
// app/verify-email/page.tsx
import { verifyEmail } from '@/app/actions/auth';
import { redirect } from 'next/navigation';

export default async function VerifyEmailPage({
  searchParams
}: {
  searchParams: { token: string }
}) {
  const result = await verifyEmail(searchParams.token);

  if (result.success) {
    return (
      <div className="max-w-md mx-auto mt-10 text-center">
        <h1 className="text-2xl font-bold mb-4">이메일 인증 완료!</h1>
        <p className="mb-4">이제 로그인할 수 있습니다.</p>
        <a href="/login" className="text-blue-500 hover:underline">
          로그인하기
        </a>
      </div>
    );
  }

  return (
    <div className="max-w-md mx-auto mt-10 text-center">
      <h1 className="text-2xl font-bold mb-4 text-red-500">
        인증 실패
      </h1>
      <p>{result.error}</p>
    </div>
  );
}
```

## 핵심 요약

1. **인증 방식**: 세션(서버 저장) vs JWT(토큰 기반)
2. **Next.js 구현**: Server Actions + JWT + httpOnly 쿠키
3. **비밀번호 보안**: bcrypt로 해싱, 강도 검증
4. **인증 확인**: `getCurrentUser()` + Middleware
5. **추가 기능**: 비밀번호 재설정, Remember Me, 이메일 인증

[← Chapter 17: API 통합](../part06-forms-data/chapter17-api-integration.md) | [Chapter 19: NextAuth.js →](./chapter19-nextauth.md)