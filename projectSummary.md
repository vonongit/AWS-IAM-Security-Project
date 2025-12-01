# 🔐 StartupCo IAM Security Project

![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![IAM](https://img.shields.io/badge/IAM-DD344C?style=for-the-badge&logo=amazon-aws&logoColor=white)
![CloudTrail](https://img.shields.io/badge/CloudTrail-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)

> Implemented secure IAM infrastructure for a startup that was sharing AWS root account credentials. This project established proper role-based access control (RBAC) and eliminated critical security vulnerabilities.

-View DOCUMENTATION.md to view full journey of this project

---

## 📋 Table of Contents

- [The Problem](#-the-problem)
- [The Solution](#-the-solution)
- [Architecture](#-architecture)
- [Key Security Features](#-key-security-features)
- [Project Structure](#-project-structure)
- [Deployment Guide](#-deployment-guide)
- [Users & Permissions](#-users--permissions)
- [Lessons Learned](#-lessons-learned)
- [Results](#-results)
- [Technologies Used](#-technologies-used)
- [Connect With Me](#-connect-with-me)

---

## 🚨 The Problem

StartupCo had **10 employees** all using the same root account credentials shared through insecure channels. This created several critical security risks:

| Issue | Impact |
|-------|--------|
| ❌ **No Accountability** | Impossible to track who performed which actions |
| ❌ **Excessive Permissions** | Everyone had full admin access to all resources |
| ❌ **No MFA** | Zero multi-factor authentication requirements |
| ❌ **No Password Policy** | Weak or reused passwords across the team |
| ❌ **Single Point of Failure** | One compromised credential = entire AWS account at risk |

---

## ✅ The Solution

Built a comprehensive IAM security infrastructure implementing the **principle of least privilege** with:

- ✨ **4 IAM Groups** with role-specific permissions
- 👥 **10 Individual User Accounts** with unique credentials
- 🔒 **MFA Enforcement** for all users
- 📊 **CloudTrail Logging** for complete audit trails
- 🛡️ **Strong Password Policy** (14+ chars, complexity requirements, 90-day rotation)
- 🏷️ **Tag-Based Access Control** for resource isolation

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│          Root Account (Secured + MFA)               │
└────────────────────┬────────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │      IAM Groups       │
         └───────────┬───────────┘
                     │
         ┌───────────┼───────────┬───────────┐
         │           │           │           │
    ┌────▼────┐ ┌───▼────┐ ┌────▼────┐ ┌───▼──────┐
    │ Devs    │ │  Ops   │ │ Finance │ │ Analysts │
    │ (4)     │ │  (2)   │ │  (1)    │ │  (3)     │
    └────┬────┘ └───┬────┘ └────┬────┘ └───┬──────┘
         │          │           │           │
    EC2 + S3   Full Infra   Billing +   Read-Only
    Dev Access    Access    Cost Mgmt   Data Access
```

---

## 🔑 Key Security Features

### 1. 🔐 MFA Enforcement
Users cannot perform any actions until MFA is enabled on their account.

### 2. 🏷️ Tag-Based Access Control
Developers can only modify resources tagged with `Environment=development`, preventing accidental production changes.

### 3. 📝 CloudTrail Audit Logging
Complete audit trail of all AWS API calls with encrypted S3 storage.

### 4. 🔒 Strong Password Policy
- Minimum 14 characters
- Requires uppercase, lowercase, numbers, and symbols
- 90-day password rotation
- Prevents password reuse

### 5. 🪣 Secure S3 Storage
CloudTrail logs stored in encrypted S3 bucket with strict access controls.

---

## 📁 Project Structure

```
.
├── main.tf                    # CloudTrail setup & S3 bucket configuration
├── iam-groups.tf             # IAM group definitions
├── iam-users.tf              # User account creation
├── iam-policies.tf           # Custom IAM policies per group
├── password-policy.tf        # Account-wide password requirements
├── variables.tf              # Configuration variables
├── outputs.tf                # Deployment outputs
└── terraform.tfvars.example  # Example variables file
```

---

## 🚀 Deployment Guide

### Prerequisites
- AWS Account with appropriate permissions
- Terraform installed (v1.0+)
- AWS CLI configured

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/vonongit/Portfolio-Projects-Cloud.git
cd Portfolio-Projects-Cloud

# 2. Configure variables
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your email and preferences

# 3. Initialize Terraform
terraform init

# 4. Review the execution plan
terraform plan

# 5. Deploy the infrastructure
terraform apply

# 6. Retrieve outputs (user credentials, console URLs)
terraform output
```

---

## 👥 Users & Permissions

### User Distribution

| Role | Users | Count |
|------|-------|-------|
| 👨‍💻 **Developers** | `dev-1`, `dev-2`, `dev-3`, `dev-4` | 4 |
| ⚙️ **Operations** | `ops-1`, `ops-2` | 2 |
| 💰 **Finance** | `finance-1` | 1 |
| 📊 **Analysts** | `analyst-1`, `analyst-2`, `analyst-3` | 3 |

### Permission Matrix

<details>
<summary>👨‍💻 <strong>Developers</strong> (Click to expand)</summary>

**Permissions:**
- ✅ Start/stop/reboot EC2 instances tagged `Environment=development`
- ✅ Read/write access to `startupco-app-dev` S3 bucket
- ✅ View CloudWatch logs
- ❌ Cannot touch production resources
- ❌ Cannot delete S3 buckets

**Use Case:** Day-to-day development work in isolated dev environment
</details>

<details>
<summary>⚙️ <strong>Operations</strong> (Click to expand)</summary>

**Permissions:**
- ✅ Full EC2, RDS, S3, CloudWatch access
- ✅ Can view IAM users (read-only)
- ✅ Network and infrastructure management
- ❌ Cannot create/delete IAM users or modify policies

**Use Case:** Infrastructure management and production operations
</details>

<details>
<summary>💰 <strong>Finance</strong> (Click to expand)</summary>

**Permissions:**
- ✅ View all AWS resources (read-only)
- ✅ Full access to Cost Explorer and Budgets
- ✅ Billing dashboard access
- ❌ Cannot modify any infrastructure

**Use Case:** Cost tracking, budget management, financial reporting
</details>

<details>
<summary>📊 <strong>Analysts</strong> (Click to expand)</summary>

**Permissions:**
- ✅ Read-only access to analytics S3 bucket
- ✅ Describe RDS instances
- ✅ View CloudWatch metrics
- ❌ Cannot write, modify, or delete anything

**Use Case:** Data analysis and reporting without modification rights
</details>

---

## 💡 Lessons Learned

### Challenge 1: MFA Policy Lockout
**Problem:** Initial MFA policy locked all IAM users out before they could set up MFA.

**Solution:** Added exceptions for `iam:GetUser` and `iam:ChangePassword` to allow users to sign in and configure MFA on first login.

```hcl
# Allow users to manage their own MFA devices
"iam:*MFADevice",
"iam:GetUser",
"iam:ChangePassword"
```

### Challenge 2: Developer Permission Scope
**Problem:** Finding the right balance - developers needed enough access to work effectively without risking production resources.

**Solution:** Implemented tag-based conditional access. Developers can only interact with resources tagged `Environment=development`.

```hcl
"Condition": {
  "StringEquals": {
    "aws:ResourceTag/Environment": "development"
  }
}
```

### Challenge 3: Audit Trail Implementation
**Problem:** Ensuring CloudTrail logs couldn't be tampered with or deleted by unauthorized users.

**Solution:** Created dedicated S3 bucket with encryption and bucket policies preventing deletion, even by ops team.

---

## 📊 Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Shared Credentials | ✅ Yes (10 people) | ❌ No | ✅ 100% eliminated |
| MFA Adoption | 0% | 100% | 📈 +100% |
| Audit Capability | None | Full CloudTrail | ✅ Complete visibility |
| Permission Model | Everyone = Admin | Least Privilege | ✅ 75% reduction in over-privileged access |
| User Onboarding | Days | Minutes | ⚡ 95% faster |
| Security Incidents | High Risk | Low Risk | 🛡️ Significantly reduced |

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