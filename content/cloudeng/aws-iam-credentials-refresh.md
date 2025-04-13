+++ 
title = "Automated refresh of AWS IAM credentials"
permalink = "https://codepurrpurr.io/automation/aws-iam-credentials-refresh/"
date = "2023-10-23" 
tags = ["aws", "iam", "credentials", "authentication"] 
categories = ["automation", "software", "dev", "iam"]
images = ["/images/aws-iam.png"]
author = "codepurrpurr.io"
+++

## The Challenge

I just wrapped up some automation code within AWS using Boto3, the code runs for quite a while (I/O bound, mostly waiting on AWS to finish), and as a result the temperory IAM credentials used by the automation code needs to be constantly refreshed.

## Solution

It turns out botocore already provides a way to facilite automated credentials refresh, but the feature is extremely poorly documented.

In essense, you need to write a method that returns the temporary IAM credentials, as shown below.

```python
def get_sts_creds():
    role_arn = "arn:aws:iam::your_account:role/your_role" # This is the IAM role the automation code assumes in order to carry out automated tasks

    session = boto3.Session()
    sts = session.client("sts")

    credentials = sts.assume_role(
        RoleArn=role_arn, RoleSessionName="test", DurationSeconds=900
    ).get("Credentials")
    return {
        "access_key": credentials.get("AccessKeyId"),
        "secret_key": credentials.get("SecretAccessKey"),
        "token": credentials.get("SessionToken"),
        "expiry_time": credentials.get("Expiration").isoformat(),
    }
```

You would then create a self-refreshable credentials object as below, and assign it to a session's `_credentials` attribute.

This would make the session always having a valid set of credentials.

```python
credentials = botocore.credentials.RefreshableCredentials.create_from_metadata(
    metadata=get_sts_creds(),
    refresh_using=get_sts_creds,
    method="sts-assume-role",
)

credentials.refresh_needed(refresh_in=910)
session = get_session()
session._credentials = credentials
session.set_config_variable("region", "ap-southeast-2")
```

The `refresh_needed()` method is part of the botocore library (<https://github.com/boto/botocore/blob/develop/botocore/credentials.py#L443>). You can check out the comments to see how to use it.

Lastly, let's do a quick test.

```python
while True:
    log_message = f"Access Key: {credentials.access_key}\nSecret Key: {credentials.secret_key}\nToken: {credentials.token}\n"
    print(log_message)

    time.sleep(20)
```

The output is shown below. For security, I masked a portion of the credentials.

```bash
Access Key: XXXXXXXXXXXXXXXXDBCS
Secret Key: XXXXXXXXXXXXXXXXnlD8iYjO/U8vO1rToHdLkoLF
Token: XXXXXXXXXXXXXXXX////XXXXXXXXXXXXXXXX8JOkuxTdGF4vwnKsS0VeBzIo/1NB7FJqJpJiGWc88v0Q7U2KP05+FpqqorBxraV8q8+TKT181zquOx77+ECO3VVIkUEcEVgLBxdMu5cy+BGWkjhxK99ThMI89cabH+T2YVQ3EL+Q/3xXgH4YtwfXC01zLAgSHRx3XFvdIvwJUsBRYIJxz6PyJQPFy8yhuV4xmK34Hyd0yrEQMR3qPth7KuEYyjurMipBjIth18Srup650iL705IqaSVVToyDG90jhFEaoUSOCtZz35sCIr2BQNWVViS8fVs

Access Key: XXXXXXXXXXXXXXXX2PUFA
Secret Key: XXXXXXXXXXXXXXXXWqRipr1CYKkBdAeTqkYKVrFm
Token: XXXXXXXXXXXXXXXX/////XXXXXXXXXXXXXXXX/GZdKYUHZ+mSf9HBYByQjy8F0PNWRngHmQJ6gBKMRJVxGonUHWKlmwgqot8BaAc1jiFRttYwfUEte0o50FZTjxjto3MdxHKCpvprGgh8wlc8qav//XU/ND7SC0Uey/tqF3m4GOV9Q7iA/JWrResvCtTV1VMMlIW7mRCb88O691sTHjnsmhHw2PWVX3J8ZqNeal/fPTuURkrnY3eWHbr52iimrcipBjItVkyMEeR/O7/N3eyM6tPGNEVGZoC9vvJyal5AvsD19AfsXD6n1fHGj9biP3VJ

Access Key: XXXXXXXXXXXXXXXXB2BNP
Secret Key: XXXXXXXXXXXXXXXXkf47kCug+w7FyEfZHQhNNzI4
Token: XXXXXXXXXXXXXXXX////XXXXXXXXXXXXXXXXEERJqL8g+nrySgiDKUebzVTTAu8sFbOQqz95aDpeONkmoBhI0WdqJ8TKq5ol7REKzhDAZ4W+vuoICo+h0B0A9tG5iuRhz+9ALfTCRkOLTbIoX3pp0hSH1DdImdxnZJdv93J/7QpDEio2exZyTVIErv/GCfurieida+SdPmURtVoD10fds9FhZeKMyJWFOs3gnAQC4/+Zc8MMC9jDphpFTTCistMipBjItwZFZVk/BtL+LFGsqcXqpe+7G+6bHij1TTCniUwNPOxE0UMvkR/yObKUeKdwj
```

As you can see, the credentials refresh on their own, and it is all taken care of by botocore itself.

Once you have a never expiring session, you can then go and do something with it.

```python
s3 = session.client('s3')
ddb = session.resource('dynamodb')
...
```

## Conclusion

Boto3 has already built the refreshable credentials function into the library, albeit documentation needs some work.

It's really easy to use and probably not a bad idea to have it included in all automation code.
