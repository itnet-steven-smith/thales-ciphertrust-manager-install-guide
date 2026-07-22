---
title: Home
layout: home
nav_order: 1
description: "Step-by-step guide to installing and configuring a Thales CipherTrust Manager virtual appliance on VMware and AWS, with commands, sample output, and explanations."
permalink: /
---

# Thales CipherTrust Manager — Installation & Setup Guide
{: .fs-9 }

A practical, command-level walkthrough for standing up a CipherTrust Manager
virtual appliance from zero — on a private cloud (VMware) and a public cloud
(AWS) — with the exact commands, sample output, and an explanation of *what each
step actually does* and *why it matters*.
{: .fs-6 .fw-300 }

[Start with the Overview]({% link docs/overview.md %}){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Jump to Deployment]({% link docs/deployment.md %}){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## Who this guide is for

Security engineers, platform teams, and administrators who need to bring a
**CipherTrust Manager** (CM) key-management appliance online and connect it to
the systems that will consume its keys. It assumes comfort with a hypervisor or
cloud console and a terminal, but it explains the CipherTrust-specific pieces in
full.

## What you'll accomplish

By the end you will have:

1. A running CipherTrust Manager virtual appliance, deployed either on **VMware
   vSphere/ESXi** or **AWS**.
2. Console and network configuration completed, with the **web console** and
   **`ksctl` CLI** reachable.
3. The default administrator credentials rotated and a hardened first login.
4. An understanding of how to register endpoints over **KMIP**, **CTE**, and
   **CTE-U**, and how to establish an **HSM root of trust**.
5. CipherTrust Manager (and CTE) audit logs flowing into a **SIEM such as
   Splunk**.

## How the guide is organized

| Section | What it covers |
|:--------|:---------------|
| [Overview]({% link docs/overview.md %}) | What CipherTrust Manager is, its cloud use cases, and key-management capabilities |
| [Architecture & Root of Trust]({% link docs/architecture.md %}) | The key hierarchy, HSM root of trust, and how endpoints connect |
| [Prerequisites]({% link docs/prerequisites.md %}) | Sizing, downloads, network, and access you need before you start |
| [Deployment → Private Cloud (VMware)]({% link docs/deploy-vmware.md %}) | Deploying the OVA on vSphere/ESXi |
| [Deployment → Public Cloud (AWS)]({% link docs/deploy-aws.md %}) | Deploying the Marketplace/AMI image on EC2 |
| [Initial Configuration]({% link docs/initial-configuration.md %}) | First boot, networking, web console, `ksctl`, hardening |
| [Connecting Endpoints]({% link docs/connectors.md %}) | KMIP, CTE, CTE-U, and other protocols |
| [SIEM Integration (Splunk)]({% link docs/siem-splunk.md %}) | Forwarding CM and CTE logs to Splunk |
| [Automation with KSCTL]({% link docs/ksctl.md %}) | Command-line automation of CM with `ksctl` — install to CTE, with all commands and examples |
| [Verification & Next Steps]({% link docs/verification.md %}) | Confirming a healthy install and where to go next |

{: .note }
> Commands and sample output in this guide are drawn from Thales' public
> documentation for CipherTrust Manager 2.x. Exact strings (IP addresses,
> interface names such as `ens4`/`eth0`, and version numbers) will differ in
> your environment — treat them as templates, not literals.
