---
title: "Implementing SSO Integration and Session Authentication in Spring Boot"
date: 2025-07-03
draft: false
tags: ["springboot", "security", "sso", "authentication", "java"]
categories: ["DevOps"]
description: "How SSO integration with a token-based system works in a Spring Boot WAS: token validation via a shared JAR, employee lookup, Spring Security configuration, and common failure modes in a closed-network enterprise environment."
showToc: true
---

## What SSO Looks Like in Enterprise

Modern consumer apps use OAuth 2.0 or OpenID Connect. Enterprise systems at Korean financial institutions often use a different model: a proprietary SSO system that issues tokens in a custom format, distributed as a JAR library that the application must embed.

The flow looks like this:

1. The user logs into the organization's portal, which issues an SSO token.
2. The portal redirects the user to your application, passing the token as a request parameter or cookie.
3. Your application calls the SSO JAR's validation method with the token.
4. The SSO JAR contacts the SSO server (or validates locally using a cached key) and returns the user's employee ID and other attributes.
5. Your application uses the employee ID to look up the user's roles and permissions.

This is not OAuth. There is no redirect flow you control, no refresh token, and the token format is proprietary. But the end result — a validated identity with a session — is similar.

---

## Integrating the SSO JAR

The SSO library is distributed as a JAR and is not in any public Maven repository. Install it into your local Maven repository manually:

```bash
mvn install:install-file \
  -Dfile=sso-client-3.2.1.jar \
  -DgroupId=com.example.sso \
  -DartifactId=sso-client \
  -Dversion=3.2.1 \
  -Dpackaging=jar
```

Then reference it in `pom.xml`:

```xml
<dependency>
    <groupId>com.example.sso</groupId>
    <artifactId>sso-client</artifactId>
    <version>3.2.1</version>
</dependency>
```

For a team working in a closed network, put the JAR in the internal Maven repository (Nexus, Artifactory, or a shared network drive configured as a local repository). This avoids every developer having to run the install command manually.

---

## The Token Validation Controller

Build a dedicated endpoint that accepts the SSO token and returns employee information:

```java
@RestController
@RequestMapping("/api/auth")
public class SsoAuthController {

    private final SsoTokenValidator ssoValidator;
    private final EmployeeService employeeService;

    public SsoAuthController(SsoTokenValidator ssoValidator, EmployeeService employeeService) {
        this.ssoValidator = ssoValidator;
        this.employeeService = employeeService;
    }

    @PostMapping("/sso")
    public ResponseEntity<EmployeeInfo> validateSsoToken(@RequestParam String token) {
        SsoResult result = ssoValidator.validate(token);

        if (!result.isValid()) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        String employeeId = result.getEmployeeId();
        EmployeeInfo employee = employeeService.findByEmployeeId(employeeId);

        if (employee == null) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }

        return ResponseEntity.ok(employee);
    }
}
```

Notice: the controller does not build a session here — that belongs in the filter layer. The controller's job is to validate the token and return identity data.

---

## SSO Validation in a Spring Security Filter

For requests that carry an SSO token, validate it in a `OncePerRequestFilter` and set the `SecurityContextHolder`:

```java
@Component
public class SsoAuthenticationFilter extends OncePerRequestFilter {

    private final SsoTokenValidator ssoValidator;
    private final EmployeeService employeeService;

    public SsoAuthenticationFilter(SsoTokenValidator ssoValidator,
                                   EmployeeService employeeService) {
        this.ssoValidator = ssoValidator;
        this.employeeService = employeeService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null) {
            SsoResult result = ssoValidator.validate(token);
            if (result.isValid()) {
                EmployeeInfo employee = employeeService.findByEmployeeId(result.getEmployeeId());
                if (employee != null) {
                    UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(
                            employee,
                            null,
                            employee.getAuthorities()
                        );
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String param = request.getParameter("ssoToken");
        if (param != null) return param;

        Cookie[] cookies = request.getCookies();
        if (cookies == null) return null;
        for (Cookie cookie : cookies) {
            if ("SSO_TOKEN".equals(cookie.getName())) return cookie.getValue();
        }
        return null;
    }
}
```

Register the filter before Spring Security's `UsernamePasswordAuthenticationFilter`:

```java
// Spring Boot 2.x / Spring Security 5.x
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .addFilterBefore(ssoAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
        .authorizeRequests()
            .antMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        .and()
        .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
}
```

---

## Developing Without the SSO Server

The SSO server is only available inside the organization's network. During early development, stub the validator:

```java
@Component
@Profile("dev")
public class MockSsoTokenValidator implements SsoTokenValidator {

    private static final Map<String, String> TOKEN_MAP = Map.of(
        "dev-token-admin",   "E1000001",
        "dev-token-viewer",  "E1000002"
    );

    @Override
    public SsoResult validate(String token) {
        String employeeId = TOKEN_MAP.get(token);
        if (employeeId != null) {
            return SsoResult.valid(employeeId);
        }
        return SsoResult.invalid();
    }
}
```

The `@Profile("dev")` annotation ensures this mock is only active in the dev profile. The real implementation is active in `test` and `prod` profiles.

This lets you develop and test the entire authentication flow locally without needing access to the SSO server, which may require a network configuration change to reach from your laptop.

---

## Employee Lookup and Role Mapping

After the SSO token yields an employee ID, you need to determine what the employee is allowed to do. Map this in the database:

```sql
CREATE TABLE employees (
    employee_id     VARCHAR2(20)  NOT NULL,
    display_name    VARCHAR2(100),
    org_code        VARCHAR2(10),
    role            VARCHAR2(20)  NOT NULL,  -- 'ADMIN', 'VIEWER', 'OPERATOR'
    active          CHAR(1)       DEFAULT 'Y' NOT NULL,
    PRIMARY KEY (employee_id)
);
```

Validation query — run this after importing employee data from the organization directory:

```sql
-- Check for employees with no role assignment
SELECT employee_id, display_name
FROM employees
WHERE role IS NULL OR role NOT IN ('ADMIN', 'VIEWER', 'OPERATOR');

-- Check for duplicate employee IDs (should be zero)
SELECT employee_id, COUNT(*)
FROM employees
GROUP BY employee_id
HAVING COUNT(*) > 1;

-- Verify active employees can be looked up by org
SELECT e.org_code, COUNT(*) AS headcount
FROM employees e
WHERE e.active = 'Y'
GROUP BY e.org_code
ORDER BY headcount DESC;
```

These queries run against the real database after each import cycle. Silent data quality problems (missing roles, duplicate records, inactive accounts that should be flagged) surface here before they cause a 403 at runtime.

---

## Common Failure Modes

### SSO JAR not found at runtime

If the JAR was installed in your local Maven repository but not in the team's shared repository, a fresh build on another machine will fail. Always check that the internal Maven repo has the artifact before announcing a successful build.

### Token from cookie vs. parameter

The SSO server may send the token as a URL parameter on the redirect, but the browser subsequently sends it as a cookie. Your extraction logic needs to handle both. If the filter only checks parameters, subsequent requests after the initial redirect will be unauthenticated.

### SecurityContextHolder cleared between threads

If you use `@Async` methods that run on a different thread pool, the `SecurityContext` is not automatically propagated. Configure `DelegatingSecurityContextAsyncTaskExecutor`:

```java
@Bean
public TaskExecutor securityAwareExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.initialize();
    return new DelegatingSecurityContextAsyncTaskExecutor(executor);
}
```

### SSO server unreachable in test environment

The test environment network segment may not have a route to the SSO server. Confirm the route exists before assuming the code is broken. A simple `curl` from the test server to the SSO server's validation endpoint narrows the failure to a network configuration issue rather than a code bug.

---

## Summary

SSO integration in a financial enterprise environment is less about OAuth flows and more about embedding a library, extracting a token from a request, and mapping the resulting employee ID to your application's permission model. The code is not complex; the complexity is in the coordination (getting the JAR, confirming the network route, handling both initial redirect and subsequent requests) and in validating the employee data in the database matches what the SSO system returns.
