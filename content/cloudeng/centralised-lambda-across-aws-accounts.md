+++ 
title = "Centralized Lambda execution across AWS accounts"
permalink = "https://codepurrpurr.io/cloudeng/centralised-lambda-across-aws-accounts/"
date = "2025-04-22" 
tags = ["aws", "lambda", "iam"] 
categories = ["lambda", "software", "dev", "iam"]
images = ["/images/aws-iam.png"]
author = "codepurrpurr.io"
+++

## The Challenge

In a multi-account AWS environment, it’s common to have centralized control requirements: one account to rule them all. When it comes to AWS Lambda functions, many teams struggle with duplicating code across multiple accounts or managing deployment pipelines for each. But what if you could execute logic *across accounts* using a *single Lambda*, deployed only once?

This post shows how to architect **a centralized Lambda function** in one AWS account that can perform actions across multiple other accounts **without deploying Lambda functions in the target accounts**.

## The Goal

- Run one Lambda in a **central account**
- No Lambda deployment in **child accounts**
- Let this Lambda assume roles in **other accounts** to perform actions (e.g. read/write S3, describe EC2 instances)

## Use Case Example

Let’s say your central Lambda wants to:

- List S3 buckets in each child account
- Collect resource tags
- Run compliance checks

Rather than deploying a Lambda in every account, we can:

1. Deploy one Lambda in the central account
2. Create IAM roles in the child accounts that it can assume
3. Use STS AssumeRole in code to dynamically access each child account

## IAM Setup

### 1. Child Account IAM Role (`CrossAccountAccessRole`)

**Trust policy** (in child account):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<CENTRAL_ACCOUNT_ID>:role/LambdaExecutionRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Permissions** (example):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets"],
      "Resource": "*"
    }
  ]
}
```

### 2. Central Account Lambda Execution Role

**Inline policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<CHILD_ACCOUNT_ID>:role/CrossAccountAccessRole"
    }
  ]
}
```

## Lambda

```python
import boto3

def assume_role(account_id, role_name):
    sts = boto3.client("sts")
    response = sts.assume_role(
        RoleArn=f"arn:aws:iam::{account_id}:role/{role_name}",
        RoleSessionName="crossAccountSession"
    )
    creds = response['Credentials']
    return boto3.client(
        's3',
        aws_access_key_id=creds['AccessKeyId'],
        aws_secret_access_key=creds['SecretAccessKey'],
        aws_session_token=creds['SessionToken']
    )

def lambda_handler(event, context):
    accounts = ['111122223333', '444455556666']  # child accounts
    for account in accounts:
        s3 = assume_role(account, 'CrossAccountAccessRole')
        buckets = s3.list_buckets()
        print(f"Buckets in {account}: {[b['Name'] for b in buckets['Buckets']]}")
```

## Benefits

- Only one Lambda function to manage
- Full visibility and control from a central location
- Scales easily to 10s or 100s of accounts

## Caveats

- Cannot run VPC-local tasks in child accounts
- Needs careful IAM setup to avoid overly permissive roles
- Slight increase in latency

## Conclusion

This centralized Lambda pattern gives you a clean and scalable architecture for managing cross-account tasks in AWS, especially in organizations adopting Control Tower, Organizations, or Landing Zones. It reduces duplication, simplifies deployment, and enables better governance — without compromising flexibility.