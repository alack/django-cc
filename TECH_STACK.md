# 기술 스택 및 라이브러리

## 1. 백엔드 프레임워크

### Core Framework
- **Python 3.11+**
- **Django 5.0+** - 메인 웹 프레임워크
- **Django REST Framework (DRF) 3.14+** - RESTful API 구축

### 데이터베이스
- **PostgreSQL 15+** - 메인 데이터베이스
- **psycopg2-binary** - PostgreSQL 어댑터

## 2. 인증 및 권한

### 기본 인증
- **djangorestframework-simplejwt** - JWT 토큰 기반 인증
- **django-allauth** - 회원가입/로그인 기능

### 소셜 로그인
- **dj-rest-auth** - REST API 인증
- **django-allauth[socialaccount]** - 소셜 로그인 기능
  - Kakao Provider
  - Naver Provider (커스텀 또는 django-allauth-naver)
  - Google Provider

## 3. 이미지 및 파일 처리

- **Pillow** - 이미지 처리
- **django-imagekit** - 이미지 썸네일 자동 생성
- **django-storages** - AWS S3 연동 (선택사항)
- **boto3** - AWS SDK (S3 사용 시)

## 4. 결제 시스템

### 결제 게이트웨이 옵션
1. **PortOne (구 아임포트)** - 권장
   - 한국 주요 PG사 통합 지원
   - 다양한 결제 수단 지원
   - 라이브러리: `requests` (REST API 호출)

2. **Toss Payments**
   - 간편 결제
   - 라이브러리: `requests` (REST API 호출)

## 5. 캐싱 및 성능

- **Redis 7+** - 캐싱 및 세션 저장소
- **django-redis** - Django-Redis 통합
- **hiredis** - Redis 성능 향상

## 6. 비동기 작업

- **Celery 5.3+** - 비동기 작업 큐
- **django-celery-beat** - 주기적 작업 스케줄링
- **flower** - Celery 모니터링 (개발용)

## 7. 유틸리티

### 개발 생산성
- **django-debug-toolbar** - 디버깅 도구
- **django-extensions** - 유용한 확장 기능
- **ipython** - 향상된 Python 셸

### 코드 품질
- **black** - 코드 포매터
- **flake8** - 린터
- **isort** - Import 정렬
- **pylint** - 코드 분석

### 환경 설정
- **python-decouple** - 환경 변수 관리
- **python-dotenv** - .env 파일 지원

## 8. API 문서화

- **drf-spectacular** - OpenAPI 3.0 스키마 생성
- **drf-spectacular[sidecar]** - Swagger UI 내장

## 9. 데이터 검증

- **django-phonenumber-field** - 전화번호 검증
- **phonenumbers** - 국제 전화번호 처리

## 10. CORS 및 보안

- **django-cors-headers** - CORS 처리
- **django-ratelimit** - API Rate Limiting
- **django-csp** - Content Security Policy

## 11. 테스팅

- **pytest** - 테스트 프레임워크
- **pytest-django** - Django 테스트 지원
- **pytest-cov** - 코드 커버리지
- **factory-boy** - 테스트 데이터 생성
- **faker** - 더미 데이터 생성

## 12. 배포 및 서버

- **gunicorn** - WSGI 서버
- **whitenoise** - 정적 파일 서빙
- **sentry-sdk** - 에러 트래킹 (선택사항)

## 13. 프론트엔드 (선택사항)

### 옵션 1: Django Templates (서버 사이드 렌더링)
- **django-crispy-forms** - 폼 스타일링
- **crispy-bootstrap5** - Bootstrap 5 지원

### 옵션 2: 분리된 프론트엔드
- React + TypeScript
- Next.js
- Vue.js + Nuxt

## 14. 추천 패키지 구성

### requirements/base.txt
```
Django>=5.0,<5.1
djangorestframework>=3.14
psycopg2-binary>=2.9
Pillow>=10.0
django-environ>=0.11
djangorestframework-simplejwt>=5.3
django-allauth>=0.57
dj-rest-auth>=5.0
django-cors-headers>=4.3
django-phonenumber-field>=7.2
phonenumbers>=8.13
django-imagekit>=5.0
redis>=5.0
django-redis>=5.4
celery>=5.3
django-celery-beat>=2.5
requests>=2.31
drf-spectacular>=0.27
pydantic>=2.0
pydantic-settings>=2.0
```

### requirements/development.txt
```
-r base.txt
django-debug-toolbar>=4.2
django-extensions>=3.2
ipython>=8.17
black>=23.11
flake8>=6.1
isort>=5.12
mypy>=1.7
django-stubs>=4.2
types-requests>=2.31
pytest>=7.4
pytest-django>=4.7
pytest-cov>=4.1
factory-boy>=3.3
faker>=20.1
flower>=2.0
django-import-export>=3.3
django-admin-rangefilter>=0.11
```

### requirements/production.txt
```
-r base.txt
gunicorn>=21.2
whitenoise>=6.6
sentry-sdk>=1.38
django-storages>=1.14
boto3>=1.29
hiredis>=2.2
```

## 15. 인프라 및 데이터베이스

### 개발 환경
- Docker + Docker Compose
- PostgreSQL (Docker 컨테이너)
- Redis (Docker 컨테이너)

### 프로덕션 환경
- PostgreSQL (managed service 권장)
- Redis (managed service 또는 Docker)
- Nginx (리버스 프록시)
- Supervisor 또는 systemd (프로세스 관리)

## 16. 구현 로드맵

자세한 Phase별 구현 계획은 **ROADMAP.md**를 참조하세요.

### 로드맵 개요

- **Phase 1-2**: 프로젝트 초기 설정 및 회원 관리
  - Django 프로젝트 설정, Docker 환경 구성
  - CustomUser 모델, 인증 시스템
  - 회원 관리 Admin 페이지

- **Phase 3-5**: 상품, 장바구니, 주문
  - Product/Category 모델 및 CRUD API
  - 장바구니 기능 및 세션 관리
  - 주문 시스템 및 재고 관리

- **Phase 6-7**: 결제 및 소셜 로그인
  - PortOne/Toss 결제 연동
  - Kakao, Naver, Google 소셜 로그인

- **Phase 8-11**: 리뷰, Admin 강화, 최적화
  - 리뷰 시스템, 이미지 업로드
  - Admin 통계 대시보드 및 고급 기능
  - Redis 캐싱, Celery 비동기 작업

- **Phase 12-15**: 테스트, 문서화, 배포
  - 통합 테스트, E2E 테스트
  - API 문서 자동화 (drf-spectacular)
  - 보안 강화 및 프로덕션 배포
  - 성능 최적화 및 모니터링

## 17. 개발 도구

### IDE
- PyCharm Professional (권장)
- VS Code + Python Extensions

### API 테스팅
- Postman
- HTTPie
- curl

### 데이터베이스 관리
- pgAdmin
- DBeaver
- TablePlus

### Git
- Git + GitHub/GitLab
- Conventional Commits

## 18. 모니터링 (프로덕션)

- **Sentry** - 에러 트래킹
- **New Relic** - APM (선택사항)
- **Prometheus + Grafana** - 메트릭 모니터링
- **ELK Stack** - 로그 분석 (대규모 서비스)

## 19. 보안 체크리스트

- [ ] HTTPS 적용
- [ ] CSRF 보호 활성화
- [ ] XSS 방지
- [ ] SQL Injection 방지
- [ ] Rate Limiting 적용
- [ ] 민감 정보 암호화
- [ ] 환경 변수로 시크릿 관리
- [ ] 정기적인 의존성 업데이트
- [ ] 보안 헤더 설정
- [ ] 결제 정보 직접 저장 금지

## 20. 다음 액션 아이템

1. **즉시 시작 가능한 작업**
   - Django 프로젝트 생성
   - 가상환경 설정
   - requirements 파일 작성
   - Docker Compose 설정

2. **결정이 필요한 사항**
   - 프론트엔드 방식 (Django Templates vs 분리된 SPA)
   - 결제 게이트웨이 선택 (PortOne vs Toss)
   - 이미지 저장소 (로컬 vs S3)
   - 배포 플랫폼 (AWS vs GCP vs Azure vs 온프레미스)

3. **선택적 기능**
   - 실시간 알림 (WebSocket, Django Channels)
   - 재고 알림 (Celery + 이메일/SMS)
   - 상품 추천 시스템
   - 쿠폰/할인 시스템
   - 포인트/적립금 시스템
