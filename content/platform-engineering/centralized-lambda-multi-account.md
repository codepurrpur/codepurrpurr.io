+++
title = "Centralized Control Plane for Multi-Account AWS Automation"
permalink = "/platform/centralized-lambda-multi-account/"
date = "2025-04-22"
tags = ["aws", "lambda", "platform"]
categories = ["platform-engineering"]
images = ["/images/aws-iam.png"]
author = "Chris Zhang"
+++

## Problem

Large AWS environments use multiple accounts for isolation and governance.

Automation still needs to cross account boundaries:

- compliance scans  
- inventory collection  
- tagging enforcement  
- remediation tasks  

Deploying the same automation into every account creates duplication and drift.

## Architecture

Use a central control plane.

```
Central Account
       ↓
Central Lambda
       ↓
STS AssumeRole
       ↓
Target Accounts
```

Automation runs once.

Target accounts expose access through IAM roles.

## IAM Model

Each target account exposes a role that trusts the central account.

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

The target role defines what automation can do inside that account.

The Lambda execution role must be allowed to assume it:

```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/CrossAccountAccessRole"
}
```

## Runtime

At runtime, Lambda assumes a target role and creates a scoped client.

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

The same path works for any account that exposes the role.

## Benefits

- single automation runtime  
- no code duplication across accounts  
- centralized governance  
- scales to hundreds of accounts

## Limitations

Avoid this pattern when work requires:

- VPC-local execution  
- high-volume data processing  
- low-latency operations

## Conclusion

Multi-account automation needs a control plane, not repeated deployments.

STS AssumeRole lets one Lambda runtime operate across accounts while preserving account isolation.
