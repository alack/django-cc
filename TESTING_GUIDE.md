# Django 이커머스 테스트 가이드

## 개요

이 문서는 Django 이커머스 프로젝트의 테스트 작성 및 실행 가이드입니다.

**핵심 원칙**: 테스트는 개발과 함께 작성되며, 3단계로 진행됩니다.

## 테스트 3단계 전략

```
┌─────────────────────────────────────────────────────┐
│  1. 단위 테스트 (Unit Tests)                        │
│     - 각 Phase에서 코드와 함께 작성                │
│     - 모델, 비즈니스 로직, 유틸리티 함수           │
│     - pytest 사용                                  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  2. API 테스트                                      │
│     - 각 Phase에서 API 구현과 함께 작성            │
│     - 엔드포인트, 인증, 권한, 예외 처리            │
│     - DRF TestCase 사용                            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  3. Admin 수동 테스트                               │
│     - 각 Phase 완료 후 Admin에서 수동 확인         │
│     - 기능 동작, UI, 통계 위젯 검증                │
│     - 체크리스트 기반                              │
└─────────────────────────────────────────────────────┘
```

---

## 1. 단위 테스트 (Unit Tests)

### 1.1 환경 설정

```bash
# 개발 의존성 설치
pip install -r requirements/development.txt

# pytest, pytest-django, factory-boy, faker 포함
```

### 1.2 프로젝트 구조

```
apps/
├── accounts/
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_models.py
│   │   ├── test_views.py
│   │   └── factories.py
├── products/
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_models.py
│   │   ├── test_views.py
│   │   ├── test_services.py
│   │   └── factories.py
└── conftest.py  # 공통 fixture
```

### 1.3 모델 테스트 예시

#### Phase 2: CustomUser 모델 테스트

```python
# apps/accounts/tests/test_models.py
import pytest
from django.db import IntegrityError
from apps.accounts.models import CustomUser, Address

@pytest.mark.django_db
class TestCustomUser:
    def test_create_user(self):
        """회원 생성 테스트"""
        user = CustomUser.objects.create_user(
            email='test@example.com',
            username='testuser',
            password='testpass123'
        )
        assert user.email == 'test@example.com'
        assert user.check_password('testpass123')
        assert user.is_active == True

    def test_email_uniqueness(self):
        """이메일 유니크 제약 테스트"""
        CustomUser.objects.create_user(
            email='test@example.com',
            username='user1',
            password='pass123'
        )

        with pytest.raises(IntegrityError):
            CustomUser.objects.create_user(
                email='test@example.com',
                username='user2',
                password='pass123'
            )

    def test_default_address(self):
        """기본 배송지 설정 테스트"""
        user = CustomUser.objects.create_user(
            email='test@example.com',
            password='pass123'
        )

        # 첫 번째 배송지는 자동으로 기본 배송지
        address1 = Address.objects.create(
            user=user,
            name='집',
            recipient='홍길동',
            postal_code='12345',
            address='서울시 강남구'
        )
        assert address1.is_default == True

        # 두 번째 배송지 추가
        address2 = Address.objects.create(
            user=user,
            name='회사',
            recipient='홍길동',
            postal_code='54321',
            address='서울시 서초구',
            is_default=True
        )

        # 새로운 기본 배송지 설정 시 기존 배송지는 기본에서 해제
        address1.refresh_from_db()
        assert address1.is_default == False
        assert address2.is_default == True
```

#### Phase 3: Product 모델 테스트

```python
# apps/products/tests/test_models.py
import pytest
from apps.products.models import Product, Category, ProductOption

@pytest.mark.django_db
class TestProduct:
    def test_create_product(self):
        """상품 생성 테스트"""
        category = Category.objects.create(name='전자제품')
        product = Product.objects.create(
            category=category,
            name='노트북',
            price=1500000,
            stock_quantity=10
        )
        assert product.name == '노트북'
        assert product.is_available() == True

    def test_out_of_stock(self):
        """품절 상태 테스트"""
        category = Category.objects.create(name='전자제품')
        product = Product.objects.create(
            category=category,
            name='품절상품',
            price=10000,
            stock_quantity=0
        )
        assert product.is_available() == False

    def test_discount_price(self):
        """할인 가격 테스트"""
        category = Category.objects.create(name='전자제품')
        product = Product.objects.create(
            category=category,
            name='노트북',
            price=1000000,
            discount_price=900000,
            stock_quantity=10
        )
        assert product.get_final_price() == 900000
        assert product.get_discount_rate() == 10  # 10% 할인

    def test_category_hierarchy(self):
        """카테고리 계층 구조 테스트"""
        parent = Category.objects.create(name='전자제품')
        child = Category.objects.create(
            name='노트북',
            parent=parent
        )

        assert child.parent == parent
        assert parent.children.count() == 1
        assert child in parent.children.all()
```

### 1.4 비즈니스 로직 테스트 예시

#### Phase 5: 주문 생성 서비스 테스트

```python
# apps/orders/tests/test_services.py
import pytest
from decimal import Decimal
from apps.orders.services import OrderService
from apps.orders.exceptions import InsufficientStockError
from .factories import CartFactory, ProductFactory

@pytest.mark.django_db
class TestOrderService:
    def test_create_order_from_cart(self, user, address):
        """장바구니에서 주문 생성 테스트"""
        # 테스트 데이터 준비
        product1 = ProductFactory(price=10000, stock_quantity=10)
        product2 = ProductFactory(price=20000, stock_quantity=5)
        cart = CartFactory(user=user)
        cart.add_item(product1, quantity=2)
        cart.add_item(product2, quantity=1)

        # 주문 생성
        order = OrderService.create_from_cart(cart, user, address)

        # 검증
        assert order.user == user
        assert order.items.count() == 2
        assert order.total_amount == Decimal('40000')  # 10000*2 + 20000*1

        # 재고 차감 확인
        product1.refresh_from_db()
        product2.refresh_from_db()
        assert product1.stock_quantity == 8
        assert product2.stock_quantity == 4

    def test_order_creation_fails_on_insufficient_stock(self, user, address):
        """재고 부족 시 주문 생성 실패 테스트"""
        product = ProductFactory(price=10000, stock_quantity=1)
        cart = CartFactory(user=user)
        cart.add_item(product, quantity=5)  # 재고보다 많은 수량

        with pytest.raises(InsufficientStockError):
            OrderService.create_from_cart(cart, user, address)

        # 재고는 변경되지 않아야 함
        product.refresh_from_db()
        assert product.stock_quantity == 1

    def test_order_cancellation_restores_stock(self, order):
        """주문 취소 시 재고 복구 테스트"""
        # 주문 아이템의 상품 재고 확인
        order_item = order.items.first()
        product = order_item.product
        original_stock = product.stock_quantity
        ordered_quantity = order_item.quantity

        # 주문 취소
        OrderService.cancel_order(order)

        # 재고 복구 확인
        product.refresh_from_db()
        assert product.stock_quantity == original_stock + ordered_quantity
        assert order.status == 'cancelled'
```

### 1.5 Factory 작성 예시

```python
# apps/products/tests/factories.py
import factory
from factory.django import DjangoModelFactory
from apps.products.models import Product, Category

class CategoryFactory(DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Faker('word')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower())
    is_active = True

class ProductFactory(DjangoModelFactory):
    class Meta:
        model = Product

    category = factory.SubFactory(CategoryFactory)
    name = factory.Faker('word')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower())
    description = factory.Faker('text')
    price = factory.Faker('random_int', min=1000, max=100000)
    stock_quantity = factory.Faker('random_int', min=0, max=100)
    is_active = True
```

### 1.6 테스트 실행

```bash
# 전체 테스트 실행
pytest

# 특정 앱 테스트
pytest apps/accounts/tests/

# 특정 파일 테스트
pytest apps/products/tests/test_models.py

# 특정 테스트 함수
pytest apps/products/tests/test_models.py::TestProduct::test_create_product

# 커버리지 포함
pytest --cov=apps --cov-report=html

# 실패한 테스트만 재실행
pytest --lf

# verbose 모드
pytest -v
```

---

## 2. API 테스트

### 2.1 API 테스트 구조

```python
# apps/products/tests/test_views.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from apps.products.models import Product
from .factories import ProductFactory, UserFactory

class ProductAPITestCase(APITestCase):
    def setUp(self):
        """각 테스트 전 실행"""
        self.client = APIClient()
        self.user = UserFactory()
        self.product = ProductFactory()

    def test_product_list(self):
        """상품 목록 조회 테스트"""
        response = self.client.get('/api/products/')

        assert response.status_code == status.HTTP_200_OK
        assert len(response.data['results']) > 0

    def test_product_detail(self):
        """상품 상세 조회 테스트"""
        response = self.client.get(f'/api/products/{self.product.id}/')

        assert response.status_code == status.HTTP_200_OK
        assert response.data['name'] == self.product.name

    def test_product_search(self):
        """상품 검색 테스트"""
        ProductFactory(name='노트북')
        ProductFactory(name='마우스')

        response = self.client.get('/api/products/', {'search': '노트북'})

        assert response.status_code == status.HTTP_200_OK
        assert len(response.data['results']) == 1
        assert response.data['results'][0]['name'] == '노트북'
```

### 2.2 인증 테스트

```python
# apps/accounts/tests/test_auth.py
import pytest
from rest_framework.test import APIClient
from rest_framework import status
from apps.accounts.models import CustomUser

@pytest.mark.django_db
class TestAuthentication:
    def test_user_registration(self):
        """회원가입 테스트"""
        client = APIClient()
        data = {
            'email': 'newuser@example.com',
            'username': 'newuser',
            'password': 'testpass123',
            'password_confirm': 'testpass123'
        }

        response = client.post('/api/auth/register/', data)

        assert response.status_code == status.HTTP_201_CREATED
        assert 'access' in response.data
        assert 'refresh' in response.data

        # DB에 사용자 생성 확인
        user = CustomUser.objects.get(email='newuser@example.com')
        assert user.username == 'newuser'

    def test_login(self):
        """로그인 테스트"""
        client = APIClient()
        user = CustomUser.objects.create_user(
            email='test@example.com',
            username='testuser',
            password='testpass123'
        )

        data = {
            'email': 'test@example.com',
            'password': 'testpass123'
        }

        response = client.post('/api/auth/login/', data)

        assert response.status_code == status.HTTP_200_OK
        assert 'access' in response.data
        assert 'refresh' in response.data

    def test_authenticated_request(self):
        """인증된 요청 테스트"""
        client = APIClient()
        user = CustomUser.objects.create_user(
            email='test@example.com',
            password='testpass123'
        )

        # 로그인하여 토큰 획득
        response = client.post('/api/auth/login/', {
            'email': 'test@example.com',
            'password': 'testpass123'
        })
        token = response.data['access']

        # 인증된 요청
        client.credentials(HTTP_AUTHORIZATION=f'Bearer {token}')
        response = client.get('/api/accounts/me/')

        assert response.status_code == status.HTTP_200_OK
        assert response.data['email'] == 'test@example.com'

    def test_unauthorized_request(self):
        """인증되지 않은 요청 테스트"""
        client = APIClient()
        response = client.get('/api/accounts/me/')

        assert response.status_code == status.HTTP_401_UNAUTHORIZED
```

### 2.3 권한 테스트

```python
# apps/orders/tests/test_permissions.py
import pytest
from rest_framework.test import APIClient
from rest_framework import status
from .factories import UserFactory, OrderFactory

@pytest.mark.django_db
class TestOrderPermissions:
    def test_user_can_view_own_orders(self):
        """본인 주문만 조회 가능 테스트"""
        client = APIClient()
        user1 = UserFactory()
        user2 = UserFactory()
        order1 = OrderFactory(user=user1)
        order2 = OrderFactory(user=user2)

        # user1로 로그인
        client.force_authenticate(user=user1)

        # 본인 주문 조회 성공
        response = client.get(f'/api/orders/{order1.id}/')
        assert response.status_code == status.HTTP_200_OK

        # 타인 주문 조회 실패
        response = client.get(f'/api/orders/{order2.id}/')
        assert response.status_code == status.HTTP_404_NOT_FOUND

    def test_user_can_cancel_own_order(self):
        """본인 주문만 취소 가능 테스트"""
        client = APIClient()
        user1 = UserFactory()
        user2 = UserFactory()
        order = OrderFactory(user=user1, status='pending')

        # user2로 로그인 (다른 사용자)
        client.force_authenticate(user=user2)

        # 타인 주문 취소 실패
        response = client.post(f'/api/orders/{order.id}/cancel/')
        assert response.status_code == status.HTTP_404_NOT_FOUND

        # user1로 로그인 (주문 소유자)
        client.force_authenticate(user=user1)

        # 본인 주문 취소 성공
        response = client.post(f'/api/orders/{order.id}/cancel/')
        assert response.status_code == status.HTTP_200_OK
```

### 2.4 외부 API Mock 테스트

```python
# apps/payments/tests/test_services.py
import pytest
from unittest.mock import patch, Mock
from apps.payments.services import PortOnePaymentService
from .factories import OrderFactory

@pytest.mark.django_db
class TestPaymentService:
    @patch('apps.payments.services.requests.post')
    def test_prepare_payment(self, mock_post):
        """결제 준비 테스트 (Mock)"""
        # Mock 설정
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'code': 0,
            'response': {
                'merchant_uid': 'ORDER-123',
                'amount': 50000
            }
        }
        mock_post.return_value = mock_response

        # 결제 준비
        order = OrderFactory(total_amount=50000)
        service = PortOnePaymentService()
        result = service.prepare_payment(order)

        # 검증
        assert result['merchant_uid'] == 'ORDER-123'
        assert result['amount'] == 50000

        # Mock이 올바르게 호출되었는지 확인
        mock_post.assert_called_once()

    @patch('apps.payments.services.requests.post')
    def test_verify_payment(self, mock_post):
        """결제 검증 테스트 (Mock)"""
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'code': 0,
            'response': {
                'status': 'paid',
                'amount': 50000
            }
        }
        mock_post.return_value = mock_response

        service = PortOnePaymentService()
        result = service.verify_payment('imp_123456', 50000)

        assert result['status'] == 'paid'
        assert result['amount'] == 50000
```

---

## 3. Admin 수동 테스트

### 3.1 Phase별 체크리스트

#### Phase 2: 회원 관리 Admin

**테스트 시나리오**:

1. **회원 목록 확인**
   - [ ] `/admin/accounts/customuser/` 접속
   - [ ] 회원 목록이 표시되는지 확인
   - [ ] list_display 필드 확인 (이메일, 이름, 가입일, 활성 상태)

2. **검색 기능**
   - [ ] 이메일로 검색
   - [ ] 이름으로 검색
   - [ ] 전화번호로 검색

3. **필터링**
   - [ ] 활성 상태 필터
   - [ ] 가입일 필터
   - [ ] 로그인 방법 필터 (일반/소셜)

4. **회원 상세 페이지**
   - [ ] 회원 클릭 → 상세 페이지 이동
   - [ ] Address Inline에서 배송지 목록 확인
   - [ ] 기본 배송지 표시 확인

5. **관리 액션**
   - [ ] 여러 회원 선택
   - [ ] "선택한 회원 활성화" 액션 실행
   - [ ] "선택한 회원 비활성화" 액션 실행
   - [ ] 상태 변경 확인

6. **통계 위젯**
   - [ ] 총 회원 수 표시
   - [ ] 오늘 가입자 수 표시
   - [ ] 활성/비활성 회원 비율 표시

#### Phase 3: 상품 관리 Admin

**테스트 시나리오**:

1. **카테고리 관리**
   - [ ] `/admin/products/category/` 접속
   - [ ] 부모 카테고리 생성 (예: 전자제품)
   - [ ] 자식 카테고리 생성 (예: 노트북)
   - [ ] 계층 구조 표시 확인
   - [ ] 카테고리별 상품 수 표시 확인

2. **상품 생성**
   - [ ] `/admin/products/product/add/` 접속
   - [ ] 상품 정보 입력
   - [ ] ProductImage Inline에서 이미지 3개 업로드
   - [ ] 메인 이미지 설정
   - [ ] ProductOption Inline에서 옵션 추가 (색상, 사이즈)
   - [ ] 저장 후 목록으로 이동

3. **상품 목록**
   - [ ] 썸네일 표시 확인
   - [ ] list_editable로 가격/재고 빠른 수정
   - [ ] 수정 후 저장

4. **관리 액션**
   - [ ] 여러 상품 선택
   - [ ] "상품 일괄 활성화" 액션 실행
   - [ ] "가격 10% 인상" 액션 실행 (커스텀 액션)
   - [ ] 변경 사항 확인

5. **통계 위젯**
   - [ ] 총 상품 수
   - [ ] 활성 상품 수
   - [ ] 품절 상품 수
   - [ ] 카테고리별 상품 분포 그래프

#### Phase 5: 주문 관리 Admin

**테스트 시나리오**:

1. **주문 목록**
   - [ ] `/admin/orders/order/` 접속
   - [ ] 주문 목록 표시 확인
   - [ ] 상태별 색상 구분 확인
     - pending: 회색
     - paid: 파란색
     - shipped: 주황색
     - delivered: 초록색

2. **상태별 필터링**
   - [ ] "미결제" 필터 적용
   - [ ] "결제완료" 필터 적용
   - [ ] "배송중" 필터 적용
   - [ ] "배송완료" 필터 적용

3. **주문 상세**
   - [ ] 주문 클릭 → 상세 페이지
   - [ ] OrderItem Inline에서 주문 상품 목록 확인
   - [ ] 배송 정보 섹션 확인
     - 수령인 정보
     - 배송지 주소
     - 배송 메시지

4. **주문 상태 변경**
   - [ ] 여러 주문 선택 (미결제 상태)
   - [ ] "결제완료로 변경" 액션 실행
   - [ ] 상태 변경 및 색상 변경 확인
   - [ ] "배송중으로 변경" 액션 실행
   - [ ] 송장번호 입력

5. **통계 위젯**
   - [ ] 오늘 주문 수
   - [ ] 총 판매액
   - [ ] 상태별 주문 수 (파이 차트)
   - [ ] 취소된 주문 수

### 3.2 Admin 테스트 데이터 준비

각 Phase의 Admin 테스트를 위해 충분한 테스트 데이터를 준비해야 합니다.

```python
# scripts/create_test_data.py
from django.core.management.base import BaseCommand
from apps.accounts.models import CustomUser, Address
from apps.products.models import Product, Category
from apps.orders.models import Order, OrderItem

class Command(BaseCommand):
    help = '테스트 데이터 생성'

    def handle(self, *args, **options):
        # 회원 생성 (20명)
        for i in range(20):
            user = CustomUser.objects.create_user(
                email=f'user{i}@example.com',
                username=f'user{i}',
                password='testpass123'
            )
            # 배송지 2개씩
            Address.objects.create(
                user=user,
                name='집',
                recipient=f'사용자{i}',
                postal_code='12345',
                address=f'서울시 강남구 {i}번지'
            )

        # 카테고리 생성
        electronics = Category.objects.create(name='전자제품')
        Category.objects.create(name='노트북', parent=electronics)
        Category.objects.create(name='마우스', parent=electronics)

        fashion = Category.objects.create(name='패션')
        Category.objects.create(name='상의', parent=fashion)

        # 상품 생성 (50개)
        for i in range(50):
            Product.objects.create(
                category=electronics,
                name=f'상품 {i}',
                price=10000 + (i * 1000),
                stock_quantity=100 if i % 5 != 0 else 0,  # 5개 중 1개는 품절
                is_active=True if i % 10 != 0 else False  # 10개 중 1개는 비활성
            )

        self.stdout.write(self.style.SUCCESS('테스트 데이터 생성 완료!'))
```

실행:
```bash
python manage.py create_test_data
```

---

## 4. 테스트 도구 및 설정

### 4.1 pytest.ini

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = --reuse-db --nomigrations -v
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

### 4.2 conftest.py (공통 fixture)

```python
# conftest.py
import pytest
from rest_framework.test import APIClient
from apps.accounts.tests.factories import UserFactory
from apps.products.tests.factories import ProductFactory

@pytest.fixture
def api_client():
    """API 클라이언트"""
    return APIClient()

@pytest.fixture
def user():
    """테스트 사용자"""
    return UserFactory()

@pytest.fixture
def authenticated_client(api_client, user):
    """인증된 API 클라이언트"""
    api_client.force_authenticate(user=user)
    return api_client

@pytest.fixture
def product():
    """테스트 상품"""
    return ProductFactory()

@pytest.fixture
def products():
    """여러 테스트 상품"""
    return ProductFactory.create_batch(10)
```

### 4.3 Coverage 설정

```ini
# .coveragerc
[run]
source = apps
omit =
    */tests/*
    */migrations/*
    */admin.py
    */apps.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
```

---

## 5. CI/CD 통합

### 5.1 GitHub Actions

```yaml
# .github/workflows/test.yml
name: Django Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements/development.txt

      - name: Run tests
        run: |
          pytest --cov=apps --cov-report=xml
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost/test_db

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

---

## 6. 테스트 작성 체크리스트

### 각 Phase 완료 시 확인사항

- [ ] 모든 모델에 대한 단위 테스트 작성
- [ ] 비즈니스 로직 함수 테스트 작성
- [ ] API 엔드포인트 테스트 작성
  - [ ] 정상 케이스
  - [ ] 예외 케이스
  - [ ] 권한 테스트
- [ ] Admin 수동 테스트 체크리스트 완료
- [ ] 커버리지 80% 이상 달성
- [ ] CI 파이프라인 통과

---

## 7. 참고 자료

- [pytest 공식 문서](https://docs.pytest.org/)
- [Django REST Framework Testing](https://www.django-rest-framework.org/api-guide/testing/)
- [factory_boy 문서](https://factoryboy.readthedocs.io/)
- [pytest-django](https://pytest-django.readthedocs.io/)
