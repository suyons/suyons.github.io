---
title: "Moving Spring Boot from Inline Datasources to JNDI for Enterprise WAS Deployments"
date: 2025-04-23
draft: false
tags: ["springboot", "jndi", "datasource", "enterprise", "was", "deployment"]
categories: ["Development"]
description: "In enterprise environments, the application server — not the application — manages database connections. Here is how to move a Spring Boot app from hardcoded connection strings to JNDI-bound datasources, and the gotchas that trip people up during the transition."
showToc: true
---

## Why JNDI Datasources?

When deploying to a corporate WAS (Web Application Server) like LENA, JEUS, or WebLogic, the standard expectation is that connection pools are managed at the container level. The DBA team creates connection pools in the WAS admin console, binds them to a JNDI name, and the application looks up that name at runtime. The application itself never holds a connection string, username, or password.

The benefits are operational:

- Credentials are stored once in the WAS configuration, not in every application's `application.yml`.
- The DBA or operations team can rotate credentials, adjust pool sizes, or switch the target database without touching application code or triggering a redeploy.
- Connection pool metrics are visible in the WAS admin console.

The downside: local development, where there is no corporate WAS, needs a different datasource configuration.

---

## The Before State

A typical Spring Boot development config:

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:oracle:thin:@localhost:1521/ORCL
    username: appuser
    password: apppassword
    driver-class-name: oracle.jdbc.OracleDriver
```

This works perfectly in development. It breaks in the enterprise WAS environment because the WAS-managed connection pool supersedes any Spring-configured pool, and Spring Boot cannot wire up a `DataSource` bean from properties that the WAS does not expose.

---

## The After State

### Step 1: Remove datasource properties from `application-prod.yml`

Delete everything under `spring.datasource` for the production profile. The WAS will inject the datasource at runtime.

### Step 2: Create a `DataSourceConfig` class that uses JNDI lookup

```java
@Configuration
@Profile("prod")
public class DataSourceConfig {

    @Bean(name = "primaryDataSource")
    @Primary
    public DataSource primaryDataSource() throws NamingException {
        JndiDataSourceLookup lookup = new JndiDataSourceLookup();
        return lookup.getDataSource("java:comp/env/jdbc/primaryDS");
    }

    @Bean(name = "secondaryDataSource")
    public DataSource secondaryDataSource() throws NamingException {
        JndiDataSourceLookup lookup = new JndiDataSourceLookup();
        return lookup.getDataSource("java:comp/env/jdbc/secondaryDS");
    }

    @Bean(name = "legacyDataSource")
    public DataSource legacyDataSource() throws NamingException {
        JndiDataSourceLookup lookup = new JndiDataSourceLookup();
        return lookup.getDataSource("java:comp/env/jdbc/legacyDS");
    }
}
```

Replace `java:comp/env/jdbc/primaryDS` with the JNDI name registered in the WAS admin console. The WAS administrator creates a resource reference in the WAS, binds it to the application by its JNDI name, and the Spring context resolves it at startup.

For a single-datasource application using the Spring Boot auto-configured `DataSource`, you can also set the JNDI name in properties without writing a config class:

```yaml
# application-prod.yml
spring:
  datasource:
    jndi-name: java:comp/env/jdbc/primaryDS
```

### Step 3: Keep the dev profile using inline config

```java
@Configuration
@Profile("dev")
public class DevDataSourceConfig {

    @Bean
    @Primary
    public DataSource primaryDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:oracle:thin:@localhost:1521/ORCL");
        ds.setUsername("appuser");
        ds.setPassword("apppassword");
        return ds;
    }
}
```

The `@Profile("dev")` and `@Profile("prod")` annotations ensure only one configuration activates. Set `SPRING_PROFILES_ACTIVE=prod` in the WAS startup script (`setenv.sh` or equivalent) and `SPRING_PROFILES_ACTIVE=dev` in your local IDE run configuration.

---

## Deployment Artifact Considerations

### The `.yml` file extension problem

Some corporate document management and file transfer systems (DMZ gateways, EDMS solutions) reject `.yml` and `.yaml` files by extension because they are on a blocklist of file types considered "configuration" or "code."

Workaround: rename `application-prod.yml` to `application-prod.txt` for transit, then rename it back on the target server before deployment. Ugly, but common.

### Include `pom.xml` when dependencies change

When the datasource migration adds a new dependency — for example, if JNDI lookup requires a version bump of the Oracle JDBC driver — include `pom.xml` in the deployment package. Otherwise the WAS classpath will not match the build and you will get `ClassNotFoundException` at runtime.

---

## Multiple Datasources: MyBatis Mapper Routing

When the application has multiple datasources (primary, secondary, legacy), MyBatis mappers need to know which `SqlSessionFactory` to use. The `@MapperScan` annotation on each config class routes mapper interfaces by package:

```java
@Configuration
@Profile("prod")
@MapperScan(
    basePackages = "com.example.mapper.primary",
    sqlSessionFactoryRef = "primarySqlSessionFactory"
)
public class PrimaryDataSourceConfig {

    @Bean
    public DataSource primaryDataSource() throws NamingException {
        return new JndiDataSourceLookup().getDataSource("java:comp/env/jdbc/primaryDS");
    }

    @Bean
    public SqlSessionFactory primarySqlSessionFactory() throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(primaryDataSource());
        factory.setMapperLocations(
            new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/primary/*.xml")
        );
        return factory.getObject();
    }
}
```

Repeat for each datasource with a different `basePackages` and `sqlSessionFactoryRef`.

---

## Verifying the JNDI Lookup at Startup

If the JNDI name is wrong or the WAS binding is missing, the application will fail to start with:

```
javax.naming.NameNotFoundException: java:comp/env/jdbc/primaryDS
```

Before deploying to the WAS, confirm the binding with the WAS admin console or, if available, with a JNDI browser tool. The error is cryptic but the fix is always the same: either the JNDI name in the code does not match the name registered in the WAS, or the resource reference was not associated with the application deployment.

---

## Summary

Moving from inline datasource properties to JNDI involves:

1. Deleting datasource connection strings from the production application config.
2. Adding a `DataSourceConfig` class (or `jndi-name` property) that performs the JNDI lookup.
3. Keeping the development config as-is under a separate Spring profile.
4. Routing MyBatis mappers to the correct `SqlSessionFactory` when multiple datasources exist.

The result is an application binary that contains no credentials, where all connection management is delegated to the WAS container — the standard expectation for enterprise WAS deployments.
