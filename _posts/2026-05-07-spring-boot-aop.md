---
layout: post
title: "Spring Boot AOP로 구현한 권한 분리 및 주문 시간 검증: 실무 코드와 함께 살펴보기"
date: 2026-05-07 12:48:55 +0000
description: "Spring Boot 기반 정산 서비스에서 AOP와 전역 인증 필터를 활용해 역할별 권한 분리와 주문 시간 검증을 구현한 사례를 소개합니다. 업무 가능 시간 정책과 코드 조건을 일치시키고, 안정적인 예외 처리와 응답 구조 설계 경험을 공유합니다."
tags:
  - "Spring Boot"
  - "JPA"
  - "MyBatis"
  - "Authentication"
  - "Authorization"
  - "AOP"
  - "REST API"
  - "Settlement Service"
  - "Order-Settlement Relation"
  - "Enum Code Management"
  - "Response Standardization"
  - "Date Handling"
  - "Transaction Management"
  - "MyBatis XML"
  - "Inner Join Optimization"
---

## 개요

Spring Boot 기반 정산 서비스에서는 전역 인증 필터와 AOP(Aspect-Oriented Programming)를 활용하여 직원, 일반 사용자, 관리자 등 역할별 권한 분리와 주문 시 업무 가능 시간 검증을 구현했습니다. 이번 글에서는 해당 기능들의 구현 원리와 실무 코드를 중심으로 자세히 살펴봅니다.

---

## 1. 전역 인증 필터와 권한 분리

### 1-1. 전역 인증 필터 구현

로그인 및 회원가입 API를 제외한 모든 요청에 대해 Authorization 헤더에 포함된 토큰의 유효성을 검사합니다. 유효하지 않으면 401 Unauthorized 응답을 반환해 비인가 접근을 막는 방식입니다.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Unauthorized\"}");
            return;
        }

        String token = authHeader.substring(7);
        try {
            Authentication auth = JwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        } catch (JwtException ex) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"Invalid Token\"}");
            return;
        }
        filterChain.doFilter(request, response);
    }
}
```

---

### 1-2. AOP 기반 역할별 권한 검사

`@RequireRoles` 커스텀 애노테이션과 AOP의 `@Before` 어드바이스를 이용해 메서드 실행 전에 권한을 점검합니다. 권한이 없으면 `AccessDeniedException`을 던져 요청을 차단합니다. 이렇게 권한 검사 로직을 한 곳에 모으기 때문에 유지보수가 용이합니다.

```java
@Aspect
@Component
public class AuthorizationAspect {

    @Before("@annotation(requireRoles)")
    public void checkRole(JoinPoint joinPoint, RequireRoles requireRoles) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || auth.getAuthorities().stream()
                .noneMatch(a -> Arrays.asList(requireRoles.value()).contains(a.getAuthority()))) {
            throw new AccessDeniedException("권한이 없습니다.");
        }
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RequireRoles {
    String[] value();
}

// 활용 예시
@RequireRoles({"STAFF", "ADMIN"})
public List<Settlement> getDailySettlementList() {
    // 서비스 로직
}
```

---

## 2. 업무 가능 시간 검증

### 2-1. 업무 가능 시간 정책 정리

주문 가능한 시간은 두 구간으로 명확히 정의했습니다. 기본 주문 허용 시간은 자정(00:00)부터 밤 23시 59분(23:59)까지이고, 추가로 다음 날 새벽 1시 30분까지 연장 주문을 허용합니다. 정책 문서와 코드가 일치하도록 관리하여 혼란을 방지했습니다.

### 2-2. AOP를 통한 주문 시간 검증

주문 처리 메서드 실행 전 현재 시간이 정책 범위 내인지 확인합니다. 범위를 벗어나면 비즈니스 예외를 발생시켜 주문을 제한합니다.

```java
@Aspect
@Component
public class OrderTimeValidationAspect {

    private static final LocalTime ORDER_START = LocalTime.of(0, 0);
    private static final LocalTime ORDER_END = LocalTime.of(23, 59);
    private static final LocalTime EXTENDED_ORDER_START = LocalTime.of(0, 0);
    private static final LocalTime EXTENDED_ORDER_END = LocalTime.of(1, 30);

    @Before("execution(* com.example.settlement.service.OrderService.placeOrder(..))")
    public void validateOrderTime(JoinPoint joinPoint) {
        LocalTime now = LocalTime.now();

        boolean inAllowedTime =
                // 기본 주문 가능 시간 범위
                (!now.isBefore(ORDER_START) && !now.isAfter(ORDER_END))
                // 익일 새벽 확장 주문 가능 시간
                || (!now.isBefore(EXTENDED_ORDER_START) && now.isBefore(EXTENDED_ORDER_END));

        if (!inAllowedTime) {
            throw new BusinessException(ErrorCode.ORDER_OUT_OF_TIME, "주문 가능 시간이 아닙니다.");
        }
    }
}
```

- 예를 들어, 01시 15분에 주문 요청 시 허용되며
- 01시 45분에는 주문 시간이 아니라 비즈니스 예외가 발생합니다.

---

## 3. 공통 API 응답 구조 및 예외 처리

### 3-1. 일관된 응답 포맷 설계

클라이언트가 응답을 쉽게 처리할 수 있도록 제네릭 공통 응답 클래스를 정의했습니다.

```java
@Data
@AllArgsConstructor
public class ApiResponse<T> {
    private boolean success;
    private String errorCode;
    private String message;
    private T data;
}
```

### 3-2. 전역 예외 처리

`@RestControllerAdvice`로 비즈니스 예외, 권한 예외, 일반 예외를 구분하여 적절한 HTTP 상태 코드와 메시지를 반환합니다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusinessException(BusinessException ex) {
        ApiResponse<Void> response = new ApiResponse<>(false, ex.getErrorCode().name(), ex.getMessage(), null);
        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiResponse<Void>> handleAccessDenied(AccessDeniedException ex) {
        ApiResponse<Void> response = new ApiResponse<>(false, "ACCESS_DENIED", ex.getMessage(), null);
        return new ResponseEntity<>(response, HttpStatus.FORBIDDEN);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleException(Exception ex) {
        ApiResponse<Void> response = new ApiResponse<>(false, "INTERNAL_ERROR", "서버 에러가 발생했습니다.", null);
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

---

## 4. 구현 경험과 개선 사항

전역 인증 필터와 AOP를 이용한 권한 분리 및 주문 시간 검증 구현이 한층 깔끔해졌음을 체감했습니다. 특히 업무 가능 시간 정책과 코드 조건이 일치하도록 관리하는 것이 서비스 안정성에 매우 중요하다는 점을 확인했습니다.

주문 시간 검증 로직은 정책 변경 시 문서와 코드가 항상 일치하는지 주기적으로 점검하는 것을 권장합니다.

---

## 5. 부가 안내

본 글은 내부 비공개 저장소 기반으로 작성된 사례로, 예제 코드를 별도 공개하지 않습니다. 관련 구현 사례에 관한 최신 개발 정보는 뉴스레터나 공식 기술 문서에서 확인하시길 바랍니다.

추가 문의나 상세 구현 관련 내용은 내부 협의 채널 등을 통해 별도로 공유받으실 수 있습니다.
