# Chapter 25: 디자인 패턴 (Design Patterns)

## 학습 목표
- FastAPI에서 유용한 디자인 패턴 학습
- Repository 패턴 구현
- Service 패턴과 비즈니스 로직 분리
- Factory 패턴으로 객체 생성 관리
- Dependency Injection 심화
- Spring의 디자인 패턴과 비교

## 1. Repository 패턴

### 1.1 Repository 패턴이란?

데이터 접근 로직을 캡슐화하여 비즈니스 로직과 분리합니다.

**장점:**
- 데이터 소스 변경 용이 (PostgreSQL → MongoDB)
- 테스트 시 Mock 구현 가능
- 비즈니스 로직과 데이터 접근 분리

### 1.2 기본 Repository 구현

```python
# app/repositories/base.py
from typing import Generic, TypeVar, Type, List, Optional
from sqlalchemy.orm import Session
from app.core.database import Base

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(Generic[ModelType]):
    """기본 Repository"""

    def __init__(self, model: Type[ModelType], db: Session):
        self.model = model
        self.db = db

    def get_by_id(self, id: int) -> Optional[ModelType]:
        """ID로 조회"""
        return self.db.query(self.model).filter(self.model.id == id).first()

    def get_all(self, skip: int = 0, limit: int = 100) -> List[ModelType]:
        """전체 조회"""
        return self.db.query(self.model).offset(skip).limit(limit).all()

    def create(self, obj_in: dict) -> ModelType:
        """생성"""
        db_obj = self.model(**obj_in)
        self.db.add(db_obj)
        self.db.commit()
        self.db.refresh(db_obj)
        return db_obj

    def update(self, id: int, obj_in: dict) -> Optional[ModelType]:
        """업데이트"""
        db_obj = self.get_by_id(id)
        if not db_obj:
            return None

        for key, value in obj_in.items():
            setattr(db_obj, key, value)

        self.db.commit()
        self.db.refresh(db_obj)
        return db_obj

    def delete(self, id: int) -> bool:
        """삭제"""
        db_obj = self.get_by_id(id)
        if not db_obj:
            return False

        self.db.delete(db_obj)
        self.db.commit()
        return True

# app/repositories/user_repository.py
from typing import Optional
from sqlalchemy.orm import Session
from app.repositories.base import BaseRepository
from app.models.user import User

class UserRepository(BaseRepository[User]):
    """사용자 Repository"""

    def __init__(self, db: Session):
        super().__init__(User, db)

    def get_by_email(self, email: str) -> Optional[User]:
        """이메일로 조회"""
        return self.db.query(User).filter(User.email == email).first()

    def get_by_username(self, username: str) -> Optional[User]:
        """사용자명으로 조회"""
        return self.db.query(User).filter(User.username == username).first()

    def get_active_users(self) -> List[User]:
        """활성 사용자 목록"""
        return self.db.query(User).filter(User.is_active == True).all()

    def search_by_username(self, query: str) -> List[User]:
        """사용자명 검색"""
        return self.db.query(User).filter(
            User.username.ilike(f"%{query}%")
        ).all()
```

### 1.3 Repository 사용

```python
# app/api/v1/endpoints/users.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from app.core.database import get_db
from app.repositories.user_repository import UserRepository
from app.schemas.user import User, UserCreate

router = APIRouter()

@router.get("/{user_id}", response_model=User)
def get_user(user_id: int, db: Session = Depends(get_db)):
    """사용자 조회"""
    repo = UserRepository(db)
    user = repo.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.post("/", response_model=User)
def create_user(user_in: UserCreate, db: Session = Depends(get_db)):
    """사용자 생성"""
    repo = UserRepository(db)

    # 중복 확인
    if repo.get_by_email(user_in.email):
        raise HTTPException(status_code=409, detail="Email already exists")

    return repo.create(user_in.dict())
```

### 1.4 Spring Boot와 비교

**Spring Boot JPA Repository:**
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    List<User> findByIsActiveTrue();

    @Query("SELECT u FROM User u WHERE u.username LIKE %:query%")
    List<User> searchByUsername(@Param("query") String query);
}

// 사용
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("User not found"));
    }
}
```

## 2. Service 패턴

### 2.1 Service 레이어의 역할

비즈니스 로직을 캡슐화하고 Repository를 조율합니다.

```python
# app/services/user_service.py
from typing import List, Optional
from sqlalchemy.orm import Session
from fastapi import HTTPException, status
from app.repositories.user_repository import UserRepository
from app.repositories.item_repository import ItemRepository
from app.schemas.user import UserCreate, UserUpdate
from app.models.user import User
from app.core.security import get_password_hash, verify_password

class UserService:
    """사용자 비즈니스 로직"""

    def __init__(self, db: Session):
        self.user_repo = UserRepository(db)
        self.item_repo = ItemRepository(db)

    def create_user(self, user_in: UserCreate) -> User:
        """사용자 생성"""
        # 비즈니스 규칙: 이메일 중복 불가
        if self.user_repo.get_by_email(user_in.email):
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="Email already registered"
            )

        # 비즈니스 규칙: 사용자명 중복 불가
        if self.user_repo.get_by_username(user_in.username):
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="Username already taken"
            )

        # 비밀번호 해싱
        user_data = user_in.dict()
        user_data["hashed_password"] = get_password_hash(user_data.pop("password"))

        return self.user_repo.create(user_data)

    def authenticate(self, username: str, password: str) -> Optional[User]:
        """인증"""
        user = self.user_repo.get_by_username(username)
        if not user:
            return None
        if not verify_password(password, user.hashed_password):
            return None
        return user

    def get_user_with_items(self, user_id: int) -> dict:
        """사용자와 아이템 함께 조회"""
        user = self.user_repo.get_by_id(user_id)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found"
            )

        items = self.item_repo.get_by_owner_id(user_id)

        return {
            "user": user,
            "items": items,
            "total_items": len(items)
        }

    def deactivate_user(self, user_id: int, current_user: User) -> User:
        """사용자 비활성화"""
        # 비즈니스 규칙: 자기 자신 또는 관리자만 가능
        if user_id != current_user.id and not current_user.is_superuser:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions"
            )

        user = self.user_repo.get_by_id(user_id)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found"
            )

        # 비즈니스 규칙: 이미 비활성 상태면 에러
        if not user.is_active:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="User is already inactive"
            )

        return self.user_repo.update(user_id, {"is_active": False})

    def search_users(self, query: str, min_length: int = 3) -> List[User]:
        """사용자 검색"""
        # 비즈니스 규칙: 최소 3자 이상
        if len(query) < min_length:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Search query must be at least {min_length} characters"
            )

        return self.user_repo.search_by_username(query)
```

### 2.2 Service를 Dependency로 주입

```python
# app/api/deps.py
from fastapi import Depends
from sqlalchemy.orm import Session
from app.core.database import get_db
from app.services.user_service import UserService

def get_user_service(db: Session = Depends(get_db)) -> UserService:
    """UserService 의존성"""
    return UserService(db)

# app/api/v1/endpoints/users.py
from fastapi import APIRouter, Depends
from app.api.deps import get_user_service, get_current_user
from app.services.user_service import UserService
from app.schemas.user import User, UserCreate

router = APIRouter()

@router.post("/", response_model=User)
def create_user(
    user_in: UserCreate,
    service: UserService = Depends(get_user_service)
):
    """사용자 생성"""
    return service.create_user(user_in)

@router.get("/{user_id}/with-items")
def get_user_with_items(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    """사용자와 아이템 조회"""
    return service.get_user_with_items(user_id)

@router.post("/{user_id}/deactivate", response_model=User)
def deactivate_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_user)
):
    """사용자 비활성화"""
    return service.deactivate_user(user_id, current_user)
```

## 3. Factory 패턴

### 3.1 객체 생성 팩토리

```python
# app/factories/user_factory.py
from typing import Optional
from app.models.user import User
from app.core.security import get_password_hash

class UserFactory:
    """사용자 객체 생성 팩토리"""

    @staticmethod
    def create_regular_user(
        username: str,
        email: str,
        password: str
    ) -> User:
        """일반 사용자 생성"""
        return User(
            username=username,
            email=email,
            hashed_password=get_password_hash(password),
            is_active=True,
            is_superuser=False
        )

    @staticmethod
    def create_superuser(
        username: str,
        email: str,
        password: str
    ) -> User:
        """슈퍼유저 생성"""
        return User(
            username=username,
            email=email,
            hashed_password=get_password_hash(password),
            is_active=True,
            is_superuser=True
        )

    @staticmethod
    def create_from_oauth(
        provider: str,
        provider_id: str,
        email: str,
        username: Optional[str] = None
    ) -> User:
        """OAuth 사용자 생성"""
        return User(
            username=username or f"{provider}_{provider_id}",
            email=email,
            hashed_password="",  # OAuth 사용자는 비밀번호 불필요
            is_active=True,
            is_superuser=False,
            oauth_provider=provider,
            oauth_provider_id=provider_id
        )

# 사용
user = UserFactory.create_regular_user("john", "john@example.com", "password123")
admin = UserFactory.create_superuser("admin", "admin@example.com", "admin123")
oauth_user = UserFactory.create_from_oauth("google", "123456", "user@gmail.com")
```

### 3.2 Response 팩토리

```python
# app/factories/response_factory.py
from typing import Any, Optional
from fastapi.responses import JSONResponse
from app.schemas.common import ErrorResponse, SuccessResponse

class ResponseFactory:
    """API 응답 팩토리"""

    @staticmethod
    def success(
        data: Any,
        message: str = "Success",
        status_code: int = 200
    ) -> JSONResponse:
        """성공 응답"""
        return JSONResponse(
            status_code=status_code,
            content={
                "success": True,
                "message": message,
                "data": data
            }
        )

    @staticmethod
    def error(
        message: str,
        errors: Optional[dict] = None,
        status_code: int = 400
    ) -> JSONResponse:
        """에러 응답"""
        content = {
            "success": False,
            "message": message
        }
        if errors:
            content["errors"] = errors

        return JSONResponse(
            status_code=status_code,
            content=content
        )

    @staticmethod
    def not_found(resource: str = "Resource") -> JSONResponse:
        """404 응답"""
        return ResponseFactory.error(
            message=f"{resource} not found",
            status_code=404
        )

    @staticmethod
    def forbidden(message: str = "Access forbidden") -> JSONResponse:
        """403 응답"""
        return ResponseFactory.error(
            message=message,
            status_code=403
        )

# 사용
@router.get("/users/{user_id}")
def get_user(user_id: int):
    user = get_user_from_db(user_id)
    if not user:
        return ResponseFactory.not_found("User")
    return ResponseFactory.success(user)
```

## 4. Singleton 패턴

### 4.1 데이터베이스 연결 싱글톤

```python
# app/core/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from typing import Generator

class DatabaseManager:
    """데이터베이스 관리 싱글톤"""
    _instance = None
    _engine = None
    _session_factory = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def initialize(self, database_url: str):
        """초기화"""
        if self._engine is None:
            self._engine = create_engine(database_url, pool_pre_ping=True)
            self._session_factory = sessionmaker(
                autocommit=False,
                autoflush=False,
                bind=self._engine
            )

    def get_session(self) -> Generator[Session, None, None]:
        """세션 생성"""
        if self._session_factory is None:
            raise RuntimeError("DatabaseManager not initialized")

        session = self._session_factory()
        try:
            yield session
        finally:
            session.close()

    @property
    def engine(self):
        return self._engine

# 사용
from app.core.config import settings

db_manager = DatabaseManager()
db_manager.initialize(settings.DATABASE_URL)

def get_db():
    return db_manager.get_session()
```

### 4.2 캐시 매니저 싱글톤

```python
# app/core/cache.py
from typing import Optional, Any
import redis
from app.core.config import settings

class CacheManager:
    """캐시 관리 싱글톤"""
    _instance = None
    _redis_client = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def initialize(self, redis_url: str):
        """초기화"""
        if self._redis_client is None:
            self._redis_client = redis.from_url(redis_url)

    def get(self, key: str) -> Optional[str]:
        """값 가져오기"""
        if self._redis_client is None:
            return None
        return self._redis_client.get(key)

    def set(self, key: str, value: Any, expire: int = 3600):
        """값 설정"""
        if self._redis_client is not None:
            self._redis_client.setex(key, expire, value)

    def delete(self, key: str):
        """값 삭제"""
        if self._redis_client is not None:
            self._redis_client.delete(key)

# 싱글톤 인스턴스
cache = CacheManager()
if settings.REDIS_URL:
    cache.initialize(settings.REDIS_URL)
```

## 5. Strategy 패턴

### 5.1 알림 전송 전략

```python
# app/strategies/notification_strategy.py
from abc import ABC, abstractmethod
from typing import Dict, Any

class NotificationStrategy(ABC):
    """알림 전송 전략 인터페이스"""

    @abstractmethod
    def send(self, recipient: str, message: str, **kwargs) -> bool:
        """알림 전송"""
        pass

class EmailNotification(NotificationStrategy):
    """이메일 알림"""

    def send(self, recipient: str, message: str, **kwargs) -> bool:
        print(f"Sending email to {recipient}: {message}")
        # 실제 이메일 전송 로직
        return True

class SMSNotification(NotificationStrategy):
    """SMS 알림"""

    def send(self, recipient: str, message: str, **kwargs) -> bool:
        print(f"Sending SMS to {recipient}: {message}")
        # 실제 SMS 전송 로직
        return True

class PushNotification(NotificationStrategy):
    """푸시 알림"""

    def send(self, recipient: str, message: str, **kwargs) -> bool:
        device_token = kwargs.get("device_token")
        print(f"Sending push to {device_token}: {message}")
        # 실제 푸시 알림 전송 로직
        return True

# app/services/notification_service.py
class NotificationService:
    """알림 서비스"""

    def __init__(self, strategy: NotificationStrategy):
        self.strategy = strategy

    def set_strategy(self, strategy: NotificationStrategy):
        """전략 변경"""
        self.strategy = strategy

    def notify(self, recipient: str, message: str, **kwargs) -> bool:
        """알림 전송"""
        return self.strategy.send(recipient, message, **kwargs)

# 사용
email_service = NotificationService(EmailNotification())
email_service.notify("user@example.com", "Welcome!")

sms_service = NotificationService(SMSNotification())
sms_service.notify("+1234567890", "Your code is 123456")

# 전략 동적 변경
service = NotificationService(EmailNotification())
service.notify("user@example.com", "Email notification")

service.set_strategy(SMSNotification())
service.notify("+1234567890", "SMS notification")
```

### 5.2 인증 전략

```python
# app/strategies/auth_strategy.py
from abc import ABC, abstractmethod
from typing import Optional
from app.models.user import User

class AuthStrategy(ABC):
    """인증 전략 인터페이스"""

    @abstractmethod
    def authenticate(self, credentials: dict) -> Optional[User]:
        """인증"""
        pass

class PasswordAuthStrategy(AuthStrategy):
    """비밀번호 인증"""

    def authenticate(self, credentials: dict) -> Optional[User]:
        username = credentials.get("username")
        password = credentials.get("password")

        # 사용자 조회 및 비밀번호 검증
        # ...
        return user

class TokenAuthStrategy(AuthStrategy):
    """토큰 인증"""

    def authenticate(self, credentials: dict) -> Optional[User]:
        token = credentials.get("token")

        # 토큰 검증
        # ...
        return user

class OAuthStrategy(AuthStrategy):
    """OAuth 인증"""

    def authenticate(self, credentials: dict) -> Optional[User]:
        provider = credentials.get("provider")
        access_token = credentials.get("access_token")

        # OAuth 토큰 검증
        # ...
        return user

# app/services/auth_service.py
class AuthService:
    """인증 서비스"""

    def __init__(self):
        self.strategies = {
            "password": PasswordAuthStrategy(),
            "token": TokenAuthStrategy(),
            "oauth": OAuthStrategy(),
        }

    def authenticate(self, method: str, credentials: dict) -> Optional[User]:
        """인증"""
        strategy = self.strategies.get(method)
        if not strategy:
            raise ValueError(f"Unknown auth method: {method}")

        return strategy.authenticate(credentials)

# 사용
auth_service = AuthService()

# 비밀번호 인증
user1 = auth_service.authenticate("password", {
    "username": "john",
    "password": "secret"
})

# OAuth 인증
user2 = auth_service.authenticate("oauth", {
    "provider": "google",
    "access_token": "ya29.xxx"
})
```

## 6. Observer 패턴

### 6.1 이벤트 시스템

```python
# app/events/event_manager.py
from typing import Callable, Dict, List

class EventManager:
    """이벤트 관리자"""

    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}

    def subscribe(self, event_type: str, callback: Callable):
        """이벤트 구독"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(callback)

    def unsubscribe(self, event_type: str, callback: Callable):
        """구독 취소"""
        if event_type in self._subscribers:
            self._subscribers[event_type].remove(callback)

    def notify(self, event_type: str, data: dict):
        """이벤트 발행"""
        if event_type in self._subscribers:
            for callback in self._subscribers[event_type]:
                callback(data)

# 싱글톤 인스턴스
event_manager = EventManager()

# app/events/handlers.py
def send_welcome_email(data: dict):
    """환영 이메일 발송"""
    user_email = data.get("email")
    print(f"Sending welcome email to {user_email}")

def log_user_creation(data: dict):
    """사용자 생성 로그"""
    username = data.get("username")
    print(f"User created: {username}")

def update_analytics(data: dict):
    """분석 데이터 업데이트"""
    print(f"Updating analytics for new user")

# 이벤트 핸들러 등록
event_manager.subscribe("user_created", send_welcome_email)
event_manager.subscribe("user_created", log_user_creation)
event_manager.subscribe("user_created", update_analytics)

# app/services/user_service.py (수정)
class UserService:
    def create_user(self, user_in: UserCreate) -> User:
        """사용자 생성"""
        user = self.user_repo.create(user_in.dict())

        # 이벤트 발행
        event_manager.notify("user_created", {
            "username": user.username,
            "email": user.email,
            "id": user.id
        })

        return user
```

## 7. Decorator 패턴

### 7.1 캐싱 데코레이터

```python
# app/decorators/cache.py
from functools import wraps
from typing import Callable
import json
import hashlib
from app.core.cache import cache

def cached(expire: int = 3600):
    """캐시 데코레이터"""
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 캐시 키 생성
            key_data = f"{func.__name__}:{args}:{kwargs}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # 캐시 확인
            cached_value = cache.get(cache_key)
            if cached_value:
                return json.loads(cached_value)

            # 함수 실행
            result = await func(*args, **kwargs)

            # 캐시 저장
            cache.set(cache_key, json.dumps(result), expire)

            return result
        return wrapper
    return decorator

# 사용
@cached(expire=300)
async def get_expensive_data(user_id: int):
    """비용이 큰 데이터 조회"""
    # 복잡한 계산 또는 외부 API 호출
    await asyncio.sleep(2)
    return {"user_id": user_id, "data": "expensive"}
```

### 7.2 로깅 데코레이터

```python
# app/decorators/logging.py
from functools import wraps
import logging
import time

logger = logging.getLogger(__name__)

def log_execution_time(func):
    """실행 시간 로깅 데코레이터"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start_time = time.time()

        try:
            result = await func(*args, **kwargs)
            duration = time.time() - start_time

            logger.info(
                f"{func.__name__} executed in {duration:.2f}s",
                extra={"function": func.__name__, "duration": duration}
            )

            return result
        except Exception as e:
            duration = time.time() - start_time
            logger.error(
                f"{func.__name__} failed after {duration:.2f}s: {e}",
                extra={"function": func.__name__, "duration": duration, "error": str(e)}
            )
            raise

    return wrapper

# 사용
@log_execution_time
async def process_data(data: dict):
    """데이터 처리"""
    await asyncio.sleep(1)
    return {"processed": True}
```

## 8. 실전 예제: 패턴 통합

```python
# app/services/order_service.py
from app.repositories.order_repository import OrderRepository
from app.repositories.user_repository import UserRepository
from app.repositories.product_repository import ProductRepository
from app.strategies.payment_strategy import PaymentStrategy
from app.events.event_manager import event_manager
from app.decorators.cache import cached
from app.decorators.logging import log_execution_time

class OrderService:
    """주문 서비스 (여러 패턴 통합)"""

    def __init__(
        self,
        order_repo: OrderRepository,
        user_repo: UserRepository,
        product_repo: ProductRepository
    ):
        # Repository 패턴
        self.order_repo = order_repo
        self.user_repo = user_repo
        self.product_repo = product_repo

    @log_execution_time  # Decorator 패턴
    async def create_order(
        self,
        user_id: int,
        items: List[dict],
        payment_strategy: PaymentStrategy  # Strategy 패턴
    ) -> Order:
        """주문 생성"""
        # 사용자 확인
        user = self.user_repo.get_by_id(user_id)
        if not user:
            raise ValueError("User not found")

        # 상품 확인 및 재고 검증
        total_amount = 0
        for item in items:
            product = self.product_repo.get_by_id(item["product_id"])
            if not product or product.stock < item["quantity"]:
                raise ValueError(f"Product {item['product_id']} not available")
            total_amount += product.price * item["quantity"]

        # 결제 처리 (Strategy 패턴)
        payment_result = payment_strategy.process_payment(total_amount, user.id)
        if not payment_result.success:
            raise ValueError("Payment failed")

        # 주문 생성
        order = self.order_repo.create({
            "user_id": user_id,
            "items": items,
            "total_amount": total_amount,
            "payment_id": payment_result.transaction_id
        })

        # 재고 감소
        for item in items:
            self.product_repo.decrease_stock(
                item["product_id"],
                item["quantity"]
            )

        # 이벤트 발행 (Observer 패턴)
        event_manager.notify("order_created", {
            "order_id": order.id,
            "user_id": user_id,
            "total_amount": total_amount
        })

        return order

    @cached(expire=60)  # Decorator 패턴 + Singleton (cache)
    async def get_user_orders(self, user_id: int) -> List[Order]:
        """사용자 주문 목록 (캐시됨)"""
        return self.order_repo.get_by_user_id(user_id)
```

## 9. 요약

### 핵심 포인트

1. **Repository 패턴**
   - 데이터 접근 캡슐화
   - 테스트 용이성
   - 데이터 소스 독립성

2. **Service 패턴**
   - 비즈니스 로직 분리
   - Repository 조율
   - 트랜잭션 관리

3. **Factory 패턴**
   - 객체 생성 캡슐화
   - 생성 로직 중앙화

4. **Singleton 패턴**
   - 단일 인스턴스 보장
   - 리소스 공유

5. **Strategy 패턴**
   - 알고리즘 캡슐화
   - 런타임 전략 변경

6. **Observer 패턴**
   - 이벤트 기반 아키텍처
   - 느슨한 결합

7. **Decorator 패턴**
   - 기능 동적 추가
   - 코드 재사용

### 다음 챕터 예고

**Chapter 26: 성능 최적화 (Performance Optimization)**

다음 챕터에서는 FastAPI 애플리케이션의 성능 최적화를 다룹니다:
- 쿼리 최적화 (N+1 문제)
- 캐싱 전략 (Redis)
- 연결 풀링
- 비동기 최적화
- 프로파일링과 모니터링
- 부하 테스트

디자인 패턴을 활용하면 유지보수하기 쉽고 확장 가능한 코드를 작성할 수 있습니다!
