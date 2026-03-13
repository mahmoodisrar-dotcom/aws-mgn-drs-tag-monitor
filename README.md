
```
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

> **Per AWS Documentation:** *"Do not alter the Name tag of resources created by AWS DRS (replication servers, EBS volumes, EBS snapshots, Conversion servers)."*

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

**Prerequisites:**
- AWS account with CloudTrail enabled (enabled by default for management events)
- Email address to receive alerts

---

### Option 1: AWS Console (Recommended for first-time users)

#### Step 1: Download the template

1. In this repository, click on [cloudformation-template.yaml](cloudformation-template.yaml)
2. Click the **Raw** button (top-right of the file view)
3. Right-click the page and select **Save As** → save as `cloudformation-template.yaml`

#### Step 2: Open CloudFormation Console

1. Sign in to your AWS account
2. Go to the [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
3. Make sure you are in the correct **Region** (e.g. us-east-1) using the region selector in the top-right

#### Step 3: Create the Stack

1. Click **Create stack** → **With new resources (standard)**
2. Under **Specify template**, select **Upload a template file**
3. Click **Choose file** and select the `cloudformation-template.yaml` you downloaded
4. Click **Next**

#### Step 4: Configure Stack Parameters

1. **Stack name:** Enter `tag-change-monitor`
2. **NotificationEmail:** Enter the email address where you want to receive alerts (e.g. `your-team@company.com`)
3. **EnableAuditLog:** Leave as `true` (recommended)
4. **AuditRetentionDays:** Leave as `90` or adjust as needed
5. Click **Next**

#### Step 5: Configure Stack Options

1. Leave all default settings (Tags, Permissions, Advanced options)
2. Click **Next**

#### Step 6: Review and Create

1. Scroll to the bottom
2. Check the box: **"I acknowledge that AWS CloudFormation might create IAM resources with custom names"**
3. Click **Submit**

#### Step 7: Wait for Completion

1. The stack status will show **CREATE_IN_PROGRESS**
2. Wait approximately 2 minutes
3. Click the **Refresh** button until you see **CREATE_COMPLETE**
4. If you see **ROLLBACK**, click on the **Events** tab to see what went wrong

#### Step 8: Confirm Email Subscription

1. Check the email inbox for the address you entered in Step 4
2. You will receive an email from **AWS Notifications** with subject "AWS Notification - Subscription Confirmation"
3. Click the **Confirm subscription** link in the email
4. You should see a page saying "Subscription confirmed!"

#### Step 9: Verify Resources (Optional)

1. In the CloudFormation Console, click on your stack name `tag-change-monitor`
2. Click the **Resources** tab
3. You should see all 9 resources with status **CREATE_COMPLETE**:
   - AlertTopic (SNS)
   - AuditTable (DynamoDB)
   - MonitorFunction (Lambda)
   - LambdaRole (IAM)
   - EC2TagRule (EventBridge)
   - TaggingAPIRule (EventBridge)
   - TerminationRule (EventBridge)
   - DRSTagRule (EventBridge)
   - Plus Lambda permissions

**Deployment is complete!** The automation is now monitoring your account.

---

### Option 2: AWS CLI (Deploy directly from this repo)

Open **AWS CloudShell** (click the terminal icon in the AWS Console top bar) or any terminal with AWS CLI configured:

```bash
# Step 1: Download the CloudFormation template from this repo
curl -O https://raw.githubusercontent.com/mahmoodisrar-dotcom/aws-mgn-drs-tag-monitor/main/cloudformation-template.yaml

# Step 2: Deploy the stack (replace email with yours)
aws cloudformation deploy \
  --template-file cloudformation-template.yaml \
  --stack-name tag-change-monitor \
  --parameter-overrides \
    NotificationEmail=your-team@company.com \
    EnableAuditLog=true \
    AuditRetentionDays=90 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Step 3: Go to your email and click "Confirm subscription"

# Step 4: Verify deployment
aws cloudformation describe-stacks \
  --stack-name tag-change-monitor \
  --query 'Stacks[0].StackStatus' \
  --output text \
  --region us-east-1
# Expected output: CREATE_COMPLETE

# Step 5: View all created resources
aws cloudformation describe-stack-resources \
  --stack-name tag-change-monitor \
  --query 'StackResources[].{Type:ResourceType,Name:LogicalResourceId,Status:ResourceStatus}' \
  --output table \
  --region us-east-1
```

---

### What Gets Created

| Resource | Name | Purpose |
|----------|------|---------|
| SNS Topic | tag-change-monitor-alerts | Email notifications |
| DynamoDB Table | tag-change-monitor-audit | Audit log with automatic TTL cleanup |
| Lambda Function | tag-change-monitor-function | Tag evaluation engine |
| IAM Role | tag-change-monitor-lambda-role | Least-privilege Lambda permissions |
| EventBridge Rule | tag-change-monitor-ec2-tags | EC2 CreateTags and DeleteTags |
| EventBridge Rule | tag-change-monitor-tagging-api | Tag Editor operations |
| EventBridge Rule | tag-change-monitor-termination | MGN and DRS service terminations |
| EventBridge Rule | tag-change-monitor-drs-tags | DRS TagResource and UntagResource |

---

## Testing

After deployment, verify the automation works.

### Quick Test via AWS Console

1. Go to the [AWS Lambda Console](https://console.aws.amazon.com/lambda/)
2. Click on the function **tag-change-monitor-function**
3. Click the **Test** tab
4. Click **Create new event**
5. **Event name:** `test-mgn-critical`
6. Replace the default JSON with the content from [tests/test-mgn-critical.json](tests/test-mgn-critical.json) in this repo
7. Click **Save** then click **Test**
8. You should see:
   - **Execution result: succeeded**
   - Response: `{"statusCode": 200, "body": "Alert sent: CRITICAL"}`
9. Check your email for the CRITICAL alert!

### Quick Test via AWS CLI

Download the test events from this repo and invoke the Lambda function directly:

```bash
# Download all test events
curl -O https://raw.githubusercontent.com/mahmoodisrar-dotcom/aws-mgn-drs-tag-monitor/main/tests/test-mgn-critical.json
curl -O https://raw.githubusercontent.com/mahmoodisrar-dotcom/aws-mgn-drs-tag-monitor/main/tests/test-drs-critical.json
curl -O https://raw.githubusercontent.com/mahmoodisrar-dotcom/aws-mgn-drs-tag-monitor/main/tests/test-normal-no-alert.json

# Test 1: MGN Critical — should send CRITICAL email
aws lambda invoke \
  --function-name tag-change-monitor-function \
  --payload file://test-mgn-critical.json \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region us-east-1
cat response.json
# Expected: {"statusCode": 200, "body": "Alert sent: CRITICAL"}
# Check your email!

# Test 2: DRS Critical — should send CRITICAL email
aws lambda invoke \
  --function-name tag-change-monitor-function \
  --payload file://test-drs-critical.json \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region us-east-1
cat response.json
# Expected: {"statusCode": 200, "body": "Alert sent: CRITICAL"}

# Test 3: Normal tags — should NOT send email
aws lambda invoke \
  --function-name tag-change-monitor-function \
  --payload file://test-normal-no-alert.json \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region us-east-1
cat response.json
# Expected: {"statusCode": 200, "body": "No critical tags affected"}
```

### Test Events Included

| File | What It Tests | Expected Result |
|------|--------------|-----------------|
| tests/test-mgn-critical.json | MGN dangerous tag combination applied | CRITICAL alert email |
| tests/test-drs-critical.json | DRS protected tags removed | CRITICAL alert email |
| tests/test-normal-no-alert.json | Normal tags (Environment, map-migrated) | No alert |

### Live Test (Real CloudTrail Flow)

#### Via Console:
1. Go to [EC2 Console](https://console.aws.amazon.com/ec2/) → select any test instance
2. Click **Tags** tab → **Manage tags**
3. Add tag: Key = `AWSApplicationMigrationServiceManaged`, Value = `mgn.amazonaws.com`
4. Click **Save**
5. Wait 10-15 minutes for CloudTrail to deliver the event
6. Check your email for the CRITICAL alert
7. **Remove the tag immediately** after testing

#### Via CLI:
```bash
# Apply test tag (triggers CRITICAL alert after 10-15 min)
aws ec2 create-tags \
  --resources i-YOUR_TEST_INSTANCE \
  --tags Key=AWSApplicationMigrationServiceManaged,Value=mgn.amazonaws.com

# Wait 10-15 minutes then check email

# Clean up the test tag immediately
aws ec2 delete-tags \
  --resources i-YOUR_TEST_INSTANCE \
  --tags Key=AWSApplicationMigrationServiceManaged
```

> **Warning:** Do not leave MGN/DRS system tags on non-managed instances. Remove test tags immediately after testing.

---

## Cleanup

### Via Console:
1. Go to the [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
2. Select the stack **tag-change-monitor**
3. Click **Delete**
4. Click **Delete** to confirm
5. Wait for the stack to be deleted (about 1 minute)

### Via CLI:
```bash
aws cloudformation delete-stack \
  --stack-name tag-change-monitor \
  --region us-east-1

# Verify deletion
aws cloudformation describe-stacks \
  --stack-name tag-change-monitor \
  --region us-east-1 2>&1
# Expected: "Stack with id tag-change-monitor does not exist"
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

## Recommended: Preventive SCP

In addition to this monitoring (detective control), deploy this Service Control Policy as a **preventive control** to block unauthorized tag modifications at the organization level:

```json
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

### How to Apply the SCP:

#### Via Console:
1. Go to [AWS Organizations Console](https://console.aws.amazon.com/organizations/)
2. Click **Policies** → **Service control policies**
3. Click **Create policy**
4. Name: `DenyMGNDRSTagModification`
5. Paste the JSON above
6. Click **Create policy**
7. Attach to the desired OUs or accounts

#### Via CLI:
```bash
aws organizations create-policy \
  --name DenyMGNDRSTagModification \
  --description "Prevents manual modification of MGN and DRS system tags" \
  --type SERVICE_CONTROL_POLICY \
  --content file://scp-policy.json
```

---

## Cost

| Resource | Cost |
|----------|------|
| Lambda | Free tier covers this (low-volume tag events) |
| EventBridge | Free for AWS service events |
| SNS | Free tier: 1,000 email notifications per month |
| DynamoDB | Free tier: 25 GB storage |
| **Total estimated** | **$0/month for most accounts** |

---

## References

- [AWS MGN - Replication Settings](https://docs.aws.amazon.com/mgn/latest/ug/replication-settings.html)
- [AWS DRS - Managing Tags](https://docs.aws.amazon.com/drs/latest/userguide/managing-tags.html)
- [MGN Service-Linked Role](https://docs.aws.amazon.com/mgn/latest/ug/using-service-linked-roles.html)
- [DRS Service-Linked Role](https://docs.aws.amazon.com/drs/latest/userguide/using-service-linked-roles.html)
- [AWS Tag Editor](https://docs.aws.amazon.com/ARG/latest/userguide/tag-editor.html)
- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [AWS CloudShell](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html)

---

## License

MIT License — Free to use, modify, and distribute.
```

