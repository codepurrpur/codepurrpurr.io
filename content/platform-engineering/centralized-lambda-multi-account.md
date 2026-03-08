+++
title = "Centralized Control Plane for Multi-Account AWS Automation"
permalink = "https://codepurrpurr.io/platform/centralized-lambda-multi-account/"
date = "2025-04-22"
tags = ["aws", "lambda", "platform"]
categories = ["platform-engineering"]
images = ["/images/aws-iam.png"]
author = "Chris Zhang"
+++

## Problem

Large AWS environments typically use **multiple accounts** for isolation and governance.

Automation tasks often need to run across all accounts:

- compliance scans  
- inventory collection  
- tagging enforcement  
- remediation tasks  

A naive approach deploys automation into every account.  
This creates deployment duplication and operational drift.

## Architecture

Use a **central control plane** that executes automation across accounts.

```
Central Account
       ↓
Central Lambda
       ↓
STS AssumeRole
       ↓
Target Accounts
```

Automation runs once in the central account.  
Target accounts expose access through IAM roles.

## IAM Model

Each target account exposes a role trusted by the central account.

**Trust policy (target account)**

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<CENTRAL_ACCOUNT_ID>:role/LambdaExecutionRole"
  },
  "Action": "sts:AssumeRole"
}
```

Permissions on the role define what operations automation can perform.

The Lambda role must allow:

```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/CrossAccountAccessRole"
}
```

## Runtime

The Lambda assumes a role in each account and executes tasks.

```python
import boto3

def assume_role(account_id, role_name):

    sts = boto3.client("sts")

    response = sts.assume_role(
        RoleArn=f"arn:aws:iam::{account_id}:role/{role_name}",
        RoleSessionName="automation"
    )

    creds = response["Credentials"]

    return boto3.client(
        "s3",
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"]
    )
```

The same logic operates across any number of accounts.

## Benefits

- single automation runtime  
- no code duplication across accounts  
- centralized governance  
- scales to hundreds of accounts

## Limitations

This pattern is not suitable for workloads requiring:

- VPC-local execution  
- high-volume data processing  
- low-latency operations

## Conclusion

Multi-account AWS environments benefit from a **centralized automation control plane**.

Using STS AssumeRole allows a single Lambda runtime to operate across accounts while maintaining strong account isolation.