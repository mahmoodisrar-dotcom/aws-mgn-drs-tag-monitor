# AWS Tag Change Monitor for MGN and DRS

**Automated monitoring and alerting for critical AWS Application Migration Service (MGN) and AWS Elastic Disaster Recovery (DRS) tag modifications.**

---

## The Problem

AWS MGN and DRS use **system-managed tags** to track and manage migration and disaster recovery infrastructure. When these tags are manually modified — whether intentionally or accidentally — serious consequences can follow:

- **Replication stalls** across source servers
- **Instances get terminated** — MGN/DRS reconciliation can terminate ANY instance with service tags that is not recognized as an active replicator
- **Non-managed instances are at risk** — Even manually-created EC2 instances (not part of any migration or DR workflow) can be terminated if someone applies service tags to them
- **Data loss** — If deleteOnTermination=true on root volumes, termination results in permanent data loss

### How It Happens

Common scenarios that lead to these issues:

1. **Bulk tagging via Tag Editor** — A user applies MAP (map-migrated) tags using Tag Editor and accidentally modifies or removes MGN/DRS system tags in the same operation

2. **Tag cleanup gone wrong** — A user removes unfamiliar tags from instances, not realizing they are service-managed tags required by MGN/DRS

3. **Copy-paste tag errors** — A user copies tags from a replication server to another instance, causing MGN/DRS to claim that instance as managed infrastructure

4. **Re-applying tags incorrectly** — After removing service tags, a user re-applies only some of them (e.g. the managed tag and Name tag, but not the workflow association tags), triggering the service to terminate the instance as stale infrastructure

### What Happens Next

Once service tags are modified:

- MGN/DRS reconciliation process scans for instances matching its tag patterns
- Any instance with the service-managed tag and service Name tag that is NOT an active replicator will be TERMINATED
- This can happen in under 5 minutes — sometimes within seconds during active replication disruption events
- The service does NOT verify whether it originally created the instance

---

## The Solution

This automation deploys a serverless monitoring pipeline that:

1. **Detects** any modification to MGN/DRS system tags in real-time
2. **Evaluates** the severity (CRITICAL / HIGH / MEDIUM)
3. **Identifies** if non-managed instances are receiving service tags
4. **Alerts** your team via email within seconds
5. **Logs** all tag changes to DynamoDB for audit

### Architecture

You're running everything in **CloudShell** (same place you've been). Here we go:

---

## Step 1: Create the project folder

Run this in CloudShell:

```bash
mkdir -p ~/tag-monitor-project/{src,docs,tests}
cd ~/tag-monitor-project
echo "Current directory: $(pwd)"
```

---

## Step 2: Create the README.md (no customer data)

Copy and paste this entire block in CloudShell:

```bash
cat > ~/tag-monitor-project/README.md << 'READMEEOF'
# AWS Tag Change Monitor for MGN and DRS

**Automated monitoring and alerting for critical AWS Application Migration Service (MGN) and AWS Elastic Disaster Recovery (DRS) tag modifications.**

---

## The Problem

AWS MGN and DRS use **system-managed tags** to track and manage migration and disaster recovery infrastructure. When these tags are manually modified — whether intentionally or accidentally — serious consequences can follow:

- **Replication stalls** across source servers
- **Instances get terminated** — MGN/DRS reconciliation can terminate ANY instance with service tags that is not recognized as an active replicator
- **Non-managed instances are at risk** — Even manually-created EC2 instances (not part of any migration or DR workflow) can be terminated if someone applies service tags to them
- **Data loss** — If deleteOnTermination=true on root volumes, termination results in permanent data loss

### How It Happens

Common scenarios that lead to these issues:

1. **Bulk tagging via Tag Editor** — A user applies MAP (map-migrated) tags using Tag Editor and accidentally modifies or removes MGN/DRS system tags in the same operation

2. **Tag cleanup gone wrong** — A user removes unfamiliar tags from instances, not realizing they are service-managed tags required by MGN/DRS

3. **Copy-paste tag errors** — A user copies tags from a replication server to another instance, causing MGN/DRS to claim that instance as managed infrastructure

4. **Re-applying tags incorrectly** — After removing service tags, a user re-applies only some of them (e.g. the managed tag and Name tag, but not the workflow association tags), triggering the service to terminate the instance as stale infrastructure

### What Happens Next

Once service tags are modified:

- MGN/DRS reconciliation process scans for instances matching its tag patterns
- Any instance with the service-managed tag and service Name tag that is NOT an active replicator will be TERMINATED
- This can happen in under 5 minutes — sometimes within seconds during active replication disruption events
- The service does NOT verify whether it originally created the instance

---

## The Solution

This automation deploys a serverless monitoring pipeline that:

1. **Detects** any modification to MGN/DRS system tags in real-time
2. **Evaluates** the severity (CRITICAL / HIGH / MEDIUM)
3. **Identifies** if non-managed instances are receiving service tags
4. **Alerts** your team via email within seconds
5. **Logs** all tag changes to DynamoDB for audit

### Architecture

```
CloudTrail Events
    |-- EC2 CreateTags / DeleteTags
    |-- Tag Editor TagResources / UntagResources
    |-- DRS TagResource / UntagResource
    |-- MGN/DRS TerminateInstances
         |
         v
    EventBridge Rules (4 rules)
         |
         v
    Lambda Function
    |-- Evaluate tag criticality
    |-- Check for non-managed instances
    |-- Calculate severity
    |
    |---> SNS --> Email Alert
    |---> DynamoDB Audit Log
```

---

## What Tags Are Monitored

### MGN System Tags
| Tag Key | Type |
|---------|------|
| AWSApplicationMigrationServiceManaged | System-managed |
| AWSApplicationMigrationServiceSourceServerID | System-managed |
| AwsApplicationMigrationService-SourcePath | System-managed |
| AwsApplicationMigrationService-SourceServerID | System-managed |
| mgn.amazonaws.com-job | System-managed |
| mgn.amazonaws.com-source-server | System-managed |
| Name = AWS Application Migration Service Replication Server | Dangerous if applied to non-MGN instances |
| Name = AWS Application Migration Service Conversion Server | Dangerous if applied to non-MGN instances |

### DRS System Tags
| Tag Key | Type |
|---------|------|
| AWSElasticDisasterRecoveryManaged | System-managed |
| AWSElasticDisasterRecoverySourceServerID | System-managed |
| AwsElasticDisasterRecovery-SourcePath | System-managed |
| AwsElasticDisasterRecovery-SourceServerID | System-managed |
| elasticDisasterRecovery.amazonaws.com-job | System-managed |
| elasticDisasterRecovery.amazonaws.com-source-server | System-managed |
| Name = AWS Elastic Disaster Recovery Replication Server | Protected - do not alter |
| Name = AWS Elastic Disaster Recovery Conversion Server | Protected - do not alter |
| Name = AWS Elastic Disaster Recovery Recovery Instance | Protected - do not alter |

Per AWS Documentation: Do not alter the Name tag of resources created by AWS DRS (replication servers, EBS volumes, EBS snapshots, Conversion servers).

---

## Severity Levels

| Severity | Trigger | Example |
|----------|---------|---------|
| CRITICAL | Dangerous tag combination applied or removed | AWSApplicationMigrationServiceManaged=mgn.amazonaws.com applied to any instance |
| CRITICAL | DRS protected Name tag altered | Name tag removed from DRS replication server |
| CRITICAL | Service tags applied to non-managed instance | MGN tags applied to a manually-created EC2 instance |
| CRITICAL | MGN/DRS service terminates instances | Service-linked role calls TerminateInstances |
| CRITICAL | Bulk operation on 3+ critical tags or 3+ resources | Tag Editor bulk deletion of MGN tags |
| HIGH | Single critical tag removed | mgn.amazonaws.com-job deleted from one instance |
| HIGH | Single critical tag added | AWSElasticDisasterRecoveryManaged added |
| MEDIUM | Tag with protected prefix modified | Custom tag starting with mgn.amazonaws.com- |
| None | Normal tags such as Environment or map-migrated | No alert sent |

---

## Deployment

### Option 1: CloudFormation (Recommended)

Prerequisites:
- AWS account with CloudTrail enabled (enabled by default for management events)
- Email address to receive alerts

Steps:

1. Download cloudformation-template.yaml from this repository

2. Open AWS CloudFormation Console then Create Stack then Upload a template file

3. Fill in parameters:
   - Stack name: tag-change-monitor
   - NotificationEmail: your-team@company.com
   - EnableAuditLog: true
   - AuditRetentionDays: 90

4. Click Next then Next then check I acknowledge that AWS CloudFormation might create IAM resources with custom names then Submit

5. Wait for stack status: CREATE_COMPLETE (about 2 minutes)

6. Go to your email and click Confirm subscription in the AWS notification email

7. Done

### Option 2: AWS CLI

```
aws cloudformation deploy \
  --template-file cloudformation-template.yaml \
  --stack-name tag-change-monitor \
  --parameter-overrides \
    NotificationEmail=your-team@company.com \
    EnableAuditLog=true \
    AuditRetentionDays=90 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### What Gets Created

| Resource | Name | Purpose |
|----------|------|---------|
| SNS Topic | stack-name-alerts | Email notifications |
| DynamoDB Table | stack-name-audit | Audit log with TTL |
| Lambda Function | stack-name-function | Tag evaluation engine |
| IAM Role | stack-name-lambda-role | Lambda permissions |
| EventBridge Rule | stack-name-ec2-tags | EC2 CreateTags and DeleteTags |
| EventBridge Rule | stack-name-tagging-api | Tag Editor operations |
| EventBridge Rule | stack-name-termination | MGN and DRS terminations |
| EventBridge Rule | stack-name-drs-tags | DRS TagResource and UntagResource |

---

## Testing

After deployment verify the automation works.

### Quick Test (Instant - No CloudTrail Delay)

```
aws lambda invoke \
  --function-name tag-change-monitor-function \
  --payload file://tests/test-mgn-critical.json \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region us-east-1

cat response.json
```

Expected: {"statusCode": 200, "body": "Alert sent: CRITICAL"}
Check your email for the alert.

### Test Events Included

| File | What It Tests | Expected Result |
|------|--------------|-----------------|
| tests/test-mgn-critical.json | MGN dangerous tag combination applied | CRITICAL alert email |
| tests/test-drs-critical.json | DRS protected tags removed | CRITICAL alert email |
| tests/test-normal-no-alert.json | Normal tags (Environment, map-migrated) | No alert |

### Live Test (Real CloudTrail Flow)

Apply an MGN tag to any EC2 instance and wait 10-15 minutes:

```
aws ec2 create-tags \
  --resources i-YOUR_TEST_INSTANCE \
  --tags Key=AWSApplicationMigrationServiceManaged,Value=mgn.amazonaws.com

# Wait 10-15 minutes then check email

# Clean up
aws ec2 delete-tags \
  --resources i-YOUR_TEST_INSTANCE \
  --tags Key=AWSApplicationMigrationServiceManaged
```

---

## Cleanup

Delete the CloudFormation stack to remove all resources:

```
aws cloudformation delete-stack \
  --stack-name tag-change-monitor \
  --region us-east-1
```

---

## Sample Alert Email

```
Subject: [CRITICAL] [MGN] Tag Change - CreateTags on 3 resource(s)

TAG CHANGE ALERT - CRITICAL SEVERITY
Affected Service(s): MGN
============================================================

Event:       Tags ADDED to 3 resource(s)
Account:     123456789012
API Action:  CreateTags

WHO MADE THE CHANGE:
  Role: SomeRole / Session: user@company.com
  Source IP:   203.0.113.50

AFFECTED RESOURCES (3):
  - i-0abc1111111111111
  - i-0abc2222222222222
  - i-0abc3333333333333

DANGEROUS TAG COMBINATIONS:
  [MGN] AWSApplicationMigrationServiceManaged=mgn.amazonaws.com

================================================================
  WARNING: MGN SYSTEM TAG MODIFICATION

  - Replication can STALL across all source servers
  - MGN can TERMINATE instances in UNDER 5 MINUTES
  - Permanent DATA LOSS if deleteOnTermination=true
================================================================
```

---

## Recommended Preventive SCP

In addition to this monitoring (detective control), deploy this SCP as a preventive control:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyMGNDRSTagModification",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "tag:TagResources",
        "tag:UntagResources"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringLike": {
          "aws:TagKeys": [
            "AWSApplicationMigrationService*",
            "AwsApplicationMigrationService*",
            "mgn.amazonaws.com*",
            "AWSElasticDisasterRecovery*",
            "AwsElasticDisasterRecovery*",
            "elasticDisasterRecovery.amazonaws.com*",
            "aws:elasticDisasterRecovery*",
            "aws:drs*"
          ]
        },
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/aws-service-role/mgn.amazonaws.com/*",
            "arn:aws:iam::*:role/aws-service-role/drs.amazonaws.com/*"
          ]
        }
      }
    }
  ]
}
```

---

## Cost

| Resource | Cost |
|----------|------|
| Lambda | Free tier covers this (low-volume tag events) |
| EventBridge | Free for AWS service events |
| SNS | Free tier: 1000 email notifications per month |
| DynamoDB | Free tier: 25 GB storage |
| Total estimated | USD 0 per month for most accounts |

---

## References

- AWS MGN Replication Settings: https://docs.aws.amazon.com/mgn/latest/ug/replication-settings.html
- AWS DRS Managing Tags: https://docs.aws.amazon.com/drs/latest/userguide/managing-tags.html
- MGN Service-Linked Role: https://docs.aws.amazon.com/mgn/latest/ug/using-service-linked-roles.html
- DRS Service-Linked Role: https://docs.aws.amazon.com/drs/latest/userguide/using-service-linked-roles.html
- AWS Tag Editor: https://docs.aws.amazon.com/ARG/latest/userguide/tag-editor.html

---

## License
