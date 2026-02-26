# MSA 아키텍처 로드맵

> 4가지 아키텍처 비교 + 인증 처리 방식의 진화

---

## 목차

1. [아키텍처 1: Ingress + auth-url](#1-아키텍처-1-ingress--auth-url)
2. [아키텍처 2: Gateway API + 각 서비스 인증](#2-아키텍처-2-gateway-api--각-서비스-인증)
3. [아키텍처 3: Gateway 인증 내장 (미래)](#3-아키텍처-3-gateway-인증-내장-미래)
4. [아키텍처 4: K8s Gateway + Spring Cloud Gateway (WebFlux)](#4-아키텍처-4-k8s-gateway--spring-cloud-gateway-webflux)
5. [Gateway API의 최대 장점: 로컬 = EKS](#5-gateway-api의-최대-장점-로컬--eks)
6. [4개 아키텍처 비교 + 순위](#6-4개-아키텍처-비교--순위)
7. [전환 판단 기준](#7-전환-판단-기준)
8. [book.md 핵심 요약](#8-bookmd-핵심-요약)
9. [KubeKanvas로 배포 관리하기](#9-kubekanvas로-배포-관리하기)

---

## 1. 아키텍처 1: Ingress + auth-url (현재 최선)

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Client                                                     │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────┐                                               │
│  │   NLB    │  L4 TCP 통로 (EKS 자동 생성)                   │
│  └────┬─────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           NGINX Ingress Controller                    │   │
│  │                                                      │   │
│  │  ┌─ public-ingress ──────────────────────────────┐   │   │
│  │  │  auth-url 없음                                 │   │   │
│  │  │  /api/auth/login    → user-svc                │   │   │
│  │  │  /api/auth/register → user-svc                │   │   │
│  │  └────────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌─ main-ingress ────────────────────────────────┐   │   │
│  │  │  auth-url → auth-svc (서브리퀘스트 인증)       │   │   │
│  │  │  /api/orders   → order-svc                    │   │   │
│  │  │  /api/users    → user-svc                     │   │   │
│  │  │  /api/shipping → shipping-svc                 │   │   │
│  │  │  /api/delivery → delivery-svc                 │   │   │
│  │  │  /ws           → websocket-svc                │   │   │
│  │  └─────────────────────────┬──────────────────────┘   │   │
│  └────────────────────────────┼──────────────────────────┘   │
│                               │                              │
│                               ▼                              │
│                        ┌──────────────┐                      │
│                        │  auth-svc    │                      │
│                        │  JWT 검증     │                      │
│                        │  200 → 통과   │                      │
│                        │  401 → 차단   │                      │
│                        └──────────────┘                      │
│                                                             │
│  인증 코드: auth-service 1곳에만 존재                         │
│  마이크로서비스: 비즈니스 로직만 (인증 코드 0줄)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```
장점:
  ✅ 인증 로직이 auth-service 1곳에만 존재
  ✅ 마이크로서비스에 인증 코드 없음 (깔끔)
  ✅ NGINX Ingress Controller 10년+ 검증됨
  ✅ 인증 변경 시 auth-service만 재배포

단점:
  ⚠️ auth-url은 NGINX 전용 (벤더 종속)
  ⚠️ Ingress annotation 기반 (표준 아님)
  ⚠️ auth-service가 죽으면 전체 API 502
  ⚠️ 모든 요청에 서브리퀘스트 1회 추가 (latency)

필요 리소스:
  Deployment 6개 (auth + order + user + shipping + delivery + ws)
  Service 6개 | Ingress 2개 | Secret 3개
```

---

## 2. 아키텍처 2: Gateway API + 각 서비스 인증 (과도기)

> Gateway API는 GA지만 인증 표준이 없다.
> 그래서 각 마이크로서비스에서 직접 JWT를 검증한다.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Client                                                     │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────┐                                               │
│  │   NLB    │  L4 TCP 통로                                   │
│  └────┬─────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Gateway (port 80, 443)                               │   │
│  │  GatewayClass: nginx (NGINX Gateway Fabric)           │   │
│  └──────────────────────────┬────────────────────────────┘   │
│                             │                                │
│       ┌─────────────────────┼──────────────────────┐         │
│       │                     │                      │         │
│       ▼                     ▼                      ▼         │
│  ┌──────────┐    ┌────────────────┐    ┌──────────────┐     │
│  │HTTPRoute │    │  HTTPRoute     │    │  HTTPRoute   │     │
│  │auth-route│    │  order-route   │    │  ship-route  │     │
│  │/api/auth │    │  /api/orders   │    │  /api/ship   │     │
│  └────┬─────┘    └───────┬────────┘    └──────┬───────┘     │
│       │                  │                     │             │
│       ▼                  ▼                     ▼             │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │user-svc  │    │ order-svc    │    │shipping-svc  │       │
│  │로그인     │    │ [JWT검증]    │    │ [JWT검증]     │       │
│  │회원가입   │    │ [비즈니스]    │    │ [비즈니스]    │       │
│  └──────────┘    └──────────────┘    └──────────────┘       │
│                                                             │
│  ⚠️ 모든 서비스에 JWT 검증 코드가 들어감!                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```
장점:
  ✅ Gateway API 표준 사용 (벤더 종속 탈피)
  ✅ 역할 분리 (GatewayClass/Gateway/HTTPRoute)
  ✅ auth-service 별도 Pod 불필요 (5개로 줄어듦)
  ✅ 서비스 하나 죽어도 다른 서비스 인증에 영향 없음
  ✅ HTTPRoute를 서비스별로 분리 관리 가능

단점:
  ⚠️ JWT 검증 코드가 모든 서비스에 중복
  ⚠️ 인증 방식 변경 시 전 서비스 수정 + 재배포
  ⚠️ 서비스마다 jwt-secret을 주입해야 함
  ⚠️ NGINX Gateway Fabric은 아직 Ingress Controller보다 덜 성숙

필요 리소스:
  Deployment 5개 (order + user + shipping + delivery + ws)
  Service 5개 | Gateway 1개 | HTTPRoute 5~7개 | Secret 3개

  ※ auth-service 없어짐 → Deployment/Service 각 1개 감소
  ※ Ingress 2개 → Gateway 1개 + HTTPRoute 여러 개
```

```
JWT 검증 중복 문제를 줄이는 방법:

  Spring Boot 기준:
    공통 JWT 라이브러리를 만들어서 의존성으로 추가

    msa-common (라이브러리)
      └─ JwtFilter.java        ← JWT 검증 필터
      └─ UserContext.java      ← userId, role 보관

    order-service → msa-common 의존
    user-service  → msa-common 의존
    shipping-svc  → msa-common 의존
    ...

    코드 중복은 없어지지만,
    라이브러리 버전 관리 + 전체 재배포는 여전히 필요
```

---

## 3. 아키텍처 3: Gateway 인증 내장 (미래)

> Gateway API 표준에 JWT 인증이 포함되면 가능해지는 구조.
> 현재는 Istio 등 일부 구현체에서만 가능.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Client                                                     │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────┐                                               │
│  │   NLB    │  L4 트래픽 분산                                 │
│  └────┬─────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Gateway (HTTPS :443 + TLS 종료)                      │   │
│  │                                                      │   │
│  │  ┌─ 인증 처리 (Gateway 레벨) ──────────────────────┐  │   │
│  │  │                                                 │  │   │
│  │  │  JWT 검증 내장 (표준 스펙)                        │  │   │
│  │  │  ① Authorization 헤더에서 토큰 추출               │  │   │
│  │  │  ② 서명 검증 + 만료 확인                          │  │   │
│  │  │  ③ claims → X-User-Id, X-User-Role 헤더 주입     │  │   │
│  │  │  ④ 유효하지 않으면 401 즉시 반환                   │  │   │
│  │  │                                                 │  │   │
│  │  │  Public 경로 (/api/auth/*) → 인증 Skip           │  │   │
│  │  │  Private 경로 (나머지)     → 인증 필수             │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └──────────────────────────┬────────────────────────────┘   │
│                             │                                │
│       ┌─────────────────────┼──────────────────────┐         │
│       ▼                     ▼                      ▼         │
│  ┌──────────┐    ┌────────────────┐    ┌──────────────┐     │
│  │HTTPRoute │    │  HTTPRoute     │    │  HTTPRoute   │     │
│  │/api/auth │    │  /api/orders   │    │  /api/ship   │     │
│  └────┬─────┘    └───────┬────────┘    └──────┬───────┘     │
│       │                  │                     │             │
│       ▼                  ▼                     ▼             │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │user-svc  │    │ order-svc    │    │shipping-svc  │       │
│  │로그인     │    │ [비즈니스만]  │    │ [비즈니스만]  │       │
│  │회원가입   │    │ 인증코드 0줄  │    │ 인증코드 0줄  │       │
│  └──────────┘    └──────────────┘    └──────────────┘       │
│                                                             │
│  🎯 아키텍처 1의 장점 (중앙 인증) +                            │
│     아키텍처 2의 장점 (표준, 역할 분리) = 최종 형태            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```
장점:
  ✅ 인증이 Gateway 레벨에서 표준으로 처리
  ✅ 마이크로서비스에 인증 코드 없음
  ✅ auth-service Pod 불필요
  ✅ 벤더 종속 없음 (표준 스펙)
  ✅ 역할 분리 (인프라/운영/개발)
  ✅ auth-service 장애 위험 없음 (Gateway 자체가 처리)

단점 / 현실:
  ❌ 아직 Gateway API 표준에 JWT 인증 스펙 없음
  ❌ Istio에서는 가능하지만 Istio 자체가 무거움 (~2GB 추가)
  ❌ NGINX Gateway Fabric은 아직 미지원
  ❌ 언제 표준화될지 불확실

필요 리소스 (미래):
  Deployment 5개 (auth-service 없음!)
  Service 5개 | Gateway 1개 | HTTPRoute 5~7개 | Secret 3개
```

---

## 4. 아키텍처 4: K8s Gateway + Spring Cloud Gateway (WebFlux)

> SCG(Spring Cloud Gateway)는 WebFlux(리액티브) 기반이 정석.
> K8s Gateway API로 HTTPS 종료 후, SCG가 JWT 인증 + 라우팅을 모두 처리.

### 4-1. 핵심 구조

```
⚠️ HTTPRoute가 없으면 K8s Gateway → SCG 연결이 안 됨!
   HTTPRoute 최소 1개 필요 (모든 트래픽을 SCG로 보내는 용도)

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Client                                                     │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────┐                                               │
│  │   NLB    │  L4 트래픽 분산                                 │
│  └────┬─────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  K8s Gateway (HTTPS :443, TLS 종료)                   │   │
│  └──────────────────────────┬────────────────────────────┘   │
│                             │                                │
│                             ▼                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  HTTPRoute (단 1개)                                   │   │
│  │  /* (모든 요청) → scg-service:80                       │   │
│  │                                                      │   │
│  │  ※ HTTPRoute는 "통로" 역할만                           │   │
│  │  ※ 실제 경로 분기는 SCG가 함                            │   │
│  └──────────────────────────┬────────────────────────────┘   │
│                             │                                │
│                             ▼                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Spring Cloud Gateway (WebFlux + Netty)               │   │
│  │  ═══════════════════════════════════════              │   │
│  │                                                      │   │
│  │  [Pre-Filter 체인]                                    │   │
│  │    ① JwtAuthFilter: JWT 검증 + X-User-Id 헤더 주입    │   │
│  │    ② (선택) RateLimitFilter: 요청 제한                 │   │
│  │    ③ (선택) LoggingFilter: 요청 로깅                   │   │
│  │                                                      │   │
│  │  [라우팅 — Java DSL 또는 application.yml]              │   │
│  │    /api/auth/**    → user-svc:8082   (JWT 필터 Skip)  │   │
│  │    /api/orders/**  → order-svc:8081  (JWT 필터 적용)   │   │
│  │    /api/users/**   → user-svc:8082   (JWT 필터 적용)   │   │
│  │    /api/shipping/**→ shipping-svc:8083                │   │
│  │    /api/delivery/**→ delivery-svc:8084                │   │
│  │    /ws/**          → websocket-svc:8085               │   │
│  │                                                      │   │
│  └───────┬───────────┬───────────┬───────────┬──────────┘   │
│          │           │           │           │              │
│          ▼           ▼           ▼           ▼              │
│     ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │
│     │order   │ │user    │ │ship    │ │deliv   │           │
│     │-svc    │ │-svc    │ │-svc    │ │-svc    │           │
│     │비즈니스 │ │비즈니스 │ │비즈니스 │ │비즈니스 │           │
│     │로직만!  │ │로직만!  │ │로직만!  │ │로직만!  │           │
│     └────────┘ └────────┘ └────────┘ └────────┘           │
│                                                             │
│  인증 코드: SCG 1곳에만 존재                                  │
│  마이크로서비스: 비즈니스 로직만 (인증 코드 0줄)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4-2. 왜 WebFlux인가?

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  SCG는 내부적으로 WebFlux + Project Reactor + Netty 기반     │
│  = 논블로킹(Non-blocking) I/O                               │
│                                                             │
│  API Gateway가 하는 일:                                      │
│    요청 받음 → JWT 검증 → 다른 서비스로 전달 → 응답 반환      │
│    = I/O 작업의 연속 (CPU 연산 거의 없음)                     │
│                                                             │
│  전통적 Spring MVC (블로킹):                                  │
│    요청 1개 = 스레드 1개 점유                                 │
│    동시 요청 200개 → 스레드 200개 필요                        │
│    스레드 풀 고갈 → 요청 대기                                 │
│                                                             │
│  WebFlux (논블로킹):                                         │
│    요청 1개가 스레드를 점유하지 않음                           │
│    소수의 이벤트루프 스레드로 수천 요청 동시 처리              │
│    I/O 대기 중에도 스레드가 다른 요청 처리                     │
│                                                             │
│  → API Gateway는 I/O 중심 작업이므로                         │
│    WebFlux가 최적의 선택 (정석)                               │
│                                                             │
│  ⚠️ 마이크로서비스(order, user 등)까지 WebFlux일 필요는 없음!  │
│    마이크로서비스는 Spring MVC(블로킹)로 구현해도 됨           │
│    SCG만 WebFlux면 충분                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4-3. SCG의 인증 처리 흐름

```
[Public 경로 — JWT 필터 Skip]

  POST /api/auth/login
    │
    ▼
  K8s Gateway (TLS 종료)
    │
    ▼
  HTTPRoute → SCG
    │
    SCG 라우팅: /api/auth/** → JWT 필터 Skip!
    │
    ▼
  user-svc → 로그인 처리 → JWT 토큰 발급


[Private 경로 — JWT 필터 적용]

  GET /api/orders
  Authorization: Bearer eyJhb...
    │
    ▼
  K8s Gateway (TLS 종료)
    │
    ▼
  HTTPRoute → SCG
    │
    ├─ JwtAuthFilter 실행
    │    ├─ Authorization 헤더에서 토큰 추출
    │    ├─ JWT 서명 검증 + 만료 확인
    │    ├─ claims에서 userId, role 추출
    │    ├─ X-User-Id: 42 헤더 추가
    │    └─ X-User-Role: USER 헤더 추가
    │
    ├─ 라우팅: /api/orders → order-svc:8081
    │
    ▼
  order-svc
    @RequestHeader("X-User-Id") ← SCG가 넣어준 헤더
    비즈니스 로직만 수행


[토큰 없이 Private 접근 — 차단]

  GET /api/orders (토큰 없음)
    │
    ▼
  SCG → JwtAuthFilter → 토큰 없음! → 401 즉시 반환
    (order-svc까지 도달하지 않음)
```

### 4-4. SCG의 단점

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ❌ 단점 1: WebFlux 러닝커브                                  │
│                                                             │
│    Mono<Void>, Flux, ServerWebExchange...                   │
│    기존 Spring MVC 개발자에게는 완전히 다른 프로그래밍 모델    │
│                                                             │
│    필터 하나 짤 때도:                                         │
│      MVC:     doFilter(request, response, chain)            │
│      WebFlux: filter(exchange, chain).then(Mono.fromRunnable│
│               (() -> ...))                                  │
│                                                             │
│    디버깅도 어려움:                                           │
│      블로킹: 스택트레이스가 명확                              │
│      리액티브: 스택트레이스가 읽기 어려움 (비동기 체인)         │
│                                                             │
│  ❌ 단점 2: SPOF (Single Point of Failure)                   │
│                                                             │
│    모든 트래픽이 SCG Pod를 거침                               │
│    SCG가 죽으면 → 전체 서비스 접근 불가 (502)                  │
│    → replicas 2 이상 필수                                    │
│    → 아키텍처 1의 auth-service SPOF와 동일한 문제             │
│                                                             │
│  ❌ 단점 3: JVM 메모리 오버헤드                                │
│                                                             │
│    SCG Pod 1개당 512MB ~ 1GB 메모리 필요                     │
│    replicas 2개면 1~2GB                                     │
│    vs NGINX auth-service: ~256MB × 2 = 512MB               │
│    → 2~4배 더 많은 메모리 소모                                │
│                                                             │
│  ❌ 단점 4: Spring Cloud 버전 종속                            │
│                                                             │
│    Spring Boot 버전 ↔ Spring Cloud 버전 호환 매트릭스        │
│    Boot 3.2 → Cloud 2023.0.x                               │
│    Boot 올리면 Cloud도 따라 올려야 함                         │
│    마이크로서비스 Boot 버전과 맞춰야 할 수도 있음              │
│                                                             │
│  ❌ 단점 5: K8s 라우팅과 SCG 라우팅 이중 관리                  │
│                                                             │
│    K8s 레이어: Gateway + HTTPRoute (/* → SCG)               │
│    Java 레이어: SCG route 설정 (/api/orders → order-svc)    │
│    → 라우팅 설정이 두 곳에 분산됨                              │
│    → HTTPRoute는 사실상 빈 껍데기 (SCG 통로)                  │
│    → K8s native하지 않음                                     │
│                                                             │
│  ❌ 단점 6: 빌드/배포 사이클                                   │
│                                                             │
│    NGINX auth-url: Ingress YAML 수정 → kubectl apply (초 단위)
│    SCG 라우팅 변경: Java 코드 수정 → 빌드 → 이미지 푸시 →    │
│                     kubectl rollout (분 단위)                │
│    → 경로 하나 추가하는데 재빌드 + 재배포 필요                 │
│                                                             │
│  ⚠️ 단점 7: 마이크로서비스와 패러다임 불일치                    │
│                                                             │
│    SCG: WebFlux (리액티브, 논블로킹)                          │
│    MSA: Spring MVC (블로킹) — 대부분 이렇게 구현              │
│    → 동작에는 문제없지만, 두 가지 패러다임을 팀이 이해해야 함   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4-5. 장점

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ✅ 장점 1: 인증 로직의 자유도가 가장 높음                     │
│                                                             │
│    auth-url: JWT 검증 → 200/401 반환 (단순)                  │
│    SCG:      JWT 검증 + DB 조회 + Redis 블랙리스트 +          │
│              IP 차단 + 사용자별 Rate Limit + ...              │
│    → Java로 짜니까 뭐든 가능                                  │
│                                                             │
│  ✅ 장점 2: 중앙 집중 인증 (auth-url과 동일)                   │
│                                                             │
│    SCG에서 인증 처리 → 마이크로서비스에 인증 코드 0줄          │
│    X-User-Id, X-User-Role 헤더로 전달                        │
│    → 아키텍처 1과 결과가 같음                                 │
│                                                             │
│  ✅ 장점 3: Gateway 부가기능 내장                              │
│                                                             │
│    Circuit Breaker (Resilience4j 연동)                       │
│    Rate Limiting (Redis 연동)                                │
│    Retry                                                    │
│    Request/Response 변환                                     │
│    → NGINX annotation으로는 한계가 있는 것들                  │
│                                                             │
│  ✅ 장점 4: Spring 생태계 통합                                 │
│                                                             │
│    Spring Security 연동                                      │
│    Spring Boot Actuator (헬스체크, 메트릭)                    │
│    Spring Config Server 연동                                 │
│    → Spring 기반 팀에게 익숙한 도구들                          │
│                                                             │
│  ✅ 장점 5: Public/Private 경로 분리가 자연스러움              │
│                                                             │
│    Ingress: annotation이 Ingress 단위 → 2개 분리 필요        │
│    SCG: 필터에서 path별로 Skip/적용 분기 가능                  │
│    → Ingress/HTTPRoute 분리 없이 코드에서 해결                │
│                                                             │
│  ✅ 장점 6: 높은 동시 처리 성능                                │
│                                                             │
│    WebFlux + Netty = 논블로킹                                │
│    적은 스레드로 높은 동시 접속 처리                           │
│    API Gateway에 최적화된 I/O 모델                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4-6. 필요 리소스

```
  Deployment 6개 (scg + order + user + shipping + delivery + ws)
  Service 6개
  K8s Gateway 1개 | HTTPRoute 1개 (/* → scg-service)
  Secret 3개

  ※ auth-service 대신 scg-service (역할이 더 큼)
  ※ HTTPRoute는 1개면 충분 (SCG가 내부 라우팅)
  ※ Ingress 불필요 (K8s Gateway API 사용)
```

---

## 5. Gateway API의 최대 장점: 로컬 = EKS

> 로컬에서 Gateway + HTTPRoute를 다 세팅해두면,
> AWS 배포 시 NLB는 트래픽 분산만 하면 된다. 추가 설정 없음!

### 4-1. 왜 그런가?

```
핵심: Gateway가 "모든 설정을 품고 있다"

  ┌─────────────────────────────────────────────────────────────┐
  │  Gateway 리소스 안에 이미 다 들어있음:                         │
  │                                                             │
  │  ✅ HTTPS 설정 (port 443, TLS 인증서 참조)                    │
  │  ✅ HTTP → HTTPS 리다이렉트                                   │
  │  ✅ 리스너 포트 설정 (80, 443)                                 │
  │                                                             │
  │  HTTPRoute 안에 이미 다 들어있음:                              │
  │                                                             │
  │  ✅ 경로 라우팅 (/api/orders → order-svc)                     │
  │  ✅ 헤더 기반 라우팅                                          │
  │  ✅ 가중치 분배 (카나리 배포)                                   │
  │                                                             │
  │  → NLB가 해야 할 일이 없다!                                    │
  │  → NLB는 TCP 패킷을 Gateway Pod으로 전달만 하면 끝            │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 4-2. Ingress 방식과 비교

```
[Ingress 방식 — 로컬 → EKS 전환 시]

  로컬에서 설정:
    Ingress YAML: 라우팅 규칙 + annotation(auth-url, ssl 등)
    minikube tunnel로 접근

  EKS 올릴 때 추가 작업:
    ✅ Helm으로 Ingress Controller 설치 (별도 작업)
    ✅ NLB 자동 생성 (Controller 설치 시 따라옴)
    ⚠️ SSL annotation 수정 (ssl-redirect: "true")
    ⚠️ Ingress Controller 버전/설정 맞추기

  → Controller 설치가 별도 단계
  → annotation이 Controller 구현체에 종속적
  → 로컬 Ingress ≠ EKS Ingress (미묘한 차이 가능)


[Gateway API 방식 — 로컬 → EKS 전환 시]

  로컬에서 설정:
    GatewayClass: 구현체 지정
    Gateway:      HTTPS(443) + TLS 설정
    HTTPRoute:    경로 라우팅 규칙

  EKS 올릴 때:
    ✅ 같은 YAML 그대로 apply
    ✅ NLB 자동 생성 (Gateway 리소스 적용 시 따라옴)
    ✅ 끝!

  → Gateway YAML 자체가 "HTTPS + 라우팅" 전부를 선언
  → NLB는 L4 TCP 통로 역할만 (설정할 것 없음)
  → 로컬 Gateway YAML = EKS Gateway YAML (동일!)
```

### 4-3. 도식으로 보면

```
[로컬 (minikube / kind)]

  curl https://localhost/api/orders
    │
    ▼
  ┌──────────────────────────────────────┐
  │  Gateway (port 443, TLS 종료)        │ ← K8s 리소스
  │  HTTPRoute (/api/orders → order-svc) │ ← K8s 리소스
  └──────────────┬───────────────────────┘
                 │
                 ▼
            order-svc → order Pod


[EKS (AWS) — 같은 YAML 그대로!]

  https://api.myservice.com/api/orders
    │
    ▼
  ┌──────────┐
  │   NLB    │  TCP 트래픽 분산만 (L4)
  │          │  HTTPS? 모름. 경로? 모름. 인증? 모름.
  │          │  그냥 TCP 패킷을 Gateway Pod으로 전달.
  └────┬─────┘
       │
       ▼
  ┌──────────────────────────────────────┐
  │  Gateway (port 443, TLS 종료)        │ ← 로컬과 동일!
  │  HTTPRoute (/api/orders → order-svc) │ ← 로컬과 동일!
  └──────────────┬───────────────────────┘
                 │
                 ▼
            order-svc → order Pod

  NLB 추가 설정: 없음!
  YAML 수정: 없음! (TLS 인증서 Secret만 프로덕션용으로 교체)
```

### 4-4. Ingress vs Gateway — 로컬→EKS 전환 비교

```
┌───────────────────────┬─────────────────────┬────────────────────┐
│                       │  Ingress 방식        │  Gateway API 방식   │
├───────────────────────┼─────────────────────┼────────────────────┤
│ Controller 설치       │  Helm 별도 설치 필요  │  GatewayClass 적용  │
│ NLB 생성              │  Controller 설치 시   │  Gateway 적용 시    │
│ HTTPS 설정 위치       │  annotation (흩어짐)  │  Gateway 안 (명확)  │
│ 라우팅 설정 위치       │  Ingress YAML        │  HTTPRoute          │
│ EKS 올릴 때 수정사항  │  SSL annotation 변경  │  TLS Secret만 교체  │
│                       │  + Controller 설정    │                    │
│ NLB 추가 설정         │  없음 (L4 통로)       │  없음 (L4 통로)     │
│ YAML 이식성           │  annotation 종속적    │  표준 (동일 YAML)   │
│ "로컬 = 프로덕션"     │  거의 같음 (90%)      │  완전히 같음 (99%)  │
└───────────────────────┴─────────────────────┴────────────────────┘
```

### 4-5. 핵심 정리

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Gateway API의 가장 실용적인 장점:                            │
│                                                             │
│  "로컬에서 다 세팅하면 EKS에서 건드릴 게 없다"                │
│                                                             │
│  Gateway:   HTTPS(443) + TLS  → 로컬/EKS 동일               │
│  HTTPRoute: 경로 라우팅        → 로컬/EKS 동일               │
│  NLB:       TCP 분산만         → 설정할 것 없음               │
│                                                             │
│  유일한 차이:                                                │
│    TLS Secret (로컬: self-signed → EKS: ACM/실제 인증서)     │
│    Secret 값 (DB/Kafka 접속 주소)                            │
│                                                             │
│  이것은 Ingress 방식에서도 바꿔야 하는 것이므로               │
│  Gateway API 고유의 추가 작업은 사실상 0개!                   │
│                                                             │
│  ∴ Ingress: "EKS 올릴 때 Controller 설치 + annotation 확인" │
│    Gateway: "같은 YAML 그대로 apply"                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 4개 아키텍처 비교 + 순위

### 6-1. 전체 비교표

```
┌──────────────────┬────────────────┬────────────────┬────────────────┬─────────────────┐
│                  │ 1. Ingress     │ 2. Gateway API │ 3. Gateway     │ 4. K8s Gateway  │
│                  │ + auth-url     │ + 서비스별 인증  │ 인증 내장(미래) │ + SCG(WebFlux)  │
├──────────────────┼────────────────┼────────────────┼────────────────┼─────────────────┤
│ 상태             │ ✅ 사용 가능    │ ✅ 사용 가능    │ ⏳ 미래        │ ✅ 사용 가능     │
│ 인증 위치        │ auth-service   │ 각 서비스 내부  │ Gateway 자체   │ SCG (Java)      │
│ 인증 코드 중복   │ 1곳            │ 모든 서비스     │ 0곳            │ 1곳 (SCG)       │
│ 별도 인증 Pod    │ auth-svc       │ 불필요         │ 불필요         │ SCG Pod         │
│ Deployment 수    │ 6개            │ 5개            │ 5개            │ 6개             │
│ 인증 자유도      │ 중간 (HTTP)    │ 높음 (Java)    │ 낮음 (설정만)   │ 최고 (Java)     │
│ 벤더 종속        │ NGINX 종속     │ 표준           │ 표준           │ Spring 종속     │
│ 인증 변경 시     │ 1곳 수정       │ 전체 서비스     │ Gateway 설정   │ SCG만 재빌드    │
│ 장애 영향        │ auth 죽으면    │ 각자 독립      │ Gateway 의존   │ SCG 죽으면      │
│                  │ 전체 502       │                │                │ 전체 502        │
│ 부가기능         │ annotation만   │ 없음           │ 표준 Policy    │ CB/Rate/Retry   │
│ 메모리 오버헤드  │ 낮음 (~256MB)  │ 없음           │ 없음           │ 높음 (~1GB)     │
│ 성숙도           │ 매우 높음      │ 중간           │ 낮음 (미완성)   │ 높음            │
│ 학습 난이도      │ 낮음           │ 중간           │ 중간           │ 높음 (WebFlux)  │
│ 라우팅 변경 시   │ YAML 수정      │ YAML 수정      │ YAML 수정      │ 재빌드+재배포   │
│ 로컬→EKS 이식   │ 90%            │ 99%            │ 99%            │ 95%             │
│ NLB 추가 설정    │ 없음           │ 없음           │ 없음           │ 없음            │
└──────────────────┴────────────────┴────────────────┴────────────────┴─────────────────┘
```

### 6-2. 종합 순위

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  채점 기준 (10점 만점):                                          │
│    안정성, 단순함, 인증 중앙화, 벤더 독립, 확장성,                │
│    학습비용, 운영비용, 배포 편의성, 인증 자유도, 미래 대비        │
│                                                                 │
├────────────┬──────┬──────────────────────────────────────────────┤
│            │ 점수 │ 한 줄 평가                                    │
├────────────┼──────┼──────────────────────────────────────────────┤
│            │      │                                              │
│  🥇 1위    │      │                                              │
│  아키텍처1 │ 82   │ "지금 당장 가장 안전한 선택"                   │
│  Ingress   │      │  단순하고 검증됨. 인증 중앙화.                │
│  +auth-url │      │  벤더 종속이 유일한 약점.                     │
│            │      │                                              │
├────────────┼──────┼──────────────────────────────────────────────┤
│            │      │                                              │
│  🥈 2위    │      │                                              │
│  아키텍처4 │ 74   │ "기능은 최고, 대가도 최고"                     │
│  K8s GW    │      │  인증 자유도/부가기능 1등.                    │
│  + SCG     │      │  WebFlux 러닝커브 + JVM 메모리 + 재빌드 필요. │
│            │      │  Spring 팀에게는 강력한 선택.                  │
│            │      │                                              │
├────────────┼──────┼──────────────────────────────────────────────┤
│            │      │                                              │
│  🥉 3위    │      │                                              │
│  아키텍처2 │ 68   │ "표준은 좋지만, 인증 중복이 아픔"              │
│  GW API    │      │  벤더 독립 + YAML 이식성 최고.                │
│  +서비스별 │      │  JWT 코드 전 서비스 중복이 큰 단점.           │
│            │      │                                              │
├────────────┼──────┼──────────────────────────────────────────────┤
│            │      │                                              │
│  4위       │      │                                              │
│  아키텍처3 │ 90   │ "완성되면 1등, 하지만 아직 미완성"             │
│  GW 인증   │  ⏳  │  이론상 최고 (중앙인증 + 표준 + 추가Pod 없음). │
│  내장      │      │  현실: 표준 미확정. 사용 불가.                │
│            │      │  ※ 사용 가능해지면 잠재 점수 90점              │
│            │      │                                              │
└────────────┴──────┴──────────────────────────────────────────────┘
```

### 6-3. 상황별 추천

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  "빠르게 프로덕션 올려야 해" → 아키텍처 1 (Ingress+auth-url)│
│    이유: 가장 단순, 가장 검증됨, 러닝커브 최소               │
│                                                             │
│  "Spring 팀이고, 인증 로직이 복잡해" → 아키텍처 4 (SCG)      │
│    이유: JWT + DB블랙리스트 + Rate Limit 등                  │
│         Java로 원하는 대로 구현 가능                          │
│         Circuit Breaker, Retry도 내장                        │
│    조건: 팀이 WebFlux를 다룰 수 있거나 배울 의지가 있을 때    │
│                                                             │
│  "벤더 종속 싫고, K8s 표준으로 가고 싶어"                     │
│    → 아키텍처 2 (Gateway API + 서비스별 인증)                 │
│    이유: 표준 기반, YAML 이식성 99%                           │
│    단점: JWT 코드 중복 (공통 라이브러리로 완화)               │
│                                                             │
│  "1~2년 후" → 아키텍처 3 대기                                │
│    조건: Gateway API JWT 인증 표준 확정 + 구현체 지원         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6-4. 아키텍처 1 vs 4 심층 비교 (현실적으로 고민되는 조합)

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  둘 다 "인증 중앙화 + 마이크로서비스 인증 코드 0줄"          │
│  구조적으로 가장 비슷한 두 아키텍처.                          │
│  어떤 걸 고를까?                                             │
│                                                             │
├─────────────────────┬─────────────────┬─────────────────────┤
│                     │ 1. auth-url     │ 4. SCG (WebFlux)    │
├─────────────────────┼─────────────────┼─────────────────────┤
│ 인증 처리자         │ auth-service    │ SCG                 │
│                     │ (Spring Boot)   │ (Spring WebFlux)    │
│ 인증 트리거         │ NGINX 서브리퀘스트│ SCG Pre-Filter     │
│ 라우팅 주체         │ NGINX (Ingress) │ SCG (Java)          │
│ 경로 추가 시        │ YAML 한 줄 추가  │ Java 코드 + 재빌드  │
│ 인증 로직 변경 시   │ auth-svc 재배포  │ SCG 재빌드+재배포   │
│ Public/Private 분리 │ Ingress 2개     │ 필터에서 path 분기   │
│ 메모리 (인증 Pod)   │ ~256MB × 2      │ ~512MB~1GB × 2      │
│ Circuit Breaker     │ 별도 구현 필요   │ Resilience4j 내장   │
│ Rate Limiting       │ NGINX 기본만     │ Redis 연동 가능     │
│ 배포 속도           │ 빠름 (YAML)     │ 느림 (빌드 필요)     │
│ 디버깅 난이도       │ 쉬움            │ 어려움 (리액티브)     │
│ 기술 스택 복잡도    │ 낮음            │ 높음                 │
├─────────────────────┼─────────────────┼─────────────────────┤
│ 결론                │ 단순한 JWT 인증  │ 복잡한 인증 로직     │
│                     │ 이것으로 충분    │ 이쪽이 더 유리       │
└─────────────────────┴─────────────────┴─────────────────────┘

  우리 프로젝트 (JWT 검증만 필요):
    → 아키텍처 1이 더 적합 (단순하니까)

  만약 나중에 필요해지면:
    토큰 블랙리스트 (로그아웃 처리) → SCG 고려
    사용자별 API 호출 제한           → SCG 고려
    서비스 장애 전파 방지            → SCG 고려
    → 그때 아키텍처 4로 전환해도 늦지 않음
```

---

## 7. 전환 판단 기준

```
아키텍처 1 → 4 전환 체크 (auth-url → SCG):
  □ 단순 JWT 검증으로 부족해졌는가? (블랙리스트, Rate Limit 등)
  □ 팀이 WebFlux/리액티브 프로그래밍을 할 수 있는가?
  □ JVM 메모리 추가 (~1GB)를 감당할 수 있는가?
  □ 라우팅 변경마다 재빌드+재배포가 괜찮은가?

아키텍처 1 → 2 전환 체크 (auth-url → Gateway API):
  □ NGINX Gateway Fabric이 안정화되었는가?
  □ 팀에서 Gateway API 리소스(GatewayClass/Gateway/HTTPRoute)에 익숙한가?
  □ 공통 JWT 라이브러리를 만들 여력이 있는가?
  □ 벤더 종속 탈피가 우선인가?

아키텍처 2 → 3 전환 체크 (서비스별 인증 → Gateway 내장):
  □ Gateway API 표준에 인증(JWT/ext-auth) 스펙이 포함되었는가?
  □ 사용 중인 구현체(NGINX/Envoy/etc)가 해당 스펙을 지원하는가?
  □ Public/Private 경로 분리가 표준으로 가능한가?
  □ 프로덕션 적용 사례가 충분한가?
```

---

## 8. book.md 핵심 요약

> book.md의 12개 섹션을 한 페이지로 압축

### 전체 구조 (Section 1)

```
Client → NLB → NGINX Ingress Controller
                 ├─ public-ingress (인증 없음): login, register, refresh
                 └─ main-ingress (auth-url): orders, users, shipping, delivery, ws
                      └─→ auth-svc (JWT 검증) → 통과 시 서비스로 전달
```

### 핵심 개념 (Section 2)

```
Ingress Controller ≠ Ingress 리소스
  Controller = 프로그램 (NGINX Pod, Helm으로 설치)
  Ingress    = 설정 파일 (YAML, kubectl apply로 적용)
  Controller 없이 Ingress만 올리면 아무 일도 안 일어남

Pod → Deployment → Service
  Deployment가 Pod을 만들고 관리
  Service가 Pod에 접근할 고정 주소 제공

Service 타입
  ClusterIP:    내부 전용 (우리 MSA 전부)
  NodePort:     개발용
  LoadBalancer: NLB 자동 생성 (Ingress Controller만 사용)

auth-url은 Ingress 단위 적용
  → path별 ON/OFF 불가 → Ingress 2개로 분리
```

### 인증 흐름 (Section 3)

```
인증 필요:  Client → NGINX → auth-svc(서브리퀘스트) → 200이면 서비스로 전달
인증 불필요: Client → NGINX → 바로 서비스로 전달 (auth-url 호출 안 함)
```

### NLB vs ALB (Section 4)

```
auth-url은 NGINX 전용 기능
  → NGINX가 필수 → NGINX 앞에는 NLB(L4)가 자연스러움
  → ALB 단독은 auth-url 못 씀 → 우리 구조에서 불가
  → DNS: Route 53이면 Alias, 외부 DNS면 CNAME
```

### Gateway API (Section 5)

```
Ingress의 후계자: GatewayClass / Gateway / HTTPRoute (3개로 역할 분리)
GA 2023년, 라우팅은 표준화됨
하지만 auth-url 같은 인증은 표준 스펙에 아직 없음
→ 지금은 Ingress 유지, 인증 표준화 후 전환 검토
```

### 리소스 목록 (Section 6)

```
Ingress 2개 | Deployment 6개 | Service 6개 (ClusterIP)
Secret 3개 (jwt, db, kafka) | ConfigMap 1개
로컬: + StatefulSet 2개 (kafka, postgres)
EKS:  + NLB(자동), MSK, RDS, ECR, Route53
```

### auth-service (Section 7)

```
순수 Spring Boot (Spring Cloud 아님)
GET /auth/verify 엔드포인트 1개
→ JWT 검증 → 200 + X-User-Id/X-User-Role 또는 401
user-service가 로그인 시 JWT 발급 (같은 시크릿 키 공유)
```

### 서비스 코드 패턴 (Section 8)

```
@RequestHeader("X-User-Id")로 인증 정보 수신
JWT 검증 코드 0줄 — NGINX가 다 처리해줬으니까
```

### 로컬 vs EKS (Section 9)

```
변경점 딱 3개: Secret 값(DB/Kafka 주소), 이미지 주소(ECR), SSL 설정
나머지(Ingress 규칙, Service, Deployment 구조)는 동일
파일 구조: k8s/base/ (공통) + k8s/local/ + k8s/eks/
```

### 배포 순서 (Section 10)

```
로컬: minikube start → addon ingress → 이미지 빌드 → apply → tunnel → 테스트
EKS:  eksctl create → helm ingress-nginx → RDS/MSK → ECR push → apply → Route53 → 테스트
```

### 트러블슈팅 (Section 11)

```
public 경로인데 401 → path 겹침 확인
auth-service 죽으면 전체 502 → replicas 2 이상
WebSocket 끊김 → timeout 3600으로 설정
이미지 못 찾음 → minikube docker-env 확인
Ingress ADDRESS 비어있음 → Controller 설치 확인
```

---

## 9. KubeKanvas로 배포 관리하기

> https://www.kubekanvas.io/editor/
> 드래그앤드롭으로 K8s 리소스를 배치하고, YAML을 자동 생성/내보내기

### 8-1. KubeKanvas란?

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  KubeKanvas = Kubernetes 시각적 설계 도구 (Visual K8s IDE)   │
│                                                             │
│  YAML을 직접 안 써도 됨!                                     │
│  드래그앤드롭으로 리소스 배치 → YAML 자동 생성                 │
│                                                             │
│  핵심 기능:                                                  │
│    ① 드래그앤드롭으로 리소스 배치 (Deployment, Service 등)    │
│    ② 리소스 간 연결선으로 관계 표현                            │
│    ③ 설정값 입력 (포트, 이미지, replicas 등)                  │
│    ④ YAML 자동 생성 + 내보내기 (Export)                       │
│    ⑤ 실시간 검증 (잘못된 설정 즉시 경고)                      │
│    ⑥ 템플릿 저장/재사용                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8-2. 우리 프로젝트에서의 활용 워크플로우

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  STEP 1: 설계 (Design)                                      │
│  ═══════════════════                                        │
│                                                             │
│  KubeKanvas 에디터에서 드래그앤드롭:                          │
│                                                             │
│    Deployment 6개 배치                                       │
│      auth / order / user / shipping / delivery / websocket  │
│      → 각각 이미지, 포트, replicas, env 설정                  │
│                                                             │
│    Service 6개 배치 + Deployment에 연결선                     │
│      auth-svc / order-svc / user-svc / ...                  │
│      → ClusterIP, 포트 매핑 설정                              │
│                                                             │
│    Ingress 2개 배치 + Service에 연결선                        │
│      main-ingress (auth-url annotation 설정)                 │
│      public-ingress (annotation 없음)                        │
│                                                             │
│    Secret 3개 배치 + 연결선                                   │
│      jwt-secret → auth, user                                │
│      db-credentials → order, user, shipping, delivery       │
│      kafka-credentials → 전체                                │
│                                                             │
│  → 화면에서 전체 아키텍처가 한눈에 보임!                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  STEP 2: 검증 (Validate)                                    │
│  ═══════════════════════                                    │
│                                                             │
│  KubeKanvas가 자동으로 확인:                                  │
│    ✅ 포트 매핑이 맞는지                                      │
│    ✅ Service selector와 Deployment label이 일치하는지        │
│    ✅ Secret 참조가 올바른지                                   │
│    ✅ 필수 필드 누락 없는지                                    │
│                                                             │
│  → YAML 문법 에러를 배포 전에 잡음!                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  STEP 3: 내보내기 (Export)                                   │
│  ═════════════════════════                                  │
│                                                             │
│  Export 버튼 → YAML 파일 다운로드                             │
│                                                             │
│  다운받은 YAML을 그대로:                                      │
│    로컬: kubectl apply -f exported-yaml/                     │
│    EKS:  kubectl apply -f exported-yaml/                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8-3. 배포 변경이 편해지는 이유

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  기존 방식 (YAML 직접 편집):                                  │
│                                                             │
│  "order-service replicas를 2 → 3으로 바꿔야지"              │
│    ① 에디터에서 order-service.yaml 열기                      │
│    ② spec.replicas 찾기                                     │
│    ③ 2 → 3으로 수정                                        │
│    ④ 들여쓰기 안 틀렸는지 확인                                │
│    ⑤ kubectl apply -f order-service.yaml                    │
│                                                             │
│  "새 서비스 notification-service 추가해야지"                  │
│    ① Deployment YAML 새로 작성 (30줄)                       │
│    ② Service YAML 새로 작성 (15줄)                          │
│    ③ Ingress에 path 추가 (5줄)                              │
│    ④ 들여쓰기, label, selector 다 맞는지 확인                 │
│    ⑤ kubectl apply                                         │
│                                                             │
│  → YAML 실수 가능성 높음 (들여쓰기, 오타, selector 불일치)    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  KubeKanvas 방식 (시각적 편집):                               │
│                                                             │
│  "order-service replicas를 2 → 3으로 바꿔야지"              │
│    ① order-service 블록 클릭                                 │
│    ② replicas: 3 으로 변경                                  │
│    ③ Export → kubectl apply                                 │
│                                                             │
│  "새 서비스 notification-service 추가해야지"                  │
│    ① Deployment 드래그앤드롭 + 설정 입력                      │
│    ② Service 드래그앤드롭 + 연결선                            │
│    ③ Ingress에서 path 추가 + 연결선                          │
│    ④ Export → kubectl apply                                 │
│                                                             │
│  → YAML 문법 실수 불가능 (자동 생성)                          │
│  → 전체 구조가 시각적으로 보여서 빠뜨리는 것 없음              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8-4. 아키텍처별 KubeKanvas 활용

```
[아키텍처 1: Ingress + auth-url]

  KubeKanvas에서 배치할 것:
    Ingress 2개 (main + public)
    Deployment 6개 + Service 6개
    Secret 3개 + ConfigMap 1개
    → 연결선으로 관계 표현
    → Export → kubectl apply -f

  변경 시:
    서비스 추가 → 블록 추가 + Ingress에 path 추가 + Export
    replicas 변경 → 블록 클릭 + 값 수정 + Export
    환경변수 추가 → 블록 클릭 + env 추가 + Export


[아키텍처 2: Gateway API]

  KubeKanvas에서 배치할 것:
    Gateway 1개
    HTTPRoute 5~7개
    Deployment 5개 + Service 5개
    Secret 3개
    → 연결선으로 관계 표현
    → Export → kubectl apply -f

  변경 시:
    서비스 추가 → Deployment + Service + HTTPRoute 추가 + Export
    경로 변경 → HTTPRoute 블록 클릭 + path 수정 + Export
    → 개발자가 자기 서비스의 HTTPRoute만 관리 가능!
```

### 8-5. 실전 배포 사이클

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. 최초 설계                                                │
│     KubeKanvas에서 전체 아키텍처 드래그앤드롭 배치             │
│     → 템플릿으로 저장                                        │
│                │                                            │
│                ▼                                            │
│  2. YAML Export                                              │
│     → k8s/base/ 폴더에 저장                                  │
│                │                                            │
│                ▼                                            │
│  3. 로컬 배포                                                │
│     kubectl apply -f k8s/base/                               │
│     → 테스트 → 문제 있으면 KubeKanvas로 돌아가서 수정          │
│                │                                            │
│                ▼                                            │
│  4. EKS 배포                                                 │
│     Secret만 EKS용으로 교체 (k8s/eks/secrets.yaml)            │
│     kubectl apply -f k8s/base/ && kubectl apply -f k8s/eks/  │
│                │                                            │
│                ▼                                            │
│  5. 변경사항 발생 (새 서비스, 설정 변경 등)                     │
│     KubeKanvas에서 저장된 템플릿 열기                          │
│     → 수정 → Export → kubectl apply                          │
│     → YAML 직접 편집 없이 배포 변경 완료!                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8-6. 핵심 정리

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  KubeKanvas가 해결하는 것:                                    │
│                                                             │
│  ❌ YAML 들여쓰기 실수    → ✅ 자동 생성 (실수 불가)          │
│  ❌ label/selector 불일치 → ✅ 연결선으로 자동 매핑            │
│  ❌ 전체 구조 파악 어려움  → ✅ 시각적으로 한눈에 보임          │
│  ❌ 리소스 빠뜨림          → ✅ 누락 시 검증에서 경고          │
│  ❌ 변경 시 여러 YAML 수정 → ✅ 블록 클릭 + 값 변경 + Export  │
│                                                             │
│  특히 유용한 상황:                                            │
│    새 서비스 추가 → 드래그앤드롭 + 연결선 + Export             │
│    아키텍처 리뷰 → 팀원에게 화면 공유하며 설명                  │
│    설정 변경     → 클릭 + 수정 + Export (YAML 안 열어도 됨)   │
│    템플릿 재활용 → 다른 프로젝트에서 같은 구조 빠르게 복제     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 타임라인 요약

```
  ┌──────────────┐         ┌──────────────┐     ┌──────────────────┐
  │ 현재 🥇       │         │ 과도기        │     │ 미래              │
  │              │         │              │     │                  │
  │ 1. Ingress   │──(A)──→ │ 2. GW API    │ ──→ │ 3. GW 인증 내장  │
  │ + auth-url   │         │ + 서비스별인증 │     │ (표준 확정 후)    │
  │              │         └──────────────┘     └──────────────────┘
  │              │
  │              │         ┌──────────────┐
  │              │──(B)──→ │ 4. K8s GW    │  인증이 복잡해지면
  │              │         │ + SCG 🥈      │  WebFlux 전환
  └──────────────┘         └──────────────┘

  (A) 벤더 독립이 우선일 때
  (B) 인증 로직이 복잡해질 때 (블랙리스트, Rate Limit, CB 등)

  모든 단계에서 KubeKanvas로:
    설계(드래그앤드롭) → 검증(자동) → Export(YAML) → 배포(kubectl apply)
```
