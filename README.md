# Spring Cloud Microservices Playground

Inspiration: [in28minutes](https://github.com/in28minutes/spring-microservices-v3/tree/main)

## Introduction
This repository contains a lightweight microservices landscape built with Spring Boot 4.0.0, Spring Cloud 2025.1, and Java 25. It mirrors common production patterns such as centralized configuration, service discovery, gateway routing, resilience, and observability while remaining small enough to run locally. Each module now appears in three sibling directories so you can switch between Maven driven development under `local/`, container workflows under `docker-compose/`, and cluster ready deployments under `kubernetes/` without relearning the domain.

| Module (local path) | Port | Purpose | Highlights |
| --- | --- | --- | --- |
| `local/spring-cloud-config-server` | 8888 | Serves shared configuration from the Git repository at `local/git-localconfig-repo`. | `@EnableConfigServer`, file or remote Git support.
| `local/naming-server` | 8761 | Eureka discovery server so every service can register and locate peers. | `eureka.client.register-with-eureka=false` keeps the registry authoritative.
| `local/limits-service` | 8080 (default) | Simple REST API that reads `limits-service.minimum` and `limits-service.maximum` via Spring Cloud Config. | `@ConfigurationProperties` binding in `local/limits-service/src/main/java/com/springboot/microservices/limitsservice/configuration/Configuration.java`.
| `local/currency-exchange-service` | 8000 | Provides foreign-exchange quotes backed by the in-memory H2 database seeded by `data.sql`. | Resilience4j demos (`/sample-api`), Micrometer tracing, and Eureka client support.
| `local/currency-conversion-service` | 8100 | Calls the exchange service using the new `RestClient` and OpenFeign (`CurrencyExchangeProxy`). | Shows load-balanced discovery, Feign tracing, and Zipkin exporters.
| `local/api-gateway` | 8765 | Spring Cloud Gateway front door that consolidates the REST APIs and rewrites paths. | Custom routes plus `LoggingFilter` for request traces in `local/api-gateway/src/main/java/com/springboot/microservices/apigateway/LoggingFilter.java`.

Key architectural elements:
* **Externalized config**: Every service imports `spring.config.import=optional:configserver:` so shared values live in one Git repo. When Config Server is unavailable the services fall back to their local `application.properties` (the limits fallback of `3-997` lives in `local/limits-service/src/main/resources/application.properties`).
* **Service discovery**: `local/currency-exchange-service`, `local/currency-conversion-service`, `local/limits-service`, and `local/api-gateway` point to `http://localhost:8761/eureka` so there is no need to hardcode host and port pairs once Eureka is running.
* **Gateway routing**: `local/api-gateway/src/main/java/com/springboot/microservices/apigateway/ApiGatewayConfiguration.java` directs `/currency-exchange/**`, `/currency-conversion/**`, and `/currency-conversion-feign/**` to the right downstream service and rewrites `/currency-conversion-new/**` to the Feign endpoint.
* **Resilience and observability**: `local/currency-exchange-service` wires Resilience4j retry, rate, and bulkhead configs and exposes Micrometer tracing to Zipkin (see `local/currency-exchange-service/pom.xml`). `local/api-gateway` logs every request path, and actuators are enabled so you can hit `/actuator/health` everywhere.

## Repository layout
| Directory | Description |
| --- | --- |
| `local/` | Maven focused projects plus the sample Git config repo under `local/git-localconfig-repo` for development on a laptop.
| `docker-compose/` | Docker ready copies of the core services with Spring Boot image builds wired into the Maven plugin and Compose files under `docker-compose/backup/`.
| `kubernetes/` | Kubernetes tuned services, manifests, and ConfigMaps for the exchange and conversion workloads.

## Tutorial
Follow the sequences below to run the stack in each environment. All commands assume you are at the repository root.

### Local stack
#### 1. Prerequisites
1. Install JDK 25.
2. Install Maven 3.9 or newer.
3. Use the sample configuration Git repository under `local/git-localconfig-repo` or point `spring.cloud.config.server.git.uri` in `local/spring-cloud-config-server/src/main/resources/application.properties` to your own Git location.
4. Optionally start Zipkin if you want to visualize traces (update `spring.zipkin.baseUrl` in the services before enabling).

#### 2. Start the Spring Cloud Config Server (port 8888)
```bash
cd local/spring-cloud-config-server
mvn spring-boot:run
```
Verify `http://localhost:8888/limits-service/default` returns the merged configuration. Leave this window running.

#### 3. Launch the Eureka Naming Server (port 8761)
```bash
cd local/naming-server
mvn spring-boot:run
```
Once it starts visit `http://localhost:8761` to watch instances register as they come online.

#### 4. Bring up the backend services
Start each service in its own terminal. They automatically register with Eureka and fetch configuration from the Config Server.
```bash
# Limits Service (defaults to 8080 unless overridden in the config repo)
cd local/limits-service
mvn spring-boot:run
```

```bash
# Currency Exchange Service (port 8000)
cd local/currency-exchange-service
mvn spring-boot:run
```

```bash
# Currency Conversion Service (port 8100)
cd local/currency-conversion-service
mvn spring-boot:run
```

Smoke test the APIs directly:
```bash
curl http://localhost:8080/limits 
curl http://localhost:8000/currency-exchange/from/USD/to/TWD
curl http://localhost:8100/currency-conversion/from/USD/to/TWD/quantity/10
curl http://localhost:8100/currency-conversion-feign/from/EUR/to/TWD/quantity/10
curl http://localhost:8000/sample-api
```
Each response includes an `environment` field so you can see which instance served the call.

#### 5. Start the API Gateway (port 8765)
```bash
cd local/api-gateway
mvn spring-boot:run
```
From now on you can call every downstream service through the gateway and let it perform routing, discovery, and path rewriting:
```bash
curl http://localhost:8765/currency-exchange/from/USD/to/TWD
curl http://localhost:8765/currency-conversion/from/USD/to/TWD/quantity/10
curl http://localhost:8765/currency-conversion-new/from/USD/to/TWD/quantity/10
```
`local/api-gateway/src/main/java/com/springboot/microservices/apigateway/LoggingFilter.java` logs the path of each proxied request, so keep an eye on that terminal while testing.

#### 6. Explore observability and configuration updates
* Toggle values in your config Git repo (for example, change `limits-service.maximum`). Once committed, hit `/actuator/refresh` on the service or restart it to pick up the new value.
* Point the Micrometer Zipkin exporter (see the commented properties in `local/currency-exchange-service/src/main/resources/application.properties`) at a running Zipkin instance to correlate traces that start at the gateway and flow through Feign and Resilience4j.
* Use the Eureka dashboard to stop or start services and watch how the gateway automatically routes around missing instances.

### Docker Compose stack
#### 1. Prerequisites
1. Install Docker Desktop or another Docker engine that supports the `docker compose` command.
2. Install Maven 3.9 or newer.
3. Log in to the container registry that matches the image names configured in each `docker-compose/*/pom.xml` (the defaults publish `in28min/mmv3-...` images).

#### 2. Build the container images
Package each service in the `docker-compose` tree using the Spring Boot image builder. Run the commands below from the repository root so that the Maven wrapper picks up the proper source folders.
```bash
cd docker-compose/naming-server
mvn -DskipTests spring-boot:build-image
cd ../currency-exchange-service
mvn -DskipTests spring-boot:build-image
cd ../currency-conversion-service
mvn -DskipTests spring-boot:build-image
cd ../api-gateway
mvn -DskipTests spring-boot:build-image
cd ../..
```
Tag the images differently if you are pushing to your own registry, then update the Compose files under `docker-compose/` to match those names.

#### 3. Start the stack
Use the Compose bundles to start as many services as you need. The most complete bundle is `docker-compose/docker-compose-04-api-gateway.yaml` which launches the naming server, exchange, conversion, and gateway services.
```bash
docker compose -f docker-compose/docker-compose.yaml up -d
```
For a smaller footprint pick one of the earlier bundles (for example `docker-compose-02-naming-server.yaml`) or enable Zipkin with `docker-compose-05-zipkin.yaml` once you want tracing data. Compose publishes the familiar ports (8000, 8100, 8761, 8765) to your host machine.

#### 4. Exercise the services
`curl` the same endpoints you used locally. The requests should behave the same while the backing services talk over the Compose network.
```bash
curl http://localhost:8765/currency-conversion/from/USD/to/TWD/quantity/10
curl http://localhost:8765/currency-conversion-new/from/USD/to/TWD/quantity/10
```
Inspect logs with `docker compose logs -f currency-exchange` (swap the service name as needed) to confirm discovery and tracing.

#### 5. Tear down
```bash
docker compose -f docker-compose/docker-compose.yaml down --remove-orphans
```
Repeat the command with whichever Compose file you used. Removing orphans ensures containers from earlier experiments do not linger.

Remove all images:
```bash
docker rmi \
  paketobuildpacks/ubuntu-noble-run-tiny:0.0.47 \
  springboot/currency-exchange-service:0.0.1 \
  springboot/api-gateway:0.0.1 \
  springboot/naming-server:0.0.1 \
  springboot/currency-conversion-service:0.0.1 \
  paketobuildpacks/builder-noble-java-tiny:latest
```

### Kubernetes stack
#### 1. Prerequisites
1. Install Docker and Maven.
2. Install `kubectl` and connect it to a Kubernetes cluster (local `kind`, `minikube`, or any managed provider).
3. Choose a container registry that your cluster can pull from and sign in before building images.

#### 2. Build and push the images
The Kubernetes projects already contain Spring Boot image metadata that tags images as `in28min/mmv3-...`. Build each service and push it to your registry (or change the tag in the POM files before running these commands).
```bash
cd kubernetes/currency-exchange-service
mvn -DskipTests spring-boot:build-image
# push the resulting image if your registry is remote
cd ../currency-conversion-service
mvn -DskipTests spring-boot:build-image
cd ../..
```
Use `docker image tag` and `docker push` if you need to rename the images for your own registry.

#### 3. Apply the manifests
Deploy the exchange and conversion services with the final manifests supplied in the `backup` folders.
```bash
kubectl apply -f kubernetes/currency-exchange-service/backup/deployment-04-final.yaml
kubectl apply -f kubernetes/currency-conversion-service/backup/deployment-03-final.yaml
```
These manifests create Deployments, Services, and (for the conversion service) a ConfigMap that injects `CURRENCY_EXCHANGE_URI` pointing to `http://currency-exchange`.

#### 4. Verify and test
Watch the pods until they become ready.
```bash
kubectl get pods
kubectl get services currency-exchange currency-conversion
```
If your cluster provides LoadBalancer addresses, hit the services directly. Otherwise port-forward traffic from your machine.
```bash
kubectl port-forward service/currency-exchange 8000:8000
kubectl port-forward service/currency-conversion 8100:8100
curl http://localhost:8100/currency-conversion-feign/from/USD/to/TWD/quantity/10
```
Update `kubernetes/currency-conversion-service/backup/deployment-03-final.yaml` if your image names or namespaces differ, then reapply the manifest.

#### 5. Explore further
* Scale replicas with `kubectl scale deployment currency-exchange --replicas=3` and watch how the LoadBalancer or port-forward rotates traffic.
* Update the ConfigMap to change downstream URLs, then restart the conversion pods to pick up the change.
* Integrate Zipkin or another tracing backend by setting the same `MANAGEMENT_ZIPKIN_TRACING_ENDPOINT` environment variable that appears in the Compose files.

With these three workflows you can move the microservices playground from a laptop to Docker Compose and all the way to Kubernetes while keeping one README that explains every path.

## Others

You can use the command below to remove all `target` directories.

```bash
find . -name target -type d -prune -exec rm -rf {} +
```

This removes every stopped Docker container, keeping only the running ones.
```bash
docker container prune -f
```

This clears dangling (“null”) images that no longer have a tag or container reference.
```bash
docker image prune -f
```
