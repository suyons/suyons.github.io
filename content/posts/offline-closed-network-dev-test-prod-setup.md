---
title: "Setting Up an Air-Gapped Work Environment: Dev, Test, and Production"
date: 2025-04-10
draft: false
tags: ["devops", "java", "springboot", "tomcat", "deployment"]
categories: ["DevOps"]
description: "How to build and manage a non-resident, air-gapped development environment across three tiers — dev, test, and production — in a financial IT project. Covers JDK downgrade, WAR deployment, and environment-specific integration constraints."
showToc: true
---

## What Air-Gapped Means in Practice

Financial institutions in Korea operate under strict network segregation rules. The internal network is physically isolated from the internet — no `apt`, no `pip`, no direct GitHub pulls. This is what practitioners call a **폐쇄망** (closed network), and it fundamentally changes how you set up and maintain a development environment.

You do not SSH into the server and run `git pull`. You build a WAR file locally, hand it to someone who physically carries it to the server room, and wait while they deploy it. Or if you are lucky, you have a shared file server with a controlled transfer channel. Either way, feedback loops are long.

This post covers how I structured the work environment for a Spring Boot WAS project under these constraints, and the JDK downgrade that triggered a cascade of dependency changes.

---

## Three-Tier Environment Strategy

The project had three named environments, and not every integrated system was available in all three:

| System | Dev | Test | Prod |
|--------|-----|------|------|
| Core banking | No | Yes | Yes |
| Document management | No | Yes | Yes |
| Organization directory | Yes | No | Yes |

This asymmetry forced a clear rule: **dev is for unit-level logic only; test is where real integration starts**. Any feature that touched core banking or document management could not be verified locally. You built the stub locally, deployed to test, and checked logs.

The practical implication: every deployment to test had to be close to correct before it left your hands, because a round-trip through the physical transfer process cost at least half a day.

---

## JDK Downgrade: 17 to 8

The production server runs JDK 8. I initially built on JDK 17 to use modern APIs, then had to backport. The changes were mechanical but spread across multiple layers.

### Spring Boot Version

```xml
<!-- Before -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.1</version>
</parent>

<!-- After -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
</parent>
```

### MyBatis Starter

```xml
<!-- Before -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.4</version>
</dependency>

<!-- After -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.3.2</version>
</dependency>
```

### Swagger (springdoc-openapi)

```xml
<!-- Before (Spring Boot 3.x) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.3</version>
</dependency>

<!-- After (Spring Boot 2.x / JDK 8) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.8.0</version>
</dependency>
```

### Namespace Migration: jakarta → javax

Spring Boot 3.x uses the Jakarta EE 9+ namespace; 2.x uses the old javax namespace. Every servlet-related import needed updating:

```java
// Before
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.FilterChain;

// After
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.FilterChain;
```

### springdoc Annotation Package

```java
// Before
import org.springdoc.core.annotations.ParameterObject;

// After
import org.springdoc.api.annotations.ParameterObject;
```

### HttpStatusCode → HttpStatus

Spring Framework 6.x introduced `HttpStatusCode` as an interface; 5.x used `HttpStatus` directly:

```java
// Before (Spring Boot 3.x / Framework 6.x)
import org.springframework.http.HttpStatusCode;

// After (Spring Boot 2.x / Framework 5.x)
import org.springframework.http.HttpStatus;
```

### WebSecurityConfig Lambda Syntax

Spring Security 6.x introduced a fluent lambda-only API that is not fully backward compatible with Spring Security 5.x's method chaining style. The config had to be reverted to the older form:

```java
// Before (Spring Security 6.x)
http
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/public/**").permitAll()
        .anyRequest().authenticated()
    )
    .sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
    );

// After (Spring Security 5.x)
http
    .authorizeRequests()
        .antMatchers("/api/public/**").permitAll()
        .anyRequest().authenticated()
    .and()
    .sessionManagement()
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

Also note: `NoResourceFoundException` from Spring MVC 6.x does not exist in 5.x — any handler referencing it needs to be removed or replaced with a generic exception handler.

---

## WAR Build and Deployment Flow

The application server runs Apache Tomcat (or a compatible servlet container). Deployment means dropping a WAR file into the `webapps/` directory.

### Packaging as WAR

In `pom.xml`:

```xml
<packaging>war</packaging>
```

Exclude the embedded Tomcat:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

Extend `SpringBootServletInitializer`:

```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Build:

```bash
mvn clean package -DskipTests
```

The output is `target/app-0.0.1-SNAPSHOT.war`. Rename it to match the desired context path (`ROOT.war` for root deployment) before transfer.

---

## Environment-Specific Configuration

Each environment needed different database URLs, ESB endpoints, and feature flags. Rather than a single `application.properties`, the project used Spring profiles:

```
application.properties          # shared defaults
application-dev.properties      # dev overrides
application-test.properties     # test overrides
application-prod.properties     # prod overrides
```

Activate the profile by passing it to Tomcat's JVM options in `catalina.sh` (or `setenv.sh` on the server):

```bash
JAVA_OPTS="-Dspring.profiles.active=test"
```

This keeps environment-specific values out of the WAR file and avoids hard-coding server addresses that would expose network topology in source control.

---

## Lessons

Working in a closed network removes the feedback shortcuts most developers take for granted. The compensating discipline is this: **verify locally what you can, document what you cannot, and build deployment artifacts that are deterministic**. If the WAR behaves differently in test than in dev, the culprit is almost always a missing environment variable, a profile not being activated, or a dependency that resolved differently because the artifact repository inside the closed network is out of sync with Maven Central.

The JDK downgrade showed how far the Java ecosystem has moved. A two-version jump (JDK 8 → 17) in Spring Boot meant touching build files, imports, Spring Security configuration, and Swagger packages simultaneously. Running `mvn clean package` in CI before handing off to the deployment team caught most of these early. The ones that slipped through were always in Spring Security configuration, which fails at runtime, not compile time.
