# Spring Cloud Microservices Playground

## Introduction
This repository contains a lightweight microservices landscape built with Spring Boot 4.0.0, Spring Cloud 2025.1, and Java 25. It mirrors common production patterns—centralized configuration, service discovery, gateway routing, resilience, and observability—while remaining small enough to run locally. Each module can be started independently, but they shine when launched together:

| Module | Port | Purpose | Highlights |
| --- | --- | --- | --- |
| `spring-cloud-config-server` | 8888 | Serves shared configuration from a Git repository (`spring-cloud-config-server/src/main/resources/application.properties`). | `@EnableConfigServer`, file/remote Git support. |
| `naming-server` | 8761 | Eureka discovery server so every service can register/locate peers. | `eureka.client.register-with-eureka=false` keeps registry authoritative. |
| `limits-service` | 8080 (default) | Simple REST API that reads `limits-service.minimum/maximum` via Spring Cloud Config. | `@ConfigurationProperties` binding in `limits-service/.../configuration/Configuration.java`. |
| `currency-exchange-service` | 8000 | Provides foreign-exchange quotes backed by an in-memory H2 database (`data.sql`). | Resilience4j demos (`/sample-api`), Micrometer tracing, Eureka client. |
| `currency-conversion-service` | 8100 | Calls the exchange service using both the new `RestClient` and OpenFeign (`CurrencyExchangeProxy`). | Shows load-balanced discovery, Feign tracing, zipkin exporters. |
| `api-gateway` | 8765 | Spring Cloud Gateway front door that consolidates the REST APIs and rewrites paths. | Custom routes + `LoggingFilter` for request traces. |

Key architectural elements:
- **Externalized config**: Every service imports `spring.config.import=optional:configserver:` (or with explicit URL) so shared values live in one Git repo. When Config Server is unavailable, services fall back to their local `application.properties` (e.g., the limits fallback of `3-997`).
- **Service discovery**: `currency-exchange`, `currency-conversion`, `limits-service`, and `api-gateway` point to `http://localhost:8761/eureka` so you never hardcode host:port pairs once Eureka is running.
- **Gateway routing**: `api-gateway/.../ApiGatewayConfiguration.java` directs `/currency-exchange/**`, `/currency-conversion/**`, and `/currency-conversion-feign/**` to the right downstream service and even rewrites `/currency-conversion-new/**` to the Feign endpoint.
- **Resilience & observability**: `currency-exchange-service` wires Resilience4j retry/rate/bulkhead configs and exposes Micrometer tracing to Zipkin (see `currency-exchange-service/pom.xml`). `api-gateway` logs every request path, and actuators are enabled so you can hit `/actuator/health` everywhere.

## Tutorial
Follow the sequence below to spin up the full stack locally. All commands assume you are at the repository root. Use another shell per service so logs stay readable.

### 1. Prerequisites
1. Install JDK 25 (matching `<java.version>` in each `pom.xml`).
2. Install Maven 3.9+.
3. Provide a configuration Git repository that contains the `limits-service`, `currency-exchange`, `currency-conversion`, etc. property files (the Udemy repo used by default is referenced as `file:///Users/rangakaranam/...`). Point `spring.cloud.config.server.git.uri` in `spring-cloud-config-server/src/main/resources/application.properties` to a path you control, or switch to a public Git URL.
4. Optionally start Zipkin if you want to visualize traces (update `spring.zipkin.baseUrl` in the services before enabling).

### 2. Start the Spring Cloud Config Server (port 8888)
```bash
cd spring-cloud-config-server
mvn spring-boot:run
```
Verify `http://localhost:8888/limits-service/default` returns the merged configuration. Leave this window running.

### 3. Launch the Eureka Naming Server (port 8761)
```bash
cd naming-server
mvn spring-boot:run
```
Once it starts, visit `http://localhost:8761`. You will see registered applications appear as you bring them up.

### 4. Bring up the backend services
Start each service in its own terminal. They automatically register with Eureka and fetch config from the Config Server.

```bash
# Limits Service (defaults to 8080 unless overridden in config repo)
cd limits-service
mvn spring-boot:run

# Currency Exchange Service (port 8000)
cd currency-exchange-service
mvn spring-boot:run

# Currency Conversion Service (port 8100)
cd currency-conversion-service
mvn spring-boot:run
```

Smoke-test the APIs directly:
```bash
# Limits pulled from the config repo
curl http://localhost:8080/limits

# Exchange quote sourced from H2 data.sql
curl http://localhost:8000/currency-exchange/from/USD/to/INR

# Conversion using RestClient
curl http://localhost:8100/currency-conversion/from/USD/to/INR/quantity/10

# Conversion using OpenFeign (load-balanced through Eureka)
curl http://localhost:8100/currency-conversion-feign/from/EUR/to/INR/quantity/10

# Resilience4j sample endpoint demonstrating the configured Bulkhead
curl http://localhost:8000/sample-api
```
Each response includes an `environment` field so you can see which instance/transport served the call (for example `8000 rest client`).

### 5. Start the API Gateway (port 8765)
```bash
cd api-gateway
mvn spring-boot:run
```
From now on you can call every downstream service through the gateway and let it perform routing, discovery, and path rewriting:
```bash
# Routed to currency-exchange via lb://currency-exchange
curl http://localhost:8765/currency-exchange/from/USD/to/INR

# Routed to currency-conversion (RestClient endpoint)
curl http://localhost:8765/currency-conversion/from/USD/to/INR/quantity/10

# Path rewrite demo: /currency-conversion-new/** -> /currency-conversion-feign/**
curl http://localhost:8765/currency-conversion-new/from/USD/to/INR/quantity/10
```
`api-gateway/src/main/java/com/in28minutes/microservices/apigateway/LoggingFilter.java` logs the path of each proxied request, so keep an eye on that terminal while testing.

### 6. Explore observability & configuration updates
- Toggle values in your config Git repo (for example, change `limits-service.maximum`). Once committed, hit `/actuator/refresh` on the service or restart it to pick up the new value.
- Point the Micrometer Zipkin exporter (see the commented properties in `currency-exchange-service/src/main/resources/application.properties`) at a running Zipkin to correlate traces that start at the gateway and flow through Feign and Resilience4j.
- Use the Eureka dashboard to stop/start services and watch how the gateway automatically routes around missing instances.

You now have a working microservices playground: centralized configuration, discovery, resilience, gateway routing, and two collaborating business services ready for experimentation.
