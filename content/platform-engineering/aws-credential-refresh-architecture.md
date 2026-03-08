+++
title = "Credential Lifecycle Management for Long-Running AWS Automation"
permalink = "https://codepurrpurr.io/platform/aws-credential-refresh/"
date = "2023-10-23"
tags = ["aws", "iam", "automation"]
categories = ["platform-engineering"]
images = ["/images/aws-iam.png"]
author = "Chris Zhang"
+++

## Problem

AWS automation commonly uses **temporary STS credentials**.

These credentials expire after a short period.

Long-running workloads such as:

- infrastructure provisioning  
- migration automation  
- large orchestration workflows  

may outlive the credential lifetime.

When credentials expire, API calls fail.

## Architecture

Automation should maintain a **refreshable credential session**.

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

Credentials are refreshed automatically before expiration.

## STS Credential Retrieval

The refresh mechanism requires a function that retrieves credentials from STS.

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

Botocore provides a credential wrapper that refreshes automatically.

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

Once configured, the session refreshes credentials transparently.

Clients inherit the refreshed credentials.

```python
s3 = session.client("s3")
ddb = session.resource("dynamodb")
```

## Benefits

- avoids long-lived IAM credentials  
- maintains continuous authentication  
- reduces credential management logic in automation code

## Conclusion

Long-running AWS automation should rely on **refreshable STS sessions** rather than static credentials.

The AWS SDK supports automatic credential renewal, allowing automation runtimes to maintain continuous access while preserving the security benefits of temporary credentials.