+++
title = "Identity-Centric Access Control with Vault and Okta"
permalink = "https://codepurrpurr.io/auth/vault-okta-identity-architecture/"
date = "2024-05-31"
tags = ["vault", "okta", "identity"]
categories = ["authentication"]
images = ["/images/vault-feature.png"]
author = "Chris Zhang"
+++

## Problem

Cloud automation typically relies on **long-lived API credentials**.

Common issues:

- credentials embedded in code  
- credentials leaked to source control  
- complex rotation processes  
- poor auditability  

A better model derives infrastructure access **directly from authenticated identity**.

## Architecture

Authentication and credential generation are separated.

```
User / Automation
        ↓
Identity Provider (Okta)
        ↓
Vault Authentication
        ↓
Dynamic Cloud Credentials
        ↓
AWS / Azure APIs
```

Identity verification happens in the identity provider.  
Vault converts identity into **temporary infrastructure credentials**.

## Authentication Flow

```
User
  ↓
Authenticate with Okta
  ↓
OIDC identity assertion
  ↓
Vault login
  ↓
Vault token issued
```

Vault attaches policies based on identity attributes such as group membership.

## Dynamic Credential Generation

After authentication, the client requests cloud credentials.

```
Client
   ↓
Vault token
   ↓
Vault policy check
   ↓
Vault generates credentials
   ↓
Temporary AWS / Azure credentials
```

Vault communicates with the cloud provider using privileged credentials stored internally.

The resulting credentials are short-lived.

## Infrastructure Integration

Automation tools such as Terraform retrieve credentials dynamically.

```
Terraform
    ↓
Request credentials from Vault
    ↓
Receive temporary IAM credentials
    ↓
Provision infrastructure
```

No long-lived credentials are stored in the automation environment.

## Key Properties

**Identity-derived access**

Authorization decisions are based on identity attributes rather than shared secrets.

**Short credential lifetime**

Credentials typically expire within minutes.

**Centralized policy**

Access rules are defined once in Vault.

**Full audit trail**

Credential issuance can be traced to an authenticated identity.

## Conclusion

An identity-centric access model replaces static infrastructure credentials with dynamically generated access tied to authenticated identity.

Integrating Vault with an identity provider such as Okta allows organizations to issue short-lived credentials while maintaining centralized policy and auditability.