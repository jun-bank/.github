# Spring Boot 4.0 @MockUser 커스텀 어노테이션 트러블슈팅

## 개요

Spring Boot 4.0 / Spring Security 7.0 환경에서 `@WebMvcTest`와 커스텀 `@MockUser` 어노테이션(`@WithSecurityContext` 기반)이 동작하지 않는 문제를 해결한 과정을 정리합니다.

## 문제 상황

### 증상
- `@MockUser` 어노테이션을 적용한 테스트에서 `403 Forbidden` 발생
- MockMvc 출력에서 `Handler: Type = null` → Controller에 도달하지 못함
- 총 411개 테스트 중 11개 실패

### 실패 테스트 예시
```java
@WebMvcTest(UserController.class)
@Import({TestSecurityConfig.class, UserApiMapper.class, GlobalExceptionHandler.class})
class UserControllerTest {
    
    @Test
    @MockUser(userId = "USR-12345678")
    void getMyInfo_Success() throws Exception {
        mockMvc.perform(get("/api/v1/users/me"))
            .andExpect(status().isOk());  // 실제: 403
    }
}
```

## 시도한 해결 방법들

### 1. `@AutoConfigureMockMvc` 추가 ❌
```java
@WebMvcTest(UserController.class)
@AutoConfigureMockMvc
@Import({TestSecurityConfig.class, ...})
```
**결과**: 여전히 403 발생

### 2. TestSecurityConfig 설정 확인 ❌
```java
@TestConfiguration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class TestSecurityConfig {
    @Bean
    public SecurityFilterChain testSecurityFilterChain(HttpSecurity http) {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> session.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(POST, "/api/v1/users").permitAll()
                .requestMatchers(GET, "/api/v1/users/check-email").permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }
}
```
**결과**: 설정 자체는 문제 없음

### 3. WithMockUserSecurityContextFactory 구현 확인 ❌
```java
public class WithMockUserSecurityContextFactory 
        implements WithSecurityContextFactory<MockUser> {
    
    @Override
    public SecurityContext createSecurityContext(MockUser mockUser) {
        UserPrincipal userPrincipal = new UserPrincipal(
            mockUser.userId(), 
            mockUser.role(), 
            mockUser.email()
        );
        Authentication authentication = new UsernamePasswordAuthenticationToken(
            userPrincipal, null, userPrincipal.getAuthorities());
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        return context;
    }
}
```
**결과**: Factory 구현도 정상

### 4. 테스트 클래스 위치 확인 ✅ (올바름)
```
src/test/java/com/jun_bank/user_service/global/security/
├── MockUser.java
├── WithMockUserSecurityContextFactory.java
└── TestSecurityConfig.java
```
**결과**: 테스트 코드에 올바르게 위치함

## 근본 원인 발견

### Spring Boot 4.0 Migration Guide 핵심 변경사항

> **`@WithMockUser` and `@WithUserDetails` from `spring-security-test` now requires `spring-boot-starter-security-test` to operate properly.**

Spring Boot 4.0에서 모듈화가 크게 변경되면서, Security Test도 별도 starter가 필요해졌습니다.

### 기존 의존성 (문제)
```groovy
// build.gradle
testImplementation 'org.springframework.security:spring-security-test'
```

### 수정된 의존성 (해결)
```groovy
// build.gradle
testImplementation 'org.springframework.boot:spring-boot-starter-security-test'
```

## 해결 방법

### build.gradle 수정

```groovy
dependencies {
    // ========================================
    // Security
    // ========================================
    implementation 'org.springframework.boot:spring-boot-starter-security'
    
    // Security Test - Spring Boot 4.0에서는 starter 필요!
    // 기존: testImplementation 'org.springframework.security:spring-security-test'
    testImplementation 'org.springframework.boot:spring-boot-starter-security-test'
    
    // ...
}
```

## Spring Boot 4.0 테스트 의존성 변경 요약

| 기술 | 메인 의존성 | 테스트 의존성 |
|------|------------|--------------|
| Spring Security | `spring-boot-starter-security` | `spring-boot-starter-security-test` |
| Spring Web MVC | `spring-boot-starter-webmvc` | `spring-boot-starter-webmvc-test` |
| Spring WebFlux | `spring-boot-starter-webflux` | `spring-boot-starter-webflux-test` |
| Spring Data JPA | `spring-boot-starter-data-jpa` | `spring-boot-starter-data-jpa-test` |
| Kafka | `spring-boot-starter-kafka` | `spring-boot-starter-kafka-test` |

## 추가 이슈: AccessDeniedException이 500으로 반환되는 문제

### 증상
`@MockUser`가 동작한 후에도 권한 검증 실패 시:
- 기대값: `403 Forbidden`
- 실제값: `500 Internal Server Error`

### 원인
`GlobalExceptionHandler`에서 `AccessDeniedException`과 `AuthorizationDeniedException`을 적절히 처리하지 않음.

### 해결: GlobalExceptionHandler에 핸들러 추가
```java
@ExceptionHandler(AccessDeniedException.class)
public ResponseEntity<ApiResponse<Void>> handleAccessDeniedException(
        AccessDeniedException e, HttpServletRequest request) {
    log.warn("접근 거부: {}", e.getMessage());
    return ResponseEntity.status(HttpStatus.FORBIDDEN)
        .body(ApiResponse.error(
            "SECURITY_002",
            e.getMessage(),
            request.getRequestURI(),
            HttpStatus.FORBIDDEN.value()
        ));
}

@ExceptionHandler(AuthorizationDeniedException.class)
public ResponseEntity<ApiResponse<Void>> handleAuthorizationDeniedException(
        AuthorizationDeniedException e, HttpServletRequest request) {
    log.warn("인가 거부: {}", e.getMessage());
    return ResponseEntity.status(HttpStatus.FORBIDDEN)
        .body(ApiResponse.error(
            "SECURITY_003",
            "접근 권한이 없습니다.",
            request.getRequestURI(),
            HttpStatus.FORBIDDEN.value()
        ));
}
```

## 참고 자료

- [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Security Testing Documentation](https://docs.spring.io/spring-security/reference/servlet/test/method.html)
- [Setting Up MockMvc and Spring Security](https://docs.spring.io/spring-security/reference/servlet/test/mockmvc/setup.html)

## 결론

Spring Boot 4.0으로 마이그레이션 시 테스트 의존성도 함께 업데이트해야 합니다. 특히 `@WithSecurityContext` 기반 커스텀 어노테이션은 `spring-boot-starter-security-test`가 필수입니다.