+++
title = "Credential Lifecycle Management for Long-Running AWS Automation"
permalink = "/platform/aws-credential-refresh/"
date = "2023-10-23"
tags = ["aws", "iam", "automation"]
categories = ["platform-engineering"]
images = ["/images/aws-iam.png"]
author = "Chris Zhang"
+++

## Problem

AWS automation commonly uses **temporary STS credentials**.

These credentials expire after a short period.

Long-running workloads may outlive them:

- infrastructure provisioning  
- migration automation  
- large orchestration workflows  

When credentials expire, API calls fail.

## Architecture

Use a refreshable credential session.

```
Automation Runtime
        ↓
Refreshable Credentials
        ↓
STS AssumeRole
        ↓
Temporary AWS Credentials
        ↓
AWS APIs
```

The runtime refreshes credentials before expiry.

## STS Credential Retrieval

The refresh function retrieves a new STS session.

```python
def get_sts_creds():

    session = boto3.Session()
    sts = session.client("sts")

    creds = sts.assume_role(
        RoleArn="arn:aws:iam::ACCOUNT:role/automation-role",
        RoleSessionName="automation",
        DurationSeconds=900
    )["Credentials"]

    return {
        "access_key": creds["AccessKeyId"],
        "secret_key": creds["SecretAccessKey"],
        "token": creds["SessionToken"],
        "expiry_time": creds["Expiration"].isoformat(),
    }
```

## Refreshable Session

Botocore can wrap the retrieval function and refresh automatically.

```python
credentials = botocore.credentials.RefreshableCredentials.create_from_metadata(
    metadata=get_sts_creds(),
    refresh_using=get_sts_creds,
    method="sts-assume-role",
)

session = get_session()
session._credentials = credentials
session.set_config_variable("region", "ap-southeast-2")
```

After configuration, clients inherit refreshed credentials.

```python
s3 = session.client("s3")
ddb = session.resource("dynamodb")
```

## Benefits

- no long-lived IAM credentials  
- uninterrupted automation runtime  
- less credential handling in application code

## Conclusion

Long-running AWS automation should not stretch temporary credentials beyond their lifetime.

Refreshable STS sessions preserve short-lived access without breaking long-running workflows.
