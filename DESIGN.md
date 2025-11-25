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

## 8. 테스트 전략

1. **단위 테스트**
   - 모델 테스트
   - 비즈니스 로직 테스트

2. **통합 테스트**
   - API 엔드포인트 테스트
   - 결제 플로우 테스트

3. **E2E 테스트**
   - 주요 사용자 플로우 테스트

## 9. 배포 전략

1. **개발 환경**
   - Docker Compose
   - SQLite (빠른 개발용)

2. **프로덕션 환경**
   - PostgreSQL
   - Gunicorn + Nginx
   - AWS/GCP/Azure 또는 Docker 배포

## 10. 다음 단계

1. 기술 스택 최종 확정
2. 개발 환경 설정
3. 기본 Django 프로젝트 생성
4. 각 앱별 순차적 구현
