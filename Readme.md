//create patient db image
docker network create internal
docker run -d --name patient-service-db --network internal -p 5000:5432 -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=password -e POSTGRES_DB=db -v C:\Users\EHSAAAAAN\Desktop\Files\db_volumes\patient-service-db:/var/lib/postgresql/data postgres:15
// Patient-Service image
docker rm -f patient-service
docker build -t patient-service:latest .
docker run -d --name patient-service --network internal -e SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db -e SPRING_DATASOURCE_USERNAME=admin -e SPRING_DATASOURCE_PASSWORD=password -e SPRING_JPA_HIBERNATE_DDL_AUTO=update -e SPRING_SQL_INIT_MODE=always -e SPRING_JPA_DEFER_DATASOURCE_INITIALIZATION=true -e BILLING_SERVICE_ADDRESS=billing-service -e BILLING_SERVICE_GRPC_PORT=9001 -e SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092 patient-service:latest
//billing Service image
docker rm -f billing-service
docker build -t billing-service:latest -f billing-service/Dockerfile billing-service
docker run -d --name billing-service --network internal -p 4010:4001 -p 9010:9001 --restart unless-stopped billing-service:latest

//Kafka  image
to create kafka
docker run -d --name kafka --network internal -p 9092:9092 -p 9093:9093 -p 9094:9094 -e KAFKA_CFG_ADVERTISED_LISTENERS="PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094" -e
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS="0@kafka:9093" -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP="CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT" -e KAFKA_CFG_LISTENERS="PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094" -e KAFKA_CFG_NODE_ID=0 -e KAFKA_CFG_PROCESS_ROLES="controller,broker" -e KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT bitnamilegacy/kafka:3.6.0
to test with kafka ui
docker run -d --name kafka-ui --network internal -p 8080:8080 -e KAFKA_CLUSTERS_0_NAME=local -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092 provectuslabs/kafka-ui

//analytics Service image
to create analytics-server
docker build -t analytics-service:latest ./analytics-service
docker run -d --name analytics-service --network internal -p 4002:4002 -e SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092 analytics-service:latest

//api gateway
docker build -t api-gateway:latest -f api-gateway/Dockerfile api-gateway
docker run -d --name api-gateway --network internal -p 4004:4004 api-gateway:latest

//create auth db image
docker run -d --name auth-service-db --network internal -p 5001:5432 -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=password -e POSTGRES_DB=db -v C:\Users\EHSAAAAAN\Desktop\Files\db_volumes\auth-service-db:/var/lib/postgresql/data postgres:15
//create auth server
docker build -t auth-service:latest -f auth-service/Dockerfile auth-service
docker run -d --name auth-service --network internal -p 4005:4005 -e SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db -e SPRING_DATASOURCE_USERNAME=admin -e SPRING_DATASOURCE_PASSWORD=password -e SPRING_JPA_HIBERNATE_DDL_AUTO=update -e SPRING_SQL_INIT_MODE=always -e jwt.secret=bXktc3VwZXItc2VjcmV0LWtleS0xMjM0NTY3ODkwMTIzNDU2 auth-service:latest