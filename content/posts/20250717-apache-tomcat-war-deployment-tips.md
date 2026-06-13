---
title: "Apache Tomcat Deep Dive: WAR Deployment, Tuning, and Troubleshooting"
date: 2025-07-17
draft: false
tags: ["tomcat", "java", "springboot", "deployment", "jvm", "was"]
categories: ["DevOps"]
description: "A practical guide to Apache Tomcat for engineers who are deploying Spring Boot WAR files in enterprise environments: directory structure, connector configuration, JVM memory tuning, log analysis, and common deployment failures."
showToc: true
---

## Why Tomcat Still Matters

Spring Boot ships with an embedded Tomcat, which is perfect for self-contained JAR deployments. But in financial enterprise environments, the application server is often managed separately by a server team, and the deployment artifact is expected to be a WAR file dropped into an existing Tomcat installation. The embedded Tomcat is excluded from the build, and you are working with a standalone Tomcat managed by someone else.

Understanding how that Tomcat works — its directory layout, how it loads WARs, how to tune its thread pool, and how to read its logs — is essential when your application behaves differently in the server environment than it does locally.

---

## Directory Structure

A default Tomcat installation looks like this:

```
$CATALINA_HOME/
├── bin/
│   ├── startup.sh       # Start the server
│   ├── shutdown.sh      # Stop the server
│   └── setenv.sh        # JVM options (create this file if it doesn't exist)
├── conf/
│   ├── server.xml       # Connector ports, thread pool, host config
│   ├── context.xml      # Data source definitions, session config
│   └── web.xml          # Default servlet, MIME types
├── logs/
│   ├── catalina.out     # Main log (stdout + stderr merged)
│   ├── localhost.YYYY-MM-DD.log  # Per-application error log
│   └── access_log.YYYY-MM-DD.txt # HTTP access log
├── webapps/
│   ├── ROOT/            # Default application (context path /)
│   └── yourapp.war      # Deployed WAR
└── work/                # JSP compilation cache, temp files
```

Key files to know when troubleshooting:
- `catalina.out` — startup errors, uncaught exceptions, JVM crashes
- `localhost.YYYY-MM-DD.log` — application-level errors including Spring initialization failures
- `access_log` — HTTP request/response status codes and response times

---

## WAR Deployment

Drop the WAR file into `webapps/`:

```bash
cp target/app-0.0.1-SNAPSHOT.war $CATALINA_HOME/webapps/app.war
```

Tomcat's auto-deployment is enabled by default. It detects the new WAR, expands it, and deploys it. The context path becomes `/app`. For root deployment:

```bash
cp target/app.war $CATALINA_HOME/webapps/ROOT.war
```

This replaces the default ROOT application and makes your app available at `/`.

### Forcing a Clean Redeploy

Tomcat reuses the expanded directory if it exists, which can cause stale class loading:

```bash
# Stop Tomcat
$CATALINA_HOME/bin/shutdown.sh

# Remove the expanded directory and cached work
rm -rf $CATALINA_HOME/webapps/app
rm -rf $CATALINA_HOME/work/Catalina/localhost/app

# Copy the new WAR
cp target/app.war $CATALINA_HOME/webapps/

# Start Tomcat
$CATALINA_HOME/bin/startup.sh
```

If you see old code behavior after a redeploy, the first thing to check is whether the old expanded directory was cleaned up.

---

## JVM Options via setenv.sh

Create `$CATALINA_HOME/bin/setenv.sh` to set JVM options without modifying `startup.sh` (which gets overwritten on Tomcat upgrades):

```bash
#!/bin/bash
JAVA_OPTS="-server"
JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx4096m"
JAVA_OPTS="$JAVA_OPTS -XX:+UseParallelGC"
JAVA_OPTS="$JAVA_OPTS -XX:GCTimeRatio=4"
JAVA_OPTS="$JAVA_OPTS -Dspring.profiles.active=test"
JAVA_OPTS="$JAVA_OPTS -Dfile.encoding=UTF-8"
export JAVA_OPTS
```

Key options explained:

| Option | Purpose |
|--------|---------|
| `-Xms` | Initial heap size — set equal to `-Xmx` to avoid heap resizing overhead at startup |
| `-Xmx` | Maximum heap size — size according to workload; see the OOM post for guidance |
| `-XX:+UseParallelGC` | Throughput-oriented GC, appropriate for batch-heavy workloads |
| `-XX:GCTimeRatio=4` | Tells the GC to target spending ≤20% of time on collection (1/(1+4)) |
| `-Dspring.profiles.active` | Spring profile to activate |

Make the file executable:

```bash
chmod +x $CATALINA_HOME/bin/setenv.sh
```

---

## Connector Configuration (server.xml)

The HTTP connector configuration lives in `$CATALINA_HOME/conf/server.xml`:

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxThreads="200"
           minSpareThreads="10"
           acceptCount="100"
           compression="on"
           compressionMinSize="2048" />
```

| Attribute | Description |
|-----------|-------------|
| `maxThreads` | Maximum worker threads; requests beyond this queue up |
| `minSpareThreads` | Threads kept alive when idle |
| `acceptCount` | Queue depth when all threads are busy; beyond this, connections are refused |
| `connectionTimeout` | Time to wait for the client to send the request headers |

For a WAS handling ESB-driven integration (short, frequent requests) rather than a user-facing web app:
- `maxThreads`: 100–200 is usually sufficient; tune based on load test results
- `acceptCount`: keep it low (50–100) to fail fast rather than queue indefinitely when overloaded

---

## Reading catalina.out for Startup Failures

Spring Boot's `ApplicationContext` failure manifests in `catalina.out`, not in `localhost.YYYY-MM-DD.log`. Look for:

```
SEVERE: Context [] startup failed due to previous errors
```

Scroll up from this line to find the root cause. Common patterns:

**Bean creation failure** — usually a missing dependency, misconfigured `@Value`, or a database connection that fails at startup:
```
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException:
Error creating bean with name 'caseUpdateController'
```

**Profile-specific property missing**:
```
Caused by: java.lang.IllegalArgumentException:
Could not resolve placeholder 'datasource.url' in value "${datasource.url}"
```

Fix: check that the active profile's `application-{profile}.properties` file exists and contains the key.

**Port already in use**:
```
Address already in use: bind
```

Fix: `lsof -i :8080` to find what is holding the port, or change the connector port in `server.xml`.

---

## Log Rotation and Disk Management

On a long-running server, `catalina.out` grows indefinitely. It is not rotated by Tomcat by default. Options:

**Use logrotate** (Linux):

```
/opt/tomcat/logs/catalina.out {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
```

`copytruncate` copies the file then truncates it in-place, so Tomcat does not need to be restarted.

**Switch to application-level logging** (recommended for production):

Configure Spring Boot to log to a file via Logback, and disable stdout logging. This gives you per-day rotation and compression without touching Tomcat internals.

```xml
<!-- logback-spring.xml -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/opt/app/logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>/opt/app/logs/app.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

---

## Confirming Deployment via Tomcat Manager

If the Tomcat Manager app is enabled (it usually is not in production for security reasons, but often is in test), check the deployment status:

```
http://server:8080/manager/text/list
```

The response shows all deployed applications and their status:
```
/app:running:0:app
/manager:running:0:manager
```

A status of `stopped` instead of `running` means the WAR deployed but the Spring context failed to initialize. Check `localhost.YYYY-MM-DD.log` for the cause.

If Tomcat Manager is not available, verify deployment by:

```bash
# Check the expanded directory exists
ls $CATALINA_HOME/webapps/app/WEB-INF/classes/

# Tail the log and look for Spring's "Started Application"
tail -f $CATALINA_HOME/logs/catalina.out | grep -E "(Started|ERROR|SEVERE)"
```

---

## Summary

Tomcat is a stable, well-understood application server. The friction in using it comes from manual deployment steps, shared environments, and the need to understand log files rather than having structured log aggregation. Once you know where to look — `catalina.out` for startup failures, `setenv.sh` for JVM config, `server.xml` for thread pool — troubleshooting becomes straightforward. The discipline of cleaning up the work and expanded directories before each redeploy prevents the class of "why is old code still running" problems that waste the most time.
