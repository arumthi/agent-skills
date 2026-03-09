# Spring Boot 3.x with Java 21: Project Setup & Best Practices

## Overview

Spring Boot 3.x with Java 21 LTS provides modern Java enterprise development with GraalVM native images, virtual threads, and pattern matching support. This guide covers project initialization, configuration, and production-ready patterns for microservices.

## Prerequisites

- **Java 21 LTS**: Download from [adoptium.net](https://adoptium.net/) or [oracle.com](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html)
- **Maven 3.9+**: `mvn -version`
- **Spring Boot 3.2.x**: Latest stable version
- **IDE**: IntelliJ IDEA, VS Code with Extension Pack for Java

## Step 1: Create Spring Boot Project

### Via Spring Initializr (Recommended)

```bash
# Navigate to https://start.spring.io/
# Or use curl

curl "https://start.spring.io/starter.zip" \
  -d language=java \
  -d javaVersion=21 \
  -d packaging=jar \
  -d groupId=com.example \
  -d artifactId=order-service \
  -d version=1.0.0 \
  -d name=Order%20Service \
  -d dependencies=web,data-jpa,validation,security,actuator,openapi3,aop,lombok,cache \
  -d bootVersion=3.2.0 \
  -d type=maven-project \
  -o order-service.zip

unzip order-service.zip
cd order-service
```

### Via Maven Archetype

```bash
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=order-service \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

## Step 2: Configure pom.xml

### Parent POM with Java 21

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
    <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <groupId>com.example</groupId>
  <artifactId>order-service</artifactId>
  <version>1.0.0</version>
  <name>Order Service</name>
  <description>Microservice for order management</description>

  <properties>
    <java.version>21</java.version>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring-cloud.version>2023.0.0</spring-cloud.version>
  </properties>

  <dependencies>
    <!-- Web Framework -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Data Access -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Oracle Driver -->
    <dependency>
      <groupId>com.oracle.database.jdbc</groupId>
      <artifactId>ojdbc11</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- Connection Pooling -->
    <dependency>
      <groupId>com.zaxxer</groupId>
      <artifactId>HikariCP</artifactId>
      <version>5.0.1</version>
    </dependency>

    <!-- Security & JWT -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-api</artifactId>
      <version>0.12.3</version>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-impl</artifactId>
      <version>0.12.3</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-jackson</artifactId>
      <version>0.12.3</version>
      <scope>runtime</scope>
    </dependency>

    <!-- API Documentation -->
    <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.0.2</version>
    </dependency>

    <!-- Validation -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Observability -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-registry-prometheus</artifactId>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-exporter-jaeger</artifactId>
      <version>1.32.0</version>
    </dependency>

    <!-- Logging -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-logging</artifactId>
    </dependency>

    <!-- Caching -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <!-- Utilities -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers</artifactId>
      <version>1.19.3</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>oracle-xe</artifactId>
      <version>1.19.3</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>1.19.3</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
          <excludes>
            <exclude>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
            </exclude>
          </excludes>
        </configuration>
      </plugin>

      <!-- Maven Compiler with Java 21 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>21</source>
          <target>21</target>
          <release>21</release>
          <compilerArgs>--enable-preview</compilerArgs>
        </configuration>
      </plugin>

      <!-- Surefire for tests -->
      <plugin>
        <groupId>org.apache.maven.surefire</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration>
          <argLine>--enable-preview</argLine>
        </configuration>
      </plugin>

      <!-- Code Coverage -->
      <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <version>0.8.10</version>
        <executions>
          <execution>
            <goals>
              <goal>prepare-agent</goal>
            </goals>
          </execution>
          <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
              <goal>report</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

## Step 3: Application Configuration

### application.yml (Production-ready)

```yaml
spring:
  application:
    name: order-service
    version: 1.0.0
  
  # Profiles: dev, staging, production
  profiles:
    active: dev
  
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/XEPDB1
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: oracle.jdbc.OracleDriver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 900000
      max-lifetime: 1800000
      auto-commit: false
  
  jpa:
    database-platform: org.hibernate.dialect.OracleDialect
    hibernate:
      ddl-auto: validate  # Use Flyway for migrations, not auto
    properties:
      hibernate:
        dialect: org.hibernate.dialect.OracleDialect
        format_sql: true
        jdbc:
          batch_size: 20
          fetch_size: 50
        order_inserts: true
        order_updates: true
  
  # Caching
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m
  
  # Security
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER_URI:http://auth-service:8080}
          jwk-set-uri: ${JWT_JWK_SET_URI:http://auth-service:8080/.well-known/jwks.json}
  
  # Actuator
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
      base-path: /actuator
  
  # Jackson
  jackson:
    serialization:
      write-dates-as-timestamps: false
    deserialization:
      fail-on-unknown-properties: false
  
  # Graceful Shutdown
  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  port: 8080
  servlet:
    context-path: /
  shutdown: graceful
  compression:
    enabled: true
    min-response-size: 1024

logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/order-service.log
    max-size: 10MB
    max-history: 30

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 1.0  # Sample all traces in dev, adjust in prod
```

### application-dev.yml (Development Override)

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/XEPDB1
    username: system
    password: oracle
  
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true

logging:
  level:
    root: DEBUG
    com.example: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### application-production.yml

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@//${DB_HOST}:${DB_PORT}/${DB_SID}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50
  
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

logging:
  level:
    root: WARN
    com.example: INFO
  file:
    name: /var/log/order-service/application.log

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
```

## Step 4: Main Application Class

```java
package com.example.orderservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableCaching
@EnableAsync
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

    // Health indicator (for Kubernetes liveness probes)
    @Bean
    public org.springframework.boot.actuate.health.HealthIndicator applicationHealthIndicator() {
        return () -> org.springframework.boot.actuate.health.Health.up()
            .withDetail("application", "Order Service")
            .withDetail("version", "1.0.0")
            .build();
    }
}
```

## Step 5: Create Core Project Structure

```
src/
├── main/
│   ├── java/com/example/orderservice/
│   │   ├── OrderServiceApplication.java
│   │   ├── domain/
│   │   │   ├── entity/
│   │   │   │   ├── Order.java           (JPA entity)
│   │   │   │   └── OrderItem.java
│   │   │   ├── repository/
│   │   │   │   └── OrderRepository.java (Spring Data interface)
│   │   │   ├── service/
│   │   │   │   └── OrderService.java    (Business logic)
│   │   │   └── dto/
│   │   │       ├── CreateOrderRequest.java
│   │   │       └── OrderResponse.java
│   │   ├── api/
│   │   │   └── OrderController.java     (REST endpoints)
│   │   ├── exception/
│   │   │   ├── OrderNotFoundException.java
│   │   │   └── GlobalExceptionHandler.java
│   │   ├── config/
│   │   │   ├── SecurityConfig.java
│   │   │   └── DatabaseConfig.java
│   │   └── event/
│   │       └── OrderEventPublisher.java
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-production.yml
│       ├── logback-spring.xml
│       └── db/migration/
│           ├── V1__create_orders_table.sql
│           └── V2__create_order_items_table.sql
└── test/
    └── java/com/example/orderservice/
        ├── OrderServiceApplicationTests.java
        ├── api/
        │   └── OrderControllerIntegrationTest.java
        ├── service/
        │   └── OrderServiceTest.java
        └── testcontainers/
            └── OracleTestContainer.java
```

## Step 6: Build and Run

```bash
# Build
mvn clean package

# Run locally
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"

# Or run JAR
java -jar target/order-service-1.0.0.jar --spring.profiles.active=dev

# Check health
curl http://localhost:8080/actuator/health

# View Swagger UI
open http://localhost:8080/swagger-ui.html
```

## Key Java 21 Features to Use

```java
// Records for DTOs (immutable data carriers)
public record OrderResponse(
    UUID orderId,
    String status,
    BigDecimal totalAmount
) {}

// Pattern matching
if (order instanceof Order o && o.getStatus() == OrderStatus.PENDING) {
    processOrder(o);
}

// Virtual threads for async operations (Spring 3.2+)
@Async
public Future<OrderResponse> asyncCreateOrder(CreateOrderRequest request) {
    return CompletableFuture.completedFuture(createOrder(request));
}

// Text blocks for SQL
String query = """
    SELECT * FROM orders
    WHERE customer_id = ? AND status = ?
    ORDER BY created_date DESC
    """;
```

## Testing with Java 21 & Spring Boot 3.x

```java
@SpringBootTest
class OrderServiceIntegrationTest {
    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", () -> oracle.getJdbcUrl());
        registry.add("spring.datasource.username", () -> "test");
        registry.add("spring.datasource.password", () -> "test");
    }
}
```

---

## Success Criteria

✅ Project builds with `mvn clean package`  
✅ Application starts: `mvn spring-boot:run`  
✅ Health check accessible: `curl http://localhost:8080/actuator/health`  
✅ Swagger UI available: `/swagger-ui.html`  
✅ Tests pass: `mvn test` with 80%+ coverage
