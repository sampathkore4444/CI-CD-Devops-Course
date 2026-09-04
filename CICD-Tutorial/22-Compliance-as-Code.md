# 22 — Compliance as Code: Automated Regulatory Compliance

> **Goal:** Automate compliance checks — how banks enforce PCI-DSS, GDPR, RBI, and SOX in CI/CD.

---

## 🔍 What is Compliance as Code?

**Compliance as Code** means writing automated checks that verify regulatory requirements in every deployment. Instead of manual audits, compliance is continuously enforced.

```
Traditional Compliance:
  Annual audit → Manual review → Findings → Remediation → Re-audit
  (Takes 3-6 months, expensive, one-time snapshot)

Compliance as Code:
  Every deployment → Automated checks → Pass/Fail → Real-time compliance
  (Continuous, fast, always current)
```

---

## 🏗️ Banking Regulatory Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BANKING COMPLIANCE FRAMEWORK                      │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  PCI-DSS    │  │  GDPR       │  │  RBI        │               │
│  │  (Card Data)│  │  (Privacy)  │  │  (India)    │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  SOX        │  │  Basel III   │  │  ISO 27001  │               │
│  │  (Financial)│  │  (Capital)   │  │  (Security) │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Compliance Checks in CI/CD

```yaml
# .gitlab-ci.yml - Compliance pipeline
stages:
  - build
  - compliance
  - deploy

# PCI-DSS Checks
pci-dss-check:
  stage: compliance
  script:
    # Requirement 3: Protect stored cardholder data
    - python compliance/pci_dss/req3_card_data_encryption.py
    # ✅ All card numbers encrypted at rest (AES-256)
    
    # Requirement 4: Encrypt transmission of cardholder data
    - python compliance/pci_dss/req4_encryption_transit.py
    # ✅ All API endpoints use TLS 1.2+
    
    # Requirement 6: Develop secure systems
    - python compliance/pci_dss/req6_secure_development.py
    # ✅ No hardcoded credentials
    # ✅ Input validation implemented
    # ✅ Error handling doesn't expose sensitive data
    
    # Requirement 10: Track and monitor access
    - python compliance/pci_dss/req10_audit_logging.py
    # ✅ All access attempts logged
    # ✅ Log retention: 365 days

# GDPR Checks
gdpr-check:
  stage: compliance
  script:
    # Article 5: Data minimization
    - python compliance/gdpr/data_minimization.py
    # ✅ Only necessary PII collected
    
    # Article 17: Right to erasure
    - python compliance/gdpr/right_to_erasure.py
    # ✅ API endpoint exists for data deletion
    # ✅ Deletion propagates to all data stores
    
    # Article 25: Data protection by design
    - python compliance/gdpr/data_protection_by_design.py
    # ✅ PII fields identified and tagged
    # ✅ Data masking in non-production environments

# RBI Checks
rbi-check:
  stage: compliance
  script:
    # Transaction Limits
    - python compliance/rbi/transaction_limits.py
    # ✅ NEFT: Max ₹5,00,000 per transaction
    # ✅ RTGS: Min ₹2,00,000, Max ₹10,00,000
    # ✅ UPI: Max ₹1,00,000 per transaction
    
    # Data Localization
    - python compliance/rbi/data_localization.py
    # ✅ All customer data stored in India
    # ✅ No cross-border data transfer without approval
    
    # Audit Trail
    - python compliance/rbi/audit_trail.py
    # ✅ All transactions have immutable audit log
    # ✅ Audit logs retained for 8 years
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: PCI-DSS Compliance Pipeline

**Context:** Every deployment must pass PCI-DSS checks.

```python
# compliance/pci_dss/req3_card_data_encryption.py
import re
import ast

def check_card_data_encryption():
    """Verify cardholder data is encrypted at rest"""
    
    issues = []
    
    # Check for unencrypted card numbers in code
    with open('src/main/java/com/bank/card/', 'r') as f:
        for line_num, line in enumerate(f, 1):
            # Check for plain text card storage
            if re.search(r'cardNumber\s*=\s*[^e]', line):
                issues.append(f"Line {line_num}: Card number stored in plain text")
            
            # Check for weak encryption
            if 'DES' in line or 'MD5' in line:
                issues.append(f"Line {line_num}: Weak encryption algorithm detected")
    
    # Check database schema
    with open('src/main/resources/db/migration/', 'r') as f:
        content = f.read()
        if 'card_number VARCHAR' in content:
            issues.append("Card number column not encrypted in database")
    
    # Check API responses
    if '"cardNumber":' in open('src/main/java/com/bank/api/').read():
        if 'mask' not in open('src/main/java/com/bank/api/').read():
            issues.append("Card number exposed in API response without masking")
    
    if issues:
        print("❌ PCI-DSS Requirement 3 FAILED:")
        for issue in issues:
            print(f"  - {issue}")
        exit(1)
    else:
        print("✅ PCI-DSS Requirement 3 PASSED: Card data encrypted")

if __name__ == "__main__":
    check_card_data_encryption()
```

```bash
# Run compliance check
$ python compliance/pci_dss/req3_card_data_encryption.py
❌ PCI-DSS Requirement 3 FAILED:
  - Line 45: Card number stored in plain text
  - Line 78: Weak encryption algorithm detected (DES)

# Developer fixes the issues
$ sed -i 's/cardNumber = /encryptedCardNumber = encrypt(cardNumber, AES256) /' src/...
$ sed -i 's/DES/AES256/' src/...

# Re-run compliance check
$ python compliance/pci_dss/req3_card_data_encryption.py
✅ PCI-DSS Requirement 3 PASSED: Card data encrypted
```

### E2E Example 2: GDPR Data Subject Request

**Context:** Customer requests data deletion (Right to Erasure).

```python
# compliance/gdpr/data_deletion.py
import psycopg2
import boto3

def delete_customer_data(customer_id):
    """Delete all customer data across all systems"""
    
    print(f"Starting data deletion for customer: {customer_id}")
    
    # 1. Database deletion
    conn = psycopg2.connect("dbname=banking host=prod-db")
    cur = conn.cursor()
    
    # Anonymize transactions (keep for audit, remove PII)
    cur.execute("""
        UPDATE transactions 
        SET customer_name = 'DELETED',
            customer_email = 'DELETED',
            customer_phone = 'DELETED',
            card_number = 'DELETED'
        WHERE customer_id = %s
    """, (customer_id,))
    
    # Delete account data
    cur.execute("DELETE FROM accounts WHERE customer_id = %s", (customer_id,))
    
    # Delete KYC documents
    cur.execute("DELETE FROM kyc_documents WHERE customer_id = %s", (customer_id,))
    
    conn.commit()
    print(f"✅ Database: {cur.rowcount} records deleted")
    
    # 2. S3 deletion (KYC documents, statements)
    s3 = boto3.client('s3')
    response = s3.list_objects_v2(Bucket='bank-documents', Prefix=f'customer/{customer_id}/')
    
    for obj in response.get('Contents', []):
        s3.delete_object(Bucket='bank-documents', Key=obj['Key'])
    
    print(f"✅ S3: {len(response.get('Contents', []))} objects deleted")
    
    # 3. Redis deletion (cached data)
    redis_client = Redis(host='redis.bank.com')
    keys = redis_client.keys(f'customer:{customer_id}:*')
    redis_client.delete(*keys)
    
    print(f"✅ Redis: {len(keys)} cached entries deleted")
    
    # 4. Log deletion (anonymize)
    # Note: Keep audit logs but remove PII
    
    # 5. Create deletion record
    cur.execute("""
        INSERT INTO data_deletion_log (customer_id, deleted_by, deleted_at, status)
        VALUES (%s, 'SYSTEM', NOW(), 'COMPLETED')
    """, (customer_id,))
    conn.commit()
    
    print(f"✅ Data deletion complete for customer: {customer_id}")
    print(f"✅ Audit log created for compliance")

if __name__ == "__main__":
    import sys
    delete_customer_data(sys.argv[1])
```

### E2E Example 3: RBI Data Localization Check

**Context:** Verify all customer data stays within India.

```python
# compliance/rbi/data_localization.py
import json
import subprocess

def check_data_localization():
    """Verify no data leaves India without approval"""
    
    issues = []
    
    # Check Terraform configurations
    result = subprocess.run(['grep', '-r', 'region=', 'terraform/'], 
                          capture_output=True, text=True)
    
    non_india_regions = ['us-east', 'us-west', 'eu-west', 'ap-southeast', 'ap-northeast']
    
    for line in result.stdout.split('\n'):
        for region in non_india_regions:
            if region in line and 'india' not in line.lower():
                issues.append(f"Non-India region detected: {line}")
    
    # Check Kubernetes network policies
    result = subprocess.run(['kubectl', 'get', 'networkpolicy', '--all-namespaces', '-o', 'json'],
                          capture_output=True, text=True)
    
    policies = json.loads(result.stdout)
    for policy in policies.get('items', []):
        egress = policy.get('spec', {}).get('egress', [])
        for rule in egress:
            ipBlocks = rule.get('ipBlock', {}).get('cidr', '')
            if ipBlocks and not ipBlocks.startswith('10.') and not ipBlocks.startswith('172.'):
                issues.append(f"External egress detected: {ipBlocks}")
    
    # Check for cross-border data transfer
    result = subprocess.run(['grep', '-r', 'aws.s3', 'src/'], 
                          capture_output=True, text=True)
    
    for line in result.stdout.split('\n'):
        if 'ap-south' not in line and 'bucket' in line:
            issues.append(f"Potential cross-border S3 access: {line}")
    
    if issues:
        print("❌ RBI Data Localization FAILED:")
        for issue in issues:
            print(f"  - {issue}")
        exit(1)
    else:
        print("✅ RBI Data Localization PASSED: All data in India")

if __name__ == "__main__":
    check_data_localization()
```

---

## 📋 Interview Questions

### Q1: What is the difference between compliance and security?
**Answer:** **Security** protects systems from threats (firewalls, encryption, access control). **Compliance** proves you're meeting regulatory requirements (PCI-DSS, GDPR, RBI). Security is a subset of compliance — compliance includes security plus administrative controls, documentation, and audit trails. In banking, you need both: security to protect data, compliance to prove it to regulators.

### Q2: How do you implement PCI-DSS in a CI/CD pipeline?
**Answer:** (1) **Automated scanning** — check for card data in code (no plain text storage). (2) **Encryption verification** — ensure all card data encrypted at rest and in transit. (3) **Access control** — verify RBAC for card data access. (4) **Audit logging** — confirm all card data access logged. (5) **Network segmentation** — verify cardholder data environment isolated. (6) **Vulnerability scanning** — scan for known vulnerabilities. All checks run automatically on every commit.

### Q3: How do you handle GDPR "Right to Erasure" in a banking system?
**Answer:** (1) **Data mapping** — know where all customer PII is stored. (2) **Deletion API** — endpoint to delete customer data across all systems. (3) **Cascade deletion** — delete from databases, file storage, caches, logs. (4) **Anonymization** — keep transaction records but remove PII. (5) **Audit trail** — log deletion request for compliance. (6) **Verification** — confirm data is deleted from all systems. Note: Some data (audit logs) must be retained for legal reasons — anonymize instead of delete.

### Q4: What is "shift-left" compliance and why does it matter?
**Answer:** "Shift-left" means moving compliance checks earlier in the development lifecycle. Instead of catching compliance issues during annual audit, catch them during code development. Benefits: (1) **Faster fixes** — compliance issue found during coding costs $100 to fix; during audit costs $100,000. (2) **Continuous compliance** — every commit is checked, not just annual. (3) **Developer education** — developers learn compliance requirements. (4) **Audit readiness** — always audit-ready, not scrambling before audit.

### Q5: How do you generate compliance reports for regulators?
**Answer:** (1) **Automated collection** — CI/CD logs, Git history, pipeline results. (2) **Evidence generation** — screenshots, logs, test results. (3) **Report templates** — pre-defined formats for PCI-DSS, GDPR, RBI. (4) **Continuous monitoring** — real-time compliance dashboard. (5) **Audit trail** — every change linked to JIRA, PR, pipeline run. Tools: AWS Config, Azure Policy, custom scripts. Frequency: real-time dashboards for ops, quarterly reports for management, annual reports for regulators.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Compliance as Code | Automate regulatory checks in CI/CD |
| PCI-DSS | Card data protection requirements |
| GDPR | Data privacy and right to erasure |
| RBI | Indian banking regulations |
| Shift-Left | Catch compliance issues early |
| Banking Relevance | Continuous compliance, audit readiness |

**Next:** [23-API-Gateway-CICD.md](./23-API-Gateway-CICD.md) — Learn API management for banking.
