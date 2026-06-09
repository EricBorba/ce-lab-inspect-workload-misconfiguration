# Lab M8.02 — Inspect Workload Misconfiguration Examples

![AWS Config](https://img.shields.io/badge/AWS-Config-FF9900?logo=amazonaws&logoColor=white)
![AWS IAM](https://img.shields.io/badge/AWS-IAM-DD344C?logo=amazonaws&logoColor=white)
![Amazon S3](https://img.shields.io/badge/AWS-S3-FF9900?logo=amazons3&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS-EC2%20Security%20Groups-FF9900?logo=amazonec2&logoColor=white)
![Security](https://img.shields.io/badge/Security-STRIDE%20Threat%20Model-red?logo=amazonaws&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-CLI%20Only-4EAA25?logo=gnubash&logoColor=white)

Hands-on lab intentionally creating and remediating cloud security misconfigurations, detecting them with AWS Config, and documenting risks using the STRIDE threat model.

---

## What This Lab Does

Three intentional misconfigurations are created, detected with AWS Config rules, then remediated and verified:

| # | Misconfiguration | Detection Method | STRIDE |
|---|-----------------|-----------------|--------|
| 1 | S3 bucket with Block Public Access disabled | AWS Config `s3-bucket-public-read-prohibited` | I |
| 2 | Security group with SSH open to 0.0.0.0/0 | AWS Config `restricted-ssh` | S |
| 3 | IAM role with AdministratorAccess | Manual review | E |

---

## Files

| File | Description |
|------|-------------|
| `findings-report.md` | Full misconfiguration findings with real resource IDs, screenshots, and remediations |
| `threat-model.md` | STRIDE threat model for a simple web application (S3 + ALB + EC2 + RDS) |
| `app-policy.json` | Least-privilege IAM policy applied during IAM remediation |
| `cli-commands.md` | Chronological log of all CLI commands executed during the lab |
| `screenshots/` | Evidence screenshots for all misconfigurations and compliance results |

---

## Screenshots

| File | Content |
|------|---------|
| `01-s3-misconfiguration-bucket-public-access-disabled.png` | S3 bucket creation with Block Public Access disabled |
| `02-security-group-ssh-open-to-internet.png` | Security group created with SSH from 0.0.0.0/0 |
| `03-iam-role-administrator-access-attached.png` | IAM role with AdministratorAccess policy attached |
| `04-config-role-and-s3-bucket-setup.png` | Config IAM role and S3 log bucket creation |
| `05-config-recorder-delivery-channel-started.png` | Config recorder and delivery channel enabled |
| `06-config-rules-s3-and-ssh-created.png` | Config rules created for S3 and SSH detection |
| `07-compliance-non-compliant-before-remediation.png` | Both rules NON_COMPLIANT before remediation |
| `08-security-group-ssh-remediated.png` | SSH rule revoked and restricted CIDR applied |
| `09-iam-role-least-privilege-applied.png` | AdministratorAccess detached, least-privilege policy applied |
| `10-s3-bucket-compliant-after-remediation.png` | S3 bucket COMPLIANT after Block Public Access re-enabled |
| `11-security-group-compliant-after-remediation.png` | Security group COMPLIANT after SSH restriction |

---

## Key Findings

- The `restricted-ssh` Config rule found **6 non-compliant security groups** in the account — not just the one created for this lab. This highlights how security groups accumulate open rules over time.
- Modern AWS accounts enforce **BucketOwnerEnforced** Object Ownership by default, disabling ACLs entirely. This is an additional protection layer beyond Block Public Access.
- AWS Config was set up from scratch — the lab's original instructions had a missing IAM role and incorrect recorder syntax, both fixed during execution.
