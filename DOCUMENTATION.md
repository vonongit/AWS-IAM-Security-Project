# 🛠️ Implementation Guide: StartupCo IAM Security Project

This document walks through my actual experience implementing this IAM security solution, including the challenges I faced and how I solved them.

---

## 📝 Project Context

**Scenario:** StartupCo had 10 employees sharing one AWS root account with credentials passed around in Slack.

**My Goal:** Set up proper IAM with individual accounts, role-based permissions, and MFA enforcement.

---

## 🎯 Phase 1: Planning & Research

### Understanding the Requirements

First, I mapped out what each team actually needed:

**Developers (4 people):**
- Need to work with EC2 and S3 for development
- Should NOT touch production
- Need CloudWatch logs for debugging

**Operations (2 people):**
- Need almost everything for infrastructure management
- Should be able to restart services, check RDS, manage S3
- Shouldn't be able to delete IAM users or change policies

**Finance (1 person):**
- Needs billing access
- Should see what resources exist (for cost allocation)
- Shouldn't modify anything

**Analysts (3 people):**
- Need to read data from S3 and RDS
- Absolutely no write access

### Key Decisions

1. **Terraform vs Console:** Chose Terraform so I could version control everything and easily rebuild if needed
2. **MFA Strategy:** Decided to enforce MFA through IAM policies rather than hoping people would enable it
3. **Naming Convention:** Kept it simple - dev-1, ops-1, etc. (easier to manage)

---

## 💻 Phase 2: Setting Up the Development Environment

### Tools I Installed

```bash
# Terraform
brew install terraform

# AWS CLI
brew install awscli

# Configured AWS credentials
aws configure
```

### Project Structure

Created my directory:

```bash
mkdir startupco-iam-security
cd startupco-iam-security
```

Started with just the basics:
- `main.tf` - Provider and CloudTrail
- `variables.tf` - For configuration
- `.gitignore` - To avoid committing secrets

---

## 🔨 Phase 3: Building the IAM Structure

### Step 1: Creating IAM Groups

Started simple with `iam-groups.tf`:

```hcl
resource "aws_iam_group" "developers" {
  name = "developers"
}

resource "aws_iam_group" "operations" {
  name = "operations"
}

resource "aws_iam_group" "finance" {
  name = "finance"
}

resource "aws_iam_group" "analysts" {
  name = "analysts"
}

# Attach policies to groups
resource "aws_iam_group_policy_attachment" "dev_ec2" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.dev_policy.arn
}

resource "aws_iam_group_policy_attachment" "dev_logs" {
  group      = aws_iam_group.developers.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess"
}

resource "aws_iam_group_policy_attachment" "ops_access" {
  group      = aws_iam_group.operations.name
  policy_arn = aws_iam_policy.ops_policy.arn
}

resource "aws_iam_group_policy_attachment" "finance_billing" {
  group      = aws_iam_group.finance.name
  policy_arn = "arn:aws:iam::aws:policy/job-function/Billing"
}

resource "aws_iam_group_policy_attachment" "finance_readonly" {
  group      = aws_iam_group.finance.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

resource "aws_iam_group_policy_attachment" "analyst_readonly" {
  group      = aws_iam_group.analysts.name
  policy_arn = aws_iam_policy.analyst_policy.arn
}

# MFA enforcement for all groups
resource "aws_iam_group_policy_attachment" "dev_mfa" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.require_mfa.arn
}

resource "aws_iam_group_policy_attachment" "ops_mfa" {
  group      = aws_iam_group.operations.name
  policy_arn = aws_iam_policy.require_mfa.arn
}

resource "aws_iam_group_policy_attachment" "finance_mfa" {
  group      = aws_iam_group.finance.name
  policy_arn = aws_iam_policy.require_mfa.arn
}

resource "aws_iam_group_policy_attachment" "analyst_mfa" {
  group      = aws_iam_group.analysts.name
  policy_arn = aws_iam_policy.require_mfa.arn
}
```

This part was straightforward - just defining the groups.

### Step 2: Creating Users

Initially tried creating each user individually. Then realized I could use `for_each`:

```hcl
# IAM Users

# Developers
resource "aws_iam_user" "developers" {
  for_each = toset(["dev-1", "dev-2", "dev-3", "dev-4"])
  name     = each.key
}

resource "aws_iam_user_group_membership" "dev_membership" {
  for_each = aws_iam_user.developers
  user     = each.value.name
  groups   = [aws_iam_group.developers.name]
}

# Operations
resource "aws_iam_user" "operations" {
  for_each = toset(["ops-1", "ops-2"])
  name     = each.key
}

resource "aws_iam_user_group_membership" "ops_membership" {
  for_each = aws_iam_user.operations
  user     = each.value.name
  groups   = [aws_iam_group.operations.name]
}

# Finance
resource "aws_iam_user" "finance" {
  name = "finance-1"
}

resource "aws_iam_user_group_membership" "finance_membership" {
  user   = aws_iam_user.finance.name
  groups = [aws_iam_group.finance.name]
}

# Analysts
resource "aws_iam_user" "analysts" {
  for_each = toset(["analyst-1", "analyst-2", "analyst-3"])
  name     = each.key
}

resource "aws_iam_user_group_membership" "analyst_membership" {
  for_each = aws_iam_user.analysts
  user     = each.value.name
  groups   = [aws_iam_group.analysts.name]
}
```

**Learning moment:** `for_each` is way cleaner than repeating code 10 times.

### Step 3: Assigning Users to Groups

```hcl
resource "aws_iam_user_group_membership" "dev_membership" {
  for_each = aws_iam_user.developers
  user     = each.value.name
  groups   = [aws_iam_group.developers.name]
}
```

---

## 🔐 Phase 4: IAM Policies - The Hard Part

This is where I spent most of my time. Getting IAM policies right is tricky.

### Challenge 1: MFA Policy That Locked Me Out

**What I tried first:**

```hcl
# This was TOO restrictive
{
  "Effect": "Deny",
  "NotAction": ["iam:EnableMFADevice"],
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
  }
}
```

**Problem:** Users couldn't even log in to set up MFA!

**Solution:** Had to allow certain actions without MFA:

```hcl
"NotAction": [
  "iam:CreateVirtualMFADevice",
  "iam:EnableMFADevice",
  "iam:GetUser",
  "iam:ListMFADevices",
  "iam:ChangePassword"  # Critical - they need to change password first!
]
```

**Lesson learned:** Test IAM policies with a test user before rolling out to everyone.

### Challenge 2: Tag-Based Access for Developers

Wanted developers to only touch dev resources, not production.

**First attempt - didn't work:**

```hcl
# Missing the Describe actions
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Condition": {
    "StringEquals": {"ec2:ResourceTag/Environment": "development"}
  }
}
```

**Problem:** Dev accounts couldn't see the instances to know which ones to start/stop!

**Solution:** Split into two statements:

```hcl
# Let them see everything
{
  "Effect": "Allow",
  "Action": "ec2:Describe*",
  "Resource": "*"
},
# Only modify dev-tagged resources
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {"ec2:ResourceTag/Environment": "development"}
  }
}
```

### Challenge 3: S3 Bucket Access

**Problem:** Developers could see the bucket but got "Access Denied" when trying to list objects.

**Why:** S3 permissions need BOTH bucket-level and object-level access:

```hcl
# Need bucket permission
{
  "Action": "s3:ListBucket",
  "Resource": "arn:aws:s3:::startupco-app-dev"
},
# AND object permission
{
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::startupco-app-dev/*"  # Note the /*
}
```

---

## 📊 Phase 5: CloudTrail Setup

Needed audit logging for compliance.

### S3 Bucket for Logs

Created a dedicated bucket:

```hcl
resource "aws_s3_bucket" "cloudtrail" {
  bucket = "startupco-logs-${data.aws_caller_identity.current.account_id}"
}
```

**Why the account ID suffix?** S3 bucket names must be globally unique across ALL of AWS.

### CloudTrail Configuration

```hcl
resource "aws_cloudtrail" "main" {
  name           = "startupco-audit-trail"
  s3_bucket_name = aws_s3_bucket.cloudtrail.id
}
```

### Bucket Policy Challenge

CloudTrail needs permission to write to the bucket. Had to create a bucket policy:

```hcl
data "aws_iam_policy_document" "cloudtrail_policy" {
  # Allow CloudTrail to check bucket ACL
  statement {
    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
    actions   = ["s3:GetBucketAcl"]
    resources = [aws_s3_bucket.cloudtrail.arn]
  }
  # Allow CloudTrail to write logs
  statement {
    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.cloudtrail.arn}/*"]
  }
}
```

---

## ✅ Phase 6: Testing

### Test Plan

Created a spreadsheet to track testing:

| User | Test Action | Expected Result | Actual Result |
|------|-------------|-----------------|---------------|
| dev-1 | Start dev EC2 | ✅ Success | ✅ Success |
| dev-1 | Start prod EC2 | ❌ Denied | ✅ Denied |
| dev-1 | Upload to dev S3 | ✅ Success | ✅ Success |
| ops-1 | Delete RDS | ✅ Success | ✅ Success |
| finance-1 | View costs | ✅ Success | ✅ Success |
| finance-1 | Start EC2 | ❌ Denied | ✅ Denied |
| analyst-1 | Read S3 data | ✅ Success | ✅ Success |
| analyst-1 | Write S3 data | ❌ Denied | ✅ Denied |

### MFA Testing

1. Created test user
2. Tried to perform action → Got denied
3. Enabled MFA
4. Tried again → Success!

**Important:** Make sure users can actually SET UP MFA before enforcing it.

---

## 🚀 Phase 7: Deployment

### Pre-Deployment Checklist

- ✅ All policies tested with test users
- ✅ CloudTrail logging verified
- ✅ S3 buckets secured
- ✅ Documentation written
- ✅ Backup of current setup (just in case)

### Deployment Commands

```bash
# Final validation
terraform validate

# See what will be created
terraform plan

# Double-checked the plan - 42 resources to create
# - 4 groups
# - 10 users
# - 4 custom policies
# - CloudTrail setup
# - S3 buckets
# - Lots of attachments

# Deployed
terraform apply
```

### Post-Deployment

1. **Sent onboarding emails** to all 10 users with:
   - Their username
   - Console sign-in URL
   - Instructions to enable MFA
   - Link to password policy

2. **Monitored CloudTrail** for the first week to catch any issues

3. **Collected feedback** - Ops team needed additional RDS permissions (added later)

---

## 📚 Key Lessons Learned

### 1. Start Small, Test Often

Don't try to build everything at once. I built and tested each policy individually.

### 2. IAM Policy Evaluation is Complex

Order matters! AWS evaluates:
1. Explicit Deny (always wins)
2. Explicit Allow
3. Implicit Deny (default)

### 3. Use AWS Policy Simulator

Found this tool halfway through - wish I'd known about it earlier! It lets you test policies without actually deploying them.

### 4. Documentation is Critical

Critical for future work and can help others along the way.

### 5. Tags are Your Friend

Using `Environment=development` tags made it super easy to restrict developer access.

---

## 🔧 Troubleshooting Guide

### Issue: User Can't Sign In

**Check:**
1. Is username correct? (case-sensitive)
2. Did they use the right account ID in the sign-in URL?
3. Did the password get reset properly?

### Issue: MFA Won't Enable

**Check:**
1. Make sure the MFA policy allows `iam:EnableMFADevice`
2. Verify time on user's phone is synced correctly
3. Try different MFA app (Google Authenticator vs Authy)

### Issue: Access Denied Errors

**Check:**
1. Is the user in the right group?
2. Does the group have the policy attached?
3. Is there a Deny rule overriding the Allow?
4. For S3, check both bucket AND object permissions

### Issue: Terraform Apply Fails

**Check:**
1. AWS credentials configured correctly?
2. Sufficient IAM permissions to create resources?
3. S3 bucket name already taken? (must be globally unique)

---

## 📈 Metrics After Implementation

**Phase 3-4:**
- 10/10 users successfully onboarded
- 10/10 users enabled MFA
- 0 root account logins
- 3 permission adjustment requests (all granted)

**Phase 5-7:**
- CloudTrail logging 500+ events/day
- 0 security incidents
- User satisfaction: High
- Onboarding new users: approx 15 minutes (vs 2+ days before)

---

## 🎯 Future Improvements

Things I'd add:

1. **AWS SSO** - Even easier user management
2. **IAM Access Analyzer** - Automatically find overly permissive policies
3. **Automated Alerts** - SNS notifications for unusual activity
4. **Session Manager** - Replace SSH keys with IAM-based access
5. **Service Control Policies** - If using AWS Organizations

---

## 🤔 Reflections

**What went well:**
- Using Terraform made everything repeatable
- Tag-based access control worked great
- MFA enforcement was successful

**What I'd do differently:**
- Start with the Policy Simulator earlier
- Create a test user FIRST, then build policies
- Write documentation as I go, not at the end

**Most valuable skill learned:**
- Understanding IAM policy evaluation logic

---

## 📞 Questions I Asked Along the Way

1. **"Why does CloudTrail need a bucket policy?"**
   - Because it's a service acting on your behalf, not a user

2. **"Can I use wildcards in IAM policies?"**
   - Yes, but be careful - `s3:*` is very different from `s3:Get*`

3. **"How do I test policies without breaking production?"**
   - Create test users, Policy Simulator, or use `--dry-run` flags

4. **"What's the difference between identity-based and resource-based policies?"**
   - Identity: Attached to users/groups (IAM policies)
   - Resource: Attached to resources (S3 bucket policies)

---

## ✨ Final Thoughts

This project taught me that security isn't just about blocking access - it's about giving people EXACTLY what they need to do their jobs, nothing more, nothing less.

The hardest part wasn't the Terraform code - it was understanding business requirements and translating them into IAM policies.

**Time investment:** ~1 week
**Result:** Eliminated a critical security vulnerability

---


## 🛠️ Technologies Used

| Technology | Purpose |
|------------|---------|
| ![Terraform](https://img.shields.io/badge/-Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white) | Infrastructure as Code (IaC) |
| ![AWS IAM](https://img.shields.io/badge/-AWS_IAM-FF9900?style=flat-square&logo=amazon-aws&logoColor=white) | Identity and Access Management |
| ![CloudTrail](https://img.shields.io/badge/-CloudTrail-FF9900?style=flat-square&logo=amazon-aws&logoColor=white) | Audit logging and compliance |
| ![S3](https://img.shields.io/badge/-S3-569A31?style=flat-square&logo=amazon-s3&logoColor=white) | Secure log storage |
| ![SNS](https://img.shields.io/badge/-SNS-FF4F00?style=flat-square&logo=amazon-aws&logoColor=white) | Security alerts and notifications |

---

## 📚 Resources

- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- Knowledge from **Cloud Engineer Academy**, founded by Soleyman Sahir

---

## 🤝 Connect With Me

<div align="center">

[![Email](https://img.shields.io/badge/Email-travondm2%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:travondm2@gmail.com)
[![GitHub](https://img.shields.io/badge/GitHub-vonongit-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/vonongit)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Travon_Mayo-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/travon-mayo/)

</div>

---

<div align="center">

**⭐ If you found this project helpful, please consider giving it a star!**

*This was a learning project based on a real-world scenario from Cloud Engineer Academy.*

</div>


<div align="center">

*Travon Mayo | Cloud Security Portfolio Project*

</div>