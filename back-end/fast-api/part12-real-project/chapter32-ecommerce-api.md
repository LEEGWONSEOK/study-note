# Chapter 32. E-Commerce API 만들기

**목표**: 주문, 결제, 재고 관리가 포함된 실전 이커머스 API 구축

---

## 32.1 프로젝트 개요

### 32.1.1 요구사항 정의

**기능 요구사항**:
- 상품 관리 (CRUD, 카테고리, 재고)
- 장바구니 관리
- 주문 처리 (생성, 취소, 상태 변경)
- 결제 연동 (시뮬레이션)
- 재고 차감 및 동시성 제어
- 주문 히스토리
- 상품 검색 및 필터링

**Spring Boot와 비교**:
```
Spring Boot                     FastAPI
───────────────────────────────────────────────
@Transactional                  트랜잭션 컨텍스트 매니저
Pessimistic Locking             SQLAlchemy with_for_update()
@Async                          BackgroundTasks
@EventListener                  의존성 주입 + 콜백
@Scheduled                      Celery
JPA Auditing                    SQLAlchemy onupdate
```

---

### 32.1.2 프로젝트 구조

```
ecommerce-api/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── product.py
│   │   ├── category.py
│   │   ├── cart.py
│   │   ├── order.py
│   │   └── payment.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── product.py
│   │   ├── cart.py
│   │   ├── order.py
│   │   └── payment.py
│   │
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── products.py
│   │   ├── carts.py
│   │   ├── orders.py
│   │   └── payments.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── product_service.py
│   │   ├── cart_service.py
│   │   ├── order_service.py
│   │   ├── payment_service.py
│   │   └── inventory_service.py
│   │
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── product_repo.py
│   │   ├── cart_repo.py
│   │   └── order_repo.py
│   │
│   └── utils/
│       ├── __init__.py
│       ├── enums.py
│       └── exceptions.py
│
├── alembic/
├── tests/
├── .env
└── requirements.txt
```

---

## 32.2 데이터베이스 설계

### 32.2.1 ERD

```
┌──────────────┐         ┌──────────────┐
│   Category   │         │   Product    │
├──────────────┤         ├──────────────┤
│ id (PK)      │<───────<│ category_id  │
│ name         │         │ id (PK)      │
│ description  │         │ name         │
└──────────────┘         │ description  │
                         │ price        │
                         │ stock        │       ┌──────────────┐
                         │ is_active    │       │   CartItem   │
                         └──────────────┘       ├──────────────┤
                                 ^              │ id (PK)      │
                                 │              │ cart_id (FK) │
                         ┌───────┴───────┐      │ product_id   │
                         │               │      │ quantity     │
┌──────────────┐         │               │      │ price        │
│     User     │         │               │      └──────────────┘
├──────────────┤         │               │              ^
│ id (PK)      │────┬───>│               │              │
│ email        │    │    │               │      ┌───────┴──────┐
│ username     │    │    │               │      │     Cart     │
└──────────────┘    │    │               │      ├──────────────┤
                    │    v               v      │ id (PK)      │
                    │ ┌──────────────┐          │ user_id (FK) │
                    │ │  OrderItem   │          │ created_at   │
                    │ ├──────────────┤          │ updated_at   │
                    │ │ id (PK)      │          └──────────────┘
                    │ │ order_id (FK)│
                    │ │ product_id   │
                    │ │ quantity     │
                    │ │ price        │
                    │ └──────────────┘
                    │         ^
                    │         │
                    │ ┌───────┴──────┐       ┌──────────────┐
                    │ │    Order     │       │   Payment    │
                    │ ├──────────────┤       ├──────────────┤
                    └>│ user_id (FK) │<─────<│ order_id (FK)│
                      │ id (PK)      │       │ id (PK)      │
                      │ status       │       │ amount       │
                      │ total_amount │       │ method       │
                      │ created_at   │       │ status       │
                      └──────────────┘       │ created_at   │
                                             └──────────────┘
```

---

## 32.3 모델 정의

### 32.3.1 Enums 정의

**utils/enums.py**:
```python
from enum import Enum

class OrderStatus(str, Enum):
    """주문 상태"""
    PENDING = "pending"           # 대기중
    CONFIRMED = "confirmed"       # 확인됨
    PROCESSING = "processing"     # 처리중
    SHIPPED = "shipped"           # 배송중
    DELIVERED = "delivered"       # 배송완료
    CANCELLED = "cancelled"       # 취소됨

class PaymentStatus(str, Enum):
    """결제 상태"""
    PENDING = "pending"           # 대기중
    COMPLETED = "completed"       # 완료
    FAILED = "failed"             # 실패
    REFUNDED = "refunded"         # 환불

class PaymentMethod(str, Enum):
    """결제 수단"""
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    PAYPAL = "paypal"
    BANK_TRANSFER = "bank_transfer"
```

---

### 32.3.2 상품 및 카테고리 모델

**models/category.py**:
```python
from sqlalchemy import Column, Integer, String, Text
from sqlalchemy.orm import relationship
from app.database import Base

class Category(Base):
    __tablename__ = "categories"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, nullable=False, index=True)
    description = Column(Text, nullable=True)

    # Relationships
    products = relationship("Product", back_populates="category")
```

**models/product.py**:
```python
from sqlalchemy import Column, Integer, String, Text, Numeric, Boolean, ForeignKey, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(200), nullable=False, index=True)
    description = Column(Text, nullable=True)
    price = Column(Numeric(10, 2), nullable=False)
    stock = Column(Integer, default=0, nullable=False)  # 재고
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    category_id = Column(Integer, ForeignKey("categories.id"), nullable=False)

    # Relationships
    category = relationship("Category", back_populates="products")
    cart_items = relationship("CartItem", back_populates="product")
    order_items = relationship("OrderItem", back_populates="product")
```

**Spring Boot 비교**:
```java
// Spring Boot
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private BigDecimal price;
    private Integer stock;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @Version  // Optimistic Locking
    private Long version;
}
```

---

### 32.3.3 장바구니 모델

**models/cart.py**:
```python
from sqlalchemy import Column, Integer, ForeignKey, DateTime, Numeric
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class Cart(Base):
    __tablename__ = "carts"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    user = relationship("User", back_populates="cart")
    items = relationship("CartItem", back_populates="cart", cascade="all, delete-orphan")

class CartItem(Base):
    __tablename__ = "cart_items"

    id = Column(Integer, primary_key=True, index=True)
    cart_id = Column(Integer, ForeignKey("carts.id", ondelete="CASCADE"), nullable=False)
    product_id = Column(Integer, ForeignKey("products.id", ondelete="CASCADE"), nullable=False)
    quantity = Column(Integer, default=1, nullable=False)
    price = Column(Numeric(10, 2), nullable=False)  # 담을 당시의 가격

    # Relationships
    cart = relationship("Cart", back_populates="items")
    product = relationship("Product", back_populates="cart_items")
```

---

### 32.3.4 주문 및 결제 모델

**models/order.py**:
```python
from sqlalchemy import Column, Integer, String, Numeric, ForeignKey, DateTime, Enum as SQLEnum
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base
from app.utils.enums import OrderStatus

class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    status = Column(SQLEnum(OrderStatus), default=OrderStatus.PENDING, nullable=False)
    total_amount = Column(Numeric(10, 2), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    user = relationship("User", back_populates="orders")
    items = relationship("OrderItem", back_populates="order", cascade="all, delete-orphan")
    payment = relationship("Payment", back_populates="order", uselist=False)

class OrderItem(Base):
    __tablename__ = "order_items"

    id = Column(Integer, primary_key=True, index=True)
    order_id = Column(Integer, ForeignKey("orders.id", ondelete="CASCADE"), nullable=False)
    product_id = Column(Integer, ForeignKey("products.id"), nullable=False)
    quantity = Column(Integer, nullable=False)
    price = Column(Numeric(10, 2), nullable=False)  # 주문 당시 가격

    # Relationships
    order = relationship("Order", back_populates="items")
    product = relationship("Product", back_populates="order_items")
```

**models/payment.py**:
```python
from sqlalchemy import Column, Integer, String, Numeric, ForeignKey, DateTime, Enum as SQLEnum
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base
from app.utils.enums import PaymentStatus, PaymentMethod

class Payment(Base):
    __tablename__ = "payments"

    id = Column(Integer, primary_key=True, index=True)
    order_id = Column(Integer, ForeignKey("orders.id", ondelete="CASCADE"), unique=True, nullable=False)
    amount = Column(Numeric(10, 2), nullable=False)
    method = Column(SQLEnum(PaymentMethod), nullable=False)
    status = Column(SQLEnum(PaymentStatus), default=PaymentStatus.PENDING, nullable=False)
    transaction_id = Column(String(100), nullable=True)  # 외부 결제 시스템 ID
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    order = relationship("Order", back_populates="payment")
```

---

## 32.4 스키마 정의

### 32.4.1 상품 스키마

**schemas/product.py**:
```python
from pydantic import BaseModel, Field
from decimal import Decimal
from typing import Optional
from datetime import datetime

class CategoryResponse(BaseModel):
    """카테고리 응답"""
    id: int
    name: str
    description: Optional[str] = None

    class Config:
        from_attributes = True

class ProductCreate(BaseModel):
    """상품 생성 요청"""
    name: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    price: Decimal = Field(..., gt=0)
    stock: int = Field(..., ge=0)
    category_id: int

class ProductUpdate(BaseModel):
    """상품 수정 요청"""
    name: Optional[str] = Field(None, min_length=1, max_length=200)
    description: Optional[str] = None
    price: Optional[Decimal] = Field(None, gt=0)
    stock: Optional[int] = Field(None, ge=0)
    is_active: Optional[bool] = None
    category_id: Optional[int] = None

class ProductResponse(BaseModel):
    """상품 응답"""
    id: int
    name: str
    description: Optional[str]
    price: Decimal
    stock: int
    is_active: bool
    category: CategoryResponse
    created_at: datetime

    class Config:
        from_attributes = True
```

---

### 32.4.2 장바구니 스키마

**schemas/cart.py**:
```python
from pydantic import BaseModel, Field
from decimal import Decimal
from typing import List

class CartItemCreate(BaseModel):
    """장바구니 아이템 추가"""
    product_id: int
    quantity: int = Field(..., ge=1)

class CartItemUpdate(BaseModel):
    """장바구니 아이템 수량 변경"""
    quantity: int = Field(..., ge=1)

class CartItemResponse(BaseModel):
    """장바구니 아이템 응답"""
    id: int
    product_id: int
    product_name: str
    quantity: int
    price: Decimal
    subtotal: Decimal  # price * quantity

    class Config:
        from_attributes = True

class CartResponse(BaseModel):
    """장바구니 응답"""
    id: int
    items: List[CartItemResponse]
    total_amount: Decimal

    class Config:
        from_attributes = True
```

---

### 32.4.3 주문 스키마

**schemas/order.py**:
```python
from pydantic import BaseModel
from decimal import Decimal
from typing import List
from datetime import datetime
from app.utils.enums import OrderStatus, PaymentMethod

class OrderItemResponse(BaseModel):
    """주문 아이템 응답"""
    id: int
    product_id: int
    product_name: str
    quantity: int
    price: Decimal
    subtotal: Decimal

    class Config:
        from_attributes = True

class OrderCreate(BaseModel):
    """주문 생성 요청"""
    payment_method: PaymentMethod

class OrderResponse(BaseModel):
    """주문 응답"""
    id: int
    status: OrderStatus
    total_amount: Decimal
    created_at: datetime
    items: List[OrderItemResponse]

    class Config:
        from_attributes = True

class OrderDetail(OrderResponse):
    """주문 상세 (결제 정보 포함)"""
    payment_status: str
    payment_method: str
    transaction_id: str | None
```

---

## 32.5 재고 관리 서비스

### 32.5.1 동시성 제어를 위한 재고 서비스

**services/inventory_service.py**:
```python
from sqlalchemy.orm import Session
from sqlalchemy import select
from fastapi import HTTPException, status
from app.models.product import Product
from typing import List, Tuple

class InventoryService:
    """재고 관리 서비스 (동시성 제어)"""

    def __init__(self, db: Session):
        self.db = db

    def check_availability(self, product_id: int, quantity: int) -> bool:
        """재고 확인"""
        product = self.db.query(Product).filter(Product.id == product_id).first()
        if not product:
            return False
        return product.stock >= quantity

    def reserve_stock(self, items: List[Tuple[int, int]]) -> None:
        """
        재고 차감 (비관적 락 사용)
        Spring Boot의 @Lock(LockModeType.PESSIMISTIC_WRITE)와 동일

        items: [(product_id, quantity), ...]
        """
        for product_id, quantity in items:
            # SELECT ... FOR UPDATE (비관적 락)
            product = (
                self.db.query(Product)
                .filter(Product.id == product_id)
                .with_for_update()  # Pessimistic Lock
                .first()
            )

            if not product:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail=f"Product {product_id} not found"
                )

            if not product.is_active:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail=f"Product {product.name} is not available"
                )

            if product.stock < quantity:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail=f"Insufficient stock for {product.name}. Available: {product.stock}"
                )

            # 재고 차감
            product.stock -= quantity

        # 커밋은 호출자(주문 서비스)에서 수행
        self.db.flush()

    def release_stock(self, items: List[Tuple[int, int]]) -> None:
        """재고 복구 (주문 취소시)"""
        for product_id, quantity in items:
            product = (
                self.db.query(Product)
                .filter(Product.id == product_id)
                .with_for_update()
                .first()
            )

            if product:
                product.stock += quantity

        self.db.flush()
```

**Spring Boot 비교**:
```java
// Spring Boot - Pessimistic Lock
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLock(@Param("id") Long id);

@Transactional
public void reserveStock(Long productId, int quantity) {
    Product product = productRepository.findByIdWithLock(productId)
        .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

    if (product.getStock() < quantity) {
        throw new InsufficientStockException("Insufficient stock");
    }

    product.setStock(product.getStock() - quantity);
}
```

---

## 32.6 주문 서비스

### 32.6.1 트랜잭션 처리를 포함한 주문 서비스

**services/order_service.py**:
```python
from sqlalchemy.orm import Session
from sqlalchemy.exc import SQLAlchemyError
from fastapi import HTTPException, status
from decimal import Decimal
from typing import List
from app.models.order import Order, OrderItem
from app.models.cart import Cart
from app.models.payment import Payment
from app.models.product import Product
from app.schemas.order import OrderCreate, OrderResponse, OrderDetail
from app.utils.enums import OrderStatus, PaymentStatus
from app.services.inventory_service import InventoryService
from app.services.payment_service import PaymentService

class OrderService:
    """주문 서비스 (Spring @Transactional과 유사한 트랜잭션 처리)"""

    def __init__(self, db: Session):
        self.db = db
        self.inventory_service = InventoryService(db)
        self.payment_service = PaymentService(db)

    def create_order(self, order_data: OrderCreate, user_id: int) -> OrderResponse:
        """
        주문 생성 (트랜잭션 처리)

        1. 장바구니 조회
        2. 재고 확인 및 차감 (락 걸림)
        3. 주문 생성
        4. 결제 처리
        5. 장바구니 비우기
        """
        try:
            # 1. 장바구니 조회
            cart = self.db.query(Cart).filter(Cart.user_id == user_id).first()
            if not cart or not cart.items:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Cart is empty"
                )

            # 2. 재고 확인 및 차감
            items_to_reserve = [(item.product_id, item.quantity) for item in cart.items]
            self.inventory_service.reserve_stock(items_to_reserve)

            # 3. 주문 생성
            total_amount = sum(item.price * item.quantity for item in cart.items)

            order = Order(
                user_id=user_id,
                status=OrderStatus.PENDING,
                total_amount=total_amount
            )
            self.db.add(order)
            self.db.flush()  # order.id 생성

            # 주문 아이템 생성
            for cart_item in cart.items:
                order_item = OrderItem(
                    order_id=order.id,
                    product_id=cart_item.product_id,
                    quantity=cart_item.quantity,
                    price=cart_item.price
                )
                self.db.add(order_item)

            # 4. 결제 처리
            payment = self.payment_service.process_payment(
                order_id=order.id,
                amount=total_amount,
                method=order_data.payment_method
            )

            if payment.status == PaymentStatus.COMPLETED:
                order.status = OrderStatus.CONFIRMED
            else:
                # 결제 실패시 재고 복구
                self.inventory_service.release_stock(items_to_reserve)
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Payment failed"
                )

            # 5. 장바구니 비우기
            for item in cart.items:
                self.db.delete(item)

            # 커밋
            self.db.commit()
            self.db.refresh(order)

            return self._to_order_response(order)

        except SQLAlchemyError as e:
            self.db.rollback()
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail=f"Order creation failed: {str(e)}"
            )

    def get_order(self, order_id: int, user_id: int) -> OrderDetail:
        """주문 상세 조회"""
        order = (
            self.db.query(Order)
            .filter(Order.id == order_id, Order.user_id == user_id)
            .first()
        )

        if not order:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Order not found"
            )

        return OrderDetail(
            id=order.id,
            status=order.status,
            total_amount=order.total_amount,
            created_at=order.created_at,
            items=[
                {
                    "id": item.id,
                    "product_id": item.product_id,
                    "product_name": item.product.name,
                    "quantity": item.quantity,
                    "price": item.price,
                    "subtotal": item.price * item.quantity
                }
                for item in order.items
            ],
            payment_status=order.payment.status.value if order.payment else "none",
            payment_method=order.payment.method.value if order.payment else "none",
            transaction_id=order.payment.transaction_id if order.payment else None
        )

    def cancel_order(self, order_id: int, user_id: int) -> OrderResponse:
        """주문 취소 (재고 복구)"""
        try:
            order = (
                self.db.query(Order)
                .filter(Order.id == order_id, Order.user_id == user_id)
                .first()
            )

            if not order:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail="Order not found"
                )

            # 취소 가능한 상태 확인
            if order.status not in [OrderStatus.PENDING, OrderStatus.CONFIRMED]:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail=f"Cannot cancel order with status {order.status}"
                )

            # 재고 복구
            items_to_release = [(item.product_id, item.quantity) for item in order.items]
            self.inventory_service.release_stock(items_to_release)

            # 결제 환불 처리
            if order.payment and order.payment.status == PaymentStatus.COMPLETED:
                self.payment_service.refund_payment(order.payment.id)

            # 주문 상태 변경
            order.status = OrderStatus.CANCELLED

            self.db.commit()
            self.db.refresh(order)

            return self._to_order_response(order)

        except SQLAlchemyError as e:
            self.db.rollback()
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail=f"Order cancellation failed: {str(e)}"
            )

    def _to_order_response(self, order: Order) -> OrderResponse:
        """Order 엔티티를 OrderResponse로 변환"""
        return OrderResponse(
            id=order.id,
            status=order.status,
            total_amount=order.total_amount,
            created_at=order.created_at,
            items=[
                {
                    "id": item.id,
                    "product_id": item.product_id,
                    "product_name": item.product.name,
                    "quantity": item.quantity,
                    "price": item.price,
                    "subtotal": item.price * item.quantity
                }
                for item in order.items
            ]
        )
```

**Spring Boot 비교**:
```java
// Spring Boot
@Service
@Transactional
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private PaymentService paymentService;

    public OrderResponse createOrder(OrderCreateRequest request, Long userId) {
        try {
            // 1. 장바구니 조회
            Cart cart = cartRepository.findByUserId(userId)
                .orElseThrow(() -> new EmptyCartException());

            // 2. 재고 차감
            inventoryService.reserveStock(cart.getItems());

            // 3. 주문 생성
            Order order = new Order();
            order.setUserId(userId);
            order.setStatus(OrderStatus.PENDING);
            order.setTotalAmount(cart.getTotalAmount());
            order = orderRepository.save(order);

            // 4. 결제 처리
            Payment payment = paymentService.processPayment(order, request.getPaymentMethod());

            if (payment.getStatus() == PaymentStatus.COMPLETED) {
                order.setStatus(OrderStatus.CONFIRMED);
            } else {
                throw new PaymentFailedException();
            }

            return convertToResponse(order);

        } catch (Exception e) {
            // 롤백 자동 처리
            throw e;
        }
    }
}
```

---

## 32.7 결제 서비스

**services/payment_service.py**:
```python
from sqlalchemy.orm import Session
from app.models.payment import Payment
from app.utils.enums import PaymentStatus, PaymentMethod
from decimal import Decimal
import uuid

class PaymentService:
    """결제 서비스 (실제로는 외부 PG사 연동)"""

    def __init__(self, db: Session):
        self.db = db

    def process_payment(
        self,
        order_id: int,
        amount: Decimal,
        method: PaymentMethod
    ) -> Payment:
        """
        결제 처리 (시뮬레이션)
        실제로는 PG사 API 호출
        """
        # 결제 시뮬레이션
        transaction_id = f"TXN-{uuid.uuid4().hex[:12].upper()}"

        payment = Payment(
            order_id=order_id,
            amount=amount,
            method=method,
            status=PaymentStatus.COMPLETED,  # 시뮬레이션이므로 항상 성공
            transaction_id=transaction_id
        )

        self.db.add(payment)
        self.db.flush()

        return payment

    def refund_payment(self, payment_id: int) -> Payment:
        """결제 환불"""
        payment = self.db.query(Payment).filter(Payment.id == payment_id).first()

        if payment:
            payment.status = PaymentStatus.REFUNDED
            self.db.flush()

        return payment
```

---

## 32.8 Router (Controller) 구현

### 32.8.1 주문 Router

**routers/orders.py**:
```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session
from typing import List
from app.database import get_db
from app.services.order_service import OrderService
from app.schemas.order import OrderCreate, OrderResponse, OrderDetail
from app.utils.dependencies import get_current_user
from app.models.user import User

router = APIRouter(prefix="/orders", tags=["orders"])

def get_order_service(db: Session = Depends(get_db)) -> OrderService:
    return OrderService(db)

@router.post(
    "/",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
    summary="주문 생성"
)
def create_order(
    order_data: OrderCreate,
    current_user: User = Depends(get_current_user),
    service: OrderService = Depends(get_order_service)
):
    """
    주문 생성

    - 장바구니의 모든 아이템을 주문으로 변환
    - 재고 차감 및 결제 처리
    - 트랜잭션으로 처리되어 실패시 자동 롤백
    """
    return service.create_order(order_data, current_user.id)

@router.get(
    "/{order_id}",
    response_model=OrderDetail,
    summary="주문 상세 조회"
)
def get_order(
    order_id: int,
    current_user: User = Depends(get_current_user),
    service: OrderService = Depends(get_order_service)
):
    """주문 상세 조회 (결제 정보 포함)"""
    return service.get_order(order_id, current_user.id)

@router.post(
    "/{order_id}/cancel",
    response_model=OrderResponse,
    summary="주문 취소"
)
def cancel_order(
    order_id: int,
    current_user: User = Depends(get_current_user),
    service: OrderService = Depends(get_order_service)
):
    """
    주문 취소

    - 재고 복구
    - 결제 환불 처리
    """
    return service.cancel_order(order_id, current_user.id)

@router.get(
    "/",
    response_model=List[OrderResponse],
    summary="내 주문 목록"
)
def get_my_orders(
    current_user: User = Depends(get_current_user),
    service: OrderService = Depends(get_order_service)
):
    """내 주문 목록 조회"""
    return service.get_user_orders(current_user.id)
```

---

### 32.8.2 장바구니 Router

**routers/carts.py**:
```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session
from app.database import get_db
from app.services.cart_service import CartService
from app.schemas.cart import CartItemCreate, CartItemUpdate, CartResponse
from app.utils.dependencies import get_current_user
from app.models.user import User

router = APIRouter(prefix="/cart", tags=["cart"])

def get_cart_service(db: Session = Depends(get_db)) -> CartService:
    return CartService(db)

@router.get(
    "/",
    response_model=CartResponse,
    summary="장바구니 조회"
)
def get_cart(
    current_user: User = Depends(get_current_user),
    service: CartService = Depends(get_cart_service)
):
    """내 장바구니 조회"""
    return service.get_cart(current_user.id)

@router.post(
    "/items",
    status_code=status.HTTP_201_CREATED,
    summary="장바구니에 상품 추가"
)
def add_item(
    item: CartItemCreate,
    current_user: User = Depends(get_current_user),
    service: CartService = Depends(get_cart_service)
):
    """장바구니에 상품 추가"""
    return service.add_item(current_user.id, item)

@router.put(
    "/items/{item_id}",
    summary="장바구니 아이템 수량 변경"
)
def update_item(
    item_id: int,
    item_update: CartItemUpdate,
    current_user: User = Depends(get_current_user),
    service: CartService = Depends(get_cart_service)
):
    """장바구니 아이템 수량 변경"""
    return service.update_item(current_user.id, item_id, item_update)

@router.delete(
    "/items/{item_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="장바구니 아이템 제거"
)
def remove_item(
    item_id: int,
    current_user: User = Depends(get_current_user),
    service: CartService = Depends(get_cart_service)
):
    """장바구니에서 아이템 제거"""
    service.remove_item(current_user.id, item_id)

@router.delete(
    "/",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="장바구니 비우기"
)
def clear_cart(
    current_user: User = Depends(get_current_user),
    service: CartService = Depends(get_cart_service)
):
    """장바구니 전체 비우기"""
    service.clear_cart(current_user.id)
```

---

## 32.9 테스트

**tests/test_orders.py**:
```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_order_flow():
    """주문 생성 플로우 테스트"""
    # 1. 회원가입 및 로그인
    client.post("/auth/register", json={
        "email": "buyer@example.com",
        "username": "buyer",
        "password": "password123"
    })
    login_response = client.post("/auth/login", data={
        "username": "buyer@example.com",
        "password": "password123"
    })
    token = login_response.json()["access_token"]
    headers = {"Authorization": f"Bearer {token}"}

    # 2. 상품 생성 (관리자)
    product_response = client.post(
        "/products/",
        json={
            "name": "Test Product",
            "price": 100.00,
            "stock": 10,
            "category_id": 1
        },
        headers=headers
    )
    product_id = product_response.json()["id"]

    # 3. 장바구니에 추가
    client.post(
        "/cart/items",
        json={"product_id": product_id, "quantity": 2},
        headers=headers
    )

    # 4. 주문 생성
    order_response = client.post(
        "/orders/",
        json={"payment_method": "credit_card"},
        headers=headers
    )

    assert order_response.status_code == 201
    order = order_response.json()
    assert order["status"] == "confirmed"
    assert len(order["items"]) == 1
    assert order["total_amount"] == "200.00"

    # 5. 재고 확인
    product_response = client.get(f"/products/{product_id}")
    assert product_response.json()["stock"] == 8  # 10 - 2
```

---

## 32.10 정리

### 32.10.1 핵심 개념 요약

| 개념 | Spring Boot | FastAPI |
|------|-------------|---------|
| **트랜잭션** | `@Transactional` | try-except + commit/rollback |
| **비관적 락** | `@Lock(PESSIMISTIC_WRITE)` | `with_for_update()` |
| **낙관적 락** | `@Version` | SQLAlchemy `version_id_col` |
| **Enum** | Java Enum | Python Enum (str 상속) |
| **결제 처리** | 외부 API 호출 | 동일 (httpx 사용) |
| **백그라운드 작업** | `@Async` | `BackgroundTasks` or Celery |
| **이벤트** | `@EventListener` | 콜백 함수 |

---

### 32.10.2 주요 학습 포인트

1. **트랜잭션 관리**: 명시적 commit/rollback으로 ACID 보장
2. **동시성 제어**: `with_for_update()`를 사용한 비관적 락
3. **재고 관리**: 락을 통한 동시 주문 처리
4. **결제 연동**: 외부 API 시뮬레이션 및 에러 처리
5. **복잡한 비즈니스 로직**: 주문-결제-재고 연계 처리
6. **예외 처리**: 실패시 일관성 유지 (롤백, 재고 복구)

---

### 32.10.3 실전 고려사항

**성능 최적화**:
- 데이터베이스 인덱스 설정
- N+1 쿼리 방지 (joinedload)
- 캐싱 (Redis)

**보안**:
- 가격 조작 방지 (서버에서 가격 재조회)
- 권한 체크 (내 주문만 조회/취소)
- Rate Limiting (무분별한 주문 방지)

**확장성**:
- Celery를 사용한 비동기 처리 (이메일, 알림)
- 이벤트 기반 아키텍처 (Kafka, RabbitMQ)
- 마이크로서비스로 분리 (주문, 결제, 재고)

---

## 다음 단계

- 부록에서 FastAPI vs Spring Boot 완전 비교
- 실전 배포 및 운영 팁
- 더 고급 패턴 (CQRS, Event Sourcing)
