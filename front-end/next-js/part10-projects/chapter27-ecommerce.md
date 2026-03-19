# Chapter 27: E-Commerce 프로젝트

## 1. 프로젝트 개요

### 1.1 기능 요구사항

```
✅ 상품 목록 및 상세
✅ 장바구니
✅ 주문 및 결제 (Stripe)
✅ 주문 내역
✅ 관리자 대시보드
✅ 재고 관리
✅ 이미지 최적화
```

### 1.2 기술 스택

```
Frontend:   Next.js 14 (App Router)
Styling:    Tailwind CSS
Database:   PostgreSQL + Prisma
Payment:    Stripe
State:      Zustand
Deployment: Vercel
```

## 2. 프로젝트 설정

### 2.1 초기 설정

```bash
npx create-next-app@latest ecommerce
cd ecommerce

# 의존성 설치
npm install prisma @prisma/client
npm install stripe @stripe/stripe-js
npm install zustand
npm install zod
npm install sharp
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

model Product {
  id          String      @id @default(cuid())
  name        String
  description String      @db.Text
  price       Decimal     @db.Decimal(10, 2)
  image       String
  stock       Int
  category    String
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  orderItems  OrderItem[]
}

model Order {
  id              String      @id @default(cuid())
  userId          String
  status          String      @default("pending") // pending, paid, shipped, delivered
  total           Decimal     @db.Decimal(10, 2)
  stripeSessionId String?     @unique
  shippingAddress Json
  items           OrderItem[]
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt
}

model OrderItem {
  id        String  @id @default(cuid())
  orderId   String
  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId String
  product   Product @relation(fields: [productId], references: [id])
  quantity  Int
  price     Decimal @db.Decimal(10, 2)
}
```

```bash
npx prisma migrate dev --name init
npx prisma generate
```

### 2.3 환경 변수

```bash
# .env.local
DATABASE_URL="postgresql://user:password@localhost:5432/ecommerce"

STRIPE_PUBLISHABLE_KEY="pk_test_..."
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."

NEXT_PUBLIC_URL="http://localhost:3000"
```

## 3. 상품 목록 및 상세

### 3.1 상품 목록

```tsx
// app/products/page.tsx
import Link from 'next/link';
import Image from 'next/image';
import { prisma } from '@/lib/prisma';

export default async function ProductsPage({
  searchParams
}: {
  searchParams: { category?: string }
}) {
  const products = await prisma.product.findMany({
    where: searchParams.category
      ? { category: searchParams.category }
      : {},
    orderBy: { createdAt: 'desc' }
  });

  const categories = await prisma.product.findMany({
    select: { category: true },
    distinct: ['category']
  });

  return (
    <div className="max-w-7xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">상품 목록</h1>

      {/* 카테고리 필터 */}
      <div className="flex gap-2 mb-8">
        <Link
          href="/products"
          className="px-4 py-2 border rounded hover:bg-gray-100"
        >
          전체
        </Link>
        {categories.map(({ category }) => (
          <Link
            key={category}
            href={`/products?category=${category}`}
            className="px-4 py-2 border rounded hover:bg-gray-100"
          >
            {category}
          </Link>
        ))}
      </div>

      {/* 상품 그리드 */}
      <div className="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-6">
        {products.map(product => (
          <Link
            key={product.id}
            href={`/products/${product.id}`}
            className="border rounded-lg p-4 hover:shadow-lg transition"
          >
            <div className="relative aspect-square mb-4">
              <Image
                src={product.image}
                alt={product.name}
                fill
                style={{ objectFit: 'cover' }}
                className="rounded"
              />
            </div>

            <h2 className="font-bold mb-2">{product.name}</h2>
            <p className="text-gray-600 text-sm mb-2 line-clamp-2">
              {product.description}
            </p>
            <div className="flex justify-between items-center">
              <span className="text-xl font-bold">
                ${product.price.toString()}
              </span>
              <span className="text-sm text-gray-500">
                재고: {product.stock}
              </span>
            </div>
          </Link>
        ))}
      </div>
    </div>
  );
}
```

### 3.2 상품 상세

```tsx
// app/products/[id]/page.tsx
import { notFound } from 'next/navigation';
import Image from 'next/image';
import { prisma } from '@/lib/prisma';
import AddToCartButton from '@/app/components/AddToCartButton';

export async function generateMetadata({
  params
}: {
  params: { id: string }
}) {
  const product = await prisma.product.findUnique({
    where: { id: params.id }
  });

  if (!product) return {};

  return {
    title: product.name,
    description: product.description,
  };
}

export default async function ProductPage({
  params
}: {
  params: { id: string }
}) {
  const product = await prisma.product.findUnique({
    where: { id: params.id }
  });

  if (!product) {
    notFound();
  }

  return (
    <div className="max-w-7xl mx-auto p-4">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        {/* 이미지 */}
        <div className="relative aspect-square">
          <Image
            src={product.image}
            alt={product.name}
            fill
            style={{ objectFit: 'cover' }}
            className="rounded-lg"
            priority
          />
        </div>

        {/* 상품 정보 */}
        <div>
          <h1 className="text-3xl font-bold mb-4">{product.name}</h1>

          <div className="flex items-center gap-4 mb-6">
            <span className="text-3xl font-bold text-blue-600">
              ${product.price.toString()}
            </span>
            <span
              className={`px-3 py-1 rounded ${
                product.stock > 0
                  ? 'bg-green-100 text-green-800'
                  : 'bg-red-100 text-red-800'
              }`}
            >
              {product.stock > 0 ? `재고 ${product.stock}개` : '품절'}
            </span>
          </div>

          <p className="text-gray-700 mb-6 leading-relaxed">
            {product.description}
          </p>

          <div className="mb-6">
            <span className="inline-block px-3 py-1 bg-gray-200 rounded-full text-sm">
              {product.category}
            </span>
          </div>

          {product.stock > 0 && (
            <AddToCartButton product={product} />
          )}
        </div>
      </div>
    </div>
  );
}
```

## 4. 장바구니 (Zustand)

### 4.1 Store

```typescript
// store/cart.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type CartItem = {
  id: string;
  name: string;
  price: number;
  image: string;
  quantity: number;
};

type CartStore = {
  items: CartItem[];
  addItem: (product: any) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  total: () => number;
};

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: product => {
        const items = get().items;
        const existing = items.find(item => item.id === product.id);

        if (existing) {
          set({
            items: items.map(item =>
              item.id === product.id
                ? { ...item, quantity: item.quantity + 1 }
                : item
            )
          });
        } else {
          set({
            items: [
              ...items,
              {
                id: product.id,
                name: product.name,
                price: parseFloat(product.price.toString()),
                image: product.image,
                quantity: 1
              }
            ]
          });
        }
      },

      removeItem: id => {
        set({ items: get().items.filter(item => item.id !== id) });
      },

      updateQuantity: (id, quantity) => {
        if (quantity <= 0) {
          get().removeItem(id);
        } else {
          set({
            items: get().items.map(item =>
              item.id === id ? { ...item, quantity } : item
            )
          });
        }
      },

      clearCart: () => {
        set({ items: [] });
      },

      total: () => {
        return get().items.reduce(
          (sum, item) => sum + item.price * item.quantity,
          0
        );
      }
    }),
    {
      name: 'cart-storage'
    }
  )
);
```

### 4.2 AddToCartButton

```tsx
// app/components/AddToCartButton.tsx
'use client';

import { useCartStore } from '@/store/cart';

export default function AddToCartButton({ product }: { product: any }) {
  const addItem = useCartStore(state => state.addItem);

  function handleClick() {
    addItem(product);
    alert('장바구니에 추가되었습니다!');
  }

  return (
    <button
      onClick={handleClick}
      className="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
    >
      장바구니에 추가
    </button>
  );
}
```

### 4.3 장바구니 페이지

```tsx
// app/cart/page.tsx
'use client';

import Image from 'next/image';
import Link from 'next/link';
import { useCartStore } from '@/store/cart';

export default function CartPage() {
  const { items, removeItem, updateQuantity, total } = useCartStore();

  if (items.length === 0) {
    return (
      <div className="max-w-7xl mx-auto p-4 text-center">
        <h1 className="text-3xl font-bold mb-4">장바구니가 비어있습니다</h1>
        <Link
          href="/products"
          className="text-blue-600 hover:underline"
        >
          쇼핑 계속하기
        </Link>
      </div>
    );
  }

  return (
    <div className="max-w-7xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">장바구니</h1>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
        {/* 장바구니 아이템 */}
        <div className="lg:col-span-2 space-y-4">
          {items.map(item => (
            <div
              key={item.id}
              className="flex gap-4 border rounded-lg p-4"
            >
              <div className="relative w-24 h-24 flex-shrink-0">
                <Image
                  src={item.image}
                  alt={item.name}
                  fill
                  style={{ objectFit: 'cover' }}
                  className="rounded"
                />
              </div>

              <div className="flex-1">
                <h3 className="font-bold mb-2">{item.name}</h3>
                <p className="text-gray-600">${item.price}</p>

                <div className="flex items-center gap-2 mt-2">
                  <button
                    onClick={() => updateQuantity(item.id, item.quantity - 1)}
                    className="px-2 py-1 border rounded"
                  >
                    -
                  </button>
                  <span>{item.quantity}</span>
                  <button
                    onClick={() => updateQuantity(item.id, item.quantity + 1)}
                    className="px-2 py-1 border rounded"
                  >
                    +
                  </button>
                  <button
                    onClick={() => removeItem(item.id)}
                    className="ml-auto text-red-600 hover:underline"
                  >
                    삭제
                  </button>
                </div>
              </div>

              <div className="font-bold">
                ${(item.price * item.quantity).toFixed(2)}
              </div>
            </div>
          ))}
        </div>

        {/* 주문 요약 */}
        <div className="border rounded-lg p-6 h-fit">
          <h2 className="text-xl font-bold mb-4">주문 요약</h2>

          <div className="space-y-2 mb-4">
            <div className="flex justify-between">
              <span>상품 금액</span>
              <span>${total().toFixed(2)}</span>
            </div>
            <div className="flex justify-between">
              <span>배송비</span>
              <span>$5.00</span>
            </div>
            <div className="border-t pt-2 flex justify-between font-bold text-lg">
              <span>총 금액</span>
              <span>${(total() + 5).toFixed(2)}</span>
            </div>
          </div>

          <Link
            href="/checkout"
            className="block w-full px-6 py-3 bg-blue-600 text-white text-center rounded-lg hover:bg-blue-700 transition"
          >
            결제하기
          </Link>
        </div>
      </div>
    </div>
  );
}
```

## 5. Stripe 결제

### 5.1 Stripe 설정

```typescript
// lib/stripe.ts
import Stripe from 'stripe';

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-11-20.acacia'
});
```

### 5.2 Checkout Session 생성

```typescript
// app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe';
import { prisma } from '@/lib/prisma';

export async function POST(req: NextRequest) {
  try {
    const { items, shippingAddress } = await req.json();

    // 가격 검증 (서버에서 다시 확인)
    const productIds = items.map((item: any) => item.id);
    const products = await prisma.product.findMany({
      where: { id: { in: productIds } }
    });

    const lineItems = items.map((item: any) => {
      const product = products.find(p => p.id === item.id);

      if (!product) {
        throw new Error(`Product ${item.id} not found`);
      }

      return {
        price_data: {
          currency: 'usd',
          product_data: {
            name: product.name,
            images: [product.image]
          },
          unit_amount: Math.round(parseFloat(product.price.toString()) * 100)
        },
        quantity: item.quantity
      };
    });

    // Stripe Checkout Session 생성
    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      line_items: lineItems,
      mode: 'payment',
      success_url: `${process.env.NEXT_PUBLIC_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.NEXT_PUBLIC_URL}/cart`,
      metadata: {
        items: JSON.stringify(items),
        shippingAddress: JSON.stringify(shippingAddress)
      }
    });

    return NextResponse.json({ sessionId: session.id });
  } catch (error: any) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }
}
```

### 5.3 Checkout 페이지

```tsx
// app/checkout/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { useCartStore } from '@/store/cart';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);

export default function CheckoutPage() {
  const router = useRouter();
  const { items, total } = useCartStore();

  const [shippingAddress, setShippingAddress] = useState({
    name: '',
    address: '',
    city: '',
    zipCode: '',
    country: ''
  });

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    const stripe = await stripePromise;

    if (!stripe) {
      alert('Stripe 로딩 실패');
      return;
    }

    const res = await fetch('/api/checkout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        items,
        shippingAddress
      })
    });

    const { sessionId, error } = await res.json();

    if (error) {
      alert(error);
      return;
    }

    const result = await stripe.redirectToCheckout({ sessionId });

    if (result.error) {
      alert(result.error.message);
    }
  }

  return (
    <div className="max-w-2xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">결제</h1>

      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label className="block mb-2 font-medium">이름</label>
          <input
            type="text"
            value={shippingAddress.name}
            onChange={e =>
              setShippingAddress({ ...shippingAddress, name: e.target.value })
            }
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div>
          <label className="block mb-2 font-medium">주소</label>
          <input
            type="text"
            value={shippingAddress.address}
            onChange={e =>
              setShippingAddress({ ...shippingAddress, address: e.target.value })
            }
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div className="grid grid-cols-2 gap-4">
          <div>
            <label className="block mb-2 font-medium">도시</label>
            <input
              type="text"
              value={shippingAddress.city}
              onChange={e =>
                setShippingAddress({ ...shippingAddress, city: e.target.value })
              }
              required
              className="w-full px-4 py-2 border rounded"
            />
          </div>

          <div>
            <label className="block mb-2 font-medium">우편번호</label>
            <input
              type="text"
              value={shippingAddress.zipCode}
              onChange={e =>
                setShippingAddress({ ...shippingAddress, zipCode: e.target.value })
              }
              required
              className="w-full px-4 py-2 border rounded"
            />
          </div>
        </div>

        <div>
          <label className="block mb-2 font-medium">국가</label>
          <input
            type="text"
            value={shippingAddress.country}
            onChange={e =>
              setShippingAddress({ ...shippingAddress, country: e.target.value })
            }
            required
            className="w-full px-4 py-2 border rounded"
          />
        </div>

        <div className="border-t pt-4">
          <div className="flex justify-between mb-2">
            <span>상품 금액:</span>
            <span>${total().toFixed(2)}</span>
          </div>
          <div className="flex justify-between mb-2">
            <span>배송비:</span>
            <span>$5.00</span>
          </div>
          <div className="flex justify-between font-bold text-lg">
            <span>총 금액:</span>
            <span>${(total() + 5).toFixed(2)}</span>
          </div>
        </div>

        <button
          type="submit"
          className="w-full px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
        >
          Stripe로 결제하기
        </button>
      </form>
    </div>
  );
}
```

### 5.4 Webhook (주문 생성)

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe';
import { prisma } from '@/lib/prisma';
import { headers } from 'next/headers';

export async function POST(req: NextRequest) {
  const body = await req.text();
  const signature = headers().get('stripe-signature')!;

  let event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (error: any) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;

    const items = JSON.parse(session.metadata!.items);
    const shippingAddress = JSON.parse(session.metadata!.shippingAddress);

    // 주문 생성
    const order = await prisma.order.create({
      data: {
        userId: 'guest', // 실제로는 인증된 사용자 ID
        status: 'paid',
        total: session.amount_total! / 100,
        stripeSessionId: session.id,
        shippingAddress,
        items: {
          create: items.map((item: any) => ({
            productId: item.id,
            quantity: item.quantity,
            price: item.price
          }))
        }
      }
    });

    // 재고 차감
    for (const item of items) {
      await prisma.product.update({
        where: { id: item.id },
        data: {
          stock: {
            decrement: item.quantity
          }
        }
      });
    }
  }

  return NextResponse.json({ received: true });
}
```

## 6. 주문 내역

```tsx
// app/orders/page.tsx
import { prisma } from '@/lib/prisma';
import Link from 'next/link';

export default async function OrdersPage() {
  const orders = await prisma.order.findMany({
    include: {
      items: {
        include: { product: true }
      }
    },
    orderBy: { createdAt: 'desc' }
  });

  return (
    <div className="max-w-7xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">주문 내역</h1>

      <div className="space-y-4">
        {orders.map(order => (
          <div key={order.id} className="border rounded-lg p-6">
            <div className="flex justify-between items-start mb-4">
              <div>
                <h2 className="font-bold text-lg">
                  주문 #{order.id.slice(0, 8)}
                </h2>
                <p className="text-sm text-gray-600">
                  {new Date(order.createdAt).toLocaleDateString()}
                </p>
              </div>
              <span
                className={`px-3 py-1 rounded ${
                  order.status === 'paid'
                    ? 'bg-green-100 text-green-800'
                    : 'bg-gray-100 text-gray-800'
                }`}
              >
                {order.status}
              </span>
            </div>

            <div className="space-y-2 mb-4">
              {order.items.map(item => (
                <div key={item.id} className="flex justify-between">
                  <span>
                    {item.product.name} x {item.quantity}
                  </span>
                  <span>${(parseFloat(item.price.toString()) * item.quantity).toFixed(2)}</span>
                </div>
              ))}
            </div>

            <div className="border-t pt-4 flex justify-between font-bold">
              <span>총 금액</span>
              <span>${order.total.toString()}</span>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 7. 관리자 대시보드

```tsx
// app/admin/page.tsx
import { prisma } from '@/lib/prisma';
import Link from 'next/link';

export default async function AdminPage() {
  const stats = await Promise.all([
    prisma.product.count(),
    prisma.order.count(),
    prisma.order.aggregate({
      _sum: { total: true }
    })
  ]);

  const [productCount, orderCount, revenue] = stats;

  return (
    <div className="max-w-7xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-6">관리자 대시보드</h1>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
        <div className="border rounded-lg p-6">
          <h2 className="text-gray-600 mb-2">총 상품</h2>
          <p className="text-3xl font-bold">{productCount}</p>
        </div>

        <div className="border rounded-lg p-6">
          <h2 className="text-gray-600 mb-2">총 주문</h2>
          <p className="text-3xl font-bold">{orderCount}</p>
        </div>

        <div className="border rounded-lg p-6">
          <h2 className="text-gray-600 mb-2">총 매출</h2>
          <p className="text-3xl font-bold">
            ${revenue._sum.total?.toString() || '0'}
          </p>
        </div>
      </div>

      <div className="flex gap-4">
        <Link
          href="/admin/products"
          className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        >
          상품 관리
        </Link>
        <Link
          href="/admin/orders"
          className="px-6 py-3 bg-green-600 text-white rounded-lg hover:bg-green-700"
        >
          주문 관리
        </Link>
      </div>
    </div>
  );
}
```

## 핵심 요약

1. **Prisma**: 타입 안전 데이터베이스 ORM
2. **Zustand**: 간단한 상태 관리 (장바구니)
3. **Stripe**: 안전한 결제 처리
4. **Webhook**: 서버 측 주문 처리
5. **이미지 최적화**: Next.js Image 컴포넌트

[← Chapter 26: 블로그 플랫폼](./chapter26-blog-platform.md) | [처음으로 →](../../README.md)