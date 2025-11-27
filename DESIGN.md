# Django 이커머스 프로젝트 설계 문서

## 1. 프로젝트 개요

온라인 쇼핑몰 플랫폼으로, 상품 관리, 주문 처리, 결제, 회원 관리 등 전자상거래의 핵심 기능을 제공합니다.

## 2. 시스템 아키텍처

### 2.1 전체 구조
```
┌─────────────────────────────────────────────┐
│           Frontend (Optional)               │
│        (React/Vue 또는 Django Templates)    │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│            Django REST API                  │
│  ┌────────────────────────────────────────┐ │
│  │  Authentication & Authorization        │ │
│  │  (Django Auth + Social Auth)           │ │
│  └────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────┐ │
│  │  Business Logic Layer                  │ │
│  │  - 상품 관리                            │ │
│  │  - 장바구니                             │ │
│  │  - 주문 처리                            │ │
│  │  - 결제 통합                            │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│          PostgreSQL Database                │
└─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│        External Services                    │
│  - 결제 게이트웨이 (PortOne/Toss)          │
│  - 소셜 로그인 (Kakao, Naver, Google)      │
│  - 이미지 스토리지 (AWS S3 or Local)       │
└─────────────────────────────────────────────┘
```

### 2.2 Django 앱 구조
```
django-ecommerce/
├── config/                 # 프로젝트 설정
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── accounts/          # 회원 관리
│   │   ├── models.py      # User 모델 확장
│   │   ├── views.py
│   │   ├── serializers.py
│   │   └── urls.py
│   ├── products/          # 상품 관리
│   │   ├── models.py      # Product, Category, Image
│   │   ├── views.py
│   │   ├── serializers.py
│   │   └── urls.py
│   ├── cart/              # 장바구니
│   │   ├── models.py      # Cart, CartItem
│   │   ├── views.py
│   │   └── urls.py
│   ├── orders/            # 주문 관리
│   │   ├── models.py      # Order, OrderItem
│   │   ├── views.py
│   │   └── urls.py
│   ├── payments/          # 결제 처리
│   │   ├── models.py      # Payment, Transaction
│   │   ├── views.py
│   │   ├── services.py    # 결제 API 통합
│   │   └── urls.py
│   └── reviews/           # 상품 리뷰
│       ├── models.py      # Review, Rating
│       ├── views.py
│       └── urls.py
├── media/                 # 업로드된 파일
├── static/                # 정적 파일
├── templates/             # 템플릿 (필요시)
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
├── manage.py
└── README.md
```

## 3. 데이터베이스 모델 설계

### 3.1 회원 관리 (accounts)
```python
# CustomUser (Django User 확장)
- id (PK)
- email (unique)
- username
- phone_number
- date_of_birth
- profile_image
- is_active
- is_staff
- date_joined
- last_login
- provider (null, 'kakao', 'naver', 'google')
- provider_id

# Address (배송지)
- id (PK)
- user (FK)
- name (배송지명)
- recipient
- phone
- postal_code
- address
- detail_address
- is_default
- created_at
```

### 3.2 상품 관리 (products)
```python
# Category (카테고리)
- id (PK)
- name
- slug
- parent (FK to self, nullable)
- description
- is_active
- created_at

# Product (상품)
- id (PK)
- category (FK)
- name
- slug
- description
- price
- discount_price (nullable)
- stock_quantity
- is_active
- is_featured
- view_count
- created_at
- updated_at

# ProductImage (상품 이미지)
- id (PK)
- product (FK)
- image
- is_main
- order
- created_at

# ProductOption (상품 옵션)
- id (PK)
- product (FK)
- name (예: 색상, 사이즈)
- value
- price_adjustment
- stock_quantity
```

### 3.3 장바구니 (cart)
```python
# Cart (장바구니)
- id (PK)
- user (FK, nullable) # 비회원 대응
- session_key (nullable)
- created_at
- updated_at

# CartItem (장바구니 아이템)
- id (PK)
- cart (FK)
- product (FK)
- product_option (FK, nullable)
- quantity
- created_at
- updated_at
```

### 3.4 주문 관리 (orders)
```python
# Order (주문)
- id (PK)
- user (FK)
- order_number (unique)
- status (pending, paid, shipped, delivered, cancelled)
- total_amount
- discount_amount
- shipping_fee
- final_amount
- # 배송 정보
- recipient_name
- recipient_phone
- postal_code
- address
- detail_address
- delivery_message
- # 타임스탬프
- created_at
- paid_at
- shipped_at
- delivered_at

# OrderItem (주문 아이템)
- id (PK)
- order (FK)
- product (FK)
- product_option (FK, nullable)
- product_name (스냅샷)
- product_price (스냅샷)
- quantity
- subtotal
```

### 3.5 결제 (payments)
```python
# Payment (결제)
- id (PK)
- order (FK)
- payment_method (card, bank_transfer, virtual_account, etc.)
- payment_provider (portone, toss, etc.)
- transaction_id (외부 결제 시스템 ID)
- amount
- status (pending, completed, failed, cancelled)
- paid_at
- cancelled_at
- failure_reason
- created_at
```

### 3.6 리뷰 (reviews)
```python
# Review (리뷰)
- id (PK)
- product (FK)
- user (FK)
- order_item (FK) # 구매 확인용
- rating (1-5)
- title
- content
- images (JSONField or separate table)
- is_verified_purchase
- helpful_count
- created_at
- updated_at
```

## 4. 주요 기능별 요구사항

### 4.1 회원 관리
- [x] 이메일 회원가입/로그인
- [x] 소셜 로그인 (카카오, 네이버, 구글)
- [x] 프로필 관리
- [x] 배송지 관리 (다중 배송지 지원)
- [x] 비밀번호 찾기/재설정

### 4.2 상품 관리
- [x] 상품 CRUD
- [x] 카테고리별 분류
- [x] 다중 이미지 업로드
- [x] 상품 옵션 (색상, 사이즈 등)
- [x] 재고 관리
- [x] 상품 검색 및 필터링

### 4.3 장바구니
- [x] 장바구니 추가/수정/삭제
- [x] 수량 조절
- [x] 비회원 장바구니 (세션 기반)
- [x] 회원 전환 시 장바구니 병합

### 4.4 주문 및 결제
- [x] 주문서 작성
- [x] 배송지 선택/입력
- [x] 결제 수단 선택
- [x] 결제 처리 (PortOne/Toss API 연동)
- [x] 주문 내역 조회
- [x] 주문 취소/환불

### 4.5 관리자 기능
- [x] Django Admin 커스터마이징
- [x] 상품 관리
- [x] 주문 관리
- [x] 회원 관리
- [x] 통계 대시보드

### 4.6 리뷰 시스템
- [x] 리뷰 작성 (구매자만 가능)
- [x] 별점 및 텍스트 리뷰
- [x] 이미지 첨부
- [x] 도움됨 기능

## 5. API 엔드포인트 설계

### 5.1 인증 (Authentication)
```
POST   /api/auth/register/              # 회원가입
POST   /api/auth/login/                 # 로그인
POST   /api/auth/logout/                # 로그아웃
POST   /api/auth/refresh/               # 토큰 갱신
POST   /api/auth/password/reset/        # 비밀번호 재설정 요청
POST   /api/auth/password/confirm/      # 비밀번호 재설정 확인

# 소셜 로그인
GET    /api/auth/social/{provider}/     # 소셜 로그인 시작
POST   /api/auth/social/{provider}/callback/  # 콜백
```

### 5.2 회원 (Accounts)
```
GET    /api/accounts/me/                # 내 정보 조회
PUT    /api/accounts/me/                # 내 정보 수정
GET    /api/accounts/addresses/         # 배송지 목록
POST   /api/accounts/addresses/         # 배송지 추가
PUT    /api/accounts/addresses/{id}/    # 배송지 수정
DELETE /api/accounts/addresses/{id}/    # 배송지 삭제
```

### 5.3 상품 (Products)
```
GET    /api/products/                   # 상품 목록
GET    /api/products/{id}/              # 상품 상세
GET    /api/products/search/            # 상품 검색
GET    /api/categories/                 # 카테고리 목록
GET    /api/categories/{id}/products/   # 카테고리별 상품
```

### 5.4 장바구니 (Cart)
```
GET    /api/cart/                       # 장바구니 조회
POST   /api/cart/items/                 # 장바구니 추가
PUT    /api/cart/items/{id}/            # 수량 변경
DELETE /api/cart/items/{id}/            # 항목 삭제
DELETE /api/cart/clear/                 # 장바구니 비우기
```

### 5.5 주문 (Orders)
```
GET    /api/orders/                     # 주문 내역
POST   /api/orders/                     # 주문 생성
GET    /api/orders/{id}/                # 주문 상세
POST   /api/orders/{id}/cancel/         # 주문 취소
```

### 5.6 결제 (Payments)
```
POST   /api/payments/prepare/           # 결제 준비
POST   /api/payments/complete/          # 결제 완료 확인
POST   /api/payments/webhook/           # 결제 웹훅
POST   /api/payments/{id}/cancel/       # 결제 취소
```

### 5.7 리뷰 (Reviews)
```
GET    /api/products/{id}/reviews/      # 상품 리뷰 목록
POST   /api/reviews/                    # 리뷰 작성
PUT    /api/reviews/{id}/               # 리뷰 수정
DELETE /api/reviews/{id}/               # 리뷰 삭제
POST   /api/reviews/{id}/helpful/       # 도움됨 추가
```

## 6. 보안 고려사항

1. **인증/인가**
   - JWT 토큰 기반 인증
   - CSRF 보호
   - Rate Limiting

2. **결제 보안**
   - PCI DSS 준수 (결제 정보는 직접 저장 안함)
   - HTTPS 필수
   - 웹훅 검증

3. **데이터 보호**
   - 개인정보 암호화
   - SQL Injection 방지
   - XSS 방지

4. **접근 제어**
   - 권한 기반 접근 제어
   - 본인 데이터만 접근 가능

## 7. 성능 최적화

1. **데이터베이스**
   - 인덱스 최적화
   - Query 최적화 (select_related, prefetch_related)
   - 커넥션 풀링

2. **캐싱**
   - Redis 캐싱 (상품 목록, 카테고리)
   - Django 캐시 프레임워크

3. **이미지 최적화**
   - 썸네일 생성
   - CDN 활용 (선택사항)

4. **비동기 처리**
   - Celery (이메일 발송, 주문 처리 등)
   - 백그라운드 작업 큐

## 8. 테스트 전략 🧪

테스트는 **3단계**로 진행되며, 각 Phase마다 개발과 함께 작성됩니다.

### 8.1 단위 테스트 (Unit Tests) - 개발 단계

**목적**: 개별 컴포넌트의 정확성 검증

**언제**: 각 Phase에서 코드와 함께 작성

**대상**:
1. **모델 테스트**
   - 모델 생성 및 필드 검증
   - 제약 조건 (unique, nullable 등)
   - 커스텀 메서드 로직
   - 관계 (ForeignKey, ManyToMany)

   ```python
   # 예시: Product 모델 테스트
   def test_product_creation():
       product = Product.objects.create(
           name="테스트 상품",
           price=10000,
           stock_quantity=100
       )
       assert product.price == 10000
       assert product.is_available() == True

   def test_out_of_stock():
       product = Product.objects.create(
           name="품절 상품",
           price=10000,
           stock_quantity=0
       )
       assert product.is_available() == False
   ```

2. **비즈니스 로직 테스트**
   - 서비스 계층 함수
   - 유틸리티 함수
   - 계산 로직 (가격, 할인, 재고 등)

   ```python
   # 예시: 주문 생성 로직 테스트
   def test_create_order_from_cart():
       cart = create_test_cart()
       order = OrderService.create_from_cart(cart, user, address)
       assert order.total_amount == cart.calculate_total()
       assert order.items.count() == cart.items.count()
   ```

3. **권한 및 검증 로직**
   - 접근 권한 검사
   - 데이터 검증
   - 예외 처리

**도구**:
- pytest
- pytest-django
- factory-boy (테스트 데이터 생성)
- faker (더미 데이터)

### 8.2 API 테스트 - 개발 단계

**목적**: API 엔드포인트의 정확성 및 보안 검증

**언제**: 각 Phase에서 API 구현과 함께 작성

**대상**:
1. **CRUD 테스트**
   - 생성, 조회, 수정, 삭제 동작
   - 페이지네이션
   - 필터링 및 검색
   - 정렬

2. **인증/권한 테스트**
   - JWT 토큰 검증
   - 로그인/로그아웃
   - 권한별 접근 제어
   - 본인 데이터만 접근 가능 여부

3. **예외 처리 테스트**
   - 잘못된 데이터 입력
   - 존재하지 않는 리소스
   - 권한 없음 (403)
   - 인증 실패 (401)

4. **외부 API Mock 테스트**
   - 결제 API (PortOne/Toss)
   - 소셜 로그인 API

   ```python
   # 예시: 결제 API Mock 테스트
   @patch('payments.services.PortOneAPI.request_payment')
   def test_payment_request(mock_payment):
       mock_payment.return_value = {'status': 'success'}
       response = client.post('/api/payments/prepare/', data)
       assert response.status_code == 200
   ```

**도구**:
- Django REST Framework TestCase
- APIClient
- Mock/patch (unittest.mock)

### 8.3 Admin 수동 테스트 - 각 Phase 완료 후

**목적**: 관리자 기능 및 통계 위젯 확인

**언제**: 각 Phase 완료 시 Admin 페이지에서 수동 확인

**체크리스트** (Phase별):

**Phase 2 - 회원 관리**:
- [ ] 회원 목록 표시 여부
- [ ] 검색 및 필터 동작
- [ ] 배송지 Inline 표시
- [ ] 활성화/비활성화 액션
- [ ] 통계 위젯 (회원 수, 가입자)

**Phase 3 - 상품 관리**:
- [ ] 카테고리 계층 구조
- [ ] 상품 이미지 다중 업로드
- [ ] 가격/재고 빠른 수정
- [ ] 일괄 액션 (활성화, 가격 조정)
- [ ] 통계 위젯 (품절 상품 수)

**Phase 5 - 주문 관리**:
- [ ] 주문 목록 및 상태별 색상
- [ ] OrderItem Inline
- [ ] 상태 일괄 변경
- [ ] 통계 위젯 (판매액, 주문 수)

**목적**:
- 실제 사용 시나리오 검증
- UI/UX 확인
- 통계 데이터 정확성
- 액션 동작 확인

### 8.4 통합 테스트 (Integration Tests) - Phase 12

**목적**: 여러 컴포넌트 간 상호작용 검증

**언제**: Phase 12에서 전체 기능 완성 후

**대상**:
1. **전체 주문 플로우**
   ```
   회원가입 → 로그인 → 상품 조회 → 장바구니 추가
   → 주문 생성 → 결제 → 주문 완료
   ```

2. **결제 플로우**
   ```
   주문 생성 → 결제 준비 → 결제 완료 → 웹훅 처리
   → 주문 상태 업데이트 → 재고 차감
   ```

3. **취소/환불 플로우**
   ```
   주문 취소 요청 → 결제 취소 API → 재고 복구
   → 주문 상태 변경
   ```

4. **리뷰 작성 플로우**
   ```
   주문 완료 → 배송 완료 → 리뷰 작성 권한 확인
   → 리뷰 작성 → 평균 별점 업데이트
   ```

### 8.5 E2E 테스트 (End-to-End Tests) - Phase 12

**목적**: 실제 사용자 관점에서 전체 시스템 동작 검증

**도구**: Selenium, Playwright (선택사항)

**시나리오**:
1. **신규 사용자 회원가입 및 첫 구매**
2. **소셜 로그인 사용자 상품 구매**
3. **장바구니 방치 후 재접속하여 구매**
4. **주문 취소 및 환불 요청**
5. **리뷰 작성 및 수정**

### 8.6 테스트 커버리지 목표

- **전체 코드 커버리지**: 80% 이상
- **핵심 비즈니스 로직**: 95% 이상
- **모델**: 100%
- **API 엔드포인트**: 90% 이상

**도구**: pytest-cov

```bash
pytest --cov=apps --cov-report=html
```

### 8.7 테스트 자동화

**CI/CD 파이프라인**:
1. Git Push 시 자동 테스트 실행
2. PR 생성 시 테스트 통과 확인
3. 커버리지 리포트 생성
4. 배포 전 필수 테스트 통과

**GitHub Actions 예시**:
```yaml
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: |
          pip install -r requirements/dev.txt
          pytest --cov=apps
```

### 8.8 테스트 데이터 관리

**Factory Pattern 사용**:
```python
# factories.py
import factory
from apps.accounts.models import CustomUser
from apps.products.models import Product

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = CustomUser

    email = factory.Faker('email')
    username = factory.Faker('user_name')

class ProductFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Product

    name = factory.Faker('word')
    price = factory.Faker('random_int', min=1000, max=100000)
    stock_quantity = factory.Faker('random_int', min=0, max=1000)
```

**Fixture 사용**:
```python
# conftest.py
import pytest

@pytest.fixture
def authenticated_client(client, user_factory):
    user = user_factory()
    client.force_authenticate(user=user)
    return client

@pytest.fixture
def product_with_images(product_factory, image_factory):
    product = product_factory()
    image_factory.create_batch(3, product=product)
    return product
```

### 8.9 Phase별 테스트 전략 요약

| Phase | 단위 테스트 | API 테스트 | Admin 수동 테스트 |
|-------|------------|-----------|------------------|
| 2. 회원 | 모델, 인증 로직 | 회원가입, 로그인 API | 회원 목록, 통계 |
| 3. 상품 | 모델, 재고 로직 | 상품 CRUD API | 상품 관리, 이미지 |
| 4. 장바구니 | 모델, 병합 로직 | 장바구니 API | 장바구니 목록 |
| 5. 주문 | 모델, 주문 생성 | 주문 API | 주문 관리, 상태 |
| 6. 결제 | 결제 로직 (Mock) | 결제 API (Mock) | 결제 내역 |
| 7. 소셜 | 연동 로직 | 소셜 로그인 API | 소셜 계정 목록 |
| 8. 리뷰 | 모델, 권한 로직 | 리뷰 API | 리뷰 관리 |
| 12. 통합 | - | 전체 플로우 | - |

## 9. 배포 전략

1. **개발 환경**
   - Docker Compose
   - SQLite (빠른 개발용)

2. **프로덕션 환경**
   - PostgreSQL
   - Gunicorn + Nginx
   - AWS/GCP/Azure 또는 Docker 배포

## 10. Django Admin 점진적 발전 전략 🎨

Django Admin을 각 Phase마다 함께 발전시켜 실시간으로 데이터를 관리하고 모니터링할 수 있도록 합니다.

### Phase 1: 기본 Admin 설정
**목표**: Admin 기본 환경 구축
- Admin 사이트 타이틀 커스터마이징
- Superuser 생성
- 기본 인증 설정

### Phase 2: 회원 관리 Admin
**추가 기능**:
- CustomUser Admin 등록
  - 회원 목록 (이메일, 가입일, 활성 상태)
  - 검색 및 필터링 (가입일, 활성 상태)
  - 배송지 Inline 관리
- 관리 액션: 회원 활성화/비활성화
- **통계 위젯**: 총 회원 수, 오늘 가입자

### Phase 3: 상품 관리 Admin
**추가 기능**:
- Category Admin
  - 계층 구조 표시
  - 카테고리별 상품 수
- Product Admin
  - 썸네일 미리보기
  - 가격/재고 빠른 수정 (list_editable)
  - ProductImage Inline (다중 이미지)
  - ProductOption Inline (색상/사이즈)
- 관리 액션: 상품 일괄 활성화, 가격 일괄 조정
- **통계 위젯**: 총 상품 수, 품절 상품, 카테고리별 분포

### Phase 4: 장바구니 Admin
**추가 기능**:
- Cart Admin (읽기 전용)
  - 장바구니 목록
  - CartItem Inline
- **통계 위젯**: 활성 장바구니, 평균 금액, 방치된 장바구니

### Phase 5: 주문 관리 Admin
**추가 기능**:
- Order Admin
  - 주문 목록 (주문번호, 주문자, 금액, 상태)
  - 상태별 색상 구분
  - OrderItem Inline
  - 배송 정보 섹션
- 관리 액션: 주문 상태 일괄 변경, 송장번호 등록
- **통계 위젯**: 오늘 주문 수, 총 판매액, 상태별 주문 수

### Phase 6: 결제 관리 Admin
**추가 기능**:
- Payment Admin (읽기 전용)
  - 결제 내역 (주문번호, 결제수단, 금액, 상태)
  - 주문 상세와 연결
- 관리 액션: 결제 취소, 환불 처리
- **통계 위젯**: 오늘 결제 금액, 결제 성공률, 결제수단별 통계

### Phase 7: 소셜 로그인 Admin
**추가 기능**:
- SocialAccount Admin (allauth 기본)
  - 연동된 소셜 계정 목록
  - 제공자별 필터
- CustomUser Admin 확장
  - 소셜 계정 Inline
  - 로그인 방법 표시
- **통계 위젯**: 로그인 방법별 회원 수

### Phase 8: 리뷰 관리 Admin
**추가 기능**:
- Review Admin
  - 리뷰 목록 (상품, 작성자, 별점, 구매확인)
  - 리뷰 이미지 Inline
- 관리 액션: 부적절한 리뷰 숨김/삭제, 베스트 리뷰 선정
- Product Admin 확장: 평균 별점, 리뷰 개수 표시
- **통계 위젯**: 오늘 리뷰 수, 평균 별점, 별점 분포

### Phase 9: 통합 대시보드 (Admin 강화)
**추가 기능**:
- **종합 대시보드**
  - 총 판매액 (일간/주간/월간)
  - 주문 통계 차트
  - 회원 증가 추이
  - 인기 상품 Top 10
- **고급 필터 및 검색**
  - 날짜 범위 필터
  - 다중 조건 검색
- **일괄 작업 액션**
  - 엑셀 다운로드
  - 이메일 일괄 발송
- **권한 관리**
  - 스태프 권한 세분화
  - 모델별 접근 권한

### Phase 10-11: 성능 및 모니터링
**추가 기능**:
- Admin 쿼리 최적화
  - select_related/prefetch_related 적용
  - 대용량 데이터 페이지네이션 개선
- 통계 위젯 캐싱 (Redis)
- Celery 작업 모니터링 위젯
- Flower 대시보드 링크

### Admin 커스터마이징 기술 스택
- **django-admin-interface**: 모던한 테마 (선택사항)
- **django-import-export**: 엑셀 import/export
- **django-admin-rangefilter**: 날짜 범위 필터
- **django-admin-autocomplete-filter**: 자동완성 필터
- **Chart.js**: 통계 차트

### Admin UI 개선 요소
1. **색상 코딩**
   - 주문 상태별 색상 (빨강/파랑/초록)
   - 재고 부족 상품 강조
   - 결제 실패 표시

2. **인라인 편집**
   - list_editable로 빠른 수정
   - Inline으로 관련 데이터 함께 관리

3. **검색 최적화**
   - 자동완성 검색
   - 다중 필드 검색
   - 날짜 범위 필터

4. **통계 시각화**
   - Dashboard 위젯
   - 차트 및 그래프
   - 실시간 갱신

## 11. 다음 단계

1. 기술 스택 최종 확정
2. 개발 환경 설정
3. 기본 Django 프로젝트 생성
4. 각 앱별 순차적 구현 (Admin과 함께!)
