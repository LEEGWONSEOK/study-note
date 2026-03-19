# Chapter 20: 권한 관리

## 1. 권한(Authorization)이란?

### 1.1 인증 vs 권한

```
인증 (Authentication): "당신은 누구인가?"
권한 (Authorization): "당신이 무엇을 할 수 있는가?"
```

**예시:**
```typescript
// 인증: 사용자가 로그인했는가?
const session = await getServerSession();
if (!session) redirect('/login');

// 권한: 사용자가 이 작업을 할 수 있는가?
if (session.user.role !== 'admin') {
  throw new Error('권한이 없습니다');
}
```

### 1.2 권한 모델

**역할 기반 (RBAC - Role-Based Access Control)**
```typescript
roles = {
  user: ['read'],
  editor: ['read', 'write'],
  admin: ['read', 'write', 'delete']
}
```

**권한 기반 (PBAC - Permission-Based Access Control)**
```typescript
permissions = {
  'post:read': true,
  'post:write': true,
  'post:delete': false,
  'user:manage': false
}
```

## 2. 역할 기반 권한 (RBAC)

### 2.1 역할 정의

```typescript
// lib/roles.ts
export enum Role {
  USER = 'user',
  EDITOR = 'editor',
  ADMIN = 'admin'
}

export const roleHierarchy = {
  [Role.USER]: 0,
  [Role.EDITOR]: 1,
  [Role.ADMIN]: 2
};

// 역할 비교
export function hasRole(userRole: Role, requiredRole: Role): boolean {
  return roleHierarchy[userRole] >= roleHierarchy[requiredRole];
}
```

### 2.2 역할 확인 유틸리티

```typescript
// lib/auth-utils.ts
import { getServerSession } from 'next-auth';
import { Role, hasRole } from './roles';

export async function requireAuth() {
  const session = await getServerSession();

  if (!session) {
    throw new Error('인증이 필요합니다');
  }

  return session;
}

export async function requireRole(role: Role) {
  const session = await requireAuth();

  if (!hasRole(session.user.role as Role, role)) {
    throw new Error('권한이 없습니다');
  }

  return session;
}
```

### 2.3 Server Action에서 권한 확인

```typescript
// app/actions/post.ts
'use server';

import { requireAuth, requireRole } from '@/lib/auth-utils';
import { Role } from '@/lib/roles';
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  // 로그인 필요
  const session = await requireAuth();

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

export async function deletePost(postId: string) {
  // Editor 이상 권한 필요
  const session = await requireRole(Role.EDITOR);

  const post = await db.post.findUnique({
    where: { id: postId }
  });

  if (!post) {
    throw new Error('게시글을 찾을 수 없습니다');
  }

  // 작성자 본인이거나 Admin인 경우만 삭제 가능
  if (post.authorId !== session.user.id && session.user.role !== Role.ADMIN) {
    throw new Error('이 게시글을 삭제할 권한이 없습니다');
  }

  await db.post.delete({
    where: { id: postId }
  });

  revalidatePath('/posts');
}

export async function deleteUser(userId: string) {
  // Admin만 가능
  await requireRole(Role.ADMIN);

  await db.user.delete({
    where: { id: userId }
  });

  revalidatePath('/admin/users');
}
```

### 2.4 페이지 레벨 권한

```typescript
// app/admin/page.tsx
import { requireRole } from '@/lib/auth-utils';
import { Role } from '@/lib/roles';

export default async function AdminPage() {
  await requireRole(Role.ADMIN);

  return (
    <div>
      <h1>관리자 페이지</h1>
      <p>Admin 역할만 접근 가능합니다.</p>
    </div>
  );
}
```

### 2.5 UI 조건부 렌더링

```typescript
// app/components/PostActions.tsx
'use client';

import { useSession } from 'next-auth/react';
import { deletePost } from '@/app/actions/post';
import { Role } from '@/lib/roles';

export default function PostActions({
  postId,
  authorId
}: {
  postId: string;
  authorId: string;
}) {
  const { data: session } = useSession();

  const canDelete =
    session?.user?.id === authorId ||
    session?.user?.role === Role.ADMIN ||
    session?.user?.role === Role.EDITOR;

  if (!canDelete) {
    return null;
  }

  return (
    <form action={deletePost.bind(null, postId)}>
      <button
        type="submit"
        className="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600"
      >
        삭제
      </button>
    </form>
  );
}
```

## 3. 권한 기반 접근 제어 (PBAC)

### 3.1 권한 정의

```typescript
// lib/permissions.ts
export enum Permission {
  // 게시글
  POST_READ = 'post:read',
  POST_CREATE = 'post:create',
  POST_UPDATE = 'post:update',
  POST_DELETE = 'post:delete',

  // 사용자
  USER_READ = 'user:read',
  USER_UPDATE = 'user:update',
  USER_DELETE = 'user:delete',

  // 관리
  ADMIN_ACCESS = 'admin:access',
  SETTINGS_MANAGE = 'settings:manage'
}

// 역할별 권한 매핑
export const rolePermissions: Record<Role, Permission[]> = {
  [Role.USER]: [
    Permission.POST_READ,
    Permission.POST_CREATE,
    Permission.USER_READ
  ],

  [Role.EDITOR]: [
    Permission.POST_READ,
    Permission.POST_CREATE,
    Permission.POST_UPDATE,
    Permission.POST_DELETE,
    Permission.USER_READ
  ],

  [Role.ADMIN]: [
    Permission.POST_READ,
    Permission.POST_CREATE,
    Permission.POST_UPDATE,
    Permission.POST_DELETE,
    Permission.USER_READ,
    Permission.USER_UPDATE,
    Permission.USER_DELETE,
    Permission.ADMIN_ACCESS,
    Permission.SETTINGS_MANAGE
  ]
};
```

### 3.2 권한 확인 함수

```typescript
// lib/permissions.ts
export function hasPermission(
  userRole: Role,
  permission: Permission
): boolean {
  const permissions = rolePermissions[userRole];
  return permissions.includes(permission);
}

export function hasAnyPermission(
  userRole: Role,
  permissions: Permission[]
): boolean {
  return permissions.some(p => hasPermission(userRole, p));
}

export function hasAllPermissions(
  userRole: Role,
  permissions: Permission[]
): boolean {
  return permissions.every(p => hasPermission(userRole, p));
}
```

### 3.3 권한 확인 유틸리티

```typescript
// lib/auth-utils.ts
import { Permission, hasPermission } from './permissions';

export async function requirePermission(permission: Permission) {
  const session = await requireAuth();

  if (!hasPermission(session.user.role as Role, permission)) {
    throw new Error(`${permission} 권한이 없습니다`);
  }

  return session;
}
```

### 3.4 Server Action에서 사용

```typescript
// app/actions/post.ts
'use server';

import { requirePermission } from '@/lib/auth-utils';
import { Permission } from '@/lib/permissions';

export async function updatePost(postId: string, formData: FormData) {
  await requirePermission(Permission.POST_UPDATE);

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.update({
    where: { id: postId },
    data: { title, content }
  });

  revalidatePath(`/posts/${postId}`);
}

export async function deletePost(postId: string) {
  await requirePermission(Permission.POST_DELETE);

  await db.post.delete({
    where: { id: postId }
  });

  revalidatePath('/posts');
}
```

### 3.5 UI 조건부 렌더링

```typescript
// app/components/PostActions.tsx
'use client';

import { useSession } from 'next-auth/react';
import { hasPermission, Permission } from '@/lib/permissions';

export default function PostActions({ postId }: { postId: string }) {
  const { data: session } = useSession();

  const canUpdate = session?.user?.role &&
    hasPermission(session.user.role, Permission.POST_UPDATE);

  const canDelete = session?.user?.role &&
    hasPermission(session.user.role, Permission.POST_DELETE);

  return (
    <div className="flex gap-2">
      {canUpdate && (
        <button className="px-4 py-2 bg-blue-500 text-white rounded">
          수정
        </button>
      )}

      {canDelete && (
        <button className="px-4 py-2 bg-red-500 text-white rounded">
          삭제
        </button>
      )}
    </div>
  );
}
```

## 4. 리소스 소유권 확인

### 4.1 소유자 확인

```typescript
// lib/ownership.ts
import { getServerSession } from 'next-auth';

export async function isOwner(resourceId: string, tableName: string) {
  const session = await getServerSession();

  if (!session) {
    return false;
  }

  const resource = await db[tableName].findUnique({
    where: { id: resourceId },
    select: { authorId: true }
  });

  return resource?.authorId === session.user.id;
}

export async function requireOwnership(resourceId: string, tableName: string) {
  const session = await getServerSession();

  if (!session) {
    throw new Error('인증이 필요합니다');
  }

  const resource = await db[tableName].findUnique({
    where: { id: resourceId },
    select: { authorId: true }
  });

  if (!resource) {
    throw new Error('리소스를 찾을 수 없습니다');
  }

  if (resource.authorId !== session.user.id) {
    throw new Error('소유자만 접근할 수 있습니다');
  }

  return session;
}
```

### 4.2 소유자 또는 Admin 확인

```typescript
// lib/ownership.ts
export async function requireOwnerOrAdmin(
  resourceId: string,
  tableName: string
) {
  const session = await requireAuth();

  // Admin은 모든 리소스 접근 가능
  if (session.user.role === Role.ADMIN) {
    return session;
  }

  // 아니면 소유자 확인
  await requireOwnership(resourceId, tableName);

  return session;
}
```

### 4.3 Server Action에서 사용

```typescript
// app/actions/post.ts
'use server';

import { requireOwnerOrAdmin } from '@/lib/ownership';

export async function updatePost(postId: string, formData: FormData) {
  // 소유자 또는 Admin만 수정 가능
  await requireOwnerOrAdmin(postId, 'post');

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.update({
    where: { id: postId },
    data: { title, content }
  });

  revalidatePath(`/posts/${postId}`);
}
```

## 5. 동적 권한 관리

### 5.1 데이터베이스에 권한 저장

**Prisma 스키마**
```prisma
model User {
  id          String       @id @default(cuid())
  email       String       @unique
  name        String
  role        String       @default("user")
  permissions Permission[]
}

model Permission {
  id     String @id @default(cuid())
  userId String
  user   User   @relation(fields: [userId], references: [id])
  name   String // "post:delete", "user:manage" 등

  @@unique([userId, name])
}
```

### 5.2 권한 확인

```typescript
// lib/dynamic-permissions.ts
export async function hasCustomPermission(
  userId: string,
  permissionName: string
): Promise<boolean> {
  const permission = await db.permission.findUnique({
    where: {
      userId_name: {
        userId,
        name: permissionName
      }
    }
  });

  return !!permission;
}

export async function requireCustomPermission(permissionName: string) {
  const session = await requireAuth();

  // 1. 역할 기반 권한 확인
  const rolePermission = hasPermission(
    session.user.role as Role,
    permissionName as Permission
  );

  if (rolePermission) {
    return session;
  }

  // 2. 사용자별 커스텀 권한 확인
  const customPermission = await hasCustomPermission(
    session.user.id,
    permissionName
  );

  if (!customPermission) {
    throw new Error(`${permissionName} 권한이 없습니다`);
  }

  return session;
}
```

### 5.3 권한 부여/제거

```typescript
// app/actions/permissions.ts
'use server';

import { requireRole } from '@/lib/auth-utils';
import { Role } from '@/lib/roles';

export async function grantPermission(
  userId: string,
  permissionName: string
) {
  // Admin만 권한 부여 가능
  await requireRole(Role.ADMIN);

  await db.permission.create({
    data: {
      userId,
      name: permissionName
    }
  });

  revalidatePath('/admin/permissions');
}

export async function revokePermission(
  userId: string,
  permissionName: string
) {
  await requireRole(Role.ADMIN);

  await db.permission.delete({
    where: {
      userId_name: {
        userId,
        name: permissionName
      }
    }
  });

  revalidatePath('/admin/permissions');
}
```

## 6. API Route 권한 관리

### 6.1 권한 확인 미들웨어

```typescript
// app/api/middleware.ts
import { getServerSession } from 'next-auth';
import { NextRequest, NextResponse } from 'next/server';
import { Permission, hasPermission } from '@/lib/permissions';

export async function withAuth(
  handler: (req: NextRequest, session: any) => Promise<Response>
) {
  return async (req: NextRequest) => {
    const session = await getServerSession();

    if (!session) {
      return NextResponse.json(
        { error: '인증이 필요합니다' },
        { status: 401 }
      );
    }

    return handler(req, session);
  };
}

export function withPermission(
  permission: Permission,
  handler: (req: NextRequest, session: any) => Promise<Response>
) {
  return withAuth(async (req, session) => {
    if (!hasPermission(session.user.role, permission)) {
      return NextResponse.json(
        { error: '권한이 없습니다' },
        { status: 403 }
      );
    }

    return handler(req, session);
  });
}
```

### 6.2 API Route에서 사용

```typescript
// app/api/posts/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { withAuth, withPermission } from '../../middleware';
import { Permission } from '@/lib/permissions';

export const GET = withAuth(async (req, session) => {
  const { id } = req.nextUrl.searchParams;

  const post = await db.post.findUnique({
    where: { id }
  });

  return NextResponse.json(post);
});

export const PUT = withPermission(
  Permission.POST_UPDATE,
  async (req, session) => {
    const { id } = req.nextUrl.searchParams;
    const body = await req.json();

    const post = await db.post.update({
      where: { id },
      data: body
    });

    return NextResponse.json(post);
  }
);

export const DELETE = withPermission(
  Permission.POST_DELETE,
  async (req, session) => {
    const { id } = req.nextUrl.searchParams;

    await db.post.delete({
      where: { id }
    });

    return NextResponse.json({ success: true });
  }
);
```

## 7. 실습 문제

### 문제 1: 팀 기반 권한
팀(Team) 개념을 도입하고, 팀 내에서만 게시글을 공유할 수 있도록 구현하세요.

**요구사항:**
- 팀 생성/가입 기능
- 팀 내 게시글만 조회 가능
- 팀 관리자(Team Admin) 역할

### 문제 2: 임시 권한 부여
특정 사용자에게 일정 기간 동안만 권한을 부여하는 기능을 구현하세요.

### 문제 3: 권한 상속
부모-자식 관계의 리소스에서 권한을 상속하는 구조를 만드세요.
(예: 프로젝트 접근 권한이 있으면 하위 작업도 접근 가능)

## 8. 실습 해답

### 문제 1 해답

**Prisma 스키마**
```prisma
model Team {
  id      String       @id @default(cuid())
  name    String
  members TeamMember[]
  posts   Post[]
}

model TeamMember {
  id     String @id @default(cuid())
  teamId String
  team   Team   @relation(fields: [teamId], references: [id])
  userId String
  user   User   @relation(fields: [userId], references: [id])
  role   String @default("member") // "member" | "admin"

  @@unique([teamId, userId])
}

model Post {
  id       String  @id @default(cuid())
  title    String
  content  String
  authorId String
  author   User    @relation(fields: [authorId], references: [id])
  teamId   String?
  team     Team?   @relation(fields: [teamId], references: [id])
}
```

**팀 권한 확인**
```typescript
// lib/team-permissions.ts
export async function isTeamMember(
  userId: string,
  teamId: string
): Promise<boolean> {
  const member = await db.teamMember.findUnique({
    where: {
      teamId_userId: {
        teamId,
        userId
      }
    }
  });

  return !!member;
}

export async function isTeamAdmin(
  userId: string,
  teamId: string
): Promise<boolean> {
  const member = await db.teamMember.findUnique({
    where: {
      teamId_userId: {
        teamId,
        userId
      }
    }
  });

  return member?.role === 'admin';
}

export async function requireTeamMember(teamId: string) {
  const session = await requireAuth();

  const isMember = await isTeamMember(session.user.id, teamId);

  if (!isMember) {
    throw new Error('팀 멤버만 접근할 수 있습니다');
  }

  return session;
}
```

**팀 게시글 조회**
```typescript
// app/teams/[id]/posts/page.tsx
import { requireTeamMember } from '@/lib/team-permissions';

export default async function TeamPostsPage({
  params
}: {
  params: { id: string }
}) {
  await requireTeamMember(params.id);

  const posts = await db.post.findMany({
    where: { teamId: params.id },
    include: { author: true }
  });

  return (
    <div>
      <h1>팀 게시글</h1>
      {posts.map(post => (
        <div key={post.id}>
          <h2>{post.title}</h2>
          <p>작성자: {post.author.name}</p>
        </div>
      ))}
    </div>
  );
}
```

### 문제 2 해답

**Prisma 스키마**
```prisma
model TemporaryPermission {
  id         String   @id @default(cuid())
  userId     String
  user       User     @relation(fields: [userId], references: [id])
  permission String
  expiresAt  DateTime
  createdAt  DateTime @default(now())

  @@index([userId, expiresAt])
}
```

**임시 권한 확인**
```typescript
// lib/temporary-permissions.ts
export async function hasTemporaryPermission(
  userId: string,
  permissionName: string
): Promise<boolean> {
  const permission = await db.temporaryPermission.findFirst({
    where: {
      userId,
      permission: permissionName,
      expiresAt: {
        gt: new Date() // 만료되지 않음
      }
    }
  });

  return !!permission;
}

export async function grantTemporaryPermission(
  userId: string,
  permissionName: string,
  durationHours: number
) {
  await requireRole(Role.ADMIN);

  const expiresAt = new Date();
  expiresAt.setHours(expiresAt.getHours() + durationHours);

  await db.temporaryPermission.create({
    data: {
      userId,
      permission: permissionName,
      expiresAt
    }
  });
}

export async function requirePermissionWithTemporary(
  permissionName: string
) {
  const session = await requireAuth();

  // 1. 영구 권한 확인
  const hasPermanent = hasPermission(
    session.user.role as Role,
    permissionName as Permission
  );

  if (hasPermanent) {
    return session;
  }

  // 2. 임시 권한 확인
  const hasTemporary = await hasTemporaryPermission(
    session.user.id,
    permissionName
  );

  if (!hasTemporary) {
    throw new Error(`${permissionName} 권한이 없습니다`);
  }

  return session;
}
```

**임시 권한 정리 (Cron Job)**
```typescript
// app/api/cron/cleanup-permissions/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  // 만료된 임시 권한 삭제
  await db.temporaryPermission.deleteMany({
    where: {
      expiresAt: {
        lt: new Date()
      }
    }
  });

  return NextResponse.json({ success: true });
}
```

### 문제 3 해답

**Prisma 스키마**
```prisma
model Project {
  id    String @id @default(cuid())
  name  String
  tasks Task[]
  permissions ProjectPermission[]
}

model Task {
  id        String  @id @default(cuid())
  title     String
  projectId String
  project   Project @relation(fields: [projectId], references: [id])
}

model ProjectPermission {
  id         String  @id @default(cuid())
  projectId  String
  project    Project @relation(fields: [projectId], references: [id])
  userId     String
  user       User    @relation(fields: [userId], references: [id])
  permission String  // "read", "write", "admin"

  @@unique([projectId, userId])
}
```

**권한 상속 확인**
```typescript
// lib/project-permissions.ts
export async function hasProjectPermission(
  userId: string,
  projectId: string,
  requiredPermission: 'read' | 'write' | 'admin'
): Promise<boolean> {
  const permission = await db.projectPermission.findUnique({
    where: {
      projectId_userId: {
        projectId,
        userId
      }
    }
  });

  if (!permission) {
    return false;
  }

  // 권한 계층: admin > write > read
  const hierarchy = { read: 0, write: 1, admin: 2 };

  return hierarchy[permission.permission] >= hierarchy[requiredPermission];
}

export async function hasTaskPermission(
  userId: string,
  taskId: string,
  requiredPermission: 'read' | 'write' | 'admin'
): Promise<boolean> {
  // 1. Task의 Project 찾기
  const task = await db.task.findUnique({
    where: { id: taskId },
    select: { projectId: true }
  });

  if (!task) {
    return false;
  }

  // 2. Project 권한 확인 (상속)
  return hasProjectPermission(userId, task.projectId, requiredPermission);
}

export async function requireTaskPermission(
  taskId: string,
  requiredPermission: 'read' | 'write' | 'admin'
) {
  const session = await requireAuth();

  const hasPermission = await hasTaskPermission(
    session.user.id,
    taskId,
    requiredPermission
  );

  if (!hasPermission) {
    throw new Error('권한이 없습니다');
  }

  return session;
}
```

**사용 예시**
```typescript
// app/actions/tasks.ts
'use server';

import { requireTaskPermission } from '@/lib/project-permissions';

export async function updateTask(taskId: string, formData: FormData) {
  // Task 수정 권한 확인 (Project 권한에서 상속)
  await requireTaskPermission(taskId, 'write');

  const title = formData.get('title') as string;

  await db.task.update({
    where: { id: taskId },
    data: { title }
  });

  revalidatePath(`/tasks/${taskId}`);
}
```

## 핵심 요약

1. **RBAC**: 역할 기반 권한 제어 (user, editor, admin)
2. **PBAC**: 세밀한 권한 제어 (post:read, post:write 등)
3. **소유권**: 리소스 소유자 확인 + Admin 예외
4. **동적 권한**: DB에 권한 저장하여 유연하게 관리
5. **API 보호**: `withAuth`, `withPermission` 미들웨어

[← Chapter 19: NextAuth.js](./chapter19-nextauth.md) | [Part 08: 성능 최적화 →](../part08-optimization/README.md)