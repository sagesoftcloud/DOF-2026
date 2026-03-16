# DOF Control Tower & IAM Identity Center Implementation Guide
**Session 2 - Implementation Phase**
**Date:** March 2026
**Client:** Department of Finance (DOF)
**Architecture:** Option 2 - Production-Ready (7 Accounts)

---

## Session Agenda

1. Control Tower Architecture Implementation
2. OU Creation (Production & Development)
3. Account Provisioning (DOF-Staging, DOF-Development)
4. Account Rename (Angelica Sarmiento → DOF-Production)
5. IAM Identity Center Configuration
6. Permission Sets & Group Setup
7. Validation & Testing

---

## AWS Account & Resource Naming Convention

### Why Naming Matters (AWS Well-Architected & Multi-Account Best Practices)

Per the AWS whitepaper *"Organizing Your AWS Environment Using Multiple Accounts"* and the Well-Architected Framework (COST02-BP03), account naming should:
- Clearly identify the **organization**, **workload/system**, and **environment**
- Enable quick identification in billing, CloudTrail logs, and IAM Identity Center
- Support cost allocation and governance at scale
- Be consistent across all accounts and resources

> **References:**
> - [COST02-BP03 Implement an account structure](https://docs.aws.amazon.com/wellarchitected/2023-10-03/framework/cost_govern_usage_account_structure.html)
> - [Organizing Your AWS Environment - Application OUs](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/application-ous.html)
> - [AWS Control Tower multi-account strategy](https://docs.aws.amazon.com/controltower/latest/userguide/aws-multi-account-landing-zone.html)

### Recommended Pattern

```
{Org}-{System/WebApp}-{Environment}
```

| Component | Description | Examples |
|-----------|-------------|---------|
| `{Org}` | Organization abbreviation | `DOF` |
| `{System/WebApp}` | Application or workload name | `DOFWEB`, `B2BI`, `OreTool`, `TES`, `GCF` |
| `{Environment}` | Lifecycle stage | `Prod`, `Staging`, `Dev` |

### DOF Account Naming Convention

| Account | Naming Pattern | Account Name |
|---------|---------------|-------------|
| Management | `{Org}-Management` | `DOF-Management` (currently RadentaDOF) |
| Log Archive | *(AWS default - keep as-is)* | `Log Archive` |
| Audit | *(AWS default - keep as-is)* | `Audit` |
| Operations | `{Org}-Operations` | `DOF-Operations` (currently Operations) |
| Production | `{Org}-{System}-Prod` | `DOF-DOFWEB-Prod` or `DOF-Production` |
| Staging | `{Org}-{System}-Staging` | `DOF-DOFWEB-Staging` or `DOF-Staging` |
| Development | `{Org}-{System}-Dev` | `DOF-DOFWEB-Dev` or `DOF-Development` |

### Two Approaches for DOF

**Approach A: One Account Per Environment (Current Plan - 7 Accounts)**

All workloads share the same account per environment. Simpler, good for DOF's current size.

```
DOF-Production      ← All prod workloads (DOFWEB, B2BI, OreTool, TES, GCF)
DOF-Staging         ← All staging workloads
DOF-Development     ← All dev workloads
```

**Approach B: One Account Per Workload Per Environment (Future Scale)**

Each workload gets its own account per environment. Better isolation, but more accounts.

```
DOF-DOFWEB-Prod     ← DOFWEB production only
DOF-B2BI-Prod       ← B2BI production only
DOF-DOFWEB-Staging  ← DOFWEB staging only
DOF-DOFWEB-Dev      ← DOFWEB development only
```

### ✅ Recommendation for DOF

**Start with Approach A** (environment-based accounts) — this aligns with Option 2's 7-account structure and DOF's current workload size. The AWS whitepaper recommends separating production from non-production environments as the primary separation, with per-workload accounts as an advanced pattern when you scale.

If DOF grows to many more applications in the future, you can migrate to Approach B by creating additional accounts under the same OUs.

### Resource Naming Convention (Inside Accounts)

For resources within each account, use this pattern:

```
{Environment}-{Org}-{ResourceType}-{System/WebApp}-{Purpose}-{Sequence}
```

**Examples (based on DOF's existing EC2 instances):**

| Current Name | Follows Convention? | Recommended Name |
|-------------|-------------------|-----------------|
| `PROD-DOF-EC2-DOFWEB-APP-01` | ✅ Yes | Keep as-is |
| `PROD-DOF-EC2-DOFWEB-DB-01` | ✅ Yes | Keep as-is |
| `PROD-DOF-EC2-ASW-B2BI-APP-01` | ✅ Yes | Keep as-is |
| `PROD-DOF-EC2-ASW-B2BI-DB-01` | ✅ Yes | Keep as-is |
| `PROD-DOF-EC2-ORETOOL-01` | ✅ Yes | Keep as-is |
| `PROD-DOF-EC2-TESDB-01` | ⚠️ Missing type | `PROD-DOF-EC2-TES-DB-01` |
| `PROD-DOF-EC2-GCF-Server` | ⚠️ Inconsistent | `PROD-DOF-EC2-GCF-APP-01` |
| `UAT-DOF-EC2-ASW-B2BI-APP-01` | ✅ Yes | Keep as-is |
| `DEV-DOF-EC2-TES-APP-01` | ✅ Yes | Keep as-is |

DOF's existing EC2 naming is already well-structured. Only minor inconsistencies to fix.

### Email Convention for New Accounts

```
DOF-Staging:      aws+dof-staging@<domain>
DOF-Development:  aws+dof-development@<domain>
```

Using the `+` alias trick routes all emails to the same `aws@<domain>` inbox.

### Tagging Strategy (Complements Naming)

Apply these tags to all resources for cost allocation and governance:

| Tag Key | Example Value | Purpose |
|---------|-------------|---------|
| `Organization` | `DOF` | Cost allocation |
| `Environment` | `Production` / `Staging` / `Development` | Environment identification |
| `System` | `DOFWEB` / `B2BI` / `OreTool` | Workload identification |
| `Owner` | `DOF-Infrastructure` | Accountability |
| `CostCenter` | `DOF-IT` | Billing |
| `ManagedBy` | `Sagesoft` / `DOF` | Support routing |

---

## Pre-Implementation Checklist

- [ ] Management account access confirmed (239423589015)
- [ ] Control Tower console accessible
- [ ] IAM Identity Center enabled
- [ ] DOF user list received from Sir Dennis (names, emails, roles)
- [ ] AMI preparation status confirmed
- [ ] Drift resolved (Re-Register Workloads OU)

---

## Part 1: Control Tower Implementation

### Current State

| # | Account | ID | OU |
|---|---------|----|----|
| 1 | RadentaDOF (Management) | 239423589015 | Root |
| 2 | Log Archive | 100116628655 | Security OU |
| 3 | Audit | 113307936889 | Security OU |
| 4 | Operations | 482496759415 | Infrastructure OU |
| 5 | Angelica Sarmiento | 536225268327 | Workloads OU > DOF OU |

### Target State (Option 2)

| # | Account | ID | OU | Status |
|---|---------|----|----|--------|
| 1 | RadentaDOF (Management) | 239423589015 | Root | ✅ Existing |
| 2 | Log Archive | 100116628655 | Security OU | ✅ Existing |
| 3 | Audit | 113307936889 | Security OU | ✅ Existing |
| 4 | Operations | 482496759415 | Infrastructure OU | ✅ Existing |
| 5 | **DOF-Production** | 536225268327 | Production OU | 🔄 Rename + Move |
| 6 | **DOF-Staging** | NEW | Production OU | 🆕 Create |
| 7 | **DOF-Development** | NEW | Development OU | 🆕 Create |

---

### Step 1: Resolve Control Tower Drift

**Why:** Workloads OU currently only has `FullAWSAccess` SCP — missing guardrails.

**Console Steps:**
1. Go to **AWS Control Tower** → Dashboard
2. Look for drift warning banner
3. Click **"Re-Register OU"** on Workloads OU
4. Wait 30-60 minutes for completion
5. Verify drift warning disappears

**CLI Verification:**
```bash
aws controltower get-landing-zone \
  --landing-zone-identifier "arn:aws:controltower:ap-southeast-2:239423589015:landingzone/4TI064V1CN56ZB7I" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws organizations list-policies-for-target \
  --target-id ou-7u73-xe7gbtty \
  --filter SERVICE_CONTROL_POLICY \
  --profile dof-management-account
```

**Expected Result:** Workloads OU should now have `FullAWSAccess` + Control Tower guardrail SCPs.

---

### Step 2: Rename Angelica Sarmiento Account → DOF-Production

**From Within the Angelica Account:**
1. Sign in to account 536225268327
2. Go to **Account Settings** (top-right → Account)
3. Click **Edit** next to Account Name
4. Change to: `DOF-Production`
5. Save changes

**CLI:**
```bash
aws account put-account-name \
  --account-name "DOF-Production" \
  --region ap-southeast-2 \
  --profile dof-angelica-production-account
```

---

### Step 3: Create Production OU

**Console Steps:**
1. Go to **AWS Control Tower** → Organization
2. Click **"Create organizational unit"**
3. OU Name: `Production`
4. Parent OU: `Root`
5. Click **Create**

**CLI:**
```bash
aws organizations create-organizational-unit \
  --parent-id r-7u73 \
  --name "Production" \
  --profile dof-management-account
```

**Register OU with Control Tower:**
1. Go to **AWS Control Tower** → Organization
2. Find the new `Production` OU
3. Click **"Register OU"**
4. Wait for registration to complete (10-15 minutes)

---

### Step 4: Create Development OU

**Console Steps:**
1. Go to **AWS Control Tower** → Organization
2. Click **"Create organizational unit"**
3. OU Name: `Development`
4. Parent OU: `Root`
5. Click **Create**

**CLI:**
```bash
aws organizations create-organizational-unit \
  --parent-id r-7u73 \
  --name "Development" \
  --profile dof-management-account
```

**Register OU with Control Tower:**
1. Go to **AWS Control Tower** → Organization
2. Find the new `Development` OU
3. Click **"Register OU"**
4. Wait for registration to complete

---

### Step 5: Move DOF-Production Account to Production OU

**Console Steps:**
1. Go to **AWS Control Tower** → Organization
2. Find **DOF-Production** (536225268327)
3. Select the account → Actions → **Move**
4. Select destination: `Production` OU
5. Confirm move

**CLI:**
```bash
# Get the new Production OU ID first
aws organizations list-organizational-units-for-parent \
  --parent-id r-7u73 \
  --profile dof-management-account

# Move account to Production OU
aws organizations move-account \
  --account-id 536225268327 \
  --source-parent-id ou-7u73-idle80jb \
  --destination-parent-id <production-ou-id> \
  --profile dof-management-account
```

---

### Step 6: Create DOF-Staging Account (via Control Tower Account Factory)

**Console Steps:**
1. Go to **AWS Control Tower** → Account Factory
2. Click **"Create account"**
3. Fill in:
   - Account name: `DOF-Staging`
   - Account email: `aws+dof-staging@<domain>` *(confirm with DOF)*
   - IAM Identity Center user email: *(admin email)*
   - Organizational unit: `Production`
4. Click **Create account**
5. Wait 20-30 minutes for provisioning

**CLI (via Service Catalog):**
```bash
aws servicecatalog provision-product \
  --product-name "AWS Control Tower Account Factory" \
  --provisioning-artifact-name "AWS Control Tower Account Factory" \
  --provisioned-product-name "DOF-Staging" \
  --provisioning-parameters \
    '[{"Key":"AccountName","Value":"DOF-Staging"},
      {"Key":"AccountEmail","Value":"aws+dof-staging@<domain>"},
      {"Key":"SSOUserEmail","Value":"<admin-email>"},
      {"Key":"SSOUserFirstName","Value":"DOF"},
      {"Key":"SSOUserLastName","Value":"Staging"},
      {"Key":"ManagedOrganizationalUnit","Value":"Production"}]' \
  --region ap-southeast-2 \
  --profile dof-management-account
```

**Verification:**
```bash
aws servicecatalog describe-provisioned-product \
  --name "DOF-Staging" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

---

### Step 7: Create DOF-Development Account (via Control Tower Account Factory)

**Console Steps:**
1. Go to **AWS Control Tower** → Account Factory
2. Click **"Create account"**
3. Fill in:
   - Account name: `DOF-Development`
   - Account email: `aws+dof-development@<domain>` *(confirm with DOF)*
   - IAM Identity Center user email: *(admin email)*
   - Organizational unit: `Development`
4. Click **Create account**
5. Wait 20-30 minutes for provisioning

**CLI:**
```bash
aws servicecatalog provision-product \
  --product-name "AWS Control Tower Account Factory" \
  --provisioning-artifact-name "AWS Control Tower Account Factory" \
  --provisioned-product-name "DOF-Development" \
  --provisioning-parameters \
    '[{"Key":"AccountName","Value":"DOF-Development"},
      {"Key":"AccountEmail","Value":"aws+dof-development@<domain>"},
      {"Key":"SSOUserEmail","Value":"<admin-email>"},
      {"Key":"SSOUserFirstName","Value":"DOF"},
      {"Key":"SSOUserLastName","Value":"Development"},
      {"Key":"ManagedOrganizationalUnit","Value":"Development"}]' \
  --region ap-southeast-2 \
  --profile dof-management-account
```

---

### Step 8: Clean Up Old OUs (Optional)

```bash
# Check if old DOF OU is empty after moving account
aws organizations list-accounts-for-parent \
  --parent-id ou-7u73-idle80jb \
  --profile dof-management-account

# If empty, delete it
# aws organizations delete-organizational-unit \
#   --organizational-unit-id ou-7u73-idle80jb \
#   --profile dof-management-account
```

---

### Step 9: Verify Final Organization Structure

```bash
# List all OUs under Root
aws organizations list-organizational-units-for-parent \
  --parent-id r-7u73 \
  --profile dof-management-account

# List accounts in Production OU
aws organizations list-accounts-for-parent \
  --parent-id <production-ou-id> \
  --profile dof-management-account

# List accounts in Development OU
aws organizations list-accounts-for-parent \
  --parent-id <development-ou-id> \
  --profile dof-management-account

# Verify SCPs on Production OU
aws organizations list-policies-for-target \
  --target-id <production-ou-id> \
  --filter SERVICE_CONTROL_POLICY \
  --profile dof-management-account

# Verify SCPs on Development OU
aws organizations list-policies-for-target \
  --target-id <development-ou-id> \
  --filter SERVICE_CONTROL_POLICY \
  --profile dof-management-account
```

**Expected Final Structure:**
```
Root (r-7u73)
├── Management: RadentaDOF (239423589015)
├── Security OU (ou-7u73-1ogo3gub)
│   ├── Log Archive (100116628655)
│   └── Audit (113307936889)
├── Infrastructure OU (ou-7u73-3ignnemi)
│   └── Operations (482496759415)
├── Production OU (NEW)
│   ├── DOF-Production (536225268327) ← renamed + moved
│   └── DOF-Staging (NEW)
└── Development OU (NEW)
    └── DOF-Development (NEW)
```

---

## Part 2: IAM Identity Center Configuration

### Step 1: Verify IAM Identity Center is Enabled

```bash
aws sso-admin list-instances \
  --region ap-southeast-2 \
  --profile dof-management-account
```

---

### Step 2: Create Permission Sets

#### 2a: AdministratorAccess (Full Admin)

1. Go to **IAM Identity Center** → Permission Sets
2. Click **Create permission set**
3. Select **Predefined** → `AdministratorAccess`
4. Session duration: 4 hours
5. Click **Create**

```bash
INSTANCE_ARN=$(aws sso-admin list-instances \
  --region ap-southeast-2 \
  --profile dof-management-account \
  --query 'Instances[0].InstanceArn' --output text)

aws sso-admin create-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --name "AdministratorAccess" \
  --description "Full administrator access for DOF admins" \
  --session-duration "PT4H" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

#### 2b: PowerUserAccess (Developers)

```bash
aws sso-admin create-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --name "PowerUserAccess" \
  --description "Developer access - all services except IAM/Organizations" \
  --session-duration "PT8H" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

#### 2c: ReadOnlyAccess (Auditors)

```bash
aws sso-admin create-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --name "ReadOnlyAccess" \
  --description "Read-only access for auditors and reviewers" \
  --session-duration "PT4H" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

#### 2d: OperationsAccess (Custom - Operations Team)

```bash
aws sso-admin create-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --name "OperationsAccess" \
  --description "Operations team - EC2, RDS, CloudWatch, S3 management" \
  --session-duration "PT8H" \
  --region ap-southeast-2 \
  --profile dof-management-account

# Attach managed policies (use permission set ARN from response above)
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --permission-set-arn "<operations-permission-set-arn>" \
  --managed-policy-arn "arn:aws:iam::aws:policy/AmazonEC2FullAccess" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --permission-set-arn "<operations-permission-set-arn>" \
  --managed-policy-arn "arn:aws:iam::aws:policy/CloudWatchFullAccess" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --permission-set-arn "<operations-permission-set-arn>" \
  --managed-policy-arn "arn:aws:iam::aws:policy/AmazonRDSFullAccess" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --permission-set-arn "<operations-permission-set-arn>" \
  --managed-policy-arn "arn:aws:iam::aws:policy/AmazonS3FullAccess" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

#### 2e: SecurityAuditAccess (Security Team)

```bash
aws sso-admin create-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --name "SecurityAuditAccess" \
  --description "Security team - Security Hub, GuardDuty, Config access" \
  --session-duration "PT4H" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

---

### Step 3: Create Groups

| Group Name | Description | Permission Set | Accounts |
|------------|-------------|---------------|----------|
| `DOF-Administrators` | Full admin access | AdministratorAccess | All accounts |
| `DOF-Developers` | Development team | PowerUserAccess | DOF-Development, DOF-Staging |
| `DOF-Operations` | Operations team | OperationsAccess | DOF-Production |
| `DOF-Auditors` | Read-only reviewers | ReadOnlyAccess | All accounts |
| `DOF-Security` | Security team | SecurityAuditAccess | Audit, Log Archive |

```bash
IDENTITY_STORE_ID=$(aws sso-admin list-instances \
  --region ap-southeast-2 \
  --profile dof-management-account \
  --query 'Instances[0].IdentityStoreId' --output text)

aws identitystore create-group \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --display-name "DOF-Administrators" \
  --description "Full administrator access to all DOF accounts" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws identitystore create-group \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --display-name "DOF-Developers" \
  --description "Developer access to Development and Staging accounts" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws identitystore create-group \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --display-name "DOF-Operations" \
  --description "Operations access to Production account" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws identitystore create-group \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --display-name "DOF-Auditors" \
  --description "Read-only access to all accounts" \
  --region ap-southeast-2 \
  --profile dof-management-account

aws identitystore create-group \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --display-name "DOF-Security" \
  --description "Security team access to Audit and Log Archive" \
  --region ap-southeast-2 \
  --profile dof-management-account
```

---

### Step 4: Create Users in Identity Center

**Dependency:** Requires DOF user list from Sir Dennis.

```bash
# Example: Create a user
aws identitystore create-user \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --user-name "<username>" \
  --name '{"GivenName":"<FirstName>","FamilyName":"<LastName>"}' \
  --emails '[{"Value":"<email>","Type":"Work","Primary":true}]' \
  --display-name "<Full Name>" \
  --region ap-southeast-2 \
  --profile dof-management-account

# Add user to group
aws identitystore create-group-membership \
  --identity-store-id "$IDENTITY_STORE_ID" \
  --group-id "<group-id>" \
  --member-id '{"UserId":"<user-id>"}' \
  --region ap-southeast-2 \
  --profile dof-management-account
```

---

### Step 5: Assign Groups to Accounts with Permission Sets

```bash
# DOF-Administrators → AdministratorAccess → All Accounts
for ACCOUNT_ID in 239423589015 100116628655 113307936889 482496759415 536225268327; do
  aws sso-admin create-account-assignment \
    --instance-arn "$INSTANCE_ARN" \
    --target-id "$ACCOUNT_ID" \
    --target-type AWS_ACCOUNT \
    --permission-set-arn "<admin-permission-set-arn>" \
    --principal-type GROUP \
    --principal-id "<dof-administrators-group-id>" \
    --region ap-southeast-2 \
    --profile dof-management-account
done

# DOF-Operations → OperationsAccess → DOF-Production
aws sso-admin create-account-assignment \
  --instance-arn "$INSTANCE_ARN" \
  --target-id 536225268327 \
  --target-type AWS_ACCOUNT \
  --permission-set-arn "<operations-permission-set-arn>" \
  --principal-type GROUP \
  --principal-id "<dof-operations-group-id>" \
  --region ap-southeast-2 \
  --profile dof-management-account

# DOF-Auditors → ReadOnlyAccess → All Accounts
for ACCOUNT_ID in 239423589015 100116628655 113307936889 482496759415 536225268327; do
  aws sso-admin create-account-assignment \
    --instance-arn "$INSTANCE_ARN" \
    --target-id "$ACCOUNT_ID" \
    --target-type AWS_ACCOUNT \
    --permission-set-arn "<readonly-permission-set-arn>" \
    --principal-type GROUP \
    --principal-id "<dof-auditors-group-id>" \
    --region ap-southeast-2 \
    --profile dof-management-account
done

# DOF-Security → SecurityAuditAccess → Audit + Log Archive
for ACCOUNT_ID in 100116628655 113307936889; do
  aws sso-admin create-account-assignment \
    --instance-arn "$INSTANCE_ARN" \
    --target-id "$ACCOUNT_ID" \
    --target-type AWS_ACCOUNT \
    --permission-set-arn "<security-audit-permission-set-arn>" \
    --principal-type GROUP \
    --principal-id "<dof-security-group-id>" \
    --region ap-southeast-2 \
    --profile dof-management-account
done

# DOF-Developers → PowerUserAccess → DOF-Staging + DOF-Development
# (Use new account IDs once created)
```

---

### Step 6: Remove Legacy IAM Users

**⚠️ Only after verifying Identity Center access works.**

```bash
# Audit IAM users
aws iam list-users --profile dof-management-account
aws iam list-users --profile dof-angelica-production-account

# For each user to remove:
aws iam list-access-keys --user-name <username> --profile <profile>
aws iam delete-access-key --user-name <username> --access-key-id <key-id> --profile <profile>
aws iam delete-login-profile --user-name <username> --profile <profile>
aws iam list-attached-user-policies --user-name <username> --profile <profile>
aws iam detach-user-policy --user-name <username> --policy-arn <policy-arn> --profile <profile>
aws iam delete-user --user-name <username> --profile <profile>
```

**Keep one break-glass IAM user per account** with MFA enabled, stored in secure vault.

---

## Part 3: Enable Security Hub

```bash
aws securityhub enable-security-hub \
  --enable-default-standards \
  --region ap-southeast-2 \
  --profile dof-management-account

aws securityhub enable-security-hub \
  --enable-default-standards \
  --region ap-southeast-2 \
  --profile dof-angelica-production-account
```

Enable in DOF-Staging and DOF-Development after account creation.

---

## Access Matrix

| Group | Management | Log Archive | Audit | Operations | Production | Staging | Development |
|-------|-----------|-------------|-------|------------|------------|---------|-------------|
| DOF-Administrators | ✅ Admin | ✅ Admin | ✅ Admin | ✅ Admin | ✅ Admin | ✅ Admin | ✅ Admin |
| DOF-Developers | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ PowerUser | ✅ PowerUser |
| DOF-Operations | ❌ | ❌ | ❌ | ❌ | ✅ Ops | ❌ | ❌ |
| DOF-Auditors | ✅ ReadOnly | ✅ ReadOnly | ✅ ReadOnly | ✅ ReadOnly | ✅ ReadOnly | ✅ ReadOnly | ✅ ReadOnly |
| DOF-Security | ❌ | ✅ SecAudit | ✅ SecAudit | ❌ | ❌ | ❌ | ❌ |

---

## Validation Checklist

### Control Tower
- [ ] Drift resolved (no warnings in console)
- [ ] Production OU created and registered
- [ ] Development OU created and registered
- [ ] DOF-Production account renamed and moved to Production OU
- [ ] DOF-Staging account created in Production OU
- [ ] DOF-Development account created in Development OU
- [ ] SCPs applied to all OUs
- [ ] Old empty OUs cleaned up

### IAM Identity Center
- [ ] 5 permission sets created
- [ ] 5 groups created
- [ ] Users created (pending DOF user list)
- [ ] Users assigned to correct groups
- [ ] Groups assigned to accounts with correct permission sets
- [ ] Test login via Identity Center portal
- [ ] Legacy IAM users removed (after verification)
- [ ] Break-glass users configured with MFA

### Security Hub
- [ ] Enabled in all accounts
- [ ] Security standards configured

---

## Rollback Plan

| Issue | Rollback Action |
|-------|----------------|
| OU creation fails | Delete OU, retry |
| Account provisioning fails | Check Service Catalog errors, retry |
| Account move fails | Move back to original OU |
| Permission set wrong | Update or delete and recreate |
| User can't access | Check group membership and account assignment |
| Identity Center issues | Legacy IAM users still available (don't delete until verified) |

---

## Timeline Estimate

| Task | Duration | Dependencies |
|------|----------|-------------|
| Resolve drift | 30-60 min | None |
| Create OUs (2) | 20-30 min | Drift resolved |
| Register OUs with Control Tower | 20-30 min | OUs created |
| Rename account | 5 min | None |
| Move account to Production OU | 5-10 min | Production OU registered |
| Create DOF-Staging | 20-30 min | Production OU registered |
| Create DOF-Development | 20-30 min | Development OU registered |
| Create permission sets (5) | 30 min | None |
| Create groups (5) | 15 min | None |
| Create users | 30-60 min | DOF user list received |
| Assign groups to accounts | 30 min | All above complete |
| Testing & validation | 60 min | All above complete |
| Enable Security Hub | 15 min | Accounts created |
| **Total Estimated Time** | **4-6 hours** | |

---

**Prepared by:** Sagesoft Solutions
**Version:** 1.0
**Last Updated:** March 16, 2026
