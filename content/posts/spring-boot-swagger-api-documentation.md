---
title: "Spring Boot API Documentation with Swagger: Version Gotchas and Best Practices"
date: 2025-06-05
draft: false
tags: ["springboot", "swagger", "openapi", "java", "documentation"]
categories: ["DevOps"]
description: "A practical guide to integrating Swagger (springdoc-openapi) into a Spring Boot project, including the version compatibility matrix between Spring Boot 2.x and 3.x, common configuration errors, and tips for generating useful API documentation."
showToc: true
---

## Why API Documentation Matters in Multi-Team Projects

In a project where your WAS exposes endpoints consumed by multiple upstream and downstream systems — a document management system, a core banking ESB, a front-end admin dashboard — a clean Swagger UI is not a nice-to-have. It is the primary reference the integration teams use to write their test scripts and stub your service.

The alternative is a Word document that goes stale in week two. Swagger keeps the spec in sync with the code because it is generated from the code.

---

## Choosing the Right Library Version

The most common mistake when adding Swagger to a Spring Boot project is picking the wrong version of `springdoc-openapi`. The library has two major branches:

| Spring Boot version | springdoc-openapi artifact | Version |
|---------------------|---------------------------|---------|
| 3.x (Spring Framework 6.x) | `springdoc-openapi-starter-webmvc-ui` | 2.x |
| 2.x (Spring Framework 5.x) | `springdoc-openapi-ui` | 1.x |

This distinction matters because Spring Boot 3.x migrated from the `javax` namespace to `jakarta`. The `springdoc-openapi` 2.x branch was rewritten to target `jakarta`; the 1.x branch targets `javax`. Using the wrong branch produces a `ClassNotFoundException` at startup.

### Spring Boot 3.x

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.3</version>
</dependency>
```

### Spring Boot 2.x (JDK 8 or 11 target)

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.8.0</version>
</dependency>
```

After adding the dependency, the Swagger UI is available at `http://localhost:8080/swagger-ui.html` (or `/swagger-ui/index.html` on newer versions).

---

## Annotation Package Migration

Between springdoc 1.x and 2.x, some annotation packages moved:

```java
// springdoc 1.x
import org.springdoc.api.annotations.ParameterObject;

// springdoc 2.x
import org.springdoc.core.annotations.ParameterObject;
```

If you downgrade from Spring Boot 3.x to 2.x (a common requirement when the production server runs JDK 8), you need to update these imports in reverse. An IDE global search for `org.springdoc.core.annotations` → `org.springdoc.api.annotations` covers most cases.

---

## Basic Configuration

### OpenAPI Info Bean

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Case Management API")
                .version("1.0")
                .description("Internal API for case update processing"));
    }
}
```

### Grouping Endpoints by Controller

For large projects with many controllers, group endpoints by functional area:

```java
@Bean
public GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
        .group("admin")
        .pathsToMatch("/api/admin/**")
        .build();
}

@Bean
public GroupedOpenApi integrationApi() {
    return GroupedOpenApi.builder()
        .group("integration")
        .pathsToMatch("/api/integration/**")
        .build();
}
```

This creates separate Swagger groups in the UI dropdown, which helps when different teams consume different endpoint sets.

---

## Documenting Endpoints

### Controller-Level Annotations

```java
@RestController
@RequestMapping("/api/case-updates")
@Tag(name = "Case Updates", description = "Receives and processes case update events from ESB")
public class CaseUpdateController {

    @PostMapping
    @Operation(
        summary = "Receive case update",
        description = "Accepts a case update message from the ESB and persists it"
    )
    @ApiResponse(responseCode = "200", description = "Successfully processed")
    @ApiResponse(responseCode = "400", description = "Invalid message format")
    public ResponseEntity<Void> receiveCaseUpdate(@RequestBody CaseUpdateMessage message) {
        // ...
    }
}
```

### Documenting Request Models

```java
@Schema(description = "Case update message received from ESB")
public class CaseUpdateMessage {

    @Schema(description = "Unique case identifier", example = "C20250327001", required = true)
    private String caseId;

    @Schema(description = "Incident classification code", example = "AUTO")
    private String incidentType;

    @Schema(description = "Fault rate as a decimal string", example = "0.35")
    private String faultRateRaw;

    @Schema(description = "Submission status: A0=draft, A1=final", example = "A1")
    private String status;
}
```

The `example` field in `@Schema` is rendered directly in the Swagger UI's "Try it out" feature. Providing realistic examples here saves integration teams from having to guess valid test values.

---

## Handling Spring Security Conflicts

When Spring Security is active, it blocks the Swagger UI paths by default. Add an explicit permit rule:

```java
// Spring Boot 2.x (Spring Security 5.x style)
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers(
            "/swagger-ui.html",
            "/swagger-ui/**",
            "/v3/api-docs/**",
            "/swagger-resources/**",
            "/webjars/**"
        ).permitAll()
        .anyRequest().authenticated();
}
```

```java
// Spring Boot 3.x (Spring Security 6.x style)
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers(
            "/swagger-ui.html",
            "/swagger-ui/**",
            "/v3/api-docs/**"
        ).permitAll()
        .anyRequest().authenticated()
    );
    return http.build();
}
```

If you are using a custom JWT or SSO filter, make sure these paths are excluded from token validation as well — it is a common oversight to add the Spring Security permit rule but forget the custom filter.

---

## Disabling Swagger in Production

Swagger should not be accessible in a production environment. Options:

**Option 1: application profile conditional**

```yaml
# application-prod.yaml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

**Option 2: Conditional bean**

```java
@Configuration
@Profile("!prod")
public class OpenApiConfig {
    @Bean
    public OpenAPI openAPI() { ... }
}
```

For financial projects where the production network is segregated, this is belt-and-suspenders — the UI would not be reachable from outside the internal network anyway — but it is still good hygiene.

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Swagger UI returns 404 | Wrong artifact (1.x vs 2.x branch) | Match the artifact version to your Spring Boot version |
| `ClassNotFoundException: jakarta.servlet...` | Using springdoc 2.x with Spring Boot 2.x | Downgrade to springdoc 1.x |
| `ClassNotFoundException: javax.servlet...` | Using springdoc 1.x with Spring Boot 3.x | Upgrade to springdoc 2.x |
| Swagger UI accessible but returns 401 | Spring Security blocking `/swagger-ui/**` | Add permit rules for all Swagger paths |
| `ParameterObject` import not found | Package moved between 1.x and 2.x | Update import to correct package |

---

## Summary

Swagger integration in Spring Boot is three lines of `pom.xml` when the versions align. When they do not, the errors are confusing because they appear at startup and look like classpath issues rather than version mismatches. The version table at the top of this post is the single most useful reference for avoiding that confusion. Everything else — annotation placement, security exclusions, profile-based disabling — is incremental improvement on top of a working base.
