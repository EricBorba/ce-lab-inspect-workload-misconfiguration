# Lab M8.02 — Instruction Fixes & Corrections

This document records issues found in the original lab instructions and the fixes applied during execution.

---

## Fix 1: Missing IAM Role for AWS Config

**Where:** Step 2 — "Enable AWS Config"

**Original instruction:**
```bash
aws configservice put-configuration-recorder \
  --configuration-recorder \
  name=default,roleARN=arn:aws:iam::YOUR_ACCOUNT:role/config-role
```

**Problem:** The lab references `config-role` but never creates it. AWS Config requires a service-linked IAM role with permissions to read resource configurations and write to S3. Running the command as written would fail with `InvalidRoleException`.

**Fix — add these two commands before enabling Config:**
```bash
aws iam create-role \
  --role-name config-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "config.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name config-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWS_ConfigRole
```

---

## Fix 2: Missing S3 Bucket Policy for Config Delivery

**Where:** Step 2 — "Create S3 bucket for Config logs"

**Original instruction:**
```bash
aws s3 mb s3://aws-config-bucket-$(date +%s)
```

**Problem:** The lab creates the bucket but does not add a bucket policy. AWS Config requires explicit `s3:GetBucketAcl` and `s3:PutObject` permissions on the bucket to deliver configuration snapshots. Without the policy, the delivery channel setup silently fails or Config cannot write logs.

**Fix — add a bucket policy immediately after `s3 mb`:**
```bash
aws s3api put-bucket-policy \
  --bucket <config-bucket-name> \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AWSConfigBucketPermissionsCheck",
        "Effect": "Allow",
        "Principal": {"Service": "config.amazonaws.com"},
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::<config-bucket-name>"
      },
      {
        "Sid": "AWSConfigBucketDelivery",
        "Effect": "Allow",
        "Principal": {"Service": "config.amazonaws.com"},
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::<config-bucket-name>/AWSLogs/<account-id>/Config/*",
        "Condition": {
          "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
        }
      }
    ]
  }'
```

---

## Fix 3: Missing Delivery Channel Creation

**Where:** Step 2 — between "Create S3 bucket" and "Enable Config"

**Original instruction:** Jumps directly from bucket creation to `put-configuration-recorder`.

**Problem:** AWS Config requires a delivery channel to know where to send configuration snapshots. Without it, `start-configuration-recorder` fails with `NoAvailableDeliveryChannelException`.

**Fix — add this command before `put-configuration-recorder`:**
```bash
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "<config-bucket-name>"
  }'
```

---

## Fix 4: Broken `--configuration-recorder` Syntax

**Where:** Step 2 — "Enable Config"

**Original instruction:**
```bash
aws configservice put-configuration-recorder \
  --configuration-recorder \
  name=default,roleARN=arn:aws:iam::YOUR_ACCOUNT:role/config-role
```

**Problem:** The shorthand `key=value,key=value` format is not valid for this parameter. The AWS CLI expects a JSON object. Running the original command produces a parsing error.

**Fix — use JSON format:**
```bash
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::<account-id>:role/config-role",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResourceTypes": true
    }
  }'
```

---

## Fix 5: `put-object-acl` Fails on Modern AWS Accounts

**Where:** Step 1 — Misconfiguration 1, "Make it public"

**Original instruction:**
```bash
aws s3api put-object-acl \
  --bucket my-insecure-test-bucket-1234567890 \
  --key test-file.txt \
  --acl public-read
```

**Problem:** Modern AWS accounts have **Object Ownership set to `BucketOwnerEnforced`** by default, which disables ACLs entirely. Running this command returns:
```
An error occurred (AccessControlListNotSupported) when calling the PutObjectAcl operation: The bucket does not allow ACLs
```

**Impact:** This is not a lab-breaking issue — the misconfiguration goal (Block Public Access disabled) is still achieved and detected by AWS Config. However, the lab does not mention this behavior or explain why the command fails, which is confusing.

**Clarification to add to the lab:** The `put-object-acl` step may fail on accounts created after April 2023. The AWS Config rule `s3-bucket-public-read-prohibited` detects Block Public Access being disabled, which is the actual risk — ACL-based public access is a separate (older) mechanism.

---

## Fix 6: `YOUR_ACCOUNT` Placeholder With No Guidance

**Where:** Step 2 — multiple commands

**Original instruction:** Uses `YOUR_ACCOUNT` as a literal placeholder without explaining how to retrieve it.

**Fix — retrieve the account ID first:**
```bash
aws sts get-caller-identity --query Account --output text
```

Use the returned 12-digit value everywhere `YOUR_ACCOUNT` appears in the lab instructions.
