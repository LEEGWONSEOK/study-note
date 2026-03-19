# Chapter 16. Server Actions

## 16.1 Server Actions란?

Server Actions는 **서버에서 실행되는 비동기 함수**입니다. 폼 제출이나 데이터 변경 작업을 서버에서 안전하게 처리할 수 있습니다.

### 16.1.1 전통적인 방식 vs Server Actions

```typescript
// ❌ 전통적인 방식: API Route 필요
// app/api/posts/route.ts
export async function POST(request: Request) {
  const data = await request.json();
  // DB에 저장
  await db.post.create({ data });
  return Response.json({ success: true });
}

// Client Component
'use client';
async function handleSubmit(data) {
  await fetch('/api/posts', {
    method: 'POST',
    body: JSON.stringify(data),
  });
}
```

```typescript
// ✅ Server Actions: 직접 서버 함수 호출
'use server';

async function createPost(formData: FormData) {
  const title = formData.get('title');
  // DB에 저장
  await db.post.create({ data: { title } });
}
```

**장점:**
- ✅ API Route 불필요
- ✅ 타입 안전성
- ✅ 자동 재검증
- ✅ 간단한 코드
- ✅ Progressive Enhancement 지원

## 16.2 Server Actions 생성

### 16.2.1 'use server' 지시어

```typescript
// 방법 1: 파일 최상단에 'use server'
// actions/posts.ts
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.create({
    data: { title, content },
  });
}

export async function deletePost(id: number) {
  await db.post.delete({ where: { id } });
}
```

```typescript
// 방법 2: 함수 안에 'use server'
export default async function MyComponent() {
  async function createPost(formData: FormData) {
    'use server';

    const title = formData.get('title') as string;
    await db.post.create({ data: { title } });
  }

  return <form action={createPost}>...</form>;
}
```

### 16.2.2 기본 사용법

```typescript
// app/create-post/page.tsx
import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

async function createPost(formData: FormData) {
  'use server';

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  // 유효성 검사
  if (!title || !content) {
    return { error: '제목과 내용을 입력해주세요' };
  }

  // DB에 저장
  await db.post.create({
    data: { title, content },
  });

  // 캐시 재검증
  revalidatePath('/posts');

  // 리다이렉트
  redirect('/posts');
}

export default function CreatePostPage() {
  return (
    <form action={createPost}>
      <input name="title" type="text" placeholder="제목" required />
      <textarea name="content" placeholder="내용" required />
      <button type="submit">작성</button>
    </form>
  );
}
```

## 16.3 폼 제출 처리

### 16.3.1 기본 폼 제출

```typescript
// app/contact/page.tsx
async function submitContact(formData: FormData) {
  'use server';

  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  const message = formData.get('message') as string;

  // 데이터 처리
  await db.contact.create({
    data: { name, email, message },
  });

  console.log('문의 접수:', { name, email, message });
}

export default function ContactPage() {
  return (
    <div>
      <h1>문의하기</h1>
      <form action={submitContact}>
        <input name="name" type="text" placeholder="이름" required />
        <input name="email" type="email" placeholder="이메일" required />
        <textarea name="message" placeholder="메시지" required rows={5} />
        <button type="submit">제출</button>
      </form>
    </div>
  );
}
```

### 16.3.2 FormData 처리

```typescript
'use server';

export async function createUser(formData: FormData) {
  // 개별 필드 가져오기
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  const age = Number(formData.get('age'));

  // 객체로 변환
  const data = {
    name: formData.get('name') as string,
    email: formData.get('email') as string,
    age: Number(formData.get('age')),
  };

  // 또는 Object.fromEntries 사용
  const entries = Object.fromEntries(formData);

  await db.user.create({ data });
}
```

### 16.3.3 파일 업로드

```typescript
'use server';

import { writeFile } from 'fs/promises';
import path from 'path';

export async function uploadFile(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file) {
    return { error: '파일을 선택해주세요' };
  }

  // 파일 읽기
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  // 파일 저장
  const filename = Date.now() + '-' + file.name;
  const filepath = path.join(process.cwd(), 'public/uploads', filename);
  await writeFile(filepath, buffer);

  return { success: true, filename };
}
```

```typescript
// app/upload/page.tsx
import { uploadFile } from '@/actions/files';

export default function UploadPage() {
  return (
    <form action={uploadFile}>
      <input name="file" type="file" required />
      <button type="submit">업로드</button>
    </form>
  );
}
```

## 16.4 유효성 검사

### 16.4.1 기본 유효성 검사

```typescript
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  // 유효성 검사
  if (!title || title.length < 3) {
    return { error: '제목은 최소 3자 이상이어야 합니다' };
  }

  if (!content || content.length < 10) {
    return { error: '내용은 최소 10자 이상이어야 합니다' };
  }

  // DB에 저장
  await db.post.create({ data: { title, content } });

  return { success: true };
}
```

### 16.4.2 Zod를 사용한 유효성 검사

```bash
npm install zod
```

```typescript
// lib/validations.ts
import { z } from 'zod';

export const createPostSchema = z.object({
  title: z.string().min(3, '제목은 최소 3자 이상'),
  content: z.string().min(10, '내용은 최소 10자 이상'),
  published: z.boolean().optional(),
});

export type CreatePostInput = z.infer<typeof createPostSchema>;
```

```typescript
// actions/posts.ts
'use server';

import { createPostSchema } from '@/lib/validations';
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  // 데이터 파싱
  const data = {
    title: formData.get('title') as string,
    content: formData.get('content') as string,
    published: formData.get('published') === 'on',
  };

  // Zod로 유효성 검사
  const result = createPostSchema.safeParse(data);

  if (!result.success) {
    return {
      error: result.error.flatten().fieldErrors,
    };
  }

  // DB에 저장
  await db.post.create({ data: result.data });

  revalidatePath('/posts');

  return { success: true };
}
```

### 16.4.3 에러 처리

```typescript
// app/create-post/page.tsx
import { createPost } from '@/actions/posts';

export default function CreatePostPage() {
  return (
    <form action={async (formData) => {
      const result = await createPost(formData);

      if (result.error) {
        // 에러 처리
        console.error(result.error);
      }
    }}>
      <input name="title" type="text" placeholder="제목" />
      <textarea name="content" placeholder="내용" />
      <button type="submit">작성</button>
    </form>
  );
}
```

## 16.5 Revalidation (재검증)

### 16.5.1 경로 재검증

```typescript
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;

  await db.post.create({ data: { title } });

  // /posts 페이지 캐시 무효화
  revalidatePath('/posts');
}
```

### 16.5.2 태그 재검증

```typescript
'use server';

import { revalidateTag } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;

  await db.post.create({ data: { title } });

  // 'posts' 태그가 붙은 모든 캐시 무효화
  revalidateTag('posts');
}
```

```typescript
// 태그 붙이기
const posts = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});
```

### 16.5.3 여러 경로 재검증

```typescript
'use server';

import { revalidatePath } from 'next/cache';

export async function updatePost(id: number, formData: FormData) {
  await db.post.update({
    where: { id },
    data: { title: formData.get('title') as string },
  });

  // 여러 페이지 재검증
  revalidatePath('/posts');
  revalidatePath(`/posts/${id}`);
  revalidatePath('/');
}
```

## 16.6 Client Component에서 Server Actions 사용

### 16.6.1 useFormState 사용

```typescript
// actions/posts.ts
'use server';

export async function createPost(prevState: any, formData: FormData) {
  const title = formData.get('title') as string;

  if (!title) {
    return { error: '제목을 입력해주세요' };
  }

  await db.post.create({ data: { title } });

  return { success: '포스트가 생성되었습니다' };
}
```

```typescript
// app/create-post/page.tsx
'use client';

import { useFormState } from 'react-dom';
import { createPost } from '@/actions/posts';

export default function CreatePostPage() {
  const [state, formAction] = useFormState(createPost, null);

  return (
    <form action={formAction}>
      <input name="title" type="text" placeholder="제목" />
      <button type="submit">작성</button>

      {state?.error && <p style={{ color: 'red' }}>{state.error}</p>}
      {state?.success && <p style={{ color: 'green' }}>{state.success}</p>}
    </form>
  );
}
```

### 16.6.2 useFormStatus 사용 (로딩 상태)

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? '제출 중...' : '제출'}
    </button>
  );
}

export default function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" type="text" placeholder="제목" />
      <SubmitButton />
    </form>
  );
}
```

### 16.6.3 useTransition 사용

```typescript
'use client';

import { useTransition } from 'react';
import { deletePost } from '@/actions/posts';

export default function DeleteButton({ postId }: { postId: number }) {
  const [isPending, startTransition] = useTransition();

  return (
    <button
      onClick={() => {
        startTransition(async () => {
          await deletePost(postId);
        });
      }}
      disabled={isPending}
    >
      {isPending ? '삭제 중...' : '삭제'}
    </button>
  );
}
```

## 16.7 실전 예제: TODO 앱

```typescript
// actions/todos.ts
'use server';

import { revalidatePath } from 'next/cache';
import { db } from '@/lib/db';

export async function createTodo(formData: FormData) {
  const title = formData.get('title') as string;

  if (!title) {
    return { error: '할 일을 입력해주세요' };
  }

  await db.todo.create({
    data: { title, completed: false },
  });

  revalidatePath('/todos');
  return { success: true };
}

export async function toggleTodo(id: number, completed: boolean) {
  await db.todo.update({
    where: { id },
    data: { completed },
  });

  revalidatePath('/todos');
}

export async function deleteTodo(id: number) {
  await db.todo.delete({ where: { id } });
  revalidatePath('/todos');
}
```

```typescript
// app/todos/page.tsx
import { db } from '@/lib/db';
import TodoItem from './TodoItem';
import AddTodoForm from './AddTodoForm';

export default async function TodosPage() {
  const todos = await db.todo.findMany({
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="max-w-2xl mx-auto p-8">
      <h1 className="text-3xl font-bold mb-8">할 일 목록</h1>

      <AddTodoForm />

      <div className="mt-8 space-y-2">
        {todos.map(todo => (
          <TodoItem key={todo.id} todo={todo} />
        ))}
      </div>
    </div>
  );
}
```

```typescript
// app/todos/AddTodoForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { createTodo } from '@/actions/todos';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button
      type="submit"
      disabled={pending}
      className="px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-50"
    >
      {pending ? '추가 중...' : '추가'}
    </button>
  );
}

export default function AddTodoForm() {
  const [state, formAction] = useFormState(createTodo, null);

  return (
    <form action={formAction} className="flex gap-2">
      <input
        name="title"
        type="text"
        placeholder="새 할 일"
        className="flex-1 border px-4 py-2 rounded"
        required
      />
      <SubmitButton />

      {state?.error && (
        <p className="text-red-500 mt-2">{state.error}</p>
      )}
    </form>
  );
}
```

```typescript
// app/todos/TodoItem.tsx
'use client';

import { useTransition } from 'react';
import { toggleTodo, deleteTodo } from '@/actions/todos';

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

export default function TodoItem({ todo }: { todo: Todo }) {
  const [isPending, startTransition] = useTransition();

  return (
    <div className="flex items-center gap-2 p-4 border rounded">
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={(e) => {
          startTransition(async () => {
            await toggleTodo(todo.id, e.target.checked);
          });
        }}
        disabled={isPending}
      />

      <span
        className={`flex-1 ${todo.completed ? 'line-through text-gray-500' : ''}`}
      >
        {todo.title}
      </span>

      <button
        onClick={() => {
          startTransition(async () => {
            await deleteTodo(todo.id);
          });
        }}
        disabled={isPending}
        className="px-3 py-1 bg-red-500 text-white rounded text-sm disabled:opacity-50"
      >
        삭제
      </button>
    </div>
  );
}
```

## 16.8 Server Actions 모범 사례

### 16.8.1 별도 파일로 분리

```typescript
// ✅ 좋은 패턴
// actions/posts.ts
'use server';

export async function createPost(formData: FormData) { }
export async function updatePost(formData: FormData) { }
export async function deletePost(id: number) { }
```

### 16.8.2 에러 처리

```typescript
'use server';

export async function createPost(formData: FormData) {
  try {
    const title = formData.get('title') as string;

    // 유효성 검사
    if (!title) {
      return { error: '제목을 입력해주세요' };
    }

    // DB 저장
    await db.post.create({ data: { title } });

    return { success: true };
  } catch (error) {
    console.error('Post creation error:', error);
    return { error: '포스트 생성 중 오류가 발생했습니다' };
  }
}
```

### 16.8.3 인증 확인

```typescript
'use server';

import { auth } from '@/lib/auth';

export async function createPost(formData: FormData) {
  const session = await auth();

  if (!session?.user) {
    return { error: '로그인이 필요합니다' };
  }

  // 포스트 생성...
}
```

## 핵심 요약

1. **Server Actions**
   - 서버에서 실행되는 비동기 함수
   - `'use server'` 지시어
   - API Route 불필요

2. **폼 처리**
   - FormData로 데이터 접수
   - 유효성 검사 (Zod)
   - 에러 처리

3. **Revalidation**
   - `revalidatePath()`: 경로 재검증
   - `revalidateTag()`: 태그 재검증
   - 자동 캐시 갱신

4. **Client에서 사용**
   - `useFormState`: 상태 관리
   - `useFormStatus`: 로딩 상태
   - `useTransition`: 비동기 처리

5. **모범 사례**
   - 별도 파일로 분리
   - 에러 처리
   - 인증 확인
   - 타입 안전성

## 연습 문제

### 문제 1
Server Action으로 게시글을 생성하세요.

<details>
<summary>정답</summary>

```typescript
// actions/posts.ts
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;

  await db.post.create({ data: { title } });
}

// page.tsx
export default function Page() {
  return (
    <form action={createPost}>
      <input name="title" type="text" />
      <button type="submit">작성</button>
    </form>
  );
}
```
</details>

### 문제 2
useFormState로 에러 메시지를 표시하세요.

<details>
<summary>정답</summary>

```typescript
// actions/posts.ts
'use server';

export async function createPost(prevState: any, formData: FormData) {
  const title = formData.get('title') as string;

  if (!title) {
    return { error: '제목을 입력해주세요' };
  }

  await db.post.create({ data: { title } });
  return { success: '생성되었습니다' };
}

// page.tsx
'use client';

import { useFormState } from 'react-dom';

export default function Page() {
  const [state, formAction] = useFormState(createPost, null);

  return (
    <form action={formAction}>
      <input name="title" type="text" />
      <button type="submit">작성</button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}
```
</details>

## 다음 단계

다음 챕터에서는:
- REST API 연동
- 외부 API 호출
- API 프록시 패턴
- 에러 핸들링

[Chapter 17. 외부 API 연동 →](chapter17-api-integration.md)