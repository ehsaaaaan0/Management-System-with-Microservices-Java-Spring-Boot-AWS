# Microservices Architecture with Spring Boot, gRPC, Kafka, and Docker

A complete microservices project built with Java and Spring Boot. This is a real-world setup that demonstrates how to build distributed systems with gRPC communication, event streaming via Kafka, and containerized deployment.

## What's in Here

This project includes several services working together:

- **Patient Service**: Core service for managing patient data. Talks to a PostgreSQL database and communicates with other services via gRPC and Kafka.
- **Billing Service**: Handles billing operations, exposed through gRPC endpoints.
- **Analytics Service**: Consumes Kafka events from the patient service and processes analytics data.
- **Auth Service**: Manages user authentication and JWT token generation/validation.
- **API Gateway**: Single entry point for all requests, handles routing and applies JWT validation filters.

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Java 21
- Maven

### Running Everything Locally

First, create a Docker network so the containers can talk to each other:

```bash
docker network create internal
```

Then fire up each service. I'll walk through the main ones:

#### Patient Service and Database

```bash
# Build the image
docker build -t patient-service:latest -f patient-service/Dockerfile patient-service

# Run the database
docker run -d --name patient-service-db --network internal \
  -p 5000:5432 \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=db \
  -v C:\Users\EHSAAAAAN\Desktop\Files\db_volumes\patient-service-db:/var/lib/postgresql/data \
  postgres:15

# Run the service
docker run -d --name patient-service --network internal \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db \
  -e SPRING_DATASOURCE_USERNAME=admin \
  -e SPRING_DATASOURCE_PASSWORD=password \
  -e SPRING_JPA_HIBERNATE_DDL_AUTO=update \
  -e SPRING_SQL_INIT_MODE=always \
  -e SPRING_JPA_DEFER_DATASOURCE_INITIALIZATION=true \
  -e BILLING_SERVICE_ADDRESS=billing-service \
  -e BILLING_SERVICE_GRPC_PORT=9001 \
  -e SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  patient-service:latest
```

#### Billing Service

```bash
docker build -t billing-service:latest -f billing-service/Dockerfile billing-service

docker run -d --name billing-service --network internal \
  -p 4010:4001 \
  -p 9010:9001 \
  --restart unless-stopped \
  billing-service:latest
```

#### Kafka and Analytics Service

Set up Kafka first:

```bash
docker run -d --name kafka --network internal \
  -p 9092:9092 \
  -p 9093:9093 \
  -p 9094:9094 \
  -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094 \
  -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093 \
  -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094 \
  -e KAFKA_CFG_NODE_ID=0 \
  -e KAFKA_CFG_PROCESS_ROLES=controller,broker \
  -e KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT \
  bitnamilegacy/kafka:3.6.0
```

Optional: Add Kafka UI to visualize what's happening:

```bash
docker run -d --name kafka-ui --network internal \
  -p 8080:8080 \
  -e KAFKA_CLUSTERS_0_NAME=local \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092 \
  provectuslabs/kafka-ui
```

Then the analytics service:

```bash
docker build -t analytics-service:latest ./analytics-service

docker run -d --name analytics-service --network internal \
  -p 4002:4002 \
  -e SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  analytics-service:latest
```

#### Auth Service and Database

```bash
# Database first
docker run -d --name auth-service-db --network internal \
  -p 5001:5432 \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=db \
  -v C:\Users\EHSAAAAAN\Desktop\Files\db_volumes\auth-service-db:/var/lib/postgresql/data \
  postgres:15

# Auth service
docker build -t auth-service:latest -f auth-service/Dockerfile auth-service

docker run -d --name auth-service --network internal \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db \
  -e SPRING_DATASOURCE_USERNAME=admin \
  -e SPRING_DATASOURCE_PASSWORD=password \
  -e SPRING_JPA_HIBERNATE_DDL_AUTO=update \
  -e SPRING_SQL_INIT_MODE=always \
  -e jwt.secret=your_secret_key \
  auth-service:latest
```

#### API Gateway

```bash
docker build -t api-gateway:latest -f api-gateway/Dockerfile api-gateway

docker run -d --name api-gateway --network internal \
  -p 4004:4004 \
  -e AUTH_SERVICE_URL=http://auth-service:4005 \
  api-gateway:latest
```

#### LocalStack (for AWS simulation)

If you want to test AWS infrastructure locally:

```bash
docker pull localstack/localstack

docker run -d --name localstack \
  -p 4566:4566 \
  localstack/localstack
```

## How It All Works

The architecture follows a typical microservices pattern:

1. **API Gateway** receives requests and validates JWT tokens before routing them to the appropriate service.
2. **Patient Service** handles patient operations, stores data in PostgreSQL, and publishes events to Kafka when patients are created or modified.
3. **Billing Service** exposes gRPC endpoints that the Patient Service calls synchronously for billing operations.
4. **Analytics Service** subscribes to Kafka events and processes analytics independently.
5. **Auth Service** handles login and token generation for the entire system.

The communication happens two ways:
- Synchronous: via gRPC between Patient and Billing services
- Asynchronous: via Kafka for event-driven operations between Patient and Analytics services

## Key Features

- **REST APIs** with proper validation and error handling
- **gRPC** for inter-service communication
- **Kafka** for event streaming
- **JWT Authentication** at the API Gateway
- **Docker containerization** for easy deployment
- **PostgreSQL** for persistent data storage
- **OpenAPI/Swagger** documentation for endpoints
- **Integration tests** included
- **Infrastructure as Code** with CloudFormation templates

## Ports

Here's where everything lives:

- API Gateway: `4004`
- Patient Service: (internal, accessed through gateway)
- Patient DB: `5000`
- Billing Service: `4010` (REST), `9010` (gRPC)
- Auth Service: `4005`
- Auth DB: `5001`
- Analytics Service: `4002`
- Kafka: `9092` (broker), `9093` (controller), `9094` (external)
- Kafka UI: `8080`
- LocalStack: `4566`

## What Each Service Does

### Patient Service
Manages patient records. When you create or update a patient, it:
1. Validates the data
2. Calls the Billing Service via gRPC to set up billing
3. Publishes a Kafka event so Analytics can process it

### Billing Service
Receives gRPC calls from Patient Service to handle billing-related operations.

### Analytics Service
Listens to Kafka events from Patient Service and processes analytics data. It's completely decoupled so the Patient Service doesn't wait for it.

### Auth Service
Manages user accounts and JWT tokens. The API Gateway validates every request using the token validation endpoint here.

### API Gateway
Routes all incoming requests to the right service and enforces authentication via JWT validation filters.

## Testing

The project includes integration tests. To run them:

```bash
mvn clean test
```

Tests include:
- Authentication flow (login and token validation)
- Patient CRUD operations through the gateway
- Full request/response validation

## Infrastructure as Code

CloudFormation templates are included for deploying to AWS:
- VPC setup
- RDS databases
- MSK (Managed Streaming for Kafka)
- ECS clusters and services
- Application Load Balancer

Deploy to LocalStack for testing, or modify for real AWS infrastructure.

## Development Notes

All services are built with Spring Boot and use Maven for dependency management. Each service has its own `Dockerfile` for containerization.

The project demonstrates:
- Proper separation of concerns
- Asynchronous communication patterns
- Synchronous RPC patterns
- API authentication and authorization
- Database per service principle (Patient and Auth have separate databases)
- Event-driven architecture

## Troubleshooting

**Services can't talk to each other**: Make sure they're all on the same Docker network (`internal`). Check with `docker network inspect internal`.

**Database connection errors**: Verify the database container is running and the credentials match in the service environment variables.

**Kafka issues**: Make sure the Kafka container is fully started before running producers/consumers. Check logs with `docker logs kafka`.

## Next Steps

Consider adding:
- Docker Compose file to simplify startup
- Health checks for all services
- Monitoring with Prometheus and Grafana
- Distributed tracing with Jaeger
- Service mesh (Istio) for advanced traffic management

## License

MIT

---

Built as a complete microservices learning project.