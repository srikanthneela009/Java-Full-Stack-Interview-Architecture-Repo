# Section 06: Spring Security, OAuth2, JWT & Testing Strategy (52 Deep-Dive Questions)

[← Back to Main README](../README.md)

---

## Part A: Basic Foundational Questions (Q6.1 – Q6.18)

<a id="q6-1"></a>
### Q6.1: Compare Authentication (`AuthN` - HTTP 401) vs Authorization (`AuthZ` - HTTP 403) and how Spring Security separates them.

#### 1. Formal Technical Definition & Internal Mechanics
- **Authentication (`AuthN` - "Who are you?")**: Verifies the identity of a principal using credentials (password, JWT, mTLS client certificate, WebAuthn passkey). Represented in Spring Security by `AuthenticationManager` and `AuthenticationEntryPoint` (which returns **`401 Unauthorized`** when credentials are missing or invalid).
- **Authorization (`AuthZ` - "What are you allowed to do?")**: Evaluates whether an already-authenticated principal has permission (`ROLE_ADMIN`, `SCOPE_payments:write`, or resource ownership) to perform an action. Represented by `AuthorizationManager` and `AccessDeniedHandler` (which returns **`403 Forbidden`** when an authenticated user lacks permission).

#### 2. Technical Architecture & Execution Diagram
```
HTTP Request
     │
     ▼
[1. Authentication Filter (BearerTokenAuthenticationFilter)]
     ├── Invalid/Missing Token? ──► AuthenticationEntryPoint ──► HTTP 401 Unauthorized
     │
     ▼ (Valid Token -> Populates SecurityContextHolder)
[2. AuthorizationFilter (AuthorizationManager)]
     ├── Lacks Required Role/Scope? ──► AccessDeniedHandler ──► HTTP 403 Forbidden
     │
     ▼
[@RestController Invoked]
```

#### 3. Detailed Technical Example & Production Code
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint()) // 401
            .accessDeniedHandler(new BearerTokenAccessDeniedHandler())           // 403
        )
        .build();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Bean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `securityFilterChain(...)`, execution proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **The HTTP Spec Naming Quirk**: Notice that HTTP status `401 Unauthorized` actually means **Unauthenticated**, whereas `403 Forbidden` means **Unauthorized**! Confusing 401 and 403 in REST APIs breaks frontend OAuth2 token-refresh interceptors (which specifically trigger refresh-token flows on `401`, not `403`).

---

<a id="q6-2"></a>
### Q6.2: How does the Spring Security Architecture work from the Servlet Container (`DelegatingFilterProxy` -> `FilterChainProxy` -> `SecurityFilterChain`)?

#### 1. Formal Technical Definition & Internal Mechanics
- The Servlet Container (Tomcat) only knows about standard Jakarta `Filter` instances, not Spring ApplicationContext beans.
- **`DelegatingFilterProxy`**: A standard Servlet `Filter` registered with Tomcat that delegates execution to a Spring Bean named `springSecurityFilterChain`.
- **`FilterChainProxy`**: The Spring bean behind `springSecurityFilterChain`. It inspects the incoming request URL against one or more configured **`SecurityFilterChain`** beans (picking the first chain whose `RequestMatcher` matches) and runs that chain's ordered list of 15+ security filters (`CorsFilter`, `CsrfFilter`, `BearerTokenAuthenticationFilter`, `AuthorizationFilter`, `ExceptionTranslationFilter`).

#### 2. Technical Architecture & Execution Diagram
```
[Tomcat Servlet Container FilterChain]
           │
           ▼
[DelegatingFilterProxy] (Bridges Servlet Container to Spring IoC Context)
           │
           ▼
[FilterChainProxy (Bean: "springSecurityFilterChain")]
           ├── Matches /actuator/** ──► [SecurityFilterChain #1 (mTLS / Internal Auth)]
           └── Matches /api/**      ──► [SecurityFilterChain #2 (OAuth2 JWT Resource Server)]
                                              ├── 1. CorsFilter
                                              ├── 2. BearerTokenAuthenticationFilter
                                              ├── 3. ExceptionTranslationFilter
                                              └── 4. AuthorizationFilter
```

#### 3. Detailed Technical Example & Production Code
```java
@Configuration
@EnableWebSecurity
public class MultiChainSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorSecurityChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .anyRequest().hasRole("OPS_ADMIN")
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain apiSecurityChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `MultiChainSecurityConfig` (annotated with `@Configuration`, `@EnableWebSecurity`, `@Bean`, `@Order`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `actuatorSecurityChain(...)` and `apiSecurityChain(...)`, execution proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Multiple `SecurityFilterChain` Ordering**: When defining multiple `SecurityFilterChain` beans with `@Order`, every chain except the final fallback chain **must** declare `.securityMatcher(...)`. If `@Order(1)` omits `.securityMatcher(...)`, it matches `/**` (all requests) and `@Order(2)` is never reached!

---

<a id="q6-3"></a>
### Q6.3: How do `SecurityContextHolder`, `SecurityContext`, `Authentication`, and `Principal` work, and how is context cleared after a request?

#### 1. Formal Technical Definition & Internal Mechanics
- **`SecurityContextHolder`**: Holds the `SecurityContext` for the current execution thread using `MODE_THREADLOCAL` by default.
- **`SecurityContext`**: Wraps the currently authenticated `Authentication` object.
- **`Authentication`**: Contains:
  1. `Principal`: The user identity (`UserDetails`, `Jwt`, or custom `AuthenticatedUser` record).
  2. `Credentials`: Password/Token (automatically erased to `null` after authentication via `ProviderManager.eraseCredentialsAfterAuthentication`!).
  3. `Authorities`: `Collection<? extends GrantedAuthority>` (roles/scopes).
- **Cleanup**: `SecurityContextHolderFilter` (formerly `SecurityContextPersistenceFilter`) guarantees `SecurityContextHolder.clearContext()` runs in a `finally` block when the HTTP request finishes so pooled Tomcat threads never leak user identities!

#### 2. Technical Architecture & Execution Diagram
```
SecurityContextHolder (ThreadLocal<SecurityContext>)
  └── SecurityContext
        └── Authentication (e.g., JwtAuthenticationToken, isAuthenticated = true)
              ├── Principal:   Jwt (sub="user-42", email="alice@acme.com")
              ├── Credentials: null (Cleared after AuthN)
              └── Authorities: [SCOPE_orders:read, ROLE_CUSTOMER]
```

#### 3. Detailed Technical Example & Production Code
```java
@RestController
@RequestMapping("/api/v1/profile")
public class ProfileController {

    // Injects the Principal or Jwt directly without static SecurityContextHolder calls:
    @GetMapping
    public Map<String, Object> getMyProfile(@AuthenticationPrincipal Jwt jwt) {
        return Map.of(
            "userId", jwt.getSubject(),
            "tenantId", jwt.getClaimAsString("tenant_id")
        );
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `ProfileController` (annotated with `@RestController`, `@RequestMapping`, `@GetMapping`, `@AuthenticationPrincipal`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **`@Async` ThreadLocal Loss**: Because `SecurityContextHolder` uses a `ThreadLocal`, spawning a new thread via `CompletableFuture.supplyAsync()` or `@Async` has an empty `SecurityContext`! Wrap your executor in `DelegatingSecurityContextAsyncTaskExecutor` (never use `MODE_INHERITABLETHREADLOCAL` with pooled threads, as pooled threads keep stale inherited values!).

---

<a id="q6-4"></a>
### Q6.4: Why must passwords be hashed with `BCrypt` or `Argon2` instead of `SHA-256`, and how does `DelegatingPasswordEncoder` enable zero-downtime hash upgrades?

#### 1. Formal Technical Definition & Internal Mechanics
- **Why `SHA-256` is broken for Passwords**: `SHA-256` is designed to be **ultra-fast** on GPUs/ASICs (an RTX 4090 computes **> 20 billion SHA-256 hashes per second**!). Even with a random salt, a 10-character password hashed with `SHA-256` can be brute-forced in minutes if the database is breached.
- **Adaptive Key Derivation Functions (`BCrypt`, `SCrypt`, `PBKDF2`, `Argon2id`)**:
  - Intentionally **slow and memory-hard** (`Argon2id` requires configurable RAM megabytes + CPU iterations per hash, defeating GPU parallelism) and automatically embed a **unique 128-bit cryptographic random salt** directly inside the output hash string (`$2a$12$<22-char-salt><31-char-hash>`).
- **`DelegatingPasswordEncoder` (Spring Security Default)**: Prefixes hashes with `{bcrypt}$2a$12...` or `{argon2}$argon2id...`, allowing you to verify legacy `{bcrypt}` hashes while transparently upgrading users to `{argon2}` the next time they log in!

#### 2. Technical Architecture & Execution Diagram
```
BCrypt Output String (60 chars):
  {bcrypt}$2a$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
  └──┬───┘└─┬┘└┬┘└──────────┬──────────┘└──────────────┬──────────────┘
  Encoder  Alg Cost    128-bit Salt (22c)       184-bit Hash (31c)
   Prefix      (2^12)
```

#### 3. Detailed Technical Example & Production Code
```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Supports verifying {bcrypt}, {pbkdf2}, {scrypt}, and defaults new hashes to Argon2id or BCrypt:
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Bean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `return PasswordEncoderFactories.createDelegatingPasswordEncoder();` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Timing Attack on Non-Existent Users**: If `UserDetailsService.loadUserByUsername(email)` returns `UsernameNotFoundException` in `1ms` when the email doesn't exist, vs taking `150ms` to run `BCrypt.matches()` when the email *does* exist, attackers can enumerate valid customer emails via response timing! Spring Security's `DaoAuthenticationProvider` mitigates this by hashing against a dummy password (`userNotFoundEncodedPassword`) even when the user is not found.

---

<a id="q6-5"></a>
### Q6.5: Compare Stateful Session Cookie Authentication vs Stateless JWT Bearer Token Authentication.

#### 1. Formal Technical Definition & Internal Mechanics
- **Stateful Session (`JSESSIONID` + Redis Session Store)**:
  - Server stores session data in memory/Redis and sends an opaque `JSESSIONID` cookie (`HttpOnly; Secure; SameSite=Lax`).
  - **Pros**: Instant server-side session revocation (`DEL session:xyz`), tiny cookie size.
  - **Cons**: Every API request requires a Redis lookup; requires CSRF protection for browser cookies.
- **Stateless JWT (`Authorization: Bearer <jwt>`)**:
  - Self-contained signed JSON payload verified locally using the issuer's public RSA/ECDSA key (`JWKS`).
  - **Pros**: Zero database/Redis lookup on every microservice hop; ideal for mobile apps and microservice meshes.
  - **Cons**: Cannot be revoked before `exp` without a short TTL (`5–15m`) or a Redis blocklist (`jti`).

#### 2. Technical Architecture & Execution Diagram
```
Stateful Session:
  Client ──(Cookie: JSESSIONID=xyz)──► [Service] ──(Network Hop)──► [Redis Session Store]

Stateless JWT:
  Client ──(Bearer eyJhbGci...)──────► [Service] (Verifies ECDSA/RSA Signature locally in 0.05ms!)
```

#### 3. Detailed Technical Example & Production Code
```java
// Best Practice Hybrid in Production:
// 1. Short-lived Stateless Access JWT (TTL = 5 to 15 minutes) verified in-memory across microservices.
// 2. Stateful Refresh Token (TTL = 7 to 30 days) stored in DB/Redis with Refresh Token Rotation for instant revocation!
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Never issue a stateless Access JWT with a 30-day expiration! Keep Access JWT TTL short (`5–15 minutes`) and pair it with a revocable Refresh Token.

---

<a id="q6-6"></a>
### Q6.6: Explain the 3 Parts of a JSON Web Token (`Header.Payload.Signature`) and why Base64URL encoding is NOT encryption (`JWS` vs `JWE`).

#### 1. Formal Technical Definition & Internal Mechanics
- A standard JWT (`JWS` - JSON Web Signature) consists of 3 Base64URL-encoded segments separated by dots:
  1. **Header**: `{"alg": "RS256", "typ": "JWT", "kid": "key-2025-02"}`
  2. **Payload (Claims)**: `{"iss": "https://auth.acme.com", "sub": "user-42", "aud": "order-api", "exp": 1740050900, "iat": 1740050000, "jti": "uuid", "scope": "orders:write"}`
  3. **Signature**: `RSASHA256(base64Url(header) + "." + base64Url(payload), privateKey)`
- **CRITICAL**: A signed JWT (`JWS`) provides **Integrity and Authenticity** (nobody can tamper with the claims without invalidating the signature), **NOT Confidentiality**! Anyone can run `atob()` / `base64 -d` on the Payload and read every claim in plaintext.

#### 2. Technical Architecture & Execution Diagram
```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImtleS0xIn0 . eyJzdWIiOiJ1c2VyLTQyIiwicm9sZSI6IkFETUlOIn0 . <RSA-256 Signature>
└──────────────────┬──────────────────┘   └────────────────────┬────────────────────┘   └─────────┬─────────┘
        1. Header (Base64URL)                    2. Payload (Plaintext JSON!)          3. Cryptographic Sig
```

#### 3. Detailed Technical Example & Production Code
```java
// Never place PII, credit cards, SSNs, or database passwords inside a JWS payload!
// If payload confidentiality is required, use JWE (JSON Web Encryption) or Phantom Token Pattern
// (Opaque token at the edge -> API Gateway exchanges with Redis for internal JWT).
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **The Phantom Token Pattern**: At high-security banks, external browsers/mobile apps receive an **opaque random reference token**, and the edge API Gateway swaps that opaque token for a internal **JWT** before forwarding the request to internal microservices—keeping internal claims completely hidden from the public internet.

---

<a id="q6-7"></a>
### Q6.7: Explain CSRF (Cross-Site Request Forgery), when you need CSRF protection vs when to disable it, and how `SameSite` cookies work.

#### 1. Formal Technical Definition & Internal Mechanics
- **How CSRF Works**: If a user is logged into `bank.com` via a session cookie (`Cookie: session=abc`), and visits `evil.com` which submits a hidden `<form action="https://bank.com/transfer" method="POST">`, the browser **automatically attaches `Cookie: session=abc`** to the cross-origin request!
- **Defense 1 (`Synchronizer Token / Cookie-to-Header Double Submit`)**: Server requires a random anti-CSRF token sent in a custom HTTP header (`X-XSRF-TOKEN`) that JavaScript on `evil.com` cannot read due to the Same-Origin Policy.
- **Defense 2 (`SameSite=Lax` / `SameSite=Strict` Cookie Attribute)**: Prevents the browser from sending the cookie on cross-site `POST` requests.
- **When CSRF Protection Can Be Safely Disabled**: When your API is **100% stateless** and authenticates **exclusively via the `Authorization: Bearer <token>` HTTP header** (never via cookies or `HTTP Basic`), because browsers never automatically attach `Authorization: Bearer` headers across origins!

#### 2. Technical Architecture & Execution Diagram
```
Cookie-Based Auth (Vulnerable to CSRF without Token/SameSite):
  [evil.com Form POST] ──(Browser auto-attaches Cookie: session=abc!)──► [bank.com/transfer]

Bearer Header Auth (Immune to CSRF):
  [evil.com JS fetch]  ──(Cannot read or auto-attach Authorization header)──► Blocked!
```

#### 3. Detailed Technical Example & Production Code
```java
// For Stateless Bearer Token APIs:
http.csrf(AbstractHttpConfigurer::disable);

// For SPA Cookie-Based Auth (Angular/React reading XSRF-TOKEN cookie and sending X-XSRF-TOKEN header):
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler())
);
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- In interviews, never say *"We disable CSRF because it's a REST API."* Say: *"We disable CSRF only because our API is stateless and authenticates strictly via the `Authorization: Bearer` header rather than ambient browser cookies."*

---

<a id="q6-8"></a>
### Q6.8: How does CORS (Cross-Origin Resource Sharing) work (`OPTIONS` Preflight requests), and why must CORS be integrated into Spring Security?

#### 1. Formal Technical Definition & Internal Mechanics
- Browsers enforce the **Same-Origin Policy (SOP)**: JavaScript on `https://app.acme.com` cannot read responses from `https://api.acme.com` unless `api.acme.com` returns `Access-Control-Allow-Origin: https://app.acme.com`.
- **Preflight `OPTIONS` Request**: Before sending a cross-origin request with `Authorization: Bearer ...` or `Content-Type: application/json`, the browser automatically sends an **unauthenticated `OPTIONS` preflight request** (with `Origin` and `Access-Control-Request-Headers` headers, **without** the `Authorization` header!).
- **Why `@CrossOrigin` / MVC CORS fails without `http.cors()`**: Spring Security filters run **before** `DispatcherServlet`. If `CorsFilter` is not registered in the `SecurityFilterChain`, `BearerTokenAuthenticationFilter` intercepts the unauthenticated `OPTIONS` preflight request and rejects it with **`401 Unauthorized`**!

#### 2. Technical Architecture & Execution Diagram
```
Browser ──(1. OPTIONS /api/orders [No Auth Header!])──► [Spring Security CorsFilter] ──► 200 OK + CORS Headers
Browser ──(2. POST /api/orders [Bearer JWT Header])───► [BearerTokenAuthenticationFilter] ──► Controller
```

#### 3. Detailed Technical Example & Production Code
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.acme.com")); // Never use "*" with allowCredentials=true!
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "Idempotency-Key"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L); // Cache preflight response in browser for 1 hour!

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Bean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `config.setAllowCredentials(true);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **CORS is NOT a server-side security firewall**: CORS only instructs a *legitimate web browser* whether to expose the response to frontend JavaScript. `curl`, Postman, or an attacker's Python script completely ignores CORS headers!

---

<a id="q6-9"></a>
### Q6.9: Compare Spring Security Method Security Annotations (`@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, `@PostFilter`) and `@EnableMethodSecurity`.

#### 1. Formal Technical Definition & Internal Mechanics
- In Spring Security 6 (`Spring Boot 3`), `@EnableMethodSecurity` replaces the deprecated `@EnableGlobalMethodSecurity`.
- **`@PreAuthorize("...")`**: Evaluates a SpEL expression **before** entering the method. Supports checking roles, scopes, and method arguments (`#orderId`, `#request.customerId == authentication.name`).
- **`@PostAuthorize("...")`**: Executes the method first, then evaluates a SpEL expression against the `returnObject` (`returnObject.ownerId == authentication.name`). If false, throws `AccessDeniedException` (and rolls back `@Transactional`!).
- **`@PreFilter` / `@PostFilter`**: Filters elements of a collection parameter or return list in memory.

#### 2. Technical Architecture & Execution Diagram
```
Caller ──► [AuthorizationManagerBeforeMethodInterceptor (@PreAuthorize)]
                │
                ▼
         [Target Service Method]
                │
                ▼
         [AuthorizationManagerAfterMethodInterceptor (@PostAuthorize inspects returnObject)]
```

#### 3. Detailed Technical Example & Production Code
```java
@Service
public class OrderQueryService {

    // Evaluates custom bean security checker OR role check before method execution:
    @PreAuthorize("hasRole('ADMIN') or @orderPermissionEvaluator.isOwner(#orderId, authentication.name)")
    public OrderDto getOrderById(Long orderId) {
        return orderRepository.findDtoById(orderId).orElseThrow();
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `OrderQueryService` (annotated with `@Service`, `@PreAuthorize`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `public OrderDto getOrderById(Long orderId) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never use `@PostFilter` on large database queries**: `@PostFilter("filterObject.tenantId == authentication.principal.tenantId")` loads 100,000 rows from PostgreSQL into JVM memory and filters them in a Java loop! Instead, push security predicates down into the SQL query using Spring Security Data Integration (`?#{principal.tenantId}`).

---

<a id="q6-10"></a>
### Q6.10: What is the difference between `hasRole('ADMIN')` and `hasAuthority('ROLE_ADMIN')` (and OAuth2 `SCOPE_` prefixes) in Spring Security?

#### 1. Formal Technical Definition & Internal Mechanics
- Internally, Spring Security represents all permissions as `GrantedAuthority` strings.
- **`hasAuthority("ROLE_ADMIN")`** or **`hasAuthority("SCOPE_orders:write")`**: Checks for an **exact string match** against the user's `GrantedAuthority` collection.
- **`hasRole("ADMIN")`**: Automatically prepends the default **`ROLE_` prefix** and checks for `ROLE_ADMIN`! (Passing `hasRole("ROLE_ADMIN")` throws an `IllegalArgumentException` in Spring Security 6 to warn you not to double-prefix).
- **OAuth2 JWT Scopes**: By default, `JwtGrantedAuthoritiesConverter` reads the `scope` or `scp` claim from a JWT and prefixes each scope with **`SCOPE_`** (e.g., `"scope": "orders:read"` $\rightarrow$ `hasAuthority("SCOPE_orders:read")`).

#### 2. Technical Architecture & Execution Diagram
```
JWT Claim: "scope": "orders:write"
  └── JwtGrantedAuthoritiesConverter ──► GrantedAuthority("SCOPE_orders:write")
        └── Matched via: @PreAuthorize("hasAuthority('SCOPE_orders:write')")

JWT Claim: "roles": ["ADMIN"]
  └── Custom Converter ──► GrantedAuthority("ROLE_ADMIN")
        └── Matched via: @PreAuthorize("hasRole('ADMIN')")
```

#### 3. Detailed Technical Example & Production Code
```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter scopesConverter = new JwtGrantedAuthoritiesConverter();

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> authorities = new ArrayList<>(scopesConverter.convert(jwt));
        List<String> roles = jwt.getClaimAsStringList("roles");
        if (roles != null) {
            roles.forEach(r -> authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
        }
        return authorities;
    });
    return converter;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Bean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- A very common production bug when migrating to OAuth2 JWTs is writing `@PreAuthorize("hasRole('ADMIN')")` when the Identity Provider (Keycloak, Okta, Auth0) places roles in a custom `"roles"` or `"realm_access.roles"` claim that `JwtGrantedAuthoritiesConverter` ignores by default! Always register a custom `JwtAuthenticationConverter`.

---

<a id="q6-11"></a>
### Q6.11: Explain the 4 OAuth 2.0 Roles (`Resource Owner`, `Client`, `Authorization Server`, `Resource Server`) and how OpenID Connect (`OIDC`) extends OAuth 2.0.

#### 1. Formal Technical Definition & Internal Mechanics
- **OAuth 2.0 Roles**:
  1. **Resource Owner**: The end user who owns the data and grants consent.
  2. **Client**: The application (SPA, Mobile App, BFF) requesting access on behalf of the user.
  3. **Authorization Server (IdP - Keycloak, Okta, Entra ID)**: Authenticates the user and issues tokens (`id_token`, `access_token`, `refresh_token`).
  4. **Resource Server**: Your Spring Boot microservice (`/api/orders`) that verifies the `access_token` and serves protected resources.
- **OAuth 2.0 vs OpenID Connect (OIDC)**:
  - **OAuth 2.0 is purely an Authorization delegation protocol** (it defines how to get an `access_token` for an API, not who the user is).
  - **OIDC (`scope=openid profile email`)** is an **Authentication layer on top of OAuth 2.0** that adds a standardized **`id_token` (JWT)** representing the user's login identity for the Client application, plus the `/userinfo` endpoint.

#### 2. Technical Architecture & Execution Diagram
```
[Resource Owner (User)]
         │ (1. Authenticates & Consents)
         ▼
[Authorization Server (IdP)] ──(2. Issues ID Token + Access Token + Refresh Token)──► [Client App]
                                                                                          │
                                                           (3. Sends ONLY Access Token)   │
                                                                                          ▼
                                                                             [Resource Server API]
```

#### 3. Detailed Technical Example & Production Code
```java
// Crucial Distinction:
// - ID Token (aud = client_id): Consumed ONLY by the Frontend/BFF Client to know who logged in.
// - Access Token (aud = api://order-service): Sent in Authorization: Bearer header to Resource Servers!
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never accept an OIDC `id_token` to authorize API calls on a Resource Server**: Your Spring Boot Resource Server must verify the `aud` (Audience) claim of the incoming JWT to ensure it is an `access_token` intended for your API (`aud = "api://order-service"`), not an `id_token` stolen from a third-party client!

---

<a id="q6-12"></a>
### Q6.12: Compare OAuth 2.1 Grant Types: `Authorization Code + PKCE` vs `Client Credentials` vs why `Implicit` and `Password (ROPC)` Grants were removed.

#### 1. Formal Technical Definition & Internal Mechanics
- **1. `Authorization Code + PKCE` (For All User-Facing Apps: SPAs, Mobile, Server Web Apps)**:
  - Browser redirects user to IdP login page; IdP redirects back with a short-lived one-time `?code=xyz`; Client exchanges `code + code_verifier` at the `/oauth/token` backchannel endpoint for tokens.
- **2. `Client Credentials` (Machine-to-Machine / Cron / Microservice-to-Microservice)**:
  - No end user involved. `Service A` authenticates directly with the IdP using its `client_id` + `client_secret` (or mTLS / private key JWT) to obtain a service `access_token`.
- **Why `Implicit Grant` (`response_type=token`) was removed in OAuth 2.1**: Exposed the raw `access_token` in the browser URL fragment (`#access_token=...`), leaking tokens into browser history, referrer headers, and XSS scripts without any refresh mechanism.
- **Why `Resource Owner Password Credentials (ROPC)` was removed**: Required the user to type their raw username/password directly into the client app (exposing credentials to the client and breaking MFA/SSO/Passkeys).

#### 2. Technical Architecture & Execution Diagram
```
+-----------------------------------+---------------------------------------------+
| Use Case                          | OAuth 2.1 Standard Grant Type               |
+-----------------------------------+---------------------------------------------+
| React/Angular SPA, iOS/Android    | Authorization Code + PKCE (Public Client)   |
| BFF / Backend Web App             | Authorization Code + PKCE (Confidential)    |
| Microservice / Daemon / Cron Job  | Client Credentials Grant                    |
| Smart TV / CLI without Browser    | Device Authorization Grant (RFC 8628)       |
+-----------------------------------+---------------------------------------------+
```

#### 3. Detailed Technical Example & Production Code
```yaml
# Spring Boot OAuth2 Client Configuration for Machine-to-Machine (Client Credentials):
spring:
  security:
    oauth2:
      client:
        registration:
          inventory-m2m:
            provider: internal-idp
            client-id: order-service-client
            client-secret: ${OAUTH_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: inventory:reserve
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The YAML block begins with `spring:`, establishing the target resource, dependency descriptor, or execution context for **Compare OAuth 2.1 Grant Types: `Authorization Code + PKCE` vs `Client Credentials` vs why `Implicit` and `Password (ROPC)` Grants were removed.**.
  2. **Core Rule / Parameter Evaluation**: At `inventory-m2m:`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `scope: inventory:reserve`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- In OAuth 2.1, **PKCE is required for ALL Authorization Code flows**—both public clients (SPAs/mobile) and confidential clients (backend servers with a `client_secret`).

---

<a id="q6-13"></a>
### Q6.13: Explain the Testing Pyramid (Unit vs Slice/Integration vs Contract vs End-to-End Tests) and why the "Ice Cream Cone" anti-pattern kills velocity.

#### 1. Formal Technical Definition & Internal Mechanics
- **70% Unit Tests (`JUnit 5 + Mockito + AssertJ`)**: Test domain entities, value objects, and service logic in pure memory (`< 5ms` per test, zero Spring context startup).
- **20% Slice & Integration Tests (`@WebMvcTest`, `@DataJpaTest` + `Testcontainers`, `WireMock`)**: Verify SQL queries, Flyway migrations, Jackson serialization, and Spring Security filter chains against real Dockerized PostgreSQL/Kafka containers (`1–5s`).
- **8% Contract Tests (`Pact`)**: Verify API/Event producer-consumer schemas without deploying full environments.
- **2% End-to-End Smoke Tests**: Verify critical happy-path checkout flows.

#### 2. Technical Architecture & Execution Diagram
```
       /\          2%  E2E Smoke Tests (Slow, Expensive)
      /  \
     /────\        8%  Consumer-Driven Contract Tests (Pact)
    /      \
   /────────\      20% Slice & Testcontainers Integration (@DataJpaTest, @WebMvcTest)
  /          \
 /────────────\    70% Fast Domain & Service Unit Tests (JUnit 5 + Mockito, < 5ms)
```

#### 3. Detailed Technical Example & Production Code
```java
// Fast Pure-Java Unit Test (Runs in 2ms without @SpringBootTest!):
@ExtendWith(MockitoExtension.class)
class PricingCalculatorTest {
    @Mock private TaxRateProvider taxProvider;
    @InjectMocks private PricingCalculator calculator;

    @Test
    void shouldApplyVipDiscountAndTaxExactToFourDecimals() {
        when(taxProvider.getRate("US-NY")).thenReturn(new BigDecimal("0.08875"));
        BigDecimal total = calculator.calculateTotal(new BigDecimal("100.00"), true, "US-NY");
        assertThat(total).isEqualByComparingTo("97.9875");
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `PricingCalculatorTest` (annotated with `@SpringBootTest`, `@ExtendWith`, `@Mock`, `@InjectMocks`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldApplyVipDiscountAndTaxExactToFourDecimals(...)`, execution proceeds step-by-step: the core statement `void shouldApplyVipDiscountAndTaxExactToFourDecimals() {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **BigDecimal Assertion Pitfall**: Never use `assertEquals(new BigDecimal("10.0"), new BigDecimal("10.00"))` in JUnit—it fails because `BigDecimal.equals()` checks `scale` (`1 != 2`)! Always use AssertJ's **`assertThat(actual).isEqualByComparingTo("10.00")`**.

---

<a id="q6-14"></a>
### Q6.14: Compare JUnit 5 Lifecycle Annotations (`@BeforeAll`, `@BeforeEach`, `@ParameterizedTest`, `@Nested`) and Mockito (`@Mock`, `@Spy`, `@Captor`, `BDDMockito`).

#### 1. Formal Technical Definition & Internal Mechanics
- **`@Mock`**: Creates a complete shell proxy where all methods return default empty values (`null`, `0`, `emptyList`) unless stubbed (`when(...).thenReturn(...)`).
- **`@Spy`**: Wraps a **real object instance**, executing real methods unless a specific method is stubbed (`doReturn(...).when(spy).someMethod()`).
- **`@Captor` (`ArgumentCaptor<T>`)**: Captures the exact object passed to a dependency (e.g., `outboxRepository.save(captor.capture())`) so you can assert on fields generated inside the method!
- **`@ParameterizedTest` (`@CsvSource`, `@MethodSource`, `@EnumSource`)**: Runs the same test method across dozens of boundary-condition inputs cleanly.

#### 2. Technical Architecture & Execution Diagram
```
Service Under Test ──► Calls outboxRepository.save(outboxEvent)
                              │
                              ▼
                 ArgumentCaptor<OutboxEvent> captures instance
                              │
                              ▼
                 assertThat(captor.getValue().getAggregateId()).isEqualTo("99")
```

#### 3. Detailed Technical Example & Production Code
```java
@ParameterizedTest(name = "amount={0}, status={1} -> allowed={2}")
@CsvSource({
    "100.00, ACTIVE, true",
    "0.00,   ACTIVE, false",
    "100.00, FROZEN, false"
})
void shouldValidateTransferEligibility(BigDecimal amount, AccountStatus status, boolean expected) {
    Account acc = new Account(1L, status, new BigDecimal("500.00"));
    assertThat(acc.canTransfer(amount)).isEqualTo(expected);
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@ParameterizedTest`, `@CsvSource`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldValidateTransferEligibility(...)`, execution proceeds step-by-step: the core statement `})` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Stubbing a `@Spy` Trap**: When stubbing a `@Spy`, always use `doReturn(val).when(spy).method()` instead of `when(spy.method()).thenReturn(val)`—because `when(spy.method())` actually **invokes the real `spy.method()` first**!

---

<a id="q6-15"></a>
### Q6.15: Compare `@SpringBootTest` vs Spring Boot Test Slices (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`) and why `@SpringBootTest` everywhere slows CI builds by 10x.

#### 1. Formal Technical Definition & Internal Mechanics
- **`@SpringBootTest`**: Bootstraps the **entire `ApplicationContext`** (all controllers, services, repositories, Kafka listeners, schedulers, HikariCP).
- **`@WebMvcTest(OrderController.class)`**: Loads **ONLY the Web & Security layer** (`@RestController`, `@ControllerAdvice`, `SecurityFilterChain`, `MockMvc`) and ignores all `@Service` and `@Repository` beans. Starts in `< 1s`!
- **`@DataJpaTest`**: Loads **ONLY JPA repositories, `EntityManager`, and Flyway**, rolling back each test transaction automatically at the end of the test.
- **`@JsonTest`**: Loads **ONLY Jackson `ObjectMapper`** and `@JsonComponent` serializers to test JSON serialization/deserialization edge cases.

#### 2. Technical Architecture & Execution Diagram
```
+-------------------+------------------------------------------------+-------------------+
| Annotation        | Beans Loaded into Context                      | Typical Speed     |
+-------------------+------------------------------------------------+-------------------+
| @JsonTest         | ObjectMapper, @JsonComponent                   | ~0.3s             |
| @WebMvcTest       | @Controller, @ControllerAdvice, Security, MVC  | ~0.8s             |
| @DataJpaTest      | @Repository, EntityManager, Flyway, DataSource | ~1.5s             |
| @SpringBootTest   | Entire Application (All layers + Messaging)    | ~8.0s - 25.0s     |
+-------------------+------------------------------------------------+-------------------+
```

#### 3. Detailed Technical Example & Production Code
```java
@WebMvcTest(controllers = OrderController.class)
@Import(SecurityConfig.class)
class OrderControllerSliceTest {

    @Autowired private MockMvc mockMvc;
    @MockitoBean private OrderService orderService; // Spring Boot 3.4+ @MockitoBean replaces @MockBean

    @Test
    void shouldReturn403WhenUserLacksWriteScope() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders:read")))
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"productId\":10,\"quantity\":1}"))
            .andExpect(status().isForbidden());
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `OrderControllerSliceTest` (annotated with `@WebMvcTest`, `@Import`, `@Autowired`, `@MockitoBean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldReturn403WhenUserLacksWriteScope(...)`, execution proceeds step-by-step: the core statement `mockMvc.perform(post("/api/v1/orders")` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Spring Context Caching Trap with `@MockBean` / `@MockitoBean`**: Every distinct combination of `@MockitoBean` declarations in a `@SpringBootTest` forces Spring TestContext Framework to **destroy and recreate a brand-new `ApplicationContext`**! Keep `@MockitoBean` inside fast `@WebMvcTest` slices, or define a single shared `AbstractIntegrationTest` base class for `@SpringBootTest` so the context is cached and reused across all integration tests.

---

<a id="q6-16"></a>
### Q6.16: How do you test Spring Security rules using `@WithMockUser`, `@WithSecurityContext`, and `SecurityMockMvcRequestPostProcessors.jwt()`?

#### 1. Formal Technical Definition & Internal Mechanics
- Generating real RSA-signed JWT strings in every controller test is tedious and brittle.
- `spring-security-test` provides:
  - **`mockMvc.perform(get(...).with(jwt().jwt(j -> j.subject("user-42").claim("tenant_id", "acme")).authorities(...)))`**: Injects a fully populated `JwtAuthenticationToken` directly into the `SecurityContext` for `MockMvc` requests, testing your exact `SecurityFilterChain` and `@PreAuthorize` rules!
  - **`@WithMockUser(username = "alice", roles = {"ADMIN"})`**: Populates `SecurityContextHolder` for service-layer method security (`@PreAuthorize`) tests.

#### 2. Technical Architecture & Execution Diagram
```
MockMvc Request + .with(jwt().claim("tenant_id", "acme"))
       │
       ▼
Populates JwtAuthenticationToken in SecurityContextHolder
       │
       ▼
Tests real AuthorizationFilter + @PreAuthorize + @AuthenticationPrincipal Jwt!
```

#### 3. Detailed Technical Example & Production Code
```java
@Test
void shouldAllowCustomerToReadOwnOrderViaJwtClaim() throws Exception {
    mockMvc.perform(get("/api/v1/orders/99")
            .with(jwt()
                .jwt(builder -> builder.subject("cust-42").claim("tenant_id", "acme"))
                .authorities(new SimpleGrantedAuthority("SCOPE_orders:read"))))
        .andExpect(status().isOk());
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Test`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldAllowCustomerToReadOwnOrderViaJwtClaim(...)`, execution proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- When testing `POST`/`PUT`/`DELETE` endpoints with `@WebMvcTest` where CSRF is enabled, remember to add `.with(csrf())` (and `.with(csrf().asHeader())` for invalid token tests) to verify CSRF enforcement.

---

<a id="q6-17"></a>
### Q6.17: How do you protect Spring Boot applications against OWASP Top 10 Vulnerabilities (SQL Injection, XSS, SSRF, Broken Object Level Authorization / IDOR)?

#### 1. Formal Technical Definition & Internal Mechanics
- **1. BOLA / IDOR (Broken Object Level Authorization - OWASP API #1)**:
  - User `cust-42` is authenticated and calls `GET /api/v1/orders/999` (which belongs to `cust-88`). Checking only `hasAuthority('SCOPE_orders:read')` is NOT enough! You must verify **object ownership**: `WHERE id = :orderId AND customer_id = :authenticatedCustomerId`.
- **2. SQL Injection**:
  - Never concatenate user input into JPQL/SQL (`"SELECT * FROM users WHERE name = '" + input + "'"`). Always use parameterized queries (`:name` or `?`), `JdbcClient`, or Criteria API. Even for dynamic `ORDER BY` column names (which cannot use `?` bind parameters), validate against a strict Java `enum` or `Set.of("created_at", "amount")` allowlist!
- **3. Security Response Headers (`HttpSecurity.headers()`)**:
  - Configure `Strict-Transport-Security` (HSTS), `Content-Security-Policy` (CSP), `X-Content-Type-Options: nosniff`, and `X-Frame-Options: DENY`.

#### 2. Technical Architecture & Execution Diagram
```
OWASP API Security #1: BOLA / IDOR (Broken Object Level Authorization)
  Vulnerable Query: SELECT * FROM orders WHERE id = :orderId; (Any logged-in user can read Order #999!)
  Secure Query:     SELECT * FROM orders WHERE id = :orderId AND customer_id = :jwtSubjectId;
```

#### 3. Detailed Technical Example & Production Code
```java
// Safe Dynamic Sort Column Allowlisting (Prevents SQL Injection in ORDER BY):
private static final Map<String, String> ALLOWED_SORT_COLUMNS = Map.of(
    "createdAt", "created_at",
    "amount", "total_amount"
);

public String resolveSafeSortColumn(String userSortParam) {
    String column = ALLOWED_SORT_COLUMNS.get(userSortParam);
    if (column == null) {
        throw new IllegalArgumentException("Invalid sort field: " + userSortParam);
    }
    return column;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `resolveSafeSortColumn(...)`, execution proceeds step-by-step: the core statement `String column = ALLOWED_SORT_COLUMNS.get(userSortParam);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Highlighting BOLA / IDOR as the #1 API Vulnerability**: In senior security interviews, point out that SQLi and XSS are largely solved by modern frameworks (PreparedStatements + React auto-escaping), whereas **BOLA / IDOR** (forgetting `AND customer_id = :currentUserId` or `AND tenant_id = :currentTenantId`) is the #1 cause of real-world API data breaches today.

---

<a id="q6-18"></a>
### Q6.18: How do you securely manage Secrets in Production (Vault, AWS Secrets Manager, Kubernetes External Secrets) and prevent Secret Leakage in Logs/Actuator?

#### 1. Formal Technical Definition & Internal Mechanics
- **Zero Secrets in Git**: Never commit passwords, API keys, or private keys to `application.yml`.
- **Runtime Injection & Rotation**: Use **External Secrets Operator (ESO)** or **Vault Agent Injector** in Kubernetes to mount secrets from AWS Secrets Manager / HashiCorp Vault into memory-backed volumes (`/mnt/secrets-store`), and reload HikariCP credentials dynamically on rotation without restarting pods.
- **Actuator Hardening**: Only expose `health`, `info`, `prometheus` on the management port (`management.server.port=8081`), never `env`, `heapdump`, or `configprops` on the public internet!

#### 2. Technical Architecture & Execution Diagram
```
[AWS Secrets Manager / HashiCorp Vault]
                 │
                 ▼ (External Secrets Operator / CSI Driver)
[K8s tmpfs Volume Mount] ──► [Spring Boot configtree:/mnt/secrets/]
```

#### 3. Detailed Technical Example & Production Code
```yaml
# application.yml: Load mounted Kubernetes secret files via Spring Boot ConfigTree:
spring:
  config:
    import: "optional:configtree:/mnt/secrets/"

management:
  server:
    port: 8081 # Isolate Actuator on internal non-ingress port!
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The YAML block begins with `spring:`, establishing the target resource, dependency descriptor, or execution context for **How do you securely manage Secrets in Production (Vault, AWS Secrets Manager, Kubernetes External Secrets) and prevent Secret Leakage in Logs/Actuator?**.
  2. **Core Rule / Parameter Evaluation**: At `port: 8081 # Isolate Actuator on internal non-ingress port!`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `include: health,info,prometheus`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- **The `/actuator/heapdump` & `/actuator/env` Breach Vector**: Exposing `/actuator/heapdump` publicly allows an attacker to download the entire JVM memory image—including plaintext database passwords, JWT signing keys, and in-flight customer credit cards! Always restrict Actuator endpoints to `health,info,prometheus` on a separate internal port.

---

## Part B: Intermediate Questions (Q6.19 – Q6.35)

<a id="q6-19"></a>
### Q6.19: Walk through the complete OAuth 2.1 Authorization Code + PKCE (`S256`) Flow step-by-step, and explain what attack PKCE prevents.

#### 1. Formal Technical Definition & Internal Mechanics
- **The Attack PKCE Prevents (Authorization Code Interception)**: On mobile OSs (custom URL schemes `myapp://callback?code=xyz`) or browser extensions, a malicious app can intercept the one-time `authorization_code` returned in the redirect URL. If the client is a public client (no `client_secret`), the attacker could immediately exchange that stolen `code` for an `access_token`!
- **How PKCE (`Proof Key for Code Exchange`, RFC 7636) Defeats It**:
  1. Before redirecting to the IdP, the client generates a high-entropy random secret called **`code_verifier`** (43–128 chars) in memory.
  2. Client hashes it: **`code_challenge = Base64UrlEncode(SHA256(code_verifier))`**.
  3. **Front-Channel Redirect**: Client sends `code_challenge` + `code_challenge_method=S256` to `/oauth2/authorize`. The IdP stores `code_challenge` alongside the issued `authorization_code`.
  4. **Back-Channel Token Exchange**: Client sends `authorization_code` + the raw **`code_verifier`** to `POST /oauth2/token`.
  5. IdP computes `Base64UrlEncode(SHA256(code_verifier))` and verifies it matches the stored `code_challenge`. Even if an attacker intercepted `code`, they do not know `code_verifier` (because `SHA-256` is one-way)!

#### 2. Technical Architecture & Execution Diagram
```
[Client App]                                                    [Authorization Server (IdP)]
     │ 1. Generate random code_verifier
     │    Compute code_challenge = BASE64URL(SHA256(code_verifier))
     │
     ├── 2. GET /authorize?response_type=code&code_challenge=XYZ&code_challenge_method=S256 ──►│
     │                                                          (Stores code_challenge=XYZ)    │
     │◄─ 3. Redirect 302: /callback?code=AUTH_CODE_123 ────────────────────────────────────────┤
     │
     ├── 4. POST /token (code=AUTH_CODE_123 + raw code_verifier) ─────────────────────────────►│
     │                                                          (Verifies SHA256(verifier)==XYZ│
     │◄─ 5. 200 OK: { access_token, id_token, refresh_token } ─────────────────────────────────┘
```

#### 3. Detailed Technical Example & Production Code
```java
// Generating a RFC-7636 compliant PKCE code_verifier & S256 code_challenge in Java:
public record PkcePair(String codeVerifier, String codeChallenge) {
    public static PkcePair generate() throws Exception {
        byte[] randomBytes = new byte[32];
        new SecureRandom().nextBytes(randomBytes);
        String verifier = Base64.getUrlEncoder().withoutPadding().encodeToString(randomBytes);

        byte[] sha256 = MessageDigest.getInstance("SHA-256")
                .digest(verifier.getBytes(StandardCharsets.US_ASCII));
        String challenge = Base64.getUrlEncoder().withoutPadding().encodeToString(sha256);
        return new PkcePair(verifier, challenge);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `PkcePair` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `byte[] sha256 = MessageDigest.getInstance("SHA-256")` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never allow `code_challenge_method=plain`**: Always enforce `S256` on the Authorization Server so an attacker who observes the initial `/authorize` request cannot read the verifier.

---

<a id="q6-20"></a>
### Q6.20: Compare Symmetric (`HMAC-SHA256` / `HS256`) vs Asymmetric (`RSA-256` / `RS256`, `ECDSA` / `ES256`, `EdDSA`) JWT Signing in Microservices.

#### 1. Formal Technical Definition & Internal Mechanics
- **Symmetric Signing (`HS256`)**: The **exact same shared secret key** is used to both **sign** and **verify** the JWT.
  - **Fatal Flaw in Microservices**: If you give the shared `HS256` secret to 30 downstream microservices so they can verify tokens, **any compromised microservice now has the ability to forge arbitrary Admin JWTs**!
- **Asymmetric Signing (`RS256`, `ES256`, `Ed25519` - Production Standard)**:
  - **Only the Authorization Server (IdP)** holds the **Private Key** to sign tokens.
  - All 50 Resource Server microservices fetch the **Public Key** via the standard **`JWKS` (`/.well-known/jwks.json`)** endpoint. Microservices can **verify** tokens locally, but can **never forge** a token!

#### 2. Technical Architecture & Execution Diagram
```
Asymmetric Signing (ES256 / RS256):
  [Auth Server (IdP)] ──► Signs JWT using PRIVATE Key (Never leaves IdP HSM/Vault!)
  [50 Microservices]  ──► Verify JWT using PUBLIC Key (Fetched from /.well-known/jwks.json)
                          └── Zero ability to mint/forge tokens if a microservice is breached!
```

#### 3. Detailed Technical Example & Production Code
```yaml
# Spring Boot OAuth2 Resource Server configuring JWKS discovery:
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.acme.com/realms/production
          jws-algorithms: ES256, RS256
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The YAML block begins with `spring:`, establishing the target resource, dependency descriptor, or execution context for **Compare Symmetric (`HMAC-SHA256` / `HS256`) vs Asymmetric (`RSA-256` / `RS256`, `ECDSA` / `ES256`, `EdDSA`) JWT Signing in Microservices.**.
  2. **Core Rule / Parameter Evaluation**: At `resourceserver:`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `jws-algorithms: ES256, RS256`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- **Why `ES256` (ECDSA P-256) or `EdDSA` beats `RS256` (RSA-2048)**: `ES256` produces a **64-byte signature** (vs `256 bytes` for RSA-2048), making every HTTP `Authorization` header smaller, and signs tokens **5x–10x faster** with equivalent 128-bit cryptographic security!

---

<a id="q6-21"></a>
### Q6.21: How do JWKS (`/.well-known/jwks.json`), Key ID (`kid`), and Zero-Downtime Cryptographic Key Rotation work?

#### 1. Formal Technical Definition & Internal Mechanics
- When rotating JWT signing keys every 30–90 days, tokens signed with the old key (`kid = "key-2025-01"`) are still valid in users' hands for the next 15 minutes.
- **Zero-Downtime Key Rotation Protocol**:
  1. **Publish Phase**: IdP generates new key pair (`kid = "key-2025-02"`) and publishes **both** `"key-2025-01"` and `"key-2025-02"` public keys in `/.well-known/jwks.json` while still signing with `"key-2025-01"`.
  2. **Sign Phase**: After JWKS caches propagate, IdP starts signing new JWTs with `"key-2025-02"` (placing `"kid": "key-2025-02"` in the JWT Header).
  3. **Lookup**: When a Resource Server receives a JWT with `"kid": "key-2025-02"`, Spring Security's `NimbusJwtDecoder` checks its local JWK cache by `kid`. If `"key-2025-02"` isn't in cache yet, it automatically refreshes `/.well-known/jwks.json`!
  4. **Retire Phase**: Once all tokens signed by `"key-2025-01"` have expired (`> 15m`), remove `"key-2025-01"` from JWKS.

#### 2. Technical Architecture & Execution Diagram
```
JWT Header: {"alg": "ES256", "kid": "key-2025-02"}
                    │
                    ▼
[NimbusJwtDecoder in Resource Server]
  ├── Checks in-memory JWK Cache for kid="key-2025-02"
  └── Cache Miss? ──► Fetches https://auth.acme.com/.well-known/jwks.json (Rate-limited refresh!)
                      ──► Verifies signature with Public Key "key-2025-02"
```

#### 3. Detailed Technical Example & Production Code
```json
{
  "keys": [
    { "kty": "EC", "crv": "P-256", "kid": "key-2025-01", "use": "sig", "x": "...", "y": "..." },
    { "kty": "EC", "crv": "P-256", "kid": "key-2025-02", "use": "sig", "x": "...", "y": "..." }
  ]
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `{ "kty": "EC", "crv": "P-256", "kid": "key-2025-02", "use": "sig", "x": "..` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Random `kid` DoS Attack**: What if an attacker sends 10,000 requests/sec with random fake `"kid": "fake-uuid-999"` headers? If your JWT decoder fetches `/.well-known/jwks.json` on every unknown `kid` without rate-limiting, the attacker will DDoS your Identity Provider! Nimbus JOSE+JWT (`JWKSourceBuilder`) enforces a **refresh cooldown window (default 30s–5m)** to prevent JWKS refresh storms.

---

<a id="q6-22"></a>
### Q6.22: How do you implement Refresh Token Rotation, Reuse Detection, and Instant JWT Revocation in Production?

#### 1. Formal Technical Definition & Internal Mechanics
- **Access Tokens** are short-lived (`5–15 minutes`) and stateless.
- **Refresh Tokens** are long-lived (`7–30 days`), stored in the database/Redis, and belong to a **Token Family (`family_id`)**.
- **Refresh Token Rotation + Automatic Reuse Detection (OAuth 2.1 Standard)**:
  - Every time `RefreshToken_1` is used at `/oauth2/token`, the server **marks `RefreshToken_1` as `USED`** and issues a brand-new `RefreshToken_2` (same `family_id`).
  - **The Breach Detector**: If an attacker stole `RefreshToken_1` and tries to use `RefreshToken_1` *after* the legitimate user already rotated to `RefreshToken_2` (or vice versa), the server sees a **`USED` token being presented again**!
  - The server immediately **revokes the ENTIRE `family_id`** (invalidating `RefreshToken_2` as well) and forces the user to re-authenticate!

#### 2. Technical Architecture & Execution Diagram
```
Token Family #88:
  1. Login issues RT_1 (Status = ACTIVE)
  2. Legitimate User refreshes with RT_1 ──► RT_1 marked USED; Server issues RT_2 (ACTIVE)
  3. Attacker replays stolen RT_1!
       └── Server sees RT_1 is ALREADY USED!
       └── BREACH DETECTED! Server revokes ALL tokens in Family #88 (RT_2 immediately invalidated!)
```

#### 3. Detailed Technical Example & Production Code
```java
@Transactional
public TokenPair rotateRefreshToken(String rawRefreshToken) {
    String tokenHash = sha256(rawRefreshToken);
    RefreshTokenEntity token = refreshTokenRepo.findByTokenHashForUpdate(tokenHash)
            .orElseThrow(() -> new InvalidTokenException("Unknown token"));

    if (token.getStatus() == TokenStatus.USED || token.getStatus() == TokenStatus.REVOKED) {
        // Compromised token family detected! Revoke entire family immediately!
        refreshTokenRepo.revokeEntireFamily(token.getFamilyId());
        securityAuditService.alertTokenReuseAttack(token.getUserId(), token.getFamilyId());
        throw new CompromisedTokenFamilyException("Refresh token reuse detected");
    }
    token.markUsed();
    return issueNewTokenPairInFamily(token.getUserId(), token.getFamilyId());
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Transactional`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: Spring and Hibernate coordinate the `PersistenceContext` (`Session`), SQL statement generation, JDBC connection binding, and transaction commit/rollback interception.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Always hash Refresh Tokens (`SHA-256`) before storing them in the database**: Unlike passwords, Refresh Tokens are already high-entropy 256-bit random strings, so a single fast `SHA-256` hash is sufficient to ensure that a database read breach cannot be used to hijack active user sessions.

---

<a id="q6-23"></a>
### Q6.23: Explain JWT Algorithm Confusion (`alg: none` and `RS256`-to-`HS256` Public Key Downgrade Attacks) and how to prevent them.

#### 1. Formal Technical Definition & Internal Mechanics
- **Attack 1 (`"alg": "none"`)**: The JWT spec historically included an `"alg": "none"` unsigned token type. Naive custom parsers trusted the `alg` header inside the incoming untrusted token and accepted unsigned forged tokens!
- **Attack 2 (`RS256` $\rightarrow$ `HS256` Key Confusion)**:
  - Server expects `RS256` and passes its **RSA Public Key** (which is publicly downloadable by anyone at `/.well-known/jwks.json`!) into a naive `jwt.verify(token, key)` method.
  - Attacker modifies the JWT Header to `"alg": "HS256"` and signs the forged JWT using the server's **public RSA PEM bytes as an HMAC symmetric secret**!
  - If the library trusts the header's `alg`, it verifies the HMAC signature using the public key bytes and accepts the forged token!

#### 2. Technical Architecture & Execution Diagram
```
RS256-to-HS256 Key Confusion Attack:
  1. Attacker downloads Server's Public RSA Key (public.pem)
  2. Attacker crafts JWT with Header {"alg": "HS256"} & signs with HMAC(public.pem)
  3. Vulnerable Verifier reads "alg":"HS256" from header & verifies HMAC using public.pem ──► PWNED!

Fix: Hardcode expected algorithm (ES256/RS256) in JwtDecoder and reject any other 'alg'!
```

#### 3. Detailed Technical Example & Production Code
```java
@Bean
public JwtDecoder jwtDecoder(OAuth2ResourceServerProperties properties) {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri(properties.getJwt().getJwkSetUri())
        .jwsAlgorithm(SignatureAlgorithm.ES256) // Explicitly pins expected Asymmetric Algorithm!
        .build();

    // Enforce Issuer, Timestamp (exp/nbf), AND Audience validation:
    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(properties.getJwt().getIssuerUri());
    OAuth2TokenValidator<Jwt> audienceValidator = new JwtClaimValidator<List<String>>(
        JwtClaimNames.AUD, aud -> aud != null && aud.contains("api://order-service")
    );
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(withIssuer, audienceValidator));
    return decoder;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Bean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Mandatory JWT Validations Checklist**: Whenever validating a JWT in production, verify all 5: **1)**Pinned Cryptographic Algorithm (`ES256`/`RS256`), **2)** Signature against JWKS `kid`, **3)** `iss` (Issuer), **4)** `exp` & `nbf` (Timestamps), and **5)** `aud` (Audience matches *this* specific microservice!).

---

<a id="q6-24"></a>
### Q6.24: Where should Access Tokens and Refresh Tokens be stored in Browser SPAs (`localStorage` vs `HttpOnly` Cookies vs the BFF Pattern)?

#### 1. Formal Technical Definition & Internal Mechanics
- **Why `localStorage` / `sessionStorage` is Dangerous**: Any Cross-Site Scripting (**XSS**) vulnerability (even from a single compromised npm package or third-party script) can execute `window.localStorage.getItem('access_token')` and exfiltrate all tokens!
- **The OAuth 2.1 Backend-For-Frontend (BFF) Pattern (Gold Standard)**:
  - The React/Angular SPA **never touches or sees raw OAuth2 tokens at all**!
  - A lightweight **Spring Cloud Gateway BFF** performs the OAuth2 Authorization Code + PKCE exchange with the IdP, stores the `access_token` and `refresh_token` in Redis, and issues an encrypted **`HttpOnly; Secure; SameSite=Strict`** session cookie to the browser.
  - When the browser calls `/api/orders`, the BFF swaps the cookie for `Authorization: Bearer <access_token>` before forwarding to downstream microservices.

#### 2. Technical Architecture & Execution Diagram
```
[Browser SPA (Zero Tokens in JS!)]
       │
       │ (Cookie: __Host-session=xyz; HttpOnly; Secure; SameSite=Strict)
       ▼
[Spring Cloud Gateway BFF (Token Relay Filter)]
       │ (Looks up Access Token in Redis & attaches Authorization: Bearer <JWT>)
       ▼
[Internal Microservice Resource Server]
```

#### 3. Detailed Technical Example & Production Code
```yaml
# Spring Cloud Gateway BFF with TokenRelay Filter:
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay= # Automatically swaps OAuth2 session for Authorization: Bearer <access_token>!
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The YAML block begins with `spring:`, establishing the target resource, dependency descriptor, or execution context for **Where should Access Tokens and Refresh Tokens be stored in Browser SPAs (`localStorage` vs `HttpOnly` Cookies vs the BFF Pattern)?**.
  2. **Core Rule / Parameter Evaluation**: At `gateway:`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `- TokenRelay= # Automatically swaps OAuth2 session for Authorization: Bearer <ac`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- **Use the `__Host-` Cookie Prefix**: Naming your session cookie `__Host-session` instructs modern browsers to enforce that the cookie **must** have `Secure`, `Path=/`, and **no `Domain` attribute** (preventing subdomain cookie-tossing attacks!).

---

<a id="q6-25"></a>
### Q6.25: How do you write a Custom `OncePerRequestFilter` in Spring Security, and why must you use `OncePerRequestFilter` instead of raw `Filter`?

#### 1. Formal Technical Definition & Internal Mechanics
- In Jakarta Servlet containers, a single HTTP request can be internally dispatched multiple times (`RequestDispatcher.forward()`, `/error` error-page dispatches, or `ASYNC` dispatches).
- A raw `jakarta.servlet.Filter` will execute **2 or 3 times** for the same HTTP request during `ERROR` or `FORWARD` dispatches!
- **`OncePerRequestFilter`** stores a request attribute (`alreadyFilteredAttributeName`) to guarantee `doFilterInternal()` executes **at most once per logical request**.

#### 2. Technical Architecture & Execution Diagram
```
HTTP Request ──► [MyCustomOncePerRequestFilter (Sets Attribute 'FILTERED'=true)]
                       │
                       ▼
                 Controller throws Exception ──► Tomcat dispatches to /error
                       │
                       ▼
                 [MyCustomOncePerRequestFilter sees 'FILTERED'==true ──► Skips duplicate run!]
```

#### 3. Detailed Technical Example & Production Code
```java
public class TenantHeaderValidationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        String tenantId = request.getHeader("X-Tenant-Id");
        if (tenantId == null || tenantId.isBlank()) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
            response.getWriter().write("{\"title\":\"Missing X-Tenant-Id header\",\"status\":400}");
            return; // Short-circuit filter chain!
        }
        filterChain.doFilter(request, response);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `TenantHeaderValidationFilter` (annotated with `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `doFilterInternal(...)`, execution proceeds step-by-step: the core statement `response.setStatus(HttpServletResponse.SC_BAD_REQUEST);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Double-Registration Trap with `@Component` on a Security Filter**: If you annotate a custom security `Filter` with `@Component` AND also add it via `http.addFilterBefore(myFilter, ...)`, Spring Boot automatically registers `@Component` filters in the **root Tomcat filter chain** as well as inside `SecurityFilterChain`—running your filter twice and executing it even on excluded paths! Either omit `@Component` (`new MyFilter()`) or register a `FilterRegistrationBean` with `registration.setEnabled(false)`.

---

<a id="q6-26"></a>
### Q6.26: How do `ExceptionTranslationFilter`, `AuthenticationEntryPoint`, and `AccessDeniedHandler` work, and why do exceptions thrown in earlier Filters miss `@RestControllerAdvice`?

#### 1. Formal Technical Definition & Internal Mechanics
- `@RestControllerAdvice` (`@ExceptionHandler`) is part of **Spring MVC (`DispatcherServlet`)**.
- Security filters run **before** `DispatcherServlet` is ever entered! Therefore, an exception thrown inside a Security Filter will **NOT** be caught by `@RestControllerAdvice` unless you explicitly delegate to **`HandlerExceptionResolver`**!
- Inside the security chain, `ExceptionTranslationFilter` sits right before `AuthorizationFilter`:
  - If `AuthenticationException` (or `AccessDeniedException` for an anonymous user) is thrown, it invokes **`AuthenticationEntryPoint`** (`401`).
  - If `AccessDeniedException` is thrown for an already-authenticated user, it invokes **`AccessDeniedHandler`** (`403`).

#### 2. Technical Architecture & Execution Diagram
```
[SecurityFilterChain]
  ├── CustomFilter throws Exception ──► Delegated to HandlerExceptionResolver ──► @RestControllerAdvice!
  └── AuthorizationFilter throws AccessDeniedException
        └── Caught by ExceptionTranslationFilter
              ├── Is Anonymous?     ──► AuthenticationEntryPoint (401)
              └── Is Authenticated? ──► AccessDeniedHandler      (403)
```

#### 3. Detailed Technical Example & Production Code
```java
// Delegating Security Filter exceptions cleanly to your @RestControllerAdvice (RFC 7807 ProblemDetail):
@Component
public class DelegatedAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private final HandlerExceptionResolver resolver;

    public DelegatedAuthenticationEntryPoint(
            @Qualifier("handlerExceptionResolver") HandlerExceptionResolver resolver) {
        this.resolver = resolver;
    }

    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res, AuthenticationException ex) {
        resolver.resolveException(req, res, null, ex);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `DelegatedAuthenticationEntryPoint` (annotated with `@RestControllerAdvice`, `@Component`, `@Qualifier`, `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `commence(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Injecting `@Qualifier("handlerExceptionResolver") HandlerExceptionResolver` into your custom `AuthenticationEntryPoint` and `AccessDeniedHandler` allows your `@RestControllerAdvice` to format `401` and `403` errors with the exact same RFC 7807 `ProblemDetail` JSON structure as all other application errors.

---

<a id="q6-27"></a>
### Q6.27: How do you configure Reusable Singleton Testcontainers (`@ServiceConnection`) in Spring Boot 3.1+ so 50 Integration Test classes share 1 PostgreSQL & Kafka container?

#### 1. Formal Technical Definition & Internal Mechanics
- If you put `@Testcontainers` and `@Container` on 50 test classes, Testcontainers starts and stops PostgreSQL and Kafka **50 times** (`50 * 6s = 5 minutes` wasted!).
- **The Singleton Container Pattern + `@ServiceConnection` (Spring Boot 3.1+)**:
  - Start the containers **once** in a `static` block inside an `AbstractIntegrationTest` base class (Testcontainers' **Ryuk** sidecar container automatically cleans them up when the JVM exits!).
  - Annotate the container bean/field with **`@ServiceConnection`**, which automatically configures `spring.datasource.url/username/password` or `spring.kafka.bootstrap-servers` without needing verbose `@DynamicPropertySource` boilerplate!

#### 2. Technical Architecture & Execution Diagram
```
Without Singleton Pattern:
  TestClass1 (Starts PG #1) -> Stops -> TestClass2 (Starts PG #2) -> Stops (Slow!)

With Singleton Container + @ServiceConnection:
  JVM Starts ──► Starts ONE PostgreSQL & ONE Kafka Container
                   ├── TestClass1..TestClass50 reuse the EXACT same containers & Spring Context!
  JVM Exits  ──► Ryuk sidecar container automatically destroys containers!
```

#### 3. Detailed Technical Example & Production Code
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class AbstractIntegrationTest {

    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withCommand("postgres", "-c", "fsync=off"); // Speeds up test writes by 5x!

    static {
        POSTGRES.start();
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `AbstractIntegrationTest` (annotated with `@SpringBootTest`, `@ServiceConnection`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `.withCommand("postgres", "-c", "fsync=off"); // Speeds up test writes by 5x` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never use H2 in-memory database to test PostgreSQL applications**: H2 lacks PostgreSQL `JSONB` operators, `FOR UPDATE SKIP LOCKED`, `ON CONFLICT DO UPDATE`, window framing nuances, and partitioning. Always use `Testcontainers` with the exact production `postgres:16` image.

---

<a id="q6-28"></a>
### Q6.28: How does `@Transactional` behave on Test Methods vs Service Methods, and why can `@Transactional` on a test give False Positives?

#### 1. Formal Technical Definition & Internal Mechanics
- When you annotate a `@Test` method with `@Transactional` (or use `@DataJpaTest`, which is meta-annotated with `@Transactional`), Spring wraps the **entire test method** in a single transaction and **rolls it back at the end of the test** (without ever committing or flushing by default!).
- **Why `@Transactional` on Tests Causes Dangerous False Positives**:
  1. **Lazy Loading Masks `LazyInitializationException`**: Because the test method itself keeps the Hibernate `Session` open across the service call and assertions, calling `order.getItems().size()` in your test succeeds—even though in production (with `open-in-view=false`), the `Session` closes when the service method returns and throws `LazyInitializationException`!
  2. **Skipped SQL Flush**: Unless you call `TestEntityManager.flush()`, Hibernate may never even execute the `UPDATE` or `INSERT` SQL statement against database constraints before rolling back at the end of the test!

#### 2. Technical Architecture & Execution Diagram
```
@Test @Transactional (False Positive Trap!):
  [Test Transaction Opens Session]
      ├── orderService.createOrder() (Saves entity into 1st-level cache)
      ├── orderService.getOrder()    (Returns SAME cached Java reference from 1st-level cache!)
      └── Test Ends -> ROLLBACK (Database FK/Unique constraints may NEVER be flushed!)
```

#### 3. Detailed Technical Example & Production Code
```java
// In @DataJpaTest, ALWAYS call flush() and clear() between write and read assertions:
@Test
void shouldPersistOrderAndLoadItemsFromDatabase(@Autowired TestEntityManager tem) {
    Order saved = orderRepository.save(new Order("ORD-1"));
    tem.flush(); // Forces SQL INSERT execution against DB constraints!
    tem.clear(); // Evicts 1st-level cache so findById() actually queries PostgreSQL!

    Order loaded = orderRepository.findById(saved.getId()).orElseThrow();
    assertThat(loaded.getOrderNumber()).isEqualTo("ORD-1");
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@DataJpaTest`, `@Test`, `@Autowired`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldPersistOrderAndLoadItemsFromDatabase(...)`, execution proceeds step-by-step: Spring and Hibernate coordinate the `PersistenceContext` (`Session`), SQL statement generation, JDBC connection binding, and transaction commit/rollback interception.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Mentioning `flush()` + `clear()` in `@DataJpaTest` and explaining why `@Transactional` on `@SpringBootTest` integration tests hides `LazyInitializationException` and `REQUIRES_NEW` behavior shows true testing mastery.

---

<a id="q6-29"></a>
### Q6.29: How do you test Asynchronous Code (Kafka Consumers, `@Async`, Event Listeners) reliably without `Thread.sleep()` using `Awaitility`?

#### 1. Formal Technical Definition & Internal Mechanics
- Using `Thread.sleep(3000)` in async tests is an anti-pattern:
  1. On a fast developer laptop where the Kafka message processes in `40ms`, you waste `2,960ms` on every test.
  2. On a heavily loaded CI runner where the message takes `3,100ms`, the test fails flakily!
- **`Awaitility`** polls an assertion condition every `50ms` and succeeds the instant the condition becomes true (e.g., at `40ms`), up to a configurable `atMost(Duration.ofSeconds(10))` timeout.

#### 2. Technical Architecture & Execution Diagram
```
Thread.sleep(3000): Always waits 3,000ms (Slow) & fails if CI takes 3,050ms (Flaky!)
Awaitility:         Polls every 50ms -> Condition passes at 50ms -> Test finishes immediately!
```

#### 3. Detailed Technical Example & Production Code
```java
@Test
void shouldConsumeOrderCreatedEventAndReserveInventory() {
    kafkaTemplate.send("orders.events.v1", "99", new OrderCreatedEvent(99L, "SKU-1", 2));

    Awaitility.await()
        .atMost(Duration.ofSeconds(10))
        .pollInterval(Duration.ofMillis(50))
        .untilAsserted(() -> {
            Optional<Reservation> res = reservationRepository.findByOrderId(99L);
            assertThat(res).isPresent();
            assertThat(res.get().getStatus()).isEqualTo(ReservationStatus.RESERVED);
        });
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Test`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldConsumeOrderCreatedEventAndReserveInventory(...)`, execution proceeds step-by-step: the core statement `.untilAsserted(() -> {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always use `untilAsserted(() -> assertThat(...))` with Awaitility so that if the timeout expires, Awaitility throws the descriptive AssertJ assertion diff rather than a generic boolean timeout message.

---

<a id="q6-30"></a>
### Q6.30: How do you test External HTTP Clients (`RestClient` / `WebClient`), Timeouts, Retries, and Circuit Breakers using `WireMock`?

#### 1. Formal Technical Definition & Internal Mechanics
- Mocking `RestClient` with Mockito does not test JSON serialization, HTTP status code mapping (`4xx` vs `5xx`), Resilience4j retry backoff, or socket read timeouts!
- **`WireMock`** starts a real lightweight HTTP server on a random local port (`@EnableWireMock`) that can simulate:
  - Fixed or random network delays (`withFixedDelay(3500)` to test `ReadTimeout`!).
  - Fault injection (`Fault.CONNECTION_RESET_BY_PEER`, `Fault.MALFORMED_RESPONSE_CHUNK`).
  - **Stateful Scenarios** (Attempt 1 returns `503 Service Unavailable`, Attempt 2 returns `503`, Attempt 3 returns `200 OK`—verifying Resilience4j `@Retry`!).

#### 2. Technical Architecture & Execution Diagram
```
[PaymentGatewayClient + Resilience4j @Retry]
       ├── Attempt 1 ──► [WireMock (State: STARTED)]   ──► Returns 503 (Transitions to RETRY_1)
       ├── Attempt 2 ──► [WireMock (State: RETRY_1)]   ──► Returns 200 OK!
       └── WireMock.verify(2, postRequestedFor(urlEqualTo("/v1/charges")))
```

#### 3. Detailed Technical Example & Production Code
```java
@Test
void shouldRetryOn503AndSucceedOnSecondAttempt() {
    stubFor(post(urlEqualTo("/v1/charges"))
        .inScenario("RetryScenario")
        .whenScenarioStateIs(Scenario.STARTED)
        .willReturn(aResponse().withStatus(503))
        .willSetStateTo("SecondAttempt"));

    stubFor(post(urlEqualTo("/v1/charges"))
        .inScenario("RetryScenario")
        .whenScenarioStateIs("SecondAttempt")
        .willReturn(okJson("{\"transactionId\":\"tx_123\",\"status\":\"APPROVED\"}")));

    ChargeResult result = paymentClient.charge(new ChargeRequest("idem-1", 5000L));

    assertThat(result.status()).isEqualTo("APPROVED");
    verify(2, postRequestedFor(urlEqualTo("/v1/charges"))
        .withHeader("Idempotency-Key", equalTo("idem-1")));
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Test`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldRetryOn503AndSucceedOnSecondAttempt(...)`, execution proceeds step-by-step: the core statement `.inScenario("RetryScenario")` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice how `verify(2, postRequestedFor(...).withHeader("Idempotency-Key", equalTo("idem-1")))` proves not only that the client retried, but also that both retry attempts carried the **exact same `Idempotency-Key`**!

---

<a id="q6-31"></a>
### Q6.31: How do you enforce Architectural Rules, Layer Boundaries, and Coding Standards automatically in JUnit using `ArchUnit`?

#### 1. Formal Technical Definition & Internal Mechanics
- Code reviews inevitably miss architectural drift (e.g., a developer importing a JPA `@Entity` or `@Repository` directly inside a `@RestController`, or introducing a cyclic package dependency between `order` and `payment`).
- **`ArchUnit`** analyzes compiled Java bytecode inside standard JUnit 5 tests and fails the CI build if any class violates your architectural rules!

#### 2. Technical Architecture & Execution Diagram
```
ArchUnit Bytecode Check in JUnit:
  [Controller Layer] ──► [Service Layer] ──► [Repository Layer] (Allowed)
  [Controller Layer] ───────(Direct Call)──► [Repository Layer] (Build Fails!)
```

#### 3. Detailed Technical Example & Production Code
```java
@AnalyzeClasses(packages = "com.acme.orders", importOptions = ImportOption.DoNotIncludeTests.class)
public class ArchitectureEnforcementTest {

    @ArchTest
    static final ArchRule layeredArchitectureShouldBeRespected = layeredArchitecture()
        .consideringAllDependencies()
        .layer("Controllers").definedBy("..controller..")
        .layer("Services").definedBy("..service..")
        .layer("Repositories").definedBy("..repository..")
        .whereLayer("Controllers").mayNotBeAccessedByAnyLayer()
        .whereLayer("Services").mayOnlyBeAccessedByLayers("Controllers", "Services")
        .whereLayer("Repositories").mayOnlyBeAccessedByLayers("Services");

    @ArchTest
    static final ArchRule noFieldInjectionAllowed = NO_CLASSES_SHOULD_USE_FIELD_INJECTION;

    @ArchTest
    static final ArchRule packageCyclesShouldBeFree =
        slices().matching("com.acme.orders.(*)..").should().beFreeOfCycles();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `ArchitectureEnforcementTest` (annotated with `@AnalyzeClasses`, `@ArchTest`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `.whereLayer("Controllers").mayNotBeAccessedByAnyLayer()` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Freezing Legacy Violations (`FreezingArchRule`)**: When introducing `ArchUnit` into a 5-year-old codebase with 80 existing violations, wrap rules in `FreezingArchRule.freeze(...)`, which records existing violations in a baseline file and fails the build **only if a PR introduces a new violation**!

---

<a id="q6-32"></a>
### Q6.32: Why is 80% Line Coverage (`JaCoCo`) often a vanity metric, and how does Mutation Testing (`PITest`) measure true test effectiveness?

#### 1. Formal Technical Definition & Internal Mechanics
- **Line Coverage Fallacy**: A test that calls `orderService.calculateDiscount(order)` without **any `assertThat(...)` statement** (or only asserts `assertNotNull(result)`) achieves **100% JaCoCo line coverage** while catching 0% of bugs!
- **Mutation Testing (`PITest`)**:
  - Automatically modifies ("mutates") your production bytecode in memory—e.g., flipping `>` to `>=`, changing `+` to `-`, or replacing a boolean return with `false` (creating a **Mutant**).
  - Runs your fast unit tests against each Mutant:
    - If a unit test **fails**, the Mutant is **Killed** (Good—your test actually verifies that logic!).
    - If all unit tests **still pass**, the Mutant **Survived** (Bad—your test has a blind spot on that boundary condition!).

#### 2. Technical Architecture & Execution Diagram
```
Production Code:  if (orderAmount >= 100) { applyDiscount(); }
PITest Mutant #1: if (orderAmount >  100) { applyDiscount(); } (Flipped >= to >)
  ├── Unit Test only tested orderAmount = 150 ──► Mutant SURVIVES! (Missing boundary test for 100!)
  └── Add Unit Test for orderAmount = 100     ──► Mutant KILLED!
```

#### 3. Detailed Technical Example & Production Code
```xml
<!-- Run PITest on critical domain/pricing packages in Maven: -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <configuration>
        <targetClasses><param>com.acme.domain.pricing.*</param></targetClasses>
        <mutationThreshold>85</mutationThreshold>
    </configuration>
</plugin>
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The XML block begins with `<!-- Run PITest on critical domain/pricing packages in Maven: -->`, establishing the target resource, dependency descriptor, or execution context for **Why is 80% Line Coverage (`JaCoCo`) often a vanity metric, and how does Mutation Testing (`PITest`) measure true test effectiveness?**.
  2. **Core Rule / Parameter Evaluation**: At `<configuration>`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `</plugin>`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- Run PITest with incremental history (`withHistory=true`) on core domain, pricing, and security modules in PR builds to guarantee boundary conditions (`>=` vs `>`) are genuinely asserted.

---

<a id="q6-33"></a>
### Q6.33: How do you make Time-Dependent Code (`LocalDateTime.now()`, `Instant.now()`, Token Expiration, Market Hours) 100% deterministic in tests using `java.time.Clock`?

#### 1. Formal Technical Definition & Internal Mechanics
- Calling `Instant.now()` or `LocalDate.now()` statically inside domain services makes tests impossible to run deterministically for leap years, end-of-month billing (`Jan 31`), or Daylight Saving Time transitions.
- **Solution**: Define a Spring `@Bean public Clock systemClock() { return Clock.systemUTC(); }`, inject `Clock` into your services, and call `Instant.now(clock)` / `LocalDate.now(clock)`. In unit tests, pass `Clock.fixed(Instant.parse("2025-02-28T23:59:59Z"), ZoneOffset.UTC)`!

#### 2. Technical Architecture & Execution Diagram
```
Production Bean: Clock.systemUTC() ──────────────────────────► Instant.now(clock) (Real Wall Clock)
Unit Test:       Clock.fixed("2025-02-28T23:59:59Z", UTC) ───► Instant.now(clock) (Frozen Time!)
```

#### 3. Detailed Technical Example & Production Code
```java
@Service
public class SubscriptionBillingService {
    private final Clock clock;

    public SubscriptionBillingService(Clock clock) {
        this.clock = clock;
    }

    public boolean isGracePeriodExpired(Instant dueAt) {
        return Instant.now(clock).isAfter(dueAt.plus(Duration.ofDays(3)));
    }
}

// Unit Test:
@Test
void shouldExpireExactlyOneNanosecondAfterThreeDayGracePeriod() {
    Instant dueAt = Instant.parse("2025-03-01T00:00:00Z");
    Clock frozenClock = Clock.fixed(dueAt.plus(Duration.ofDays(3)).plusNanos(1), ZoneOffset.UTC);
    SubscriptionBillingService service = new SubscriptionBillingService(frozenClock);

    assertThat(service.isGracePeriodExpired(dueAt)).isTrue();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `SubscriptionBillingService` (annotated with `@Service`, `@Test`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `isGracePeriodExpired(...)` and `shouldExpireExactlyOneNanosecondAfterThreeDayGracePeriod(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Enforce via `ArchUnit` that no class in `..service..` or `..domain..` calls the zero-arg `Instant.now()` or `LocalDateTime.now()`, requiring `now(Clock)` everywhere.

---

<a id="q6-34"></a>
### Q6.34: Compare Role-Based Access Control (`RBAC`), Attribute-Based Access Control (`ABAC`), and Relationship-Based Access Control (`ReBAC` / Google Zanzibar).

#### 1. Formal Technical Definition & Internal Mechanics
- **RBAC (`Role-Based`)**: Permissions are grouped into static roles (`ROLE_ADMIN`, `ROLE_SUPPORT`, `ROLE_VIEWER`). Simple, fast JWT evaluation, but leads to **"Role Explosion"** when permissions depend on dynamic context (e.g., `ROLE_US_EAST_BRANCH_42_MANAGER`).
- **ABAC (`Attribute-Based`)**: Evaluates policies against **Subject attributes** (`user.department`, `user.clearanceLevel`), **Resource attributes** (`document.classification`, `order.amount < 10000`), and **Environment attributes** (`timeOfDay`, `ipRange`).
- **ReBAC (`Relationship-Based` - Google Zanzibar / OpenFGA / SpiceDB)**: Models authorization as a graph of relationship tuples (`user:alice is editor of folder:eng`, `doc:design_spec has parent folder:eng` $\implies$ `alice` can edit `doc:design_spec`).

#### 2. Technical Architecture & Execution Diagram
```
+---------+----------------------------------------+----------------------------------------+
| Model   | Rule Example                           | Best Fit                               |
+---------+----------------------------------------+----------------------------------------+
| RBAC    | User has role 'BILLING_ADMIN'          | Coarse-grained internal backoffice     |
| ABAC    | user.region == order.region && amt<5k  | Context/Compliance/Limit policies      |
| ReBAC   | user:42 -> member -> org:9 -> owns -> doc| Hierarchical B2B SaaS (Drive/GitHub)   |
+---------+----------------------------------------+----------------------------------------+
```

#### 3. Detailed Technical Example & Production Code
```java
// Combining RBAC + ABAC cleanly in Spring Security via a Domain Permission Component:
@Component("paymentAuthz")
public class PaymentAuthorizationPolicy {
    public boolean canApprove(Authentication auth, PaymentOrder order) {
        Jwt jwt = (Jwt) auth.getPrincipal();
        String userRegion = jwt.getClaimAsString("region");
        BigDecimal approvalLimit = new BigDecimal(jwt.getClaimAsString("approval_limit"));

        return userRegion.equals(order.getRegion())
                && order.getAmount().compareTo(approvalLimit) <= 0
                && !jwt.getSubject().equals(order.getCreatedByUserId()); // 4-Eyes Maker-Checker rule!
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `PaymentAuthorizationPolicy` (annotated with `@Component`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `canApprove(...)`, execution proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice the **4-Eyes (Maker-Checker) Segregation of Duties** rule in the ABAC example above (`!jwt.getSubject().equals(order.getCreatedByUserId())`): in financial systems, the user who initiates a wire transfer can never be the same user who approves it!

---

<a id="q6-35"></a>
### Q6.35: How do you implement API Key Authentication alongside OAuth2 JWT in the same Spring Security application?

#### 1. Formal Technical Definition & Internal Mechanics
- B2B platforms often support **OAuth2 JWT Bearer tokens** for interactive web/mobile users AND **API Keys (`X-API-Key: ak_live_...`)** for merchant server integrations.
- **Secure API Key Storage Design**:
  1. Never store plaintext API keys in the database!
  2. Format the key with a public prefix + high-entropy secret: `ak_live_<8-char-key-id>_<32-char-random-secret>`.
  3. Store `key_id` (indexed for fast lookup) and the **SHA-256 hash** (with HMAC pepper) of the full API key in the database/cache.
  4. Compare hashes using **`MessageDigest.isEqual(expectedBytes, actualBytes)`** (constant-time byte comparison to prevent timing side-channel attacks!).

#### 2. Technical Architecture & Execution Diagram
```
API Key: ak_live_9f8a7b6c_X9mP2qL8vR4nK1wZ...
                 └───┬──┘ └────────┬────────┘
                Key ID Prefix    Secret
                     │
                     ▼
Lookup by key_id='9f8a7b6c' ──► Verify MessageDigest.isEqual(storedSha256, sha256(incomingKey))
```

#### 3. Detailed Technical Example & Production Code
```java
public boolean verifyApiKeyConstantTime(String incomingRawKey, byte[] storedSha256Hash) throws Exception {
    byte[] incomingHash = MessageDigest.getInstance("SHA-256")
            .digest(incomingRawKey.getBytes(StandardCharsets.UTF_8));
    // Constant-time comparison prevents byte-by-byte timing attacks!
    return MessageDigest.isEqual(incomingHash, storedSha256Hash);
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `verifyApiKeyConstantTime(...)`, execution proceeds step-by-step: the core statement `.digest(incomingRawKey.getBytes(StandardCharsets.UTF_8));` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why `String.equals()` or `Arrays.equals()` is insecure for HMAC/API Key verification**: Standard `equals()` returns `false` on the very first mismatched byte, allowing an attacker to measure nanosecond response differences to guess hashes byte-by-byte. Always use **`MessageDigest.isEqual()`**.

---

## Part C: Senior & Production Deep-Dive Questions (Q6.36 – Q6.52)

<a id="q6-36"></a>
### Q6.36: Compare Zero-Trust Microservice-to-Microservice Authentication: Token Relay vs OAuth 2.0 Token Exchange (RFC 8693) vs mTLS (SPIFFE/SPIRE).

#### 1. Formal Technical Definition & Internal Mechanics
- When `User Alice` calls `API Gateway` $\rightarrow$ `Order Service` $\rightarrow$ `Payment Service`:
  1. **Naive Token Relay (Passing Alice's initial JWT everywhere)**:
     - **Risk**: If `Order Service` forwards Alice's broad-audience JWT to an internal `Analytics Service`, `Analytics Service` could replay Alice's token to call `Payment Service` (**Confused Deputy / Token Replay Attack**)!
  2. **OAuth 2.0 Token Exchange (`RFC 8693` - On-Behalf-Of Flow)**:
     - `Order Service` exchanges Alice's incoming token at the Auth Server for a **down-scoped, audience-restricted token** (`sub = "alice"`, `act = {"sub": "order-service"}`, `aud = "payment-service"`, `scope = "payments:charge"`).
  3. **mTLS + SPIFFE (`spiffe://prod.acme.com/ns/payments/sa/order-service`)**:
     - Service Mesh (Istio / Linkerd) provisions short-lived X.509 workload certificates rotated hourly.
     - **Best Practice Defense-in-Depth**: **mTLS authenticates the calling *Service* (`act`)** at the transport layer, while the **JWT authenticates the end *User* (`sub`)** at the application layer!

#### 2. Technical Architecture & Execution Diagram
```
Zero-Trust Defense-in-Depth (mTLS + Down-Scoped JWT):
  [Order Service Pod]
     │  ├── Transport Layer (mTLS X.509 SAN): spiffe://acme/ns/default/sa/order-service (Who is calling?)
     │  └── App Layer (JWT Header):           sub="user-42", aud="payment-service"      (On whose behalf?)
     ▼
  [Payment Service Pod] (Verifies BOTH mTLS caller identity AND User JWT permissions!)
```

#### 3. Detailed Technical Example & Production Code
```java
// Verifying both the End-User ('sub') and the Immediate Acting Service ('act.sub') in RFC 8693 JWT:
@PreAuthorize("hasAuthority('SCOPE_payments:charge') and #jwt.getClaimAsMap('act')['sub'] == 'order-service'")
public PaymentReceipt chargeOrder(ChargeCommand cmd, @AuthenticationPrincipal Jwt jwt) {
    return paymentProcessor.execute(cmd, jwt.getSubject());
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@PreAuthorize`, `@AuthenticationPrincipal`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Combining **mTLS (SPIFFE workload identity)** for machine authentication with **JWT (`sub` + `aud`)** for user delegation is the gold-standard Zero-Trust architecture at top fintech and cloud companies.

---

<a id="q6-37"></a>
### Q6.37: Explain OAuth 2.0 DPoP (Demonstrating Proof-of-Possession, RFC 9449) and mTLS Sender-Constrained Tokens.

#### 1. Formal Technical Definition & Internal Mechanics
- A standard `Bearer` token means *"Whoever bears (holds) this string gets access."* If a Bearer token is leaked via a proxy log, browser bug, or network appliance, the attacker can replay it from any machine on Earth!
- **Sender-Constrained (Proof-of-Possession) Tokens** bind the `access_token` cryptographically to the client's private key:
  1. **`DPoP` (Demonstrating Proof-of-Possession, RFC 9449)**:
     - Client generates a non-exportable `WebCrypto` ECDSA key pair in the browser/mobile enclave.
     - When requesting a token, the IdP embeds the thumbprint of the client's public key inside the Access Token: `"cnf": {"jkt": "0ZcOCORZNYy..."}`.
     - On every API request (`Authorization: DPoP <access_token>`), the client sends a fresh **`DPoP` Proof JWT header** signed with its private key covering the exact HTTP method (`htm: "POST"`), URL (`htu: "https://api.acme.com/orders"`), timestamp (`iat`), and access token hash (`ath`).
     - Even if an attacker steals the `access_token`, it is **100% useless** without the client's private key!

#### 2. Technical Architecture & Execution Diagram
```
Access Token Payload: { "sub": "user-42", "cnf": { "jkt": "Thumbprint_of_Client_PublicKey" } }
HTTP Request Headers:
  Authorization: DPoP <access_token>
  DPoP: <Signed Proof JWT containing {htm:"POST", htu:"https://...", iat:..., ath:Hash(access_token)}>
```

#### 3. Detailed Technical Example & Production Code
```java
// Spring Security 6.2+ / Spring Authorization Server native DPoP validation:
// Verifies DPoP header signature, matches public key JWK thumbprint against token 'cnf.jkt',
// and checks 'htm', 'htu', and 'iat' freshness to prevent token replay!
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Mentioning **RFC 9449 `DPoP`** (or `RFC 8705` mTLS Certificate-Bound Tokens for OpenBanking/FAPI) when discussing how to prevent stolen Bearer token replay attacks immediately signals Staff-level security awareness.

---

<a id="q6-38"></a>
### Q6.38: How do you implement Field-Level Encryption (Envelope Encryption with AWS KMS / Vault Transit + JPA `AttributeConverter`) and Blind Indexing for searchable PII?

#### 1. Formal Technical Definition & Internal Mechanics
- Full-disk encryption (EBS/RDS encryption at rest) only protects against someone physically stealing a hard drive from the AWS data center—it does **not** protect against SQL injection, leaked database backups, or DBA snooping!
- **Envelope Encryption (`AES-256-GCM` + KMS)**:
  1. AWS KMS / HashiCorp Vault holds the **Master Key (`KEK` - Key Encryption Key)**.
  2. Application generates a **Data Encryption Key (`DEK`)**, encrypts the sensitive column (`ssn`, `bank_account_number`) using **authenticated `AES-256-GCM` with a unique 12-byte random IV per value**, and caches the decrypted `DEK` in memory for 15 minutes.
- **How do you query an `AES-GCM` encrypted column (`WHERE ssn = ?`)?**:
  - Because `AES-GCM` uses a random 12-byte IV on every write, encrypting `"123-45-6789"` twice produces **two completely different ciphertexts**! Therefore, you cannot run `WHERE ssn_encrypted = ?`.
  - **Solution (Blind Index)**: Store a second column `ssn_blind_index = HMAC_SHA256(blindIndexSecretKey, normalizedSsn)` and put a B+Tree `UNIQUE INDEX` on `ssn_blind_index`!

#### 2. Technical Architecture & Execution Diagram
```
Plaintext SSN: "123-45-6789"
  ├──► AES-256-GCM(DEK, random_12B_IV, "123-45-6789") ──► Column: ssn_ciphertext (Decryptable)
  └──► HMAC-SHA256(BlindIndexKey, "123-45-6789")      ──► Column: ssn_blind_idx  (Indexed for WHERE =)
```

#### 3. Detailed Technical Example & Production Code
```java
@Converter
public class AesGcmAttributeConverter implements AttributeConverter<String, String> {
    private static final int IV_LENGTH_BYTES = 12;
    private static final int TAG_LENGTH_BITS = 128;
    private final SecretKey dek;

    public AesGcmAttributeConverter(SecretKey dek) { this.dek = dek; }

    @Override
    public String convertToDatabaseColumn(String plaintext) {
        if (plaintext == null) return null;
        try {
            byte[] iv = new byte[IV_LENGTH_BYTES];
            SecureRandom.getInstanceStrong().nextBytes(iv);
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE, dek, new GCMParameterSpec(TAG_LENGTH_BITS, iv));
            byte[] cipherText = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
            byte[] combined = ByteBuffer.allocate(iv.length + cipherText.length)
                    .put(iv).put(cipherText).array();
            return "v1:" + Base64.getEncoder().encodeToString(combined);
        } catch (GeneralSecurityException e) {
            throw new IllegalStateException("Encryption failed", e);
        }
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        try {
            byte[] combined = Base64.getDecoder().decode(dbData.substring(3)); // Strip "v1:"
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            GCMParameterSpec spec = new GCMParameterSpec(TAG_LENGTH_BITS, combined, 0, IV_LENGTH_BYTES);
            cipher.init(Cipher.DECRYPT_MODE, dek, spec);
            byte[] plainBytes = cipher.doFinal(combined, IV_LENGTH_BYTES, combined.length - IV_LENGTH_BYTES);
            return new String(plainBytes, StandardCharsets.UTF_8);
        } catch (GeneralSecurityException e) {
            throw new IllegalStateException("Decryption/Tag verification failed", e);
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `AesGcmAttributeConverter` (annotated with `@Converter`, `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `convertToDatabaseColumn(...)` and `convertToEntityAttribute(...)`, execution proceeds step-by-step: the core statement `} catch (GeneralSecurityException e) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never use `AES/ECB` or static IVs with `AES/GCM`**: Reusing the same IV twice with the same key in `AES-GCM` ("nonce reuse") completely breaks GCM confidentiality and allows an attacker to forge authentication tags! Always generate a fresh 12-byte `SecureRandom` IV per encryption and prepend `"v1:"` key version prefix for key rotation.

---

<a id="q6-39"></a>
### Q6.39: How do you prevent Mass Assignment / Over-Posting Attacks and JSON Deserialization Gadget Attacks in Spring Boot?

#### 1. Formal Technical Definition & Internal Mechanics
- **Mass Assignment (OWASP API #3)**: If a `@RestController` accepts a JPA `@Entity` (`User`) directly in `@RequestBody`, an attacker can add `"role": "ADMIN"` or `"balance": 999999` to the JSON payload, and Jackson will bind those fields directly onto the managed entity!
  - **Fix**: **Never bind `@RequestBody` to JPA `@Entity` classes!** Always use dedicated command Java `record`s (`CreateUserRequest(String email, String name)`) containing *only* the fields the user is permitted to modify.
- **Jackson Polymorphic Deserialization Gadget Vulnerability (`@JsonTypeInfo` / `enableDefaultTyping`)**:
  - Enabling global default typing (`ObjectMapper.activateDefaultTyping()`) or `Id.CLASS` allows an attacker to pass `"@class": "com.sun.rowset.JdbcRowSetImpl"` in JSON and achieve Remote Code Execution (RCE)!
  - **Fix**: Never enable global default typing; if polymorphism is needed, use `@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)` with explicit `@JsonSubTypes` allowlists.

#### 2. Technical Architecture & Execution Diagram
```
VULNERABLE (@RequestBody UserEntity):
  JSON {"name": "Bob", "isAdmin": true} ──► Jackson sets userEntity.setAdmin(true)!

SECURE (@RequestBody UpdateProfileRequest Record):
  record UpdateProfileRequest(@NotBlank String name) {} ──► "isAdmin" doesn't exist on Record!
```

#### 3. Detailed Technical Example & Production Code
```java
public record UpdateProfileCommand(
    @NotBlank @Size(max = 100) String displayName,
    @Pattern(regexp = "^[a-z]{2}-[A-Z]{2}$") String locale
) {}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `UpdateProfileCommand` (annotated with `@NotBlank`, `@Size`, `@Pattern`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `@Pattern(regexp = "^[a-z]{2}-[A-Z]{2}$") String locale` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Enforce via `ArchUnit` that no `@RestController` method parameter or return type references any class annotated with `@jakarta.persistence.Entity`.

---

<a id="q6-40"></a>
### Q6.40: How does Reactive Spring Security (`@EnableWebFluxSecurity`, `SecurityWebFilterChain`, `ReactiveSecurityContextHolder`) differ from Servlet Security?

#### 1. Formal Technical Definition & Internal Mechanics
- In **Spring MVC (Servlet)**, a request is bound to a single thread (or Virtual Thread), so `SecurityContextHolder` stores the `SecurityContext` in a **`ThreadLocal`**.
- In **Spring WebFlux (Project Reactor / Netty)**, a single request hops across multiple non-blocking EventLoop threads as reactive operators execute! Therefore, `ThreadLocal` does **not** work!
- Instead, **`ReactiveSecurityContextHolder`** stores the `SecurityContext` inside **Project Reactor's immutable subscriber `Context` (`Mono.deferContextual(...)`)**, which flows downstream through the reactive pipeline regardless of which thread executes each step.

#### 2. Technical Architecture & Execution Diagram
```
Servlet Security:  [HTTP Thread] ──► SecurityContextHolder (ThreadLocal)
WebFlux Security:  [EventLoop-1] ──► [EventLoop-2] ──► ReactiveSecurityContextHolder (Reactor ContextView)
```

#### 3. Detailed Technical Example & Production Code
```java
@Bean
public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
    return http
        .csrf(ServerHttpSecurity.CsrfSpec::disable)
        .authorizeExchange(exchanges -> exchanges
            .pathMatchers("/actuator/health/**").permitAll()
            .anyExchange().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Bean`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never call blocking JDBC or `RestTemplate` inside a custom `ReactiveAuthenticationManager` or `WebFilter`**: Blocking an Netty EventLoop thread stalls hundreds of concurrent requests sharing that EventLoop core.

---

<a id="q6-41"></a>
### Q6.41: How do you prevent Brute-Force Login, Credential Stuffing, and Account Enumeration without allowing attackers to DoS legitimate users via Account Lockout?

#### 1. Formal Technical Definition & Internal Mechanics
- **The Naive Account Lockout DoS Trap**: If you permanently lock an account after 5 failed password attempts, an attacker can simply run a script sending 5 wrong passwords per hour for `ceo@company.com`, **permanently locking the CEO out of their account**!
- **Multi-Layered Production Defense**:
  1. **Progressive Exponential Delay per `(username, IP)`**: After 3 failures from an IP, require CAPTCHA / Turnstile + progressive delay (`2s -> 5s -> 15s -> 60s`).
  2. **Known-Good Device Fingerprinting**: If `alice@acme.com` is being brute-forced from a botnet in another country, allow Alice to still log in normally from her trusted browser/device cookie or MFA/Passkey!
  3. **Uniform Response Message & Constant-Time Dummy Hash**: Always return `"Invalid email or password"` in identical `~150ms` whether the email exists or not.

#### 2. Technical Architecture & Execution Diagram
```
Login Attempt (email, ip):
  ├── Check IP Rate Limit & ASN Reputation
  ├── Check (email + ip) Failed Counter in Redis
  ├── Verify Password (Runs dummy BCrypt hash if user not found -> Constant ~150ms!)
  └── If > 5 failures for email from NEW device ──► Require Step-Up MFA / Email Magic Link (Don't brick trusted devices!)
```

#### 3. Detailed Technical Example & Production Code
```java
@Component
public class LoginFailureRateLimiter {
    private final StringRedisTemplate redis;

    public void recordFailedAttempt(String email, String clientIp) {
        String key = "login:fail:" + email.toLowerCase() + ":" + clientIp;
        Long count = redis.opsForValue().increment(key);
        if (count != null && count == 1L) {
            redis.expire(key, Duration.ofMinutes(15));
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `LoginFailureRateLimiter` (annotated with `@Component`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `recordFailedAttempt(...)`, execution proceeds step-by-step: the core statement `Long count = redis.opsForValue().increment(key);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Trusted Proxy `X-Forwarded-For` Spoofing**: If you rate-limit by `request.getHeader("X-Forwarded-For")` without configuring `server.forward-headers-strategy=native` and restricting `server.tomcat.remoteip.internal-proxies`, an attacker can bypass IP rate limiting by sending a random fake `X-Forwarded-For: 1.2.3.4` header on every request!

---

<a id="q6-42"></a>
### Q6.42: How do you implement Step-Up Authentication (MFA / WebAuthn Passkeys) using the OIDC `acr` (Authentication Context Class Reference) and `amr` claims?

#### 1. Formal Technical Definition & Internal Mechanics
- A user logs in with standard password/SSO (`acr = "urn:acme:bronze"`, single-factor) to browse their dashboard.
- When the user clicks **"Wire $50,000"** or **"Change Bank Routing Number"**, the Payment API requires **Step-Up MFA** (`acr = "urn:acme:gold"` and `auth_time` within the last `5 minutes`!).
- If the JWT's `acr` is too low or `auth_time` is older than 5 minutes, the Resource Server returns `401 Unauthorized` with RFC 9470 header:
  `WWW-Authenticate: Bearer error="insufficient_user_authentication", acr_values="urn:acme:gold", max_age="300"`
  prompting the frontend to trigger a WebAuthn Passkey / TOTP challenge!

#### 2. Technical Architecture & Execution Diagram
```
1. User calls POST /wires ($50k) with Standard JWT (acr="bronze")
2. Resource Server rejects with 401 + WWW-Authenticate: error="insufficient_user_authentication", acr_values="gold"
3. Client prompts WebAuthn Passkey / Biometric prompt -> Obtains upgraded JWT (acr="gold", auth_time=now)
4. Client retries POST /wires ($50k) with upgraded JWT ──► 200 OK!
```

#### 3. Detailed Technical Example & Production Code
```java
@Component("stepUpEvaluator")
public class StepUpAuthenticationEvaluator {
    public boolean hasRecentMfa(Authentication authentication, int maxAgeSeconds) {
        if (!(authentication.getPrincipal() instanceof Jwt jwt)) return false;
        String acr = jwt.getClaimAsString("acr");
        Instant authTime = jwt.getClaimAsInstant("auth_time");
        return "urn:acme:gold".equals(acr)
                && authTime != null
                && authTime.isAfter(Instant.now().minusSeconds(maxAgeSeconds));
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `StepUpAuthenticationEvaluator` (annotated with `@Component`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `hasRecentMfa(...)`, execution proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Referencing **RFC 9470 (OAuth 2.0 Step Up Authentication Challenge Protocol)** (`acr`, `amr`, `auth_time`, and `insufficient_user_authentication`) is a textbook senior fintech security pattern.

---

<a id="q6-43"></a>
### Q6.43: How do you write Database Concurrency & Race Condition Integration Tests in JUnit 5 using `CountDownLatch` and `CyclicBarrier`?

#### 1. Formal Technical Definition & Internal Mechanics
- A sequential unit test will **never** catch a Lost Update, Double-Spend, or Deadlock bug!
- To prove your `@Version` optimistic lock, `SELECT ... FOR UPDATE`, or `INSERT ... ON CONFLICT` actually prevents double-spending under race conditions:
  1. Spawn $N = 20$ worker threads in an `ExecutorService`.
  2. Have all 20 threads wait at a **`CyclicBarrier(20)`** (or `CountDownLatch(1)` starting gun) so all 20 threads hit the `@Transactional` method at the **exact same microsecond**.
  3. Assert that for a wallet with `$100` balance and 20 concurrent `$100` withdrawal requests, **exactly 1 succeeds (`successCount == 1`), 19 fail cleanly, and final DB balance is `$0.00` (never `-$1,900.00`)**!

#### 2. Technical Architecture & Execution Diagram
```
20 Worker Threads ──► Wait at CyclicBarrier(20)
                             │ (All 20 released at the EXACT same microsecond!)
                             ▼
               walletService.withdraw(walletId, $100)
                 ├── 1 Thread commits ($100 -> $0)
                 └── 19 Threads throw InsufficientFunds / OptimisticLockException
               Assertion: Final Balance == $0.00 && SuccessCount == 1!
```

#### 3. Detailed Technical Example & Production Code
```java
@Test
void shouldPreventDoubleSpendUnder20ConcurrentWithdrawals() throws Exception {
    Long walletId = walletService.createWalletWithBalance(new BigDecimal("100.00"));
    int threads = 20;
    CyclicBarrier startBarrier = new CyclicBarrier(threads);
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failureCount = new AtomicInteger(0);

    try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
        List<Callable<Void>> tasks = IntStream.range(0, threads).mapToObj(i -> (Callable<Void>) () -> {
            startBarrier.await(5, TimeUnit.SECONDS); // Synchronize exact simultaneous start!
            try {
                walletService.withdraw(walletId, new BigDecimal("100.00"));
                successCount.incrementAndGet();
            } catch (Exception e) {
                failureCount.incrementAndGet();
            }
            return null;
        }).toList();
        executor.invokeAll(tasks);
    }

    assertThat(successCount.get()).isEqualTo(1);
    assertThat(failureCount.get()).isEqualTo(19);
    assertThat(walletService.getBalance(walletId)).isEqualByComparingTo("0.00");
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Test`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldPreventDoubleSpendUnder20ConcurrentWithdrawals(...)`, execution proceeds step-by-step: asynchronous task dispatch delegates execution to the target executor/carrier pool without blocking the caller OS thread.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Important**: A concurrency test **must NOT be annotated with `@Transactional`** on the `@Test` method itself! Each of the 20 worker threads needs to see the initial committed wallet row in its own independent database transaction (and you must clean up the test row manually in `@AfterEach`).

---

<a id="q6-44"></a>
### Q6.44: How do you test Network Partitions, Toxic Latency, and Broker/DB Drops using Testcontainers `Toxiproxy`?

#### 1. Formal Technical Definition & Internal Mechanics
- Standard Testcontainers verify happy-path connectivity, and `container.stop()` only tests clean container shutdown (`TCP FIN`).
- In real production outages, firewalls drop packets silently (**Blackhole / TCP Hang without `FIN`/`RST`**) or inject `2,500ms` latency jitter.
- **Testcontainers `ToxiproxyContainer`** routes traffic between your Spring Boot app and PostgreSQL/Redis/Kafka through a programmable TCP proxy that can inject latency, bandwidth limits, or total packet blackholes mid-test!

#### 2. Technical Architecture & Execution Diagram
```
[Spring Boot App] ──► [Toxiproxy Container (Injects 4s Latency / TCP Blackhole)] ──► [PostgreSQL Container]
  └── Verifies HikariCP connectionTimeout (3s) & Resilience4j CircuitBreaker trip accurately!
```

#### 3. Detailed Technical Example & Production Code
```java
// Injecting a 4,000ms network latency toxic into PostgreSQL TCP stream mid-test:
Toxic latencyToxic = pgProxy.toxics()
    .latency("db-slowdown", ToxicDirection.DOWNSTREAM, 4000);
try {
    assertThatThrownBy(() -> orderService.placeOrder(cmd))
        .isInstanceOf(QueryTimeoutException.class);
} finally {
    latencyToxic.remove(); // Restore normal connectivity and verify auto-recovery!
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `.isInstanceOf(QueryTimeoutException.class);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always test both the **failure transition** (when the toxic is added) and the **recovery transition** (when `toxic.remove()` is called) to verify HikariCP and Kafka clients self-heal without requiring a pod restart.

---

<a id="q6-45"></a>
### Q6.45: How do you perform Load, Stress, Soak, and Spike Testing using Gatling / k6, and why must you beware of the Coordinated Omission Problem?

#### 1. Formal Technical Definition & Internal Mechanics
- **Load Testing Types**:
  - **Load Test**: Steady expected peak traffic (`2,000 RPS` for 30m) to verify SLA (`p99 < 100ms`).
  - **Stress Test**: Ramp traffic until the system breaks to identify the exact bottleneck ceiling.
  - **Spike Test**: Instant jump from `200 RPS` to `10,000 RPS` in 5 seconds (flash sale simulation).
  - **Soak (Endurance) Test**: Sustained load for `12–24 hours` to uncover **memory leaks, connection leaks, or disk/WAL exhaustion**.
- **The Coordinated Omission Problem (Gil Tene)**:
  - If a load generator spawns 100 closed-loop virtual users (where each user waits for request $i$ to finish before sending request $i+1$), and the server freezes for `10 seconds` (e.g., a Stop-the-World GC pause or DB lock), those 100 virtual users only send **100 requests during the entire 10-second freeze** instead of the `10,000 requests` that real users would have sent!
  - Consequently, the load tool reports that 99% of requests were fast (`2ms`) and only 100 requests were slow, **completely hiding the 10-second outage from the `p99` percentile**!
  - **Fix**: Always use **Open Workload Models (`constantUsersPerSec(1000)` in Gatling / `constant-arrival-rate` in k6)** where new requests arrive on schedule regardless of whether previous requests have responded yet!

#### 2. Technical Architecture & Execution Diagram
```
Closed Model (Coordinated Omission Bug!):
  Server freezes for 10s ──► Load tool pauses sending new requests! ──► Fake "Good" p99!

Open Arrival-Rate Model (True Production Simulation):
  Server freezes for 10s ──► Load tool keeps firing 1,000 req/sec ──► Accurately measures queue & p99!
```

#### 3. Detailed Technical Example & Production Code
```java
// Gatling Java DSL using Open Workload Model (Immune to Coordinated Omission):
{
    setUp(
        checkoutScenario.injectOpen(
            rampUsersPerSec(10).to(1000).during(Duration.ofMinutes(2)),
            constantUsersPerSec(1000).during(Duration.ofMinutes(10))
        )
    ).protocols(httpProtocol)
     .assertions(
         global().responseTime().percentile(99.0).lt(150),
         global().successfulRequests().percent().gt(99.9)
     );
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `).protocols(httpProtocol)` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Explaining **Gil Tene's Coordinated Omission Problem** and why Open Arrival-Rate models (`constantUsersPerSec`) must be used over Closed thread-loop models is a signature Staff Engineer insight.

---

<a id="q6-46"></a>
### Q6.46: How do you write JMH (Java Microbenchmark Harness) Benchmarks, and why are `System.currentTimeMillis()` loops invalid for JVM benchmarking?

#### 1. Formal Technical Definition & Internal Mechanics
- Writing a naive `for (int i = 0; i < 1_000_000; i++)` benchmark timed with `System.nanoTime()` produces garbage numbers because of:
  1. **Dead-Code Elimination (DCE)**: If the loop result is never used, the C2 JIT compiler deletes the entire loop body (`0ns` execution time!).
  2. **Constant Folding**: Passing hardcoded literals lets JIT precompute the answer at compile time.
  3. **Lack of C2 JIT Warmup & OSR (On-Stack Replacement)**: The first 10,000 iterations run in slow interpreted/C1 mode.
- **`JMH` (`@Benchmark`, `@State`, `Blackhole`)**: Runs dedicated warmup iterations in forked JVMs, prevents constant folding via `@State` fields, and defeats Dead-Code Elimination by consuming return values into a **`Blackhole`**.

#### 2. Technical Architecture & Execution Diagram
```
Naive Loop:  [Interpreted Mode] ──► [C2 JIT Dead-Code Elimination deletes unused loop!] ──► Invalid!
JMH Harness: [5 Warmup Iterations (C2 JIT Stabilizes)] ──► [5 Measurement Iterations + Blackhole] ──► Accurate!
```

#### 3. Detailed Technical Example & Production Code
```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
@State(Scope.Benchmark)
public class HashingBenchmark {
    private String inputPayload = "order-id-984120-customer-42";

    @Benchmark
    public void sha256Hash(Blackhole bh) throws Exception {
        byte[] digest = MessageDigest.getInstance("SHA-256")
                .digest(inputPayload.getBytes(StandardCharsets.UTF_8));
        bh.consume(digest); // Prevents JIT Dead-Code Elimination!
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `HashingBenchmark` (annotated with `@BenchmarkMode`, `@OutputTimeUnit`, `@Warmup`, `@Measurement`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `sha256Hash(...)`, execution proceeds step-by-step: the core statement `private String inputPayload = "order-id-984120-customer-42";` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always return the computed result from the `@Benchmark` method (or pass it to `Blackhole.consume()`) and read inputs from `@State` fields so the JIT compiler cannot fold constants or eliminate dead code.

---

<a id="q6-47"></a>
### Q6.47: Compare Test Doubles (`Dummy`, `Stub`, `Spy`, `Mock`, `Fake`) and explain when In-Memory `Fake`s beat Mockito `@Mock`s.

#### 1. Formal Technical Definition & Internal Mechanics
- **Gerard Meszaros / Martin Fowler Taxonomy**:
  1. **Dummy**: Passed only to fill a parameter list; never actually used.
  2. **Stub**: Returns canned responses (`when(repo.findById(1L)).thenReturn(...)`). Tests **State Verification**.
  3. **Mock**: Pre-programmed with expectations of exact method calls (`verify(gateway, times(1)).charge(...)`). Tests **Behavior/Interaction Verification**.
  4. **Spy**: Wraps a real object and records calls made to it.
  5. **Fake**: A working lightweight in-memory implementation (`InMemoryOrderRepository` backed by a `ConcurrentHashMap`).
- **Why Over-Mocking Breaks Refactoring**: If a unit test has 15 `when(...)` and `verify(...)` lines tightly coupled to internal helper calls, refactoring internal code breaks 50 tests even though external behavior didn't change! Using a reusable `InMemoryOrderRepository` (**Fake**) tests true state transitions (`save` then `findById`) without brittle mock wiring.

#### 2. Technical Architecture & Execution Diagram
```
Over-Mocked Test (Brittle Interaction Coupling):
  when(repo.findById(1)).thenReturn(a); verify(repo).save(any()); (Breaks on internal refactor!)

State-Based Test with Fake (Resilient & Fast < 1ms):
  FakeOrderRepository (ConcurrentHashMap) ──► service.cancelOrder(1L) ──► assertThat(repo.findById(1L).status).isEqualTo(CANCELLED)
```

#### 3. Detailed Technical Example & Production Code
```java
// Reusable In-Memory Fake for fast <1ms State-Based Domain Service Testing:
public class FakeAccountRepository implements AccountRepositoryPort {
    private final Map<Long, Account> store = new ConcurrentHashMap<>();

    @Override
    public Optional<Account> findById(Long id) { return Optional.ofNullable(store.get(id)); }

    @Override
    public Account save(Account account) {
        store.put(account.getId(), account);
        return account;
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `FakeAccountRepository` (annotated with `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `public Account save(Account account) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Rule of Thumb**: Use **Fakes or State Verification** for repositories/data stores, and reserve **`Mockito.verify()` (Interaction Verification)** for external side-effect gateways (e.g., verifying an email or payment gateway call happened once).

---

<a id="q6-48"></a>
### Q6.48: How do you test Spring Batch Jobs (`JobLauncherTestUtils`, `JobRepositoryTestUtils`, Step-Scoped Tests, and Restartability)?

#### 1. Formal Technical Definition & Internal Mechanics
- `@SpringBatchTest` automatically registers **`JobLauncherTestUtils`** and **`JobRepositoryTestUtils`** in the test context.
- **3 Levels of Spring Batch Testing**:
  1. **End-to-End Job Test (`jobLauncherTestUtils.launchJob(jobParameters)`)**: Verifies all steps transition to `BatchStatus.COMPLETED`.
  2. **Isolated Step Test (`jobLauncherTestUtils.launchStep("settlementStep")`)**: Tests a single step without running 4 preceding steps!
  3. **Component Unit Test (`StepScopeTestExecutionListener`)**: Tests an `@StepScope` `ItemReader` or `ItemProcessor` in isolation with injected `StepExecution` parameters.

#### 2. Technical Architecture & Execution Diagram
```
@SpringBatchTest:
  ├── launchJob(params)           ──► Tests full multi-step Job orchestration
  ├── launchStep("step2", params) ──► Tests Step 2 in isolation
  └── Restartability Test         ──► Inject fail at chunk #3 -> Relaunch -> Verify resumes at chunk #3!
```

#### 3. Detailed Technical Example & Production Code
```java
@SpringBatchTest
@SpringBootTest
class SettlementBatchJobTest extends AbstractIntegrationTest {

    @Autowired private JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired private JobRepositoryTestUtils jobRepositoryTestUtils;

    @BeforeEach
    void cleanupJobExecutions() {
        jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    void shouldCompleteSettlementStepSuccessfully() {
        JobParameters params = new JobParametersBuilder()
            .addLocalDate("settlementDate", LocalDate.of(2025, 2, 20))
            .toJobParameters();

        JobExecution execution = jobLauncherTestUtils.launchStep("dailySettlementStep", params);
        assertThat(execution.getStatus()).isEqualTo(BatchStatus.COMPLETED);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `SettlementBatchJobTest` (annotated with `@SpringBatchTest`, `@SpringBootTest`, `@Autowired`, `@BeforeEach`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `cleanupJobExecutions(...)` and `shouldCompleteSettlementStepSuccessfully(...)`, execution proceeds step-by-step: the core statement `@Test` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Always test Batch Job Idempotency & Restartability**: Simulate a failure on record #150 (with chunk size = 50), verify records 1–100 were committed and 101–150 rolled back, then re-run `launchJob` with the **same `JobParameters`** and verify the job resumes from offset 100 without duplicating records 1–100!

---

<a id="q6-49"></a>
### Q6.49: How do you design Tamper-Evident Security Audit Logging (Cryptographic Hash Chaining & PII Redaction)?

#### 1. Formal Technical Definition & Internal Mechanics
- In regulated financial and healthcare systems (PCI-DSS, SOC2, HIPAA), an attacker or rogue insider with database access could execute `DELETE FROM audit_logs WHERE actor = 'rogue_admin'` or alter an amount to cover their tracks.
- **Cryptographic Hash Chaining (Blockchain-style Ledger)**:
  - Each audit log entry includes `prev_hash` (the SHA-256 hash of row $N-1$) and `entry_hash = HMAC_SHA256(secretKey, prev_hash + "|" + actor + "|" + action + "|" + timestamp + "|" + payload)`.
  - Deleting or modifying any historical row breaks the cryptographic hash chain for all subsequent rows!
  - Additionally, stream audit events in real time to an **append-only WORM (Write-Once-Read-Many) S3 Object Lock bucket** in a separate security AWS account.

#### 2. Technical Architecture & Execution Diagram
```
Audit Row #101: [actor="alice", action="WIRE", prev_hash="0000...", entry_hash="8a9f..."]
                                                                        │
                                  ┌─────────────────────────────────────┘
                                  ▼
Audit Row #102: [actor="bob",   action="APPROVE", prev_hash="8a9f...", entry_hash="3c4d..."]
  └── If anyone alters or deletes Row #101, Row #102's prev_hash verification FAILS immediately!
```

#### 3. Detailed Technical Example & Production Code
```java
public String computeAuditChainHash(
        String hmacSecret, String prevHash, String actor, String action, Instant timestamp, String payload)
        throws Exception {
    String canonical = String.join("|", prevHash, actor, action, timestamp.toString(), payload);
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(hmacSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
    return HexFormat.of().formatHex(mac.doFinal(canonical.getBytes(StandardCharsets.UTF_8)));
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `computeAuditChainHash(...)`, execution proceeds step-by-step: the core statement `Mac mac = Mac.getInstance("HmacSHA256");` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Log Injection (CRLF Injection) Prevention**: Never log raw unsanitized user strings into line-oriented text logs where an attacker can inject `\r\nINFO [main] User admin logged in successfully`. Always output **Structured JSON Logs (`Logback LogstashEncoder`)** where user inputs are JSON-escaped inside dedicated fields.

---

<a id="q6-50"></a>
### Q6.50: Explain SSRF (Server-Side Request Forgery) and Path Traversal vulnerabilities in Java, and how to write airtight defenses.

#### 1. Formal Technical Definition & Internal Mechanics
- **Path Traversal (`../../etc/passwd`)**:
  - If a file download endpoint takes `String filename` and resolves `Paths.get("/var/uploads", filename)`, passing `filename = "../../etc/shadow"` escapes the `/var/uploads` directory!
  - **Fix**: Call `.normalize().toAbsolutePath()` and strictly assert **`resolvedPath.startsWith(baseUploadDir)`**!
- **SSRF (Server-Side Request Forgery)**:
  - If an endpoint fetches a user-supplied URL (e.g., webhook URL, avatar import URL, PDF generator), an attacker can pass `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (AWS EC2/EKS metadata service) or internal VPC URLs (`http://redis.internal:6379`).
  - **Fix**: Parse URI, allow only `https`, resolve the hostname via `InetAddress.getAllByName(host)`, reject any loopback/site-local/link-local IP (`isLoopbackAddress()`, `isSiteLocalAddress()`, `isLinkLocalAddress()`, `isAnyLocalAddress()`), and pin the resolved IP in the HTTP client connection to defeat **DNS Rebinding Attacks**!

#### 2. Technical Architecture & Execution Diagram
```
Path Traversal Defense:
  baseDir = "/var/uploads"
  input   = "../../etc/passwd"
  resolved = baseDir.resolve(input).normalize() ──► "/etc/passwd"
  Check: resolved.startsWith("/var/uploads") == FALSE ──► Throw SecurityException!
```

#### 3. Detailed Technical Example & Production Code
```java
public Path resolveSafeUploadPath(Path baseDir, String untrustedFileName) {
    Path normalizedBase = baseDir.toAbsolutePath().normalize();
    Path resolved = normalizedBase.resolve(untrustedFileName).normalize();
    if (!resolved.startsWith(normalizedBase)) {
        throw new SecurityException("Path traversal attempt blocked: " + untrustedFileName);
    }
    return resolved;
}

public void validateUrlAgainstSsrf(URI uri) throws UnknownHostException {
    if (!"https".equalsIgnoreCase(uri.getScheme())) {
        throw new SecurityException("Only HTTPS URLs are permitted");
    }
    for (InetAddress addr : InetAddress.getAllByName(uri.getHost())) {
        if (addr.isLoopbackAddress() || addr.isSiteLocalAddress()
                || addr.isLinkLocalAddress() || addr.isAnyLocalAddress()
                || addr.isMulticastAddress()) {
            throw new SecurityException("Internal/private IP blocked (SSRF protection): " + addr);
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `validateUrlAgainstSsrf(...)`, execution proceeds step-by-step: the core statement `throw new SecurityException("Only HTTPS URLs are permitted");` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **DNS Rebinding Attack on SSRF**: If you validate `InetAddress.getByName(host)` (`returns public IP 1.2.3.4` with `TTL=0`) and then pass the hostname string to `RestClient`, the second DNS lookup inside `RestClient` 5ms later can return `169.254.169.254`! Always validate the IP inside a custom `DnsResolver` / `SocketFactory` hooked directly into the HTTP client's connection establishment phase, and enforce **IMDSv2 (hop-limit=1)** on AWS EKS nodes.

---

<a id="q6-51"></a>
### Q6.51: How do you secure File Uploads (`MultipartFile`) against Zip Bombs, Polyglot Files, Malicious MIME Spoofing, and XXE?

#### 1. Formal Technical Definition & Internal Mechanics
- **Never trust `MultipartFile.getContentType()` or the filename extension**: Both are plain strings sent by the client HTTP request and can be trivially spoofed (`exploit.jsp` renamed to `avatar.png` with `Content-Type: image/png`).
- **5-Step Production File Upload Hardening**:
  1. **Strict Size Limits**: Enforce `spring.servlet.multipart.max-file-size=10MB` at both the API Gateway and Spring Boot levels.
  2. **Magic-Byte Inspection (Apache Tika)**: Inspect the actual binary magic bytes of the file stream (`89 50 4E 47` for PNG, `25 50 44 46` for PDF).
  3. **Generate Random UUID Storage Key**: Never use `file.getOriginalFilename()` on disk or S3! Always rename to `UUID.randomUUID() + ".pdf"`.
  4. **Direct-to-S3 Presigned URLs**: Offload file uploads from your JVM heap directly to S3 using short-lived S3 Presigned POST policies (with `content-length-range` conditions) and scan asynchronously with ClamAV / GuardDuty before marking the file active.
  5. **Disable XML External Entities (XXE)**: When parsing XML, `.xlsx`, or `.svg`, always enable `XMLConstants.FEATURE_secure_processing` and disallow `DOCTYPE` declarations.

#### 2. Technical Architecture & Execution Diagram
```
Secure Upload Pipeline:
  [Client] ──(1. Request Presigned Upload URL)──► [Spring Boot API]
  [Client] ──(2. Upload directly to S3 Quarantine Bucket)──► [S3 Quarantine]
                                                                  │
                                             (3. Async Malware + Magic Byte Scan)
                                                                  ▼
                                                          [S3 Verified Bucket]
```

#### 3. Detailed Technical Example & Production Code
```java
// Hardening XML DocumentBuilderFactory against XXE (XML External Entity) Injection:
public DocumentBuilderFactory createSecureXmlFactory() throws ParserConfigurationException {
    DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
    dbf.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);
    dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
    dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
    dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
    dbf.setXIncludeAware(false);
    dbf.setExpandEntityReferences(false);
    return dbf;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", f` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Serving User Uploads from a Separate Sandbox Domain**: Always serve user-uploaded files from a separate cookieless domain (`https://acme-usercontent.com` instead of `https://app.acme.com`) with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` so an uploaded SVG/HTML file can never execute XSS in your main application origin.

---

<a id="q6-52"></a>
### Q6.52: Production Incident: After deploying a new microservice release, 5% of authenticated users randomly receive `401 Unauthorized` or see *another user's profile data*! Walk through your emergency response and root-cause analysis.

#### 1. Formal Technical Definition & Internal Mechanics
- Seeing **another user's data** in a Spring Boot application is a **Severity-0 (P0) Cross-User Data Leak Incident**.
- **Top 3 Root Causes of Cross-User Data Leakage / Random `401`s**:
  1. **Stateful Mutable Field in a Singleton `@Controller` / `@Service` / `Filter`**: A developer stored `private String currentUserId;` or `private Jwt currentJwt;` as an **instance field** on a singleton Spring bean! Under concurrency, Thread B overwrites `this.currentUserId` while Thread A is mid-execution!
  2. **Custom `ThreadLocal` Without `finally { threadLocal.remove(); }`**: A custom `UserContextHolder` `ThreadLocal` was populated in an interceptor/filter (or `@Async` worker thread) without a `finally` block clearing it, leaking user state across pooled Tomcat/worker threads.
  3. **Shared HTTP Cache / CDN Caching Authenticated Responses**: An API endpoint or CDN was configured without `Cache-Control: private, no-store` (or missed `Vary: Authorization`), causing Cloudflare/Nginx to cache User A's `GET /api/v1/profile` response and serve it to User B!
  4. **For Random `401`s specifically**: Clock skew (`> 60s`) between pods rejecting newly minted JWT `nbf`/`iat` claims, or a rolling key rotation where new pods cached an old JWKS set.

#### 2. Technical Architecture & Execution Diagram
```
P0 Cross-User Data Leak Triage:
  1. IMMEDIATE MITIGATION (< 2 mins):
     └── Roll back v2 deployment immediately & purge CDN/Edge cache!
  2. ROOT CAUSE AUDIT:
     ├── Check git diff for instance fields in @Service / @RestController / @Component
     ├── Check git diff for ThreadLocal.set() missing finally { ThreadLocal.remove(); }
     ├── Check Redis @Cacheable keys missing #authentication.name (e.g. @Cacheable("profile") with 0 args!)
     └── Check HTTP Response Headers (Ensure Cache-Control: no-store on all authenticated APIs)
```

#### 3. Detailed Technical Example & Production Code
```java
// Classic Bug #3: @Cacheable on a zero-argument method (Caches User A's profile under SimpleKey.EMPTY
// and returns User A's profile to EVERY subsequent user!):
// BROKEN:
// @Cacheable("userProfile")
// public UserProfile getMyProfile() { ... }

// FIXED (Explicitly key cache by authenticated principal subject ID!):
@Cacheable(cacheNames = "userProfile", key = "#jwt.subject")
public UserProfile getMyProfile(@AuthenticationPrincipal Jwt jwt) {
    return userRepository.findProfileBySub(jwt.getSubject());
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (annotated with `@Cacheable`, `@AuthenticationPrincipal`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **The Zero-Arg `@Cacheable` P0 Bug**: Pointing out that `@Cacheable("myProfile") public Profile getMyProfile()` (where the user ID is read inside the method from `SecurityContextHolder` instead of passed as a method parameter!) caches the first user's profile under `SimpleKey.EMPTY` and serves it to **every user on the platform** is one of the most impressive real-world production war stories you can share in a Senior Java interview!

---

[← Back to Main README](../README.md)
