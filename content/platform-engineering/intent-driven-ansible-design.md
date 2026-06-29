+++
title = "Intent-Driven Ansible Through Metadata Contracts"
permalink = "/platform-engineering/intent-driven-ansible-design/"
date = "2026-05-22"
tags = ["ansible", "terraform", "platform-engineering", "automation", "configuration-management"]
categories = ["platform-engineering"]
author = "Chris Zhang"
+++

## Problem

Ansible is procedural by design. That is not inherently a problem.

The problem starts when the playbook becomes responsible for deciding
what kind of system it is building.

As application families grow, orchestration logic accumulates branches:

* environment-specific conditionals
* hostname checks
* package selection logic
* server-role branching
* application-specific execution paths

The playbook stops being an execution engine and becomes a decision
engine.

It grows harder to reason about, harder to test, and harder to extend
without introducing unintended behavior.

A different pattern is to move decisions into configuration and keep the
orchestration layer thin.

Intent-driven Ansible means the playbook receives intent, resolves it
through structured configuration, and executes roles.

It should not contain application-specific branching logic.

---

## This is not declarative infrastructure

Terraform models desired state.

Ansible executes procedure.

Intent-driven design does not change that.

It does not eliminate procedural automation. It relocates
decision-making out of orchestration and into structured configuration.

From the outside, the model looks declarative:

```text
Define intent → Resolve configuration → Execute roles
```

But execution is still sequential task application.

The distinction matters when reasoning about:

* ordering
* idempotency
* failure recovery
* installer side effects
* host-level dependencies

This is not desired-state infrastructure.

It is structured procedural automation.

---

## Architecture

```text
Service intent
  ↓
Contract metadata
  ↓
Runtime lookup
  ↓
Role composition
  ↓
Host configuration resolution
```

Each layer has a defined responsibility.

The infrastructure layer provides metadata.

Ansible consumes that metadata and resolves host configuration.

The orchestration layer does not own decision logic.

---

## Pattern 1 — A single orchestration entry point

One playbook handles all application types and all environments.

```yaml
- hosts: localhost
  connection: local
  gather_facts: true

  vars:
    roleList: "{{ (['baseline'] + (configRoles.split(','))) | unique }}"
    roleData: "{{ lookup('file', playbook_dir ~ '/../config/roles.json') | from_json }}"

  tasks:
    - name: "Apply config role(s)"
      include_role:
        name: "{{ config_role }}"
      loop: "{{ roleList | map('extract', roleData, morekeys='roles') | list | flatten }}"
      loop_control:
        loop_var: config_role
```

The playbook has no application-specific knowledge.

It reads intent, expands roles, executes.

That is all.

`baseline` is always prepended so every host receives a consistent
baseline.

The orchestration layer remains stable even as application families
grow.

---

## Pattern 2 — Role composition defines system identity

`roles.json` defines what a system is.

```json
"web-application": {
  "roles": ["base-tools", "runtime-core", "web-application", "web-gateway"]
},
"worker-application": {
  "roles": ["base-tools", "runtime-core", "worker-application"]
},
"reporting-application": {
  "roles": ["web-server", "reporting-runtime", "deployment-agent"]
},
"document-service": {
  "roles": ["document-service"]
}
```

This is structural identity.

Adding a new application profile means changing configuration, not
orchestration.

Intent is expressed as composition.

Procedure remains inside roles.

---

## Pattern 3 — Platform primitives behind a common interface

The `common` role is not a utility role.

It is the abstraction boundary between application roles and platform
dependencies.

Application roles should not know:

* how secrets are retrieved
* how metadata is read
* how packages are fetched
* how trust is installed
* how shared host-level dependencies are configured

They call platform primitives instead.

Examples include:

* metadata lookup
* secret retrieval
* package retrieval
* certificate installation
* cloud account discovery
* service-account configuration

Platform logic stays centralized.

Application roles stay focused on application behavior.

---

## From infrastructure intent to Ansible configuration

This design does not begin in Ansible.

Infrastructure provisioning already knows what kind of system is being
built.

That intent can be published as controlled metadata on the instance.

For example, the infrastructure layer can provision an instance with:

```text
Environment = nonprod
```

At runtime, Ansible retrieves that metadata and uses the value as a
configuration lookup key:

```text
Environment → environmentConfig["nonprod"]
```

From that lookup, Ansible resolves:

* secrets
* domains
* storage paths
* application hostnames
* package configuration

The infrastructure layer does not tell Ansible how to configure the host.

It emits metadata.

Ansible interprets that metadata through its own configuration model.

---

## Cross-layer schema contract

Infrastructure provisioning and Ansible remain separate control planes.

They do not call each other directly.

They share a contract through infrastructure metadata.

The infrastructure layer produces:

```text
Environment = nonprod
```

Ansible consumes:

```text
environmentConfig["nonprod"]
```

This creates a shared schema contract:

```text
Infrastructure metadata schema
        ↓
Infrastructure metadata
        ↓
Runtime API lookup
        ↓
Ansible variable map
        ↓
Configuration behavior
```

The metadata key must match what Ansible expects.

The metadata value must match a key in Ansible's environment map.

This is not procedural coupling.

It is contract-based metadata integration.

---

## Pattern 4 — Instance identity becomes configuration

A shared runtime role is where instance identity becomes configuration.

At execution time, the role reads the instance's environment identity:

```yaml
tagKey: Environment
```

From that one metadata value, it resolves:

* environment suffix
* domains
* secret paths
* application hostnames
* data roots
* storage configuration

Its own identity determines configuration.

A differently tagged instance becomes a different environment without
changing automation logic.

---

## Pattern 5 — Divergence through data, not orchestration branching

Two application roles can share:

* the same orchestration model
* the same shared base role
* the same task structure

They diverge through variables.

The procedural installation logic remains largely the same.

What changes is the data model:

* package metadata
* installer arguments
* artifact coordinates
* runtime configuration values

Application identity is expressed through configuration rather than
orchestration branching.

---

## Pattern 6 — Convention-based runtime profile resolution

Some roles use a different pattern.

Instead of explicit role composition, it infers server profile at
runtime from hostname.

```yaml
matched_profile:
  {{ (profile_names | select('in', hostname) | list)[0] }}
```

From that match, the role derives:

* package profile
* host feature set
* application role combination

This trades explicit configuration for convention-based inference.

Both derive behavior from metadata rather than orchestration branching.

---

## Identity conventions as configuration contracts

This design keeps orchestration simple by deriving configuration from
instance identity.

That identity comes from two sources:

* **environment metadata** — deployment context
* **hostname conventions** — server profile identity

Examples:

**Environment metadata**

```text
Environment → environmentConfig → secrets / domains / storage / config
```

**Hostname convention**

```text
hostname → matched_profile → package profile / feature set
```

This is a deliberate trade.

The orchestration layer avoids branching because configuration identity
is derived from conventions.

That simplifies execution, but it makes identity conventions
load-bearing.

The design is not tightly coupled to procedural logic.

It is coupled to identity contracts.

---

## What makes it intent-driven

The important property is not the specific metadata source.

It can be an instance tag, a hostname convention, an inventory value, or
another controlled input.

The important property is where decisions live.

In an intent-driven Ansible design:

* the entry playbook accepts intent
* configuration maps intent to roles and variables
* roles contain the procedural implementation
* platform primitives hide shared mechanics
* application growth changes data before it changes orchestration

The playbook stays boring.

That is the point.

---

## Why use this pattern at all?

Terraform can provision:

* compute
* networking
* IAM
* infrastructure boundaries

It does not solve:

* package installation
* service account binding
* feature configuration
* host-level application bootstrap
* post-provisioning configuration

That still requires procedural host automation.

Intent-driven host configuration makes that procedural layer more
structured.

It complements provisioning infrastructure.

---

## Conclusion

Intent-driven host configuration does not make procedural automation
declarative.

It makes procedural automation structured.

Terraform resolves infrastructure intent and emits metadata.

Ansible consumes that metadata and resolves host behavior.

New application families do not require orchestration branching.

Most application growth moves into:

* configuration
* role composition
* runtime metadata resolution

rather than into the playbook itself.

The orchestration layer stops accumulating decision logic.

It becomes stable.

That is the architectural payoff.
