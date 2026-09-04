# 06 — Docker Fundamentals: Containers for CI/CD

> **Goal:** Understand Docker — why containers revolutionized deployment and how they power CI/CD.

---

## 🐳 What is Docker?

**Docker** is a platform that packages applications and their dependencies into **lightweight, portable containers** that run consistently anywhere.

### The Problem Docker Solves

```
Without Docker:
  Developer: "It works on my machine!"
  Ops Team: "Well, it doesn't work on the server."
  Developer: "But my laptop has Python 3.9..."
  Ops Team: "The server has Python 3.6."
  Developer: "And I have these 15 libraries..."
  Ops Team: "Different versions here."
  → DAYS of debugging environment issues

With Docker:
  Developer builds a Docker image (includes EVERYTHING)
  Ops deploys that exact image
  → It works everywhere, every time, guaranteed
```

---

## 🏗️ Docker Architecture

```
┌─────────────────────────────────────────────────────┐
│                    YOUR MACHINE                      │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  Container 1 │  │  Container 2 │  │  Container 3 │ │
│  │  (Payment)   │  │  (Account)   │  │  (Notification)│ │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │ │
│  │  │  App   │  │  │  │  App   │  │  │  │  App   │  │ │
│  │  ├────────┤  │  │  ├────────┤  │  │  ├────────┤  │ │
│  │  │ Libs   │  │  │  │ Libs   │  │  │  │ Libs   │  │ │
│  │  ├────────┤  │  │  ├────────┤  │  │  ├────────┤  │ │
│  │  │ Runtime│  │  │  │ Runtime│  │  │  │ Runtime│  │ │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         └────────────────┼────────────────┘         │
│                          │                          │
│                  ┌───────┴───────┐                  │
│                  │  Docker Engine │                  │
│                  │  (Container   │                  │
│                  │   Runtime)    │                  │
│                  └───────┬───────┘                  │
│                          │                          │
│                  ┌───────┴───────┐                  │
│                  │  HOST OS      │                  │
│                  │  (Linux)      │                  │
│                  └───────────────┘                  │
└─────────────────────────────────────────────────────┘
```

### Image vs Container

| Concept | Analogy | What It Is |
|---------|---------|------------|
| **Image** | Blueprint / Recipe | Read-only template with app code + dependencies + config |
| **Container** | Built House | Running instance of an image (can have many from one image) |

---

## 📝 Dockerfile — Building Images

A **Dockerfile** is a script of instructions that tells Docker how to build an image.

### Example: Banking Application Dockerfile

```dockerfile
# Stage 1: Build the application
FROM maven:3.8-openjdk-17 AS build
WORKDIR /app

# Copy dependency definitions first (caching layer)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build the application
RUN mvn package -DskipTests

# Stage 2: Create the runtime image
FROM openjdk:17-jre-slim

# Create non-root user (security best practice)
RUN groupadd -r bankapp && useradd -r -g bankapp bankapp

WORKDIR /app

# Copy the built JAR from build stage
COPY --from=build /app/target/payment-service-*.jar app.jar

# Switch to non-root user
USER bankapp

# Expose the application port
EXPOSE 8080

# Health check for container orchestration
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Start the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Multi-Stage Build Explained

```
Stage 1 (Build):          Stage 2 (Runtime):
┌──────────────────┐      ┌──────────────────┐
│ Maven + JDK 17   │      │ JRE 17 only      │
│ Source code       │      │ Compiled JAR     │
│ Dependencies      │      │ (much smaller)   │
│ Build tools       │      │                  │
│                   │      │                  │
│ Size: ~800MB      │      │ Size: ~200MB     │
└────────┬─────────┘      └──────────────────┘
         │
         └── Copy only the JAR ──▶
```

---

## 🔧 Essential Docker Commands

### Building Images
```bash
# Build an image
docker build -t payment-service:v2.3.1 .

# Build with no cache (fresh build)
docker build --no-cache -t payment-service:v2.3.1 .

# List images
docker images
# REPOSITORY          TAG     IMAGE ID       SIZE
# payment-service     v2.3.1  abc123def456   210MB
# maven               3.8     def789abc012   800MB (build only)
```

### Running Containers
```bash
# Run a container
docker run -d -p 8080:8080 --name payment payment-service:v2.3.1

# Run with environment variables
docker run -d \
  -e DB_HOST=postgres.bank.com \
  -e DB_PASSWORD=secret \
  -e JAVA_OPTS="-Xmx512m" \
  -p 8080:8080 \
  --name payment \
  payment-service:v2.3.1

# View running containers
docker ps
# CONTAINER ID  IMAGE                    STATUS       PORTS
# abc123def456  payment-service:v2.3.1   Up 5 min     0.0.0.0:8080->8080

# View logs
docker logs -f payment
# 2026-09-04 10:30:00 INFO  PaymentService - Started on port 8080
# 2026-09-04 10:30:05 INFO  HealthCheck - Status: UP
```

### Managing Containers
```bash
# Stop a container
docker stop payment

# Remove a container
docker rm payment

# Execute command inside container
docker exec -it payment /bin/bash

# View container resource usage
docker stats payment
# CONTAINER   CPU %   MEM USAGE
# payment     2.5%    256MB / 512MB
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Consistent Development Environments
**Context:** 20 developers work on the same banking application with different OS and configurations.

**Without Docker:**
```
Developer 1: MacBook Pro, M2 chip, Java 17
Developer 2: Windows 11, Java 11 (different!)
Developer 3: Ubuntu, Java 17 (but different Maven version)
→ "Works on my machine" × 20
→ 2 days lost per developer per month on environment issues
```

**With Docker:**
```bash
# One command sets up the entire environment
$ docker-compose up -d

# This starts:
# - Payment service (Java 17)
# - Account service (Java 17)
# - PostgreSQL database
# - Redis cache
# - RabbitMQ message broker
# - LocalStack (AWS mock)

# EVERY developer has the SAME environment
# No more "works on my machine"
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  payment-service:
    build: ./payment-service
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  account-service:
    build: ./account-service
    ports:
      - "8081:8081"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: banking
      POSTGRES_USER: bank_user
      POSTGRES_PASSWORD: local_dev
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

### Scenario 2: CI Pipeline Docker Usage
**Context:** Each stage of the CI pipeline runs in its own Docker container.

```yaml
# Jenkins pipeline using Docker agents
pipeline {
    agent none
    
    stages {
        // Stage 1: Build (runs in Maven container)
        stage('Build') {
            agent {
                docker { image 'maven:3.8-openjdk-17' }
            }
            steps {
                sh 'mvn clean package -DskipTests'
                stash includes: 'target/*.jar', name: 'jar'
            }
        }
        
        // Stage 2: Unit Tests (runs in Maven container)
        stage('Unit Tests') {
            agent {
                docker { image 'maven:3.8-openjdk-17' }
            }
            steps {
                unstash 'jar'
                sh 'mvn test'
            }
        }
        
        // Stage 3: Security Scan (runs in Trivy container)
        stage('Security Scan') {
            agent {
                docker { image 'aquasec/trivy:latest' }
            }
            steps {
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL payment-service:latest'
            }
        }
        
        // Stage 4: Build Docker Image
        stage('Build Image') {
            agent any
            steps {
                sh 'docker build -t registry.bank.com/payment:$BUILD_NUMBER .'
                sh 'docker push registry.bank.com/payment:$BUILD_NUMBER'
            }
        }
    }
}
```

### Scenario 3: Production Container Deployment
**Context:** Deploy banking application as containers in Kubernetes.

**Image Lifecycle:**
```
Development → Staging → Production
    │            │           │
    ▼            ▼           ▼
payment:dev  payment:stg  payment:prod
  (latest)    (v2.3.0)    (v2.3.0)
```

```bash
# Build and tag for each environment
$ docker build -t registry.bank.com/payment:dev .
$ docker push registry.bank.com/payment:dev

# When ready for staging
$ docker tag registry.bank.com/payment:dev registry.bank.com/payment:v2.3.0-staging
$ docker push registry.bank.com/payment:v2.3.0-staging

# When approved for production
$ docker tag registry.bank.com/payment:dev registry.bank.com/payment:v2.3.0
$ docker push registry.bank.com/payment:v2.3.0
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Containerizing a Legacy Banking Application

**Context:** Migrate a 10-year-old Java banking application from physical servers to Docker.

```dockerfile
# Stage 1: Build (includes Maven, JDK)
FROM maven:3.8-openjdk-11 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime (minimal JRE)
FROM openjdk:11-jre-slim

# Security: Create non-root user
RUN groupadd -r banking && useradd -r -g banking banking

# Install only necessary libraries
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy built artifact
COPY --from=build /app/target/core-banking-*.jar app.jar

# Set permissions
RUN chown -R banking:banking /app
USER banking

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# Build and test
$ docker build -t core-banking:v3.0.0 .
$ docker run -d -p 8080:8080 --name banking core-banking:v3.0.0
$ docker exec banking curl http://localhost:8080/actuator/health
# {"status":"UP"} ✅

# Verify size reduction
$ docker images core-banking
# REPOSITORY    TAG      SIZE
# core-banking  v3.0.0   215MB  (was 1.2GB on physical server)

# Deploy to Kubernetes
$ kubectl create deployment core-banking --image=core-banking:v3.0.0
$ kubectl expose deployment core-banking --port=80 --target-port=8080
$ kubectl get pods
# NAME                          READY   STATUS    AGE
# core-banking-abc123-def456    1/1     Running   2m
```

### E2E Example 2: Multi-Container Banking Stack

**Context:** Set up complete banking development environment with Docker Compose.

```yaml
# docker-compose.yml - Complete Banking Stack
version: '3.8'

services:
  # Application Services
  payment-service:
    build: ./services/payment
    ports:
      - "8081:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - REDIS_HOST=redis
      - KAFKA_BROKER=kafka:9092
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - banking-network

  account-service:
    build: ./services/account
    ports:
      - "8082:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - banking-network

  notification-service:
    build: ./services/notification
    ports:
      - "8083:8080"
    environment:
      - KAFKA_BROKER=kafka:9092
      - SMTP_HOST=mailhog
    depends_on:
      - kafka
      - mailhog
    networks:
      - banking-network

  # Infrastructure
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: banking
      POSTGRES_USER: bank_user
      POSTGRES_PASSWORD: local_dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bank_user -d banking"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - banking-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - banking-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    ports:
      - "9092:9092"
    networks:
      - banking-network

  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - banking-network

volumes:
  postgres-data:

networks:
  banking-network:
    driver: bridge
```

```bash
# Start entire banking stack
$ docker-compose up -d
# Creating payment-service    ... done
# Creating account-service    ... done
# Creating notification-service ... done
# Creating postgres           ... done
# Creating redis              ... done
# Creating kafka              ... done
# Creating mailhog            ... done

# Verify all services
$ docker-compose ps
# NAME                  STATUS    PORTS
# payment-service       Up        0.0.0.0:8081->8080
# account-service       Up        0.0.0.0:8082->8080
# notification-service  Up        0.0.0.0:8083->8080
# postgres              Up (healthy) 0.0.0.0:5432->5432
# redis                 Up        0.0.0.0:6379->6379
# kafka                 Up        0.0.0.0:9092->9092
# mailhog               Up        0.0.0.0:1025,8025

# Run integration tests
$ docker-compose -f docker-compose.test.yml run tests
# 156 integration tests passed ✅
```

### E2E Example 3: Docker-Based CI Pipeline for Banking

**Context:** Each CI stage runs in its own Docker container for isolation.

```groovy
// Jenkinsfile - Docker-based pipeline
pipeline {
    agent none
    
    stages {
        // Build stage runs in Maven container
        stage('Build') {
            agent {
                docker { image 'maven:3.8-openjdk-17' }
            }
            steps {
                sh 'mvn clean compile'
            }
        }
        
        // Test stage runs in Maven container
        stage('Test') {
            agent {
                docker { image 'maven:3.8-openjdk-17' }
            }
            steps {
                sh 'mvn test'
                sh 'mvn jacoco:report'
            }
        }
        
        // Security scan runs in Trivy container
        stage('Security') {
            agent {
                docker { image 'aquasec/trivy:latest' }
            }
            steps {
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL payment-service:latest'
            }
        }
        
        // Build Docker image
        stage('Docker Build') {
            agent any
            steps {
                script {
                    docker.build('registry.bank.com/payment:latest')
                    docker.withRegistry('https://registry.bank.com', 'registry-creds') {
                        docker.image('registry.bank.com/payment:latest').push()
                    }
                }
            }
        }
        
        // Deploy runs in kubectl container
        stage('Deploy') {
            agent {
                docker { image 'bitnami/kubectl:latest' }
            }
            steps {
                sh 'kubectl set image deployment/payment payment=registry.bank.com/payment:latest -n production'
                sh 'kubectl rollout status deployment/payment -n production --timeout=300s'
            }
        }
    }
}
```

---

## 📋 Interview Questions

### Q1: What is the difference between a Docker image and a container?
**Answer:** 
A Docker image is a **read-only template** containing the application code, runtime, libraries, and dependencies. 

A container is a **running instance** of an image — it's like a house built from a blueprint. You can create multiple containers from one image, each running independently. 

In banking, the same image might run in development, staging, and production, ensuring consistency.

### Q2: Why use multi-stage builds in Docker?
**Answer:** Multi-stage builds separate the build process from the runtime. The build stage includes compilers, build tools, and source code (large image). The runtime stage includes only the compiled artifact and runtime (small image). 

Benefits: 

(1) **Smaller images** — production images are 4-5x smaller. 

(2) **Security** — build tools don't exist in production. 

(3) **Faster pulls** — smaller images deploy faster. 

Example: An 800MB build image becomes a 200MB runtime image.

### Q3: What is a Dockerfile and what are the most important instructions?
**Answer:** A Dockerfile is a text file with instructions to build a Docker image. 

Key instructions: 

`FROM` (base image), 

`WORKDIR` (working directory), 

`COPY` (copy files into image), 

`RUN` (execute commands during build), 

`EXPOSE` (document ports), 

`ENV` (set environment variables), 

`USER` (run as non-root for security), 

`HEALTHCHECK` (define health check), 

`ENTRYPOINT` (command to run when container starts).

### Q4: How do Docker containers handle networking?
**Answer:** Docker provides several network types: 

(1) **Bridge** — default, containers on same host can communicate. 

(2) **Host** — container shares host's network stack. 

(3) **Overlay** — multi-host networking (used in Docker Swarm). 

(4) **None** — no networking. In banking, containers typically use bridge networks with explicit port mappings. Kubernetes provides its own networking model (CNI plugins like Calico, Flannel).

### Q5: What are Docker security best practices for banking?
**Answer:** 

(1) **Non-root user** — never run containers as root. 

(2) **Minimal base images** — use Alpine or distroless images. 

(3) **No secrets in images** — use environment variables or secret management. 

(4) **Scan images** — use Trivy, Snyk, or Clair to scan for vulnerabilities. 

(5) **Read-only filesystem** — prevent runtime modifications. 

(6) **Resource limits** — prevent container from consuming all host resources. 

(7) **Image signing** — use Docker Content Trust to verify image integrity.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Docker Image | Read-only template with app + dependencies |
| Container | Running instance of an image |
| Dockerfile | Instructions to build an image |
| Multi-Stage Build | Smaller, more secure production images |
| Docker Compose | Define multi-container environments |
| Banking Relevance | Consistent environments, reproducible deployments |

**Next:** [07-Docker-Registry.md](./07-Docker-Registry.md) — Learn how Docker images are stored and managed.
