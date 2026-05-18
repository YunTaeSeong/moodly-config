# Moodly Config

[moodly-api](https://github.com/YunTaeSeong/moodly-api) **Config Server**가 이 Git 저장소의 YAML을 읽어 각 마이크로서비스에 설정을 제공합니다.

| 저장소 | 역할 |
|--------|------|
| [moodly-api](https://github.com/YunTaeSeong/moodly-api) | 서비스 소스, Docker Compose |
| **moodly-config** (본 저장소) | 중앙 설정 (YAML) |
| [moodly-web](https://github.com/YunTaeSeong/moodly-web) | React 프론트 |

실행 순서·전체 아키텍처: [moodly-api README](https://github.com/YunTaeSeong/moodly-api/blob/main/README.md)

---

## Config Server 연동

```yaml
# moodly-api/configserver — 기본
spring.cloud.config.server.git.uri: https://github.com/YunTaeSeong/moodly-config.git
```

각 마이크로서비스:

```yaml
spring.config.import: optional:configserver:${SPRING_CLOUD_CONFIG_URI:http://localhost:8071}
```

Docker Compose는 `SPRING_CLOUD_CONFIG_URI=http://moodly-configserver:8071` 을 주입합니다.

**Profile 병합:** `{application}.yml` + `{application}-{profile}.yml`  
예: `auth-service` + `docker` → `auth-service.yml` + `auth-service-docker.yml`

---

## 파일 규칙

| 패턴 | 용도 |
|------|------|
| `{service}.yml` | 포트, JPA, Feign, 공통 설정 |
| `{service}-local.yml` | IDE: `localhost:331x`, `localhost:6379` |
| `{service}-docker.yml` | Docker: `moodly-*-db`, `moodly-redis` |
| `{service}-test.yml` | 테스트 프로필 |
| `gateway-service.yml` | 라우트, JWT permit-all, RateLimiter, CB, CORS |
| `eurekaserver.yml` | Eureka Server (`register-with-eureka: false`) |

`spring.application.name`과 파일명 접두사가 **동일**해야 합니다.

예: `auth-service` → `auth-service.yml`

---

## 저장소 파일 목록

| 서비스 | 공통 | local | docker | test |
|--------|------|-------|--------|------|
| auth-service | ✅ | ✅ | ✅ | ✅ |
| user-service | ✅ | ✅ | ✅ | ✅ |
| product-service | ✅ | ✅ | ✅ | ✅ |
| cart-service | ✅ | ✅ | ✅ | ✅ |
| order-service | ✅ | ✅ | ✅ | ✅ |
| notification-service | ✅ | ✅ | ✅ | ✅ |
| coupon-service | ✅ | ✅ | ✅ | ✅ |
| payment-service | ✅ | ✅ | ✅ | ✅ |
| gateway-service | ✅ | ✅ | ✅ | ✅ |
| eurekaserver | ✅ | — | — | — |

---

## Gateway (`gateway-service.yml`)

| 항목 | 내용 |
|------|------|
| 포트 | `8072` |
| 라우트 | `/AUTH-SERVICE/**` → `lb://AUTH-SERVICE` 등 |
| Rate Limiter | AUTH 라우트 (Redis) |
| Circuit Breaker | auth, product, order, payment, coupon |
| CORS | `globalcors` — `http://localhost:3000` |
| 헤더 중복 방지 | `DedupeResponseHeader` (하위 MSA CORS와 중복 시) |

프론트는 **항상 Gateway(8072)** 로 호출합니다. Docker `auth-service`는 `docker` 프로필에서 자체 CORS를 끄고 Gateway에서만 처리합니다.

Redis: `gateway-service-local.yml` → `localhost`, `gateway-service-docker.yml` → `moodly-redis`

### 구매후기 공개 API 예시

`gateway-service.yml`에서 상품별 후기 목록은 인증 없이 허용됩니다.

- `GET /PRODUCT-SERVICE/product/review/product/**` → permit-all

---

## 설정 분리 원칙

| 위치 | 내용 |
|------|------|
| `*-service.yml` | 비밀이 아닌 공통 설정 |
| `*-local.yml` / `*-docker.yml` | DB·Redis·Eureka URL 등 환경별 |
| `${ENV}` placeholder | `JWT_SECRET`, `DB_PASSWORD`, `KAKAO_*` 등 — **실값 커밋 금지** |

---

## 로컬 MySQL 포트 (`*-local.yml`)

| 서비스 | 호스트 포트 | DB 이름 |
|--------|-------------|---------|
| auth | 3310 | moodly_auth |
| user | 3311 | moodly_user |
| product | 3312 | moodly_product |
| cart | 3313 | moodly_cart |
| order | 3314 | moodly_order |
| notification | 3315 | moodly_notification |
| coupon | 3316 | moodly_coupon |
| payment | 3317 | moodly_payment |

Docker Compose의 `moodly-*-db` 포트 매핑과 동일합니다.

---

## 설정 변경 워크플로

```text
1. moodly-config YAML 수정
2. git commit & push (main)
3. Config Server / 해당 마이크로서비스 재시작
```

Fork를 사용하는 경우 `moodly-api/configserver`의 `git.uri`를 fork URL로 변경하세요.

---

## 트러블슈팅

| 증상 | 확인 |
|------|------|
| 설정 미적용 | push 후 서비스 재시작, Config Server(8071) 기동 |
| Gateway 404 | `gateway-service.yml` Path·Eureka 서비스 등록 |
| CORS 중복 | `globalcors` + MSA CORS 동시 적용, `DedupeResponseHeader` |
| local DB 연결 실패 | `*-local.yml` 포트와 Docker MySQL(3310~) 일치 |

---

## 관련 문서

- [moodly-api README](https://github.com/YunTaeSeong/moodly-api/blob/main/README.md)
- [moodly-web README](https://github.com/YunTaeSeong/moodly-web/blob/main/README.md)
