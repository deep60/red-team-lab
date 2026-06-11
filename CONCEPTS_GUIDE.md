# 📚 Cloud Security Concepts Guide

> Yeh guide un concepts ko explain karta hai jo is project mein use hue.  
> Future mein bhool jaao toh yahan dekho!

---

## 🔑 AWS IAM — Identity & Access Management

### IAM Kya Hai?
AWS mein **kaun kya kar sakta hai** — yeh IAM decide karta hai.

```
IAM ke 4 main components:
┌─────────────────────────────────────────┐
│  Users    → Real log ya applications    │
│  Groups   → Users ka collection         │
│  Roles    → Temporary identity          │
│  Policies → Permission rules (JSON)     │
└─────────────────────────────────────────┘
```

### Policy Kya Hoti Hai?
```json
{
  "Statement": [{
    "Effect": "Allow",          // Allow ya Deny
    "Action": "s3:GetObject",   // Kya kar sakta hai
    "Resource": "arn:aws:s3:::my-bucket/*"  // Kis pe
  }]
}
```

### Dangerous Permissions — Privilege Escalation Ke Liye
| Permission | Kyun Dangerous |
|-----------|----------------|
| `iam:SetDefaultPolicyVersion` | Purani admin version activate kar sakte hain |
| `iam:CreatePolicyVersion` | Nayi admin version bana sakte hain |
| `iam:PassRole` | Powerful role kisi service ko de sakte hain |
| `iam:AddUserToGroup` | Admin group mein khud ko add kar sakte hain |
| `iam:AttachUserPolicy` | Khud ko admin policy laga sakte hain |
| `sts:AssumeRole` | Doosri identity ban sakte hain |
| `lambda:CreateFunction` + `iam:PassRole` | Admin Lambda bana sakte hain |
| `ec2:RunInstances` + `iam:PassRole` | Admin EC2 launch kar sakte hain |

### Policy Versions
```
AWS mein ek policy ke max 5 versions ho sakte hain:
v1 (default) → limited permissions
v2           → deny all (lockout)
v3           → Action:* Resource:* (FULL ADMIN!) ← vulnerability
v4           → expired time-locked
v5           → S3 read only

Attacker → SetDefaultPolicyVersion → v3 activate → Admin!
```

---

## 🖥️ EC2 Instance Metadata Service (IMDS)

### IMDS Kya Hai?
```
Magic IP: 169.254.169.254
Sirf EC2 ke andar se accessible hai
EC2 ko apne baare mein information milti hai yahan se:
  - Instance ID
  - Region
  - IAM role credentials (SABSE IMPORTANT!)
  - User data
  - Network info
```

### Attack — Credentials Churaana
```bash
# EC2 ke andar se:

# Step 1: Kaunsa role attached hai?
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Output: my-ec2-role

# Step 2: Credentials nikalo
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/my-ec2-role
# Output:
# {
#   "AccessKeyId": "ASIA...",
#   "SecretAccessKey": "...",
#   "Token": "...",
#   "Expiration": "..."
# }
```

### IMDSv1 vs IMDSv2
```
IMDSv1 (OLD - DANGEROUS):
  curl http://169.254.169.254/...  ← Direct access
  SSRF attack se bhi accessible!

IMDSv2 (NEW - SAFER):
  Step 1: Token lo pehle
  TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  
  Step 2: Token use karo
  curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/...
```

### SSRF + IMDS Attack
```
SSRF (Server Side Request Forgery):
Web app → Attacker tricks server to make requests
Server → Requests IMDS on attacker's behalf
IMDS → Returns IAM credentials
Attacker → Steals credentials!

Fix: IMDSv2 enforce karo → SSRF se credentials nahi milenge
```

---

## 🔄 AWS STS — Security Token Service

### STS Kya Hai?
```
Temporary credentials banata hai:
  - AccessKeyId (ASIA... se shuru hota hai, AKIA nahi)
  - SecretAccessKey
  - SessionToken (IMPORTANT — required for temp creds)
  - Expiration (kuch ghanton mein expire hota hai)
```

### AssumeRole
```bash
# Kisi aur ki identity temporarily assume karna
aws sts assume-role \
  --role-arn arn:aws:iam::123456789:role/admin-role \
  --role-session-name my-session

# Trust Policy decide karta hai kaun assume kar sakta hai:
{
  "Principal": { "Service": "lambda.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
# Iska matlab: sirf Lambda service assume kar sakti hai
# Humans directly nahi kar sakte!
```

### Long-term vs Short-term Credentials
```
Long-term (AKIA...):
  - IAM user ki permanent keys
  - Expire nahi hoti
  - Zyada dangerous

Short-term (ASIA...):
  - Role assume karne se milti hain
  - Kuch ghanton mein expire
  - Token required
  - Safer
```

---

## ⚡ AWS Lambda

### Lambda Kya Hai?
```
Serverless function = Code run karo bina server manage kiye
Event pe trigger hota hai
Execution role ke permissions ke saath chalta hai
```

### Lambda Execution Role
```
Lambda function → Runs with attached IAM role
Role ki permissions → Lambda ke andar available
curl http://169.254.169.254 → Lambda ke andar bhi kaam karta hai!

Attack:
  Malicious Lambda + Admin execution role
  → Lambda invoke karo
  → Admin credentials milte hain
```

### Lambda Environment Variables — Common Mistake
```python
# ❌ WRONG — Kabhi mat karo
import os
ACCESS_KEY = os.environ['AWS_ACCESS_KEY']  # Plain text!

# ✅ CORRECT — Secrets Manager use karo
import boto3
client = boto3.client('secretsmanager')
secret = client.get_secret_value(SecretId='prod/myapp/keys')
```

---

## 🌐 AWS S3

### S3 Common Security Issues
```
1. Public bucket — koi bhi internet se access kar sakta hai
2. Unencrypted data — encryption at rest nahi
3. No versioning — accidental delete se recover nahi ho sakta
4. No access logging — kaun access kar raha hai pata nahi
5. Overpermissive bucket policy — s3:* instead of specific actions
```

### S3 Enumeration Commands
```bash
# Sab buckets dekho
aws s3 ls --profile <p>

# Bucket contents dekho
aws s3 ls s3://bucket-name --profile <p>

# File download karo
aws s3 cp s3://bucket-name/file.txt . --profile <p>
```

---

## 🛡️ Defense Tools

### Prowler
```
AWS security assessment tool
300+ checks karta hai
CIS Benchmarks, NIST, PCI-DSS compliance check karta hai
Output: HTML, JSON, CSV

Key checks:
  - IAM users without MFA
  - Root account usage
  - Overpermissive policies
  - Public S3 buckets
  - Unencrypted resources
  - CloudTrail disabled
```

```bash
# Basic scan
prowler aws --profile <p> --service iam --region us-east-1

# Only critical/high failures
prowler aws --profile <p> --severity critical high --status FAIL
```

### ScoutSuite
```
Multi-cloud security auditing tool
Visual interactive HTML dashboard
Service-wise breakdown
Risk categorization

Better than Prowler for:
  - Visual reporting
  - Quick overview
  - Client presentations
```

### Pacu
```
AWS exploitation framework (like Metasploit but for AWS)
Categories:
  ENUM   → Enumerate resources
  ESCALATE → Privilege escalation
  PERSIST  → Backdoors banana
  EXFIL    → Data nikalna
  EVADE    → Detection avoid karna

Most useful commands:
  iam__enum_users_roles_policies_groups → Full IAM map
  iam__privesc_scan → Auto privilege escalation paths
  ec2__enum → All EC2 resources
  secrets__enum → Secrets Manager + Parameter Store
```

---

## 🎯 Attack Methodologies

### MITRE ATT&CK for Cloud
```
Reconnaissance → Kya available hai?
  - IAM enumerate karo
  - S3 buckets dhundo
  - Lambda functions list karo

Initial Access → Kaise andar ghuse?
  - Leaked credentials
  - SSRF
  - Misconfigured permissions

Privilege Escalation → Admin kaise bane?
  - Policy version rollback
  - Role attachment
  - Lambda abuse

Lateral Movement → Aur resources kaise access kare?
  - Role chaining
  - Cross-account access

Impact → Kya nuksaan ho sakta hai?
  - Data exfiltration
  - Resource deletion
  - Backdoor creation
```

### Common AWS Attack Chains
```
1. Leaked Creds → Recon → Privesc → Admin
   (Most common in real breaches)

2. SSRF → IMDS → Role Creds → Lateral Movement
   (Web app vulnerability → Cloud compromise)

3. Lambda Secrets → Pivoting → Admin
   (Developer mistake → Full account takeover)

4. Overpermissive Role → EC2 Launch → Metadata
   (Misconfig → Admin escalation)
```

---

## 📋 Security Best Practices

### IAM Best Practices
```
✅ Least Privilege — sirf zaroori permissions
✅ MFA everywhere — especially root account
✅ Regular key rotation — 90 days max
✅ Delete old policy versions — max 2 rakhо
✅ Never use root for daily tasks
✅ Use roles instead of users for services
✅ Enable IAM Access Analyzer
✅ Review unused permissions quarterly
```

### Lambda Best Practices
```
✅ Secrets Manager use karo, env vars nahi
✅ Minimal execution role permissions
✅ VPC mein deploy karo sensitive functions ke liye
✅ Function URL restrict karo
✅ Code signing enable karo
✅ Concurrency limits set karo
```

### EC2 Best Practices
```
✅ IMDSv2 enforce karo (http-tokens=required)
✅ Minimal instance role permissions
✅ Security groups — sirf required ports
✅ No public IPs jahan zaroorat nahi
✅ EBS encryption enable karo
✅ SSM Session Manager use karo SSH ki jagah
✅ Regular patching
```

### S3 Best Practices
```
✅ Block all public access (account-level)
✅ Encryption at rest (SSE-S3 or SSE-KMS)
✅ Versioning enable karo
✅ Access logging enable karo
✅ Bucket policies review karo regularly
✅ MFA delete enable karo sensitive buckets pe
```

---

## 🔍 Quick Recon Checklist

Jab bhi naya AWS account/user milta hai, yeh commands run karo:

```bash
# 1. Kaun hoon main?
aws sts get-caller-identity --profile <p>

# 2. Meri policies?
aws iam list-attached-user-policies --user-name <user> --profile <p>
aws iam list-user-policies --user-name <user> --profile <p>

# 3. Policy content?
aws iam get-policy-version --policy-arn <arn> --version-id v1 --profile <p>

# 4. Kaun kaun se roles hain?
aws iam list-roles --profile <p>

# 5. EC2 instances?
aws ec2 describe-instances --region us-east-1 --profile <p>

# 6. Lambda functions?
aws lambda list-functions --region us-east-1 --profile <p>

# 7. S3 buckets?
aws s3 ls --profile <p>

# 8. Secrets?
aws secretsmanager list-secrets --region us-east-1 --profile <p>
```

---

## 💡 Key Takeaways

> **Cloud security = IAM security**  
> 90% attacks IAM misconfiguration se hote hain, software vulnerability se nahi.

> **Least privilege is not optional**  
> Har extra permission ek potential attack vector hai.

> **Secrets belong in Secrets Manager**  
> Lambda env vars, code comments, GitHub repos — yeh secrets ki jagah nahi hain.

> **Assume breach mentality**  
> Defense design karo is assumption se ki attacker already andar hai.

> **Automate security checks**  
> Prowler/ScoutSuite quarterly run karo, CI/CD mein integrate karo.
