# ☁️ Cloud Pentesting Red Team Lab

> **Ethical Hacking — Apni khud ki vulnerable AWS environment banao aur attack karo.**  
> Attacker ki tarah sochna seekho taaki better defend kar sako.
  
**Type:** Offensive + Defensive Cloud Security  
**Status:** ✅ Complete

---

## 🗂️ Project Structure

```
~/Developer/red-team-lab/
├── cloudgoat/              # Vulnerable AWS lab (CloudGoat 2.5.0)
├── pacu/                   # AWS exploitation framework
├── prowler-results/        # Prowler security scan outputs
├── scoutsuite-report/      # ScoutSuite visual audit report
└── reports/
    └── cloudgoat-pentest-report.md   # Professional pentest report
```

---

## 🛠️ Tech Stack

| Tool | Version | Purpose |
|------|---------|---------|
| [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat) | 2.5.0 | Deliberately vulnerable AWS environment |
| [Pacu](https://github.com/RhinoSecurityLabs/pacu) | Latest | AWS exploitation framework |
| [Prowler](https://github.com/prowler-cloud/prowler) | 5.22.0 | AWS security assessment (300+ checks) |
| [ScoutSuite](https://github.com/nccgroup/ScoutSuite) | 5.14.0 | Multi-cloud security audit |
| AWS CLI | 2.34.21 | AWS API interaction |
| Terraform | 1.14.8 | Infrastructure provisioning |
| Python | 3.9.6 | Runtime |
| Poetry | Latest | Dependency management |

---

## ⚔️ Attack Scenarios Completed

### 1. IAM Privilege Escalation by Policy Rollback
**Severity:** 🔴 Critical  
**Concept:** Old IAM policy versions enumerate karke admin version activate karna

### 2. EC2 SSRF + Lambda Secret Theft  
**Severity:** 🔴 Critical  
**Concept:** Lambda env vars mein hardcoded credentials → EC2 role abuse

### 3. IAM Privilege Escalation by Role Attachment
**Severity:** 🔴 Critical  
**Concept:** Instance profile mein powerful role swap → EC2 launch → Metadata theft

### 4. Lambda Privilege Escalation
**Severity:** 🔴 Critical  
**Concept:** LambdaManager role assume → Malicious Lambda deploy → Admin role execution

---

## 🛡️ Defense Tools Used

### Prowler Scan Results
- **IAM:** 2 Critical, 13 High findings
- **EC2:** 24 High, 25 Medium findings  
- **S3:** 2 High, 15 Medium findings
- **Total:** 99 findings across all services

### Pacu Automation
- 24 confirmed privilege escalation paths auto-detected
- Full EC2, IAM, Lambda enumeration

---

## 🔑 Key Concepts Covered

- AWS IAM privilege escalation paths
- EC2 Instance Metadata Service (IMDS) exploitation
- Lambda function secret exposure
- Lateral movement in AWS
- Automated security auditing
- Professional pentest reporting

---

## ⚠️ Legal Disclaimer

> Yeh sab **sirf CloudGoat ki apni AWS lab** pe kiya gaya hai.  
> Real AWS accounts, company infrastructure, ya kisi bhi production system pe kabhi nahi karna.  
> Unauthorized access illegal hai aur serious consequences hote hain.

---

## 📄 Reports

- [Full Pentest Report](./reports/cloudgoat-pentest-report.md)
- [Attack Walkthrough](./ATTACK_WALKTHROUGH.md)
- [Concepts Guide](./CONCEPTS_GUIDE.md)
