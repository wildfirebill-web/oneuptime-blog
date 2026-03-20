# How to Deploy a Spring Boot + MySQL Stack via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Spring Boot, MySQL, Java, Docker Compose, Microservices

Description: Learn how to deploy a Spring Boot application with MySQL via Portainer, including database health checks, environment variable configuration, and JAR deployment.

---

Spring Boot with MySQL is a standard Java enterprise stack. Portainer provides a clean interface for managing the multi-container deployment with log streaming and container lifecycle management.

## Compose Stack

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpass       # Change this
      MYSQL_DATABASE: springapp
      MYSQL_USER: spring
      MYSQL_PASSWORD: springpass          # Change this
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpass"]
      interval: 10s
      timeout: 5s
      retries: 10

  app:
    image: eclipse-temurin:21-jre-alpine
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy       # Wait for MySQL to be ready
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/springapp?useSSL=false&allowPublicKeyRetrieval=true
      SPRING_DATASOURCE_USERNAME: spring
      SPRING_DATASOURCE_PASSWORD: springpass
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      SERVER_PORT: 8080
    volumes:
      - ./app.jar:/app/app.jar    # Mount your built JAR file
    command: java -jar /app/app.jar

volumes:
  mysql_data:
```

## Spring Boot application.properties

```properties
# src/main/resources/application.properties
# These are overridden by environment variables in the container
spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:mysql://localhost:3306/springapp}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:spring}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:springpass}
spring.jpa.hibernate.ddl-auto=${SPRING_JPA_HIBERNATE_DDL_AUTO:update}

# Health actuator endpoint
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=always
```

## Building and Deploying

```bash
# Build the Spring Boot JAR
./mvnw clean package -DskipTests

# Copy the JAR to the stack directory
cp target/myapp-0.0.1-SNAPSHOT.jar ./app.jar

# Deploy via Portainer
```

## Monitoring

Use OneUptime to monitor `http://<host>:8080/actuator/health`. Spring Boot Actuator returns `{"status":"UP"}` when healthy, including database connectivity status. Alert on any non-UP status to catch database connection failures.
