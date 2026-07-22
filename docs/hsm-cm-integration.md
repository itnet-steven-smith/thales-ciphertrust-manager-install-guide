---
title: Connecting the HSM to CM
parent: HSM Root of Trust (Luna)
nav_order: 5
---

# Connecting the HSM to CipherTrust Manager
{: .no_toc }

With the Luna partition initialized and grouped for HA, this page anchors
CipherTrust Manager to it — establishing the HSM as the **root of trust** — and
extends that single root of trust across the whole CM cluster.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

{: .important }
> **This is destructive.** `ksctl hsm setup luna --reset` **deletes all data in
> CipherTrust Manager** before re-anchoring the key hierarchy to the HSM. Do this
> **before** creating production keys, registering endpoints, or onboarding
> domains.

## Prerequisites

- The Luna partition is initialized with a **Crypto Officer** credential
  (see [Initializing a Luna HSM]({% link docs/hsm-initialize.md %})).
- **NTP** is configured on every CM node and every Luna appliance — HSM setup can
  fail on time skew.
- The Luna appliance has at least one of the required TLS ciphers enabled
  (`AES256-SHA`, `AES256-SHA256`, or `AES256-GCM-SHA384`), or CM reports a "no
  shared ciphers" error.
- Certificates were generated with **Luna HSM Client 10.3** (recommended).
- You are operating at the CM **root domain** — a Luna root of trust can only be
  added there.

You will need, from the Luna side:

- The **partition name** and the **Crypto Officer password** (the challenge
  secret for a PED/Multifactor-Quorum partition).
- The Luna **host** (`https://<luna-ip>`) and partition **serial number**.
- The **server certificate** (PEM), and either the **client certificate + key**
  (NTLS) or the **partition identity file** (STC).

## Trust channel: NTLS vs STC

| Channel | Files CM needs | Notes |
|:--------|:---------------|:------|
| **NTLS** (Network Trust Link) | `--server-cert-file`, `--client-cert-file`, `--client-cert-key-file` | The standard mutually authenticated channel. |
| **STC** (Secure Trusted Channel) | `--server-cert-file`, `--partition-identity-file` | Stronger channel; requires **Luna firmware ≥ 7.7.0**. Obtain CM's client identity with `ksctl hsm clients stcidentity download`. |

## Set up the root of trust (NTLS)

```bash
ksctl hsm setup luna --reset \
  --partition-name "partition name" \
  --partition-password "sOmeP@ssword" \
  --hsm-host https://192.168.0.1 \
  --serial 1234 \
  --server-cert-file server_cert.pem \
  --client-cert-file client_cert.pem \
  --client-cert-key-file client_cert_key.pem
```

- `--reset` re-keys CM to the HSM and is destructive (see the warning above). A
  `--delay` (default 5s) precedes the reset.
- On setup **without** `--id`, CM creates a **new** root-of-trust key with a
  randomly generated name. Passing `--id <existing_key>` makes an existing
  root-of-trust key active instead of generating new material.

{: .note }
> The `ksctl hsm setup` sub-type selects the HSM class — `luna`, `lunapci`,
> `lunatct`, `dpod`, `aws`, `ibm-hpcs`, or `k160`. This guide uses `luna` for the
> Luna Network HSM.

## Manage servers, HA, and root-of-trust keys

```bash
# List / add / remove HSM servers (partitions)
ksctl hsm servers list
ksctl hsm servers add ...
ksctl hsm servers delete --id <server_id> --reset

# Download CM's STC client identity (for STC registration on the Luna)
ksctl hsm clients stcidentity download

# Root-of-trust key lifecycle
ksctl rot-keys list
ksctl rot-keys get --id <key_id>
ksctl rot-keys rotate
ksctl rot-keys delete --id <key_id>
```

{: .note }
> **The HA group is created for you.** The **first** `ksctl hsm servers add`
> automatically creates a Luna **HA group**; each subsequent `servers add`
> inserts that partition into the HA group. Add the second local HSM (and, per
> your DR strategy, remote HSMs) so CM fails over automatically.

## Extending one root of trust across the CM cluster

All nodes in a CipherTrust Manager cluster should share **one** root of trust.
Two models exist:

| Model | Flag | Meaning |
|:------|:-----|:--------|
| **Shared HSM** *(recommended)* | join with `--shared-hsm-partition` | Every cluster node uses the **same** HSM partition and the same root-of-trust keys. Thales recommends this "for the best security." |
| **Hybrid HSM** | (default / no shared flag) | Nodes may use different HSMs — or none. Convenient, but Thales warns it **reduces overall security** (the system is only as strong as its weakest node). Available since release 1.3.0. |

{: .tip }
> For the DC1/DC2 reference design, use a **shared HSM partition** across the CM
> cluster and back it with the cross-DC cloned root-of-trust key. Because all
> four Luna HSMs share a [cloning domain]({% link docs/hsm-redundancy.md %}), the
> "shared partition" is really the single cloned root-of-trust key reachable
> through each DC's HA group — so any CM node can anchor to any surviving HSM.

## Wiring the reference design

1. Initialize all four Luna partitions with the **same cloning domain** and a
   **Crypto Officer** credential.
2. In each DC, build a **per-DC HA group** of the two local HSMs.
3. Run `ksctl hsm setup luna --reset ...` **once** to establish the root of
   trust, then `ksctl hsm servers add` to bring the second (and remote) HSMs into
   the HA group.
4. Join the CM cluster nodes with `--shared-hsm-partition` so all four CMs share
   the one root of trust.
5. Confirm with `ksctl rot-keys list` and `ksctl hsm servers list`.

## Backups tied to the HSM

CM backups can be bound to a specific HSM with `--tied-to-hsm`, so a backup can
only be restored on a system with access to that partition. Because the
root-of-trust key is cloned across all same-domain HSMs, such a backup remains
restorable in either data center.

## Sources

- CipherTrust Manager — Root of Trust HSM: <https://thalesdocs.com/ctp/cm/latest/admin/cm_admin/hardware-security-module/index.html>
