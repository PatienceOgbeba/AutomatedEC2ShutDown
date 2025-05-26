#  Budget-Based EC2 Shutdown Automation

This project implements an automated cost control system using **AWS Budgets** and **AWS CloudFormation**. It monitors cloud spend and triggers the shutdown of EC2 instances when a budget threshold is exceeded — helping avoid unexpected AWS bills.

---

##  Use Case
A financial threshold is defined using AWS Budgets. When that budget is exceeded, an SNS notification is published and a Lambda function automatically stops non-critical EC2 instances.

---

##  Architecture Components

- **AWS Budgets**: Defines a cost threshold and triggers alerts/actions when exceeded.
- **Amazon SNS**: Delivers budget alert notifications.
- **AWS Lambda**: Subscribed to SNS topic, responsible for stopping EC2 instances.
- **IAM Roles**:
  - Budget Action Role (assumed by AWS Budgets to publish to SNS)
  - Lambda Execution Role (with permissions to stop EC2 instances)
- **EC2 Instances**: The targets for shutdown, identified by tag or instance ID.
- **CloudFormation**: Automates deployment of all components.

---

##  Features

-  **Cost Monitoring**: Monthly budget of $50 with alerts at 100% usage
-  **Automated Shutdown**: Lambda stops EC2 instances upon threshold breach
-  **CloudFormation Stack**: Fully reproducible infrastructure-as-code template
-  **Real-Time Alerts**: Sends alerts via email/SNS for both warnings and actions

---

##  Step-by-Step Deployment Guide

###  Step 1: Prepare IAM Roles

Create an IAM role that AWS Budgets can assume:
- Trusted entity: `budgets.amazonaws.com`
- Permissions: `sns:Publish` to the target SNS topic

Create a Lambda execution role with:
- `ec2:StopInstances`
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

> These roles can be included in your CloudFormation template.

---

###  Step 2: Define SNS Topic and Lambda Function

1. Create an SNS topic called `EC2BudgetAlerts`
2. Create a Lambda function subscribed to that topic:
   - Function logic checks running EC2 instances
   - Filters based on tag (e.g., skips those with Tag `critical=true`)
   - Stops non-critical instances

Example (Python):
```python
import boto3

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    instances = ec2.describe_instances(Filters=[
        {'Name': 'tag:critical', 'Values': ['false']}
    ])
    instance_ids = [i['InstanceId'] for r in instances['Reservations'] for i in r['Instances']]
    if instance_ids:
        ec2.stop_instances(InstanceIds=instance_ids)
```

---

###  Step 3: Create Budget and Budget Action

Use AWS Budgets to:
- Set a monthly budget (e.g., $50)
- Create an alert at 100% of the budget
- Configure it to **publish to the SNS topic** using the IAM role created earlier

---

###  Step 4: CloudFormation Stack Deployment

All resources can be defined in a CloudFormation template:
- Budget definition
- SNS topic
- Lambda function
- IAM roles
- Permissions and triggers

Deploy with:
```bash
aws cloudformation create-stack \
  --stack-name ec2-budget-shutdown \
  --template-body file://ec2-budget-shutdown.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

---

##  Logic Summary

1. EC2 instances are running normally.
2. AWS Budgets monitors the monthly spend.
3. On reaching $50, a notification is sent to SNS.
4. SNS triggers Lambda.
5. Lambda shuts down all non-critical EC2 instances.

---

##  Benefits

-  **Cost Control**: Automatically prevent budget overruns
-  **Automation**: No manual shutdowns required
-  **Monitoring**: Real-time alerting through Budgets and SNS
-  **Reusability**: Easily replicate via CloudFormation
-  **Compliance**: Enforce cost policies at scale

---

## Notes
- EC2 instances must be tagged appropriately to exclude critical workloads
- CloudFormation stack does **not** launch EC2 instances — they must be started separately
- Lambda and SNS regions must match for integration

---
 
**License:** MIT
