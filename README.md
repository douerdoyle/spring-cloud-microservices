# Spring Cloud Microservices System

This repository has 3 workspaces named local, docker-compose, and kubernetes. Each workspace has Spring Cloud services for local, Docker, and Kubernetes.

## Architecture

Spring Cloud Config Server sources properties from the Git-backed repository inside `local/git-localconfig-repo`.
Every runtime connects to the Config Server and exposes actuator endpoints for diagnostics.
The Eureka naming server collects registrations from Limits Service, Currency Exchange Service, Currency Conversion Service, and API Gateway, which means client code relies on logical service identifiers instead of literal host names.
The API Gateway becomes the single entry point in all environments, providing proxying.

```mermaid
flowchart LR
  Client[Client] --> Gateway[API Gateway]
  Gateway --> Ex[Currency Exchange]
  Gateway --> Conv[Currency Conversion]
  Gateway --> Limits[Limits Service]

  Repo[Git Config Repo] --> Config[Config Server]
  Config --> Ex
  Config --> Conv
  Config --> Limits
  Config --> Gateway
  Config --> Eureka[Naming Server]

  Ex -- register --> Eureka
  Conv -- register --> Eureka
  Limits -- register --> Eureka
  Gateway -- register --> Eureka
```

## Modules

| Module                              | Port | Purpose                                                                                                       |
| ----------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------- |
| `local/spring-cloud-config-server`  | 8888 | Shares configuration from `local/git-localconfig-repo`.                                                       |
| `local/naming-server`               | 8761 | The Eureka service, which is the discovery server so that every service can register and locate one another.  |
| `local/limits-service`              | 8080 | Reads `limits-service.minimum` and `limits-service.maximum` via Spring Cloud Config.                          |
| `local/currency-exchange-service`   | 8000 | Provides currency exchange from the in-memory H2 database.                                                    |
| `local/currency-conversion-service` | 8100 | The business logic is here, which will call currency-exchange-service to get the currency and then calculate. |
| `local/api-gateway`                 | 8765 | The gateway that provides different entry points for APIs.                                                    |

## Environment

Every environment section has instructions with commands.

### Local Development

The local workspace keeps pure Maven modules for direct execution on your environment. The config server reads the embedded Git repository, Eureka tracks each service, and the gateway provides proxying.

#### Guided Path

Ensure the JDK version is 17, Maven version is 3.9+, and Git is available.

#### Command Reference

```bash
cd local/spring-cloud-config-server
mvn spring-boot:run
```

```bash
cd local/naming-server
mvn spring-boot:run
```

Start each backend service in its own terminal:

```bash
cd local/limits-service && mvn spring-boot:run
```

```bash
cd local/currency-exchange-service && mvn spring-boot:run
```

```bash
cd local/currency-conversion-service && mvn spring-boot:run
```

Test the APIs to check if they can be called:

```bash
curl http://localhost:8080/limits
curl http://localhost:8000/currency-exchange/from/USD/to/TWD
curl http://localhost:8100/currency-conversion/from/USD/to/TWD/quantity/10
curl http://localhost:8100/currency-conversion-feign/from/EUR/to/TWD/quantity/10
```

Start the gateway and route traffic through it:

```bash
cd local/api-gateway && mvn spring-boot:run
```

```bash
curl http://localhost:8765/currency-conversion/from/USD/to/TWD/quantity/10
curl http://localhost:8765/currency-conversion-new/from/USD/to/TWD/quantity/10
```

### Docker

Spring Cloud on Docker is just like the local environment, but it is optimized for containers and uses Docker images without manual Dockerfiles.

#### Guided Path

Ensure Docker is installed and running on the host.

#### Command Reference

Build images:

```bash
cd docker-compose/

cd ./naming-server && mvn -DskipTests spring-boot:build-image
cd ../

cd ./currency-exchange-service && mvn -DskipTests spring-boot:build-image
cd ../

cd ./currency-conversion-service && mvn -DskipTests spring-boot:build-image
cd ../

cd ./api-gateway && mvn -DskipTests spring-boot:build-image
cd ../
```

Start the services:

```bash
docker compose -f docker-compose.yaml up -d
```

Test through the gateway:

```bash
curl http://localhost:8765/currency-conversion/from/USD/to/TWD/quantity/10

curl http://localhost:8765/currency-conversion-new/from/USD/to/TWD/quantity/10
```

View logs of a specific service:

```bash
docker compose logs -f currency-exchange
```

Tear down and clean up the environment:

```bash
docker compose -f docker-compose.yaml down --remove-orphans
```

Delete all images with the "springboot/" prefix.

```bash
docker images --format '{{.Repository}}:{{.Tag}}' | grep '^springboot/' | xargs -r docker rmi
```

### Kubernetes

The Kubernetes environment includes Currency Exchange Service and Currency Conversion Service. The environment shows how Spring Cloud is run on Kubernetes.
Config, Eureka, and Gateway are not required in Kubernetes, so we don't need to install them.

#### macOS Setup (Clean and Minimal)

This setup uses kind (Kubernetes in Docker). It keeps your host clean and is easy to remove.

Install tools:

```bash
brew install kind
```

Create a local cluster:

```bash
kind create cluster --name scms
```

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

Clean removal:

```bash
kind delete cluster --name scms
# Optional
# Uninstall kind
# brew uninstall kind
```

Optional cleanup if you want to remove kube config data:

```bash
rm -rf ~/.kube
```

#### Command Reference

Build Docker images and load them into Kubernetes:

```bash
cd kubernetes/

cd ./currency-exchange-service && mvn -DskipTests spring-boot:build-image

cd ../currency-conversion-service && mvn -DskipTests spring-boot:build-image

cd ../

# Delete images in Kubernetes first
docker exec -it scms-control-plane crictl rmi springboot/mmv3-currency-conversion-service:0.0.1-k8s
docker exec -it scms-control-plane crictl rmi springboot/mmv3-currency-exchange-service:0.0.1-k8s

# Load images to Kubernetes
kind load docker-image springboot/mmv3-currency-conversion-service:0.0.1-k8s --name scms
kind load docker-image springboot/mmv3-currency-exchange-service:0.0.1-k8s --name scms
```

Apply the deployments:

```bash
kubectl apply -f currency-exchange-service/deployment.yaml
kubectl apply -f currency-conversion-service/deployment.yaml
```

Inspect resources:

```bash
kubectl get deployment
kubectl get pods
kubectl get services currency-exchange currency-conversion
```

Port forward for local testing:

```bash
kubectl port-forward service/currency-exchange 8000:8000
```

```bash
kubectl port-forward service/currency-conversion 8100:8100
```

```bash
curl http://localhost:8100/currency-conversion-feign/from/USD/to/TWD/quantity/10
```

Scale and tune:

```bash
kubectl scale deployment currency-exchange --replicas=3
```

The output will change from:

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
currency-conversion   1/1     1            1           43s
currency-exchange     1/1     1            1           43s
```

To this:

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
currency-conversion   1/1     1            1           46s
currency-exchange     1/3     3            1           46s
```

## Repository Utilities

Clear all Maven target folders when you want a clean slate:

```bash
find . -name target -type d -prune -exec rm -rf {} +
```

Remove leftover containers and images from Docker experiments:

```bash
docker container prune -f
docker image prune -f
```
