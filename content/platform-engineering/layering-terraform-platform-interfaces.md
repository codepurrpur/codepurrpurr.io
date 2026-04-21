
+++
title = "Layering Terraform for Platform Interfaces"
permalink = "/platform-engineering/layering-terraform-platform-interfaces/"
date = "2026-03-08"
tags = ["terraform", "platform-engineering", "aws"]
categories = ["platform-engineering"]
author = "Chris Zhang"
+++

## Problem

Terraform is often treated as a direct infrastructure authoring tool.

Even with modules, consumers still think in VPCs, subnets, AMIs, security groups, listeners, and target groups.

That is not a platform interface.

A platform interface should let consumers describe service intent.

The platform should resolve topology, implementation, and provider behavior internally.

The goal is not to hide every infrastructure decision. Network sizing can still be a valid service-level parameter.

The goal is to hide the mechanics required to realize that decision.

---

## Architecture

The platform is structured as four layers.

```text
Layer 1 — Typed Platform Contract
Layer 2 — Service Composition
Layer 3 — Platform Implementation Modules
Layer 4 — Provider / Community Modules and AWS Resources
```

The flow:

```text
Service Intent
      ↓
Platform Composition
      ↓
Platform Modules
      ↓
Provider / Community Implementation
      ↓
AWS Resources
```

Each layer owns one responsibility.

---

## Layer 1 — Typed Platform Contract

Layer 1 is the consumer contract.

Consumers describe a service through a typed object.

The contract exposes only the decisions they are expected to make:

* whether the service is enabled
* network sizing
* compute layout
* ingress behavior
* security intent
* secrets

Example:

```hcl
services = {
  s1 = {
    enabled                    = true
    ipam_pool                  = "dev"
    vpc_netmask_length         = 22
    vpc_private_netmask_length = 24

    alb = {
      enabled           = true
      protocol          = "HTTPS"
      port              = 443
      domain            = "codepurrpurr.io"
      health_check_path = "/health"
    }

    security_groups = {
      web-sg = {
        source   = "mpl"
        rulesets = ["http-80-tcp", "https-443-tcp"]
      }
    }

    instances = {
      web-01 = {
        enabled           = true
        type              = "t3.micro"
        availability_zone = "ap-southeast-2a"
        os_ami            = "amazon-linux-2"
        security_groups   = ["web-sg"]
      }
    }
  }
}
```

Terraform typing and validation enforce the schema.

```hcl
variable "npaas_config" {
  type = object({
    services = map(object({
      enabled                    = bool
      vpc_id                     = optional(string)
      vpc_netmask_length         = optional(number)
      vpc_private_netmask_length = optional(number)
      vpc_public_netmask_length  = optional(number)
      vpc_tgw_netmask_length     = optional(number)
      ipam_pool                  = optional(string)

      alb = optional(object({
        enabled           = bool
        port              = number
        protocol          = string
        domain            = optional(string)
        health_check_path = string
      }))

      security_groups = map(object({
        source   = string
        rulesets = list(string)
      }))

      instances = map(object({
        enabled           = bool
        type              = string
        availability_zone = string
        os_ami            = string
        security_groups   = list(string)
      }))
    }))
  })
}
```

This layer defines what the consumer can ask for.

---

## Layer 2 — Service Composition

Layer 2 translates service intent into topology.

It performs orchestration:

* filtering enabled services
* deciding whether a VPC should be created
* resolving effective VPC IDs
* conditionally creating secrets
* conditionally creating ALBs
* wiring outputs from one module into another

Example:

```hcl
locals {
  services = {
    for k, v in var.npaas_config.services :
    k => v if try(v.enabled, false)
  }

  services_with_secrets = {
    for k, v in local.services :
    k => v if try(length(v.secrets), 0) > 0
  }

  services_needing_vpc = {
    for k, v in local.services :
    k => v if try(v.vpc_id, "") == ""
  }

  effective_vpc_ids = {
    for k, v in local.services :
    k => (
      try(v.vpc_id, "") != "" ? v.vpc_id : module.vpc[k].vpc_id
    )
  }
}
```

This layer does not implement AWS resources directly.

It decides how a service is assembled.

That is what turns Terraform from a module library into a platform.

---

## Layer 3 — Platform Implementation Modules

Layer 3 contains platform modules:

* `vpc`
* `security_group`
* `instance`
* `alb`
* `secrets`

These modules are abstractions, not the bottom of the stack.

Their job is to hide platform and provider behavior behind stable interfaces.

### VPC module

The VPC module hides:

* availability zone discovery
* IPAM pool lookup
* CIDR allocation
* optional public subnet behavior
* TGW subnet allocation
* NAT behavior

The caller provides network intent.

The module implements the network.

### Instance module

The instance module hides:

* AMI lookup from an OS identifier
* subnet discovery by availability zone
* security group ID mapping
* root volume defaults

Example:

```hcl
data "aws_ami" "this" {
  for_each = local.unique_os_types

  most_recent = true
  owners      = local.ami_filters[each.value].owners

  filter {
    name   = "name"
    values = [local.ami_filters[each.value].pattern]
  }
}
```

The consumer only specifies:

```hcl
os_ami = "amazon-linux-2"
```

### ALB module

The ALB module hides:

* public subnet discovery
* ACM certificate lookup
* listener protocol behavior
* redirect behavior
* routing rule construction
* target group attachments

The caller expresses ingress intent.

The module implements load balancer behavior.

This layer defines how platform capabilities are implemented behind internal abstractions.

---

## Layer 4 — Provider / Community Implementation

Layer 4 is where Terraform touches AWS.

This includes:

* community modules such as `terraform-aws-modules/vpc/aws`
* community modules such as `terraform-aws-modules/ec2-instance/aws`
* community modules such as `terraform-aws-modules/alb/aws`
* raw `aws_*` resources
* `data.aws_*` lookups

For example, the VPC module can delegate VPC construction to a community module. The ALB module can delegate load balancer behavior the same way.

Platform modules sit above this layer and shape how provider-specific tools are used.

This is implementation substrate, not platform contract.

---

## What Layering Improves

Layering provides three benefits.

### Stable service interface

Consumers use a service contract instead of infrastructure primitives.

### Encapsulation

Lower layers absorb cloud-specific behavior.

Examples include:

* AMI resolution
* IPAM lookups
* subnet discovery
* certificate discovery
* listener and routing construction

### Internal evolution

The platform can change implementation without changing consumer intent.

The platform team can evolve:

* subnet layouts
* NAT strategy
* AMI selection logic
* ALB conventions
* community module choices

without changing the consumer contract.

---

## What Should Stay in Layer 1

Not every infrastructure parameter should move downward.

If a field expresses a service-level decision, it belongs in Layer 1.

Network sizing is one example.

Different service types may require different VPC and subnet sizes.

That is service intent, not implementation leakage.

The boundary is not:

> Layer 1 must hide all infrastructure parameters.

The boundary is:

> Layer 1 should expose service-level decisions and hide implementation mechanics.

---

## Tradeoffs

This pattern is opinionated.

The platform still relies on internal conventions such as:

* subnet tagging
* security group ruleset names
* OS name to AMI mappings
* certificate lookup conventions

These conventions simplify the consumer interface.

They also become part of the platform contract.

There is a structural tradeoff.

If composition stays in the root module for too long, Layer 2 becomes crowded.

The next move is to place service composition in an explicit `service_stack` module.

---

## Conclusion

Layering Terraform is not about splitting files.

It is about separating responsibilities.

A good platform design has:

* a typed service contract
* a composition layer that assembles services
* a platform module layer that hides implementation complexity
* a provider layer that realizes those capabilities in AWS

That turns Terraform into a platform interface.

Consumers describe what they want.

The platform decides how it is built.
