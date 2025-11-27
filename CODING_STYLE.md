# Django 이커머스 프로젝트 코딩 스타일 가이드

## 개요

이 문서는 프로젝트의 코드 작성 규칙과 아키텍처 패턴을 정의합니다.

**핵심 원칙**:
- ✅ Service Layer 패턴 사용
- ✅ Type Hints 적극 활용
- ✅ PEP 8 + Django Coding Style 준수
- ✅ Pydantic으로 데이터 검증
- ✅ 가독성과 유지보수성 우선

---

## 1. 아키텍처 패턴

### 1.1 Service Layer 패턴

**계층 구조**:
```
┌─────────────────────────────────────┐
│  Views (API Layer)                  │  - 요청/응답 처리
│  - 인증 확인                         │  - 데이터 검증 (Pydantic)
│  - 권한 확인                         │  - HTTP 상태 코드 반환
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  Services (Business Logic)          │  - 비즈니스 로직
│  - 복잡한 연산                       │  - 트랜잭션 관리
│  - 외부 API 호출                     │  - 여러 모델 조작
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  Models (Data Layer)                │  - 데이터 구조
│  - 필드 정의                         │  - 간단한 메서드만
│  - 관계 정의                         │  - 프로퍼티, 유틸리티
└─────────────────────────────────────┘
```

### 1.2 책임 분리 규칙

#### ❌ 나쁜 예: Fat Models
```python
# models.py
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)

    def create_from_cart(self, cart):
        """비즈니스 로직이 모델에 있음 - 피하세요!"""
        # 장바구니 → 주문 변환
        # 재고 차감
        # 결제 API 호출
        # 이메일 발송
        pass  # 너무 많은 책임!
```

#### ✅ 좋은 예: Service Layer
```python
# models.py
class Order(models.Model):
    """주문 모델 - 데이터 구조와 간단한 메서드만"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    order_number = models.CharField(max_length=20, unique=True)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    created_at = models.DateTimeField(auto_now_add=True)

    def is_cancellable(self) -> bool:
        """간단한 비즈니스 규칙은 OK"""
        return self.status in ['pending', 'paid']

    @property
    def can_review(self) -> bool:
        """프로퍼티로 상태 확인"""
        return self.status == 'delivered'


# services.py
from typing import List
from decimal import Decimal
from django.db import transaction
from pydantic import BaseModel

class CreateOrderInput(BaseModel):
    """입력 데이터 검증"""
    user_id: int
    cart_id: int
    address_id: int

class OrderService:
    """주문 관련 비즈니스 로직"""

    @staticmethod
    @transaction.atomic
    def create_from_cart(
        cart: Cart,
        user: User,
        address: Address
    ) -> Order:
        """장바구니에서 주문 생성"""
        # 1. 재고 확인
        if not OrderService._check_stock(cart):
            raise InsufficientStockError("재고가 부족합니다")

        # 2. 주문 생성
        order = Order.objects.create(
            user=user,
            order_number=OrderService._generate_order_number(),
            total_amount=cart.calculate_total(),
            status='pending',
            # 배송지 정보 복사
            recipient_name=address.recipient,
            address=address.address,
        )

        # 3. 주문 아이템 생성 및 재고 차감
        for cart_item in cart.items.all():
            OrderItem.objects.create(
                order=order,
                product=cart_item.product,
                quantity=cart_item.quantity,
                price=cart_item.product.price,
            )
            # 재고 차감
            cart_item.product.stock_quantity -= cart_item.quantity
            cart_item.product.save()

        # 4. 장바구니 비우기
        cart.items.all().delete()

        return order

    @staticmethod
    def _check_stock(cart: Cart) -> bool:
        """재고 확인 헬퍼 메서드"""
        for item in cart.items.all():
            if item.product.stock_quantity < item.quantity:
                return False
        return True

    @staticmethod
    def _generate_order_number() -> str:
        """주문 번호 생성"""
        import uuid
        return f"ORD-{uuid.uuid4().hex[:12].upper()}"


# views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class OrderCreateView(APIView):
    """주문 생성 API - 요청/응답만 처리"""

    def post(self, request):
        # 1. 요청 데이터 검증
        try:
            input_data = CreateOrderInput(**request.data)
        except ValidationError as e:
            return Response(
                {'error': str(e)},
                status=status.HTTP_400_BAD_REQUEST
            )

        # 2. 권한 확인
        if request.user.id != input_data.user_id:
            return Response(
                {'error': '권한이 없습니다'},
                status=status.HTTP_403_FORBIDDEN
            )

        # 3. 비즈니스 로직 호출 (Service Layer)
        try:
            cart = Cart.objects.get(id=input_data.cart_id, user=request.user)
            address = Address.objects.get(id=input_data.address_id, user=request.user)

            order = OrderService.create_from_cart(cart, request.user, address)

            return Response(
                {'order_id': order.id, 'order_number': order.order_number},
                status=status.HTTP_201_CREATED
            )
        except Cart.DoesNotExist:
            return Response(
                {'error': '장바구니를 찾을 수 없습니다'},
                status=status.HTTP_404_NOT_FOUND
            )
        except InsufficientStockError as e:
            return Response(
                {'error': str(e)},
                status=status.HTTP_400_BAD_REQUEST
            )
```

### 1.3 파일 구조

```
apps/
└── orders/
    ├── __init__.py
    ├── models.py          # 모델 정의
    ├── services.py        # 비즈니스 로직 (Service Layer)
    ├── views.py           # API 뷰 (요청/응답 처리)
    ├── serializers.py     # DRF Serializer (간단한 경우)
    ├── schemas.py         # Pydantic 스키마 (추천)
    ├── exceptions.py      # 커스텀 예외
    ├── admin.py           # Django Admin
    ├── urls.py            # URL 라우팅
    └── tests/
        ├── test_models.py
        ├── test_services.py
        ├── test_views.py
        └── factories.py
```

---

## 2. Type Hints 사용 규칙

### 2.1 필수 사용 위치

**✅ 반드시 Type Hints 사용**:
1. 함수/메서드 시그니처
2. 클래스 속성
3. 공개 API
4. 서비스 계층

```python
from typing import List, Optional, Dict, Any
from decimal import Decimal

class ProductService:
    """모든 public 메서드에 타입 힌트 필수"""

    @staticmethod
    def get_active_products(category_id: Optional[int] = None) -> List[Product]:
        """활성 상품 조회"""
        queryset = Product.objects.filter(is_active=True)
        if category_id:
            queryset = queryset.filter(category_id=category_id)
        return list(queryset)

    @staticmethod
    def calculate_discount(
        original_price: Decimal,
        discount_rate: float
    ) -> Decimal:
        """할인 가격 계산"""
        return original_price * Decimal(1 - discount_rate)
```

### 2.2 Django 모델 타입 힌트

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from apps.accounts.models import User

class Order(models.Model):
    user: 'User' = models.ForeignKey('accounts.User', on_delete=models.CASCADE)

    def get_user_email(self) -> str:
        return self.user.email

    @classmethod
    def get_user_orders(cls, user: 'User') -> 'QuerySet[Order]':
        return cls.objects.filter(user=user)
```

### 2.3 Generic Types

```python
from typing import TypeVar, Generic, List

T = TypeVar('T')

class PaginatedResponse(Generic[T]):
    """제네릭 페이지네이션 응답"""

    def __init__(
        self,
        items: List[T],
        total: int,
        page: int,
        page_size: int
    ):
        self.items = items
        self.total = total
        self.page = page
        self.page_size = page_size

# 사용 예
def get_products(page: int) -> PaginatedResponse[Product]:
    products = Product.objects.all()[page*10:(page+1)*10]
    return PaginatedResponse(
        items=list(products),
        total=Product.objects.count(),
        page=page,
        page_size=10
    )
```

---

## 3. Pydantic 사용 가이드

### 3.1 API 요청/응답 스키마

```python
# apps/orders/schemas.py
from pydantic import BaseModel, Field, validator
from typing import List, Optional
from decimal import Decimal
from datetime import datetime

class OrderItemInput(BaseModel):
    """주문 아이템 입력"""
    product_id: int = Field(..., gt=0)
    quantity: int = Field(..., gt=0, le=999)

    @validator('quantity')
    def quantity_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError('수량은 1 이상이어야 합니다')
        return v

class CreateOrderInput(BaseModel):
    """주문 생성 요청"""
    items: List[OrderItemInput] = Field(..., min_items=1)
    address_id: int = Field(..., gt=0)
    delivery_message: Optional[str] = Field(None, max_length=200)

    @validator('items')
    def items_not_empty(cls, v):
        if not v:
            raise ValueError('주문 아이템이 비어있습니다')
        return v

class OrderItemOutput(BaseModel):
    """주문 아이템 응답"""
    product_id: int
    product_name: str
    quantity: int
    price: Decimal
    subtotal: Decimal

    class Config:
        orm_mode = True  # Django 모델에서 자동 변환

class OrderOutput(BaseModel):
    """주문 응답"""
    id: int
    order_number: str
    status: str
    items: List[OrderItemOutput]
    total_amount: Decimal
    created_at: datetime

    class Config:
        orm_mode = True


# views.py에서 사용
from pydantic import ValidationError

class OrderCreateView(APIView):
    def post(self, request):
        try:
            # 입력 검증
            input_data = CreateOrderInput(**request.data)

            # 서비스 호출
            order = OrderService.create_order(
                user=request.user,
                items=input_data.items,
                address_id=input_data.address_id,
                delivery_message=input_data.delivery_message
            )

            # 응답 직렬화
            output = OrderOutput.from_orm(order)
            return Response(output.dict(), status=status.HTTP_201_CREATED)

        except ValidationError as e:
            return Response(
                {'errors': e.errors()},
                status=status.HTTP_400_BAD_REQUEST
            )
```

### 3.2 설정 객체

```python
# config/settings/schemas.py
from pydantic import BaseSettings, Field

class PaymentSettings(BaseSettings):
    """결제 설정"""
    portone_api_key: str = Field(..., env='PORTONE_API_KEY')
    portone_api_secret: str = Field(..., env='PORTONE_API_SECRET')
    toss_client_key: str = Field(..., env='TOSS_CLIENT_KEY')
    toss_secret_key: str = Field(..., env='TOSS_SECRET_KEY')

    class Config:
        env_file = '.env'
        env_file_encoding = 'utf-8'

# 사용
payment_settings = PaymentSettings()
```

### 3.3 Dataclass vs Pydantic

**Pydantic 사용 (추천)**:
- API 입력/출력
- 외부 데이터 검증
- 설정 객체

**Dataclass 사용** (선택사항):
- 내부 서비스 계층의 간단한 DTO
- 검증이 필요 없는 데이터 구조

```python
from dataclasses import dataclass

@dataclass
class OrderSummary:
    """내부용 간단한 데이터 객체"""
    order_count: int
    total_amount: Decimal
    average_amount: Decimal
```

---

## 4. 코드 스타일 규칙

### 4.1 PEP 8 기본 규칙

```python
# ✅ 좋은 예
def calculate_total_price(
    items: List[OrderItem],
    discount_rate: float = 0.0,
    include_tax: bool = True
) -> Decimal:
    """총 가격 계산

    Args:
        items: 주문 아이템 리스트
        discount_rate: 할인율 (0.0 ~ 1.0)
        include_tax: 세금 포함 여부

    Returns:
        총 가격 (Decimal)
    """
    subtotal = sum(item.price * item.quantity for item in items)

    if discount_rate > 0:
        subtotal *= Decimal(1 - discount_rate)

    if include_tax:
        subtotal *= Decimal('1.1')  # 10% 세금

    return subtotal.quantize(Decimal('0.01'))


# ❌ 나쁜 예
def calc(items,disc=0,tax=True):  # 타입 힌트 없음, 이름 불명확
    tot=0  # 변수명 너무 짧음
    for i in items:tot+=i.price*i.quantity  # 한 줄에 너무 많은 코드
    if disc>0:tot*=(1-disc)
    if tax:tot*=1.1
    return tot
```

### 4.2 네이밍 컨벤션

```python
# 모듈/패키지: snake_case
# 파일명: order_service.py, payment_provider.py

# 클래스: PascalCase
class OrderService:
    pass

class PaymentProvider:
    pass

# 함수/메서드/변수: snake_case
def create_order(user_id: int, items: List[int]) -> Order:
    order_number = generate_order_number()
    total_amount = calculate_total(items)
    return Order(...)

# 상수: UPPER_SNAKE_CASE
MAX_ORDER_ITEMS = 100
DEFAULT_SHIPPING_FEE = Decimal('3000')

# Private (내부 사용): 앞에 _ 붙이기
class OrderService:
    def _check_stock(self, product: Product) -> bool:
        """내부 헬퍼 메서드"""
        return product.stock_quantity > 0
```

### 4.3 Import 순서

```python
# 1. 표준 라이브러리
import os
import sys
from datetime import datetime
from typing import List, Optional

# 2. 서드파티 라이브러리
import django
from django.db import models
from django.contrib.auth import get_user_model
from rest_framework.views import APIView
from pydantic import BaseModel

# 3. 로컬 앱
from apps.products.models import Product
from apps.orders.services import OrderService
from apps.payments.providers import PortOneProvider

# 4. 상대 import
from .models import Cart
from .exceptions import InsufficientStockError
```

**자동화 도구**: `isort`
```bash
# pyproject.toml 또는 setup.cfg에서 설정
pip install isort
isort .
```

### 4.4 줄 길이 및 포매팅

```python
# 최대 줄 길이: 88자 (Black 기준)
# 긴 함수 호출은 여러 줄로 분리

# ✅ 좋은 예
order = Order.objects.create(
    user=user,
    order_number=order_number,
    total_amount=total_amount,
    shipping_fee=shipping_fee,
    status='pending',
)

# ❌ 나쁜 예 (한 줄에 너무 길게)
order = Order.objects.create(user=user, order_number=order_number, total_amount=total_amount, shipping_fee=shipping_fee, status='pending')
```

**자동화 도구**: `black`
```bash
pip install black
black .
```

### 4.5 Docstrings

```python
def create_order_from_cart(
    cart: Cart,
    user: User,
    address: Address
) -> Order:
    """장바구니에서 주문 생성

    장바구니의 모든 아이템을 주문으로 변환하고,
    재고를 차감합니다.

    Args:
        cart: 장바구니 객체
        user: 주문하는 사용자
        address: 배송지 주소

    Returns:
        생성된 주문 객체

    Raises:
        InsufficientStockError: 재고가 부족한 경우
        InvalidCartError: 장바구니가 비어있는 경우

    Example:
        >>> cart = Cart.objects.get(user=user)
        >>> address = user.addresses.filter(is_default=True).first()
        >>> order = create_order_from_cart(cart, user, address)
        >>> print(order.order_number)
        'ORD-123456789ABC'
    """
    pass
```

---

## 5. Django 특화 규칙

### 5.1 모델 정의

```python
# ✅ 좋은 예
class Product(models.Model):
    """상품 모델"""

    # 1. 필드 정의 (논리적 순서)
    # - ID (자동)
    # - 관계 필드
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        related_name='products'
    )

    # - 기본 필드
    name = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    description = models.TextField()

    # - 가격/재고
    price = models.DecimalField(max_digits=10, decimal_places=2)
    discount_price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        null=True,
        blank=True
    )
    stock_quantity = models.PositiveIntegerField(default=0)

    # - 상태 필드
    is_active = models.BooleanField(default=True)
    is_featured = models.BooleanField(default=False)

    # - 타임스탬프
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # 2. Meta 클래스
    class Meta:
        db_table = 'products'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['name']),
            models.Index(fields=['category', 'is_active']),
        ]
        verbose_name = '상품'
        verbose_name_plural = '상품'

    # 3. __str__ 메서드
    def __str__(self) -> str:
        return self.name

    # 4. 프로퍼티
    @property
    def final_price(self) -> Decimal:
        """최종 가격 (할인 적용)"""
        return self.discount_price or self.price

    @property
    def is_available(self) -> bool:
        """구매 가능 여부"""
        return self.is_active and self.stock_quantity > 0

    # 5. 일반 메서드
    def apply_discount(self, rate: float) -> None:
        """할인 적용"""
        self.discount_price = self.price * Decimal(1 - rate)
        self.save(update_fields=['discount_price'])
```

### 5.2 QuerySet 최적화

```python
# ❌ N+1 쿼리 문제
def get_orders_bad():
    orders = Order.objects.all()
    for order in orders:
        print(order.user.email)  # 매번 DB 조회!
        for item in order.items.all():  # 또 매번 DB 조회!
            print(item.product.name)

# ✅ select_related, prefetch_related 사용
def get_orders_good():
    orders = Order.objects.select_related('user').prefetch_related(
        'items__product'
    ).all()

    for order in orders:
        print(order.user.email)  # 캐시된 데이터 사용
        for item in order.items.all():
            print(item.product.name)  # 캐시된 데이터 사용
```

### 5.3 트랜잭션 관리

```python
from django.db import transaction

class OrderService:
    @staticmethod
    @transaction.atomic
    def create_order(cart: Cart, user: User) -> Order:
        """트랜잭션으로 묶어서 원자성 보장"""
        # 1. 주문 생성
        order = Order.objects.create(user=user, ...)

        # 2. 주문 아이템 생성 및 재고 차감
        for cart_item in cart.items.all():
            OrderItem.objects.create(...)
            cart_item.product.stock_quantity -= cart_item.quantity
            cart_item.product.save()

        # 3. 장바구니 비우기
        cart.items.all().delete()

        # 모두 성공하거나, 모두 롤백됨
        return order
```

---

## 6. 예외 처리

### 6.1 커스텀 예외

```python
# apps/orders/exceptions.py
class OrderError(Exception):
    """주문 관련 기본 예외"""
    pass

class InsufficientStockError(OrderError):
    """재고 부족 예외"""
    pass

class InvalidOrderStatusError(OrderError):
    """잘못된 주문 상태 예외"""
    pass

# 사용
def cancel_order(order: Order) -> None:
    if not order.is_cancellable():
        raise InvalidOrderStatusError(
            f"주문 상태가 '{order.status}'일 때는 취소할 수 없습니다"
        )
    # 취소 로직
```

### 6.2 예외 처리 패턴

```python
# ✅ 구체적인 예외 처리
try:
    order = OrderService.create_from_cart(cart, user, address)
except InsufficientStockError as e:
    logger.warning(f"재고 부족: {e}")
    return Response({'error': str(e)}, status=400)
except InvalidCartError as e:
    logger.error(f"잘못된 장바구니: {e}")
    return Response({'error': str(e)}, status=400)
except Exception as e:
    logger.exception("예상치 못한 오류")
    return Response({'error': '서버 오류'}, status=500)

# ❌ 모든 예외를 잡지 마세요
try:
    order = OrderService.create_from_cart(cart, user, address)
except:  # 너무 광범위!
    return Response({'error': '오류 발생'}, status=500)
```

---

## 7. 코드 품질 도구

### 7.1 자동 포매팅

**pyproject.toml**:
```toml
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | migrations
)/
'''

[tool.isort]
profile = "black"
line_length = 88
skip_gitignore = true
known_first_party = ["apps", "config"]
sections = ["FUTURE", "STDLIB", "THIRDPARTY", "FIRSTPARTY", "LOCALFOLDER"]
```

### 7.2 린팅

**.flake8**:
```ini
[flake8]
max-line-length = 88
extend-ignore = E203, W503
exclude =
    .git,
    __pycache__,
    */migrations/*,
    .venv
```

### 7.3 타입 체크

**mypy.ini**:
```ini
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
plugins = mypy_django_plugin.main

[mypy.plugins.django-stubs]
django_settings_module = config.settings.development

[mypy-*.migrations.*]
ignore_errors = True
```

### 7.4 pre-commit 설정

**.pre-commit-config.yaml**:
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.11.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort

  - repo: https://github.com/pycqa/flake8
    rev: 6.1.0
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.7.0
    hooks:
      - id: mypy
        additional_dependencies: [django-stubs]
```

설치 및 사용:
```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files  # 수동 실행
```

---

## 8. 체크리스트

### 코드 작성 시 확인사항

- [ ] Type hints 추가했는가?
- [ ] Docstring 작성했는가?
- [ ] 비즈니스 로직을 Service Layer에 배치했는가?
- [ ] Pydantic으로 입력 검증했는가?
- [ ] 트랜잭션 처리가 필요한가?
- [ ] N+1 쿼리 문제가 없는가?
- [ ] 예외 처리를 적절히 했는가?
- [ ] 테스트 코드를 작성했는가?
- [ ] Black + isort로 포맷팅했는가?
- [ ] Flake8 경고가 없는가?

---

## 9. 참고 자료

- [PEP 8 – Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Django Coding Style](https://docs.djangoproject.com/en/stable/internals/contributing/writing-code/coding-style/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Type Hints Cheat Sheet](https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html)
