# 12 — Infrastructure as Code (IaC): Terraform & Ansible

> **Goal:** Understand IaC — how to automate infrastructure provisioning and configuration for banking.

---

## 🔍 What is Infrastructure as Code?

**Infrastructure as Code** means defining and managing infrastructure (servers, networks, databases) through **code files** instead of manual configuration.

### The Problem IaC Solves

```
Without IaC:
  Day 1: "Provision 5 servers for payment service"
  → Ops team manually creates servers
  → Takes 2 days
  
  Day 30: "We need 5 more identical servers for DR"
  → "Which OS version? Which packages? Which config?"
  → "I don't remember exactly what we did"
  → Takes 1 week (and it's different from production!)
  
  Day 90: "Compliance audit — show us infrastructure config"
  → "It's in John's head, he's on vacation"
  → Audit failure

With IaC:
  terraform apply   # Creates 5 servers in 5 minutes
  terraform apply   # Creates identical DR servers in 5 minutes
  Git shows exact config for every change
  → Audit: "Here's the Git history of every infrastructure change"
```

---

## 🏗️ IaC Tool Landscape

| Tool | Type | What It Does |
|------|------|--------------|
| **Terraform** | Provisioning | Creates infrastructure (servers, networks, databases) |
| **Ansible** | Configuration | Installs software, configures servers |
| **CloudFormation** | Provisioning | AWS-specific infrastructure |
| **Pulumi** | Provisioning | IaC using programming languages |
| **Chef/Puppet** | Configuration | Server configuration management |

```
┌─────────────────────────────────────────────────────────────┐
│                    IaC WORKFLOW                              │
│                                                             │
│  Terraform                    Ansible                       │
│  (Provision)                  (Configure)                   │
│                                                             │
│  "Create servers"    ──▶     "Install software"            │
│  "Set up networking" ──▶     "Configure services"          │
│  "Provision databases"──▶    "Apply security policies"     │
│                                                             │
│  Provider-agnostic            Agentless                     │
│  Declarative                  Procedural                    │
│  State management             Idempotent                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Terraform Basics

### What is Terraform?
Terraform is an IaC tool that **provisions and manages** infrastructure across multiple cloud providers (AWS, Azure, GCP) and on-premise environments.

### Terraform Workflow
```
Write → Plan → Apply → Destroy
  │       │       │        │
  ▼       ▼       ▼        ▼
Define  Preview  Execute  Cleanup
infra   changes  changes  infra
```

### Example: Provision a Kubernetes Cluster on AWS (EKS)
```hcl
# main.tf - Terraform configuration
provider "aws" {
  region = "ap-south-1"  # Mumbai region
}

# VPC for the banking cluster
resource "aws_vpc" "banking_vpc" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "banking-vpc"
    Environment = "production"
    Compliance  = "pci-dss"
  }
}

# EKS Cluster
resource "aws_eks_cluster" "banking_cluster" {
  name     = "banking-cluster"
  role_arn = aws_iam_role.eks_role.arn
  
  vpc_config {
    subnet_ids = [
      aws_subnet.subnet_1.id,
      aws_subnet.subnet_2.id,
      aws_subnet.subnet_3.id,
    ]
  }
  
  version = "1.28"
}

# RDS Database
resource "aws_db_instance" "banking_db" {
  identifier     = "banking-db"
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.r5.large"
  
  allocated_storage     = 100
  max_allocated_storage = 500
  
  db_name  = "banking"
  username = "bank_admin"
  password = var.db_password  # From variables
  
  multi_az            = true
  storage_encrypted   = true
  deletion_protection = true
  
  tags = {
    Compliance = "pci-dss"
  }
}
```

### Terraform Commands
```bash
# Initialize (download providers)
terraform init

# Preview changes
terraform plan
# Plan: 15 to add, 0 to change, 0 to destroy.

# Apply changes
terraform apply
# Apply complete! Resources: 15 added, 0 changed, 0 destroyed.

# View state
terraform state list

# Destroy infrastructure
terraform destroy
```

---

## 🔧 Ansible Basics

### What is Ansible?
Ansible **configures** existing infrastructure — installs software, manages files, runs commands. It's agentless (uses SSH).

### Ansible Playbook
```yaml
# playbook.yml - Configure banking servers
---
- name: Configure Banking Application Servers
  hosts: banking_servers
  become: yes
  
  vars:
    java_version: "17"
    app_version: "2.3.1"
  
  tasks:
    - name: Install Java
      apt:
        name: openjdk-{{ java_version }}-jdk
        state: present
        update_cache: yes
    
    - name: Create application user
      user:
        name: bankapp
        system: yes
        shell: /bin/bash
    
    - name: Create application directory
      file:
        path: /opt/banking
        state: directory
        owner: bankapp
        group: bankapp
        mode: '0755'
    
    - name: Copy application JAR
      copy:
        src: files/payment-service-{{ app_version }}.jar
        dest: /opt/banking/payment-service.jar
        owner: bankapp
        mode: '0644'
      notify: restart payment service
    
    - name: Create systemd service
      template:
        src: templates/payment-service.service.j2
        dest: /etc/systemd/system/payment-service.service
      notify: restart payment service
    
    - name: Configure firewall
      ufw:
        rule: allow
        port: '8080'
        proto: tcp
    
    - name: Enable and start service
      systemd:
        name: payment-service
        state: started
        enabled: yes
  
  handlers:
    - name: restart payment service
      systemd:
        name: payment-service
        state: restarted
```

### Ansible Commands
```bash
# Run playbook
ansible-playbook -i inventory.ini playbook.yml

# Run specific task
ansible-playbook playbook.yml --tags "install-java"

# Dry run (check mode)
ansible-playbook playbook.yml --check

# Target specific hosts
ansible-playbook playbook.yml --limit "server-01,server-02"
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Multi-Cloud Infrastructure Provisioning
**Context:** Bank needs identical infrastructure in AWS (India) and Azure (Singapore) for disaster recovery.

```hcl
# AWS Infrastructure (India)
module "aws_india" {
  source = "./modules/banking-infra"
  
  providers = {
    aws = aws.mumbai
  }
  
  environment = "production"
  region      = "ap-south-1"
  cluster_name = "banking-india"
  db_instance  = "db.r5.large"
  node_count   = 6
}

# Azure Infrastructure (Singapore)
module "azure_singapore" {
  source = "./modules/banking-infra"
  
  providers = {
    azurerm = azurerm.singapore
  }
  
  environment = "production"
  region      = "southeastasia"
  cluster_name = "banking-sg"
  db_instance  = "Standard_D4s_v3"
  node_count   = 4
}
```

```bash
# Provision both environments
$ terraform apply -target=module.aws_india
$ terraform apply -target=module.azure_singapore

# Both environments are IDENTICAL (same code, same config)
# Audit: Git history shows every infrastructure change
```

### Scenario 2: Automated Server Hardening
**Context:** PCI-DSS requires specific security configurations on all servers.

```yaml
# hardening.yml - Security hardening playbook
---
- name: PCI-DSS Server Hardening
  hosts: all_banking_servers
  become: yes
  
  tasks:
    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd
    
    - name: Enforce password policy
      lineinfile:
        path: /etc/login.defs
        regexp: '^PASS_MAX_DAYS'
        line: 'PASS_MAX_DAYS   90'
    
    - name: Enable audit logging
      apt:
        name: auditd
        state: present
    
    - name: Configure audit rules
      copy:
        src: files/audit-rules.conf
        dest: /etc/audit/rules.d/banking.rules
      notify: restart auditd
    
    - name: Install fail2ban
      apt:
        name: fail2ban
        state: present
    
    - name: Configure fail2ban
      template:
        src: templates/jail.local.j2
        dest: /etc/fail2ban/jail.local
    
    - name: Verify encryption at rest
      command: dmsetup ls --target crypt
      register: encryption_check
      failed_when: encryption_check.stdout == ""
  
  handlers:
    - name: restart sshd
      service: name=sshd state=restarted
    - name: restart auditd
      service: name=auditd state=restarted
```

### Scenario 3: Infrastructure Drift Detection
**Context:** Detect when manual changes deviate from defined infrastructure.

```bash
# Terraform detects drift
$ terraform plan
# aws_eks_cluster.banking_cluster has changed
#   ~ instance_type = "m5.xlarge" -> "m5.2xlarge"
# Warning: Infrastructure drift detected!
# Someone manually changed the instance type

# Remediation options:
# Option 1: Accept the drift (update Terraform)
$ terraform apply  # Updates config to match reality

# Option 2: Revert to desired state
$ terraform apply  # Reverts manual change
# This is the banking standard — always revert to IaC state
```

**Automated Drift Detection (Scheduled):**
```yaml
# .github/workflows/drift-detection.yml
name: Infrastructure Drift Detection
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Plan
        run: |
          terraform init
          terraform plan -detailed-exitcode
          if [ $? -eq 2 ]; then
            echo "⚠️ Drift detected! Alerting team..."
            curl -X POST $SLACK_WEBHOOK -d '{"text":"Infrastructure drift detected!"}'
          fi
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Provision Multi-Region K8s Cluster with Terraform

**Context:** Create identical banking infrastructure in Mumbai and Singapore.

```hcl
# main.tf - Multi-region banking infrastructure

provider "aws" {
  region = "ap-south-1"
  alias  = "mumbai"
}

provider "aws" {
  region = "ap-southeast-1"
  alias  = "singapore"
}

# Mumbai Cluster
module "mumbai" {
  source = "./modules/banking-cluster"
  providers = {
    aws = aws.mumbai
  }
  
  cluster_name = "banking-mumbai"
  environment  = "production"
  node_count   = 6
  node_type    = "m5.xlarge"
  db_instance  = "db.r5.large"
  db_replicas  = 2
}

# Singapore Cluster
module "singapore" {
  source = "./modules/banking-cluster"
  providers = {
    aws = aws.singapore
  }
  
  cluster_name = "banking-singapore"
  environment  = "production"
  node_count   = 4
  node_type    = "m5.xlarge"
  db_instance  = "db.r5.large"
  db_replicas  = 1
}

# Outputs
output "mumbai_cluster_endpoint" {
  value = module.mumbai.cluster_endpoint
}

output "singapore_cluster_endpoint" {
  value = module.singapore.cluster_endpoint
}
```

```bash
# Initialize and plan
$ terraform init
$ terraform plan
# Plan: 45 to add, 0 to change, 0 to destroy.

# Apply (creates everything)
$ terraform apply
# Apply complete! Resources: 45 added, 0 changed, 0 destroyed.

# Verify
$ terraform state list | wc -l
# 45 resources tracked

# Check cluster status
$ kubectl get nodes --kubeconfig=mumbai-kubeconfig
# NAME            STATUS   ROLES    AGE
# node-mumbai-01  Ready    worker   5m
# node-mumbai-02  Ready    worker   5m
# node-mumbai-03  Ready    worker   5m
# node-mumbai-04  Ready    worker   5m
# node-mumbai-05  Ready    worker   5m
# node-mumbai-06  Ready    worker   5m
```

### E2E Example 2: Server Hardening with Ansible

**Context:** Apply PCI-DSS security hardening to 100 banking servers.

```yaml
# playbooks/hardening.yml
---
- name: PCI-DSS Security Hardening
  hosts: all_banking_servers
  become: yes
  vars:
    ssh_port: 22
    max_login_attempts: 3
    password_max_days: 90
    password_min_days: 7
    min_password_length: 16
  
  tasks:
    # SSH Hardening
    - name: Disable root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd
    
    - name: Set SSH protocol 2
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^Protocol'
        line: 'Protocol 2'
      notify: restart sshd
    
    - name: Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify: restart sshd
    
    # Password Policy
    - name: Set password max days
      lineinfile:
        path: /etc/login.defs
        regexp: '^PASS_MAX_DAYS'
        line: 'PASS_MAX_DAYS   {{ password_max_days }}'
    
    - name: Set minimum password length
      lineinfile:
        path: /etc/pam.d/common-password
        regexp: 'minlen='
        line: 'password requisite pam_pwquality.so minlen={{ min_password_length }}'
    
    # Audit Logging
    - name: Install auditd
      apt:
        name: auditd
        state: present
    
    - name: Configure audit rules
      copy:
        src: files/audit-rules.conf
        dest: /etc/audit/rules.d/banking.rules
      notify: restart auditd
    
    # File Permissions
    - name: Set restrictive permissions on sensitive files
      file:
        path: "{{ item }}"
        mode: '0600'
        owner: root
        group: root
      loop:
        - /etc/shadow
        - /etc/gshadow
        - /etc/ssh/sshd_config
    
    # Disable unnecessary services
    - name: Disable telnet
      service:
        name: telnet
        state: stopped
        enabled: no
    
    - name: Disable FTP
      service:
        name: vsftpd
        state: stopped
        enabled: no
    
  handlers:
    - name: restart sshd
      service: name=sshd state=restarted
    - name: restart auditd
      service: name=auditd state=restarted
```

```bash
# Run against 100 servers
$ ansible-playbook -i inventory/production.ini playbooks/hardening.yml

# PLAY RECAP
# server-01      : ok=12  changed=8    unreachable=0    failed=0
# server-02      : ok=12  changed=7    unreachable=0    failed=0
# ...
# server-100     : ok=12  changed=9    unreachable=0    failed=0

# Total: 100 servers hardened
# Failed: 0
# Duration: 15 minutes
# PCI-DSS compliance: ✅

# Verify compliance
$ ansible all_banking_servers -m shell -a "cat /etc/ssh/sshd_config | grep PermitRootLogin"
# server-01 | PermitRootLogin no
# server-02 | PermitRootLogin no
# ...
# server-100 | PermitRootLogin no
```

### E2E Example 3: Infrastructure Drift Detection & Remediation

**Context:** Detect when manual changes deviate from Terraform configuration.

```bash
# Scheduled drift detection (every 6 hours)
$ terraform plan -detailed-exitcode
# aws_eks_cluster.banking: has been changed
#   ~ instance_type = "m5.xlarge" -> "m5.2xlarge" (drift detected!)

# Alert sent to team
# Slack: ⚠️ Infrastructure drift detected in banking cluster
# Terraform: /infra/terraform/banking
# Resource: aws_eks_cluster.banking
# Change: instance_type m5.xlarge -> m5.2xlarge
# Detected: 2026-09-04 16:00:00

# Option 1: Accept drift (update Terraform)
$ sed -i 's/m5.xlarge/m5.2xlarge/' main.tf
$ terraform apply
# aws_eks_cluster.banking: Updating...
# Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

# Option 2: Revert to desired state (standard for banking)
$ git checkout main -- main.tf  # Revert manual change
$ terraform apply
# aws_eks_cluster.banking: Updating...
# Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
# Instance type reverted to m5.xlarge ✅

# Audit trail
$ git log --oneline --all -- main.tf
# a1b2c3d Revert unauthorized instance type change
# d4e5f6g Revert unauthorized instance type change  (drift)
# h7i8j9k Update instance type to m5.xlarge
# l0m1n2o Initial infrastructure provisioning
```

---

## 📋 Interview Questions

### Q1: What is the difference between Terraform and Ansible?
**Answer:** Terraform **provisions** infrastructure (creates servers, networks, databases). Ansible **configures** existing infrastructure (installs software, applies settings). They're complementary: Terraform creates the servers, Ansible configures them. Terraform is declarative (describe desired state), Ansible is procedural (describe steps). Terraform tracks state, Ansible is stateless.

### Q2: What is Terraform state and why is it critical?
**Answer:** Terraform state is a JSON file tracking which infrastructure Terraform manages. It maps real resources to configuration. Critical because: 

(1) **Change detection** — compares desired state vs actual state. 

(2) **Dependency tracking** — knows the order to create/destroy resources. 

(3) **Locking** — prevents concurrent modifications. 

For banking, store state in a remote backend (S3, Terraform Cloud) with encryption and locking.

### Q3: How do you handle secrets in Terraform?
**Answer:** Never hardcode secrets. 

Use: 

(1) **Variables** — pass via `terraform.tfvars` (gitignored). 

(2) **Environment variables** — `TF_VAR_db_password`. 

(3) **HashiCorp Vault** — dynamic secrets with Vault provider. 

(4) **AWS Secrets Manager** — `aws_secretsmanager_secret`. 

(5) **SOPS** — encrypt values files. For banking, always use a secrets manager — never store secrets in Git or Terraform state.

### Q4: What is IaC drift and how do you handle it?
**Answer:** Drift occurs when actual infrastructure differs from the IaC definition (manual changes, emergency fixes). Detection: `terraform plan` shows differences. 

Prevention: 

(1) **GitOps** — all changes go through Git. 

(2) **CI/CD** — automated infrastructure pipelines. 

(3) **Scheduled drift detection** — run `terraform plan` every 6 hours. 

(4) **Alert on drift** — notify team immediately. 

For banking, drift should trigger an incident.

### Q5: How do you implement IaC in a CI/CD pipeline?
**Answer:** 

(1) **Store Terraform code in Git** — version-controlled infrastructure. 

(2) **CI pipeline validates** — `terraform fmt`, `terraform validate`, `terraform plan`. 

(3) **PR review** — infrastructure changes go through code review. 

(4) **CD applies** — after approval, `terraform apply` runs in pipeline. 

(5) **State management** — remote backend with encryption and locking. 

(6) **Drift detection** — scheduled `terraform plan` in CI.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| IaC | Define infrastructure in code, not manual steps |
| Terraform | Provision infrastructure across clouds |
| Ansible | Configure servers, install software |
| State Management | Track infrastructure, detect drift |
| Secrets | Never commit, use external secret managers |
| Banking Relevance | Compliance, reproducibility, audit trails |

**Next:** [13-Monitoring-Observability.md](./13-Monitoring-Observability.md) — Learn how to monitor your CI/CD pipeline and production systems.
