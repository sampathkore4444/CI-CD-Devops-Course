# 016. SSL/TLS Certificate Expiry Causing Service Outage

## Scenario

Production payment APIs suddenly start failing. Clients report SSL handshake errors: "SSL certificate has expired" and "certificate verify failed." The SSL certificate on the AWS Application Load Balancer expired at midnight. It is now 8 AM on a Monday. The certificate was automatically renewed by the certificate authority, but the renewed certificate was never uploaded to the ALB listener. The organization uses a 3rd-party CA (not Let's Encrypt) and certificates are managed through an internal ticketing process with a 2-week lead time for renewals. The DevOps team is not sure which team is responsible for certificate renewal. The payment API handles approximately 200,000 requests per hour at peak, and all traffic is now failing because clients reject the expired certificate. The compliance team has already sent an urgent message about PCI-DSS violations and potential fines. The certificate is used by multiple services beyond the payment API, including the client portal and internal dashboards.

## Interviewer Question

Production payment APIs suddenly start failing. Clients report SSL handshake errors. The SSL certificate on the load balancer expired at midnight. It's 8 AM on a Monday. How do you handle the emergency renewal and prevent this from happening again?

## What I Should Think About

- Every minute of downtime on payment APIs means lost revenue, possibly in the tens of thousands of dollars
- There may be panic and blame, but the focus must be on restoration: get a valid certificate onto the ALB as quickly as possible
- This is a classic incident response: triage (identify the expired cert), mitigation (renew and install), and prevention (automate the process)
- The immediate action is to renew the certificate with the CA and install it in the AWS Certificate Manager (ACM) or on the ALB listener
- Need to check if there is a spare/backup certificate, maybe one for an older domain, that can be temporarily deployed
- Check if the organization has a wildcard certificate that can be used as a temporary measure
- Consider the CA renewal process: some CAs have quick renewal APIs, and Let's Encrypt supports automated renewal through certbot with DNS-01 or HTTP-01 challenges
- Consider the option of temporarily using a different certificate (e.g., a previously issued but not yet expired one for the same/related domain)
- After restoration, the priority is to prevent recurrence: automation is key

## Ideal Answer

My immediate priority is to restore service and minimize revenue loss, then put in place preventative measures to ensure this never happens again.

**Immediate steps:**
1. Confirm the certificate is expired by checking expiry date and test SSL handshake
2. Check if there is any valid temporary certificate available (wildcard, backup, or pending renewal from another environment)
3. Renew the certificate through the CA. If the organization uses a 3rd-party CA, contact the CA vendor for an expedited renewal or check if there's an existing renewal API
4. If Let's Encrypt is available, use certbot with DNS-01 or HTTP-01 to get a new certificate immediately
5. Upload the new certificate to ACM and attach it to all affected ALB listeners
6. Verify SSL handshake works and certificate is served correctly

**Preventative steps:**
1. Implement automated certificate renewal using ACM (if AWS certs) with DNS validation, or certbot for Let's Encrypt
2. Set up monitoring/alerting for certificate expiry: alerts 30, 14, 7, and 1 day before expiry
3. Create a central certificate inventory with renewal tracking
4. Document the certificate ownership and renewal process

## Architecture

```
┌────────────┐ ┌────────────┐ ┌────────────┐
│  Clients   │ │  Payment   │ │  Client    │
│            │ │  API       │ │  Portal    │
│ HTTPS req  │ │  HTTPS req │ │  HTTPS req │
└─────┬──────┘ └─────┬──────┘ └─────┬──────┘
      │              │              │
      └──────────────┼──────────────┘
                     ▼
      ┌───────────────────────────┐
      │  AWS ALB Listener :443    │
      │  Certificate: EXPIRED     │
      │                           │
      │  SSL handshake ---- FAIL  │
      └─────────────┬─────────────┘
                    │
                    ▼
      ┌───────────────────────────┐
      │  Target Group             │
      │  Backend Servers (healthy)│
      └───────────────────────────┘
```

## Investigation

1. Confirm certificate expiry date and domains covered
2. Check ALB listener SSL configuration to verify which certificate is attached
3. Check ACM certificate status and validate if there are any valid renewal options
4. Check other affected services (client portal, dashboards) that may also use the expired certificate
5. Verify the exact error clients see using openssl s_client
6. Check if there are existing monitoring/alerting for certificate expiry
7. Review documentation for certificate ownership and renewal process
8. Check if CI/CD pipeline has any certificate deployment automation
9. Check if a temporary certificate from another environment can be reused

## Commands

```bash
# Check certificate expiry dates from the outside
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | openssl x509 -noout -dates
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | openssl x509 -noout -subject -issuer

# Test SSL handshake failure
openssl s_client -connect api.example.com:443 -brief 2>&1 | head -10
openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts 2>&1 | head -20

# Check for all affected endpoints
for host in api.example.com portal.example.com dashboard.example.com; do
  echo "=== $host ==="
  echo | openssl s_client -connect $host:443 -servername $host 2>/dev/null | openssl x509 -noout -dates 2>/dev/null
done

# Generate a new certificate with Let's Encrypt (if available)
sudo certbot certonly --standalone -d api.example.com --agree-tos --no-eff-email -m ops@example.com
# Or with DNS-01 validation for auto-renewal
sudo certbot certonly --manual --preferred-challenges dns -d api.example.com

# Upload certificate to ACM manually (if from a 3rd-party CA)
aws acm import-certificate \
  --certificate fileb:///path/to/cert.pem \
  --private-key fileb:///path/to/private.key \
  --certificate-chain fileb:///path/to/chain.pem \
  --region us-east-1

# List certificates in ACM and check expiry dates
aws acm list-certificates --region us-east-1

# Check current certificate on ALB listener
aws elbv2 describe-listener --listener-arn <listener-arn> --region us-east-1
aws elbv2 describe-listener-certificates --listener-arn <listener-arn> --region us-east-1

# Attach new certificate to ALB listener
aws elbv2 add-listener-certificates \
  --listener-arn <listener-arn> \
  --certificates CertificateArn=arn:aws:acm:us-east-1:<account>:certificate/<new-cert-id> \
  --region us-east-1

# Check certificate expiry remotely using curl (useful for bulk checks)
for host in api.example.com portal.example.com; do
  expiry=$(curl -v --connect-timeout 5 https://$host/ 2>&1 | grep -i "expire date" | awk '{print $NF}')
  echo "$host: $expiry"
done

# Verify certificate is correctly served after fix
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | openssl x509 -noout -dates -subject -issuer

# If using ACM-managed certificates, resubmit validation and confirm renewal
aws acm describe-certificate --certificate-arn <arn> --region us-east-1 | grep -i "Status\|Renewal"

# Test the full chain validation (including intermediate CAs)
openssl s_client -connect api.example.com:443 -servername api.example.com \
  -verify_return_error -CApath /etc/ssl/certs 2>&1 | head -30

# Check cert on the ALB after attaching (verify served cert matches renewed one)
aws elbv2 describe-listener-certificates --listener-arn <listener-arn> --region us-east-1
```

## Root Cause

- **No automated certificate renewal**: The organization relies on manual renewal through a ticketing process. The certificate expired because no one initiated the renewal
- **Lack of certificate monitoring**: No monitoring/alerting exists for certificate expiry, so the team was unaware until clients reported issues
- **Unclear ownership**: The DevOps team assumed the security team handled certificates, and the security team assumed DevOps handled them. No RACI existed
- **CA constraints**: The 3rd-party CA may not support automated renewal, requiring a manual process that missed the deadline
- **No failover**: The certificate was hosted by a single CA with no backup or alternate certificate for the same domain

## Immediate Mitigation

1. Do NOT attempt to renew the 3rd-party CA certificate if it will take days. First restore service
2. If Let's Encrypt is available for the domain, generate a new certificate immediately and attach to the ALB. This is the fastest path
3. If the ALB is in another region or the certificate is not in ACM, upload the new certificate to ACM and attach to the ALB listener
4. If no valid certificate exists, consider temporarily using a wildcard certificate for a different domain that is still valid, if the clients can accept it (some payment gateways may reject it)
5. Once a temporary certificate is active, communicate the resolution to affected teams
6. Immediately stand up a monitoring alert for certificate expiry to prevent recurrence

## Permanent Fix

1. Use ACM-managed certificates (with automatic renewal) over manually uploaded ones, when possible
2. If using 3rd-party CA certificates, integrate with certbot or a similar tool for automated renewal with DNS validation
3. Add certificate expiry monitoring with alerts at 30, 14, 7, and 1 day(s) before expiry
4. Create a central certificate inventory with renewal tracking
5. Automate deployment: certbot renew + deploy to load balancer through CI/CD pipeline
6. Document the certificate ownership and renewal process (RACI matrix)
7. Establish a certificate review gate in the deployment pipeline: any cert with <30 days to expiry fails the build

## Monitoring

- Add certificate expiry alerts (30/14/7/1 day threshold) using AWS CloudWatch or external monitoring tools
- Monitor SSL handshake success rates across all ALB listeners
- Set up synthetic SSL check: periodically connect via openssl s_client and verify certificate validity
- Track ACM certificate renewal status in CloudWatch
- Alert on certificate error rates exceeding 0.5% for 2 minutes
- Add SSL certificate expiry alerts to external monitoring (StatusCake, UptimeRobot, or similar)
- Run a daily certificate inventory report (ACM list-certificates + expiry dates) to catch any missed certs
- Use Amazon EventBridge with ACM renewal events to trigger a Lambda that alerts on renewal failures

## Security

- Ensure private keys are stored securely in the secret manager or KMS, never in plain text or in commit history
- Enforce strong signature algorithms (SHA-256 or better) with 2048-bit+ keys
- Consider HSTS headers to prevent downgrade attacks, and include certificate-based client authentication where appropriate
- Comply with PCI-DSS for payment APIs, which requires valid certificates
- Use ACM integration with AWS Certificate Manager to simplify certificate management
- Rotate certificates on a regular schedule (e.g., every 90 days) even if not expired, to reduce attack surface

## Production Considerations

- **High Availability**: All ALB listeners must have valid certificates. Use ACM to manage certificates across multiple regions
- **Scalability**: SSL termination should be offloaded to the load balancer, not the backend. This improves backend scalability and centralizes certificate management
- **Reliability**: Automated renewal ensures certificates never expire. Monitor SSL handshake rate to detect issues before they affect production
- **Cost**: Certificate management is inexpensive compared to the revenue loss from a payment API outage. Automate to avoid manual labor
- **Compliance**: Payment APIs are subject to PCI-DSS. Certificate expiry is a compliance violation and can trigger fines
- **Operational**: Create a runbook for certificate renewal. Test the renewal process quarterly. Keep a list of all certificate owners and contact points

## Senior-Level Answer

Restore service first: obtain a valid certificate immediately (using Let's Encrypt if available, or a temp certificate), upload to ACM and attach to all affected ALB listeners. Then prevent recurrence: automate certificate renewal, add expiry monitoring, and document ownership. The keys are speed of restoration and eliminating manual processes.

## Architect-Level Answer

This incident reveals a systemic failure in certificate lifecycle management. I would implement a fully automated certificate management platform: ACM for AWS-native certs with automatic renewal, certbot/cert-manager for 3rd-party domains with LEDNS01/DNS01 challenges, centralized certificate inventory with expiry tracking, and automated deployment to all load balancers. I would enforce that no certificate may ever be managed manually. I would also add certificate expiry as a blocker in CI/CD gates so a building with a near-expiring cert fails to deploy, and implement synthetic SSL checks from multiple geographic locations to catch issues early.

## Follow-Up Questions

1. If you have 200 certificates and no automation, how would you prioritize which certificates to renew first after an outage?
2. How do you handle certificate renewal for domains where the DNS is managed by a different team than the certificate owners?
3. What are the trade-offs between using ACM-managed certificates vs. 3rd-party CA certificates for a production ALB?
4. If a client cannot accept the renewed certificate because of a new intermediate or root CA, how would you troubleshoot and fix that?
5. How would you design a certificate rotation process for a fleet of 500 ALBs, each with a different certificate, without causing a single point of failure?