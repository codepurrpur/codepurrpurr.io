+++
title = "Identity-Centric Access Control with Vault and Okta"
permalink = "/auth/vault-okta-identity-architecture/"
date = "2024-05-31"
tags = ["vault", "okta", "identity"]
categories = ["authentication"]
images = ["/images/vault-feature.png"]
author = "Chris Zhang"
+++

## Problem

Cloud automation often depends on **long-lived API credentials**.

That creates predictable failure modes:

- credentials embedded in code  
- credentials leaked to source control  
- complex rotation processes  
- poor auditability  

Access should be derived from authenticated identity, not stored secrets.

## Architecture

Separate identity verification from credential issuance.

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

Okta verifies identity.

Vault turns that identity into temporary infrastructure credentials.

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

After login, the client requests cloud credentials.

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

Vault uses its internal cloud integration to issue short-lived credentials.

## Infrastructure Integration

Terraform retrieves credentials at runtime.

```
Terraform
    ↓
Request credentials from Vault
    ↓
Receive temporary IAM credentials
    ↓
Provision infrastructure
```

The automation environment stores no durable cloud keys.

## Key Properties

**Identity-derived access**

Access follows identity attributes, not copied secrets.

**Short credential lifetime**

Credentials expire quickly.

**Centralized policy**

Vault owns the policy boundary.

**Full audit trail**

Each credential can be traced to an authenticated identity.

## Conclusion

Identity-centric access replaces static infrastructure credentials with temporary access issued from verified identity.

Okta proves who the actor is.

Vault decides what that actor can receive.
