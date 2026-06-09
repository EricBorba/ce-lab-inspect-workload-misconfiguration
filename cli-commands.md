# Lab M8.02 - CLI Commands Log

## Step 1: Create Intentional Misconfigurations

### Misconfiguration 1: Public S3 Bucket
```bash
aws s3 mb s3://my-insecure-test-bucket-1780998804

aws s3api put-public-access-block \
  --bucket my-insecure-test-bucket-1780998804 \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

echo "This is sensitive data" > test-file.txt && aws s3 cp test-file.txt s3://my-insecure-test-bucket-1780998804/
```

### Misconfiguration 2: Overly Permissive Security Group
```bash
aws ec2 create-security-group \
  --group-name insecure-ssh-sg \
  --description "Intentionally insecure SG for lab"

aws ec2 authorize-security-group-ingress \
  --group-name insecure-ssh-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

### Misconfiguration 3: IAM Role with AdministratorAccess
```bash
aws iam create-role \
  --role-name InsecureAppRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name InsecureAppRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

## Step 2: Detect Misconfigurations with AWS Config

### Create Config IAM Role
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

### Create Config S3 Bucket and Policy
```bash
aws s3 mb s3://aws-config-bucket-1780999315

aws s3api put-bucket-policy \
  --bucket aws-config-bucket-1780999315 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AWSConfigBucketPermissionsCheck",
        "Effect": "Allow",
        "Principal": {"Service": "config.amazonaws.com"},
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::aws-config-bucket-1780999315"
      },
      {
        "Sid": "AWSConfigBucketDelivery",
        "Effect": "Allow",
        "Principal": {"Service": "config.amazonaws.com"},
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::aws-config-bucket-1780999315/AWSLogs/871939031886/Config/*",
        "Condition": {
          "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
        }
      }
    ]
  }'
```

### Enable Config Recorder
```bash
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "aws-config-bucket-1780999315"
  }'

aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::871939031886:role/config-role",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResourceTypes": true
    }
  }'

aws configservice start-configuration-recorder \
  --configuration-recorder-name default
```

### Add Config Rules
```bash
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'

aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "restricted-ssh",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "INCOMING_SSH_DISABLED"
    }
  }'
```

### Check Compliance (NON_COMPLIANT - before remediation)
```bash
aws configservice describe-compliance-by-config-rule \
  --config-rule-names s3-bucket-public-read-prohibited restricted-ssh

aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --compliance-types NON_COMPLIANT
```

## Step 4: Remediate Misconfigurations

### Remediation 1: Secure S3 Bucket
```bash
aws s3api put-public-access-block \
  --bucket my-insecure-test-bucket-1780998804 \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

aws s3api put-bucket-encryption \
  --bucket my-insecure-test-bucket-1780998804 \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

### Remediation 2: Restrict Security Group
```bash
aws ec2 revoke-security-group-ingress \
  --group-name insecure-ssh-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-name insecure-ssh-sg \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/24
```

### Remediation 3: Apply Least Privilege IAM
```bash
aws iam detach-role-policy \
  --role-name InsecureAppRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam put-role-policy \
  --role-name InsecureAppRole \
  --policy-name AppLeastPrivilegePolicy \
  --policy-document file://app-policy.json
```

### Verify Compliance (COMPLIANT - after remediation)
```bash
aws configservice start-config-rules-evaluation \
  --config-rule-names s3-bucket-public-read-prohibited restricted-ssh

aws configservice get-compliance-details-by-resource \
  --resource-type AWS::S3::Bucket \
  --resource-id my-insecure-test-bucket-1780998804

aws configservice get-compliance-details-by-resource \
  --resource-type AWS::EC2::SecurityGroup \
  --resource-id sg-095eee4a02ee83c98
```

## Cleanup

```bash
aws s3 rb s3://my-insecure-test-bucket-1780998804 --force

aws ec2 delete-security-group --group-name insecure-ssh-sg

aws iam delete-role-policy --role-name InsecureAppRole --policy-name AppLeastPrivilegePolicy
aws iam delete-role --role-name InsecureAppRole

aws configservice delete-config-rule --config-rule-name s3-bucket-public-read-prohibited
aws configservice delete-config-rule --config-rule-name restricted-ssh

aws configservice stop-configuration-recorder --configuration-recorder-name default
aws configservice delete-configuration-recorder --configuration-recorder-name default

aws configservice delete-delivery-channel --delivery-channel-name default

aws s3 rb s3://aws-config-bucket-1780999315 --force
aws iam detach-role-policy --role-name config-role --policy-arn arn:aws:iam::aws:policy/service-role/AWS_ConfigRole
aws iam delete-role --role-name config-role
```
