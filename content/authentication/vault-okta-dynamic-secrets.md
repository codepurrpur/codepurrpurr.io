+++ 
title = "Vault + Okta: Automating Dynamic Secrets for Identity-Centric Access Control"
permalink = "https://codepurrpurr.io/auth/vault-okta-dynamic-secrets/"
date = "2024-05-31" 
tags = ["vault", "iac", "secrets", "authentication"] 
categories = ["automation", "software", "dev"]
images = ["/images/vault-feature.png"]
author = "codepurrpurr.io"
+++

# Intro

I have been fooling around with HashiCorp Vault ( <https://www.vaultproject.io> ) for a bit now, and thought I would write about it here.

## What is it?

Vault is an identity-based secrets and encryption management system that validates and authorizes clients (users, machines, apps) before granting them access to secrets or sensitive data. The way it handles secret management, given my use case, is truly impressive.

## The Challenge

Here's the backstory: I accidentally committed my automation code to Github, which included long-lived AWS API keys. Someone promptly took advantage of this and started provisioning hundreds of EC2 instances in my account. Luckily, I caught this early and saved myself from a substantial bill shock.

Now, I work with large enterprise customers, and this incident raised a significant concern for me.

Here's the challenge:

When you have a team of developers interfacing with multiple Clouds, like AWS and Azure, how do you secure their API access without slowing them down? Requiring developers to constantly renew their API keys throughout the day can be a major bottleneck. If this process also demands MFA authentication every time, it becomes even more cumbersome. And if developers have to obtain API keys from different places, depending on the Cloud they're working on, it's a hassle for both them as consumers and the security team as providers of the infrastructure.

In my view, a security solution that burdens people with unnecessary steps is, simply put, a bad security solution.

## Solution

Ideally, developers should only need to authenticate through a central Identity Provider (IDP), like Okta, once, and that should suffice for the entire day.

It shouldn't matter whether they're working on AWS, Azure, or both; they shouldn't have to deal with the hassle of obtaining different API keys from various systems. The process of acquiring API keys should be seamless and automated in the background, requiring no intervention from the developers.

This automated process should generate short-lived keys with a brief expiration time, say 5 minutes. By using short-lived keys, the impact of a compromised key is significantly reduced. Once these keys expire, the system should automatically obtain new ones, again without requiring any action from the developers.

## Introducing the A team: Vault, Okta and Terraform

If the above solution seems like an aspiration, I have good news — it's already a reality.

Now, let's explore the high-level schematics and understand how all the components fit together. We can dive into the specific details at the code level later.

To grasp the concept better, I'll use the term **consumer** to represent the developers, as they are the ones consuming the security service.

I'll also refer to the term **producer** to represent the security team responsible for providing the security service.

The producer begins by provisioning and configuring Vault to integrate with Okta as an external Identity Provider (IDP) for authentication. For example, when a consumer needs to log into Vault, they do so through Okta.

Once the consumer completes the authentication process with Okta, using OpenID Connect, Okta sends back the consumer's group membership information as an OIDC claim to Vault.

Vault then generates a token, associating it with a policy linked to the consumer's group, and returns the token to the consumer.

Once the consumer receives the token from Vault, they can request an AWS key from Vault, enabling them to perform tasks in AWS by presenting the token.

Vault examines the token, verifies the associated policy, and grants the consumer's request. It dynamically generates a short-lived IAM user with the appropriate permissions and returns the API key to the consumer.

Now, you might be curious about how this all works.

The mechanism involves a long-lived AWS API key with the necessary IAM permissions, securely stored in Vault by the producer in advance.

Vault is configured with a role that contains the IAM role statement. When the consumer requests an AWS API key from Vault, Terraform code triggers a request to read the Vault role. Vault then communicates with AWS, dynamically provisions a short-lived IAM user assigned with the IAM role statement, and returns the resulting API key to the consumer.

Once the consumer obtains the API key, the remaining Terraform code initiates the necessary actions to build infrastructure or perform other tasks in AWS.

These steps outline the process for AWS, but the mechanism is essentially the same for Azure.

With this solution, the consumer leverages dynamic short-lived keys to enhance security. The authentication process has been abstracted, relieving developers from the need to log into different Clouds to obtain corresponding API keys.

## Seeing is believing

Let's take a look at a quick demo.

### Producer

Run Vault in dev mode and export variables, so nothing is saved to disk

```sh
% vault server -dev -dev-root-token-id=root
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201

...

You may need to set the following environment variables:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: hQT+p5btHNUO4VmvbPD4VEZ1aW59brsOB0ihM6JzRhY=
Root Token: root
```

```sh
# Export env vars
% export TF_VAR_aws_access_key=xxx
% export TF_VAR_aws_secret_key=xxx
% export VAULT_ADDR=http://127.0.0.1:8200 
% export VAULT_TOKEN=root
```

Now run `terraform apply` to build out Okta and Vault configuration.

```tcl
% cd producer-workspace/
% terraform apply -auto-approve

...

Outputs:

aws_backend = "dynamic-aws-creds-producer"
aws_role = "dynamic-aws-creds-producer-role"
```

That's all on the producer side.

### Consumer

Since the demo is on the same machine for both producer and consumer, we must unset VAULT_TOKEN as it is only available to producer right now.

```sh
% cd ../consumer-workspace/aws
% unset VAULT_TOKEN
% vault login -method=oidc -path=okta_oidc role=okta_dev
```

A browser immediately comes up with Okta login.

![vault-login](/images/okta-auth-login.png)

After authentication, Vault acknowledges the establishment of my signed-in session.

![vault-login](/images/vault-login.png)

If you switch back to the cli, you will see the following. It shows that we just got a Vault token with a 12-hour validity.

```sh
Waiting for OIDC authentication to complete...
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.CAESIK_mxiC8ICy4RwapznKAJEIVPLLVyzctig6Yu-jwy-EEGh4KHGh2cy5xalAzTGhrWm5kSG9uemQzZDN6cm5ER0w
token_accessor       K2aaSUI2bQs672gv0bhXmOqM
token_duration       12h
token_renewable      true
token_policies       ["default" "developer"]
identity_policies    []
policies             ["default" "developer"]
token_meta_role      okta_dev
```

Now run `terraform apply`, the code picks up the Vault token automatically and uses it to get a dynamic API key from Vault

```sh
% terraform apply
...

aws_instance.ubuntu[0]: Creating...
aws_instance.ubuntu[0]: Still creating... [10s elapsed]
aws_instance.ubuntu[0]: Still creating... [20s elapsed]
aws_instance.ubuntu[0]: Creation complete after 22s [id=i-0865cffb61b74ede6]

Apply complete! Resources: 17 added, 0 changed, 0 destroyed.
```

Let's take a look at the AWS IAM console. A new IAM user was dynamically generated and was used for the terraform build.

![dynamic-iam](/images/dynamic-iam.png)

What if you run `terraform apply` again? What would you see on the IAM console? I am leaving this as an exercise for the user to find out :)

## Code Snippet Explained

The code for the demo is here  ( <https://gitlab.com/abgmbh/vault-okta-auth-aws-backend> )

### Producer code

For the producer (security team), I have included Terraform code to configure Vault and Okta.

As we are using Okta as the IDP, `okta.tf` configures the Okta side accordingly.

The code creates a user in Okta and then puts that user into the `vault-dev` group. The `okta_app_oauth` resource removes the group association every time `terraform apply` is ran.

The following code must be used to ignore the dis-association of the group at `terraform apply`

```bash
lifecycle {
    ignore_changes = [groups]
  }

```

In `aws_backend.tf`, we tell Vault to apply the IAM role definition to all dynamically generated IAM users

```bash
resource "vault_aws_secret_backend_role" "producer" {
  backend         = vault_aws_secret_backend.aws.path
  name            = "${var.name}-role"
  credential_type = "iam_user"

  policy_document = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:*", "ec2:*"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

The following code shows the Vault policy. It took me some time to find out the exact policy rules to use.

```bash
data "vault_policy_document" "dev_policy_content" {
  rule {
    path         = "dynamic-aws-creds-producer/creds/dynamic-aws-creds-producer-role"
    capabilities = ["read"]
    description  = ""
  }

  rule {
    path         = "auth/token/lookup-self"
    capabilities = ["read"]
    description  = ""
  }

  rule {
    path         = "auth/token/create"
    capabilities = ["update"]
    description  = ""
  }

}
```

As a general troubleshooting method, you can run the following commands to enable audit on Vault and see exactly what paths and operations are required.

On the Vault server side, run with root token, and then have consumer to run terraform requesting a token.

```bash
vault audit enable file file_path=audit.log
tail -f  audit.log | jq '.request | {path, operation}'

{
  "path": "auth/token/lookup-self",
  "operation": "read"
}
{
  "path": "auth/token/create",
  "operation": "update"
}
{
  "path": "dynamic-aws-creds-producer/creds/dynamic-aws-creds-producer-role",
  "operation": "read"
}
```

### Consumer code

The following code shows consumer logs into Vault via Okta (OIDC) and then `access_key` and `secret_key` is then obtained via Vault and provided to the AWS provider.

```tcl
data "vault_aws_access_credentials" "creds" {
  backend = var.aws_backend
  role    = var.aws_role
}

provider "aws" {
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
  region     = var.region
}

provider "vault" {
}
```

## Conclusion

Vault is amazing! It not only conquers complex security challenges but also provides a superb user experience. Just log in in the morning, and you're set for the day. Seriously, Vault is the epitome of elegance and craftsmanship.
