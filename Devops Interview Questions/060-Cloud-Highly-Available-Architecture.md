# 60. Design Highly Available Production Architecture on AWS

## Scenario

Your company is a digital bank launching in a new region. The product owner wants to serve 5 million users with a web application, a REST API, and a relational database. The CTO wants a highly available architecture on AWS with zero-downtime deployments, disaster recovery, and PCI compliance readiness. You're the DevOps architect. You need to design the full AWS architecture including VPC, subnets, load balancers, compute, database, CDN, DNS, and security — and be ready to defend every decision in a design review with the security team.

## Interviewer Question

"Design a highly available production architecture for a banking application serving 5 million users on AWS. Cover VPC design, subnets across availability zones, ALB, EC2 or EKS, RDS Multi-AZ, S3, CloudFront, Route53, and security. Include an ASCII diagram. Then explain the tradeoffs in your design."

## What I Should Think About

- Multi-AZ design is non-negotiable for HA — 3 AZs ideal, 2 minimum
- Every tier must be fault-tolerant: web, API, database, networking
- PCI compliance adds security requirements: encryption, VPC isolation, logging
- Route53 for DNS failover and routing policies
- ALB for layer-7 routing, health checks, auto-scaling integration
- RDS Multi-AZ for database HA, RDS Proxy for connection pooling
- S3 versioning + lifecycle for object storage, CloudFront for edge caching
- Security groups (stateful) vs NACLs (stateless) — both have roles
- Private subnets for application and database tiers
- NAT Gateway for outbound from private subnets
- Cost considerations: NAT Gateways, ALB, data transfer costs

## Ideal Answer

**Design Overview:**

1. **VPC**: 3 AZs, each with public, private (app), private (data), and private (db) subnets. CIDR 10.0.0.0/16 for 65,536 IPs.

2. **Route53**: Geolocation/weighted routing for web traffic, latency-based for API. Health checks on ALB endpoints.

3. **CloudFront**: CDN for static assets (JS, CSS, images), with S3 origin for static content and ALB origin for dynamic API calls.

4. **ALB**: Two ALBs — one for web (HTTPS), one for API. TLS termination at ALB using ACM certificates.

5. **Compute**: EKS for containerized services, or EC2 Auto Scaling Groups with ASG. Across 3 AZs. Uses Spot for non-critical, On-Demand for critical workloads.

6. **RDS**: MySQL/PostgreSQL with Multi-AZ (2 standby replicas), automated backups, RDS Proxy for connection pooling, and a cross-region read replica for DR.

7. **S3**: Web static assets, backups with versioning, lifecycle policies to Glacier for archives, cross-region replication for DR.

8. **Security**: Security groups (allow only necessary ports from specific sources), NACLs at subnet level, AWS WAF on ALB, Shield for DDoS protection, KMS for encryption at rest, CloudTrail for audit.

**Diagram:**

```
                    ┌─────────────────────────────────────────┐
                    │               AWS Route 53               │
                    │  (geolocation/latency, health checks)    │
                    └──────────────┬──────────────────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            ▼                                              ▼
   ┌─────────────────┐                            ┌─────────────────┐
   │   CloudFront    │ (static + dynamic)         │   CloudFront    │
   │  CDN - cache    │                            │  for API        │
   └────────┬────────┘                            └────────┬────────┘
            │                                              │
            ▼                                              ▼
   ┌─────────────────┐                            ┌─────────────────┐
   │     ALB Web     │                            │     ALB API     │
   │  (TLS term)     │                            │  (TLS term)     │
   └────────┬────────┘                            └────────┬────────┘
            │                                              │
   ┌────────┴───────────────────────────────────────────────┴──────┐
   │                         EKS Cluster                           │
   │   ┌────────────┐  ┌────────────┐  ┌────────────┐              │
   │   │  AZ A       │  │  AZ B       │  │  AZ C       │           │
   │   │ Web + API   │  │ Web + API   │  │ Web + API   │           │
   │   │ ngroups     │  │ NGroups     │  │ NGroups     │           │
   │   └────────────┘  └────────────┘  └────────────┘              │
   │         │                │               │                    │
   │         └────────────────┼───────────────┘                    │
   │                          ▼ RDS Proxy                         │
   │            ┌───────────────────────────┐                     │
   │            │   RDS Multi-AZ (3 AZs)    │                     │
   │            │   DB Cluster (Writer)     │                     │
   │            │   + 2 Read Replicas       │                     │
   │            └───────────────────────────┘                     │
   └──────────────────────────────────────────────────────────────┘
                 │
                 ▼
   ┌──────────────────────────────────────────────────────────┐
   │  S3: static assets, backups (CRR to DR region)          │
   │  ElastiCache Redis: session + cache                     │
   │  Secrets Manager: DB credentials                        │
   │  KMS: encryption keys                                   │
   └──────────────────────────────────────────────────────────┘
```

**Justification of choices:**
- 3 AZs rather than 2: survives a full AZ outage without degraded capacity
- EKS vs EC2: containerization enables faster deploys, better resource utilization
- ALB vs NLB: layer-7 routing needed for path-based routing to multiple services
- RDS Multi-AZ vs self-managed: Less operational overhead, auto failover
- CloudFront: reduces latency globally, reduces S3 and ALB load

## Architecture

```
     Production Architecture - Digital Bank
     ─────────────────────────────────────────

     Route53 (DNS, Health Checks)
        ●
        │  DNS
        ▼
    CloudFront (CDN, WAF, TLS)
        ●
        │
        ▼
       ALB (Web + API, WAF rules)
        ●
        │
   EKS Cluster (Containerized Services)
        ├── Karpenter/Autoscaling
        ├── HPA for Pod scaling
        ├── Cluster Autoscaler for Nodes
        │
        ▼
    RDS Proxy → RDS Multi-AZ PostgreSQL
    ElastiCache Redis (Cache/Session/Queue)
    S3 (Static + Backups + Versioning)
    Secrets Manager (DB creds, API keys)
    CloudWatch (Metrics, Logs, Alarms, Dashboards)
    CloudTrail (API audit logging)
    KMS (Encryption keys)
```

## Investigation

1. **Capacity planning**: Estimate peak concurrent users (500k concurrent → ~1000 pods)
2. **Database sizing**: 5M users → estimate DB size, recommend RDS instance sizing (e.g., db.r6g.4xlarge)
3. **Network design**: VPC CIDR allocation, subnet sizing for scale
4. **Security review**: Ensure SG rules are minimal, WAF rules configured
5. **Cost estimation**: Monthly cost for all components, identify optimization opportunities
6. **Compliance checklist**: PCI DSS requirements — encryption, logging, monitoring
7. **DR strategy**: RTO/RPO targets and replication strategy
8. **Autoscaling behavior**: Test ASG and HPA scaling policies
9. **Backup validation**: Test restore procedures
10. **Performance testing**: Load test the architecture before go-live

## Commands

```bash
# Create VPC with 3 AZs
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Production-VPC}]'

# Create subnets (one per AZ)
aws ec2 create-subnet \
  --vpc-id vpc-xxxxxxxx \
  --cidr-block 10.0.0.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=web-1a}]'

# Create NAT Gateways (one per AZ for HA)
aws ec2 create-nat-gateway \
  --subnet-id subnet-xxxxxxxx \
  --allocation-id eipalloc-xxxxxxxx

# Create RDS Multi-AZ cluster
aws rds create-db-cluster \
  --db-cluster-identifier production-db \
  --engine aurora-postgresql \
  --engine-version 14.6 \
  --db-subnet-group-name production-db-subnets \
  --vpc-security-group-ids sg-xxxxxxxx \
  --availability-zones us-east-1a us-east-1b us-east-1c \
  --backup-retention-period 30

# Create ALB
aws elbv2 create-load-balancer \
  --name production-alb \
  --subnets subnet-1a subnet-1b subnet-1c \
  --security-groups sg-xxxxxxxx \
  --scheme internet-facing \
  --type application

# Create EKS cluster
aws eks create-cluster \
  --name production \
  --role-arn arn:aws:iam::xxx:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-1a,subnet-1b,subnet-1c

# Create S3 bucket with versioning
aws s3api create-bucket --bucket production-static-assets --region us-east-1
aws s3api put-bucket-versioning --bucket production-static-assets --versioning-configuration Status=Enabled

# Create CloudFront distribution
aws cloudfront create-distribution \
  --distribution-config file://cloudfront-config.json

# Validate security groups
aws ec2 describe-security-groups --group-ids sg-xxxxxxxx --output json
```

## Root Cause (of design failures in similar architectures)

| Failure Mode | Cause | Prevention |
|---|---|---|
| Single-ZD failure | App running in 1 AZ | Multi-AZ deployment across 3 AZs |
| DB failover outage | No connection pool | RDS Proxy for connection pooling |
| Traffic spike overload | No autoscaling | ASG + HPA + load testing |
| DNS failover delay | Pointing to single IP | Route53 health checks → failover |
| SSL renewal issue | No automation | ACM auto-renewal |
| Data loss | No cross-region replication | S3 CRR + RDS cross-region replica |

## Immediate Mitigation (if an AZ fails)

1. Route53 health checks detect ALB failure and fail over to remaining AZs
2. RDS Multi-AZ automatically fails over to standby in another AZ
3. EKS node groups auto-scaling replaces lost nodes
4. ASG terminates unhealthy instances and launches new ones in healthy AZs

## Permanent Fix

1. **Regular DR drills** — test AZ failure simulation quarterly
2. **Chaos engineering** — use Chaos Monkey or AWS FIS (Fault Injection Simulator) to test resilience
3. **Capacity check** — ensure remaining capacity can handle full load during AZ outage
4. **Documentation** — runbook for each failure scenario
5. **Autoscaling validation** — test ASG scaling under load
6. **Cross-region DR** — add read replica and backup replication to DR region

## Monitoring

```yaml
# CloudWatch alarms
- alert: ALB5xxErrors
  expr: AWS/ApplicationELB 5xxErrorCount
  for: 5m
  severity: critical

- alert: RDSConnectionCount
  expr: AWS/RDS DatabaseConnections > 80%
  for: 5m
  severity: warning

- alert: EKSNodeCount
  expr: AWS/EKS node_count
  for: 5m
  severity: warning

- alert: CPUHigh
  expr: AWS/EC2 CPUUtilization > 80%
  for: 10m
  severity: warning
```

CloudWatch dashboards: ALB request counts, 5xx, latency, DB connections, node count, pods pending, NAT gateway bytes.

## Security

- **Network segmentation**: Public (web), Private (app), Private (data/db) tiers
- **Security groups**: Allow only ports from specific sources (443 from LB, 3306 from app SG)
- **NACLs**: Stateless defense at subnet level
- **WAF**: OWASP rules on ALB and CloudFront
- **Shield Advanced**: DDoS protection
- **KMS**: Encrypt EBS, RDS, S3 data at rest
- **CloudTrail + GuardDuty**: Audit and threat detection
- **IAM**: Least privilege roles, temporary credentials via STS
- **S3 Block Public Access**: Enable by default on all buckets
- **Secrets Manager**: No secrets in code or environment variables

## Production Considerations

- **Cost**: ~$8,000-$12,000/month baseline for this architecture (3 AZs, EKS, RDS HA)
- **Cost optimization**: Spot instances for stateless workloads, RDS Reserved Capacity, S3 lifecycle to reduce storage costs
- **Compliance**: PCI DSS requires quarterly QSA assessment, AWS Artifact for compliance documents
- **Data residency**: If processing EU data, use eu-west-1 with data sovereignty considerations
- **Operational**: Separate test/prod environments, automated deployments (Blue/Green), on-call rotation

## Senior-Level Answer

"I'd start with the 3-tier architecture spread across 3 AZs. VPC with a /16 CIDR split into public, app, data, and db subnets per AZ. Route53 with health-check-based routing, CloudFront for CDN and edge TLS, ALBs for web and API with WAF. Compute via EKS with managed node groups across AZs, Karpenter or Cluster Autoscaler for scaling, and HPA for pod autoscaling. RDS (Aurora) with a Multi-AZ cluster, RDS Proxy for connection pooling to absorb failover. S3 with versioning and cross-region replication for DR. ElastiCache Redis for sessions and caching. Security: SGs for instance-level, NACLs for subnet-level, KMS for encryption, IAM least privilege, CloudTrail for audit. The key decision points are the tradeoffs between cost, complexity, and resilience — 3 AZs is the sweet spot for a banking app."

## Architect-Level Answer

"The reference architecture must be built on four pillars: resilience, scalability, security, and cost optimization. For resilience, everything runs in at least 3 AZs with automated failover. For scalability, every tier scales independently — CloudFront for edge, ALB for routing, EKS for compute, RDS/Proxy for data. For security, defense-in-depth with VPC isolation, SGs, WAF, KMS, and GuardDuty. For cost, Spot for stateless, RDS reserved, S3 lifecycle. I'd add an Infrastructure-as-Code layer (Terraform) to make the architecture reproducible, and a disaster recovery plan with defined RTO/RPO targets. The design should also consider compliance requirements from day one — PCI DSS controls influence everything from logging to firewall rules. Finally, I'd implement observability as a first-class concern: CloudWatch metrics, traces via X-Ray, and logs centralized for audit."

## Follow-Up Questions

1. "What happens if an entire AWS region fails? Walk through your disaster recovery plan including RPO and RTO."
2. "How would you handle the Lambda vs EC2 vs EKS decision for this workload?"
3. "Explain the difference between a Network ACL and a Security Group, and when misconfiguration of each would cause an outage."
4. "How do you protect against credential theft and what's your incident response plan if an IAM key is compromised?"
5. "Estimate the monthly cost of this architecture and identify the top 3 cost optimization opportunities."