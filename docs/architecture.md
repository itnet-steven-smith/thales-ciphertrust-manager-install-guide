---
title: Architecture & Root of Trust
nav_order: 3
---

# Architecture & Root of Trust
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Before deploying, it helps to understand how CipherTrust Manager protects keys
internally and how endpoints connect to it. This shapes several decisions you'll
make during installation — sizing, networking, and whether to attach an HSM.

## The key hierarchy

CipherTrust Manager never protects your data keys with a single flat secret.
Instead it uses a layered hierarchy of **key-encryption keys (KEKs)**:

```
                 ┌─────────────────────────────┐
                 │   Root of Trust (RoT)       │   ← HSM-resident (optional but
                 │   master/wrapping keys      │     recommended for production)
                 └──────────────┬──────────────┘
                                │ protects
                 ┌──────────────▼──────────────┐
                 │   KEK chain (internal)      │   ← wraps the keys below
                 └──────────────┬──────────────┘
                                │ protects
                 ┌──────────────▼──────────────┐
                 │   Data Encryption Keys      │   ← served to endpoints:
                 │   (DEKs), certs, secrets    │     CTE, KMIP clients, apps…
                 └─────────────────────────────┘
```

Each layer wraps (encrypts) the layer beneath it. The practical benefit is that
you can rotate the top of the chain — or move the root into an HSM — without
re-encrypting all of the data below, and no lower-level key is ever stored in the
clear.

## Root of trust with an HSM

By default, a fresh CipherTrust Manager uses a **software root of trust**: the
top of the KEK chain is protected by keys held by the appliance itself. That is
fine for evaluation, but for production the recommended posture is to move the
root of trust into a **Hardware Security Module (HSM)**.

When an HSM is attached, CM "generates and uses a set of keys on the HSM
partition that protect the KEK chain and become the root of trust." In other
words, the master keys that sit at the very top of the hierarchy are created and
used **inside** dedicated, tamper-resistant hardware and never leave it in the
clear. If the CM VM's disk were somehow stolen, the wrapped keys on it would be
useless without the HSM.

### Supported HSMs

| HSM | Typical use |
|:----|:------------|
| Thales **Luna Network HSM** (v5/6/7) | On-prem network-attached HSM |
| TCT **Luna T-Series Network HSM** | US/Trusted Cyber Technologies variant |
| **Luna PCIe HSM** | HSM card inside the appliance/host |
| **AWS CloudHSM** | Root of trust for CM running in AWS |
| Thales **DPoD Luna Cloud HSM** | HSM-as-a-service (Data Protection on Demand) |
| **IBM Hyper Protect Crypto Services** | Root of trust in IBM Cloud |
| **Azure Dedicated HSM** | Root of trust in Azure |

{: .warning }
> **HSM setup is destructive.** Establishing an HSM root of trust performs a
> full system reset of CipherTrust Manager — all existing data on the appliance
> is wiped. Do this on a **fresh** appliance, or take a complete backup first.
> Because of this, decide on your root-of-trust strategy *before* you load keys
> and register clients.

### How it's configured (preview)

Root of trust is established from the `ksctl` CLI (covered in
[Initial Configuration]({% link docs/initial-configuration.md %})). The high-level command shape is:

```bash
# Attach a Luna Network HSM partition as the root of trust (DESTRUCTIVE — resets CM)
ksctl hsm setup luna \
  --reset \
  --partition-name my_partition \
  --partition-password '<partition-password>' \
  --server '<hsm-ip>' \
  --server-cert-file /path/to/server.pem

# List configured HSM servers
ksctl hsm servers list

# Rotate the root-of-trust keys later, without re-encrypting data below
ksctl rot-keys rotate
```

Valid HSM types for `ksctl hsm setup` include `luna`, `lunapci`, `lunatct`,
`aws`, `dpod`, and `ibm-hpcs`. Exact parameters vary per HSM — always follow the
Thales guide for your specific model.

## How endpoints connect

The other half of the architecture is *outward-facing*: how the systems that
consume keys reach CM. Each connector uses its own protocol and, usually, its own
network port.

```
                         ┌───────────────────────────┐
   Admin (you) ── 443 ──▶│                           │
   ksctl / REST ─ 443 ──▶│    CipherTrust Manager    │◀── HSM (root of trust)
                         │                           │
   KMIP clients ─ 5696 ─▶│  keys · policies · audit  │
   CTE / CTE-U agents ──▶│                           │──▶ Syslog/Splunk (SIEM)
   Apps (NAE-XML) 9000 ─▶│                           │
                         └───────────────────────────┘
```

| Consumer | Protocol / interface | Default port |
|:---------|:---------------------|:-------------|
| Administrators, `ksctl`, REST API, web console | HTTPS | **443** |
| Console/SSH management (`ksadmin`) | SSH | **22** |
| Third-party KMIP products | KMIP | **5696** |
| Applications / tokenization (legacy) | NAE-XML | **9000** |
| CTE / CTE-U agents | proprietary over TLS to CM | 443 (registration) |
| SIEM (outbound) | Syslog (UDP/TCP/TLS) | 514 / 5514 / 6514 |

These are the ports you'll open in your VMware network or AWS security group
during [deployment]({% link docs/deployment.md %}). Open only what you actually use.

{: .note }
> **Clustering.** Production deployments usually run **two or more CipherTrust
> Manager nodes in a cluster** for high availability, sharing keys and policies.
> This guide focuses on a single node so you can get one healthy appliance
> online first; add nodes afterward using the same image and the clustering
> workflow in the Thales documentation.

---

Next: [Prerequisites →]({% link docs/prerequisites.md %})
