# Part 46: Docker และ Deployment

## สารบัญ
1. [Docker Overview](#docker-overview)
2. [Dockerfile for Scala](#dockerfile)
3. [Docker Compose](#docker-compose)
4. [sbt-native-packager](#sbt-native-packager)
5. [Kubernetes Basics](#kubernetes)
6. [CI/CD Pipeline](#cicd)

---

## Docker Overview

### แนวคิด

```
Docker: containerization platform

Container vs VM:
- Container: shares OS kernel, lightweight, fast start
- VM: full OS, heavier, more isolation

Key components:
- Dockerfile: instructions to build image
- Image: read-only template
- Container: running instance of image
- Registry: stores images (Docker Hub, ECR, GCR)
- Layer: each instruction = layer (cached)

JVM in Docker considerations:
- Set -Xmx to avoid OOM
- Use -XX:+UseContainerSupport (JDK 11+) for cgroup awareness
- Set thread pool sizes based on container CPU limits
```

---

## Dockerfile

### Multi-Stage Build

```dockerfile
# Dockerfile
# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /build

# Copy build files first (layer caching)
COPY project/ project/
COPY build.sbt .

# Download dependencies (cached if no changes)
RUN apk add --no-cache bash && \
    curl -L https://github.com/sbt/sbt/releases/download/v1.9.7/sbt-1.9.7.tgz | \
    tar -xz -C /usr/local/ --strip-components=1

RUN sbt update

# Copy source
COPY src/ src/

# Build fat jar
RUN sbt assembly

# Stage 2: Runtime (minimal image)
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

# Copy jar from builder
COPY --from=builder /build/target/scala-3.3.1/myapp-assembly-1.0.0.jar app.jar

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

# JVM tuning
ENV JAVA_OPTS="-Xms256m -Xmx512m \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s \
    CMD wget -q -O - http://localhost:8080/health/live || exit 1

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]
```

### Optimized Layer Caching

```dockerfile
# Dockerfile with better caching using sbt-native-packager
FROM eclipse-temurin:17-jdk-alpine AS base

# Install sbt
RUN apk add --no-cache bash curl && \
    curl -L https://github.com/sbt/sbt/releases/download/v1.9.7/sbt-1.9.7.tgz | \
    tar -xz -C /usr/local/ --strip-components=1

WORKDIR /app

# Cache dependencies
COPY build.sbt .
COPY project/plugins.sbt project/
COPY project/build.properties project/
RUN sbt update

# Build
COPY . .
RUN sbt stage

# Runtime
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app
COPY --from=base /app/target/universal/stage .

RUN adduser -u 1001 -D appuser && chown -R appuser /app
USER appuser

EXPOSE 8080
ENTRYPOINT ["bin/myapp"]
```

---

## Docker Compose

### Development Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_URL=jdbc:postgresql://postgres:5432/mydb
      - DB_USER=postgres
      - DB_PASSWORD=password
      - REDIS_URL=redis://redis:6379
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    networks:
      - app-network

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass redispassword
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    networks:
      - app-network

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

---

## sbt-native-packager

### Build Configuration

```scala
// build.sbt
enablePlugins(JavaAppPackaging, DockerPlugin, GraalVMNativeImagePlugin)

// Docker settings
dockerBaseImage := "eclipse-temurin:17-jre-alpine"
dockerExposedPorts := Seq(8080)
dockerUsername := Some("myorg")
dockerRepository := Some("registry.mycompany.com")

Docker / packageName := "myapp"
Docker / version     := version.value

// Add JVM options
javaOptions in Universal ++= Seq(
  "-J-Xms256m",
  "-J-Xmx512m",
  "-J-XX:+UseG1GC",
  "-J-XX:+UseContainerSupport"
)

// Non-root user
daemonUser in Docker  := "appuser"
daemonGroup in Docker := "appgroup"

// Docker commands
// sbt docker:publishLocal  # build and tag locally
// sbt docker:publish       # push to registry

// GraalVM Native Image (optional, for fast startup)
GraalVMNativeImage / packageName := "myapp-native"
graalVMNativeImageOptions ++= Seq(
  "--no-fallback",
  "--static",  // statically linked binary
  "-H:+ReportExceptionStackTraces"
)
```

---

## Kubernetes

### Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-url
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-password
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'sbt'

      - name: Run tests
        run: sbt test
        env:
          TEST_DB_URL: jdbc:postgresql://localhost:5432/testdb

      - name: Generate coverage report
        run: sbt coverage test coverageReport

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'sbt'

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: registry.mycompany.com
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Build and push Docker image
        run: sbt docker:publish
        env:
          DOCKER_REGISTRY: registry.mycompany.com

  deploy:
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: k8s/
          images: registry.mycompany.com/myapp:${{ github.sha }}
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Multi-stage Docker build สำหรับ Scala
- ✅ Docker Compose สำหรับ local development
- ✅ sbt-native-packager สำหรับ Docker images
- ✅ Kubernetes deployment, service, HPA
- ✅ GitHub Actions CI/CD pipeline

---

*[← Part 45: Logging](part-45-logging.md) | [Part 47: Testing Strategies →](part-47-testing-strategies.md)*
