---
title: Initializing a Luna HSM
parent: HSM Root of Trust (Luna)
nav_order: 3
---

# Initializing a Luna HSM — Order of Operations
{: .no_toc }

Initializing a Luna Network HSM follows a strict sequence. Each step establishes
a role or trust relationship that the next step depends on, so **order matters**.
This page walks the sequence from a factory-fresh appliance to a usable,
initialized partition — the foundation for anchoring CipherTrust Manager.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

{: .note }
> Commands are shown for the appliance shell (`lunash:>`) and the client-side
> **LunaCM** (`lunacm:>`). Exact command splits (particularly which interface
> initializes the Partition SO) shift slightly across firmware/appliance versions
> — notably around 7.8.x. Confirm against your Luna firmware docs.

## The sequence at a glance

```text
 1. Install hardware, verify tamper / Secure Transport state
 2. Configure the appliance network (LunaSH)
 3. Generate the appliance server certificate      (sysconf regenCert)
 4. (PED HSMs) Set up the Luna PED / Remote PED
 5. Recover the HSM from Secure Transport Mode (STM)
 6. Initialize the HSM  ── sets HSM SO + cloning domain   (hsm init)
 7. Create the application partition                (partition create)
 8. Install Luna HSM Client + register NTLS/STC trust link
 9. Initialize the partition ── Partition SO + domain   (partition init)
10. Initialize the Crypto Officer (CO)              (role init -name co)
11. (optional) Initialize the Crypto User (CU)      (role init -name cu)
```

Steps 8–11 are where the **client trust link** and the **Crypto Officer** are
established — the CO credential is exactly what CipherTrust Manager will use in
[Connecting the HSM to CM]({% link docs/hsm-cm-integration.md %}).

---

## 1–2. Hardware and network

Rack, power, and cable the appliance. On first access (serial console or SSH to
**LunaSH**), configure the network — hostname, IP, DNS, and gateway — using the
LunaSH `network` commands. Verify the shipment's tamper state.

## 3. Generate the appliance server certificate

The appliance presents this certificate to clients when they establish the
network trust link (NTLS). Regenerate it so it reflects the appliance's real
identity:

```text
lunash:> sysconf regenCert
```

Restart NTLS afterward if prompted. This certificate is later exchanged with
each CipherTrust Manager client.

## 4. (PED HSMs only) Set up the Luna PED

For a PED-authenticated (Multifactor Quorum) HSM, connect a local Luna PED or
configure Remote PED before initializing — you will be prompted to present and
imprint PED keys during `hsm init`. Have the **blue** (SO) and **red** (domain)
keys ready. See [Roles & Authority]({% link docs/hsm-roles.md %}#ped-key-colors).

## 5. Recover from Secure Transport Mode

New Luna HSMs ship in **Secure Transport Mode (STM)** so tampering in transit is
detectable. Verify the transport verification string provided by Thales, then
recover the HSM from STM before initializing.

## 6. Initialize the HSM

Initializing sets the **HSM Security Officer** credential and the HSM's
**cloning domain**. On a password HSM you supply a domain string; on a PED HSM
you imprint the **red domain key**.

```text
lunash:> hsm init -label <HSM_label>
```

{: .warning }
> The **cloning domain** you set here (and, more importantly, the domain you set
> per partition in step 9) determines which other HSMs this one can clone key
> material to. **Every Luna that will hold a copy of the CipherTrust root-of-trust
> key must share the same cloning domain.** Decide your domain strategy *before*
> initializing — see [Redundant HSMs]({% link docs/hsm-redundancy.md %}).

## 7. Create the application partition

Logged in as the **HSM SO**, create the partition that will hold the
root-of-trust key:

```text
lunash:> partition create -partition <partition_name>
```

## 8. Register the client and establish the NTLS/STC trust link

Install the **Luna HSM Client** on the machine that will connect (for the root of
trust, this trust relationship is ultimately consumed by CipherTrust Manager).
Establish a mutually authenticated channel between client and appliance:

- **NTLS (Network Trust Link Service)** — the client and appliance exchange and
  register certificates, then the partition is assigned to the client.
- **STC (Secure Trusted Channel)** — a stronger, mutually authenticated channel;
  requires **Luna firmware ≥ 7.7.0**.

On the appliance, register the client and assign the partition:

```text
lunash:> client register -client <client_name> -hostname <client_host>
lunash:> client assignPartition -client <client_name> -partition <partition_name>
```

{: .tip }
> Thales recommends using **Luna HSM Client 10.3** when generating the
> certificates used to connect CipherTrust Manager, for compatibility.

## 9. Initialize the partition (Partition SO + domain)

From the client side in **LunaCM**, initialize the partition. This creates the
**Partition Security Officer** credential and sets the partition's **cloning
domain** (the shared secret / red key that governs redundancy):

```text
lunacm:> partition init -label <partition_label>
```

## 10. Initialize the Crypto Officer

Log in as the Partition SO and initialize the **Crypto Officer** — the role that
manages keys and that CipherTrust Manager authenticates as:

```text
lunacm:> role login -name po
lunacm:> role init -name co
```

## 11. (Optional) Initialize the Crypto User

For read-only separation of duties, log in as the CO and initialize the
**Crypto User**:

```text
lunacm:> role login -name co
lunacm:> role init -name cu
```

---

At this point you have an initialized partition with a **Crypto Officer**
credential and an authenticated client trust link. Repeat this process for every
Luna appliance, **using the same cloning domain**, then group them for high
availability in [Redundant HSMs]({% link docs/hsm-redundancy.md %}).

## Sources

- Luna 7 — Configuration sequence: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/Configure.htm>
- Luna 7 — Initialize the HSM: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_hsm/hsm_init.htm>
- Luna 7 — Initialize CO/CU: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_partition/partition_roles/init_co_cu.htm>
