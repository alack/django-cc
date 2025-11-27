# 문서 일관성 검토 및 수정 사항

## 발견된 불일치 및 수정 필요 사항

### 1. TECH_STACK.md - Pydantic 추가 필요

**현재 상태**: requirements에 pydantic 누락
**수정 필요**:

```diff
# requirements/base.txt
+pydantic>=2.0
+pydantic-settings>=2.0  # 설정 객체용
```

---

### 2. TECH_STACK.md - Type Hints 도구 추가

**현재 상태**: mypy, django-stubs 누락
**수정 필요**:

```diff
# requirements/development.txt
+mypy>=1.7
+django-stubs>=4.2
+types-requests>=2.31
```

---

### 3. TECH_STACK.md - Admin 라이브러리 추가

**현재 상태**: Admin 커스터마이징 라이브러리 누락
**수정 필요**:

```diff
# requirements/base.txt (또는 development.txt)
+django-import-export>=3.3  # 엑셀 import/export
+django-admin-rangefilter>=0.11  # 날짜 범위 필터

# 선택사항
# django-admin-interface>=0.26  # 모던 테마
# django-admin-autocomplete-filter>=0.7  # 자동완성
```

---

### 4. DESIGN.md - Django 앱 구조 수정

**현재 상태**: services.py가 payments에만 표시됨

**수정 필요** (Section 2.2):
```diff
apps/
├── accounts/
│   ├── models.py
+│   ├── services.py      # 비즈니스 로직
│   ├── views.py
│   ├── serializers.py
+│   ├── schemas.py       # Pydantic 스키마
+│   ├── exceptions.py    # 커스텀 예외
│   └── urls.py
├── products/
│   ├── models.py
+│   ├── services.py
│   ├── views.py
│   ├── serializers.py
+│   ├── schemas.py
+│   ├── exceptions.py
│   └── urls.py
├── cart/
│   ├── models.py
+│   ├── services.py
│   ├── views.py
+│   ├── schemas.py
+│   ├── exceptions.py
│   └── urls.py
├── orders/
│   ├── models.py
+│   ├── services.py
│   ├── views.py
+│   ├── schemas.py
+│   ├── exceptions.py
│   └── urls.py
├── payments/
│   ├── models.py
│   ├── services.py       # 이미 있음 ✓
│   ├── views.py
+│   ├── schemas.py
+│   ├── exceptions.py
│   └── urls.py
└── reviews/
    ├── models.py
+   ├── services.py
    ├── views.py
+   ├── schemas.py
+   ├── exceptions.py
    └── urls.py
```

---

### 5. DESIGN.md - 아키텍처 용어 통일

**현재 상태** (Section 2.1):
```
│  Business Logic Layer  │
```

**수정 필요**:
```diff
┌─────────────────────────────────────────────┐
│            Django REST API                  │
│  ┌────────────────────────────────────────┐ │
│  │  Authentication & Authorization        │ │
│  └────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────┐ │
-│  │  Business Logic Layer                  │ │
+│  │  Service Layer (Business Logic)        │ │
│  │  - 상품 관리 (ProductService)           │ │
│  │  - 장바구니 (CartService)               │ │
│  │  - 주문 처리 (OrderService)             │ │
│  │  - 결제 통합 (PaymentService)           │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

---

### 6. TECH_STACK.md - Section 16 삭제 또는 수정

**현재 상태**: Phase 구분이 ROADMAP.md와 완전히 다름

**Option A: 삭제** (권장)
```diff
-## 16. 구현 우선순위
-### Phase 1: 기본 구조 (1-2주)
-...
+## 16. 구현 로드맵
+
+자세한 Phase별 구현 계획은 **ROADMAP.md**를 참조하세요.
+
+- Phase 1-2: 프로젝트 초기 설정 및 회원 관리
+- Phase 3-5: 상품, 장바구니, 주문
+- Phase 6-7: 결제, 소셜 로그인
+- Phase 8-11: 리뷰, Admin 강화, 최적화
+- Phase 12-15: 테스트, 문서화, 배포
```

**Option B: ROADMAP.md와 통일**
- TECH_STACK.md의 Phase 1~3을 ROADMAP.md의 Phase 1~16 구조로 변경

---

### 7. CODING_STYLE.md - Admin 커스터마이징 예제 추가 (선택사항)

**현재 상태**: Admin 코드 예제 없음

**추가 권장** (Section 5 Django 특화 규칙):
```python
# ✅ 좋은 예: Admin 커스터마이징
from django.contrib import admin
from .models import Order, OrderItem

class OrderItemInline(admin.TabularInline):
    model = OrderItem
    readonly_fields = ('product_name', 'price', 'quantity', 'subtotal')
    extra = 0
    can_delete = False

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = (
        'order_number',
        'user_email',
        'total_amount',
        'status_colored',
        'created_at'
    )
    list_filter = ('status', 'created_at')
    search_fields = ('order_number', 'user__email')
    readonly_fields = ('order_number', 'created_at')
    inlines = [OrderItemInline]

    def user_email(self, obj: Order) -> str:
        return obj.user.email
    user_email.short_description = '주문자'

    def status_colored(self, obj: Order) -> str:
        colors = {
            'pending': 'gray',
            'paid': 'blue',
            'shipped': 'orange',
            'delivered': 'green',
        }
        color = colors.get(obj.status, 'black')
        return format_html(
            '<span style="color: {};">{}</span>',
            color,
            obj.get_status_display()
        )
    status_colored.short_description = '상태'
```

---

## 우선순위별 수정 계획

### 🔴 High Priority (즉시 수정 필요)
1. ✅ TECH_STACK.md - Pydantic 추가
2. ✅ TECH_STACK.md - mypy, django-stubs 추가
3. ✅ DESIGN.md - Django 앱 구조 수정 (services.py, schemas.py 추가)

### 🟡 Medium Priority (중요)
4. ✅ DESIGN.md - "Business Logic Layer" → "Service Layer" 용어 통일
5. ✅ TECH_STACK.md - Admin 라이브러리 추가
6. ✅ TECH_STACK.md - Section 16 수정/삭제

### 🟢 Low Priority (선택사항)
7. CODING_STYLE.md - Admin 예제 추가

---

## 수정 후 검증 체크리스트

- [ ] 모든 문서에서 "Service Layer" 용어 통일
- [ ] 모든 Django 앱에 services.py, schemas.py 포함
- [ ] requirements 파일에 pydantic, mypy 포함
- [ ] TECH_STACK.md의 Phase가 ROADMAP.md 참조
- [ ] Admin 라이브러리가 TECH_STACK.md에 포함
- [ ] 각 문서 간 상호 참조 추가 (예: "자세한 내용은 XXX.md 참조")

---

## 수정 작업 진행 여부

사용자 승인 후 위 수정사항을 반영하겠습니다.

1. 모든 수정사항을 한 번에 적용할까요?
2. 우선순위별로 단계적으로 적용할까요?
3. 특정 항목만 선택적으로 적용할까요?
