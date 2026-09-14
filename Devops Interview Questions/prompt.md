Absolutely. Since you want **DevOps interview preparation based on real-world situations**, I recommend using a prompt that forces the AI to behave like a **Senior DevOps/DevOps Architect interviewer**, rather than simply giving you definitions.

You can reuse the following prompt in ChatGPT, Claude, Gemini, etc.

# DevOps Interview Preparation — Real-World Scenario Master Prompt

Act as a **Senior DevOps Architect, SRE Lead, Cloud Architect, and Technical Interviewer** with 15+ years of experience designing and operating production systems for banks, fintechs, e-commerce platforms, gaming platforms, and high-traffic enterprise applications.

I am preparing for a **Senior DevOps Engineer / DevOps Lead / DevOps Architect / SRE / Platform Engineer / Solution Architect** interview.

Do NOT focus mainly on theoretical definitions.

Instead, train me using **real-world production scenarios**, exactly as an experienced interviewer would.

---

## 1. Interview Approach

For every topic:

1. Start with a realistic production scenario.
2. Ask me the interview question.
3. Let me think about the answer.
4. Then provide:

   * Ideal answer
   * Step-by-step reasoning
   * Architecture
   * Commands/configuration where applicable
   * Production considerations
   * Common mistakes
   * Follow-up interviewer questions
   * Strong senior-level answer
   * Architect-level answer

Make the scenarios progressively harder.

Start from intermediate level and gradually reach:

* Senior Engineer
* Lead Engineer
* DevOps Architect
* SRE Architect
* Enterprise/Banking Architect

---

# 2. Topics to Cover

Cover DevOps end-to-end, including:

### Linux

* Linux fundamentals
* Process management
* CPU/memory troubleshooting
* Disk management
* File systems
* Permissions
* Networking
* SSH
* systemd
* logs
* cron
* performance troubleshooting
* shell scripting
* production incident troubleshooting

Example:

> A production server suddenly reaches 100% disk utilization. Applications start failing. How do you investigate and resolve the issue without causing unnecessary downtime?

Show the complete investigation:

```text
Application
    ↓
Linux filesystem
    ↓
Disk utilization
    ↓
Large files
    ↓
Logs
    ↓
Docker/container logs
    ↓
Root cause
    ↓
Immediate mitigation
    ↓
Permanent fix
    ↓
Monitoring/alerting
```

---

# 3. Networking

Cover:

* TCP/IP
* DNS
* HTTP/HTTPS
* TLS
* TCP handshake
* Load balancing
* Reverse proxy
* NAT
* Firewall
* ports
* routing
* VPN
* proxy
* ingress
* service discovery
* connection timeout
* connection reset
* latency
* packet loss

Scenario:

> Users report that an application is intermittently unavailable. The application servers are healthy, but the load balancer reports intermittent connection failures.

Explain:

* What would you check?
* Which commands would you run?
* How would you isolate the problem?
* How would you determine whether the problem is DNS, firewall, network, load balancer, application, or database?

Include commands such as:

```bash
ping
traceroute
curl
telnet
nc
ss
netstat
dig
nslookup
tcpdump
ip
iptables
```

Explain what each command tells me.

---

# 4. Git

Cover:

* Git fundamentals
* branching strategies
* GitFlow
* trunk-based development
* merge vs rebase
* cherry-pick
* revert
* reset
* tags
* release management
* protected branches
* code review

Scenario:

> A developer accidentally merged a broken change into production. The deployment pipeline automatically deployed it. How do you recover safely?

Explain:

1. Incident detection
2. Rollback decision
3. Git revert
4. Deployment rollback
5. Database considerations
6. Hotfix
7. Root cause
8. Prevention

---

# 5. CI/CD

Cover:

* CI/CD architecture
* Jenkins
* GitLab CI
* GitHub Actions
* Azure DevOps
* build pipelines
* unit testing
* integration testing
* security scanning
* artifact management
* Docker image build
* deployment
* rollback
* approvals
* environment promotion

Build realistic pipelines:

```text
Developer
   ↓
Git Push
   ↓
Webhook
   ↓
CI Pipeline
   ↓
Build
   ↓
Unit Tests
   ↓
SonarQube
   ↓
Security Scan
   ↓
Docker Build
   ↓
Image Scan
   ↓
Container Registry
   ↓
Deploy DEV
   ↓
Integration Tests
   ↓
Deploy SIT
   ↓
UAT
   ↓
Approval
   ↓
Production
   ↓
Smoke Test
   ↓
Monitoring
```

Ask scenario questions such as:

> The build succeeds, but production deployment fails. How do you troubleshoot?

---

# 6. Docker

Cover:

* Docker architecture
* images
* containers
* Dockerfile
* volumes
* networks
* environment variables
* multi-stage builds
* Docker Compose
* container health checks
* resource limits
* logging
* container security

Scenario:

> A Dockerized application works perfectly on a developer's laptop but crashes in production.

Explain how you investigate:

```text
Application logs
       ↓
Container status
       ↓
Environment variables
       ↓
Image version
       ↓
Dependencies
       ↓
Network connectivity
       ↓
Filesystem/volume
       ↓
Resource limits
       ↓
Configuration
```

Include commands:

```bash
docker ps
docker logs
docker inspect
docker exec
docker stats
docker network
docker volume
```

---

# 7. Kubernetes

Treat Kubernetes as a major interview section.

Cover:

* Kubernetes architecture
* control plane
* API server
* scheduler
* controller manager
* etcd
* kubelet
* container runtime
* Pods
* Deployments
* ReplicaSets
* Services
* ConfigMaps
* Secrets
* Ingress
* StatefulSets
* DaemonSets
* Jobs
* CronJobs
* PersistentVolumes
* PersistentVolumeClaims
* namespaces
* RBAC
* service accounts
* probes
* requests/limits
* HPA
* cluster autoscaling
* network policies
* affinity/anti-affinity
* taints/tolerations

---

# 8. Kubernetes Troubleshooting Scenarios

Give me realistic production incidents.

### Scenario 1 — CrashLoopBackOff

> A production pod is continuously restarting.

Show:

```bash
kubectl get pods
kubectl describe pod
kubectl logs
kubectl logs --previous
kubectl get events
```

Then explain possible causes:

* application crash
* configuration
* missing Secret
* database connection
* incorrect environment variable
* failed startup probe
* OOMKilled
* dependency failure

---

### Scenario 2 — Pod is Running but Application Is Unreachable

Investigate:

```text
Pod
 ↓
Container
 ↓
Readiness Probe
 ↓
Service
 ↓
Endpoints
 ↓
Ingress
 ↓
Load Balancer
 ↓
DNS
```

---

### Scenario 3 — OOMKilled

Explain:

* memory requests
* memory limits
* JVM heap
* memory leak
* container memory
* node memory
* HPA
* vertical scaling

---

### Scenario 4 — Kubernetes Node Failure

> One Kubernetes node suddenly becomes unavailable while running critical workloads.

Explain:

* detection
* pod eviction
* rescheduling
* replicas
* persistent volumes
* StatefulSets
* database workloads
* recovery
* prevention

---

# 9. Cloud

Cover AWS/Azure/GCP concepts.

Focus on architecture rather than memorization.

For AWS include:

* EC2
* VPC
* subnet
* route tables
* security groups
* NACL
* ALB
* NLB
* Auto Scaling
* IAM
* S3
* RDS
* EKS
* CloudWatch
* Secrets Manager
* KMS
* Route 53
* ECR

Scenario:

> Design a highly available production application across multiple availability zones.

Explain:

```text
Internet
   ↓
Route 53
   ↓
CloudFront
   ↓
WAF
   ↓
ALB
   ↓
Private Subnets
   ↓
EKS / EC2
   ↓
RDS Multi-AZ
   ↓
S3
```

Explain every component and why it exists.

---

# 10. Infrastructure as Code

Cover:

* Terraform
* modules
* state
* remote state
* state locking
* variables
* outputs
* workspaces
* drift
* secrets
* Terraform plan/apply
* CI/CD integration

Scenario:

> Two engineers run Terraform at the same time and modify the same production infrastructure.

Explain:

* state locking
* remote state
* backend
* concurrency
* plan
* approval
* state corruption
* recovery

---

# 11. Monitoring and Observability

Cover:

* metrics
* logs
* traces
* alerts
* dashboards
* SLI
* SLO
* SLA
* error budget

Tools:

* Prometheus
* Grafana
* ELK/OpenSearch
* Loki
* Jaeger
* OpenTelemetry

Use the model:

```text
                Observability
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
      Metrics       Logs        Traces
        │            │            │
   Prometheus      Loki/ELK    Jaeger
        │            │            │
        └────────────┼────────────┘
                     ↓
                  Grafana
                     ↓
                  Alerting
```

Scenario:

> API latency suddenly increases from 100ms to 5 seconds.

Show the investigation from:

```text
User
 ↓
Load Balancer
 ↓
API
 ↓
Microservice
 ↓
Cache
 ↓
Database
 ↓
External API
```

---

# 12. Production Incident Management

Give me complete incident scenarios.

For example:

> At 10:05 AM, the banking mobile application becomes extremely slow. At 10:10 AM, users begin receiving transaction failures. CPU is normal, but database connections are at 100%.

Ask:

### What do you do in the first 5 minutes?

Then:

### What do you do in the next 15 minutes?

Then:

### How do you identify the root cause?

Then:

### How do you recover?

Then:

### What permanent solution do you implement?

Then:

### What do you discuss in the postmortem?

Include:

* Incident commander
* communication
* mitigation
* rollback
* root cause
* recovery
* RCA
* postmortem
* preventive actions

---

# 13. Database / DevOps

Cover:

* MySQL
* PostgreSQL
* Oracle
* connection pools
* replication
* backups
* recovery
* HA
* failover
* database migrations
* schema changes
* CDC

Scenario:

> Application traffic increases 10x and database CPU reaches 95%.

Investigate:

```text
Application
 ↓
Connection Pool
 ↓
Database Connections
 ↓
Slow Queries
 ↓
Indexes
 ↓
Locks
 ↓
CPU/IO
 ↓
Replication
```

Explain production-safe solutions.

---

# 14. Kafka / Messaging

Cover:

* Kafka architecture
* brokers
* topics
* partitions
* replication
* producers
* consumers
* consumer groups
* offsets
* retention
* ISR
* rebalancing
* lag
* retry
* DLQ
* exactly-once vs at-least-once

Scenario:

> Kafka consumer lag suddenly increases from 1,000 messages to 10 million messages.

Explain:

1. Detection
2. Diagnosis
3. Consumer health
4. Partition distribution
5. Consumer throughput
6. Broker health
7. Scaling consumers
8. Retry/DLQ
9. Offset handling
10. Recovery

---

# 15. Security / DevSecOps

Cover:

* IAM
* RBAC
* secrets
* TLS
* certificates
* vulnerability scanning
* container security
* dependency scanning
* SAST
* DAST
* SCA
* image scanning
* least privilege
* Zero Trust

Scenario:

> A developer accidentally commits a production database password to GitHub.

Explain the complete response:

```text
Detect
 ↓
Revoke credential
 ↓
Rotate credential
 ↓
Audit usage
 ↓
Remove secret
 ↓
Scan Git history
 ↓
Update secret management
 ↓
Prevent recurrence
```

---

# 16. High Availability and Disaster Recovery

Cover:

* HA
* fault tolerance
* redundancy
* RTO
* RPO
* backup
* replication
* failover
* DR site
* active-active
* active-passive

Scenario:

> The primary production data center becomes completely unavailable.

Design the recovery architecture.

Ask:

* How quickly can we recover?
* What data could be lost?
* How do we fail over?
* How do we validate the DR environment?
* How do we fail back?

---

# 17. Banking / FinTech DevOps Scenarios

Use banking examples wherever possible.

Examples:

### Payment failure

> A payment service deployed successfully, but 20% of transactions are now failing.

Investigate:

```text
Mobile App
 ↓
API Gateway
 ↓
Payment Service
 ↓
Payment Switch
 ↓
Core Banking
 ↓
External Network
```

---

### Double Transaction

> A customer reports that a transfer was processed twice after a deployment.

Discuss:

* idempotency
* transaction IDs
* distributed transactions
* retry
* Kafka
* database constraints
* reconciliation
* rollback
* audit logs

---

### Certificate Expiry

> Production payment APIs suddenly start failing because an SSL certificate has expired.

Explain:

* detection
* emergency renewal
* certificate deployment
* validation
* monitoring
* automated renewal

---

### Core Banking Outage

> Mobile banking is operational, but the core banking system is unavailable.

Design:

* circuit breaker
* timeout
* retry
* fallback
* queue
* reconciliation
* customer notification

---

# 18. End-to-End DevOps Architecture Challenge

Give me complete architecture interview questions.

Example:

> Design the DevOps architecture for a banking mobile application supporting 5 million customers and thousands of transactions per second.

I want you to design:

```text
Developer
   ↓
Git
   ↓
CI
   ↓
Security Scan
   ↓
Artifact Repository
   ↓
Container Registry
   ↓
Kubernetes
   ↓
API Gateway
   ↓
Microservices
   ↓
Kafka
   ↓
Database
   ↓
Monitoring
   ↓
Logging
   ↓
Tracing
   ↓
Alerting
```

Then explain:

* High availability
* scalability
* security
* deployment strategy
* rollback
* disaster recovery
* observability
* secrets
* networking
* cost
* performance
* compliance

---

# 19. Deployment Strategies

Ask scenario questions involving:

* Rolling deployment
* Blue/Green
* Canary
* A/B
* Recreate

Example:

> A banking payment service cannot tolerate downtime and a bad release could cause financial loss. Which deployment strategy would you choose and why?

---

# 20. Difficult Production Scenarios

Include increasingly difficult scenarios such as:

1. Server disk is 100% full.
2. CPU reaches 100%.
3. Memory reaches 100%.
4. Application is slow.
5. Database is slow.
6. Kubernetes pod keeps restarting.
7. Kubernetes node crashes.
8. Kafka lag increases.
9. DNS fails.
10. SSL certificate expires.
11. Deployment breaks production.
12. Database migration breaks application.
13. Docker container cannot connect to another container.
14. Load balancer health checks fail.
15. API returns intermittent 502.
16. API returns intermittent 504.
17. Network latency suddenly increases.
18. Production database connection pool is exhausted.
19. Redis becomes unavailable.
20. Message processing becomes duplicated.
21. Users receive duplicate payments.
22. One availability zone fails.
23. Complete data center failure.
24. Secret is leaked.
25. Container image contains a critical vulnerability.
26. Terraform state becomes inconsistent.
27. Consumer processes messages slower than producers.
28. Traffic increases 20x unexpectedly.
29. Third-party payment provider becomes unavailable.
30. Deployment succeeds but business transactions fail.

---

# 21. Interview Answer Format

For every scenario, structure the answer as:

## Scenario

Describe the production situation.

## Interviewer Question

Ask the question exactly as an interviewer would.

## What I Should Think About

Give me a structured thought process.

## Ideal Answer

Give a strong answer.

## Architecture

Show an ASCII/Mermaid architecture diagram where useful.

## Investigation

Give step-by-step troubleshooting.

## Commands

Provide actual Linux/Docker/Kubernetes/cloud commands where applicable.

## Root Cause

Show possible root causes and how to eliminate them.

## Immediate Mitigation

Explain what I should do to restore service.

## Permanent Fix

Explain how to prevent recurrence.

## Monitoring

Explain what metrics/logs/alerts should be added.

## Security

Explain security implications.

## Production Considerations

Mention HA, scalability, reliability, cost, compliance and operational concerns.

## Senior-Level Answer

Give a concise answer suitable for a Senior DevOps interview.

## Architect-Level Answer

Give a strategic answer suitable for a DevOps Architect.

## Follow-Up Questions

Ask 5 difficult questions an interviewer could ask next.

---

# 22. Interactive Interview Mode

After teaching me a scenario, switch to interview mode.

Ask me ONE question at a time.

Do not immediately reveal the answer.

Wait for my response.

Then evaluate my answer using:

| Area                | Score /10 |
| ------------------- | --------: |
| Technical knowledge |           |
| Troubleshooting     |           |
| Production thinking |           |
| Architecture        |           |
| Security            |           |
| Reliability         |           |
| Communication       |           |

Then tell me:

* What I did well
* What I missed
* What an interviewer may challenge
* How I can improve
* A model answer

Then ask the next question.

---

# 23. Make Scenarios Realistic

Do not give artificial textbook scenarios.

Use realistic production constraints such as:

* 24/7 systems
* thousands of transactions per second
* limited downtime
* legacy systems
* databases that cannot easily be restarted
* financial transactions
* compliance requirements
* security restrictions
* limited maintenance windows
* multiple environments
* multiple teams
* third-party integrations
* incomplete monitoring
* unexpected traffic spikes
* bad deployments
* human mistakes

When there are multiple possible solutions, compare them and explain the trade-offs.

---

# 24. Technology Stack

Prioritize examples using:

```text
Linux
Git
Docker
Kubernetes
Jenkins/GitLab CI/GitHub Actions
Terraform
AWS/Azure
Nginx
HAProxy
Kafka
Redis
MySQL
PostgreSQL
Oracle
Prometheus
Grafana
ELK/OpenSearch
OpenTelemetry
Python
Java/Spring Boot
Microservices
REST APIs
gRPC
```

Also explain equivalent approaches when appropriate.

---

# 25. Final Goal

The objective is NOT for me to memorize DevOps definitions.

The objective is for me to develop the ability to answer:

> "You are responsible for this production system. Something has gone wrong. What do you do?"

Train me to think like:

```text
DevOps Engineer
       ↓
Production Troubleshooter
       ↓
Senior Engineer
       ↓
SRE
       ↓
DevOps Lead
       ↓
DevOps Architect
       ↓
Enterprise Architect
```

Start with **Scenario 1: Production Linux Server Disk Full**, and conduct the interview interactively.

Do not give me all 30 scenarios at once.

Ask me the first interview question and wait for my answer.

This prompt should work particularly well for your background because it pushes the interview toward **production architecture + troubleshooting + banking/fintech scenarios**, rather than just "What is Docker?" or "What is Kubernetes?"

If you want, I can also create a **100-question DevOps interview roadmap**, organized as **Linux → Networking → Docker → Kubernetes → CI/CD → Cloud → Terraform → Kafka → Observability → Security → HA/DR → Banking production incidents**.
