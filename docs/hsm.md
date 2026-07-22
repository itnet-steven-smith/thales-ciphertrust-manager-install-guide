---
title: HSM Root of Trust (Luna)
nav_order: 7
has_children: true
---

# HSM Root of Trust with Thales Luna
{: .no_toc }

CipherTrust Manager (CM) protects its key material with an internal chain of key
encryption keys (KEKs). By binding the top of that chain to a **Thales Luna
Network HSM**, the HSM becomes the **root of trust** for the entire CipherTrust
deployment — the anchor that every wrapped key ultimately depends on. This
section covers the concepts, the Luna installation and initialization sequence,
building **redundant HSMs** for a **clustered** CipherTrust deployment across two
data centers, wiring the HSM to CM, and understanding **which keys are — and are
not — transmitted** when endpoints consume keys.
{: .fs-6 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What this section covers

| Page | What it covers |
|:-----|:---------------|
| [Concepts & Architecture]({% link docs/hsm-concepts.md %}) | What an HSM is, the root-of-trust model, how CM uses an HSM, supported models, and the redundant **2 CM + 2 Luna per DC** target design across DC1/DC2 |
| [Roles & Levels of Authority]({% link docs/hsm-roles.md %}) | HSM SO, Auditor, Partition SO, Crypto Officer (CO), Crypto User (CU); password vs PED (Multifactor Quorum) auth; PED key colors; M-of-N |
| [Initializing a Luna HSM]({% link docs/hsm-initialize.md %}) | The correct order of operations from factory to a usable partition, with `lunash` / `lunacm` commands |
| [Redundant HSMs (HA & Cloning)]({% link docs/hsm-redundancy.md %}) | Cloning domains, HA groups, NTLS/STC client links, and how to build per-DC and cross-DC HSM redundancy |
| [Connecting the HSM to CM]({% link docs/hsm-cm-integration.md %}) | Prerequisites, `ksctl hsm setup luna`, `rot-keys`, and sharing one root of trust across a CM cluster |
| [Keys & Endpoint Integration]({% link docs/hsm-key-transmission.md %}) | The key hierarchy, what stays inside the HSM/CM, and which keys are transmitted to KMIP, CTE, NAE, and TDE endpoints |

{: .important }
> **The HSM root of trust is destructive to set up.** `ksctl hsm setup luna
> --reset` **deletes all existing data in CipherTrust Manager** before
> re-anchoring the key hierarchy to the HSM. Establish the root of trust **early**
> — before you create production keys, register endpoints, or onboard domains.

## Scope & assumptions

This section is written for the **Luna Network HSM 7** (network-attached
appliance), which is the typical choice for anchoring a clustered, multi-data-center
CipherTrust deployment. The role model, cloning domain, and HA concepts also apply
to Luna 5/6 and TCT T-Series appliances; exact command syntax and firmware-gated
features vary by version.

{: .note }
> **Verify against official documentation.** Commands, role names, PED key
> behavior, and firmware-specific details change between releases. Validate every
> step against the Thales docs for your specific CM and Luna firmware version
> before production use:
> - CipherTrust Manager — Root of Trust HSM: <https://thalesdocs.com/ctp/cm/latest/admin/cm_admin/hardware-security-module/index.html>
> - Luna Network HSM 7: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/Home_Luna.htm>

{: .warning }
> **Root of Trust vs. CCKM keystore — do not confuse them.** CipherTrust Manager
> has *two* separate Luna integrations:
> - **Root of trust / HSM anchor** — configured with `ksctl hsm …` and
>   `ksctl rot-keys …`. **This is the subject of this section.**
> - **CCKM Luna keystore** — configured with `ksctl connectionmgmt luna-hsm …`.
>   This manages *application keys stored on a Luna* as a Cloud Key Manager
>   keystore and is **not** the root of trust. Thales discourages managing the
>   root-of-trust partition through CCKM.
