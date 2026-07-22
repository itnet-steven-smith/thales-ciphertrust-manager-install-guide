---
title: Roles & Levels of Authority
parent: HSM Root of Trust (Luna)
nav_order: 2
---

# Roles & Levels of Authority on a Luna HSM
{: .no_toc }

Luna enforces **separation of duties** through distinct roles, split across two
levels: roles that administer the **HSM/appliance itself**, and roles that
perform **cryptography inside a partition**. The roles at each level are
independent — the HSM Security Officer does not automatically control the
contents of a partition, and a Crypto Officer cannot re-administer the HSM.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## HSM-level roles

These roles administer the appliance and the HSM as a whole.

| Role | Abbrev. | What it can do |
|:-----|:--------|:---------------|
| **HSM Security Officer** | HSM SO / SO | Initializes the HSM and sets the SO credential; sets and changes global HSM policies; **creates and deletes application partitions**; updates HSM firmware. The top administrative authority for the HSM itself. |
| **Auditor** | AU | Initializes and controls **audit logging** — sets log file size and rotation, archives logs. Deliberately isolated: **no one else, including the HSM SO, can clear the audit logs.** Provides independent oversight. |

{: .note }
> Separately from HSM roles, the Luna **appliance OS** (LunaSH) has its own login
> accounts — `admin`, `operator`, `monitor`, and `audit` — used to manage the
> network appliance. These are not the same as the cryptographic HSM roles above.

## Partition-level roles

These roles live inside an application partition and govern cryptographic
objects (keys). This is where a **Crypto Officer (CO)** — the role most people
mean when they say "the CO of the HSM" — operates.

| Role | Abbrev. | What it can do |
|:-----|:--------|:---------------|
| **Partition Security Officer** | PO / Partition SO | **Initializes the partition**, sets the PO credential, and sets the partition's **cloning domain**; configures partition policies; activates the partition; **initializes the Crypto Officer**. |
| **Crypto Officer** | CO | The primary partition user. **Creates, deletes, and modifies keys/objects**; performs cryptographic operations; manages backup/restore; **creates and configures HA groups**; initializes the Crypto User. This is the credential CipherTrust Manager uses to drive the root-of-trust partition. |
| **Limited Crypto Officer** | LCO | A compliance role with a **subset** of CO abilities — more than a Crypto User, less than a CO. Notably **cannot clone/replicate** objects in any scenario. |
| **Crypto User** | CU | Optional **read-only** role. Uses existing keys for crypto operations and can create only *public* objects; cannot manage the partition or delete/modify others' keys. |

### The authority hierarchy at a glance

```text
        HSM Security Officer (SO)          ← initializes HSM, creates partitions
                 │  creates partition
                 ▼
        Partition Security Officer (PO)    ← initializes partition + sets cloning domain,
                 │  initializes CO             creates the Crypto Officer
                 ▼
        Crypto Officer (CO)                ← full key management; the credential CM uses
                 │  initializes CU
                 ▼
        Crypto User (CU)                   ← read-only cryptographic use

        Auditor (AU)  ── independent oversight, outside the chain above
```

{: .tip }
> **Separation of duties, applied.** In production, the person who administers
> the HSM (HSM SO) should not be the same person who manages keys in the
> partition (CO), and neither should control the audit trail (Auditor). Assign
> each role to a different individual (or M-of-N quorum) and store credentials in
> a privileged access solution.

## Password authentication vs. PED (Multifactor Quorum) authentication

Every Luna HSM is one of two authentication types, chosen at purchase — this
determines how each role above proves its identity.

### Password-authenticated HSM

Knowledge of the role **password** is sufficient to authenticate. Simple to
automate, no physical tokens. All role secrets are passwords.

### PED-authenticated HSM (Multifactor Quorum)

Authentication requires possession of a physical **PED key** (an iKey USB token)
presented on a **Luna PED (PIN Entry Device)** — "something you have." An
optional **PED PIN** adds "something you know" for true two-factor. The secrets
live **only on the portable PED keys**; the PED device itself stores nothing.
Luna 7 documentation rebrands PED authentication as **Multifactor Quorum
Authentication**.

#### PED key colors

| Color | Purpose |
|:------|:--------|
| **Blue** | HSM SO or Partition SO (Security Officer) authentication |
| **Black** | Crypto Officer authentication |
| **Gray** | Crypto User (and Limited Crypto Officer) authentication |
| **Red** | **Cloning Domain** vector — defines the set of HSMs/partitions that may clone to one another |
| **Orange** | **Remote PED Vector (RPV)** — establishes a connection to a Remote PED server |
| **White** | Auditor authentication |
| **Purple** | **Secure Recovery Key (SRK)** — a split of the tamper/transport recovery secret (largely a **legacy** concept on Luna 7; verify for your firmware) |

{: .note }
> The **red domain key** is central to redundancy: two partitions can clone key
> material to each other **only if they share the same cloning domain**. Every
> Luna that must hold a copy of the CipherTrust root-of-trust key is initialized
> with the *same* red domain key (or the same domain string on a
> password-authenticated HSM). See
> [Redundant HSMs]({% link docs/hsm-redundancy.md %}).

### M of N — split-knowledge quorum

On PED-authenticated HSMs, a single role's secret can be **split across up to 16
PED keys**, requiring a minimum quorum (**M of N**, e.g. "3 of 5") to
authenticate. No single key-holder can act alone, while the deployment tolerates
absent holders. All parties must be present when the split is first created.
M-of-N is unavailable on password-authenticated HSMs.

## Sources

- Luna 7 — HSM roles: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_hsm/hsm_roles/hsm_roles.htm>
- Luna 7 — Partition roles: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_partition/partition_roles/partition_roles.htm>
- Luna 7 — Multifactor Quorum (PED) authentication: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_hsm/PED_Auth/PED_Auth.htm>
