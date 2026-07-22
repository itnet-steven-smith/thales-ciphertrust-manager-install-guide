---
title: Overview
nav_order: 2
---

# What is CipherTrust Manager?
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

**Thales CipherTrust Manager (CM)** is the central management point of the
CipherTrust Data Security Platform. It is an enterprise key-management and data-
security appliance that lets an organization generate, store, and control the
cryptographic keys used to protect data across the data center and the cloud —
from a single, auditable place instead of scattered across dozens of
applications, databases, and storage systems.

At its simplest, CipherTrust Manager answers three questions for every piece of
sensitive data in an enterprise: *which key protects it, who is allowed to use
that key, and can we prove it.* It does this while acting as the policy and key
source for a family of connectors — transparent file/volume encryption,
application-layer encryption, tokenization, database TDE, and any KMIP-speaking
product — so that the encryption itself happens close to the data while the keys
stay under central control.

## Private and public cloud use cases

CipherTrust Manager is deliberately deployment-agnostic. The same product ships
as a **virtual appliance** for VMware and Microsoft Hyper-V, as **cloud images**
for AWS, Microsoft Azure, and Google Cloud, and as **physical appliances**
(the k160/k470/k570 hardware) for on-premises data centers. Because the API,
CLI, and console are identical across form factors, keys and policies behave the
same wherever the appliance runs.

**Private cloud / on-premises.** In a traditional data center you deploy CM as an
OVA on VMware vSphere/ESXi (covered in
[Deployment → Private Cloud]({% link docs/deploy-vmware.md %})). Here it typically anchors database
TDE (Microsoft SQL Server, Oracle), protects files and volumes on servers via
CipherTrust Transparent Encryption, and serves keys to storage arrays, backup
systems, and other KMIP clients — all without those keys ever leaving the
organization's control.

**Public cloud.** The same appliance runs as a cloud instance
(covered in [Deployment → Public Cloud]({% link docs/deploy-aws.md %})) so that cloud workloads get
the *same* key management and separation of duties they had on-premises. This is
the classic answer to the "hold your own key" requirement: the cloud provider
runs the infrastructure, but the customer owns and controls the encryption keys.
CipherTrust Cloud Key Manager (CCKM), a related component, extends this to manage
the native key services (BYOK) of AWS KMS, Azure Key Vault, Google Cloud KMS,
and others from the same console.

**Hybrid and multi-cloud.** Because one CM (or a clustered set of CMs) can serve
endpoints in the data center *and* in multiple clouds simultaneously, it becomes
the single point of key policy for a hybrid estate — the same key lifecycle,
the same access model, and one consolidated audit trail spanning everything.

{: .note }
> **Why centralize keys at all?** Encryption is only as strong as the control
> over its keys. Scattering keys across applications means scattering risk,
> duplicating effort, and making compliance audits painful. A dedicated key
> manager gives you one place to enforce access, rotate keys on a schedule,
> revoke access instantly, and produce a clean audit record.

## Key management

Key management is the core job. CipherTrust Manager handles the **full
cryptographic key lifecycle** — creation, distribution, rotation, versioning,
suspension, expiration, archival, and destruction — for symmetric keys,
asymmetric key pairs, certificates, and secrets. Around that lifecycle it layers:

- **Role-based access control (RBAC)** so that only authorized users, groups,
  and applications can use a given key, with clear separation of duties between,
  say, a key administrator and a system administrator.
- **Granular key policies** governing where and how a key may be used (which
  operations, from which clients, during which validity window).
- **Comprehensive auditing** — every key operation and administrative action is
  logged, which is exactly what feeds the [SIEM integration]({% link docs/siem-splunk.md %}).
- **Multi-tenancy via domains**, letting one physical/virtual appliance host
  cryptographically isolated tenants (for example, per business unit or per
  customer) that cannot see each other's keys.
- **Automation-friendly interfaces** — a REST API, the `ksctl` command-line
  client, and a web console, so keys can be managed by hand or driven from
  CI/CD and infrastructure-as-code.

The appliance ships in a free **Community Edition** by default (useful for
evaluation and labs) and unlocks additional connectors and capacity under
license.

## Root of trust with an HSM

Every key hierarchy has to be anchored to *something* that is trustworthy at the
bottom. In CipherTrust Manager that anchor is a **root of trust**, and for the
strongest assurance it is a **Hardware Security Module (HSM)**.

CipherTrust Manager protects its keys with a chain of key-encryption keys (KEKs).
When an HSM is attached, CM generates and uses a set of keys *on the HSM
partition* that protect the top of that KEK chain — those HSM-resident keys
**become the root of trust**. The master key material is created and guarded
inside dedicated, tamper-resistant, FIPS 140-2/140-3 Level 3 validated hardware
and never exists in the clear outside it. Supported roots of trust include the
Thales **Luna Network HSM**, **Luna PCIe HSM**, TCT **Luna T-Series**, **AWS
CloudHSM**, Thales **Data Protection on Demand (DPoD) Luna Cloud HSM**, **IBM
Hyper Protect Crypto Services**, and **Azure Dedicated HSM**.

The [Architecture & Root of Trust]({% link docs/architecture.md %}) page explains the key hierarchy
and the `ksctl hsm setup` procedure in detail.

## Compatibility with endpoints (KMIP, CTE, CTE-U, and more)

CipherTrust Manager is valuable precisely because so many things can consume its
keys. It talks to endpoints over several protocols and connectors:

- **KMIP (Key Management Interoperability Protocol).** The OASIS industry
  standard (typically on TCP **5696**) that lets third-party, KMIP-compliant
  products — storage arrays, backup software, databases, VMware vSphere/vSAN
  encryption, and more — request and use keys from CM without any proprietary
  agent.
- **CTE — CipherTrust Transparent Encryption.** A kernel-level agent that
  encrypts files and volumes transparently on servers, enforcing fine-grained
  access policies (which users and processes may see cleartext) while the keys
  live in CM. Applications keep working unmodified.
- **CTE-U — CipherTrust Transparent Encryption UserSpace.** A userspace
  (FUSE-based) variant of CTE that needs **no kernel module**, which makes it a
  natural fit for the many Linux distributions and for **containers and
  Kubernetes**, and lets servers be patched without rebuilding a kernel module.
- **Application-layer encryption and tokenization** via REST, PKCS#11, JCE,
  .NET, and the legacy **NAE-XML** interface (typically on TCP **9000**), so
  developers can encrypt or tokenize specific fields inside an application.
- **Database TDE** for Microsoft SQL Server, Oracle, and others, where CM holds
  the TDE master key.

The [Connecting Endpoints]({% link docs/connectors.md %}) page walks through registering clients
over each of these.

## SIEM integration, including Splunk

Because CipherTrust Manager sits at the center of an organization's encryption,
its logs are high-value security telemetry: who used which key, who changed a
policy, who logged in and from where. CM can **forward its server/audit logs and
its connectors' logs to a SIEM** so that this activity is correlated alongside
the rest of the security estate.

**Splunk** is a first-class target. Thales publishes the **Thales Security
Intelligence** app on Splunkbase, which provides pre-built indexes, source types,
and dashboards for CipherTrust data. CM forwards its records to Splunk over
**syslog** (UDP/TCP, with TLS available), and CipherTrust Transparent Encryption
can stream its per-client access logs to the same app in **RFC 5424** format.
Beyond Splunk, CM's log-forwarding feature can also target generic **syslog**
servers as well as **Loki** and **Elasticsearch** back ends.

The [SIEM Integration (Splunk)]({% link docs/siem-splunk.md %}) page provides the end-to-end
configuration for both CM server logs and CTE logs.

---

Next: [Architecture & Root of Trust →]({% link docs/architecture.md %})
