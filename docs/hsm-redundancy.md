---
title: Redundant HSMs (HA & Cloning)
parent: HSM Root of Trust (Luna)
nav_order: 4
---

# Building Redundant HSMs (HA & Cloning)
{: .no_toc }

A single HSM is a single point of failure for the **entire** CipherTrust
deployment — lose it and CM cannot unwrap its key hierarchy. This page builds the
redundant HSM layer for the reference design: **2 Luna HSMs per data center**,
across **DC1 and DC2**, all sharing one **cloning domain** so the single
root-of-trust key exists on every HSM, grouped for **high availability** so any
CipherTrust Manager can reach a surviving HSM.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The two mechanisms: cloning domain + HA group

Redundancy on Luna is built from two distinct concepts. You need both.

| Mechanism | What it provides | Where it lives |
|:----------|:-----------------|:---------------|
| **Cloning domain** | Lets key material be *replicated* between HSMs. Two partitions can clone to each other **only if they share the same domain**. | Set on each partition at initialization (a domain string, or the **red** PED key) |
| **HA group** | Lets a client *use* multiple HSMs as one virtual slot, with automatic load-balancing and failover. | Client-side, in the Luna HSM Client library |

{: .important }
> The cloning domain is the gate for **key replication**; the HA group is the
> mechanism for **client failover**. The root-of-trust key is created once, then
> **cloned** to every same-domain HSM; the HA group is what lets each CipherTrust
> Manager transparently fail over to another HSM that already holds that key.

## Cloning domain

A cloning domain is a shared cryptographic secret that Luna's **cloning protocol**
requires before it will move key material between partitions. Because the
CipherTrust root-of-trust key must exist on all four HSMs, **all four
partitions must be initialized with the same cloning domain.**

{: .warning }
> The cloning domain is set **at initialization** and cannot be changed
> afterward without re-initializing (which destroys partition contents). Decide
> and securely record your domain (or imprint your red PED key) **before**
> initializing any HSM that must participate. On PED HSMs, the **same red domain
> key** is presented when initializing every partition in the group.

## HA group

The Luna HSM Client presents member partitions to an application as a single
**virtual HA slot**:

- **Load balancing** — each cryptographic request is sent to the **least-busy**
  member. Note that **key-management operations and multi-part operations are not
  load-balanced** — a key-management op runs on one member and is then replicated
  to the others.
- **Key replication** — when a key is created in the virtual slot, it is
  **replicated to all active members**, and success is returned to the
  application only once the key exists on every active member.
- **Failover** — the client monitors members; a command that does not complete
  within roughly 20 seconds triggers failover, and pending work is transparently
  rescheduled onto surviving members. In-progress commands are kept alive with a
  keep-alive every 2 seconds to avoid false failovers.
- **Shared domain required** — every member partition must share the **same
  cloning domain** so key material can replicate across the group.

## Client registration: NTLS/STC to every member

To build an HA group, the client must have an authenticated **NTLS** (or **STC**)
connection to **each** appliance that will be a member — only then can the client
synchronize objects across the group. Register the client and assign the
partition on **every** appliance (see
[Initializing a Luna HSM]({% link docs/hsm-initialize.md %}#8-register-the-client-and-establish-the-ntlsstc-trust-link)):

```text
lunash:> client register -client <client_name> -hostname <client_host>
lunash:> client assignPartition -client <client_name> -partition <partition_name>
```

Then, on the client, create the HA group and add each member partition (as the
**Crypto Officer**):

```text
lunacm:> hagroup creategroup -label <ha_group_label> -slot <primary_slot> -password <co_password>
lunacm:> hagroup addmember   -group <ha_group_label> -slot <member_slot> -password <co_password>
lunacm:> hagroup listGroups
```

## Applying it to the DC1 / DC2 reference design

The goal: **one** root-of-trust key, present on **all four** HSMs, reachable by
**all four** CipherTrust Managers, surviving the loss of an HSM *or* a whole data
center.

```text
            shared CLONING DOMAIN across all 4 HSMs
   ┌──────────────────────────┴───────────────────────────┐
   ▼                                                       ▼
 DATA CENTER 1                                        DATA CENTER 2
 ┌───────────────────────────┐              ┌───────────────────────────┐
 │ Luna-1a   Luna-1b         │              │ Luna-2a   Luna-2b         │
 │   └──── HA group ────┘     │              │   └──── HA group ────┘     │
 └───────────────────────────┘              └───────────────────────────┘
        ▲        ▲                                  ▲        ▲
        │        │        NTLS / STC client links   │        │
     CM-1a     CM-1b                              CM-2a     CM-2b
        └──────── one CipherTrust Manager cluster ─────────┘
```

There are two common HA topologies — choose based on latency and failure-domain
requirements:

| Topology | How | Trade-off |
|:---------|:----|:----------|
| **Per-DC HA groups + shared domain** | Each DC's client uses a 2-member HA group (local HSMs); all four HSMs still share one cloning domain so the key is present everywhere for DR. | Lowest latency (crypto stays local); a full-DC loss is handled by failing CM traffic over to the other DC's CMs/HSMs, which already hold the cloned key. **Recommended.** |
| **Single stretched HA group** | One HA group spans all four HSMs across both DCs. | Simplest logical model, but every crypto op may cross the inter-DC link; sensitive to WAN latency and partitions. |

{: .tip }
> **Recommended pattern:** initialize **all four** HSMs with the **same cloning
> domain**, but build a **per-DC HA group** for the local CipherTrust Managers.
> Local crypto stays fast, and because the root-of-trust key is cloned into every
> HSM, a data-center failure is survivable — the surviving DC's HSMs already hold
> the key.

{: .warning }
> **Time sync is mandatory.** Every Luna appliance (and every CM node) must be
> synchronized to the same time source. Skew between HSMs breaks cloning/HA and
> can cause the CM HSM setup to fail. Configure NTP on each appliance
> independently — it is not replicated. See
> [Initial Appliance Setup → NTP]({% link docs/ksctl-initial-setup.md %}#ntp-servers).

## Sources

- Luna 7 — HA groups: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_partition/ha/ha.htm>
- Luna 7 — HA setup: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_partition/ha/setup.htm>
